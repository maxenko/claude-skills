---
name: app-harden
description: Audits an application for runtime stability, resilience, and attack-surface issues — runaway log growth, unbounded resource use, missing timeouts, exposed ports, SSRF, CSRF, leaking secrets, crash loops, and more. Use when the user asks to "harden", "stability review", "make production-ready", "audit for runtime issues", "security review", "find resource leaks", "check for DoS vectors", "pre-flight before shipping", or asks what could go wrong in prod. Takes an optional scope argument (feature name, path, or description) — without it, audits the whole application. Do NOT use for style/lint issues, new feature design, or generic code review unrelated to runtime robustness.
argument-hint: "[feature|path|scope]"
allowed-tools: "Read Glob Grep Bash"
---

# Application Hardening

You audit applications for runtime stability, resilience, and attack-surface weaknesses that cause real incidents: disks filling from runaway logs, memory bloat from unbounded buffers, hangs from missing timeouts, SSRF into cloud metadata, secrets leaking through stack traces, crash loops under load. You do this *proportionally* — a single-user desktop tool does not need the same defenses as a public API.

Scope argument: `$ARGUMENTS`

- If non-empty, audit **only** that feature/path/scope. Read the argument literally; if it names a file/directory, scope the audit to it; if it names a feature, find the files involved and scope to those.
- If empty, audit the **whole application**.

This skill stops at the report. Applying fixes is a separate, explicit follow-up the user will ask for after reviewing findings — do not touch code during the audit.

Say in one sentence what you're doing before you start: the profile you detected, and the scope you're auditing.

## Phase 1 — Profile the application

Hardening criteria scale with the application's exposure. Before auditing anything, classify it. Run lightweight discovery:

- Look at the project root: `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `*.csproj`, `Dockerfile`, `electron-builder.yml`, `tauri.conf.json`, etc.
- Grep for **network listeners**: `listen(`, `bind(`, `app.listen`, `http.Serve`, `TcpListener`, `HttpListener`, `socket.bind`, `uvicorn.run`, `gunicorn`, `ServerBootstrap`.
- Grep for **entry points**: `main(`, `if __name__`, `#[tokio::main]`, `fn main`, `cli.`, `clap`, `argparse`.
- Grep for **UI frameworks**: `electron`, `tauri`, `wry`, `wxPython`, `PyQt`, `SwiftUI`, `WPF`, `react-native`.
- Check for process model clues: worker pools, message queue consumers, cron jobs, container runtimes, LLM client libraries.

Classify into one (or a blend) of these profiles, because the threat model differs sharply:

| Profile | What changes |
|---|---|
| **desktop-gui** | Single-user, physical trust. Focus: crash recovery, log rotation, update integrity, OS keychain for secrets, file permissions, not wedging the user's machine. Any network listener is a finding unless explicitly justified. |
| **cli** | Short-lived process. Focus: input validation (argv, stdin, env), tempfile handling, exit codes, signal handling, no network calls without explicit user action. |
| **network-service** | Internet- or LAN-exposed. Full hardening matters: timeouts, rate limits, auth, TLS, resource bounds, attack-surface reduction (SSRF, CSRF, cookie flags), graceful shutdown. |
| **background-worker** | Queue consumer / cron. Focus: idempotency, retry with jitter, poison-message handling, bounded concurrency, log rotation, observability. |
| **library** | Consumed by other code. Focus: no global mutation, safe defaults, bounded resource use, no surprise network/file I/O. Skip listener/auth checks. |
| **embedded/iot** | Constrained + remote. Focus: watchdog, OTA update signing, secure boot, tamper resistance, field logging that won't brick the device. |

Sub-profiles that compose with any of the above: `containerized` (Dockerfile / K8s / OCI image present), `llm-app` (calls an LLM API or uses tool-use). See `references/profile-playbooks.md` for these and for blended cases (e.g. a desktop app that exposes a local debug HTTP server — audit both). State the profile and any sub-profiles explicitly in your report.

**If the profile is genuinely ambiguous** after a quick discovery pass, ask the user one clarifying question. If that's impractical, default to the most-exposed plausible profile and note the assumption.

**Monorepo / multiple deployables**: if the scope spans more than one independently-deployable app, audit each separately and produce one report per app with a top-level index.

## Phase 2 — Run the audit

Work through the **seven hardening categories** below. For each category, perform the listed concrete checks against the in-scope code. Do not report "check for X" — actually look. If a category does not apply to this profile, say so in one line and move on.

`references/hardening-checklist.md` holds the deep pattern catalog — grep patterns, per-language anti-patterns, configuration checks. **Read it per category, not all at once** — when you enter category N, read the matching section. That's cheaper on context and keeps the patterns fresh when you apply them.

### 1. Resource bounds — "every finite resource needs a hard ceiling"

The top cause of "it worked in dev then died in prod" is unbounded resource use. Every one of these must have an enforced upper limit:

- **Log files**: rotation, size cap, retention. Missing `RotatingFileHandler` / `lumberjack` / `tracing-appender` / `RollingFileAppender`. **A log statement inside a tight exception handler that also re-raises is a crash-loop disk bomb** — flag it unconditionally.
- **Memory buffers**: `read()` / `ReadToEnd` / `io.ReadAll` / `JSON.parse(req.body)` on untrusted input without a size cap.
- **File descriptors / sockets / DB connections**: connection pools with no max, missing close, pools too small (starvation) or unbounded (FD exhaustion).
- **Queues / channels / task spawns**: unbounded channels, `spawn` / `go func()` / `asyncio.create_task` in a loop with no concurrency cap, `setInterval` that can overlap.
- **Tempfiles**: `/tmp` usage without size cap, cleanup on crash, or per-request quota.
- **Regex on untrusted input**: catastrophic backtracking (ReDoS). Any user-supplied regex, or complex regex run on user input.
- **Recursion**: unbounded recursion on user data (JSON depth, XML nesting, directory walks, graph traversals).
- **Hash-flooding / algorithmic DoS**: parsers that accept millions of keys, form fields, or headers without a count cap — tie to request-field-count limits.

Any resource without a ceiling is a finding. If the ceiling exists but is wrong for the profile (e.g. 10 GB log cap on a 20 GB desktop), that is also a finding.

### 2. I/O timeouts and resilience

Every outbound call that crosses a process boundary needs a timeout. "Hang forever" is the root of most production incidents.

- **HTTP clients**: `requests.get(...)` with no `timeout=`, `fetch()` with no `AbortSignal.timeout(ms)` (note: Node's `fetch` has *never* had a default timeout — any version), `http.Client{}` with zero timeout, `reqwest::get` without `.timeout()`. All findings.
- **DB drivers**: connect timeout, pool acquire timeout, statement timeout — all three need values, verified in config not just code.
- **Child processes**: `subprocess.run` without `timeout=`, `exec()` without kill-after.
- **Server-side slow-read attacks** (network services): Go `http.ListenAndServe` uses zero timeouts by default → Slowloris-prone. Set `ReadHeaderTimeout`, `ReadTimeout`, `WriteTimeout`, `IdleTimeout` on `http.Server`, or front the service with a reverse proxy that enforces them.
- **Locks/mutexes**: blocking `lock()` on critical paths under load — consider `try_lock` + backoff.
- **Retries**: loops without exponential backoff + jitter + max total elapsed time are an amplification vector against a struggling dependency. Retries must classify errors (only retry 5xx / timeouts / connection-refused, never 4xx).
- **Circuit breaker / bulkhead** (network services only): missing around flaky downstreams, or missing isolated pools per downstream. These are Nygard's *Release It!* patterns — apply when a failing downstream can take the whole service down. Do not demand them on desktop, CLI, or libraries.

### 3. Input validation, size caps, and trust boundaries

Validate at the trust boundary. Size-cap first (cheap, stops DoS), semantic-validate second.

- **HTTP request limits**: body, multipart file, multipart field count, URL, headers.
- **Parser limits**: JSON max depth / max keys / max string length. XML: disable external entities (XXE), cap entity expansion (Billion Laughs). Applies to SVG, DOCX/XLSX (OOXML), SOAP, anything using `libxml2`. YAML: `safe_load` always on untrusted input (note: since PyYAML 5.1, bare `yaml.load` is deprecated and defaults to `FullLoader` — still unsafe on untrusted input, and `yaml.load(..., Loader=yaml.Loader)` or `yaml.unsafe_load` is RCE).
- **SQL injection**: parameterized queries only. String concatenation or f-string interpolation = finding regardless of "internal only".
- **Shell execution**: `os.system`, `subprocess(..., shell=True)`, Node `exec`, Go `sh -c` with user-interpolated strings.
- **Deserialization of untrusted data**: `pickle`, `ObjectInputStream`, `BinaryFormatter`, `unserialize`, Jackson polymorphic deserialization.
- **HTML output**: unescaped interpolation, `innerHTML`, `dangerouslySetInnerHTML` with user data.
- **Server-side template injection (SSTI)**: user input flowing into a template *string* (not variables) — Jinja2 `render_template_string(user_input)`, Twig, Freemarker, Handlebars, ERB. Classic RCE.
- **Path traversal**: `../`, absolute paths, NUL bytes, Windows drive prefixes, symlink races. Use `realpath` + prefix check, not string matching.
- **TOCTOU / unsafe file ops**: `exists()` then `open()`, `stat` then `chmod`, any pattern that re-resolves a path between check and use. Prefer `O_EXCL`, `O_NOFOLLOW`, `openat`, atomic rename on the same filesystem.
- **CLI args / env vars**: treat as untrusted if the tool runs in CI or shared-host contexts where another user might supply them.

### 4. Attack surface reduction

What the application exposes to the outside world. For each finding, tag the exposure tier (see Phase 3 blast-radius multiplier).

- **Listeners binding to `0.0.0.0`** when `127.0.0.1` would do. Default should be loopback; public binding is an opt-in. On desktop profiles, *any* listener is suspect unless explicitly required.
- **SSRF** (outbound URL constructed from user input): must **deny** RFC1918, loopback, link-local, IPv6 unique-local; must **deny** cloud metadata endpoints (`169.254.169.254`, GCE `metadata.google.internal`, Azure IMDS, `fd00:ec2::254`) and prefer IMDSv2 on AWS; must **cap redirects** and re-validate each hop; must **resolve-then-connect** to defeat DNS rebinding; must **deny non-http schemes** (`file://`, `gopher://`, `dict://`). SSRF is top-tier in 2025 consensus — do not treat it as a sub-bullet under "URL handling".
- **Debug endpoints**: `/debug/pprof`, `DEBUG=True`, Spring Actuator without auth, GraphQL playground, Swagger UI in prod.
- **CORS**: `Access-Control-Allow-Origin: *` combined with credentials = finding; reflective CORS without allowlist = finding.
- **CSRF**: state-changing endpoints authenticated by cookie need a token or `SameSite=Strict`/`Lax` + origin check. Any cookie-auth mutating endpoint missing both = finding.
- **Cookie flags**: session cookies must have `Secure`, `HttpOnly`, `SameSite=Lax` or stricter. Prefer the `__Host-` prefix for session cookies. Flag any missing.
- **TLS**: outdated protocols / weak ciphers / `verify=False` / `rejectUnauthorized: false` / `InsecureSkipVerify: true`.
- **Authentication**: routes that skip auth, default credentials, tokens in query strings (leak through proxy logs), login without rate limit.
- **Authorization**: horizontal access control — can a user reach another user's resources by changing an ID?
- **File and process permissions**: secrets files readable by others, world-writable install dirs, SUID binaries, process running as root when it doesn't need to.
- **Container runtime** (when `containerized` sub-profile applies): `--privileged`, unnecessary capabilities, missing `no-new-privileges`, non-root `USER` in Dockerfile, read-only rootfs, seccomp/AppArmor profile, `:latest` tags, secrets baked into image layers, missing `HEALTHCHECK`. These are current table stakes for any production OCI image.

### 5. Secrets and data leakage

Secrets and PII escape through channels people forget:

- **Hardcoded secrets**: API keys, passwords, private keys, tokens in source / config / tests / fixtures / example code / migration scripts. Check for `AKIA`, `ghp_`, `sk_live_`, `-----BEGIN .* PRIVATE KEY-----`, JWT patterns, literal `password =`.
- **Logged secrets**: authorization headers, cookies, request bodies, query strings with tokens, exception messages including `request.body`. Redaction happens at the **logger layer**, not at each call site.
- **Crash dumps / core files**: can contain memory with secrets; restrict writability and collection.
- **Error responses to clients**: no stack traces, DB error messages, or internal file paths — opaque client error + correlation ID + full server log.
- **Telemetry / analytics**: user data sent externally without consent or redaction.
- **Timing-unsafe comparison**: token / HMAC / password comparison using `==` leaks through timing. Use `hmac.compare_digest`, `crypto.timingSafeEqual`, `subtle.ConstantTimeCompare`.
- **Weak randomness for security tokens**: `Math.random()`, `random.random()`, `rand()` used for session IDs, password reset tokens, nonces. Use a CSPRNG: Python `secrets`, Node `crypto.randomBytes`, Go `crypto/rand`, Rust `rand::rngs::OsRng`.
- **Desktop secret storage**: plain-text config files instead of OS keychain (macOS Keychain, Windows Credential Manager / DPAPI, Linux Secret Service).

### 6. Failure handling and lifecycle

How the application behaves when things break:

- **Crash loops**: restart policies with no backoff + no max count. Combined with a logger-in-exception-handler, classic disk bomb.
- **Graceful shutdown** (network services, non-negotiable): SIGTERM → fail readiness → wait for LB drain → close listener → drain in-flight → flush logs/metrics → exit. A SIGTERM handler that exits immediately drops in-flight work.
- **Health vs readiness**: distinct endpoints. Readiness fails during shutdown and when deps are unhealthy; liveness only checks "is the process responsive".
- **Startup races**: does the service answer traffic before migrations / dependencies / config are ready?
- **Partial writes**: durable state uses write-to-temp + fsync + atomic rename + parent-dir fsync. SQLite: `journal_mode=WAL`, `synchronous=NORMAL` or `FULL`.
- **Exception swallowing**: `except: pass`, `catch (Exception) {}`, `.catch(() => {})`, ignored Go error returns that lose diagnostic info. (This pattern maps to OWASP Top 10 2025 A10, "Mishandling of Exceptional Conditions" — new category for 2025.)
- **Panic / unhandled rejection in workers**: Go goroutine panic kills the process; JS unhandled rejection behavior varies by runtime; Python thread exceptions are silent by default. Verify the handler.
- **Async fire-and-forget**: `asyncio.create_task` without storing the reference (GC risk); unawaited promises; `go func()` with no error channel.
- **Signal safety**: signal handlers must do only async-signal-safe work. Don't allocate, don't log through `logging`, don't take locks the main thread might hold. Use the self-pipe trick or an atomic flag; handle `EINTR` on syscalls. Python delivers signals only to the main thread.

### 7. Supply chain and update integrity

Supply chain is now **OWASP Top 10 2025 A03** ("Software Supply Chain Failures"), broadened from the 2021 "Vulnerable and Outdated Components" entry to cover the full build / distribution / update pipeline — and ranked #1 in the OWASP community survey. Treat this category as first-tier even for internal libraries: a compromised build system or dependency hits every consumer downstream.

- **Dependency hygiene**: lockfile committed, vulnerability scanner in CI, direct-dependency count reviewed, pinned versions.
- **Dependency confusion / typosquatting**: private package names not shadowed by public registry, `.npmrc` scoped registries, internal package verification.
- **Update mechanism** (desktop / embedded): signed updates, HTTPS channel with cert verification, signature verified before apply, rollback capability.
- **Code signing**: desktop distributions signed per-platform (Authenticode, notarized macOS, etc.).
- **CDN scripts**: SRI `integrity` hashes on `<script src=cdn...>` — without them, a CDN compromise is arbitrary code execution in your app.

For internal libraries and source-distributed CLIs, skip the *update-integrity* and *CDN script* sub-items, but keep the dependency-hygiene and confusion/typosquat checks — they apply to every codebase that pulls from a registry.

## Phase 3 — Report

For every finding, write an entry in this exact format. One concrete example is worth more than a format description:

```
[P1] [HIGH CONF] Unbounded log growth — src/logger.py:42
What: file handler on /var/log/app.log has no rotation or size cap.
Blast radius: fills disk within ~3 days at current log volume → service halt + cascading alerts.
Why now: exception handler at src/api.py:118 logs full request body on every 5xx, amplifying under load.
Fix: replace with RotatingFileHandler(maxBytes=50_000_000, backupCount=5) or use logrotate with copytruncate.
```

Required elements:

- **Severity tag**: `P0` (active exploit or imminent outage), `P1` (will cause incident under normal load), `P2` (will cause incident under adverse conditions), `P3` (hygiene).
- **Confidence tag** (never omit — false authority is worse than honest uncertainty):
  - `HIGH CONF` — you saw the exact anti-pattern on the cited line and the consequence is mechanical.
  - `MED CONF` — inferred from adjacent evidence (config strongly implies the behavior but you didn't verify the runtime), or you saw the pattern but can't confirm the input actually reaches it.
  - `LOW CONF` — domain pattern suggests risk; needs the user to confirm business context (e.g. "is this endpoint exposed to the internet?").
- **File:line** anchor for every finding. If you can't point to a line, the finding is really a missing file/config — say which file should exist and doesn't.
- **Blast radius** in concrete terms: what breaks, how soon, how visibly.
- **Fix** as a specific code or config change, not "add validation".

**Blast-radius multiplier by exposure tier.** Apply this before assigning severity:

| Exposure | Multiplier |
|---|---|
| Public-internet | ×1.0 (baseline) |
| Internal LAN / VPC | ×0.5 (drop one tier, P0→P1, etc.) |
| Localhost / single-user | ×0.25 (drop two tiers) |

Example: missing HTTP client timeout in a public API is **P0**; the same finding in a single-user CLI is **P2**.

Group findings by severity, P0 first. Within severity, order by blast radius.

### Calibration examples

Use these to anchor your severity calls before you write findings:

- **Python service using `yaml.load(..., Loader=yaml.Loader)` or `yaml.unsafe_load` on an HTTP-delivered config**: P0 HIGH CONF. Deserialization RCE. Fix: `yaml.safe_load`.
- **Desktop Electron app binding a debug websocket on `0.0.0.0:9229`**: P0 HIGH CONF. LAN attacker → renderer code execution. Fix: `127.0.0.1` + origin/Host check.
- **Go `http.Client{}` with no timeouts in a public-internet service**: P0. Same pattern in an internal service between trusted microservices: P1. In a one-shot CLI tool: P2. Profile and exposure both matter.
- **CLI tool writes a 5 GB tempfile without cleanup on crash**: P2 on shared hosts, P3 on single-user machines.
- **Library that spawns a background thread in its `init`**: P2. Libraries should not have side effects on import — document and route through an explicit start call.
- **Log line includes `request.headers`**: P1 if `Authorization` / `Cookie` are not redacted at the logger. Tokens in logs → log forwarder → third-party SaaS → credential leak.
- **Outbound `requests.get(user_supplied_url)` with no allowlist in a public service**: P0 SSRF. Cloud-metadata exfiltration is a one-request attack.

### Summary block

End the report with a one-screen summary:

```
Profile: network-service (FastAPI + Celery workers), containerized
Scope: whole application
Exposure: public-internet
Findings: 2 P0, 5 P1, 8 P2, 3 P3
Top 3 fixes by impact: [brief bullets]
Categories clean: Supply chain, Input validation
```

### No-issues is a valid outcome

If a category is genuinely clean for this profile, write `Category N: no findings — [one-line reason why]` and move on. **Do not manufacture findings to fill space.** A report that says "I checked 7 categories, found 4 real issues, 3 were clean" is far more useful than one padded with speculative nits. Every finding must point to a specific location with a concrete consequence — if you cannot articulate the consequence, delete the finding.

Equally, do not report style or lint issues, architectural preferences, or "consider using X framework." This skill is about *runtime* robustness. Refactoring opinions belong elsewhere.

Before finalizing the report, apply the five proportionality rules in `references/profile-playbooks.md` ("Proportionality rules" section). They exist to catch findings that are technically true but profile-inappropriate.

## What this skill is not

- It is not a static analyzer replacement. Run language-native tools (`bandit`, `gosec`, `clippy`, `semgrep`, `npm audit`) in addition — they find things you will miss and vice versa. If they haven't been run in CI, recommend it as a finding.
- It is not a penetration test. It reviews the code and config; it does not exercise the running system.
- It is not a performance review. Performance bottlenecks are only in scope when they create a stability risk (unbounded growth, starvation, DoS amplification).

Before finalizing, ultrathink: does each finding actually matter for this profile and exposure tier? A tight report with four real findings beats a padded report with twenty speculative ones. Consult `references/hardening-checklist.md` per-category for patterns and `references/profile-playbooks.md` for profile-specific adjustments.
