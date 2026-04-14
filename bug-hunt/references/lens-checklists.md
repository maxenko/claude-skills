# Lens Checklists

Exhaustive per-lens checklists for when the first pass feels shallow or the code is high-stakes. Each lens has (a) the question to hold in mind, (b) grep/search patterns to surface candidates fast, and (c) the specific things to look for in each candidate.

## Contents

- Lens 1 — Null & boundary
- Lens 2 — Error path
- Lens 3 — Concurrency & ordering
- Lens 4 — Taint / untrusted input
- Lens 5 — Inconsistency (Engler's method)
- Lens 6 — Invariants & state machines

---

## Lens 1 — Null & boundary

**The question:** "What input or state makes this read, index, or arithmetic unsafe?"

**Search patterns:**

```
rg '\.(get|find|first|last|head|tail)\(' -n
rg '\[[^:\]]*\]' -n                          # direct indexing
rg '\bunwrap\(\)|\.get!|!\.|\?\.' -n         # force-unwrap / optional chain
rg '\b(len|length|size|count)\b' -n          # boundary math
rg '==\s*(NaN|null|None|nil|undefined)' -n   # wrong-way comparisons
```

**Per candidate, check:**

- Can the container be empty? Zero / one / `max` / `max+1` elements?
- Can the key / index be missing, negative, or out of range?
- If this is after a regex match or parse — is the "no match" branch handled?
- For `get(key, default)`: is the default actually a valid value downstream, or just "something to avoid a crash"?
- For numeric: zero, negative, `NaN`, `Infinity`, overflow, signed/unsigned mix, unit mismatch.
- Floats: is equality tested with `==`? Is `NaN` handled in comparison / hash / sort?
- Unicode: is string length measured in bytes, code points, or grapheme clusters? Does it matter here?

---

## Lens 2 — Error path

**The question:** "If every fallible call on this path returned an error right now, is the program still correct?"

**Search patterns:**

```
rg '\b(try|except|catch|rescue|defer|finally|Result|Err|Ok)\b' -n
rg '\bcatch\s*\([^)]*\)\s*\{\s*\}' -n        # empty catch
rg '_ = ' -n                                  # Go ignored return
rg '\.ok\(\)|\.unwrap_or\(|\.unwrap_or_default' -n
rg 'panic!|panic\(|os\.exit|System\.exit' -n
rg '\brollback\b' -n
```

**Per candidate, check:**

- **Swallowed errors**: empty `catch`, `except: pass`, `_ = f()`, `.ok()` discarding the error.
- **Partial rollback**: step 1 succeeded, step 2 failed, step 1 never undone. Especially across DB + filesystem, DB + external API.
- **Resource release**: file/socket/lock/connection released on every exit including the error branches.
- **Error handler that itself fails**: what if the `rollback()` call in `catch` also throws?
- **Error message quality**: is the message useful for debugging a 3am page, and does it avoid leaking secrets?
- **Propagation vs conversion**: is the error converted correctly, preserving the underlying cause?
- **"Fail open" vs "fail closed"**: which did the author pick? Is that correct for this sink? (Access-control decisions should fail closed.)
- **Caller preparedness**: does the caller handle this specific error variant, or assume it can't happen?

---

## Lens 3 — Concurrency & ordering

**The question:** "What happens if another thread / goroutine / process / request runs **between these two lines**?"

**Search patterns:**

```
rg '\b(lock|mutex|unlock|atomic|volatile|sync|Mutex|RwLock|Arc)\b' -n
rg '\b(async|await|spawn|go\s|Thread|Goroutine)\b' -n
rg '\b(channel|chan\s|Queue|select\s*\{)' -n
rg '\bsleep\b' -n                            # timing-based correctness is a smell
rg '\b(if .* exists|stat|lstat|access)\b' -n # TOCTOU candidates
```

**For each piece of shared mutable state, check:**

- **TOCTOU (CWE-367)**: check on a mutable resource followed by use that assumes the check holds. Files, rows, config, auth decisions, feature flags.
- **Read-modify-write atomicity**: two workers doing `x = read(); write(x + 1)` — lost update? Needs CAS, row lock, or `UPDATE ... SET x = x + 1`.
- **Lock ordering**: multiple locks acquired; can any code path acquire them in a different order?
- **Holding a lock across `await` / IO / callback**: starvation, deadlock if the callback tries to re-acquire.
- **Yield point invariants**: before `await`, state is consistent. During the suspension, something mutates. After `await`, the code assumes the old state.
- **Notification ordering**: `signal()` / `notify()` / channel send — is it sent *after* the state the waiter is checking is actually updated? Is the waiter's check + wait pattern correct (spurious wakeups)?
- **Retry + non-idempotency**: timeout causes the client to retry; the server already completed the first request. Duplicate side effect?
- **Goroutine / thread lifecycle**: spawned and joined? Leaked? Cancel propagation via `context` / `CancellationToken`?
- **Memory ordering**: non-atomic read of a value that another thread writes. Data race.
- **Select / channel deadlock**: send on full buffered channel with no receiver; receive on empty with no sender.

---

## Lens 4 — Taint / untrusted input

**The question:** "Where does untrusted data enter, and does it reach a sink without a validating narrowing step on the exact value used at the sink?"

**Sources to color:**

- HTTP: bodies, headers, query params, cookies, URL paths.
- CLI args, env vars.
- Files — filenames **and** contents.
- IPC, message queues, pub/sub.
- DB rows written by other services (don't assume your schema is clean).
- Randomness that's actually `Date.now()` or attacker-influenced seed.

**Sinks and corresponding CWE:**

| Sink | CWE |
|---|---|
| SQL string concat / format | 89 |
| Shell / `exec` / `system` | 78 |
| HTML / template / `innerHTML` | 79 |
| File path / `open` / `join` | 22 |
| Outbound URL fetch | 918 |
| Deserializer (`pickle`, `readObject`, YAML unsafe) | 502 |
| `eval` / reflection / dynamic import | 94 |
| Authorization input | 862 / 863 |
| Regex compiled from input | ReDoS |
| LDAP / XPath / NoSQL query | 90 / 643 / 943 |

**Per flow, check:**

- Is the validation applied to the **exact same value** that reaches the sink, or to a copy / normalized form that diverged?
- Is the sink using a safe API (parameterized query, escaped template, `shell=False`, `path.Join` + prefix check)?
- Is the path canonicalized **before** the prefix check (else `../` escapes)?
- For regex compiled from input: anchored? Linear-time engine (RE2) or backtracking?
- For deserialization: is the input from a trusted source? Is the type restricted?

---

## Lens 5 — Inconsistency (Engler's method)

**The question:** "Of N similar call sites, which one is different, and why?"

**Technique:**

1. Identify a repeated pattern — a function call, an idiom, a check.
2. `rg` for it across the codebase.
3. Read each call site side by side.
4. The outlier is either a bug or an undocumented exception. Investigate.

**Specific patterns to inconsistency-check:**

- **Symmetry pairs** — for each `open` / `lock` / `begin` / `acquire` / `subscribe`, look for the matching close / unlock / commit / release / unsubscribe on **every exit path**.
- **Checked-then-used**: `if (x != null) use(x)` in ten places, one place uses `x` without the check.
- **Authorization checks**: ten endpoints check permissions, one doesn't. (Classic OWASP A01 pattern.)
- **Error wrapping**: nine handlers wrap with context, one returns bare.
- **Copy-paste near-miss**: blocks where one identifier differs between siblings — usually a forgotten rename. Candidate names: `user_id` vs `userId`, `from` vs `to`, `src` vs `dst`, `old` vs `new`.
- **Initialization**: the constructor sets nine fields; one is missing.
- **Constant drift**: the same magic number in five places; four got updated, one didn't.
- **Serialization / deserialization symmetry**: every field serialized must be deserialized, and vice versa.

---

## Lens 6 — Invariants & state machines

**The question:** "What must be true here, and what path could violate it?"

**Technique:**

For each function or module, write (mentally) the invariants:

- **Preconditions**: what must be true on entry? Validated or assumed?
- **Postconditions**: what must be true on exit? Always, or only on the success path?
- **Loop / iteration invariants**: what holds at the top of every iteration?
- **Class / struct invariants**: what fields must be consistent with each other? Can any public method leave the object in an inconsistent state?
- **State machine**: enumerate `(state, event)` pairs. For each, is there explicit handling?

**Per invariant, look for:**

- **Early return** that skips the code maintaining the invariant.
- **Exception** that exits with the invariant violated.
- **Reentrancy** during an intermediate state.
- **Concurrent mutation** while the invariant is temporarily broken (common between two writes).
- **Replay / retry** that observes an intermediate state.

**Specific invariant classes worth checking:**

- **Idempotency**: is this safe to retry? Payments, emails, webhooks, "create resource" endpoints.
- **Read-your-own-writes**: after write, will the next read see it? Replica lag, cache, eventual consistency.
- **Ordering**: are events processed in the order they occurred? At scale, usually no.
- **Monotonicity**: counters only go up, sequence numbers never reused, version numbers strictly increasing.
- **Conservation**: total money before = total money after; total inventory conserved across transfer.
- **Time**: DST, leap year, timezone, naive vs aware, wall vs monotonic clock.
- **Uniqueness**: `UNIQUE` constraint enforced at the database, not just checked in application code before insert (check-then-insert is a TOCTOU).
