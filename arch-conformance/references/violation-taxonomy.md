# Violation Taxonomy — Detection Patterns and Confidence Calibration

The 12 categories below cover ~95% of conformance violations in real codebases. For each: what it is, the grep/glob pattern that detects it, the false-positive trap, and the confidence calibration.

When dispatching Phase 3 sub-agents, copy the relevant detection pattern into the agent's prompt so it knows where to start searching.

---

## Confidence calibration (used everywhere)

- **HIGH** — clear violation, no plausible justification, the violated rule is **stated** (in docs/ADR/CLAUDE.md), and the cited line clearly does the forbidden thing. Spot-checking will confirm it.
- **MEDIUM** — clear violation, but either (a) the rule is **inferred** from naming/structure rather than stated, or (b) there is a plausible benign reason (DI wiring, test fixtures, the file is a known integration shim).
- **LOW** — pattern fragmentation, naming inconsistency, or stylistic deviation that *might* be intentional. Report as a category aggregate, not as individual findings, unless the user has stated they want LOW noise too.

A finding without a confidence label is not a finding. Refuse to write it.

---

## Category V-LAY — Layer violations

**Definition:** A file in layer A imports from layer B in violation of the declared dependency direction. Most common form: outer layer imports outer layer (lateral when not allowed), or inner layer imports outer layer (reverse).

**Detection:**
- For each pair of architectural layers `(A → B)` where `A` should not depend on `B`:
  - `grep -rn "from .*<B-path>" <A-path>/` (Python/TS)
  - `grep -rn "import .*<B-package>" <A-path>/` (Java/Go/Rust)
- Also catch transitive: `<A>` imports a re-exporter that re-exports from `<B>`. Look for barrel files (`index.ts`, `mod.rs`, `__init__.py`) under `<A>` that pull in `<B>` symbols.

**False-positive traps:**
- Type-only imports (TypeScript `import type`) sometimes don't violate — confirm with the user whether type-only crossings are allowed.
- DI / composition root files (`main.*`, `wire.go`, `Module.kt`, `composition_root.*`) intentionally cross all layers. Exclude these by path.
- Tests for the inner layer often need to import outer-layer mocks — exclude `*test*`, `*spec*`, `*__tests__*` paths from this check unless the user wants test conformance too.

**Confidence:**
- HIGH if rule was stated AND the import is from production code AND the path isn't a known wiring file.
- MEDIUM if the rule is inferred OR the violating file might be a shim.

---

## Category V-CYC — Cyclic dependencies

**Definition:** Module A transitively depends on B, and B transitively depends on A. Includes 2-cycles (direct mutual import) and longer chains.

**Detection:**
- For directly mutual: pair-wise grep. `grep -rln "<B>" <A>/` and `grep -rln "<A>" <B>/` — both non-empty means a 2-cycle.
- For longer cycles: if a real dependency-graph tool is available in the project (madge, dep-cruiser, jdeps, go mod graph, cargo-deps, pydeps), prefer running it via Bash. Parse output for cycles.
- If no tooling: build a coarse adjacency map by sampling top-level imports per module, then run a quick Python script to find SCCs (Tarjan). Bundle one in `scripts/` if you find yourself doing this often — but for one-off use, an in-context script is fine.

**False-positive traps:**
- Type-only cycles in TypeScript can be benign (compile away).
- `index.ts` barrels create apparent cycles that vanish when you look at the actual exports used. Re-resolve through the barrel before reporting.

**Confidence:**
- HIGH for 2-cycles between named architectural units.
- MEDIUM for longer cycles or cycles only visible through barrel files.

---

## Category V-ENC — Bypassed encapsulation / boundary

**Definition:** Module A imports an *internal* file of module B instead of going through B's public API. E.g., `import { _internalHelper } from 'pkg-b/src/internals/util'` instead of `import { helper } from 'pkg-b'`.

**Detection:**
- Imports that reach into a sibling package's `src/` or `internal/` (Go) or path containing `/private/`, `/_internal/`, `/impl/`.
- TypeScript: imports of `.../dist/...` or paths past the package's main barrel.
- Go: any import path containing `/internal/` from outside the parent of `internal/` (this is actually compile-enforced in Go, so violations require build-tag tricks).
- Imports that bypass an explicit `package.json` `exports` field (Node 12+ subpath exports).

**False-positive traps:**
- Test code legitimately reaches into internals.
- The package being imported may not have a defined public API — in that case, this is not a violation, this is a missing public API (note as **absence**, not divergence).

**Confidence:**
- HIGH if package has explicit `exports`, `__all__`, `pub use`, or `index.ts` barrel and the import bypasses it.
- MEDIUM if encapsulation is implicit (convention only).

---

## Category V-LEAK — Leaked cross-cutting concerns

**Definition:** A concern that should live at one architectural seam appears scattered through unrelated layers. Common offenders: logging, persistence, authentication, transport (HTTP/WebSocket), serialization, telemetry.

**Detection (per concern):**
- Logging in domain: grep for logger imports/usages inside the domain layer. Domain code should rarely log; logging is an outer-layer concern.
- HTTP in domain/application: grep for `fetch`, `axios`, `requests`, `http.Client`, `reqwest::Client`, `RestTemplate`, `HttpClient` inside layers that should not do I/O.
- ORM in domain: grep for `@Entity`, `@Column`, `@Table`, `models.Model`, `Base.metadata`, `sqlx`, `diesel::table` inside the domain layer.
- Auth in business logic: grep for token validation, role checks, `JWT`, `requireRole`, `currentUser` inside use cases or domain methods that should be auth-agnostic.

**False-positive traps:**
- Domain *interfaces* may type-reference logger or persistence types via dependency injection — the import is OK, the concrete dep is what matters.
- Some teams deliberately put structured logging primitives in the domain (e.g., `DomainEvent.log()`). If consistent across the codebase, treat as the architecture, not a violation.

**Confidence:**
- HIGH for ORM annotations on domain entities (almost never intentional).
- HIGH for raw HTTP clients in domain/application code.
- MEDIUM for logger usage in domain — depends on team convention.

---

## Category V-FRAG — Pattern fragmentation

**Definition:** The same problem solved multiple ways across the codebase. Most common: multiple HTTP clients, multiple state-management approaches, multiple validation libraries, multiple error-handling patterns coexisting.

**Detection:**
- Count distinct HTTP libraries imported anywhere: `grep -rno "from 'axios'\|from 'node-fetch'\|fetch(" --include='*.ts'` etc., then count uniques.
- Count distinct date/time libs (`moment`, `date-fns`, `dayjs`, `luxon`) — having more than one is a strong signal.
- Count distinct ORM/query approaches in the same service (e.g., raw SQL + Prisma + Knex coexisting).
- Distinct test frameworks in the same package.
- Distinct config-loading patterns (env vars read in 4 different ways).

**False-positive traps:**
- A migration in progress is a legitimate reason for two patterns coexisting. Look for an ADR / `MIGRATION.md` / TODO before flagging.
- Test code may legitimately use a different approach than production.

**Confidence:**
- MEDIUM by default — almost all FRAG findings need user confirmation that they're not in-flight migrations.
- LOW for stylistic-only fragmentation (e.g., two different `.then()` vs `await` styles).

---

## Category V-STATE — Shared mutable state across boundaries

**Definition:** Mutable state (a singleton, a global, an in-memory cache, a module-level variable) that is written by code in multiple architectural units without explicit coordination.

**Detection:**
- Find module-level mutable bindings: `let x = ...` (top-level, JS/TS), `var X = ...` (Go top-level), class-level `mut` static fields (Rust/Java), Python module-level lists/dicts.
- For each, grep for assignments (`x = `, `x.push`, `x[...] =`, `x.set`) across the codebase. If writers span more than one architectural unit, flag.
- Singletons: classes that vend an instance via `getInstance()`, `default` export, lazy init. Trace consumers and writers.

**False-positive traps:**
- DI containers and registries are intentional shared mutable state — they are usually fine if scoped to startup.
- Caches whose only writer is inside one module are fine even if many readers exist.

**Confidence:**
- HIGH if writers truly span layers (e.g., domain code mutating a UI cache).
- MEDIUM if writers are within one layer but cross feature boundaries.

---

## Category V-NAME — Naming-vs-behavior mismatch

**Definition:** A file/class/function whose name implies an architectural role it does not actually fulfill. `UserService` that talks to the DB directly. `UserRepository` that contains business rules. `OrderController` that manages state. These betray architectural intent and confuse future readers.

**Detection:**
- For each suffix-based naming convention found in Phase 1 (Repository, Service, Controller, UseCase, etc.):
  - List all files matching the suffix.
  - For each, scan for imports that contradict the role:
    - `*Repository` should not import HTTP clients or business-rule helpers.
    - `*Service` should not import the raw DB driver or HTTP request/response objects.
    - `*Controller` should not contain conditional business logic; it dispatches.
    - `*UseCase` / `*Interactor` should not import frameworks or transports.

**False-positive traps:**
- Some teams use these suffixes for a different purpose than the canonical one (e.g., "Service" meaning "long-running background process"). Phase 1 should have caught this — re-check the dominant convention.

**Confidence:**
- HIGH if the codebase has a stated convention and this file violates it.
- MEDIUM if convention is inferred.

---

## Category V-DUP — Mutant duplicates

**Definition:** Two or more code paths that solve the same problem with small variations. Often the result of copy-paste across architectural boundaries that should have led to a shared abstraction. Distinct from generic duplication: the *boundary crossing* is what makes it an architectural concern.

**Detection:**
- Look for functions across modules with the same name and similar signatures: `grep -rn "function processOrder\|def process_order\|fn process_order"`.
- Suspect any utility function defined ≥3 times in different directories.
- Schema/DTO duplication: the same shape defined in multiple places — `interface User {...}` repeated with minor variations.
- Validation rule duplication: the same regex or check repeated across layers.

**False-positive traps:**
- Microservices intentionally duplicate types to maintain independence. Inside a single deployable, that's a smell; across services, it can be deliberate.
- Generated code (gRPC stubs, OpenAPI clients) will look duplicated — exclude generated paths.

**Confidence:**
- MEDIUM by default — duplicates need a human to judge whether to consolidate.

---

## Category V-WIRE — Composition leaks

**Definition:** Object construction (especially of adapters/infrastructure) happens inside business logic instead of at the composition root. Symptoms: business code calls `new HttpClient()`, `new PostgresRepo()`, `dynamodb.connect()` inline.

**Detection:**
- For each known infrastructure class/factory, grep for `new` / `()` call sites.
- Any call site outside the wiring file (`main.*`, `composition_root.*`, `wire.go`, DI module file) is a candidate violation.

**False-positive traps:**
- Tests legitimately construct infrastructure for fixtures. Exclude test paths.
- Some patterns (factories) intentionally construct inside business logic — accept if consistent.

**Confidence:**
- HIGH if the codebase has DI tooling registered (Spring, Nest, Guice, Wire, Dagger) and the construction bypasses it.
- MEDIUM otherwise.

---

## Category V-CONTRACT — Cross-context / cross-service contract leaks

**Definition (DDD/microservices):** One bounded context or service reaches into another's internal types or storage. E.g., `billing/` imports `shipping/internal/Address`, or service A's code reads service B's database tables.

**Detection:**
- Map top-level contexts/services from Phase 1.
- For each pair, grep for cross-context imports outside any explicit `<shared-kernel>/`, `contracts/`, or `events/` directory.
- Search for raw DB credentials/connection strings in code: any service connecting to multiple schemas/databases is a candidate.

**False-positive traps:**
- A "shared kernel" directory is legitimate. Distinguish it from a god module by checking that it's small and contains only types, not behavior.

**Confidence:**
- HIGH for direct cross-service DB access.
- HIGH for source-code imports across service boundaries (when service boundary is real, not just folder).
- MEDIUM if the boundary is "feature folders" rather than deployable services.

---

## Category V-EVENT — Event/handler symmetry breaks (CQRS / event-sourced only)

**Definition:** Command handlers that return data; query handlers that mutate; events without consumers; consumers without producers; handlers that bypass the bus.

**Detection:**
- For each `*CommandHandler*` / `Handle(*Command)`, check return type — should be void / Result-only.
- For each `*QueryHandler*`, scan body for SQL `INSERT/UPDATE/DELETE`, ORM `.save()`, repo `.add()`.
- List all event types; for each, grep for both producers (`emit`, `dispatch`, `publish`) and subscribers (`@Handles`, `On(...)`, `Subscribe`). Flag orphan events and unconsumed events.
- Look for repository writes outside command handlers (state mutations bypassing the command path).

**Confidence:**
- HIGH for query handlers with INSERT/UPDATE.
- HIGH for orphan event types when an event-driven architecture is stated.

---

## Category V-MIG — Drift artifacts (incomplete migrations)

**Definition:** Two coexisting implementations of the same architectural concern, where one was supposed to replace the other but the migration stalled. Distinct from V-FRAG: here we have evidence (commit history, MIGRATION.md, deprecated annotations) that one is the intended target.

**Detection:**
- Search for `@Deprecated`, `@deprecated`, `// DEPRECATED`, `# TODO: remove after migration`.
- Look for files named `*-old.*`, `*-legacy.*`, `*-v1.*` alongside `*-new.*`, `*-v2.*`.
- Check git log for "migrate to X" commit messages and verify the migration completed.
- Look for `MIGRATION.md`, `MIGRATING.md`, ADRs about migrations and check whether the work landed.

**Confidence:**
- HIGH when there is an explicit deprecation marker still in use by production code.
- MEDIUM when only inferred from naming or commit history.

---

## Anti-categories — do NOT report under arch-conformance

These belong to other skills. If sub-agents return findings of these types, drop them and tell the user the right skill:

- **Code style / formatting** — not architecture. Drop.
- **Performance** — unless caused by a layer violation, drop. Use `app-harden`.
- **Security vulnerabilities** — drop. Use `app-harden` or security-review skill.
- **Generic complexity** ("this function is too long") — drop. Use `arch-audit` or `code-scrutiny`.
- **Single-file bugs** — drop. Use `bug-hunt`.
- **Refactoring suggestions unrelated to a rule** — drop. Use `refactor-pro`.

If a category produces only anti-category findings, the rule produced zero real conformance violations — which is a clean result.
