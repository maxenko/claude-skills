# Bug Taxonomy — Language-Agnostic

Catalog of bug classes ordered roughly by frequency × impact, with the concrete patterns to look for. Use this when you want a broader enumeration than the six-lens pass gives you, or when hunting in an unfamiliar domain and want to know what to suspect.

Every entry has the same shape: **pattern → why it bites → concrete example**.

## Contents

1. Error-path bugs
2. Null / None / undefined dereference (CWE-476)
3. Off-by-one and boundary errors
4. Concurrency — atomicity, ordering, liveness
5. Integer & numeric bugs
6. Resource leaks
7. Injection & taint (OWASP A03, CWE-89/78/79/22/94/918/502)
8. Broken access control (OWASP A01, CWE-862/863/285)
9. Time, date, timezone, locale
10. State-machine holes
11. Retry and idempotency
12. Cache & staleness
13. Copy-paste and near-miss clones
14. API misuse, logic inversion, and flipped defaults
15. Regex bugs
16. Serialization version skew
17. Resource exhaustion / unbounded growth
18. Crypto & randomness
19. Memory & undefined behavior (C / C++ / unsafe Rust)
20. Type-system escapes
21. Configuration, environment, and supply chain

---

## 1. Error-path bugs

The richest seam. Yuan et al. (OSDI '14) found ~92% of catastrophic production failures came from mishandled non-fatal errors.

- **Swallowed error** — `except: pass`, `catch (e) {}`, `_ = f()`, `.unwrap_or_default()` on real errors, logging without propagating. The caller proceeds as if nothing happened.
- **Partial rollback** — side effect #1 succeeded, #2 failed, #1 never reverted. Common in multi-step DB operations without transactions.
- **Resource leak on error** — `open` without `finally`/`defer`/`with`; connection returned to pool only on happy path.
- **Error handler that can itself fail** — `catch` block calls another fallible API with no handling for its failure.
- **Wrong error mapped to a success code** — HTTP 500 → `return nil` → caller thinks it worked.
- **Error message leaks secrets** — stack trace or raw query in the response body.
- **Fail-open where fail-closed was meant** — exception in authorization check returns "allow."

## 2. Null / None / undefined dereference (CWE-476)

The single most common crash class across languages.

- After a lookup that can miss (`map.get`, `find`, SQL `single`, regex match, cast, deserialization).
- At API boundaries where inputs aren't narrowed.
- In the error path — cleanup code that dereferences a field that may not have been initialized.
- After a type narrowing / downcast that can fail silently (`as` in TypeScript, C++ `static_cast`).
- Default-constructed values that look valid but aren't (zero-value struct used before set).

## 3. Off-by-one and boundary errors

- `<` vs `<=` in loop bounds, especially when migrating between inclusive and exclusive intervals.
- Slice upper bound: is it `len` or `len - 1`?
- Empty collection: `list[0]`, `max(list)`, `heap.pop()` on empty.
- First / last iteration edge cases.
- Pagination: `offset + limit` overflow, `offset >= total`.
- Ring buffers and circular queues — full vs empty ambiguity.
- String slicing at character boundaries in the presence of multi-byte encodings.

## 4. Concurrency — atomicity, ordering, liveness

- **Check-then-act (TOCTOU, CWE-367)**: `if (exists(f)) use(f)` — f may be removed between the two calls.
- **Read-modify-write not atomic**: two readers load `counter=5`, both write `6`; one increment lost.
- **Lost update / write-write collision** without optimistic locking.
- **Double-locking / lock-order inversion** — thread A holds X wants Y, thread B holds Y wants X.
- **Missed notification** — waiter starts waiting after the notifier sent the signal.
- **Async yield invariant break** — state is consistent before `await`, something changes during the suspension, code after `await` is wrong.
- **Reentrancy** — function called recursively in the middle of mutating shared state.
- **Goroutine / thread leak** — spawned but never joined; grows unboundedly.
- **Channel / queue deadlock** — send on a buffered channel that's full; receive on an empty channel with no sender.
- **Memory ordering / data race** — non-atomic read of a value another thread is writing.

## 5. Integer & numeric bugs

- **Overflow** (CWE-190) — especially in size calculations, `len * size`, pagination, allocation sizes, timeouts.
- **Truncation** — storing a 64-bit value in a 32-bit column, or `(int) doubleVal`.
- **Signed / unsigned confusion** — negative `int` compared to `size_t`, wraparound.
- **Unit mismatch** — ms vs s, bytes vs bits, percent vs fraction, cents vs dollars, radians vs degrees.
- **Float equality** — `x == 0.1 + 0.2`; never works. Use epsilon or rational types.
- **Float accumulation** — summing many small values in a hot loop accumulates error.
- **`NaN`** — `NaN != NaN`; breaks sorting, set membership, hash keys.
- **Division by zero** — check, or use a type that can't be zero.
- **Precision loss** — IEEE 754 for money.

## 6. Resource leaks

- Files, sockets, DB connections, HTTP clients, locks, mutexes, goroutines, timers, subscriptions, temp files, shared memory segments.
- Worst on exception paths: the happy-path cleanup is there but not covered in the error branch.
- "Forgot to `defer close`" in Go.
- Python: `open(f).read()` leaks the file handle until GC.
- Holding locks across `await` / long IO — correctness-OK but causes starvation.

## 7. Injection & taint (OWASP A03, CWE-89/78/79/22/94/918/502)

Any data from an untrusted source reaching an interpreter without narrowing:

- SQL injection — string concat, format strings, `f"SELECT ... {name}"`.
- Command injection — `shell=True`, `system()`, building command strings.
- XSS — `innerHTML`, `dangerouslySetInnerHTML`, template without escaping.
- Path traversal — joining user input into a filesystem path without canonicalizing and checking prefix.
- SSRF — user-supplied URL passed to server-side fetch.
- XXE — XML parser with external entities enabled.
- Unsafe deserialization — `pickle.loads`, Java `readObject`, YAML `load` (not `safe_load`), PHP `unserialize`.
- `eval`, reflection, dynamic import with user input.
- ReDoS — regex compiled from user input, or catastrophic backtracking on user input.
- LDAP / XPath / NoSQL injection — same story, different sink.

## 8. Broken access control (OWASP A01, CWE-862/863/285)

- Missing authorization check on an endpoint — the user is logged in, but the action is allowed without a per-resource check.
- IDOR — object ID taken from the request, used to fetch without an ownership check.
- Authorization bypassed on one code path but not another (e.g., admin panel checks but the API does not).
- Check on the wrong object — fetch `user_id` from token but then operate on `target_id` from the request body.
- Client-side-only enforcement.
- Privilege escalation via mass-assignment (`user.role = "admin"` from JSON body).

## 9. Time, date, timezone, locale

- Naive vs timezone-aware datetime comparison.
- DST transitions — 02:30 on spring-forward day doesn't exist; 01:30 on fall-back happens twice.
- Leap seconds, leap years (Feb 29), month lengths.
- `toLowerCase` in Turkish locale — `"I".toLowerCase(tr_TR)` returns `ı` (dotless i), not `i`, so `"FILE".toLowerCase() == "file"` fails. The SSL-cert hostname-matching bypass lived here for years.
- `strftime` / `strptime` format mismatches.
- Clock skew between systems, relying on wall-clock time for ordering.
- Monotonic vs wall clock — wall clock can jump backward.

## 10. State-machine holes

- "Impossible" `(state, event)` combinations that are actually reachable via retries, races, or replays.
- Terminal states with pending operations — canceled order still delivers.
- Initial state not explicitly handled.
- Transitions that should be idempotent but aren't.

## 11. Retry and idempotency

- Non-idempotent operation retried on timeout — payment charged twice.
- Webhook / message handler without deduplication (at-least-once delivery is the default).
- Exponential backoff without jitter — thundering herd.
- Retrying on non-retryable errors (4xx) — pointless and can trigger rate limits.
- Retry budget / circuit breaker missing.

## 12. Cache & staleness

- Read-your-own-writes violation — write to primary, read from replica too soon.
- Cache invalidation missed on one write path.
- TTL mismatch between cache layers.
- Cached a mutable object whose caller mutates it.
- Negative caching a transient error.

## 13. Copy-paste and near-miss clones

- Cloned block where one variable wasn't renamed — still references the original.
- Two nearly-identical blocks with one subtle divergence — one is wrong.
- Constants duplicated; updated in one place, not the other.

## 14. API misuse, logic inversion, and flipped defaults

Both classes are really the same bug in different clothes: the code *says* one thing and *does* another because a polarity, argument, or comparison was flipped. The lens 5 (inconsistency) + lens 6 (invariants) pass catches most of these.

- Wrong argument order — `(src, dst)` vs `(dst, src)` in `memcpy`, `rename`, `copy`, `symlink`.
- Wrong comparison method — `==` vs `.equals` vs `Object.is` vs `===`; shallow vs deep equality.
- Wrong hash / dictionary key — mutable object hashed then mutated.
- Misreading a return value — `indexOf` returning `-1`, `strcmp` returning 0 on equality, `write(2)` returning partial length.
- Missing required call — `commit()` not called; `flush()` not called; iterator or scope guard not closed.
- Negated condition (`!`) in the wrong place.
- Flipped default — feature flag defaults "on" for untrusted users; fail-open where fail-closed was intended.
- "Allow-list" that's actually a "deny-list" and vice versa.
- Early `return` in the wrong branch — e.g., "return early if not authorized" accidentally allows when `authorized` is `false` because of an unrelated error in the authz check itself.

## 15. Regex bugs

- **ReDoS** — catastrophic backtracking from nested quantifiers, alternation with overlap, or `(a+)+` patterns, compiled against attacker-controlled input. Any regex engine without linear-time guarantees (PCRE, Python `re`, JS, Java, .NET — but NOT RE2 / Rust `regex`) is exposed.
- **Missing anchors** — `^...$` omitted on a validation regex. Classic auth-bypass: `^admin` matches `admin@evil.com`; `admin$` matches `evil-admin`.
- **Greedy vs lazy** quantifier confusion — `.*` eats past the intended boundary.
- **Regex compiled in a hot loop** — performance bug, occasionally a correctness bug if the pattern depends on captured state.
- **Character class locale drift** — `\w` / `\d` / `[a-z]` meaning different things under Unicode vs ASCII mode.

## 16. Serialization version skew

- Adding, removing, or renumbering a protobuf / Thrift / Avro field mid-flight across producer and consumer.
- JSON field type changing (string → number, string → object) between releases.
- Enum variant added on the writer before readers know it exists.
- Struct layout change (C/C++/Rust `repr(C)`) breaking binary interop without a version bump.
- `readObject` / `pickle` reading a class whose definition has changed.

## 17. Resource exhaustion / unbounded growth

- Unbounded in-memory queue, channel, map, or cache.
- Log file written forever without rotation.
- Goroutines / threads spawned per request with no backpressure.
- Connection pool with no max or no timeout.
- Retry budget missing — amplification under load.
- Metrics cardinality explosion (a per-user-id tag on a high-cardinality metric).

## 18. Crypto & randomness

- Using `Math.random()` / `rand()` for security tokens.
- MD5 / SHA-1 for anything that needs collision resistance.
- ECB mode, static IV, reusing nonces.
- DIY crypto — rolling your own key derivation, MAC, or cipher.
- Constant-time comparison missing for secrets (`==` on HMACs leaks via timing).
- Secrets in code, logs, URLs, or error messages.

## 19. Memory & undefined behavior (C / C++ / unsafe Rust)

- Use-after-free.
- Double-free.
- Buffer overflow / out-of-bounds write (CWE-787 — #2 on the 2024 CWE Top 25, #1 in 2023, perennially top-3).
- Reading uninitialized memory.
- Signed integer overflow (UB in C/C++).
- Strict aliasing violations.
- Unaligned access.
- `memcpy` with overlapping regions (use `memmove`).

## 20. Type-system escapes

- `any` / `unknown` / `Object` / raw pointers used to bypass checks.
- `unsafe` blocks in Rust without documented invariants.
- `cast` / `reinterpret_cast` from one layout to another without guaranteeing layout.
- Serialization round-trip losing a type distinction (e.g., `int64` → JSON → `float64` — silently loses precision above 2^53).
- JS prototype pollution — user-controlled key written into `Object.prototype` via unsanitized `Object.assign` / recursive merge.

## 21. Configuration, environment, and supply chain

- Hardcoded `localhost` / `127.0.0.1` that fails in a container.
- Env var read once at startup; changes don't take effect.
- Missing env var falling back to an unsafe default (e.g., empty allowlist meaning "allow all").
- Feature flag read per-request vs per-session inconsistencies.
- Different timezone / locale / encoding between dev and prod.
- Dependency pinned loosely (`^1.2`, `latest`) on a package the attacker can update.
- Typosquat / dependency-confusion candidate — a package with a name that's a small edit away from a popular one.
- Private package name colliding with a public registry name (install order vulnerability).
