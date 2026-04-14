---
name: bug-hunt
description: >
  Finds real bugs in code — defects with a specific failing input/state and a concrete observable
  consequence — using rotating expert lenses (null/boundary, error-path, concurrency, taint,
  inconsistency, invariants). Language-agnostic, works on any stack. Use when the user asks to
  "find bugs", "hunt bugs", "look for bugs", "audit for bugs", "check this for bugs", "bug hunt",
  "bug review", "find the bug", hands over code asking "what's wrong with this", "what will break",
  "where could this fail", "does this have bugs", "any bugs in this", "what could go wrong", or
  asks for a "race condition check" or "concurrency audit". Also triggers when given a stack trace
  or failing test and asked to find the cause. Do NOT use for style, naming, formatting, refactoring
  suggestions, writing new code, or broad multi-pass reviews — prefer code-scrutiny for general
  review and refactor-pro for refactoring.
allowed-tools: "Read Glob Grep Bash Agent"
argument-hint: "[scope: file, directory, commit range, stack trace, or blank for uncommitted diff]"
---

# Bug Hunt

You hunt for **real bugs** — defects you can prove with a specific input, state, or interleaving that produces observable harm. Bugs only. Style, naming, docs, and refactoring belong in `code-scrutiny` and `refactor-pro`. (ultrathink)

The methodology is grounded in empirical bug-finding research, cited inline where it earns a specific decision — not as a reading list.

## Context

- Branch: !`git branch --show-current 2>/dev/null || echo "not a git repo"`
- Uncommitted: !`git diff --stat 2>/dev/null | tail -5`
- Staged: !`git diff --cached --stat 2>/dev/null | tail -5`
- Recent commits: !`git log --oneline -10 2>/dev/null`

## Step 1 — Determine scope

Scope requested: $ARGUMENTS

| Input | Approach |
|---|---|
| Blank | Hunt uncommitted + staged changes. Start with `git diff` and `git diff --cached`. |
| File / directory path | Read fully, then pull recent commits: `git log --oneline -20 -- <path>`. |
| Commit range (`HEAD~3..HEAD`, SHA, branch) | `git diff <range>` plus `git log <range>` to capture intent. |
| Whole repo / "find bugs in this codebase" | Do not read everything. Triage with Step 2, hunt the highest-prior files, and name what you skipped. |
| Stack trace or failing test | Skip triage. Start at the top frame, read that file, and walk down the trace applying Lens 2 (error-path) at each frame. If it's a regression, `git bisect` on a minimal repro is almost always faster than reading. |
| Pasted snippet with no path | Hunt exactly what was pasted, say you lack surrounding context, and skip history priors. |
| Minified / generated / vendored / unreadable | Stop and ask for source. Do not hallucinate bug claims about code you cannot read. |
| Ambiguous across multiple candidates | List them and ask before hunting. |

Always read the **full file** around any change, not just diff hunks. A disproportionate share of bug-finding value lives in the interaction between new code and the surrounding 100–200 unchanged lines.

## Step 2 — Triage: weight your attention

Bugs cluster. Empirical priors, in rough order of strength:

1. **Files with fix history.** `git log --oneline --grep='fix\|bug\|regression\|hotfix' -- <file>` — prior defects predict the next one better than any other signal (Zimmermann et al., *Predicting Bugs from History*).
2. **Regression hunt.** If the symptom is a regression, `git bisect` on a minimal repro before reading code.
3. **Postmortem / issue-tracker priming.** `git log --grep='postmortem\|incident\|revert'`, plus any `docs/postmortems/` dir or tracker tags for this area. Prior incidents are the best prior for where the next bug lives.
4. **High relative churn** (churn normalized to file size). Churn beats raw complexity as a predictor (Nagappan & Ball, ICSE '05).
5. **High author dispersion** — files touched by many people across teams.
6. **Flaky tests, `@flaky` / `.skip` / retry annotations.** Flakiness is a concurrency or ordering bug advertising itself.
7. **Error-handling code** — `catch`, `except`, `rescue`, `defer`, `finally`, `Err(...)`, rollback and cleanup paths.
8. **Concurrency surface** — `lock`, `mutex`, `atomic`, `async`, `await`, `go `, `spawn`, `thread`, `channel`, `select`, `Arc`, `Mutex`, `RwLock`.
9. **Trust boundaries** — HTTP handlers, deserializers, IPC, file-path builders, subprocess calls, env-var readers, cross-service DB rows.
10. **Numeric and size math** — sizes, pagination, timeouts, money, shifts, unit conversions.
11. **Recently refactored code** — regression-prone, especially "just cleanup."
12. **LLM-authored or recently LLM-edited code** has characteristic failure modes — see Lens 5.

Name which priors you applied and which files you deliberately skipped.

## Step 3 — Rotate the six lenses

Do one lens pass, **stop**, then start the next. Each pass is a full re-read with one specific question. Rotating defeats saturation — reading for nulls with the same eyes will not find concurrency bugs (analogous to Regehr's observation that fuzzers saturate on a single coverage metric, the same effect applies to human reviewers). Scale the number of lenses to change size: a 5-line diff gets the two most relevant; a 500-line feature gets all six.

Full checklists (grep patterns, exhaustive sub-bullets) live in `references/lens-checklists.md` — pull that in when a first pass feels shallow or the code is high-stakes.

### Lens 1 — Null & boundary

*"What input or state makes this read, index, or arithmetic unsafe?"*

- Nullable things: after a lookup that can miss (`.get`, `find`, regex match, cast, deserialization); at API boundaries; in cleanup code.
- Collections: empty, one element, at `max`, at `max+1`.
- Numeric edges: zero, negative, `NaN`, `Infinity`, overflow, truncation, signed↔unsigned mix, unit mismatch (ms/s, bytes/bits, percent/fraction).
- Floats: `==` on floats is almost always wrong; `NaN != NaN` breaks sets and sorts.
- Unicode: combining characters, surrogate pairs, length-in-bytes vs code-points vs grapheme clusters.

### Lens 2 — Error path

*"If every fallible call on this path failed right now, would the program still be correct?"*

Yuan et al. (OSDI '14) analyzed 198 catastrophic failures across five distributed systems and found ~92% came from mishandled non-fatal errors — almost always on paths happy-path tests never touch. This is the richest bug seam; if you have time for one lens, use this one.

- Swallowed errors: empty `catch`, `except: pass`, `_ = f()`, `.ok()` discarding the error.
- Partial rollback: step 1 succeeded, step 2 failed, step 1 never undone.
- Resource release on every exit branch: files, sockets, locks, connections, transactions.
- Error handler that can itself fail: what if the `rollback()` call in `catch` also throws?
- Fail-open vs fail-closed: access-control decisions should fail closed; retries should usually fail fast.
- Caller preparedness: does the caller handle this specific error variant, or crash further up?

### Lens 3 — Concurrency & ordering

*"What happens if another thread / goroutine / process / request runs between these two lines?"*

- TOCTOU (CWE-367): check on a mutable resource followed by use that assumes the check still holds. Files, rows, configs, auth decisions.
- Read-modify-write atomicity: two workers doing `x = read(); write(x + 1)` lose updates. Needs CAS, row lock, or `UPDATE ... SET x = x + 1`.
- Lock ordering: can two code paths acquire the same locks in different orders? Classic deadlock.
- Yield-point invariants: at every `await` / IO / callback, shared state can mutate under you — does the code after the yield still hold its assumptions?
- Notifications: is the signal sent **after** the state the waiter is checking is actually updated?
- Retry + non-idempotency: timeout → client retries → duplicate side effect (payment charged twice, email sent twice).
- Memory ordering races, channel deadlocks, thread/goroutine leaks.

### Lens 4 — Taint / untrusted input

*"Where does untrusted data enter, and does it reach a sink without a narrowing check on the exact value used at the sink?"*

Color every byte from an untrusted source: HTTP bodies/headers/cookies/params, files, env vars, CLI args, IPC, queue messages, DB rows written by other services, `Date.now()`-seeded randomness used for security. Trace forward to sinks:

| Sink | Candidate | CWE |
|---|---|---|
| SQL concat / format strings | SQLi | 89 |
| Shell / `exec` / subprocess | Command injection | 78 |
| HTML / `innerHTML` / template | XSS | 79 |
| File path / `open` / `join` | Path traversal | 22 |
| Outbound URL fetch | SSRF | 918 |
| `pickle`, `readObject`, YAML `load` | Unsafe deserialization | 502 |
| `eval`, reflection, dynamic import | Code injection | 94 |
| Authorization-decision input | Access control bypass | 862 / 863 |
| Regex compiled from input | ReDoS | — |

Key refinement: is validation running on the **exact value** that reaches the sink, or on a copy that diverged before it got there?

**Broken access control is the #1 web vulnerability** — OWASP A01 since 2021, still #1 in the 2025 list finalized at Global AppSec USA in late 2025. Notable 2025 change: SSRF (formerly A10) was consolidated into A01 under an expanded "trust boundary violations" framing, so the taint table's SSRF sink is now also an access-control concern. For every sensitive action, confirm there is an authorization check on *this specific endpoint and this specific resource*, not just "the user is logged in."

### Lens 5 — Inconsistency (Engler)

*"Of N similar call sites, which one is different, and why?"*

The highest-yield lens in mature codebases. Engler et al., *Bugs as Deviant Behavior* (SOSP '01): statistically rare deviations from rules inferred from the code itself are usually bugs or undocumented exceptions.

- `rg` the function / idiom across the repo; read call sites side by side. The outlier is your candidate.
- **Symmetry pairs** — if you see one side, look for the other on every exit path: `lock`/`unlock`, `open`/`close`, `begin`/`commit`/`rollback`, `acquire`/`release`, `malloc`/`free`, `subscribe`/`unsubscribe`, `setup`/`teardown`.
- **Checked-then-used**: `if (x) use(x)` appears ten times; the eleventh uses `x` without checking.
- **Near-miss clones**: blocks where one identifier differs between siblings. Usually a forgotten rename.
- **Constant drift**: the same magic number in five places; four got updated, one didn't.

**LLM-generated code has characteristic inconsistency failure modes**: adjacent-but-wrong API calls (`arr.length()` vs `arr.length`, `list.append` vs `list.add`), plausible-but-nonexistent method names, imports from packages that don't exist, subtly wrong type signatures that compile only because of `any` / `interface{}` / `auto`. **Slopsquatting** makes the fake-import case actively dangerous: attackers now pre-register malicious packages under the exact names LLMs commonly hallucinate, so an unverified import isn't just a missing dependency — it can be a live supply-chain payload. Verify every import against the real registry, not just against "does it resolve on this machine." When the prior says the code was LLM-authored or recently LLM-edited, add these to your Lens 5 pass.

### Lens 6 — Invariants & state machines

*"What must be true here, and what path could violate it?"*

- Preconditions callers must satisfy: validated at the boundary, or assumed?
- State machine: enumerate `(state, event)` pairs; is every pair handled? What about "impossible" combinations actually reachable via retries, races, or replays?
- **Property-based thinking** — ask "what property should hold for *all* inputs of this shape?" This is the mental model behind QuickCheck / Hypothesis / proptest. Common properties: idempotence, monotonicity, conservation, round-trip (`decode(encode(x)) == x`), commutativity, ordering stability.
- Time, date, timezone: DST transitions, leap year/second, naive vs aware datetimes, wall clock vs monotonic, clock skew.
- Cache / staleness: read-your-own-writes violated, invalidation missed on one write path.
- Uniqueness enforced at the database (`UNIQUE` constraint), not via application-level check-then-insert — that's TOCTOU.

For a deeper enumeration of bug classes during any lens, consult `references/bug-taxonomy.md`.

## Step 4 — Witness test: smell → bug

A finding is only a **bug** when you can answer all three:

1. **Witness** — a specific input, state, timing, or sequence that reaches the bad state. "Could fail somehow" is not a witness.
2. **Harm** — an observable bad outcome: wrong result, crash, corruption, data loss, leak, security boundary crossed, hang, measurable latency regression. "Confusing to read" is not harm.
3. **Reachability** — a real caller (user input, public API, retry path, scheduled job) can drive it. The upstream may already rule it out — check before claiming.

Three tiers:

- **Bug** — all three present. Confidence `high`.
- **Likely bug** — witness and harm present, reachability depends on an upstream guarantee you couldn't verify. Ask the author to confirm.
- **Smell** — no witness yet, but an inconsistency or pattern worth a look. Flag as a question, not as a bug.

**If you cannot produce a witness, do not call it a bug.** Call it a smell, a question, or drop it. A **clean hunt is a valid and preferred outcome** — if the code is clean, say so and name what you looked at. Padding with speculative findings buries the real one when it shows up next week.

For findings that are real, also estimate **blast radius**: "one user per hour" and "every request on the hot path" are both reachable + harmful but differ by six orders of magnitude. Severity should reflect harm × rate.

## Step 5 — Report

```
## Bug Hunt: [what was reviewed]

**Scope:** [files/commits + how you prioritized]
**Lenses run:** null/boundary · error-path · concurrency · taint · inconsistency · invariants
**Result:** N bugs · M likely bugs · K smells · clean areas listed

### Bugs (confirmed)
### Likely bugs (need author confirmation)
### Smells (no witness — worth a look)
### Clean areas
[What you looked at and found nothing in — so the author knows what was actually covered]
```

**Every finding follows this exact shape — if a field is missing, the finding is not ready:**

```
[severity] [confidence] file:line  function_name
  Witness:       <input/state/sequence that triggers it>
  Harm:          <observable bad outcome, with blast radius>
  Reachability:  <real caller that can drive it>
  Fix sketch:    <concrete change, not "make it better">
```

- **severity**: `critical` (data loss, security, crash in prod path), `high` (wrong result for real users), `medium` (edge-case wrong result, resource leak under load), `low` (degenerate case, defensive gap).
- **confidence**: `high` (traced end-to-end), `medium` (one unverified upstream assumption), `low` (smell with a plausible but untraced witness).

### Example

```
[high] [high] src/auth/session.go:142  refreshToken
  Witness:       two concurrent refresh requests for the same session (e.g., two
                 browser tabs with silent-refresh timers firing within the same
                 second).
  Harm:          the second request overwrites the first's new refresh token;
                 the first client's next call presents a revoked token and is
                 logged out. Observable in logs as token_revoked spikes on the
                 hour when timers align. Affects every user with 2+ tabs.
  Reachability:  any user with two open tabs.
  Fix sketch:    optimistic locking via a monotonic refresh_token_version
                 column (CAS), or SELECT ... FOR UPDATE on sessions.id.
```

## Guardrails

- **Bugs only.** No style, naming, formatting, or refactoring findings — those belong to `code-scrutiny` / `refactor-pro`.
- **Every finding needs `file:line` plus a specific witness.** No witness → not a bug.
- **Only reference APIs, functions, flags, and packages that exist.** Verify with Grep before suggesting an import. Never invent method names — when reviewing LLM-authored code, do not inherit its hallucinations.
- **Defense-in-depth is not redundancy.** Two checks at a trust boundary are usually deliberate.
- **For multi-file hunts, launch parallel sub-agents** — one per independent file or module — then merge findings. Saves wall-clock and avoids cross-file contamination of attention.

## References

- `references/bug-taxonomy.md` — catalog of bug classes with language-agnostic examples. Pull in when hunting in an unfamiliar domain or when you want to broaden a lens pass beyond the patterns listed above.
- `references/lens-checklists.md` — exhaustive per-lens checklists with grep patterns. Pull in for high-stakes code or when a first pass feels shallow. The grep patterns are candidate-surfacers, not filters — expect false positives.
