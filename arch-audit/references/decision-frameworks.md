# Decision Frameworks

When the smell catalog gives you a signal, you still have to decide whether to act and how. These frameworks handle the judgment calls that separate a good audit from a dogmatic one.

Every framework here exists because the "obvious" rule is wrong in common cases.

## Contents

1. [Duplication vs. abstraction](#1-duplication-vs-abstraction) — when DRY misfires; Metz, AHA, Rule of Three
2. [When is a cycle a bug?](#2-when-is-a-cycle-a-bug) — tolerable vs. always-kill cycles; refactor moves
3. [Is this layering useful or lasagna?](#3-is-this-layering-useful-or-lasagna) — the layer-elision test
4. [Deep vs. shallow modules (Ousterhout test)](#4-deep-vs-shallow-modules-ousterhout-test) — benefit vs. cost of interfaces
5. [Abstract domain primitive vs. raw type?](#5-abstract-domain-primitive-vs-raw-type) — when to introduce value objects
6. [When SRP violations matter](#6-when-srp-violations-matter) — rate-of-change test
7. [SOLID as tool, not dogma](#7-solid-as-tool-not-dogma) — when to apply each principle
8. [Microservices vs. modular monolith](#8-microservices-vs-modular-monolith) — prerequisites before extraction
9. [When to rewrite vs. refactor](#9-when-to-rewrite-vs-refactor) — strangler fig default
10. [Fitness functions — turning findings into ongoing enforcement](#10-fitness-functions--turning-findings-into-ongoing-enforcement)
11. [Priority scoring template](#11-priority-scoring-template) — Pain / Cost ranking
12. [Execution patterns for large-scale refactors](#12-execution-patterns-for-large-scale-refactors) — Parallel Change · Branch by Abstraction · Strangler Fig · Feature flag/dark launch · Sprout Method · Seam techniques · Boy Scout vs. dedicated PR · Big-bang criteria

---

## 1. Duplication vs. abstraction

**The question**: Two (or more) places have similar code. Abstract, duplicate, or something else?

**The default instinct is wrong.** "DRY" is often misapplied — reducing duplication at all costs creates worse problems than it solves.

**Framework**:

| Condition | Action |
|-----------|--------|
| The duplicates encode the *same concept* that will always change together | Abstract (this is real DRY) |
| The duplicates *look similar* but come from unrelated concerns (coincidental duplication) | Leave duplicated |
| The duplicates are similar now but the domains suggest they will diverge | Leave duplicated until divergence proves they are the same |
| You have already abstracted and downstream callers pass flags to disable parts of the abstraction | The abstraction is wrong — inline back to duplicates |

**Authorities**:
- Sandi Metz: *"Duplication is far cheaper than the wrong abstraction."* The cost of a wrong abstraction compounds; the cost of duplication is linear.
- Kent C. Dodds: AHA — "Avoid Hasty Abstractions." Wait until the shape of the repetition is clear.
- Rule of Three: do not abstract on the second occurrence. Wait for the third; only then is the shape reliable.

**Audit prescription**: Only call out duplication as a smell if the duplicates encode the same semantic concept. Syntactic similarity is not enough.

---

## 2. When is a cycle a bug?

**The question**: You found a dependency cycle. Kill it or tolerate it?

**Default**: Kill it. Cycles are structural lies about the module graph. But tolerate these specific cases:

- **Intra-module cycles** (within the same package/namespace) where the module is genuinely a single concept. Example: mutually recursive functions that parse a grammar. Fine.
- **Test-only cycles** where test code depends on test utilities. Annoying but low-cost.
- **Cycles mediated by an interface** where the concrete types point one way and the interface points the other. Often the intended design (DIP in action).

**Always kill**:
- Cycles between "layers" in a layered architecture (controller → service → repository → controller). Proof the layering is a fiction.
- Cycles between supposedly independent bounded contexts / features. Proof the contexts are not actually independent.
- Cycles that cross deployment units / services. These are architectural disasters.

**Refactor moves to break cycles**:
1. **Dependency inversion**: introduce an interface owned upstream.
2. **Extract shared kernel**: the thing both sides need becomes a third module both depend on.
3. **Event/callback**: one side emits, the other subscribes. Use sparingly — events can hide coupling.
4. **Merge**: if the cycle is genuinely a single concept, merge the modules. Do not pretend they are separate.

---

## 3. Is this layering useful or lasagna?

**The question**: The codebase has many layers. Is this good clean architecture or pure pass-through?

**Each layer must earn its keep by hiding something**. The test:

> Cross out the layer mentally. Can the layers above still function meaningfully, or do they now know things they previously did not?

If the layers above *now know more* (implementation details, transport concerns, storage format), the layer was hiding something real — keep it.

If the layers above function identically except that the call chain got shorter, the layer was pure pass-through — collapse it.

**Common load-bearing layers**:
- Persistence (hides ORM, transactions, serialization)
- Transport (hides HTTP, serialization, auth)
- Domain (hides business invariants)
- Application/use-case (hides coordination across domain objects)

**Common fake layers**:
- Facades that wrap a single object and delegate 1:1
- "Manager" or "Coordinator" layers that add no logic
- Anti-corruption layers that no longer guard against a changed system
- Generic "handler" layers introduced for symmetry rather than purpose

A modest layered architecture (3–4 layers) with clear purpose is fine. A "clean architecture" implementation with 8 layers where most only rename arguments is lasagna.

---

## 4. Deep vs. shallow modules (Ousterhout test)

**The question**: Is this module pulling its weight?

**The Ousterhout test** (paraphrased from *A Philosophy of Software Design* — Ousterhout frames it as benefit vs. cost, not as a literal equation):

```
Value ≈ Functionality_Hidden − Interface_Complexity
         (benefit the module provides)  (cost users pay to learn it)
```

**Deep** (large positive value): complex internals, simple surface. Hides real complexity from callers. Examples: filesystem, garbage collector, a good pricing engine.

**Shallow** (near-zero or negative value): simple internals, complex-proportional surface. Callers learn almost as much as if the module did not exist. Examples: one-method wrapper classes, pass-through services, classes with setters for every field.

**Action**:
- Deep modules: preserve; these are load-bearing.
- Moderate depth: acceptable if they serve a purpose.
- Shallow modules: either deepen (absorb more responsibility), delete and inline, or merge with a neighbor.

**Counter-example — when a shallow module is justified**: it guards a volatile boundary (third-party SDK) or enables mocking for tests. These have *strategic* depth even if their current implementation is shallow.

---

## 5. Abstract domain primitive vs. raw type?

**The question**: Should `userId` be `string` or `UserId`?

**Introduce a domain primitive when any of these is true**:
- The type participates in domain logic (not just transport).
- It has invariants (format, length, non-emptiness).
- It can be confused with another type of the same representation (`userId` vs `orderId` both as strings).
- Validation logic exists and is currently duplicated at multiple boundaries.

**Keep the raw type when**:
- It is purely transport/serialization data never used in domain logic.
- There is exactly one creation site and one consumption site.
- The ceremony of a domain primitive would exceed its value in this specific place.

**The test**: Can you write `var userId: string = emailAddress` without the type system complaining? If yes, and if `userId` is treated differently from `emailAddress` elsewhere, you have a primitive obsession smell.

---

## 6. When SRP violations matter

**The question**: This class has multiple responsibilities. Does it matter?

**SRP is about rate-of-change, not about line count.** A class with 5 responsibilities that never change is fine. A class with 2 responsibilities that change on different cadences from different teams is a problem.

**Detection**: look at git log for the file.
- Do commits cluster into distinct themes? → divergent change → real SRP violation.
- Do commits share a consistent theme? → multiple methods, one responsibility → not an SRP violation.

**Action priority**:
- **High churn × multiple authors × clustered themes** → P1 split.
- **Low churn regardless of size** → leave it. Splitting static code wastes effort and introduces risk.

---

## 7. SOLID as tool, not dogma

Each SOLID principle is a tradeoff. Apply when the tradeoff favors it:

| Principle | When to apply | When to skip |
|-----------|---------------|--------------|
| **Single Responsibility** | High-churn class that changes for different reasons | Stable class, single theme of change |
| **Open/Closed** | Adding a new variant requires editing N places | Adding variants is rare or types are stable |
| **Liskov Substitution** | Inheritance is used | Composition preferred; LSP only applies where polymorphism lives |
| **Interface Segregation** | Clients forced to depend on methods they don't use | Interface is small and clients use most of it |
| **Dependency Inversion** | High-level policy depends on volatile low-level detail | Dependencies are stable or trivial |

**Reporting rule**: cite the smell, not the principle. "God class" is actionable; "violates SRP" is jargon. Report the observable problem first, then reference the principle for readers who want the theory.

---

## 8. Microservices vs. modular monolith

**The question**: Should this system be split into services?

**Default answer**: Almost always no, at least not first. A modular monolith captures most of the structural benefits with a fraction of the operational cost.

**Prerequisites before extracting a service** (all must hold):
- The module has a clean, stable interface already (proven by the fact that the code compiles with it as a separate package).
- The module has its own data ownership — no shared tables / cross-module queries.
- The latency budget tolerates a network hop.
- There is a real organizational or scaling reason (different team, different deploy cadence, different scaling profile).

**If prerequisites don't hold**: recommend modular monolith first. Extract to service later when prerequisites hold naturally.

**Red flag patterns to avoid**:
- "Distributed monolith": services that must deploy together to work.
- Shared database across services.
- Synchronous N-level call chains between services.

---

## 9. When to rewrite vs. refactor

**Default**: refactor. Rewrites almost always fail or take far longer than projected. The knowledge encoded in existing code (bug fixes, edge cases, performance tuning) is easy to underestimate and hard to re-derive.

**Consider rewrite only when all are true**:
- The current architecture fundamentally cannot deliver the needed capability (not just awkwardly).
- The team has strong characterization of current behavior (tests, spec, or domain experts).
- There is budget and runway for the rewrite to finish plus stabilize.
- The cost of parallel maintenance during the rewrite is accounted for.

**Otherwise**: strangler fig pattern. Build new capability alongside; gradually route traffic to the new thing; retire the old piece by piece. Refactor-in-place for hotspots.

---

## 10. Fitness functions — turning findings into ongoing enforcement

The audit is a one-shot; fitness functions prevent regression. Include at least one suggested fitness function per major finding where automated enforcement is feasible.

**Examples suitable for most projects**:
- **Cycle check**: CI fails if new import cycles appear (tools: `madge`, `dep-tree`, ArchUnit, go-arch-lint, `cargo-deps`).
- **Layer boundary enforcement**: CI fails if a lower layer imports from a higher layer (ArchUnit-style rule).
- **Max fan-in/fan-out**: per-module caps on import count.
- **Max file or function size**: hard cap enforced by lint.
- **Disallowed imports**: specific modules banned from importing others (e.g., domain must not import infrastructure).
- **Public API stability**: exported surface must not shrink without explicit version bump.

A fitness function is better than a style guide because the style guide relies on reviewer vigilance; the fitness function runs every build.

---

## 11. Priority scoring template

Use this scoring when ranking findings. It is deliberately simple — precise scoring is false precision.

```
Pain    = Churn × BlastRadius × DefectCorrelation   (roughly, 1–5 each)
Cost    = RefactorEffort × CoordinationNeeded × RiskOfRegression   (roughly, 1–5 each)
Priority = Pain / Cost
```

- **P1**: Pain ≥ 12 and Cost ≤ 6. Fix now.
- **P2**: Pain ≥ 12 and Cost > 6. Plan as strategic debt.
- **P3**: Pain 6–12 and Cost ≤ 3. Cheap wins; do when nearby.
- **Drop**: everything else.

If two findings tie, prefer the one that unblocks other improvements (keystone debt).

---

## 12. Execution patterns for large-scale refactors

Knowing *what* to change is only half the job. Senior engineers land structural changes safely by choosing an **execution pattern** that keeps the system shippable at every commit. Match the pattern to the shape of the change; mixing patterns or picking the wrong one is how refactors stall or break production.

Prescribe one of these explicitly whenever a finding's refactor spans >1 PR, touches a cross-module interface, or a data schema. "Just refactor it" is not a plan.

### 12.1 Parallel Change (expand–migrate–contract) — Sato

**Use when**: the refactor changes an interface, function signature, or data schema that has multiple consumers — and those consumers cannot be updated atomically with the producer (separate deploys, external clients, DB migrations, published APIs).

**Not when**: single caller in the same deployable; atomic rename the compiler enforces (IDE rename refactor is enough).

**Steps**:
1. **Expand**: add the new form alongside the old. Both work. Producer supports both shapes.
2. **Migrate**: move consumers one by one to the new form. Each migration is its own small PR.
3. **Contract**: once no consumer uses the old form (verify via grep/telemetry), delete it.

**Failure mode / rollback signal**: the contract step never happens — old and new live forever, doubling surface area. If >30 days pass with both forms live and no consumer migrations landing, treat it as stalled: either finish or revert the expand.

**CS grounding**: backwards-compatible evolution preserves Liskov substitutability at the interface boundary across time, not just across subtypes.

### 12.2 Branch by Abstraction — Hammant / Humble

**Use when**: you need to replace a pervasive component (ORM, logger, auth module, core data structure) whose usages are scattered across the codebase and *cannot be carved off at a clean boundary* — so Strangler Fig has nowhere to strangle. Long-running change that must coexist with daily feature work on main.

**Not when**: the legacy component already has a clean seam at a network/process/module boundary (use Strangler Fig — cheaper). Or the change is small enough to ship in one PR.

**Steps**:
1. Introduce an **abstraction layer** over the current implementation. All callers go through it; behavior unchanged.
2. Build the new implementation behind the same abstraction.
3. Flip callers (or a runtime flag) from old to new — incrementally or all at once.
4. Delete the old implementation and, if the abstraction was scaffolding, inline it.

**Failure mode / rollback signal**: the abstraction leaks details of the old implementation (callers pass flags only the old one understands). Means the abstraction was drawn around the *current* shape, not the essential one — stop, redesign the abstraction before adding the new impl.

**CS grounding**: dependency inversion applied temporally — callers depend on a stable interface while the concrete provider is swapped underneath.

### 12.3 Strangler Fig — Fowler

**Use when**: there is a natural boundary to intercept at (HTTP route, message queue, facade, gateway) where traffic can be routed to either old or new. Ideal for replacing whole subsystems, services, or modules with clean external contracts.

**Detection signals it's the right approach**: legacy system has a stable external API but rotten internals; rewrite-in-place blocked by test coverage gaps; business wants continuous value delivery, not a big-bang cutover.

**Not when**: the seam doesn't exist (no proxy-able boundary — use Branch by Abstraction); or the legacy is small enough to characterize and refactor in place.

**Steps**:
1. Put a **router/proxy/facade** in front of the legacy boundary. 100% traffic still goes to legacy.
2. Build new implementation for one slice of functionality.
3. Route that slice's traffic to the new impl (feature flag or router rule). Monitor.
4. Repeat per slice. Retire legacy pieces as traffic hits zero.

**Failure mode — "strangler that never strangles"**: after 12+ months, legacy still handles >50% of traffic and new slices have stopped landing. Causes: no retirement deadline, new system accumulating its own debt, org lost interest. Rollback signal: if no slice has migrated in the last quarter, escalate — either commit resources to finish or declare legacy the system of record and stop the migration.

**CS grounding**: incremental substitution preserving external contracts; each slice is an independently verifiable refactoring step (Fowler's correctness-preserving transformation, scaled to subsystems).

### 12.4 Feature flag / dark launch / shadow traffic

**Use when**: the refactor changes behavior in a code path where "wrong output" has production consequences (pricing, auth, ranking, billing). You need to verify equivalence before cutover and retain instant rollback.

**Three intensities, pick one**:
- **Feature flag cutover**: new path behind a flag; progressive rollout (1% → 10% → 50% → 100%); kill switch on error-rate delta.
- **Dark launch / shadow**: run new implementation in parallel on live inputs; discard its output; compare against old's output asynchronously. No user impact if the new path crashes.
- **Diff testing (Scientist-style, GitHub)**: both paths run, old's result is returned, new's result is logged with a diff. Cutover when diff rate is acceptable.

**Not when**: pure internal refactor with compiler-verified equivalence; cost of running both paths exceeds the risk.

**Failure mode / rollback signal**: diff rate plateaus above 0 and the remaining diffs turn out to be pre-existing bugs the old path had. Decide explicitly: port the bugs, or fix them and accept a behavior change — do not silently cut over with unexplained diffs.

**CS grounding**: observational equivalence verified empirically on production distribution, not just on test inputs — Hyrum's Law in reverse (you learn which observable behaviors consumers actually depend on).

### 12.5 Sprout Method / Sprout Class — Feathers

**Use when**: you need to *add* behavior to a legacy function that has no tests and you cannot safely refactor it. Adding the new logic inline would entangle it with untested code.

**Not when**: the legacy function already has characterization tests (just edit it); or the new behavior is genuinely part of the old function's responsibility.

**Steps**:
1. Write the new behavior as a **new, fully-tested** function (sprout method) or class (sprout class).
2. Insert a single call to the sprout from the legacy function. That one line is the only untested change.
3. Leave the legacy function otherwise untouched. Characterize and refactor it later, separately.

**Failure mode / rollback signal**: sprout needs data only accessible via private state of the legacy function — you end up widening the legacy's interface to feed the sprout. That means the seam is wrong; reconsider as Extract-and-Override instead.

**CS grounding**: Feathers — minimize the edit distance to untested code. Every line added to an untested function is a line that may never be tested.

### 12.6 Seam techniques beyond subclass-and-override — Feathers

When the catalog says "introduce a seam," these are the specific moves, ordered from least to most invasive:

- **Preprocessing seam**: intercept at the build/text-transform layer (macros, codegen, bytecode rewrite). Use when source cannot be touched (vendored, generated).
- **Link seam**: swap the implementation at link/module-resolution time (replace a library, shim a module path, use `LD_PRELOAD` / Node's module resolution / Python import hooks). Use for third-party deps with no injection point.
- **Extract-and-override factory**: the legacy class constructs a collaborator internally; extract that `new X()` into a protected factory method; subclass in tests to override. Minimal surface change.
- **Extract-and-override call**: the legacy method calls a specific dependency inline; extract the call into its own method; override in a test subclass. Use when you cannot change the constructor.
- **Subclass-and-override**: classic — make the hard-to-test method `protected virtual`, subclass to stub it. Last resort; couples tests to inheritance shape.

**Choosing rule**: prefer the least invasive seam that gives you a test. Every seam is technical debt — scaffolding to be removed once proper dependency injection replaces it.

**Failure mode / rollback signal**: seams multiplying without any being removed. Seams are temporary; if the codebase accumulates a permanent population of extract-and-override factories, the real fix (constructor injection, module boundary) is being avoided.

**CS grounding**: Feathers — a seam is a place you can alter behavior without editing in place. It is the minimum-surgery entry point for getting untested code under test.

### 12.7 Boy Scout opportunistic refactor vs. dedicated refactor PR

**Question**: it's a small cleanup touching code you're already in. Fold it into the feature PR, or make it its own?

**Fold in (Boy Scout)** when: change is <30 LOC, mechanically obvious (rename, extract small method, delete dead branch), touches only files already in the diff, and does not alter any external signature. Reviewer can verify it at a glance.

**Separate PR** when any of: touches files the feature doesn't; changes a public signature; moves code between modules; renames across >3 files; needs its own characterization tests; bisect-hostile (would make future `git bisect` confusing by mixing refactor with behavior change).

**Rule of thumb**: if a reviewer would need to context-switch between "is the feature correct?" and "is the refactor correct?", split the PR. Mixed PRs are where regressions hide — reviewers' attention is finite and splits between the two topics.

**CS grounding**: Fowler — never mix refactoring (behavior-preserving) and feature work (behavior-changing) in the same commit. Separate commits at minimum, separate PRs when the refactor is non-trivial.

### 12.8 When big-bang is actually correct

Rare, but real. Big-bang refactor (one PR, everything changes) is correct when **all** hold:

- The change is mechanically uniform (codemod, compiler-enforced rename, type-system migration) — human judgment is not required per site.
- Partial state is worse than either endpoint (e.g., half-migrated type hierarchy that doesn't compile; half-renamed symbol that breaks grep).
- The entire change fits in one reviewer's head or is verifiable by the compiler / a codemod's test suite.
- No concurrent feature work on the same files during the migration window (or a short freeze is acceptable).

**Prefer big-bang for**: language/framework upgrades with automated codemods; type-system-wide renames; monorepo-wide import path changes; any change where the intermediate state is invalid.

**Rollback signal**: a big-bang PR that sits in review for >3 days is failing — it's too large to review, conflicts will accumulate, and trust degrades. Abort and convert to Parallel Change or codemod-with-shim.

**CS grounding**: atomicity. Some transformations have no valid intermediate state; serializing them across multiple commits introduces a window of incoherence worse than the brief disruption of an atomic cut.

---

**Choosing between patterns (quick map)**:

| Shape of change | Pattern |
|---|---|
| Interface/schema with multiple consumers | Parallel Change (§12.1) |
| Pervasive component, no clean boundary | Branch by Abstraction (§12.2) |
| Subsystem with clean external API | Strangler Fig (§12.3) |
| Behavior change with production risk | Feature flag / dark launch (§12.4) |
| Add behavior to untested legacy | Sprout (§12.5) |
| Get untested code under test | Seam techniques (§12.6) |
| Small cleanup near feature work | Boy Scout vs. separate PR (§12.7) |
| Mechanically uniform, no valid intermediate | Big-bang (§12.8) |

Patterns compose: a Strangler Fig slice cutover is usually gated by a feature flag; Branch by Abstraction's final switch can be a Parallel Change at the abstraction's call sites; getting legacy under test (seams) is the prerequisite for any of the above.
