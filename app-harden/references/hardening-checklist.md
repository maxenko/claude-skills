# Hardening Checklist — Concrete Patterns

Read this reference **per category** as you enter each section of Phase 2 — not all at once. It contains the specific grep patterns, anti-patterns, and code smells to look for. Each category below mirrors the numbering in SKILL.md.

## Contents

- Category 1 — Resource bounds (log growth, memory buffers, FDs, queues, tempfiles, regex, recursion, hash-flooding)
- Category 2 — I/O timeouts and resilience (HTTP/DB timeouts, retries, circuit breakers, Slowloris defense)
- Category 3 — Input validation (HTTP limits, parser limits, path traversal, TOCTOU, SQLi, shell, SSTI)
- Category 4 — Attack surface (SSRF, bind addresses, debug endpoints, CORS, CSRF, cookies, containers, TLS, auth)
- Category 5 — Secrets and data leakage (hardcoded secrets, log redaction, error responses, timing-safe compare, CSPRNG, keystore)
- Category 6 — Failure handling and lifecycle (graceful shutdown, exception swallowing, crash loops, signals, durable writes)
- Category 7 — Supply chain (dependency hygiene, confusion/typosquat, update integrity, CDN SRI)
- Language / framework quick hits (Python, Node, Go, Rust, Java, .NET)

**Anchor to OWASP ASVS 5.0 (released May 2025, see https://asvs.dev) and OWASP Top 10 2025** as you work. The ASVS 5.0 chapter numbering was restructured from 4.0, so verify the current chapter mapping at asvs.dev rather than relying on remembered V-numbers — the previous edition's V-numbers no longer apply directly. Use the ASVS chapters when the user asks for a compliance-flavored report.

## Category 1 — Resource bounds

### Log growth (all languages)

Grep patterns that indicate missing rotation:
- Python: `FileHandler\(` without matching `RotatingFileHandler` / `TimedRotatingFileHandler` in the same module
- Go: `log.New(` with a `*os.File`, `lumberjack` absence when writing to disk
- Rust: `tracing_appender::rolling` — if absent on disk writes, flag
- Node: `fs.createWriteStream` on a log path without `pino.transport` / `winston.transports.DailyRotateFile`
- Java: log4j/logback config missing `RollingFileAppender` or size triggering policy
- .NET: Serilog sink without `rollingInterval` / `fileSizeLimitBytes`

Crash-loop amplifier pattern — look for this anywhere:
```
try:
    ...
except Exception as e:
    logger.error(f"failed: {e}", extra={"body": request.body})
    raise  # or swallowed, either is bad combined with no rotation
```
If the handler logs + re-raises + the process restarts on exception, each retry writes more logs. Flag as **P1** even if rotation exists, because rotation only caps disk — it doesn't cap *rate*.

Log-level hygiene:
- `DEBUG` level in production config = P2 (log volume explosion)
- Logging full HTTP bodies / SQL params / exception `__dict__` = P1 (secrets + volume)

### Memory buffers

Patterns that read entire untrusted input into memory:
- Python: `f.read()`, `request.data`, `json.loads(request.body)` without size check
- Go: `io.ReadAll`, `ioutil.ReadAll`, `json.Unmarshal` without a `http.MaxBytesReader`
- Node: `req.on('data')` accumulation without length check, `JSON.parse(req.body)` with no size cap
- Java: `readAllBytes()`, `Scanner` over stream, reading multipart without `maxFileSize`
- Rust: `.collect::<Vec<_>>()` on a stream, `read_to_end`, `serde_json::from_reader` without a length-limited reader
- .NET: `StreamReader.ReadToEnd`, `request.Form` without `MaxRequestBodySize`

Fan-out patterns:
- `Promise.all(items.map(...))` / `await asyncio.gather(*[...])` / `errgroup` / `tokio::join!` on user-supplied `items` — unbounded concurrency. Use a semaphore / bounded channel / worker pool.

### File descriptors and connections

DB pool configs to check (project config files, not just code):
- PostgreSQL: `max_connections` on server, pool `max` on client, `pool_timeout`, `pool_recycle`
- Redis: `maxclients`, client `maxRetriesPerRequest`, `connectTimeout`
- HTTP client pools: `http.Transport{MaxIdleConns, MaxConnsPerHost}` in Go; `reqwest::ClientBuilder::pool_max_idle_per_host`; `requests.adapters.HTTPAdapter(pool_connections=...)`

Missing-close patterns:
- Python: `open(` without `with`; `urllib.request.urlopen` without context manager
- Java: file/socket without try-with-resources
- Go: `defer resp.Body.Close()` missing after `http.Get`
- Node: streams without `.destroy()` on error paths

Thread pool starvation:
- Java ExecutorService with `newFixedThreadPool(n)` where n < concurrent load
- Python `ThreadPoolExecutor(max_workers=None)` — default is `min(32, os.cpu_count()+4)` since 3.8, which is bounded but almost never the right number for your workload. Set it explicitly.

### Queues, channels, task spawns

- Go: `make(chan T)` on a producer-consumer channel = unbounded. Must be buffered with a max.
- Rust: `tokio::spawn` in a loop without backpressure, `mpsc::unbounded_channel`
- Python: `asyncio.create_task` in a loop without a semaphore; `Celery` with no `worker_max_tasks_per_child`
- Node: `setImmediate` / `process.nextTick` recursion; event emitters with no `maxListeners` cap

### Tempfiles

- Python: `tempfile.NamedTemporaryFile(delete=False)` without explicit cleanup
- Any language: writing to `/tmp` or `os.TempDir()` without size cap or disk-full handling
- Cleanup on crash: if your tempfile handling relies on `atexit` or `defer`, it won't run on `kill -9`. For long-lived services, periodic sweeps of stale tempfiles are the robust answer.

### Regex

ReDoS suspect patterns — any regex with:
- Nested quantifiers: `(a+)+`, `(a*)*`
- Alternation with overlap: `(a|a)*`, `(a|ab)*`
- User-supplied regex pattern is almost always a finding; use a library with linear-time guarantees (RE2 / Go's regexp / Rust's regex crate) or impose a timeout.

### Recursion on user data

- JSON parser: set max depth (most parsers have an option; use it).
- Directory walks: set max depth and max entries.
- Graph traversals: use iterative + visited-set, not recursive.

### Hash-flooding / algorithmic DoS

Flag parsers that ingest user-controlled maps/sets without count caps:
- Very large key-count JSON objects, `multipart/form-data` with millions of small fields, headers with thousands of entries, URL query strings with thousands of parameters.
- Older Go `map` (pre-1.17) and any hashmap using non-randomized hashing are worst-case; even modern ones degrade under pathological input.
- Fix: cap field counts at the parser or reverse-proxy layer. Nginx `large_client_header_buffers`, Express `parameterLimit` in `body-parser`, Go `http.Server.MaxHeaderBytes`.

## Category 2 — I/O timeouts and resilience

### HTTP client timeouts by language

| Language/Library | Unsafe default | Required fix |
|---|---|---|
| Python `requests` | no timeout → forever | `requests.get(url, timeout=(connect, read))` |
| Python `httpx` | 5s default is OK but check | verify not overridden to `None` |
| Python `urllib.request` | no timeout | pass `timeout=` |
| Go `http.Client{}` | no timeout | set `Timeout` on client + `http.Transport` dial/response timeouts |
| Node `fetch` | never times out by default, any Node version | pass `signal: AbortSignal.timeout(ms)` per request |
| Node `axios` | no timeout | pass `{ timeout }` |
| Rust `reqwest` | no timeout | `.timeout(Duration)` on builder |
| Java `HttpClient` | infinite connect | `.connectTimeout(Duration)` + per-request `.timeout(Duration)` |
| .NET `HttpClient` | 100s default may be too long | set `Timeout` explicitly |

### DB timeouts

Every driver has three separate timeouts. All three must be set:
1. **Connect timeout** — how long to wait for initial TCP/handshake
2. **Pool acquire timeout** — how long to wait for a free connection from the pool
3. **Statement timeout** — how long a query may run

Grep for driver config and verify all three are present with sensible values.

### Retry discipline

Bad retry loop:
```python
for _ in range(5):
    try: return call()
    except: pass
```
Issues: no backoff (amplifies load on failing downstream), no jitter (thundering herd), no classification (retries on 400s and other non-retryable errors), no max total time.

Good retry:
- Exponential backoff with jitter
- Cap total elapsed time, not just attempt count
- Retry only on classified retryable errors (timeout, 5xx, connection refused)
- Respect `Retry-After` headers

### Circuit breaker / bulkhead (network services only)

Apply when: a failing downstream can take the whole service down.

- Circuit breaker: open after N failures in M seconds, half-open probe after cooldown.
- Bulkhead: separate thread/connection pool per downstream — one bad downstream cannot exhaust the pool used by healthy ones.

Libraries: `resilience4j` (Java), `gobreaker` (Go), `pybreaker` / `tenacity` (Python), `@nestjs/terminus` + `opossum` (Node), `failsafe-rs` (Rust).

Do not demand these on desktop or CLI tools. Do not demand them for internal deterministic dependencies like a local Redis where failure means "the whole thing is down anyway".

### Slow-client / Slowloris defense (server-side)

A missing client-read timeout lets a hostile client hold a connection open indefinitely with slow-drip bytes, exhausting the worker pool. Defense is per-language:

- **Go**: `http.ListenAndServe` uses zero timeouts by default — vulnerable. Use `http.Server{ReadHeaderTimeout, ReadTimeout, WriteTimeout, IdleTimeout}` explicitly. The `ReadHeaderTimeout` is the single most important one.
- **Node Express**: no built-in slow-client defense — front with nginx or use `http.Server`'s `headersTimeout` / `requestTimeout` (Node 18+).
- **Python (gunicorn/uvicorn)**: set `--timeout` (worker-level) and put nginx in front for byte-level read timeouts.
- **Nginx reverse proxy**: `client_body_timeout`, `client_header_timeout`, `send_timeout`, `keepalive_timeout`.
- **Apache**: `mod_reqtimeout` with `RequestReadTimeout`.

## Category 3 — Input validation

### HTTP request limits

- Body size: Nginx `client_max_body_size`, Express `express.json({ limit })`, FastAPI via middleware, Go `http.MaxBytesReader`, ASP.NET `MaxRequestBodySize`.
- Header size: usually capped by the server but verify.
- URL length: same.
- Multipart: separate limit for individual file and total.

### Parser limits

- JSON: Python `json` has no max depth — use a limited parser or pre-check. Go `json` has an implicit stack limit but set it explicitly. Rust `serde_json` has `StreamDeserializer`; set max depth.
- XML: *always* disable external entities (XXE), cap entity expansion (Billion Laughs / quadratic blowup), disable DTD processing. Applies to direct XML parsers, SVG (common blind spot — SVGs are XML), OOXML (DOCX/XLSX/PPTX), SOAP, any `libxml2` binding. Python `defusedxml` (not stdlib `xml`), Java disable `ACCESS_EXTERNAL_DTD` + `ACCESS_EXTERNAL_SCHEMA`, .NET set `XmlResolver = null` + `DtdProcessing = Prohibit`, `lxml` with `resolve_entities=False`.
- YAML: never `yaml.load(..., Loader=yaml.Loader)` / `yaml.unsafe_load` on untrusted input — RCE. Since PyYAML 5.1, bare `yaml.load(data)` is deprecated and uses `FullLoader` (safer but still unsafe on untrusted input — `FullLoader` was further restricted in 5.4 for CVE-2020-14343 but should not be relied on). Use `yaml.safe_load` unconditionally on untrusted input.
- Pickle / Marshal / BinaryFormatter: never on untrusted input. These are RCE by design.

### Path traversal

For any code that takes a filename from user input and accesses the filesystem:
1. Reject absolute paths, `..` segments, NUL bytes, Windows drive prefixes, `/`/`\` at the start.
2. Resolve with `realpath` / `filepath.Clean` / `Path.resolve`, then verify the resolved path is still within the allowed root.
3. For downloads, use a whitelist of known files, not user-supplied names.

### TOCTOU (time-of-check-time-of-use) and unsafe file ops

Any sequence that *checks* a path and then *uses* it is a race window:
- `os.path.exists(p)` → `open(p)` — attacker symlinks between the two calls
- `stat(p)` → `chmod(p)` — attacker swaps a symlink to the target file
- `unlink(p)` → `open(p, O_CREAT)` — attacker recreates it as a symlink
- `if not isdir(p): mkdir(p)` — classic race

Fix patterns:
- `O_EXCL | O_CREAT` for "must be new" semantics
- `O_NOFOLLOW` to refuse following symlinks
- `openat` / `fstatat` for directory-fd-relative operations
- Open the file once and operate on the file descriptor — never re-resolve the path
- Atomic rename only works *within the same filesystem*; across filesystems it's copy-then-delete and not atomic

### SQL injection

String concatenation or f-string interpolation into SQL = finding, no exceptions. "It's internal" is not a defense — internal tools get compromised credentials.

Allowed patterns:
- Python: `cursor.execute("... WHERE id = %s", (id,))` with `%s` as placeholder (not f-string)
- Go: `db.Query("... WHERE id = ?", id)`
- Java: `PreparedStatement.setXxx`
- Rust: `sqlx::query!` (compile-time checked)

### Shell execution

Anything that passes user input into a shell must use:
- `subprocess.run([...])` (list, not string, no `shell=True`)
- Go `exec.Command("prog", arg1, arg2)` (not `exec.Command("sh", "-c", ...)`)
- Rust `Command::new().arg()`

### Server-side template injection (SSTI)

The dangerous pattern is passing user input as the template *string*, not as a template *variable*. All major engines have an RCE path:

- Python Jinja2: `render_template_string(user_input)` or `Template(user_input).render()` — RCE via `{{ self.__class__.__mro__[1].__subclasses__() }}` chain
- Twig (PHP), Freemarker, Velocity, Handlebars, ERB, Razor — all have documented SSTI → RCE
- Fix: pass user input as a template variable (`render_template("page.html", user_data=x)`), never as the template body. If user-authored templates are a feature requirement, use a sandboxed engine (Jinja2 `SandboxedEnvironment`, LiquidJS) and expect to spend real time on it.

## Category 4 — Attack surface

### SSRF (server-side request forgery)

Any outbound network call whose URL, hostname, or port is constructed from user input. Modern SSRF is one of the highest-impact vulnerabilities on public services because of cloud metadata endpoints.

**Required defenses (all, not some):**

1. **Scheme allowlist**: only `http`/`https`. Deny `file://`, `gopher://`, `dict://`, `ldap://`, `jar://`, `ftp://`, `tftp://`.
2. **Resolve-then-connect**: resolve the hostname yourself, validate the resolved IP, then connect by IP with `Host:` header preserved. Defeats DNS rebinding where the first resolution points to a public IP and the second points to `169.254.169.254`.
3. **IP allowlist or denylist**: deny RFC1918 (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`), loopback (`127.0.0.0/8`, `::1`), link-local (`169.254.0.0/16`, `fe80::/10`), IPv6 unique-local (`fc00::/7`), multicast, and `0.0.0.0`. Allowlist is stronger when feasible.
4. **Cloud metadata deny**: explicitly block `169.254.169.254` (AWS/Azure/GCP/OpenStack legacy IMDS), `fd00:ec2::254` (AWS IPv6 IMDS), `metadata.google.internal`, `metadata.azure.com`. On AWS, also require IMDSv2 on the instance itself so even a successful SSRF needs a token.
5. **Redirect handling**: cap redirect depth (typically 3–5) and re-validate every hop — attackers use redirects to bypass single-request validation.
6. **Port allowlist**: most APIs should only need 80/443. Deny connections to other ports.
7. **Timeouts**: SSRF to a non-responsive internal host can exhaust workers.

Framework-level helpers: `ssrf-req-filter` (Node), `django-ssrfsafe`, `url-parse`-based validators. Better: use an egress proxy with an allowlist for all outbound traffic.

### Bind addresses

Grep for all of these and verify the address:
- `listen(`, `.bind(`, `.listen(`, `Serve(`
- Look for `0.0.0.0`, `::`, or empty host string — flag unless the service is explicitly public.
- Default should be `127.0.0.1` / `localhost`. Public binding should require an env var or config flag.

### Debug endpoints

- `/debug/pprof` (Go pprof) — disable or auth-gate in prod
- Flask `app.run(debug=True)` / `FLASK_DEBUG=1`
- Django `DEBUG = True`
- Spring Actuator endpoints without `management.endpoint.*.enabled=false` or auth
- GraphQL playground / introspection enabled in prod
- Swagger UI at `/docs` on a public endpoint

### CORS

- `Access-Control-Allow-Origin: *` combined with `Access-Control-Allow-Credentials: true` — broken, browsers reject this, but if configured it signals the developer didn't understand the model.
- Reflective CORS: echoing `Origin` without allowlist = finding.
- `*` with sensitive endpoints even without credentials = P2 information exposure.

### CSRF

Applies to any state-changing endpoint (POST/PUT/PATCH/DELETE) that authenticates via a cookie the browser sends automatically. Two acceptable defenses, and modern apps should use both:

1. **SameSite cookie attribute**: `SameSite=Lax` (default in modern browsers but verify in code), `SameSite=Strict` for high-risk cookies, `SameSite=None; Secure` only when genuinely needed for cross-origin use.
2. **CSRF token / double-submit cookie / origin-header check**: per-session or per-request token validated server-side. Django `CsrfViewMiddleware`, Rails `protect_from_forgery`, Express `csurf` (note: `csurf` is deprecated — use `@edge-runtime/cookies` + origin check or migrate to a maintained fork).

JWT-in-Authorization-header auth is **not** vulnerable to CSRF because the browser doesn't auto-attach it. JWT-in-cookie auth **is** and needs CSRF defense.

### Cookie flags

Session and auth cookies must carry:
- `Secure` — HTTPS only
- `HttpOnly` — not readable from JS (defeats most XSS → session theft)
- `SameSite=Lax` at minimum, `Strict` when the app doesn't need cross-site navigation to auth state
- `__Host-` prefix for session cookies — forces `Secure`, no `Domain`, `Path=/`
- Short `Max-Age` / session-only for sensitive cookies
- For third-party contexts, `Partitioned` (CHIPS) to opt into per-top-site storage

### Container runtime (containerized sub-profile)

For any app shipped as an OCI image (Dockerfile, container registry, K8s / ECS / Cloud Run):

- `USER` directive in Dockerfile — do not run as root inside the container
- `--privileged` flag at runtime → P0. This disables most container isolation.
- Dropped capabilities: `--cap-drop ALL` then `--cap-add` only what's needed
- `--security-opt no-new-privileges` → blocks setuid escalations
- Read-only root filesystem (`--read-only`) with explicit tmpfs mounts for writable dirs
- Seccomp profile in use (Docker default is fine; custom must not weaken it)
- AppArmor / SELinux profile loaded
- No `:latest` tags in production manifests — pin to digests (`image@sha256:...`)
- No `ADD http://...` in Dockerfile — use `curl` + explicit hash verification or a multi-stage build
- No secrets baked into image layers (check with `docker history`, `dive`, or `trivy`)
- `HEALTHCHECK` defined in Dockerfile or equivalent in orchestrator config
- Resource limits set (CPU, memory) in orchestrator config, not just in the app
- Image scanned in CI with `trivy` / `grype` / `snyk container`

### TLS

- `verify=False` (Python), `rejectUnauthorized: false` (Node), `InsecureSkipVerify: true` (Go), `ServerCertificateCustomValidationCallback` that returns true (.NET)
- Hardcoded pinning that can't be updated without a release
- TLS 1.0 / 1.1 / SSLv3 explicitly enabled

### Auth and rate limiting

- Endpoints that skip auth: grep for route definitions missing the auth middleware
- Default credentials: grep for `admin/admin`, `password: changeme`, etc. in configs/docs/tests
- Token in query string: `?token=` — leaks through proxy logs, browser history, referer
- Login without rate limiting: flag unconditionally
- Missing per-user rate limits on expensive endpoints

### File permissions

- Secrets files: must be `0600` / owner-only
- Directories containing secrets: `0700`
- World-writable anything in an install location = finding
- SUID binaries in the distribution = finding unless justified

## Category 5 — Secrets and data leakage

### Hardcoded secret patterns

```
AKIA[0-9A-Z]{16}              # AWS access key
ghp_[0-9a-zA-Z]{36}           # GitHub PAT
xox[baprs]-[0-9a-zA-Z-]{10,}  # Slack token
sk_live_[0-9a-zA-Z]{24,}      # Stripe live key
-----BEGIN (RSA|EC|OPENSSH|DSA) PRIVATE KEY-----
eyJ[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}   # JWT
```

Also look for:
- `password\s*[:=]\s*["'][^"']+["']` — hardcoded literal passwords
- `api[_-]?key\s*[:=]\s*["']` — API key assignments
- `.env` / `config.json` / `secrets.yml` committed to the repo

Check test fixtures, example configs, migration scripts, and documentation — secrets end up in all of these.

### Log redaction

Anything logged under these keys should be redacted at the logger layer:
- `authorization`, `cookie`, `x-api-key`, `x-auth-token`, `token`, `access_token`, `refresh_token`
- `password`, `pwd`, `secret`, `private_key`, `client_secret`
- PII: `email`, `ssn`, `credit_card`, `phone` — unless necessary, redact

Implementation: add a log filter / formatter that walks the log record and redacts by key. Don't rely on callers remembering.

### Error responses to clients

- Production error responses should never include: stack trace, file path, SQL, DB type, dependency versions.
- Distinguish client-facing error message (opaque) from server-side log (full detail with a correlation ID).

### Timing-unsafe comparison

Any comparison of a user-supplied token, HMAC, signature, password hash, or nonce against a secret must be **constant-time**. Plain `==` / `.equals()` / `strcmp` leak via timing and enable byte-at-a-time recovery.

- Python: `hmac.compare_digest(a, b)`
- Node: `crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b))` (both must be same length)
- Go: `subtle.ConstantTimeCompare(a, b)`
- Java: `MessageDigest.isEqual(a, b)` (constant-time since Java 6u17 / 7)
- Rust: `subtle::ConstantTimeEq` from the `subtle` crate
- .NET: `CryptographicOperations.FixedTimeEquals`

### Cryptographically secure randomness

Any value that's a secret — session ID, password reset token, CSRF token, nonce, salt, API key — must come from a CSPRNG, not a PRNG.

- Python: `secrets.token_bytes()` / `secrets.token_urlsafe()` — **not** `random.*`
- Node: `crypto.randomBytes()` / `crypto.randomUUID()` — **not** `Math.random()`
- Go: `crypto/rand` — **not** `math/rand`
- Rust: `rand::rngs::OsRng` or the `getrandom` crate — **not** `rand::thread_rng()` for secrets
- Java: `SecureRandom` — **not** `Random`
- .NET: `RandomNumberGenerator.GetBytes()` — **not** `System.Random`

Grep for `Math.random`, `random.random`, `rand()`, `math/rand`, `new Random()` near code dealing with tokens / sessions / auth — each hit is a finding.

### Desktop secret storage

Do not store secrets in:
- Plain-text config files in the user's home
- Environment variables set at GUI launch time (these leak to child processes and `/proc`)
- Browser `localStorage` (Electron apps)

Use instead:
- macOS: Keychain (`security` CLI, `node-keytar`, `keyring` Python)
- Windows: Credential Manager / DPAPI
- Linux: Secret Service API (`libsecret`), fallback to an encrypted file with a user-entered passphrase

## Category 6 — Failure handling and lifecycle

### Graceful shutdown

Network service checklist:
1. Receive SIGTERM
2. Fail readiness probe (LB stops sending new traffic)
3. Wait `grace_period` seconds (give LB time to react)
4. Close listener so no new connections accepted
5. Wait for in-flight requests to complete, up to `drain_timeout`
6. Flush logs, metrics, close DB connections
7. Exit 0

Common bugs:
- SIGTERM handler exits immediately — in-flight requests dropped
- No readiness flip — LB still sends traffic during shutdown
- Drain timeout shorter than longest legitimate request
- Flushing forgotten → last batch of logs/metrics lost

### Exception swallowing

Bad patterns:
- Python: `try: ... except: pass` (bare except catches KeyboardInterrupt, SystemExit)
- Python: `except Exception: pass` in a worker loop — silently drops tasks
- Go: ignoring error return (`_ = foo()`)
- JS: unhandled `.catch(() => {})` after an async operation
- Java: `catch (Exception e) { /* */ }`
- Rust: `.unwrap_or(default)` on an error that should propagate

These are findings when they lose information needed to diagnose or recover.

### Crash loops

- Systemd unit with `Restart=always` and no `RestartSec` + `StartLimitBurst`
- Docker `restart: always` without healthcheck-gated restart
- Kubernetes pod with no liveness/readiness distinction
- `pm2` / `forever` with no max restart count

Combined with a logger-in-exception-handler, this is the classic "filled the disk in 4 hours" incident.

### Async fire-and-forget

- Python: `asyncio.create_task(foo())` without storing the task — GC can collect it mid-execution. Store in a set and remove on done-callback.
- JS: unawaited promise in an async function — unhandled rejection depends on Node version
- Go: `go func() { ... }()` without any error handling or wg — panic kills the whole process

### Signal safety

Signal handlers run asynchronously and can interrupt the main flow between any two instructions. They may only call async-signal-safe functions. Common findings:

- Calling `malloc` / `free` / allocator-backed logging from a signal handler → can deadlock if the main thread held the allocator lock
- Python `logging.error()` in a signal handler → can deadlock on the logging lock
- Taking any lock the main thread might be holding
- Modifying non-atomic global state without `sig_atomic_t` / atomics
- Not handling `EINTR` on blocking syscalls (read/write/select) — these return early when a signal fires; retry the call

Fix patterns:
- Self-pipe trick: signal handler writes a byte to a pipe; main loop reads from the pipe and handles the event synchronously
- `signalfd` (Linux) or `sigwait` / `sigtimedwait` for a dedicated signal thread
- Python: signals are only delivered on the main thread; worker threads won't see them. Plan accordingly.

### Durable writes

Critical state that must survive power loss:
- Write to a tempfile, fsync, rename atomically
- Parent directory fsync on POSIX to persist the rename
- SQLite: `PRAGMA journal_mode=WAL`, `synchronous=NORMAL` or `FULL`
- Raw files: check the OS and application docs for atomicity guarantees

## Category 7 — Supply chain

### Dependency hygiene

- Lockfile committed: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `Pipfile.lock`, `poetry.lock`, `Cargo.lock` (for binaries), `go.sum`
- Vulnerability scan in CI: `npm audit`, `pip-audit`, `cargo audit`, `gosec`, `trivy`, Dependabot / Renovate
- Review direct dependency count and transitive bloat

### Dependency confusion and typosquatting

- Internal package names must not be shadowable by public registry packages. Register the namespace publicly as a placeholder.
- `.npmrc` with `@your-scope:registry=https://internal-registry` prevents npm from falling back to the public registry for a scoped package.
- `pip` with `--index-url` (not `--extra-index-url`, which creates ambiguity) for private PyPI.
- Verify package names at install time — typosquat packages (`reqeusts`, `python-dateutils`, `lodahs`) ship malware. Lockfile review should catch these.

### Update mechanism (desktop/embedded)

- Updates must be signed
- Update channel must be HTTPS with cert verification
- Updater must verify signature before applying
- Rollback must be possible
- For desktop: code-signed installer on all platforms that support it (Authenticode on Windows, notarized on macOS)

### CDN scripts

`<script src="https://cdn..." integrity="sha384-..." crossorigin="anonymous">` — without `integrity`, a CDN compromise injects arbitrary code into your app.

## Language / framework quick hits

### Python / Django / Flask / FastAPI
- `DEBUG = True` in production — P0
- `ALLOWED_HOSTS = ['*']` — P1
- `SECRET_KEY` hardcoded or default — P0
- `pickle` / `yaml.load` / `eval` / `exec` on untrusted — P0
- `subprocess(..., shell=True)` with user input — P0
- Flask `app.run()` in production (use gunicorn/uvicorn) — P1

### Node / Express
- `app.use(express.json())` without `{ limit }` — P2
- `helmet` not installed — P2
- CORS `origin: '*'` with credentials — P1
- `child_process.exec` with interpolation — P0
- `eval`, `Function()` on user input — P0
- **Prototype pollution**: user objects merged into trusted objects via `Object.assign({}, userObj)`, recursive merge utilities (lodash `_.merge`, `_.defaultsDeep`, `deepmerge` variants), `JSON.parse` reviver attacks, `req.query` deep-merged into config. Use `Object.create(null)` or `Map` for user-keyed dicts, or a vetted safe-merge. Flag any unsafe merge of user-controlled objects as P1.
- Unawaited promise in an async handler → unhandled rejection → process behavior varies

### Go
- `http.ListenAndServe` (zero timeouts — Slowloris-prone) vs `http.Server{ReadHeaderTimeout, ReadTimeout, WriteTimeout, IdleTimeout}` — P0 for public, P1 for internal
- `http.Client{}` with no `Timeout` — P0 for public service, P1 for internal, P2 for CLI
- `json.Unmarshal` without `http.MaxBytesReader` — P2
- Unbuffered channels in producer-consumer — P2
- `defer` in a loop on a long-lived function — P3 (resource leak)

### Rust
- `.unwrap()` in a request handler that can take user-triggered failure — P2
- `tokio::spawn` in a loop without semaphore — P2
- `serde_json::from_reader` without a length-limited reader — P2

### Java / Spring
- Actuator endpoints exposed without auth — P1
- Jackson polymorphic deserialization on untrusted input — P0
- `ObjectInputStream` on untrusted — P0
- H2 / JNDI injection vectors (log4shell-style) — P0 for vulnerable versions

### .NET
- `HttpClient` created per-request (socket exhaustion) — use `IHttpClientFactory` — P1
- `BinaryFormatter` — deprecated, P0 on untrusted
- `DEBUG` / `developmentExceptionPage` in production — P1
