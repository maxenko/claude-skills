# GPU Usage, Data-Loading & Duplicate-I/O Efficiency

Performance patterns for keeping a GPU fed and never doing the same I/O twice. Target context:
Python ML inference (ONNX Runtime, batched frame scanning in a media-moderation service);
principles are framework-agnostic, ORT-specifics flagged.

**Governing idea:** the GPU is the expensive resource and **it should never wait.** Almost every
technique here removes a *stall* — silicon idle while data is prepared, copied, re-decoded, or
re-loaded. **Corollary:** you don't get to claim a stall exists until you've *measured* it (§F).
Treat this file like the pattern catalogue: each item names the smell, the cost it removes, and the
over-optimization trap.

---

## A. Keeping the GPU fed
- **A1 — Recognize GPU starvation.** The defining failure mode: GPU finishes a batch and idles while the CPU prepares the next. *Signal:* GPU util **<70% is a flag, ~30% a red flag** (`nvidia-smi dmon -s u`); PyTorch Profiler trace shows gaps between kernels = data-loading tax. *Mistake:* reading "VRAM full" as "GPU busy" — memory occupancy and compute utilization are orthogonal.
- **A2 — Overlap compute with transfer.** Prepare+copy batch N+1 while computing batch N (prefetch / double-buffer; under the hood: CUDA streams + async copies). NVIDIA's canonical demo: **8.7 ms vs 12.9 ms** purely from overlap. Three *mandatory* conditions: concurrent copy/exec support, copy+kernel on **different non-default streams**, **pinned** host memory. *Mistake:* issuing async copy + kernel on the same (default) stream and expecting overlap — they serialize.
- **A3 — Pinned memory + non-blocking copies.** Pageable host memory can't be DMA'd directly — the driver stages it through a temp pinned buffer (two copies); pinned also is the precondition for a copy to actually be async. PyTorch: `DataLoader(pin_memory=True)` + `.to(device, non_blocking=True)`. *Mistakes:* over-pinning (starves the OS), assuming `non_blocking` from pageable memory is async (it isn't), reading the copy result before the stream syncs.
- **A4 — Keep data on the GPU.** Don't round-trip tensors to CPU between ops; every hop is a PCIe transfer + an implicit sync that kills overlap. *Mistake:* a stray `.item()`/`.cpu()`/`.numpy()` or a Python-side `if tensor > x` in the inner loop — each forces a device sync.

## B. Batching for throughput
- **B1 — Why batching wins.** Amortizes per-launch kernel overhead + per-transfer setup, and fills SM occupancy. Trade-off is **latency** (a request waits for the batch) — tune the knee, not the extreme. *Mistake:* batch size 1 in prod "for simplicity," leaving the GPU 60–80% idle per call.
- **B2 — Dynamic batching / request coalescing (server-side).** Collect concurrent in-flight requests into one GPU batch transparently. Two knobs (Triton naming, universal concept): **`max_batch_size`** (ceiling) and **`max_queue_delay`** (how long to wait for more requests before firing). For a media-mod service, build a lightweight version: a queue + short coalescing window + one batched `Run()`. *Mistake:* `max_queue_delay` so high that low-traffic requests wait for a batch that never fills → inflated p99.
- **B3 — Padding/bucketing variable sizes.** To batch variable-size inputs you pad to a common shape; minimize waste by **bucketing** similar sizes. Pre-resizing frames to the model's fixed input sidesteps it entirely. *Mistake:* one global batch with a wide size spread — the tail item dictates every item's cost.
- **B4 — Choosing batch size.** Bounded above by VRAM; diminishing returns past the occupancy knee. *Signal:* memory util >90% → room for bigger batches; if GPU util is already high and throughput plateaued, bigger batches only add latency. Sweep powers of two, plot throughput vs latency, pick the knee within budget; re-check after a precision change. *Mistake:* copying a batch size from a blog on different hardware.

## C. Data-loading pipeline
- **C1 — Parallel loading.** Multiple worker processes read+decode+preprocess in parallel and prefetch ahead. PyTorch: `num_workers` (start ~core count, **4–8 common sweet spot**), `prefetch_factor` (default 2; total prefetched ≈ factor × workers), `persistent_workers=True` (avoid per-epoch respawn). *Mistake:* cranking workers blindly — documented cases where **more workers performed *worse* than `num_workers=0`** (oversubscription, IPC/serialization, memory). Measure; scaling isn't monotonic.
- **C2 — Decode/preprocess on the right device.** CPU decode (JPEG→crop→resize→normalize) can become the throughput ceiling. NVIDIA **DALI/nvJPEG** runs the whole decode/augment pipeline on the GPU with prefetch. *Mistake:* reaching for GPU decode before profiling shows decode is the bottleneck — if you're already GPU-bound it steals model cycles and makes things *worse*.
- **C3 — Sequential reads; shard, don't scatter.** Random access to millions of tiny files is **3–10× slower** than sequential, far worse over object storage (per-file round-trips). Pack samples into **tar (WebDataset)** or **parquet** shards → large sequential reads + parallel I/O across workers (8 workers/8 shards ≈ 8×). *Mistake:* a directory of millions of `.jpg` on a network filesystem — metadata/open overhead dominates.
- **C4 — Format & dtype.** fp16/bf16 **halves PCIe bytes + VRAM** and **~doubles Tensor-Core throughput**; int8 quantization further. **Channels-last (NHWC)** for CV models: PyTorch reports **8–35% speedups** under mixed precision. *Mistakes:* **channel thrashing** (mixing NHWC with ops that force NHWC↔NCHW conversions costs more than it saves); dropping to int8 without validating accuracy — a false-negative on harmful content is not free.

## D. Avoiding duplicate I/O *(first-class theme)*
Unifying principle: **read once, reuse many — identical bytes should be decoded, transferred, and inferred exactly once.** High-value for a moderation service where the same asset is re-submitted, re-scanned, or scanned by multiple models.
- **D1 — Content-addressed caching.** Cache decoded tensors and/or model outputs keyed by `hash(content) + preprocessing_params`. Identical input → cache hit → skip the work (a hit avoids the forward pass entirely; LRU+TTL serves sub-ms). *Mistake:* keying on filename/URL — a re-upload under a new name misses; a mutated file under the same name returns stale.
- **D2 — Single-flight (dedup in-flight requests).** If two callers ask for the same key while the first is computing, only **one** load/inference runs; the rest await the same result. Without it, a burst of N identical requests on a cold cache triggers N redundant GPU runs — exactly when you're busy. Implement with a per-key in-flight map of futures (`asyncio.Future`/lock keyed by hash; Go `singleflight`). *Mistake:* caching only completed results, leaving the cold-start thundering herd unhandled — you want the cache **and** single-flight.
- **D3 — Memoize at the model boundary.** Cache the model's output (embedding/classification/feature map) keyed by `hash(input) + model_version` — D1 at the most-expensive-to-recompute point. *Mistake:* omitting `model_version` from the key → after a model upgrade you keep serving the old verdicts from cache.
- **D4 — Decode once, fan out.** When several models/stages (NSFW + gore + violence) consume the same frame, decode+preprocess **once** and share the tensor. Decoding the same JPEG N times multiplies the bottleneck by N. If models need different sizes, decode once to high-res and derive each resize. *Mistake:* each model owning its own decode path "for encapsulation," silently re-decoding per frame.
- **D5 — Page-cache awareness.** The OS page cache already serves recently-read bytes from RAM for free. *Mistake:* `O_DIRECT` "for performance" without managing your own cache — turns cheap cached reads back into disk I/O.
- **D6 — Keep the session warm.** Load weights once at startup; keep the inference session resident for the process lifetime. Weight load + engine init is *seconds*. *Mistake (the most expensive duplicate-I/O bug):* `InferenceSession(model_path)` inside the request handler — reloading the model from disk every call.

## E. ONNX Runtime / inference-server specifics
- **E1 — IOBinding.** Bind inputs/outputs to device memory before `Run()` to eliminate ORT's implicit CPU↔GPU copies (by default ORT copies inputs from CPU and outputs back to CPU — "expensive, can be worse than vanilla PyTorch"). `bind_input` to an `OrtValue` on `'cuda'`; `bind_output` to a pre-allocated device tensor (or by `MemoryInfo` for dynamic shapes); call `run_with_iobinding`. Also **required** for CUDA Graphs. *Mistake:* benchmarking ORT-GPU with plain `run()` + numpy, losing to PyTorch, and blaming ORT — the gap is the implicit copies.
- **E2 — Reuse session, output buffers, threads.** One warm session (amortizes D6); rebind pre-allocated output buffers across calls (avoid per-call device allocation); set `intra_op_num_threads` deliberately (on a GPU-EP service the CPU does little — don't let intra threads fight data-loader workers); `ORT_SEQUENTIAL` is right for most models (`ORT_PARALLEL` only helps many-branch models); share a **global intra-op thread pool** across sessions. *Mistake:* every session spawning `num_cores` threads while `num_workers` loaders run → 2–3× oversubscription.
- **E3 — Execution providers + warmup.** Choose CUDA vs TensorRT deliberately; **run several warmup inferences at the real batch shapes before serving** — TensorRT's first run builds the engine (exhaustive kernel profiling = multi-second); without warmup your *first real request* eats it. Enable **engine/runtime cache** to disk to skip recompilation across restarts. *Mistakes:* skipping warmup (catastrophic p99 on first request + after each deploy); shipping a baked engine cache to different hardware/ORT/TensorRT versions — it's not portable and silently rebuilds/misbehaves.
- **E4 — Reuse pinned staging buffers.** Persistent pinned host buffers sized to `max_batch_size`, reused every request, rather than per-call allocation. Combined with IOBinding → a steady-state path with zero per-request allocation on the hot route. *Mistake:* a fresh pinned buffer per batch reintroduces the overhead.

## F. Measurement & guardrails
- **F1 — Profile before optimizing.** Each technique helps a *specific* bottleneck and harms others if misapplied (GPU decode when GPU-bound; more workers when disk-bound; bigger batches when latency-bound). Tools: `nvidia-smi dmon -s u` (cheap first look), **PyTorch Profiler + TensorBoard trace** (kernel gaps = data stalls), **ORT `enable_profiling`** (per-op timings, spot implicit copies). Read the triad: GPU util <70% (input-bound), memory util >90% (room for bigger batches), power <80% cap (not working hard).
- **F2 — The over-optimization trap.** Don't add GPU decode, a dedup cache, single-flight, or TensorRT until a measurement shows the bottleneck they target — each carries permanent maintenance cost (cache eviction/TTL/invalidation bugs; DALI dependency; fragile engine caches). **Cheapest-wins order for an ORT GPU service:** warm session reuse (D6/E2) → IOBinding (E1) → pinned + non-blocking copies (A3) → batching / dynamic batching (B) → content cache + single-flight *if duplicates measured* (D) → GPU decode / TensorRT *only if still bound* (C2/E3). *Mistake:* building the interesting dedup cache first when 90% of the win was one line — moving `InferenceSession()` out of the handler.

---

## SMELL → FIX
| Smell (observed signal) | Diagnosis | Fix |
|---|---|---|
| GPU util <40%, kernel gaps in trace | Data-loading bound | more `num_workers` + `prefetch_factor` + `pin_memory` + `non_blocking` (A2/A3/C1) |
| Still low after workers; CPU pegged 100% | CPU decode ceiling | GPU decode (DALI/nvJPEG) — *only* if GPU isn't already busy (C2) |
| Async H2D shows no overlap | default stream / pageable memory | non-default stream + pinned memory + async copy (A2/A3) |
| Same image decoded N times for N models | per-model decode paths | decode once upstream, fan out the tensor (D4) |
| Same content re-processed repeatedly | no memoization | content-addressed cache `hash(bytes)+params`; outputs `+model_version` (D1/D3) |
| Burst of identical requests → N GPU runs on cold cache | thundering herd | single-flight: coalesce concurrent misses to one load (D2) |
| Tiny-file storage, slow epoch, high open/stat | random small-file I/O | shard into tar/WebDataset or parquet; sequential + parallel (3–10×) (C3) |
| CPU↔GPU ping-pong, frequent syncs | tensors round-tripping | keep on device; ORT IOBinding (A4/E1) |
| ORT-GPU slower than PyTorch | implicit copies in `Run()` | IOBinding in/out; reuse output buffers (E1/E2) |
| First request after deploy is multi-second | cold kernel/engine build | warmup at real batch shapes; engine/runtime cache (E3) |
| Seconds/request, terrible throughput | reloading weights per request | warm resident session reused for all requests (D6/E2) |
| Batch=1, GPU util low even when busy | underfilled SMs | batch + server-side dynamic batching (B1/B2) |
| p99 spiked after enabling batching | queue delay too high | lower `max_queue_delay`, tune to arrival rate (B2) |
| Wasted compute on padding filler | over-padding | bucket by size; pre-resize to fixed input (B3) |
| VRAM full, throughput plateaued | past occupancy knee | stop growing batch; drop precision to fit useful work (B4/C4) |
| PCIe transfer dominates, large VRAM | fp32 everywhere | fp16/bf16/int8 + channels-last (½ bytes, ~2× compute) (C4) |
| Layout-conversion ops dominate CV trace | channel thrashing | channels-last end-to-end; avoid NHWC-unsupported ops (C4) |
| Re-reading hot files hits disk | page cache defeated | drop needless `O_DIRECT` (D5) |
| "I optimized X and it got worse" | wrong bottleneck | profile first; fix the measured dominant stall only (F1/F2) |

*Sources: NVIDIA (CUDA overlap blog, CUDA C++ Best Practices, DALI/nvJPEG, TensorRT cache); Triton dynamic-batching docs; ONNX Runtime (IOBinding, threading, CUDA/TensorRT EP); PyTorch (DataLoader, channels-last, Lightning speed guide); Hugging Face Optimum GPU guide & WebDataset; vLLM EmbeddingCache RFC (content-addressed + single-flight + fan-out); apxml/Spheron profiling guides.*
