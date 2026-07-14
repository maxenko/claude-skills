# Smell Catalog

Detection recipes for architectural smells. Load during Phase 3 of the audit. Each entry gives the signal (with concrete thresholds), the CS principle behind why it matters, and the canonical refactor move.

Organized by the four lenses: **Coupling**, **Cohesion & Complexity**, **Abstraction Quality**, **Evolution Signals**. A **paradigm adaptations** section at the end translates OOP-centric smells to Rust/Go, functional, frontend, and data-pipeline codebases.

Thresholds are defaults. Adjust by language convention and project size, but do not discard — "large" without a number is not a finding.

## Contents

- **[Lens 1: Coupling](#lens-1-coupling)** — cyclic dependency · strong connascence across boundaries · god component · shared-DB / data coupling · multi-writer mutable state · missing single-writer boundary (actor / agent / mailbox processor) · synchronous call chain that should be async · dual-write without outbox · missing idempotency boundary · implicit long-running workflow / missing saga · event bus as hidden dependency graph · unstable dependency · inappropriate intimacy · message chains · middle man
- **[Lens 2: Cohesion and complexity](#lens-2-cohesion-and-complexity)** — god class · long method · long parameter list · shallow module · lasagna / excessive layering · divergent change · feature envy · data class
- **[Lens 3: Abstraction quality](#lens-3-abstraction-quality)** — anemic domain model · primitive obsession · unenforced invariants / smart constructors · illegal states representable · shotgun parsing · missing aggregate · complected concerns · functional core / imperative shell absent · unidirectional data flow absent · scattered error handling / try-catch pyramid · observability complected · missing domain events · leaky abstraction · domain depends on infrastructure / missing ports · missing anti-corruption layer · missing bounded context · speculative generality · wrong abstraction · temporal coupling · implicit state machine / scattered entity state · event sourcing gap · CQRS gap · implicit dataflow pipeline
- **[Lens 4: Evolution signals](#lens-4-evolution-signals)** — hotspot · shotgun surgery · parallel inheritance / parallel switch · fossil / dead code · ownership diffusion · temporal hotspot mismatch
- **[Data ingestion & serving subsystem](#data-ingestion--serving-subsystem-cross-cutting)** (cross-cutting) — no ingest/serve boundary · load-all (no streaming) · sequential parallelizable ingest · no fault tolerance / not resumable / not idempotent · no backpressure / unbounded buffering · blocking UI on latency-bound I/O · no lifecycle/progress events for the UI · observability not designed into the boundary · **the canonical robust ingest→serve abstraction**
- **[Paradigm adaptations](#paradigm-adaptations)** — Rust/Go · Functional (Haskell/Elixir/Clojure/F#) · Frontend (React/Vue/Svelte) · Data pipelines (Airflow/dbt/Spark)
- **[Smells deliberately NOT on this list](#smells-deliberately-not-on-this-list)**

---

## Lens 1: Coupling

### Cyclic dependency
**Signal**: Module A imports B (directly or transitively), and B imports A. Grep imports; build a directed graph. Any cycle is worth investigating.
**Threshold**: any cycle. Classification of fatal vs. tolerable is in `decision-frameworks.md` §2.
**Why it matters**: Acyclic Dependencies Principle. A cycle means both modules must be understood, built, tested, and reasoned about as one unit — the named boundary is a lie. Empirically, components in cycles are more defect-prone (Al-Mutawa, Dietrich et al., "On the Shape of Circular Dependencies in Java Programs").
**Refactor**: Invert one dependency via an interface owned by the upstream module (DIP), extract the shared concept into a third module both depend on, or merge if the modules are genuinely one concept.

### Strong connascence across boundaries
**Signal**: Two modules must agree on positional argument order, mutable state, execution timing, or identity — and the modules live in different top-level packages (path prefix differs beyond `src/`).
**Ranking (Page-Jones), weakest → strongest** (from connascence.io):

Static (compile-time):
1. Name (CoN)
2. Type (CoT)
3. Meaning / Convention (CoM) — e.g., `status == 1 means active`
4. Position (CoP) — argument order
5. Algorithm (CoA) — must implement the same algorithm

Dynamic (run-time, strictly stronger than any static form):
6. Execution (CoE)
7. Timing (CoTi)
8. Value (CoV)
9. Identity (CoI)

**Rule**: Strength × Degree × Locality. Strong connascence is tolerable within a class; across a package boundary it is catastrophic.
**Refactor**: Convert CoP → CoN (keyword args, config object). Convert CoM → CoT (replace magic values with enums). Convert CoA → CoN by extracting a shared function. Push strong connascence into a single module.

### God component / high afferent coupling
**Signal**: Module imported by an unusually large share of the codebase.
**Thresholds**: imported by ≥20 distinct files OR by >10% of all source files, whichever is smaller for the project size. Cross-reference with module LOC.
**Detection**: `grep -rn "from <module>" | cut -d: -f1 | sort -u | wc -l`.
**Why it matters**: Changes have uncontrolled blast radius; the module cannot evolve without coordinated changes everywhere.
**Refactor**: Split by cohesion — which callers use which subset? Extract those subsets into their own modules. The original often becomes a thin facade or disappears.

### Shared-DB / data coupling
**Signal**: One database table has writers in multiple modules; cross-module JOINs or views that enforce a hidden contract; transactions spanning module boundaries.
**Detection**: grep for the table name across `*.sql`, ORM models, repositories. Count distinct writers. Check for cross-module `JOIN` in raw SQL.
**Why it matters**: Shared-DB coupling is often stronger than any import-graph coupling. Two modules that write the same table share an invariant that is enforced nowhere in code.
**Refactor**: Assign single-writer ownership. Other modules query via an API or read-only view. If that's infeasible, make the shared contract explicit (a `TablePolicy` class that all writers go through).

### Multi-writer mutable state
**Signal**: Global variables, module-level mutables, singletons, caches with multiple writers and no locking/coordination. Grep for `static mut`, top-level `let`/`var`, `@singleton`, unprotected dict/map assignments at module scope.
**Why it matters**: Hickey — state intertwines value with time; multi-writer state intertwines everything that touches it. A latent concurrency bug for any non-trivial system.
**Refactor**: Identify the single owner. Other modules go through its API. If the state genuinely has multiple writers, promote it to a coordinated owner — see the next entry (**Missing single-writer boundary**) for the actor / agent / `MailboxProcessor` / `GenServer` refactor, or fall back to a DB/queue for cross-process state. Do not leave it as shared memory with ad-hoc locks.

### Missing single-writer boundary (actor / agent / mailbox processor)
**Signal**: Shared mutable state is protected by fine-grained locks instead of owned by a single task that serializes access through a message queue. Two shapes — both are the same smell:

**(a) Per-entity / partitioned** — a per-key resource (per-user cart, per-order state, per-session ticker, per-account balance) is serialized by a global lock or striped-lock table, creating contention between unrelated keys.

**(b) Singleton coordinator** — a cross-cutting resource (rate limiter, connection pool, scheduler, cache manager, event dispatcher, throttle, de-duper) uses fine-grained locks, `volatile` flags, `AtomicReference`, concurrent maps, and check-then-act patterns — when one task owning the state and receiving commands by message would be far simpler. This is the case F#'s `MailboxProcessor<T>` and Elixir's `GenServer` were designed for.

**Detection**: grep for `Lock`, `Mutex`, `RwLock`, `synchronized`, `asyncio.Lock`, `AtomicReference`, `ConcurrentHashMap`, striped-lock arithmetic, `SELECT ... FOR UPDATE` + retry, distributed-lock libraries. Check-then-act patterns: `if (state == X) { ...; state = Y; }` with locks around them. Bug history: "lost update", "double-spend", "stale read", "rate limiter let too many through", "pool allocated twice".
**Threshold**: ≥3 coordinated critical sections on the same resource; OR striped-lock arithmetic in domain code; OR a single coordinator object touched from ≥3 files with any mutex.

**Why it matters**: LMAX single-writer principle (Martin Thompson) — contention-free throughput requires one writer per piece of state. The actor/agent model (Hewitt, Agha) names the same boundary. One mailbox + one worker eliminates whole classes of bugs by construction: lost updates, lock-ordering deadlocks, visibility errors, check-then-act races. Virtual actors (Orleans, Cloudflare Durable Objects) extend (a) across a cluster; OTP `GenServer` and F# `MailboxProcessor` are the canonical (b).

**Refactor**:
- **Partitioned case (a)**: identify the key (user_id, order_id, symbol); route work for key `k` to a dedicated owner keyed by `k` — actor, goroutine-per-key with channel, Kafka-partition consumer, virtual actor. The owner's mailbox replaces the lock.
- **Singleton case (b)**: introduce a single long-lived task owning the state, a typed message enum, and a command queue. Callers send messages (fire-and-forget) or send + await replies (request/response). No locks inside — only one task ever touches the state. Canonical idioms per language:
  - F#: `MailboxProcessor<Msg>` (canonical; see garydwatson/mailbox_processor for language-portable patterns)
  - Elixir / Erlang: `GenServer` / `gen_server` (OTP)
  - Akka / Akka.NET: singleton typed `Actor`
  - Rust: `tokio::spawn` consuming from `mpsc::Receiver<Msg>`
  - Go: a goroutine consuming from `chan Msg`
  - Clojure: `agent` (fire-and-forget) or a `core.async` `go` loop
  - Python: `asyncio.Queue` + consumer task
  - C# / TypeScript: `BufferBlock<Msg>` / async generator loop over a queue

**Complement — supervisor / "let it crash"** (OTP): once state is owned by a single task, add a supervisor that restarts it from a clean initial state on unhandled error. The actor model's real power is *serialization + failure isolation together* — a single-writer without supervision is half the pattern. Languages outside BEAM approximate it with framework supervisors (Akka, Orleans) or restart-on-panic patterns (tokio task monitoring, goroutine recovery).

**Alternative when serialization is unnecessary — immutable snapshot + atomic swap**: for read-heavy state where writers are rare, replace `RwLock<HashMap>` with `AtomicReference<ImmutableMap>` (or `ArcSwap` in Rust, persistent data structures in Clojure). Writers build a new snapshot and swap; readers always see a consistent value with no locks. Simpler than an actor when ordering of writes doesn't matter.

**Overkill**: low-throughput cross-entity aggregation; request-scoped state; truly lock-free structures (Disruptor, ConcurrentHashMap read-heavy) where benchmarks justify the complexity. Don't convert simple config reads into actors.

### Synchronous call chain that should be an async message
**Signal**: A request-handling path makes ≥4 synchronous cross-module or cross-service calls in series; each layer has its own retry/timeout; P99 latency is the sum of all downstream latencies plus retries.
**Detection**: grep for HTTP/RPC clients inside request handlers across ≥3 modules in one flow; `@Retryable`/`retry` annotations at multiple layers of the same call stack. Symptoms: cascading timeouts, retry amplification (N× load on downstream during incidents), "everything is slow when X is slow".
**Threshold**: ≥4 sync hops in a user-facing flow; OR retry logic at ≥2 layers of the same stack; OR P99 dominated by one slow downstream.
**Why it matters**: Synchronous chains complect request lifecycle with downstream availability; every slow dependency degrades the whole chain; retries multiply load. Async messaging decouples lifecycles and provides natural backpressure.
**Refactor**: Identify the minimal synchronous core (what the user must wait for). Move everything else behind a durable queue or bus; the handler returns after enqueuing. Consolidate retry policy to one layer (the consumer). Make backpressure explicit with a bounded queue + overflow policy (block, drop-oldest, reject).
**Overkill**: strongly-consistent read-your-writes flows (auth, checkout commit) where the user actually needs the downstream result. Fix fan-out with batching/caching instead.

### Dual-write without outbox (transactional messaging gap)
**Signal**: A handler writes to the DB and then publishes a message/event/webhook/email in the same function — without a shared transaction or an outbox table. If the publish succeeds and the DB rolls back, or the DB commits and the publisher crashes, the two systems diverge. No retry at the call site can fix this.
**Detection**: grep for `publish(`, `send(`, `emit(`, `kafka.send`, `sns.publish`, `http.post` appearing in the same function as `db.commit`, `tx.commit`, `repository.save`. Also: `@Transactional` methods calling external services inside the transaction.
**Threshold**: ≥2 sites doing DB write + external publish in the same call path, or any such site in a hot business flow (orders, payments, signup).
**Why it matters**: Atomic commit across heterogeneous resources is impossible without XA/2PC. The outbox pattern reduces it to a single-resource transaction plus at-least-once redelivery (Kleppmann, *Designing Data-Intensive Applications*, ch. 11).
**Refactor**: Add an `outbox` table in the same DB; insert the event row in the **same transaction** as the domain write. Run a relay (CDC via Debezium, or a polling worker) that reads outbox rows and publishes downstream, marking rows sent. Make consumers idempotent. Delete inline `publish()` calls.
**Overkill**: best-effort fire-and-forget (analytics pings, non-critical audit). Document the accepted loss.

### Missing idempotency boundary
**Signal**: Handlers for retryable inputs (webhooks, message-queue consumers, HTTP POST with client retries, job runners) perform side effects with no dedupe key. Bug history: "duplicate charge", "order processed twice", "email sent N times after redeploy".
**Detection**: grep for message-consumer entry points (`@KafkaListener`, `onMessage`, `@SqsListener`, webhook routes) and check whether the first lines compute or look up an idempotency key. Count side-effecting handlers without a `processed_messages` table or unique-constraint-backed dedupe.
**Threshold**: any message-driven consumer or webhook endpoint that mutates state AND lacks an idempotency key check.
**Why it matters**: All real messaging is at-least-once; exactly-once *delivery* is unachievable, so exactly-once *effect* must be built in the consumer (Kleppmann; Morling, *On Idempotency Keys*).
**Refactor**: Identify the natural idempotency key (client-supplied `Idempotency-Key` header, message ID, `(aggregate_id, version)`, business key). Add a `processed_messages` table with a unique constraint; insert inside the business transaction. On duplicate-key violation, short-circuit and return the stored prior response. Bound the table via TTL.
**Overkill**: read-only endpoints; operations that are naturally idempotent (`SET x=5`).

### Implicit long-running workflow / missing saga or process manager
**Signal**: A business process spans services, storage systems, or time windows (checkout, onboarding, subscription renewal, refund, multi-step provisioning) — implemented as a chain of direct calls, status columns, cron sweeps scanning for "stuck" rows, and ad-hoc rollback/compensation helpers. When a step fails mid-flow, state wedges or requires manual fix-up.
**Detection**: grep for status columns like `PENDING_VERIFICATION`, `AWAITING_SHIPMENT`, `RETRY_SCHEDULED`; cron jobs that `SELECT ... WHERE status = X AND updated_at < NOW() - INTERVAL`; functions that perform ≥3 writes across distinct services/repositories with no transaction envelope and only happy-path logic; compensation helpers (`refund`, `release`, `undoX`) called from catch blocks.
**Threshold**: ≥3 cross-boundary state writes in one logical operation with no explicit workflow owner; OR ≥2 distinct sweep jobs on the same entity; OR bug reports of "stuck in state X for N days".
**Why it matters**: Workflows are first-class concepts (van der Aalst, workflow patterns). Encoding a workflow as status-poll + ad-hoc rollback is the recovery-by-sweep anti-pattern; failures drop compensations and leave inconsistencies. Sagas (Garcia-Molina & Salem, 1987) make each step's compensation first-class; durable-execution engines (Temporal, Cadence, Step Functions) persist call stacks so long-running `await` is crash-safe.
**Refactor**: Name the workflow; write it as one function with explicit `await` points and compensations per step. Choose **orchestration** (central workflow engine — Temporal, Step Functions, or a DB-backed state machine) for flows with branching; choose **choreography** (events + local handlers) for linear fan-out. Each step must be idempotent. Replace sweep jobs with workflow timers; replace status columns with workflow state.
**Overkill**: 2-step flows or single-DB writes — use a transaction. Flows where a nightly reconciler is acceptable — document the reconciler.

### Event bus as hidden global dependency graph
**Signal**: A publish-subscribe bus where tracing a feature requires grepping event-type names across the repo; events named after DB rows (`UserUpdated`, `OrderRowChanged`) rather than business facts (`OrderConfirmed`); consumers reach into payload fields that match the producer's DB columns; request/response is implemented as two events on the bus.
**Detection**: event-type string constants used in ≥N files with no central schema registry; subscribers depending on payload shape that mirrors producer table columns; paired events that look like synchronous request/response; features requiring cross-repo text search to trace.
**Threshold**: ≥5 event types with no schema registry; OR any event named after a table + CRUD verb; OR any "ack" event pairing implementing sync RPC.
**Why it matters**: Pub/sub trades explicit imports for semantic coupling that is harder to audit — the compiler sees no dependency where one exists in reality. Leaking DB schema as event shape promotes internal state to a public contract, locking the producer. Connascence of Meaning across module boundaries is just as strong as across function calls.
**Refactor**: Publish business facts, not row changes (`OrderConfirmed`, not `OrdersTableUpdated`). Version events; define schemas in a shared contract (Protobuf, Avro, JSON Schema) owned by the producer. Replace request/response-on-bus with direct RPC — the bus is for broadcast, not point-to-point. Maintain a subscriber registry so the effective dependency graph is grep-able.
**Overkill**: truly broadcast signals (metrics, audit, cache invalidation); schema ceremony is overhead there.

### Unstable dependency (SDP violation)
**Signal**: A stable module (many things depend on it, it depends on few) depends on a volatile module (many outgoing, few incoming). Compute instability `I = Ce / (Ca + Ce)` per module; stable (I near 0) depending on volatile (I near 1) is a violation.
**Why it matters**: Martin's Stable Dependencies Principle. Dependencies should point toward stability; violating this propagates churn into stable code.
**Refactor**: Introduce an interface owned by the stable module; make the volatile one implement it (DIP).

### Inappropriate intimacy
**Signal**: Class A reaches into B's private fields or internal methods. Grep for `_internal`, `friend`, or leading-underscore access across module boundaries.
**Refactor**: Move the method to where the data lives, or extract a new class that owns the interaction.

### Message chains
**Signal**: `a.b().c().d().e()` — three or more chained accessors.
**Why it matters**: Law of Demeter. Every node in the chain becomes coupled; changing any intermediate breaks the caller.
**Refactor**: Hide Delegate — replace the chain with a method on the first object that encapsulates the walk.

### Middle man
**Signal**: Class whose methods do nothing but delegate. >70% of public methods are single-line forwards.
**Refactor**: Remove Middle Man — callers talk directly to the real implementer. Exception: keep it if it is a deliberate facade over a volatile boundary.

---

## Lens 2: Cohesion and complexity

### God class / Blob / large file
**Signal**: LOC > 500 (TS/Java/C#) or > 300 (Python) or > 800 (Go — convention allows larger packages), or >20 public methods, or >10 instance fields, or name ends in "Manager"/"Helper"/"Util"/"Service" without a clear bounded responsibility.
**Metrics**: high WMC (weighted methods per class), high LCOM (lack of cohesion of methods).
**Why it matters**: Violates SRP; becomes a change magnet (divergent change).
**Refactor**: Extract Class by cohesion cluster. Find methods that touch the same subset of fields — that cluster is the new class. Start with the largest cluster.

### Long method
**Threshold**: body > 50 LOC AND cyclomatic complexity > 10 (count `if`/`for`/`while`/`case`/`catch` + 1) AND nesting > 2 levels. All three must hold.
**Why it matters**: Hard to read, test, reuse. Long methods are where bugs hide.
**Refactor**: Extract Method at every meaningful sub-step. Name extracted methods by *what*, not *how*.

### Long parameter list
**Threshold**: ≥5 parameters, especially 3+ of the same primitive type.
**Refactor**: Introduce Parameter Object for parameters that travel together (data clump). Replace primitive triads with value types.

### Shallow module (Ousterhout)
**Signal**: Public interface is almost as complex as implementation. Exported names are ~1:1 with internal operations. Paraphrased Ousterhout test: *Benefit (functionality hidden) − Cost (interface complexity)*. Deep = large positive value; shallow = near-zero or negative.
**Why it matters**: Modules exist to hide complexity. A shallow module imposes cost (learning the interface) without benefit (hiding substance).
**Refactor**: Deepen (absorb more responsibility), delete and inline, or merge with a neighbor.
**Exception — strategic depth**: a shallow wrapper around a volatile third-party boundary or to enable testing is justified even if currently thin.

### Lasagna / excessive layering
**Signal**: Tracing a request through the system passes through ≥6 files that only rename arguments. Common in over-engineered "clean architecture" apps.
**Test**: mentally cross out a layer. If layers above can still function meaningfully, the layer was pass-through — collapse.
**Refactor**: Collapse layers until each genuinely hides a concern. Controllers / services / repositories are fine; managers + orchestrators + coordinators + facades on top of those is lasagna.

### Divergent change
**Signal**: `git log -- <file>` reveals commit messages clustering into distinct unrelated themes ("fix auth", "adjust pricing", "add reporting column").
**Refactor**: Split by change axis. Files that change for different reasons should be different files.

### Feature envy
**Signal (Lanza & Marinescu, *Object-Oriented Metrics in Practice*, 2006)**: `ATFD > Few AND LAA < 1/3 AND FDP ≤ Few` where **Few ≈ 2–5**. ATFD = Access To Foreign Data (count of foreign getter/field accesses). LAA = Locality of Attribute Accesses (fraction of this method's data accesses that are to own fields). FDP = Foreign Data Providers (distinct foreign classes whose data is accessed).
**Refactor**: Move the method to the class whose data it manipulates.

### Data class
**Signal**: A class that is only fields + getters/setters, with behavior living in a "service" or "manager" that operates on it.
**Why it matters**: Anemic domain model (see Abstraction lens). Invariants are enforced by convention, not by types.
**Refactor**: Push behavior onto the class where the data lives. If the type has invariants, enforce them in the constructor and guard mutators.

---

## Lens 3: Abstraction quality

### Anemic domain model (Fowler)
**Signal**: Domain types are bags of fields with no behavior. Logic lives in `*Service`/`*Manager`/`*Helper` classes — often as procedural static methods.
**Why it matters**: You pay the complexity cost of OOP without the benefit. Invariants enforced by convention; any caller can put an entity into an inconsistent state.
**Refactor**: Move behavior onto the entity. Rule: operations whose preconditions are entity invariants belong on the entity. Transaction scripts can coexist for workflow coordination.
**Exception**: for simple CRUD systems, anemic is fine and rich domain models are over-engineering. Only flag when the domain has real invariants and behavior currently lives elsewhere.

### Primitive obsession
**Signal**: Domain concepts as raw strings/ints/maps — `userId: string`, `amount: number`. You can assign an email to `userId` and the compiler does not complain. Validation is duplicated at multiple boundaries.
**Refactor**: Introduce Value Object / domain primitive with a smart constructor. Encapsulate validation in one place. Use `opaque`/`newtype` where the language supports it.
**When to skip**: transport-only fields used in exactly one creation and one consumption site; the ceremony would exceed the value.

### Unenforced invariants / missing always-valid construction (smart constructors)
**Signal**: The same predicate (`if (!order.items.length)`, `assert email.contains("@")`, `if (amount <= 0)`) appears in ≥3 call sites on the same type. Public constructors accept any argument shape; validation is a separate `validate()` method callers may or may not invoke. `Optional`/nullable fields that are "required once the object is past step X" but the type doesn't say so.
**Detection**: grep the same field on both sides of a `!= null`/`.is_some()` check across ≥3 files; `new Foo(...)` followed within 1–5 lines by `foo.validate()` / `foo.isValid()` across multiple sites; factories returning `T` that throw on bad input (contrast with `Result<T, Error>`/`Either`). Bug history: "saved an order with 0 items", "null customer on paid invoice".
**Why it matters**: Khorikov's *Always-Valid Domain Model*; Alexis King's *Parse, don't validate*. If the type admits only valid values, construction *is* the proof of validity — downstream code stops re-checking, and bugs from skipped validation vanish.
**Refactor**: Make the constructor private; expose `create(...) -> Result<T, Error>` as the only path to a `T`. Push each invariant into the smallest type that owns it (`NonEmptyList<OrderItem>`, `PositiveAmount`, `Verified<Email>`). Replace "optional field required in state X" with a sum type (`Draft | Submitted { approver: UserId }`).
**Overkill**: transport-edge DTOs; pure CRUD with no domain rules; single-site types.

### Illegal states representable (boolean-blind / flag combinatorics)
**Signal**: Records with multiple booleans or nullable fields where only *some* combinations are legal, enforced by convention. `{ isPending: bool, isApproved: bool, approvedBy: UserId? }` — `isApproved=true, approvedBy=null` is a bug the type happily permits.
**Detection**: ≥2 booleans on one type whose meanings are not independent (check if any call site asserts `a && b` or `a → c`); nullable fields with comments like "required when X"; runtime assertions like `assert (approved && approver != null) || !approved`; `Option<T>` fields whose `Some`/`None` depends on a sibling enum.
**Why it matters**: Minsky (Jane Street), Wlaschin (*Domain Modeling Made Functional*). Cardinality of `Bool × Bool × Option<T>` exceeds the set of legal states; a sum type is the algebraic way to make `|type| = |legal states|`. Broader than the "implicit state machine" smell: applies to any record shape, not just entity lifecycle.
**Refactor**: Replace flag-soup records with a sum type (tagged union, discriminated union, Rust enum, sealed class); each variant carries only fields valid in that state. In languages without sum types, use subtype-per-state or a `type` discriminator with narrowing (TS). Phantom/marker types (`Unvalidated<T>` vs. `Validated<T>`) for proof-carrying distinctions.
**Overkill**: booleans that really are independent (feature flags, preferences); DTOs mirroring an uncontrolled external schema.

### Shotgun parsing / validate-don't-parse
**Signal**: Untyped input (`dict`, `Map<String, Any>`, `JsonValue`, `serde_json::Value`) passed deep into the system; each layer re-checks "does this field exist / is it a number / is it in range" without narrowing the type. Exceptions arise anywhere downstream because no boundary declares "past this line the shape is guaranteed."
**Detection**: grep for `any`/`JsonValue`/`Map<String, Object>` crossing ≥2 module boundaries before destructuring; defensive `if "field" in payload`/`payload.get("field")` in handlers, services, *and* persistence; try/except around field access deep in business logic; same regex or format check in request-validation, service, and DB-serialization layers.
**Why it matters**: Alexis King, *Parse, don't validate* (2019); "shotgun parsing" (Bratus/Patterson/Sassaman, LangSec). Parsers are `Bytes -> Either<Error, Typed>`; validators are `Typed -> Bool` and discard the proof. Preserve the evidence in the type.
**Refactor**: Parse once at the edge into a concrete domain type; only the parsed type enters the core. Unparsed and parsed get distinct types (`RawRequest` vs. `BookingCommand`). Delete every downstream defensive check — a type mismatch downstream is a bug at the parse boundary, not a runtime branch.
**Overkill**: genuine pass-through (proxy, logging pipeline, schema-less data lake) where the system has no opinion on content.

### Missing aggregate / cross-entity invariants with no enforcement point
**Signal**: An invariant spans multiple entities ("order total = sum of line items", "team roster ≤ plan limit", "account balance never negative after any transfer") with no single object that owns enforcement. Multiple services mutate related entities independently; the rule is re-asserted (or forgotten) at each write site.
**Detection**: repositories/DAOs of related tables saved in the same transaction by >2 different services; business rules involving two entity types expressed as free-floating helpers; transactions opened outside the domain layer (`@Transactional` on controllers) spanning multi-entity writes; lock/retry/optimistic-concurrency code scattered where an aggregate root would have centralized it. Bug history: recurring "inconsistent after concurrent edit".
**Why it matters**: Evans, *DDD* (2003). An aggregate is the smallest transactional boundary that preserves a domain invariant — aligning ACID scope with semantic scope. Without the boundary, consistency is probabilistic.
**Refactor**: Identify the set of entities that must be mutated atomically to preserve invariant I — that is one aggregate; pick a root. Route all writes through the root's methods (`order.addLineItem(...)`, not `lineItemRepo.save(...)`). Only the root has a repository. Invariants that span aggregates become eventual consistency via domain events, not larger aggregates.
**Overkill**: CRUD app where each entity's invariants are self-contained; cases where the "invariant" is just a DB foreign-key constraint.

### Complected concerns (Hickey)
**Signal**: Code that interleaves two independent things that could be pulled apart:
- Business logic + transport (payment logic in HTTP handler)
- Transformation + iteration (can't reuse the transform without the loop)
- Policy + mechanism (retry strategy baked into the network client)
- What + when (compute mixed with scheduling)
- Domain logic + observability (log/trace calls scattered in core business functions)
- Domain logic + error handling (retry policies inside domain code)
- Data + time (mutable state forcing temporal reasoning)

**Decompose test**: *Could I swap one concern for a different implementation without touching the other?* If no, complected.
**Refactor**: Pull out a pure core; wrap it in an impure shell. The Hickey move is almost always: small pure functions, then a thin driver that composes them with the outside world.
**Counter-case**: in latency-sensitive, data-heavy, or imperative-by-nature domains (games, drivers, numerical kernels), pure/impure separation can add overhead that is not worth the reuse gain. Flag but soften confidence.

### Functional core, imperative shell absent (Gary Bernhardt)
**Signal**: Decision logic ("should we charge a late fee?", "which route wins?") is interleaved with I/O calls in the same function. Unit tests require mocking 3+ collaborators to exercise one decision. Pure business rules cannot be evaluated without a DB/HTTP client.
**Detection**: functions mixing `await db.`/`await http.` with non-trivial `if`/`match` branching on domain values; test files with >5 mocks/stubs per subject under test; inverted test pyramid (few unit, many integration) because unit tests are impractical; business conditionals inside repository or client methods.
**Why it matters**: Closely related to Hickey's complected concerns but with a distinct, testable shape: separating the pure kernel from the effectful boundary makes the core equationally reasonable and the shell mechanical. Gary Bernhardt, *Boundaries* talk (2012). Analogous to IO monad / effect-system split.
**Refactor**: Split each such function into (a) a pure `decide(state, inputs) -> Decision` and (b) an impure shell that loads state, calls `decide`, and performs the effects the `Decision` dictates. `Decision` is a data value — a sum type of effects — not a callback. Test the core with plain values; test the shell with a handful of integration tests.
**Overkill**: CRUD handlers with trivial branching; latency-critical code where the intermediate `Decision` allocation is measurable.

### Unidirectional data flow absent (Flux / Redux / Elm architecture)
**Signal**: UI or long-lived state is mutated directly by many components/handlers; the same logical field is written from ≥3 sites; debugging "what changed and why" requires reading every handler. Most common in frontends but also appears in CLI TUIs, game loops, desktop apps, and long-running backend daemons.
**Detection**: grep for direct assignments to shared store/context fields (`store.x =`, `setState({x:...})`, `context.y =`) across distinct components. Flag when ≥3 files mutate the same logical key AND there is no single dispatcher/reducer. Components with ≥5 unrelated pieces of `useState`/`useReducer` are candidates.
**Why it matters**: Without a single reduction point, state transitions are not observable as data; debugging requires reconstructing a causal chain from a diff. Unidirectional flow (Facebook Flux 2014, Elm Architecture by Czaplicki) restores the invariant: *state at time T+1 is a pure function of state at T and one action*.
**Refactor**: Define an `Action` sum type enumerating every legal mutation. Write one reducer per state slice; mutations become `dispatch(action)`. Colocate component-local ephemeral state (form inputs, hover) — do not centralize everything. Add a middleware/devtools seam so actions are loggable.
**Overkill**: state that is truly component-local (a modal's open flag); apps with <5 screens and trivial flows. Centralizing form-input state in Redux is the textbook over-engineering case.

### Scattered error handling / retry policy / try-catch pyramid
**Signal**: Retry/timeout/circuit-breaker logic appears inside domain classes or mixed with business rules — different layers each retry on the same error, multiplying latency and load. Or: the same error class caught and re-wrapped at 3+ layers; `try/catch` nested ≥2 deep; control flow depends on exception type; domain functions return `T` but raise on every validation.
**Detection**: count `catch`/`except`/`rescue` per file (flag >8 catches, or >40% of LOC inside catch blocks); call chains where every layer adds its own try/catch for the same exception class; retry annotations (`@Retryable`) at multiple layers of one stack.
**Why it matters**: Errors-as-exceptions complects happy-path data flow with failure control flow; each layer re-implements policy. Railway-oriented programming (Wlaschin) treats errors as values on a parallel track, making the pipeline composable and error cases enumerable in types.
**Refactor**: Move failure policy (retry/timeout/circuit-breaker) to a single middleware/decorator/interceptor that wraps calls — domain code throws, policy layer decides response. For exception-heavy domain code: define `Result<T, E>` / `Either` with a closed set of error variants per module; convert domain functions to return `Result`; compose with `bind`/`map`/`?` instead of try/catch ladders; reserve exceptions for truly unexpected failures. Keep one outer-edge adapter that converts `Result` to HTTP status / log / retry decision.
**Overkill**: languages where exceptions are deeply idiomatic (Java checked exceptions, Python domain errors) and partial adoption would create a bilingual codebase. Commit at a boundary or skip.

### Observability complected with logic
**Signal**: `log.info`, `tracer.span`, `metrics.increment` calls scattered through core domain functions; one-line "what happened" logs every 5 lines.
**Refactor**: Inject observability at boundaries (middleware, aspects, decorators). Emit structured domain events from the core; translate to logs/metrics/traces at the edge.

### Missing domain events / direct cross-module orchestration
**Signal**: When something happens in module A, A directly calls B, C, D to update them. Adding a new consumer means editing A. A's "save" method is 40 lines of "also notify billing, also send email, also update search index." Orchestrators grow unboundedly.
**Detection**: a single method calling ≥3 unrelated subsystems after a state change (`order.place()` calling email, inventory, analytics, audit); circular or near-circular imports between feature modules that "need to notify each other"; test setup requires mocking 4+ collaborators to exercise one domain operation; commit history shows every new cross-cutting feature editing the same hot method in the core.
**Why it matters**: Evans, *DDD* (2003); Udi Dahan; Vaughn Vernon. Events invert control — producers depend only on an event schema, not on consumers — converting N×M coupling into N+M through a shared vocabulary.
**Refactor**: Emit a domain event (`OrderPlaced { orderId, total, customerId }`) from the aggregate; subscribers register themselves. Keep dispatch synchronous and in-process first (in-memory bus). Upgrade to a broker only when you actually cross a process boundary. Distinguish **domain events** (internal, past-tense fact inside one bounded context) from **integration events** (cross-context); don't leak the former to external subscribers.
**Overkill**: fewer than 2 subscribers per event type — direct call is simpler and you can refactor later.

### Leaky abstraction
**Signal**: Interface users must know implementation details to use it correctly. Example: a `Cache` interface whose callers must know it's Redis-backed to handle failure modes.
**Refactor**: Either make the leak explicit in the interface or hide it properly (handle failure inside). Half-hidden leaks are the worst state.

### Domain depends on infrastructure / missing ports (Hexagonal Architecture)
**Signal**: Domain files `import` ORM classes, HTTP libraries, SDK clients, filesystem APIs, or `datetime.now()`/`uuid.v4()` directly. Switching database or mocking time requires editing domain code. Tests for domain logic spin up Postgres or a fake HTTP server.
**Detection**: grep in `domain/`, `core/`, `model/` directories for: `from sqlalchemy`, `import boto3`, `import requests`, `@Entity`, `axios`, `fetch(`, `new Date()`, `Instant.now()`, `System.currentTimeMillis()`, `os.environ`, `fs.`/`std::fs`. Tests that import both the domain module and a DB/HTTP mock library. Interfaces owned by infrastructure (`UserRepositoryImpl implements UserRepository` where the interface lives in `infra/`). "We can't unit test X without starting the whole app."
**Why it matters**: Cockburn (2005), *Hexagonal Architecture*; generalized by Martin's Clean Architecture. DIP — high-level policy must not depend on low-level mechanism. When domain depends on infrastructure, every infra change ripples through business logic.
**Refactor**: Define **ports** (interfaces) in the domain, expressed in domain vocabulary (`OrderRepository.findOpenForCustomer(CustomerId)`, not `findByStatusAndCustomerId(string, uuid)`). Adapters in `infra/` implement the ports. Composition root wires them at startup. Inject `Clock`/`IdGenerator` ports instead of calling `now()`/`uuid()` directly.
**Overkill**: scripts, one-off tools, embedded apps with a single stable adapter per concern. Hexagonal tax is real.

### Missing anti-corruption layer (foreign model bleeding into the domain)
**Signal**: Types, field names, or enums from a third-party API or legacy DB appear inside the domain — `stripe.Charge`, `SAPOrder`, `LegacyAccountDTO`. When the external provider changes a field, the diff touches dozens of domain files. Optional/nullable fields proliferate because the foreign model has them. Enum values use the foreign system's codes.
**Detection**: grep in `domain/`/`core/` for vendor SDK types or legacy class names; field names with vendor casing/abbreviations (`sfdc_id`, `stripe_cus_`, `sap_doc_num`) deep in the model; upstream schema change → shotgun-surgery diff across the codebase; integration tests asserting vendor-specific JSON shapes from business code.
**Why it matters**: Evans, *DDD* (2003), ch. 14. Interposing a translation layer at a context boundary preserves model coherence — prevents foreign ontology from overwriting the home ontology. Analogous to an FFI boundary between type systems.
**Refactor**: Introduce a translator/adapter at the integration boundary: `StripeAdapter.chargeToPayment(stripe.Charge) -> Payment`. Only `Payment` enters the domain. Model your own enums/IDs; map at the edge. For a whole foreign bounded context, the ACL is a module (façade + translators + its own error types), not a single class.
**Overkill**: thin wrapper over a well-designed vendor SDK that is itself at the right level of abstraction (crypto libraries). Don't re-wrap `Signature` as `MySignature`.

### Missing bounded context (DDD)
**Signal**: Same named concept ("User", "Order") has different meanings in different subsystems but shares one model. Optional fields proliferate; conditional logic branches on which subsystem is calling.
**Refactor**: Identify contexts; split the type into context-specific types (`BillingUser`, `AuthUser`, `ProfileUser`). Translate at boundaries. Does not require microservices; namespaces suffice.
**When to skip**: CRUD systems with no conflicting vocabularies — a single `User` is fine.

### Speculative generality (YAGNI violation)
**Signal**: Abstractions, hooks, plugin points, or generic parameters with exactly one concrete use. Config flags never toggled.
**Refactor**: Inline. Bring back the concrete thing. Add the abstraction when the second real use arrives — not the hypothetical one.

### The wrong abstraction (Metz)
**Signal**: Three or more subclasses override the same methods to do completely different things. Callers pass flags that turn off half the abstraction's behavior. Long `if kind == X` ladders inside the "shared" base.
**Why it matters**: Sandi Metz — "duplication is far cheaper than the wrong abstraction." Forced reuse compounds the mistake.
**Refactor**: Inline the abstraction. Duplicate back into each caller. Study the duplicates; a *different* abstraction may emerge.

### Temporal coupling
**Signal**: `obj.init(); obj.configure(...); obj.start()` — caller must invoke methods in a specific order; type system does not enforce it.
**Refactor**: Constructor does setup, or use type-state (`Unconfigured → Configured → Running`) where the compiler enforces order.

### Implicit state machine / scattered entity state
**Signal**: A logical entity's state is represented by multiple boolean flags or a status field, and the transition logic is scattered across many call sites with no central policy. Concrete patterns:
- Multiple booleans on one entity: `isActive`, `isPaused`, `isClosed`, `hasError` — only some combinations are legal; nothing enforces it.
- Status enums with transitions inlined across files: `if (order.status === 'pending' && x) order.status = 'confirmed'` repeated in N places.
- Per-entity state updated across different services/modules without a single owner.
- Bug history: recurring "stuck in weird state" or "inconsistent state after X" reports.

**Detection**: grep for `status ===`/`status ==`/`state =`/boolean-field assignments targeting the same entity across >2 files. Count distinct states mentioned in conditionals vs. distinct writer sites — if writers >> states, transitions are scattered.

**Threshold for action**: ≥3 distinct states OR ≥3 writer sites across different files. Below that, an explicit state machine is over-engineering.

**Why it matters**: Complecting state with transition logic (Hickey). Every caller must reimplement the rules; adding a state requires finding all write sites; invalid combinations go undetected until a bug is filed. The entity is not really modeled — only its fields are.

**Refactor**:
1. Enumerate the **conceptual states** (not the flags — what the entity *is* at each moment).
2. Enumerate legal transitions and their triggers.
3. Introduce a single owner type:
   - **Type-state** (Rust, TS discriminated unions, Haskell sum types): each state is its own type holding only the fields valid for that state; transitions return a new type. Invalid states become unrepresentable.
   - **State pattern** (Gamma et al.): one class per state, polymorphic `handle()` methods, entity delegates behavior to current state object.
   - **Actor / per-entity state machine**: one actor (or reducer) per entity instance owns the state; other code sends events/messages, actor decides transitions. Fits event-driven systems (streaming, tickers, orders, sessions).
4. Replace scattered flag updates with transition calls on the owner (`order.confirm()`, not `order.status = 'confirmed'`).
5. Delete the flags.

**CS grounding**: State pattern (GoF); type-state / "make illegal states unrepresentable" (Yaron Minsky); actor model (Hewitt) for per-entity isolation; Hickey — state and time must not be complected across the codebase.

**Rollback signal**: if the number of states balloons past ~7 or transitions form a near-fully-connected graph, the entity probably has hidden sub-concerns — split before continuing. If the FSM is becoming a combinatorial product of independent dimensions (e.g., `{connection × auth × mode}` encoded as flat states), the next level up is a **hierarchical statechart** (Harel, 1987) with parallel orthogonal regions — adopt XState / SCXML / Spring Statemachine only when hierarchy is >2 levels deep; otherwise a discriminated union is enough.

### Audit/history reconstructed from current-state snapshots (Event sourcing gap)
**Signal**: The code repeatedly asks *"how did this entity get into this state?"* and answers with audit-log tables, triggers writing change rows, shadow tables, `updated_by`/`updated_at` on every column, or periodic snapshots compared by diff. Business asks for "as-of" queries, replayable projections, or "undo this day" and the code cannot do it without custom forensic SQL.
**Detection**: grep for `*_history`, `*_audit`, `*_log` tables where >50% of columns mirror the primary table; DB triggers that duplicate writes; `ORDER BY updated_at DESC LIMIT 1` reconstruction code; projections built by repeatedly scanning current state.
**Why it matters**: The system is trying to recover events from state — the inverse of what it should do. Fowler, *Event Sourcing* (2005): events are the source of truth; current state is a derivable projection. Temporal queries, audit, and new read models become free.
**Refactor**: Append events to an event log keyed by aggregate id (`OrderPlaced`, `OrderConfirmed`). Derive current state by folding events; cache with snapshots. Build read models as projections over the log — new views no longer require schema migrations. Migrate incrementally: write events alongside existing state writes first; cut over reads last.
**Overkill**: CRUD apps with no audit requirements, no temporal queries, and a single read shape. Event sourcing pays a permanent cost (eventual consistency, snapshotting, schema evolution) — do not adopt for simple admin panels.

### Read and write models fighting each other (CQRS gap)
**Signal**: One model serves both transactional writes (needs normalization, small footprint, invariants) and reporting/listing reads (needs denormalization, joins, pagination). Symptoms: ORM entities with dozens of eager-loaded relations for list views; repeated N+1 regressions; indexes added for reads that slow writes; write-side code polluted with read-only projections; recurring "dashboard is slow" tickets.
**Detection**: compute read/write ratio per entity; ratios >100:1 with complex joins on the read side flag. Grep for DTO classes that duplicate entity structure with extra joined fields. Count indexes per hot table — >6 often indicates read-path compensation on the write table.
**Why it matters**: Meyer's Command-Query Separation generalized to aggregates (Greg Young). One model optimized for two workloads is optimal for neither; the contention is the smell.
**Refactor**: Split the write model (aggregate enforcing invariants, small schema) from the read model (materialized views, denormalized projections). Read model updated via events or CDC; staleness budget made explicit. Start within a single DB (views, materialized views) before separate stores.
**Overkill**: reads and writes with similar shape and modest volume. CQRS without a real read/write asymmetry is cargo-culted.

### Implicit dataflow pipeline hidden as nested calls (pipes-and-filters gap)
**Signal**: A linear transformation `a -> b -> c -> d` is implemented as `step4(step3(step2(step1(x))))` or as a single function body with paragraph-breaks between pure stages. Each stage is tested only end-to-end; adding or reordering a stage means editing the monolith.
**Detection**: functions with ≥4 sequential assignments to intermediate variables feeding only the next line; deeply nested comprehensions or method chains mixing transformation with I/O; "transform" functions that contain logging, retry, or persistence calls.
**Threshold**: ≥4 sequential pure stages on a stream/collection AND at least one of: reuse desired, stages tested only e2e, unclear where to add a new stage.
**Why it matters**: Shaw & Garlan, *Software Architecture*; Unix pipeline philosophy. Pipes-and-filters makes the dataflow graph the top-level object; filters become independently testable, rearrangeable, and parallelizable. Nested calls hide the graph and force readers to reverse-engineer it.
**Refactor**: Name each stage as a pure function `Stage: In -> Out` with explicit types. Compose with a pipeline combinator (`pipe`, iterator chain, Reactor/RxJS operator chain, Unix-pipe analog). Make the pipeline definition *data* — a list of stages — so stages can be inserted/skipped/reordered. Push I/O to the endpoints; filters stay pure.
**Overkill**: 2-step transforms; performance-critical hot loops where the abstraction overhead matters.

---

## Lens 4: Evolution signals

Static analysis alone misses half of real problems. The commit log carries the rest.

### Hotspot (Tornhill)
**Detection**: `git log --since=1.year --format=format: --name-only | grep -v '^$' | sort | uniq -c | sort -rn | head -30`. Cross-reference with file LOC: score = `churn_count × LOC`. Top 1% of score are hotspots.
**Why it matters**: Tornhill's empirical finding — a small fraction of files accounts for a disproportionate share of development activity and defects. Refactor ROI concentrates there.
**Action**: audit the top 5 hotspots under all four lenses. These almost always have multiple smells and warrant P1 treatment.

### Shotgun surgery
**Signal**: A feature change touches N unrelated files. Detect via co-change: `git log --name-only` and look for file pairs that recur together across commits at a rate higher than chance.
**Refactor**: The co-change pattern reveals the missing boundary. Extract a module containing the pieces that change together.

### Parallel inheritance / parallel switch
**Signal**: Every time you add a subclass of `Foo`, you also add a subclass of `Bar` — or a corresponding case in a switch statement elsewhere.
**Refactor**: Collapse the parallel hierarchy by making one responsible. `Bar`-behavior becomes a method on the corresponding `Foo` subclass.

### Fossil / dead code
**Signal**: No commits for >1 year, no apparent callers (grep returns only self or test imports), untouched by recent features.
**Caveat**: reflection, string dispatch, macros, and dynamic imports can hide call sites. Verify at runtime when possible.
**Refactor**: Delete. If unsure, mark `@deprecated` with a removal date.

### Ownership diffusion
**Signal**: `git log --format='%an' -- <file> | sort -u | wc -l` high (many distinct authors) with no dominant contributor (>50% of commits), combined with high churn.
**Why it matters**: Conway's Law in reverse — modules with diffuse ownership accumulate smells because no one defends the design.
**Refactor**: Primarily an org move. Often paired with splitting the module along ownership lines.

### Temporal hotspot mismatch
**Signal**: Top-churn files are not domain-core files. Utilities, infrastructure, or config files dominate the hotspot list while domain code is quiet.
**Why it matters**: Change should concentrate in the parts of the system where the business actually evolves. If it concentrates in infrastructure, the boundary is placed wrong or the abstractions are leaking.
**Refactor**: Re-examine the boundary between domain and infrastructure. Often indicates a leaky abstraction or an inverted dependency.

---

## Data ingestion & serving subsystem (cross-cutting)

Load this section when running the **mandatory ingest→serve pass** (SKILL.md, Phase 3). It fires whenever the system both **ingests** data from a latency-bound or fallible source (files, DB, sockets, HTTP/streaming feeds, message queues, sensors, large CSV/Parquet/columnar blobs) and **serves** it to a consumer (UI, API, another module, export). These smells span Coupling, Cohesion, and Abstraction; they are grouped here because the audit treats the data path as one thing and the canonical refactor (last entry) addresses them together.

These are **promotion-rule** smells: when the ingest→serve path exists, the absence of a robust, observable, non-blocking abstraction is *always* a candidate finding — sized by `decision-frameworks.md` §13, never silently dropped. Proportionality governs *how much* robustness, not *whether* the boundary, the status stream, and a non-blocking UI exist.

### No ingestion/serving boundary (scattered inline I/O)
**Signal**: Raw I/O — `open`, `read`, `parse`, `query`, `connect` — appears inline in UI event handlers, controllers, request handlers, and domain code, with no single `DataSource`/`Repository`/`Store`/ingestion-engine type owning it. There is no seam at which to add retry, streaming, caching, or observability because the I/O is everywhere.
**Detection**: grep for file/socket/HTTP/DB primitives (`read_to_string`, `File::open`, `fs.readFile`, `open(`, `requests.get`, `fetch(`, `SELECT`, `cursor.execute`, `socket.recv`) across UI/handler/domain directories; count distinct files containing them. Check whether a data-access module exists and whether everything routes through it.
**Threshold**: ingest/serve primitives in ≥3 files outside a dedicated data module; OR any I/O inside a UI event handler / render path; OR the same source parsed in >1 place.
**Why it matters**: Hexagonal Architecture (Cockburn) — I/O is a detail that belongs behind a port in domain vocabulary; scattering it complects transport with everything and leaves no place to enforce robustness. Without the boundary, every later fix (streaming, retry, progress) has to be made N times.
**Refactor**: see **the canonical robust ingest→serve abstraction** below — define the port, route all I/O through it, delete the inline calls.
**Overkill**: a one-shot script that reads one file once and exits; a library whose single job *is* the I/O primitive.

### Load-all ingestion (no streaming / windowing / pagination)
**Signal**: The whole input is read into memory (`read_to_string`, `io.ReadAll`, `.collect()` of an entire query, `JSON.parse(entireFile)`, `pd.read_csv(whole)`) before anything is served. Memory scales with input size; first byte served waits for last byte read.
**Detection**: grep for whole-read calls feeding a single in-memory collection that the serve path then indexes; look for absence of `BufReader`/iterator/generator/cursor/`LIMIT … OFFSET`/range reads. Symptom history: "OOM on big file", "slow to open large dataset", "works on the sample, dies on prod data".
**Threshold**: any unbounded whole-input read where input size is attacker- or user-controlled or grows over time; OR served data set materially larger than the visible/queried slice.
**Why it matters**: Memory becomes a hard ceiling on dataset size and a latency floor on first result. Streaming (bounded memory, incremental results) is the difference between "opens instantly, scrolls smoothly" and "freezes, then maybe crashes". Kleppmann, *DDIA* ch. 10–11 (batch vs. stream).
**Refactor**: stream/iterate the source; window or paginate the serve side to the visible/queried range; index incrementally as chunks arrive. Keep only the working set resident; page the rest. For columnar/large files, read by row-group/range, not whole-file.
**Overkill**: inputs provably small and bounded (a config file, a few-KB fixture); when the entire dataset must be resident for the algorithm (in-memory analytics on a known-small set).

### Sequential ingest of parallelizable work
**Signal**: Independent ingest units (one per file / day / shard / partition / key / endpoint) processed in a serial `for` loop, so the CPU sits idle on I/O wait and wall-clock is the *sum* of per-unit latencies.
**Detection**: a loop over files/days/sources each doing blocking I/O + decode with no concurrency primitive (`join_all`, worker pool, `errgroup`, `asyncio.gather`, `rayon`, channel fan-out). Profile/log signature: total time ≈ Σ per-unit time; one core busy, disk/network underutilized.
**Threshold**: ≥N independent units (N≈4) processed serially where units share nothing and the per-unit work is I/O-bound or CPU-bound-and-divisible.
**Why it matters**: Embarrassingly-parallel work run serially is throughput left on the floor. Staged, pipelined concurrency (SEDA — Welsh, Culler & Brewer, 2001) lets read overlap decode overlap index; bounded fan-out saturates the slow resource without overwhelming it.
**Refactor**: partition by the natural key; process units through a **bounded** worker pool (cap concurrency to protect the source and memory); pipeline the stages so read ∥ decode ∥ index. Preserve ordering only where the serve side needs it (merge at the end, or key-order within partition).
**Overkill**: a handful of tiny units where setup cost exceeds the gain; sources that rate-limit hard enough that one connection is already the ceiling; strict total-ordering requirements that defeat parallelism (document the constraint).

### No fault tolerance / not resumable / not idempotent
**Signal**: One bad record, dropped connection, or transient error aborts the whole ingest (`panic`/throw/`os.exit`). No retry, or retry with no backoff/jitter, or retry on non-transient errors. Re-running re-ingests from the start; re-ingesting the same input double-counts (no dedupe). Long-lived ingest has no supervisor — a crash ends it permanently.
**Detection**: grep the parse/ingest loop for `unwrap`/`expect`/`panic!`/bare `throw`/`raise` on per-record paths; check whether errors are isolated per unit or propagate to abort; look for a resume cursor/checkpoint table/offset file; look for an idempotency/dedupe key (`(source, offset)`, `(symbol, ts)`, message id, content hash); check for retry with `backoff`+`jitter` and error classification (retry only timeouts/5xx/connection-refused). Bug history: "import failed at row 4M, started over", "duplicate rows after re-run", "feed dropped overnight and never recovered".
**Threshold**: any side-effecting ingest that (a) aborts the batch on a single record error, OR (b) is not resumable across a crash, OR (c) double-applies on re-run, OR (d) has a long-lived reader with no restart.
**Why it matters**: All real I/O is at-least-once and partially-failing. Per-unit isolation + retry-with-backoff (Nygard, *Release It!*), resumable checkpoints, and idempotent effect (Kleppmann; Morling, *On Idempotency Keys*) are what separate "robust pipeline" from "demo". Supervised restart / let-it-crash (OTP) keeps long-lived ingest alive.
**Refactor**: isolate failures per unit — a bad record becomes an `Err`/`Result` pushed into the stream and skipped+reported, not a process abort. Retry *transient* errors only, with exponential backoff + jitter + a max-elapsed cap. Add a resume cursor/checkpoint so re-runs continue from the last committed position. Add an idempotency key so re-ingest is a no-op on already-seen units. Supervise long-lived readers (restart from clean state on unhandled error).
**Overkill**: pure in-process transforms of trusted, already-validated in-memory data; a one-shot import where "re-run from scratch" is acceptable and cheap (document it).

### No backpressure / unbounded buffering
**Signal**: An unbounded channel/queue/list sits between a fast producer (ingest) and a slower consumer (serve/index/UI). The producer never blocks; the buffer grows without limit. Or: no queue at all, so a burst of input is held entirely in memory.
**Detection**: grep for unbounded channels/queues (`unbounded_channel`, `chan` with no cap, `Queue()` with no `maxsize`, `Channel.CreateUnbounded`, `Subject` with no buffer policy); check whether the producer can outrun the consumer and what happens when it does. Symptom: memory climbs under load and never recovers; "fine until a burst".
**Threshold**: any producer/consumer hand-off across a latency boundary using an unbounded buffer; OR a burst-prone source with no overflow policy.
**Why it matters**: An unbounded buffer converts a transient speed mismatch into an OOM. Backpressure (Reactive Streams; SEDA admission control) makes the slow stage push back on the fast one — the system degrades by *slowing*, not by *dying*. Bounded queue + explicit overflow policy (block / drop-oldest / drop-newest / reject / sample) is a design decision the audit should force into the open.
**Refactor**: bound every inter-stage queue; pick and name the overflow policy per stage (block the producer for durable work; drop-oldest for live/lossy feeds like charts/telemetry; reject+signal for request paths). Propagate backpressure end-to-end so the slowest stage governs intake.
**Overkill**: provably bounded producers (a fixed small batch) where the buffer can never exceed a safe size.

### Blocking UI on latency-bound I/O
**Signal**: File open, network connect, query, or parse runs **on the UI thread / inside an event or render handler**, freezing the interface until it completes. The window stops repainting; input is ignored; the OS may show "Not Responding".
**Detection**: grep for I/O primitives inside UI event handlers, render functions, component bodies, `onClick`/`onwheel`/`useEffect`/`paint`/`view` paths; look for blocking calls not dispatched to a worker/async task/thread pool. Symptom: "app freezes when I open a file", "spinner but the whole window is stuck".
**Threshold**: any latency-bound I/O (anything that can exceed ~16 ms) on the UI thread; any synchronous read in a render/redraw path; re-reading/re-parsing the source on every redraw.
**Why it matters**: The UI thread is a real-time deadline (one frame). Blocking it is the single most visible robustness failure to a user. The fix is the imperative-shell/functional-core split (Bernhardt) plus moving I/O off-thread and serving from an in-memory store.
**Refactor**: run all latency-bound work off the UI thread (async task / worker / thread pool / background actor). The UI dispatches a request and returns immediately; results and progress arrive as pushed events (next smell). Render reads the in-memory `Store`, never the source — no re-parsing on redraw.
**Overkill**: trivially fast, bounded reads known to complete within a frame (a tiny cached config). Even then, prefer async for consistency.

### No lifecycle / progress events for the UI ("what are we waiting on?")
**Signal**: There is latency-bound work but the UI has no way to tell the user what is happening. Best case a bare indeterminate spinner; worst case a frozen window. The user cannot distinguish "connecting", "downloading", "parsing", "retrying", "stuck", or "done".
**Detection**: check whether the data boundary exposes a typed status/lifecycle the UI subscribes to, or whether the UI polls/guesses/shows a static spinner. Look for a `LoadStatus`/`IngestState` enum and an observer/signal/event channel the UI reads. Absence + presence of latency-bound work = finding.
**Threshold**: any user-facing latency-bound operation with no per-phase status surfaced to the UI; OR progress that the UI computes by reaching into I/O internals rather than receiving as data.
**Why it matters**: "Always keep the user informed about what is going on, through appropriate feedback within reasonable time" (Nielsen, *Visibility of system status*, heuristic #1). Uncertainty, not latency, is what makes waiting feel broken. The boundary already knows the phase; it must *publish* it.
**Refactor**: model the data boundary's life as an explicit state machine and emit it as a stream the UI observes — e.g. `Idle → Opening → Connecting → Reading{pct} → Parsing{pct} → Indexing → Ready/Serving → Retrying{attempt} → Error{cause, retryable} → Done`. The UI maps each state to a message (and progress/ETA where known) via the **Observer** pattern (signals / event bus / reactive stream). Progress is **data pushed from the boundary**, never the UI reaching in. The *same* stream feeds logs/metrics (next smell), so there is one source of truth.
**Overkill**: operations guaranteed sub-100 ms; non-interactive batch jobs with no human waiting (use logs/metrics instead).

### Observability not designed into the I/O boundary
**Signal**: To answer "is the ingest progressing, stalled, or failing, and how fast?" you must read scattered `print`/`log` lines buried in the parsing code — or there is nothing. No metrics on latency, throughput, queue depth, error/retry rate, or bytes in/out. Logs are interleaved with domain logic rather than emitted at the boundary.
**Detection**: grep for `log.`/`print`/`console.log` inside parse/decode/transform functions (observability complected with logic); check for a metrics surface (latency histogram, counter for records/errors/retries, queue-depth gauge) at the boundary; check whether the lifecycle stream (previous smell) is also the source for logs/metrics or whether they are independent and inconsistent.
**Threshold**: any non-trivial ingest/serve subsystem with no boundary-level metrics; OR observability calls scattered through ≥3 transform functions; OR logs and UI status derived from different, divergent sources.
**Why it matters**: A robust I/O subsystem is one you can *see*. Observability is a first-class concern of the boundary, not an afterthought sprinkled in the core (Hickey — don't complect *what it does* with *how it's watched*). One typed event stream at the boundary should fan out to logs, metrics/traces (OpenTelemetry-style), and the UI — keeping all three consistent by construction.
**Refactor**: emit the lifecycle state machine + structured events from the boundary; derive logs, metrics (latency histogram, throughput, queue depth, error/retry counters, bytes), and UI status from that *one* stream. Remove `log`/`print` from the pure filters. Instrument at stage edges (read/decode/index/serve), not inside the transforms.
**Overkill**: trivial bounded reads; a script where stdout is sufficient. Don't stand up a metrics pipeline for a 50-line importer — but still emit a single status value.

### The canonical robust ingest→serve abstraction (the refactor these compose into)
When the mandatory pass shows gaps, the prescription is almost always one composed design, not eight separate fixes. Name it explicitly in the finding; it is the existing catalog entries assembled for the I/O subsystem.

**Shape**:
1. **Port (Hexagonal)** — a `DataSource` interface in domain vocabulary: `open(spec) -> Stream<Result<Chunk, IngestError>>` for ingest; a serve-side `Store`/`Repository` queried by the visible/needed range. The UI and domain depend only on these interfaces; concrete file/DB/socket adapters live in `infra/`.
2. **Pipes-and-filters pipeline** — `read → decode → validate/normalize → index/store`, each a **pure filter** with explicit `In -> Out` types; I/O only at the endpoints (functional core, imperative shell — Bernhardt). The pipeline definition is *data* (a list of stages) so stages can be added/reordered/parallelized.
3. **Staged, bounded concurrency (SEDA)** — partition by natural key; a bounded worker pool per stage; read ∥ decode ∥ index overlap; **bounded** inter-stage queues with a named overflow policy (Reactive-Streams backpressure).
4. **Single-writer serve store (actor / LMAX)** — serve-side state owned by one task consuming a command/query queue (`mpsc` consumer / `MailboxProcessor` / `GenServer` / goroutine+channel). No shared `Mutex` over the dataset; the UI sends queries and receives snapshots. Supervised (let-it-crash + restart).
5. **Fault tolerance** — per-unit isolation, retry-with-backoff+jitter on transient errors only, resume cursor/checkpoint, idempotency/dedupe key.
6. **Lifecycle state machine + observability** — the boundary emits one typed status stream (`Idle → Opening → … → Ready → Retrying → Error → Done` with progress); logs, metrics, and the UI all derive from it.
7. **UI via Observer** — the UI subscribes to the status stream (signals / event bus / reactive stream), runs ingest off-thread, renders the current phase + progress, and reads the in-memory `Store` on redraw (never re-parses).

**Per-ecosystem idioms** (consult `community-defaults.md` for the current default before naming a library):
- **Rust**: `Stream`/`futures` + `tokio` tasks; `mpsc`/`broadcast` for the single-writer store and status; `rayon`/bounded `Semaphore` for CPU fan-out; `tokio::sync::mpsc` (bounded) for backpressure; `ArcSwap` for read-heavy snapshots.
- **Go**: goroutines + bounded `chan` per stage; `errgroup` for fan-out; a single goroutine owning serve state; context timeouts.
- **Elixir**: `GenStage`/`Flow`/`Broadway` (backpressure-native), `GenServer` for the store, supervisors for restart.
- **Python**: `asyncio` + bounded `Queue`; `asyncio.gather`/`TaskGroup` for fan-out; an owner task for serve state; `aiostream`/generators for streaming.
- **Node/TS**: streams with `highWaterMark` backpressure, `EventEmitter`/observable for status, worker threads for CPU work.
- **JVM**: Reactor/RxJava (backpressure), `BlockingQueue`-bounded stages, an actor (Akka) or single-thread executor for serve state.
- **F#/.NET**: `MailboxProcessor<Msg>` for the store, `Channel<T>` (bounded) + `IAsyncEnumerable` for streaming, `IObservable` for status.

**Size to the profile** — see `decision-frameworks.md` §13. A one-shot CLI importer may need only the port + per-record fault isolation + a stdout status line; a streaming desktop app or multi-tenant service needs the full composition. The *boundary*, the *status stream*, and *non-blocking UI* are non-negotiable wherever real I/O latency meets a user; everything else scales.

**Then hand to `app-harden`** to verify the runtime of whatever boundary you prescribe: resource ceilings on the queues/buffers, timeouts on every outbound call, retry-amplification caps, backpressure actually enforced under load, no secret leakage through the data path, crash-loop safety. arch-audit = where the boundary belongs; app-harden = whether it survives production.

---

## Paradigm adaptations

The smells above are OOP-flavored. Here are translations for other paradigms — apply during Phase 1 classification.

### Rust / Go (no classical inheritance)
- **God class → god module**: any package with >20 `pub` items without a clear single responsibility.
- **Deep/shallow → module depth**: same concept. A Rust module exporting one function that does a lot is deep; a module of 15 trivial wrapper functions is shallow.
- **Inheritance smells (LSP, parallel hierarchy) → trait/interface sprawl**: >5 trait bounds on a function, or multiple traits that always co-occur, suggest a missing combined trait or an over-abstracted trait.
- **Multi-writer state → `Arc<Mutex<…>>` proliferation**: count `Mutex`/`RwLock` usages. A handful is normal; dozens spread across modules signals shared-state coupling.
- **Cycles**: Go forbids import cycles at compile; any workaround (`internal/` tricks, interface-indirection-for-cycle-breaking-only) is a smell. Rust permits cycles within a crate; cycles across crates are forbidden, cycles in modules are tolerated but worth investigating.
- **Primitive obsession → raw `String`/`u64` in domain APIs**: same smell; newtype (`struct UserId(u64)`) is the fix.

### Functional (Haskell, Elixir, Clojure, F#)
- **God class → god module / "kitchen sink" module** exposing dozens of unrelated functions.
- **Feature envy → function that pattern-matches excessively on another module's data constructors**: the function belongs where the data type is defined.
- **Anemic model → type with no smart constructor** and validation scattered across call sites.
- **Effect-leak**: IO/state operations tangled with pure logic (a direct Hickey violation). Identify functions that do both; split into pure core + effectful shell.
- **Boolean-blind functions**: `foo(true, false, true)` — replace with sum types.
- **Typeclass/protocol sprawl**: the equivalent of trait sprawl. Too many narrow classes that always co-occur suggest a larger missing abstraction.

### Frontend (React, Vue, Svelte)
- **Mega-component**: >300 LOC JSX/template, or >10 props, or >5 useState/useReducer calls. Split by concern or extract hooks.
- **Prop drilling**: a prop passed through >3 component levels without use at intermediate levels. Lift state, use context, or colocate.
- **Context-as-global-state**: Context Providers used to share mutable state across unrelated trees — equivalent to global mutables.
- **useEffect dependency chains**: long dep arrays with multiple unrelated triggers; effects that re-run on unrelated changes. Split the effect per concern.
- **Barrel-file cycles**: `index.ts` barrels creating import cycles between feature directories. Flatten or remove the barrel.
- **Mixed server/client code**: in Next.js/Remix/SvelteKit, server-only code imported into client bundles. Audit with the framework's own linter.

### Data pipelines (Airflow, dbt, Spark, ETL)
> For *application-embedded* ingest→serve paths (a desktop/UI app, an API, or a service that reads data and exposes it), use **§ Data ingestion & serving subsystem** above — it covers streaming, backpressure, fault tolerance, non-blocking UI, and the lifecycle status stream. The bullets here are for *dedicated* orchestration tools.
- **DAG cycles**: self-evident bug; any cycle is fatal.
- **Fat task**: single task performing extraction + transformation + load; violates retriability and observability.
- **Unmanaged state between steps**: tasks sharing state via filesystem/Xcom/globals instead of typed outputs.
- **Schema drift**: downstream models depending on upstream columns without contract tests — any upstream change silently corrupts downstream.
- **Fan-out of downstream models on a single upstream**: high afferent coupling; changing the upstream schema breaks N things. Same fix as god-component — split or add a stable view.
- **Idempotency failures**: tasks that produce different output on re-run; trace via job-history diffs.

---

## Smells deliberately NOT on this list

Do not report these — they belong in linters, code-scrutiny, or nowhere:

- **Formatting, naming, style** → linter territory.
- **Missing comments / docstrings** → code clarity, not architecture.
- **Function length alone** without complexity or coupling → a 70-line pure function can be fine.
- **"Not using design pattern X"** → patterns are tools. Report the underlying smell, not the missing pattern.
- **Missing type annotations** in gradually-typed codebases → hygiene, not architecture, unless it causes real coupling.
- **"Code could be more idiomatic"** without a specific observable cost → vague and unactionable.

If the only thing you can say is one of these, do not put it in the report.
