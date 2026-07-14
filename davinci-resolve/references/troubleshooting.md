# Troubleshooting Playbook

Symptom → likely cause → fix. Ordered by how often each is the real culprit. Always confirm the cause before stacking fixes.

## Choppy / laggy / stuttering playback

**Disambiguation test first:** drop the raw clip on a fresh timeline with **no grade and no effects**.
- Still choppy → it's a **decode / source** problem (codec or disk).
- Smooth until you add nodes/OFX/NR → it's a **processing** problem (grade weight).
- Smooth alone but choppy in the real timeline → **timeline complexity** (track count, composites, fusion clips).

### Decode/source problem
- **Long-GOP H.264/H.265** (phone, drone, mirrorless, GoPro, screen capture) is the most common cause. Each frame depends on neighbors, so seeking/playback is CPU-heavy regardless of GPU. Fix = **Optimized Media** or **Proxies** to DNxHR SQ (Win/cross-platform) or ProRes 422 (Mac).
  - Project Settings ▸ Master Settings ▸ Optimized Media and Render Cache → set Optimized Media Format. Then Playback ▸ "Use Optimized Media if Available", right-click clips ▸ Generate Optimized Media.
- **10-bit 4:2:2 HEVC** — Studio only on most platforms; even then, transcode for smooth editing.
- **Disk too slow** — 4K/RAW off a slow external/USB or NAS starves playback. Move media (or at least cache) to a fast local SSD/NVMe.
- **Decode acceleration**: ensure hardware decode is enabled (Preferences ▸ Decode Options) on Studio.

### Processing problem
- Use **Render Cache: Smart** (Playback ▸ Render Cache ▸ Smart). Cache codec → DNxHR/ProRes in the Optimized Media settings.
- Force-cache heavy clips: right-click ▸ Render Cache Color Output. Watch the red cache bar turn blue.
- **Temporal/Spatial Noise Reduction** and **Magic Mask / Depth Map** are extremely heavy (and Studio-only) — cache them or apply late.
- Set "Enable Background Caching after" to ~1s so it caches during idle.

### "Choppy *after* I cached it" loop
- Corrupt or thrashing cache. Playback ▸ Delete Render Cache ▸ All, confirm the cache drive has free space and is fast, regenerate.
- Cache location: Project Settings ▸ Working Folders → point to a fast drive with room.

### Quick playback wins (no transcode)
- Playback ▸ **Timeline Proxy Resolution ▸ Half / Quarter** (playback-only, doesn't touch render).
- Lower **timeline resolution** to 1080 while editing (Project Settings ▸ Timeline Resolution).
- Turn off scopes, turn off the second viewer, collapse unused panels.

## Crashes, freezes, GPU errors

- **Update GPU drivers** — on NVIDIA install the **Studio driver**, not Game Ready. Mismatched/old drivers are the #1 crash cause.
- **GPU Processing Mode** (Preferences ▸ Memory and GPU): CUDA for NVIDIA, Metal on Apple, OpenCL only as fallback. Set GPU selection to the right card(s); only Studio uses multiple GPUs.
- **GPU/VRAM exhaustion** — lower the GPU memory limit slider, reduce timeline res, cache heavy nodes. 4K+ with heavy NR can exceed small-VRAM cards.
- **Crash on a specific clip** — likely a bad/unsupported codec or corrupt file; transcode it externally (HandBrake/FFmpeg) and relink.
- **Crash on launch / opening a project** — corrupt preferences (rename/clear the config folder to regenerate), a corrupt project (restore from Project Backups), or a bad third-party OFX plugin/font. Disable plugins to isolate.
- **Crash in Fusion** — often a GPU/driver issue or an effect node; try CPU fallback for that node, update drivers.
- Capture logs: macOS `~/Library/Application Support/Blackmagic Design/DaVinci Resolve/logs`; Windows `%AppData%\Blackmagic Design\DaVinci Resolve\Support\logs`. Read with the Read tool if the user shares one.

## Audio problems

- **Progressive sync drift** → VFR source (see SKILL.md). Transcode to CFR before import; relinking won't fix it.
- **Pops/clicks/crackle on playback** → buffer/hardware: Preferences ▸ Video and Audio I/O / Fairlight ▸ raise buffer size; check sample-rate mismatch (project vs clip, e.g. 48k vs 44.1k).
- **No audio out** → wrong audio output device or monitoring routing in Fairlight; check the timeline's audio output bus mapping.
- **Audio out of sync at a fixed offset** → sample-rate or pulldown mismatch, or the camera recorded a different rate; conform or re-sync.

## Project & database recovery

- **Project won't open / missing from list after moving files** → the database moved. Don't move database folders with Resolve open. Reconnect: Project Manager ▸ Databases ▸ Connect (Disk path or PostgreSQL server).
- **Recover lost work** → Project Backups (Preferences ▸ Project Save and Load): restore a `.diskdb`/backup. If Live Save was on, the latest auto-save is usually intact.
- **Move a project between machines** → export `.drp` (carries the project; not the media). Use the same relative folder structure or relink on the new machine.

## Render / export failures

- **Render fails or stops partway** → a single bad frame/clip (transcode that clip), out of disk space, or an unsupported output codec for your edition. Try In/Out around the failure point to isolate.
- **ProRes export greyed out on Windows** → requires **Studio**. Use DNxHR (free) or H.264/H.265 instead.
- **Slow renders** → enable hardware-accelerated encoding (Studio); render to a fast drive; for H.264/H.265 the NVENC/QuickSync encoder is far faster than software.
- **Wrong-looking export** → see the washed-out section in SKILL.md (viewer/levels/tagging, not the grade).
