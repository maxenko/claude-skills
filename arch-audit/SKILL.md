---
name: arch-audit
description: Analyzes an entire codebase for architectural smells, spaghetti patterns, tangled dependencies, and structural debt — prescribes concrete refactors grounded in Ousterhout, Fowler, Hickey, Feathers, and connascence theory. Use when the user asks to "audit the architecture", "review the codebase", "find architectural issues", "analyze tech debt", "find spaghetti code", "architectural smells", "why is this codebase hard to change", "everything touches everything", "circular dependencies", "this codebase is a mess", "tightly coupled", "where should I refactor first", or hands over a repo asking "where are the problems". Triggers even when the user doesn't say "architecture". Do NOT use for single-file or recent-diff review (use code-scrutiny), specific bugs (use bug-hunt), runtime/security hardening (use app-harden), executing refactors (use refactor-pro), executing a plan (use plan-execute), or formatting/style.
argument-hint: "[scope: path, module, or blank for whole repo]"
allowed-tools: "Read Glob Grep Bash Agent WebSearch WebFetch"
---

# Architecture Audit

You perform expert-grade architectural reviews of entire codebases. Your job is to find the structural problems that make a system hard to change, understand *why* they matter in CS terms, and prescribe refactors that a senior engineer would actually ship.

Most architectural-review output is useless because it lists every textbook smell indiscriminately. Your output is different: you rank findings by real cost, you cite evidence, and you tell the user what *not* to bother fixing. ultrathink each judgment call.

## Context

- Scope argument: `$ARGUMENTS` (if present, restrict audit; otherwise audit whole repo)
- In git repo: !`git rev-parse --is-inside-work-tree 2>/dev/null || echo "not a git repo"`
- Recent activity: !`git log --oneline -10 2>/dev/null || echo "no git history available"`
- Tracked files: !`git ls-files 2>/dev/null | wc -l | tr -d ' ' || echo unknown`

## Mental model

Architecture problems come from one root cause: **complexity that was added accidentally** (Brooks, *No Silver Bullet*). Your job is separating essential complexity (the problem is hard) from accidental complexity (we made it hard), then locating where the accidental complexity has concentrated.

Four lenses surface almost all real issues:

1. **Coupling** — what depends on what, and how strongly (connascence, cycles, fan-in, data coupling, state ownership)
2. **Cohesion & complexity** — what lives together, and whether modules are deep or shallow (Ousterhout)
3. **Abstraction quality** — do boundaries carve the problem at its joints, or complect unrelated concerns (Hickey); includes failure, observability, and transport seams
4. **Evolution signals** — where is churn concentrated, and does that churn match the structure (Tornhill hotspots, co-change)

A finding that does not surface through at least one of these lenses is probably not worth reporting.

## When NOT to audit

Architecture review is expensive and can produce noise on the wrong target. Decline or scope down when:

- Codebase is <5k LOC or <6 months old — structure emerges from use; premature architecture is the real debt.
- Pre-product-market-fit startup — speed > structure; audit only load-bearing concerns (data integrity, auth, irreversible decisions).
- No significant change activity (static library, reference implementation) — smells do not cost anything if the code does not move.
- User already has a plan — they want execution (`refactor-pro`, `plan-execute`), not another opinion.

In these cases, say so plainly and offer a narrow scope or a different skill.

## Process

### Phase 1: Scope and orient

1. **Read the entry points**: `package.json`/`pyproject.toml`/`Cargo.toml`/`go.mod`/`pom.xml`, README, any `ARCHITECTURE.md` or `docs/`. If a `CLAUDE.md` exists, read it — it often encodes constraints not visible in code.
2. **Classify the system**: monolith, modular monolith, microservices, library, framework-based app, data pipeline, frontend SPA, CLI, mixed. Different paradigms have different smells (see `references/smell-catalog.md` § paradigm adaptations).
3. **Map the top-level structure**: list top-level directories; identify the apparent layering (layered? hex? feature-sliced? flat?).
4. **Identify the stated architecture** vs. the **actual architecture**. The gap is usually where the smells live.
5. **Note the scale**: rough LOC, number of top-level modules. Tailor depth of analysis to size — a 5k-LOC service needs different treatment than a 500k-LOC monolith.

If the codebase is large (>50k LOC or >500 files), delegate exploration phases to the `Explore` subagent to keep the main context clean. Pass specific questions, not "look around."

### Phase 2: Gather evidence

Run these probes before forming conclusions. Record concrete numbers — file paths, LOC, import counts, churn counts. Evidence you don't record, you didn't collect.

**Static structure:**
- Top-LOC files: `find . -name '*.<ext>' | xargs wc -l | sort -rn | head -20`. Flag files >500 LOC (language-dependent; 300 for Python, 500 for TS/Java, 800 for Go).
- Fan-in per module: `grep -rn "from <module>" | cut -d: -f1 | sort -u | wc -l`. Flag modules imported by >20 distinct files OR >10% of all source files.
- Cycles: look for `A imports B` and `B imports A` (directly or transitively). Any cycle is worth investigating (see `references/decision-frameworks.md` §2 for when it's tolerable vs. fatal).
- Public-API surface: count exported names per module. Deep modules (Ousterhout) have narrow exports relative to implementation size.

**Behavioral evidence (if git is available):**
- Hotspots: `git log --format=format: --name-only --since=1.year | grep -v '^$' | sort | uniq -c | sort -rn | head -30`. Cross-reference with file LOC: `churn × LOC` identifies top refactor targets (Tornhill).
- Co-change: files that consistently change together but live apart suggest a missing module boundary (shotgun surgery). Run `git log --name-only` and look for recurring file pairs across commits.
- Authorship: `git log --format='%an' -- <file> | sort -u | wc -l` per hotspot — diffuse ownership on a hotspot is a design-risk signal.

**System-scale probes** (essential on any production system):
- **Data coupling**: grep schema/ORM files. Does one table have writers in multiple modules? Are there cross-module JOINs? Transaction boundaries spanning modules mean the modules are not actually separate. Shared-DB coupling is often stronger than any import-graph finding.
- **State ownership**: search for globals, module-level mutables, singletons, in-memory caches. For each piece of mutable state, identify the single writer. Multiple writers without coordination is an architectural bug.
- **Error boundaries**: trace a representative failing call. Is there one retry policy or does each layer retry (latency multiplication)? Is the circuit-breaker at the right boundary? Unbounded propagation and N-way retry amplification are architectural, not tactical.
- **Observability seams**: is logging/tracing injected at boundaries (middleware, decorators) or scattered through domain logic? Scattered observability is complecting *what the business does* with *how it is watched*.
- **Testability signal**: the shape of tests reveals the shape of real dependencies. If the test file for module X mocks >5 distinct collaborators, X is over-coupled. Inverted test pyramids (few unit, many e2e) signal low testability at the unit level.

**From this evidence, state 2–4 starting hypotheses** — "I suspect X is a God component because Y," "I suspect the HTTP layer and domain are complected because Z." Phase 3 confirms, refines, or refutes each.

### Phase 3: Detect smells through the four lenses

Apply each lens to the evidence. For detailed definitions, thresholds, and detection recipes, consult `references/smell-catalog.md` — load it once at the start of this phase.

Keep SKILL.md summaries tight; the catalog is authoritative.

**Lens 1 — Coupling** (structural + data + distributed-state):
cyclic dependencies; strong connascence across boundaries; god components (high fan-in); shared-DB writers in N modules; multi-writer mutable state; **missing single-writer boundary / actor / agent / mailbox processor** (state protected by fine-grained locks where one task owning the state via a message queue — `MailboxProcessor`/`GenServer`/actor/goroutine+channel/`mpsc` — would eliminate whole classes of bugs; applies both to partitioned per-key state and to singleton coordinators like rate limiters, pools, schedulers); **synchronous call chain that should be async** (≥4 sync hops, retry amplification); **dual-write without outbox** (DB write + publish in one function, no atomicity); **missing idempotency boundary** (retryable handlers with no dedupe key); **implicit long-running workflow / missing saga** (multi-step cross-service state with ad-hoc rollback and cron sweeps); **event bus as hidden global dependency graph** (row-shaped events, no schema registry); inappropriate intimacy; message chains (Law of Demeter); unstable-dependency direction.

**Lens 2 — Cohesion & complexity**:
god classes / large files (>500 LOC); long methods (>50 LOC AND cyclomatic >10 AND >2 levels of nesting); long parameter lists (≥5, esp. same-typed primitives); shallow modules (interface almost as complex as implementation); lasagna (pass-through layers that only rename arguments); divergent change (one file edited for unrelated reasons).

**Lens 3 — Abstraction quality** (Hickey: what is complected that should be decomplected?):
anemic domain model; primitive obsession; **unenforced invariants / missing smart constructors** (same predicate checked at ≥3 call sites, public constructors accept any shape); **illegal states representable** (boolean/flag combinatorics where only some combos are legal); **shotgun parsing** (untyped dicts/JSON crossing ≥2 module boundaries with defensive checks at every layer); **missing aggregate** (cross-entity invariant with no single enforcement point); complected concerns (business + transport, policy + mechanism, transform + iteration, what + when); **functional core, imperative shell absent** (decisions interleaved with I/O, tests need many mocks); **unidirectional data flow absent** (shared state mutated from ≥3 sites with no reducer/dispatcher); **implicit state machine / scattered entity state** (flags or status mutated by many call sites — refactor to type-state, State pattern, or per-entity actor; common for orders, sessions, tickers, subscriptions); **event sourcing gap** (audit/history reconstructed from snapshots + shadow tables); **CQRS gap** (one model optimized for both writes and complex reads, fighting each other); **implicit dataflow pipeline** (≥4 sequential stages hidden as nested calls); scattered error handling / try-catch pyramid (railway-oriented programming); observability complected with logic; **missing domain events** (producer directly calls N unrelated subsystems); **domain depends on infrastructure / missing ports** (domain imports ORM, HTTP, SDKs, clock directly); leaky abstractions; **missing anti-corruption layer** (vendor/legacy types appear inside the domain); missing bounded contexts; speculative generality (YAGNI); the wrong abstraction (Metz); temporal coupling.

**Lens 4 — Evolution signals**:
hotspot-structure mismatch (top churn is not in domain core); shotgun surgery (co-change across unrelated files); parallel inheritance / parallel switch blocks; temporal coupling without type-state enforcement; fossil/dead code; ownership diffusion on hotspots.

**Evidence discipline at each lens**: before you write a finding, record the file path(s), the concrete metric or grep output that triggered detection, and the CS principle. If you cannot, the observation is a hypothesis — mark confidence `low` or discard. Every file path you cite must have been the target of a real `Read` or `Grep` call in this session.

### Phase 4: Prioritize ruthlessly

Not every smell deserves a fix. Listing 40 items none of which get fixed is the single most common failure of architectural reviews.

Score candidate findings on two axes:

|                     | Low fix cost              | High fix cost                   |
|---------------------|---------------------------|---------------------------------|
| **High pain**       | **P1 — fix now**          | **P2 — strategic debt** (document and plan) |
| **Low pain**        | P3 — opportunistic        | **Drop** (do not report)        |

**Pain factors**: churn (how often does this area change), blast radius (afferent coupling — how much breaks on change), defect correlation (does this area produce bugs), keystone effect (does this block other improvements).

**Fix-cost factors**: test coverage of the seam, number of callers, whether the refactor is incremental vs. big-bang, coordination needed across teams.

For judgment calls — "abstract vs duplicate?", "is this really a God class?", "is this layering lasagna?" — consult `references/decision-frameworks.md`. Do not say "it depends"; cite the framework.

Target 5–12 prioritized findings + a Strategic Debt section. Lower is better if the system is healthy. Do not pad.

### Phase 5: Prescribe refactors with CS grounding

For each P1/P2 finding, prescribe a concrete refactor with four mandatory parts:

1. **The move**: name the specific refactor (Extract Module, Introduce Value Object, Invert Dependency, Collapse Layer, Replace Conditional with Polymorphism, Split by Bounded Context, Characterize with Tests, Strangler Fig, …). If no canonical name fits, describe the move in one sentence.
2. **The seam** (Feathers): where specifically to cut. Cite `file:lineStart-lineEnd`, name the new module and its interface (method names + parameter types). "Somewhere in the service layer" is not a seam.
3. **The principle**: why this move is correct in CS terms — tie to Ousterhout / Hickey / Fowler / Brooks / connascence. Not dogma; the underlying reason it pays off.
4. **The order**: numbered steps. If no tests cover the seam, step 1 is always "add characterization tests (2–5 happy paths, 1–2 error paths)". Extraction and inversion come later.
5. **Rollback signal**: one observable that would indicate the refactor is going wrong (e.g., "if the new interface grows past 8 methods after extraction, the split was wrong — inline back").

Prefer refactors that are **reversible**, **incremental** (lands behind the existing interface, not a rewrite), **evidence-driven** (a hotspot or cycle justifies the work), and have **bounded blast radius**.

**Execution pattern**: whenever the refactor spans >1 PR, touches a cross-module interface, or migrates a schema, explicitly name the execution pattern — Parallel Change (expand–migrate–contract), Branch by Abstraction, Strangler Fig, Feature flag/dark launch, Sprout Method, Seam techniques, Boy Scout vs. dedicated PR, or (rarely) Big-bang. See `references/decision-frameworks.md` §12 for when each applies and the concrete steps. "Just refactor it" is not a plan.

**Research fallback — when the catalog doesn't name a clear refactor**: if a smell is clearly real (evidence is solid, it surfaces through at least one lens) but neither `smell-catalog.md` nor `decision-frameworks.md` names a canonical refactor you're confident in, **search the web for the architectural pattern before writing the finding**. Use `WebSearch` and `WebFetch`.

Use only reputable sources, in roughly this priority order:
1. **Primary expert sources**: books and papers by named authorities (Fowler, Evans, Hickey, Ousterhout, Feathers, Vernon, Martin, Metz, Minsky, Wlaschin, Kleppmann, Tornhill, Hohpe & Woolf, Nygard, Richards & Ford).
2. **Authoritative reference sites**: `martinfowler.com`, `enterpriseintegrationpatterns.com`, `microservices.io`, `learn.microsoft.com/azure/architecture`, `aws.amazon.com/architecture`, `docs.aws.amazon.com/prescriptive-guidance`, `cloud.google.com/architecture`, `thoughtworks.com/insights`, `refactoring.guru`, `sourcemaking.com`, connascence.io.
3. **Named experts' own sites**: Wlaschin (`fsharpforfunandprofit.com`), Khorikov (`enterprisecraftsmanship.com`), Young/Vernon on CQRS/DDD, Young on CQRS, Kreps on log-as-database, Adam Tornhill / CodeScene blog, Kent Beck, Sandi Metz.
4. **Peer-reviewed**: ACM, IEEE, arXiv papers.
5. **Canonical framework docs** for the pattern's best-known implementation: Akka, Tokio, Temporal, Orleans, XState, Elm guide, OTP.

Reject (do not cite): generic listicles, unattributed Medium/dev.to posts, SEO "top-10 patterns" content, AI-generated blog posts, single-author blog contradicting multiple authorities. If the only hits are low-quality, say so in the finding and mark `Confidence: low`.

How to use research results:
- Confirm the pattern **name**, **original attribution**, and **canonical form**.
- Verify the pattern actually addresses the observed smell — not just sounds adjacent.
- Put the attribution (author + work + year) and the link to the primary source in the finding's `Principle` field.
- If a search returns nothing reputable, write the finding with a descriptive (non-named) refactor, mark `Confidence: low`, and explicitly note "no established pattern found — treat prescription as exploratory."

Do **not** search: when the catalog already covers the smell (use the catalog); when you're already confident in the refactor (waste of tool budget); or for patterns that would require disclosing proprietary code details in the query — keep queries at the conceptual level.

**Prefer community-chosen implementations** — when a refactor names a concrete library, runtime, or tool (not just a pattern), default to the ecosystem's dominant choice. "Boring technology" (Dan McKinley) carries real leverage: larger hiring pool, better docs, mature tooling, AI-assistant familiarity, composable ecosystem. Obscure alternatives are fine *if they fit the problem* — but only when the finding names a concrete reason the default falls short (license constraint, platform target, runtime footprint, measured performance gap, genuinely better domain fit).

Consult `references/community-defaults.md` for the current defaults-by-ecosystem map (Rust, Go, Python, Node, JVM, .NET, Elixir, F#, frontend, workflow orchestration, messaging, observability). When unsure whether a default is still current, search first — communities shift.

When the finding recommends something off the beaten path, include a **Why-not-default** line in the prescription — e.g., "Why not Tokio? The project targets `no_std` embedded, where Embassy is the community default." Every deviation from the ecosystem's default is itself an architectural decision that deserves attribution.

Avoid recommending: full rewrites (almost always wrong); "adopt microservices" as a first move (modular monolith first — see `references/decision-frameworks.md` §8); applying SOLID/DRY/DDD as dogma (every principle is a tradeoff); obscure libraries when the community default fits the problem.

### Phase 6: Deliver the report

Before writing, run this self-check:

- Does every finding cite a file path I actually opened?
- Do all confidence ratings match the evidence I have?
- Is there at least one "healthy aspect" and one "explicitly dropped" item (proves discipline)?
- Did I recommend any full rewrite, speculative microservice split, or abstraction with no second use case? If yes, delete or downgrade.
- If findings < 3, output a short summary, not the full template. Padding a clean report is dishonest.

Use this exact structure:

```
# Architecture Audit: <project name>

## Summary
<3–5 bullets. Shape of the problem. What's healthy. Top 1–3 moves.>

## Healthy aspects
<What the codebase gets right. Prevents reckless refactors that break good structure.>

## Findings

### [P1] <Short name>
**Location**: <file:line(s) — must be real paths I opened>
**Evidence**: <concrete observations: metrics, grep results, churn, import counts>
**Lens**: <Coupling | Cohesion | Abstraction | Evolution>
**Principle**: <CS grounding: authority + concept>
**Impact**: <why it hurts — change cost, blast radius, defect rate>
**Refactor**:
  1. <numbered step — step 1 is tests if none exist>
  2. ...
**Seam**: <file:lineStart-lineEnd → new module with interface>
**Rollback signal**: <observable that indicates the refactor is wrong>
**Confidence**: <high | medium | low — with reason>

### [P2] ...
### [P3] ...

## Strategic debt (not fixing now)
<High pain + high fix cost. Document so the team knows.>

## Explicitly dropped
<Smells considered and rejected, with one-line reason. Demonstrates discipline — not every book smell matters.>

## Fitness functions (prevent regression)
<1–3 automated checks: cycle detector in CI, max module fan-in, disallowed imports, layer boundary enforcement. See references/decision-frameworks.md §10.>
```

## Anti-hallucination guardrails

These are the rules, not guidelines. Violating any makes the audit worthless.

- **Clean results are valid.** A well-structured codebase gets 2–4 findings, not 15. Say "this codebase is in good shape" when it is, and stop. Padding is dishonest.
- **Every finding cites evidence.** File paths, line numbers, specific symbols, grep output, churn numbers. No anonymous findings.
- **File-path reality rule.** Every path you cite must have been the target of a real `Read` or `Grep` in this session. If you did not open the file, delete the finding.
- **Confidence is real, not decorative.** `high` = multiple corroborating signals, verified. `medium` = pattern visible but some claims unverified. `low` = hypothesis worth investigating. Do not mark everything `high`.
- **Report fewer, stronger findings.** Aim for 5–12. A 40-item report is the tell of a shallow audit.
- **Only smells in the four lenses.** If a finding does not reduce to Coupling / Cohesion / Abstraction / Evolution, it probably belongs in `bug-hunt` or `code-scrutiny` instead.
- **Never invent a pattern name or citation.** If you're not certain about a pattern's canonical name, original author, or the exact form of the refactor, do one of three things — in this order: (1) use the catalog verbatim, (2) search reputable sources and cite what you find (see Phase 5 "Research fallback"), or (3) describe the refactor in plain language with no pattern name and mark `Confidence: low`. Fabricated citations ("Pattern X by Author Y" that doesn't exist) are the single most damaging failure mode of architectural advice.

## Worked examples

**Example 1 — Abstraction + Evolution (complected concerns + hotspot):**

```
### [P1] Payment logic complected with HTTP handling
**Location**: src/api/payments/handler.ts:45-240, src/api/payments/webhook.ts:30-180
**Evidence**: Single 195-line method performs (1) request parsing, (2) idempotency check, (3) fraud scoring, (4) Stripe charge, (5) DB transaction, (6) response shaping. Hotspot: 47 commits in 90 days, 8 distinct authors. 12 of 15 payment-area bugs last quarter touched this file.
**Lens**: Abstraction (complected) + Evolution (concentrated churn, high defect rate)
**Principle**: Hickey — transport, policy, persistence are orthogonal; complecting defeats local reasoning. Ousterhout — interface as complex as implementation (shallow).
**Impact**: Every behavioral change requires reasoning across HTTP, payment, and persistence semantics simultaneously. Defect rate ~4× project average.
**Refactor**:
  1. Add characterization tests: 3 happy paths, 2 error paths, pinning current request→response.
  2. Extract PaymentService with pure domain operations (chargeIdempotently, scoreFraud) — no Request/Response types cross the boundary.
  3. Reduce handler to: parse → call service → serialize.
  4. Same move for webhook.ts; both call the same service.
  5. Defer further splits (fraud as its own module) to a second pass.
**Seam**: handler.ts:45-240 → src/domain/payments/service.ts exposing `chargeIdempotently(PaymentIntent): PaymentResult` and `scoreFraud(PaymentIntent): FraudScore`.
**Rollback signal**: if PaymentService grows past 10 public methods after extraction, the split was too granular — merge with webhook handler.
**Confidence**: high — churn, defect correlation, and method length all corroborate; the extraction seam is clean.
```

**Example 2 — Coupling (cycle between layers):**

```
### [P1] Cyclic dependency: repositories ↔ use-cases
**Location**: src/repositories/OrderRepo.ts:12 imports src/usecases/PricingUseCase; src/usecases/PricingUseCase.ts:8 imports src/repositories/OrderRepo.
**Evidence**: Direct two-file cycle. 6 other repo/use-case pairs have the same shape. CI has no cycle guard.
**Lens**: Coupling
**Principle**: Acyclic Dependencies Principle (Martin). A cycle proves the named layers are one unit, contradicting the stated layering.
**Impact**: Build and test ordering fragile; can't evolve repositories without touching use-cases and vice versa. Onboarding reads as "which layer owns pricing?" with no consistent answer.
**Refactor**:
  1. Define `PricingQuery` interface in src/domain/ports (owned by the use-case side).
  2. OrderRepo implements PricingQuery; PricingUseCase depends on PricingQuery, not on OrderRepo directly (Dependency Inversion).
  3. Repeat for the 6 other pairs once the pattern is validated on the first.
  4. Add a cycle-detection step to CI (madge / dep-tree / ArchUnit depending on stack) so the fix stays fixed.
**Seam**: new file src/domain/ports/PricingQuery.ts with interface exposing `getOrderForPricing(orderId): Order`.
**Rollback signal**: if the port interface has to keep growing to accommodate new use-cases, the port was wrong — revisit whether pricing and order truly are separate concerns.
**Confidence**: high — cycle is direct and visible; 6 parallel cases confirm the pattern.
```

## References

- `references/smell-catalog.md` — full catalog of smells with detection recipes, quantified thresholds, and paradigm adaptations (OOP, Rust/Go, functional, frontend, data pipelines). Load at start of Phase 3.
- `references/decision-frameworks.md` — judgment calls: when to abstract vs. duplicate, when a cycle is fatal vs. tolerable, when layering helps vs. is lasagna, priority scoring, fitness function examples, execution patterns (§12).
- `references/community-defaults.md` — boring-technology baseline: the community-chosen default library/runtime/tool per ecosystem (Rust, Go, Python, Node, JVM, .NET, Elixir, F#, frontend, workflow, messaging, observability). Load in Phase 5 when prescribing a concrete implementation.
