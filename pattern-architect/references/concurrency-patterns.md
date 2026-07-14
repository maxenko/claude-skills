# Stateful / Concurrency / Messaging Patterns

Patterns beyond the Gang of Four for structuring **long-lived state**, **concurrency**, and
**decoupled messaging**. These are the patterns named "Stateful Managers, Mailbox Processors,
Events, Subscriptions." Idioms target a polyglot stack: F#/.NET, JS/React, Python.

**The throughline:** every pattern here trades a *visible* coupling problem for an *invisible*
one — leaked subscriptions, starved mailboxes, untraceable event flow, replay side effects,
eventual-consistency staleness. Adopt the smallest one that fixes the actual smell, **bound
your queues, and pair every `subscribe` with an `unsubscribe`.**

**Contents:** 1. Actor / Mailbox Processor · 2. Stateful Manager / Service · 3. Event Aggregator / Bus ·
4. Publish–Subscribe / Observer / Subscriptions · 5. Reactive Streams / Observable (Rx) ·
6. CQRS & Event Sourcing · 7. Message / Command Queue + Worker · 8. State Machine · then the
**quick selection guide** table. Each entry: intent, smell, use/avoid, **lifecycle hazards**, idioms, relations.

---

## 1. Actor Model / Mailbox Processor
- **Intent:** Serialize all access to mutable state behind a private message queue so exactly one logical thread mutates it at a time.
- **Smell it solves:** shared mutable state guarded by `lock`/`Monitor`/`Mutex` scattered everywhere — lock-ordering deadlocks, forgotten locks, torn reads. The actor replaces *locks on data* with *a queue of messages*; the state is never shared, so there is nothing to lock.
- **How it works:** an actor owns private state + a mailbox (async queue), pulls one message at a time, processes to completion, and threads a new state value into the next loop iteration. Serial handling makes races structurally impossible *inside* the actor.
- **Use when:** one logical resource (counter, cache, connection, registry, session) is mutated by many concurrent callers; you model independent long-lived entities (one actor per session/topic/device); you need a seam to serialize ordered side effects.
- **Avoid when / costs:**
  - **Mailbox starvation hides as a "hang"** — a slow handler stops draining while the queue grows; F# `MailboxProcessor` has **no built-in backpressure** (fast producer + slow actor = unbounded memory).
  - **Synchronous reply between two actors can deadlock** (A waits on B, B waits on A) — the lock deadlock returns as two stalled mailboxes.
  - **Bare `MailboxProcessor` has no supervision** — an unhandled exception silently kills the agent; `Post` keeps succeeding into a dead mailbox.
  - Over-actoring trivial state (an actor to guard one bool) is ceremony — `Interlocked`/`lock` is simpler.
- **Lifecycle hazards:** in-process `Post` is a *reliable but non-durable* enqueue (it doesn't silently drop into a live mailbox — but everything is lost on crash, and `Post` to a dead agent succeeds into the void); ordering is per-sender, not global; graceful shutdown needs an explicit `Stop` message / `CancellationToken`; restart loses state unless externalized.
- **Idiom (F# request/reply):**
```fsharp
type Msg = Incr of int | Get of AsyncReplyChannel<int> | Stop
let agent = MailboxProcessor.Start(fun inbox ->
    let rec loop state = async {
        let! msg = inbox.Receive()
        match msg with
        | Incr n    -> return! loop (state + n)
        | Get reply -> reply.Reply state; return! loop state
        | Stop      -> return () }
    loop 0)
agent.Post (Incr 5)                 // fire-and-forget
let v = agent.PostAndReply Get      // request/reply
agent.Error.Add (fun ex -> log.Error ex)   // bare MailboxProcessor swallows exceptions — always trap
```
- **When to escalate:** reach for **Akka.NET** (supervision hierarchies — Resume/Restart/Stop/Escalate, DeathWatch, backoff restarts) or an Erlang/OTP-style `gen_server` precisely when you need supervision + restart that raw `MailboxProcessor` lacks.
- **Relations:** the actor's `match` on a message is often a **State machine** (§8); composes Strategy (handler per message) and acts as a Mediator; a pool of identical actors behind one queue is the **Worker** (§7).

## 2. Stateful Manager / Service
- **Intent:** A single long-lived object owning one domain's state + lifecycle (load, mutate, persist, dispose) behind a cohesive API.
- **Smell it solves:** state and its governing logic smeared across call sites — every caller reads/writes raw fields, invariants enforced nowhere. A Manager pulls state behind one boundary so invariants and lifecycle live in one place.
- **How it works (well):** expose **intent-level methods** (`SubscribeToTopic`, `MarkRead`), not getters/setters on internal collections; state is private; dependencies **injected**, not global; explicit lifecycle (start/stop, connect/disconnect); does *one* domain.
- **Use when:** a domain has real state + lifecycle (connections, caches, subscriptions, sessions); you want one authoritative place for an invariant; a frontend needs a singleton coordination point React's per-component state can't hold.
- **Avoid when / costs (the god-singleton trap):**
  - **The god object** — a Manager that grows to own unrelated domains becomes the thing "everything touches everything" through. Split by domain; cohesion is the test.
  - **Global singletons are hidden global mutable state** — untestable (no fake seam), init-order coupling, hidden deps. Prefer one *instance injected via DI* over a `static` global.
  - **Concurrency is now your problem** — a Manager touched from multiple threads needs synchronization; at that point consider making it an **Actor** (§1) so serialization is structural.
- **Lifecycle hazards:** Managers holding subscriptions **leak** if they never release them (§4); a frontend module-level singleton accumulates handlers across mounts; reconnect (socket dropped → rejoin rooms) must be explicit and idempotent.
- **Idioms:** *.NET* register as a **DI singleton** (not `static`), implement `IAsyncDisposable`; background loop → `IHostedService`. *F#* a module with one `MailboxProcessor` *is* a thread-safe Manager. *JS* a singleton class holding subscriptions in a `Map`; every `subscribe` returns a disposer.
- **Relations:** a good Manager composes Strategy (pluggable policy), Observer (notifies subscribers — §4), and Facade (one API over collaborators). Promote a contended Manager to an Actor (§1).

## 3. Event Aggregator / Event Bus / Mediator-as-Bus
- **Intent:** Channel events from many sources into one object so clients register once instead of with every source.
- **Smell it solves (Fowler):** lots of potential event sources force a client to subscribe to each — an O(sources×clients) wiring tangle. The aggregator subscribes to all sources and re-exposes itself as the sole registration point → O(sources+clients).
- **How it works:** the aggregator subscribes to every source and passes events through (or generalizes them); clients subscribe once, to the aggregator. (A Mediator *knows* its colleagues and coordinates; an aggregator is anonymous — publishers don't know subscribers exist.)
- **Use when:** many-to-many component communication; you want to break a compile-time publisher→consumer dependency (a frontend event bus decoupling sibling React trees without prop-drilling); cross-cutting notifications (auth changed, connection lost).
- **Avoid when / costs:**
  - **Control flow becomes invisible** — "publish `UserBanned`" has no static call graph to a handler; debugging is grep-driven.
  - **Generalized events deliver noise** — broadened subscriptions get irrelevant notifications, forcing client-side filtering.
  - inherits all Observer pitfalls incl. **leaks if subscriptions aren't released** (§4); for 2–3 known collaborators a direct call or Mediator is clearer.
- **Lifecycle hazards:** the aggregator references every subscriber → **a leak engine** without unsubscribe-on-teardown; ordering across publishers is not guaranteed; a handler that publishes during dispatch can recurse / mutate the list mid-iteration — snapshot listeners before dispatch.
- **Idioms:** *JS* a tiny `emitter` (`on`/`off`/`emit`) or `mitt`; **every `on` paired with `off`** in `useEffect` cleanup. *.NET* MediatR notifications, Rx `Subject<T>`. *F#* an `Event<'T>` from a Manager, or a `MailboxProcessor` fanning out.
- **Relations:** it *is* Observer (§4) with a facade in front; often paired with a Manager (§2) that owns the bus; contrast Mediator (named colleagues) and Reactive streams (§5).

## 4. Publish–Subscribe / Observer / Subscriptions
- **Intent:** Let an object notify a dynamic, unknown set of dependents when it changes, without knowing who they are.
- **Smell it solves:** a producer hard-coding calls to each consumer — adding one means editing the producer. Observer inverts this: consumers register; the producer just announces.
- **How it works:** subjects keep a subscriber list and call them on change; pub/sub adds a broker/topic so publisher and subscriber never reference each other; subscription returns a **handle/disposer** whose whole job is to let the subscriber leave.
- **Use when:** one change fans out to a variable number of parties; producer/consumer must be independently testable; live UI (a store notifies components on change).
- **Avoid when / costs:**
  - **Leaks are the default failure mode** (see hazards) — the biggest operational cost.
  - **No backpressure in naïve pub/sub** — a fast publisher overruns a slow subscriber; use a bounded queue (§7) or Rx backpressure if rates diverge.
  - update storms / diamond re-entrancy — push-based Observer has no glitch-freedom.
- **Lifecycle hazards (the core of this pattern):**
  - **Unsubscribe leak** — if the emitter outlives the subscriber, the subject holds a strong reference; the dead subscriber is never GC'd and its handler keeps firing (often throwing on unmounted state).
  - **Reconnect/rejoin** — after a socket reconnect you must re-subscribe; subscriptions aren't auto-restored, and naive resubscribe double-registers.
  - **Delivery** — in-memory = at-most-once, no replay; a missed event is gone unless you add a durable log (§6/§7). Ordering holds per-subject, not across subjects.
- **Idiom (React — every subscription returns its cleanup):**
```jsx
useEffect(() => {
  const ctrl = new AbortController();
  store.on("change", onChange, { signal: ctrl.signal });
  socket.subscribe(topic, onMsg);
  return () => { ctrl.abort(); socket.unsubscribe(topic); };  // unmount AND before re-run
}, [topic]);                                                   // re-subscribe when topic changes
```
- *.NET* `IObservable<T>.Subscribe` returns `IDisposable` — **dispose it**; `event` needs `-=` on teardown. *F#* `IEvent<'T>.Subscribe` returns `IDisposable` — keep + dispose the handle.
- **Relations:** the substrate under Event Aggregator (§3) and Reactive (§5); a Manager (§2) usually *is* the subject; pairs with a Worker queue (§7) for durability/backpressure.

## 5. Reactive Streams / Observable (Rx)
- **Intent:** Model an async, time-varying sequence of values as a first-class, composable object transformed with operators.
- **Smell it solves:** "callback hell" — nested, manually-sequenced async callbacks with hand-smeared subscription bookkeeping. Rx extends Observer to treat event streams like collections, so you `filter`/`map`/`merge`/`debounce` declaratively.
- **Use when:** event streams needing composition (debounce a search box, merge socket+poll, retry-with-backoff, `switchMap` to cancel stale requests); you'd otherwise hand-roll fragile timer/callback/flag chains.
- **Avoid when / costs:** steep learning curve + invisible control flow; a `flatMap` vs `switchMap` vs `concatMap` bug is subtle and the stack trace won't help; **backpressure** hazard in push models; **subscription leaks** like Observer; for a single async result, `Task`/`Promise` is simpler.
- **Lifecycle hazards:** hot vs cold (cold re-runs the producer per subscriber → duplicate side effects; hot shares → late subscribers miss earlier values); always dispose (or `takeUntil(unmount$)`); errors **terminate** the stream — retry must be explicit.
- **Idioms:** *.NET* `System.Reactive` over `IObservable<T>`. *JS* RxJS `Observable`/`Subject`. *F#* the `Observable` module over `IEvent<'T>`.
- **Relations:** Rx is Observer (§4) + Iterator + an operator algebra; a richer alternative to a plain Event bus (§3); a push-based cousin of Channels (§7, pull-based).

## 6. CQRS & Event Sourcing
- **Intent:** *CQRS* — use a different model to *write* (commands) than to *read* (queries). *Event Sourcing* — store every state change as an immutable ordered event; current state is a left-fold over the log.
- **Smell it solves:** *CQRS* — one shared model serving reads and writes that (Fowler) "does neither well." *Event Sourcing* — loss of history in CRUD overwrites (no "state last Tuesday", no audit, no rebuild).
- **Use when:** a bounded context where read/write shapes or scales genuinely diverge; strong audit/compliance, debug-by-replay, or temporal queries (Event Sourcing's sweet spot); task-based UIs capturing intent as commands.
- **Avoid when / costs (read twice):** Fowler — **"for most systems CQRS adds risky complexity"**; apply only to specific bounded contexts. **Event Sourcing is a large, hard-to-reverse commitment** — killers are external side effects on replay, event-schema versioning over years, and the operational weight of projections + eventual consistency. For plain CRUD with no audit/temporal/scaling driver, both are overkill.
- **Lifecycle hazards:** eventual consistency (read model lags write — UIs tolerate stale reads); replaying events that sent emails/payments re-fires them unless gated behind a replay flag; snapshotting needed once logs grow.
- **Idioms:** *.NET* MediatR to split command/query handlers (CQRS *without* sourcing — the cheap, sane subset); Marten (Postgres) for event streams + projections — Marten's projection model *is* a CQRS read-side. *F#* commands/events as DUs; `apply: State -> Event -> State` fold; `decide: State -> Command -> Event list`.
- **Relations:** Event Sourcing produces events that Observer/Pub-Sub (§4) and Workers (§7) consume; the `apply` fold is a State machine (§8); adopt CQRS alone first, add sourcing only if audit/temporal justifies it.

## 7. Message / Command Queue + Worker
- **Intent:** Decouple "work requested" from "work performed" via a queue drained by worker(s) — smoothing load, enabling retry/durability.
- **Smell it solves:** slow/risky work (transcode, moderation, email) done **inline on the request thread** — the user waits, a failure fails the whole request, a spike melts the server because nothing buffers arrival rate vs. processing rate.
- **How it works:** producers enqueue; a worker loop dequeues and processes (sequentially or a bounded pool); a **bounded** queue gives backpressure (full → producer waits or sheds). The "LISTEN/NOTIFY + poll" shape uses Postgres as a durable queue — a row is the job, `NOTIFY` wakes the worker, which `SELECT … FOR UPDATE SKIP LOCKED` claims it, surviving restarts.
- **Use when:** slow/retryable/spiky work off the request path; you need **durability** (jobs survive a crash — in-memory actors/observers can't promise this); throttle a downstream to fixed concurrency; decouple producer/consumer scaling.
- **Avoid when / costs:**
  - **At-least-once means duplicates** — workers must be **idempotent** or you double-charge/double-post.
  - an **unbounded** queue is a memory bomb hiding a too-slow consumer until OOM — always bound and choose the full-mode policy deliberately.
  - DB-as-queue works at modest scale but contends with OLTP; past a threshold use a real broker (RabbitMQ/Redis/SQS) — but don't adopt one prematurely.
  - poison messages need a dead-letter path + max-retry.
- **Lifecycle hazards:** graceful shutdown must drain/abandon in-flight jobs (honor `CancellationToken`; don't ack until done); lease/visibility timeout so a dead worker's job is reclaimed; retry = exponential backoff + cap + dead-letter; ordering usually not guaranteed across workers.
- **Idiom (.NET bounded `Channel<T>` + `BackgroundService`):**
```csharp
var ch = Channel.CreateBounded<Job>(
    new BoundedChannelOptions(1000) { FullMode = BoundedChannelFullMode.Wait });  // real backpressure
await ch.Writer.WriteAsync(job, ct);                                              // awaits when full
public sealed class Worker(ChannelReader<Job> reader) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stop) {           // statement body: await foreach is not an expression
        await foreach (var job in reader.ReadAllAsync(stop))
            await ProcessWithRetryAsync(job, stop);                                // idempotent + backoff
    }
}
```
A single-consumer channel *is* an actor's mailbox (§1); `Channel<T>` is the pull-based dual of Rx's push (§5).
- *F#* a `MailboxProcessor` worker, or `FOR UPDATE SKIP LOCKED` + `LISTEN/NOTIFY` for durability. *Python* a worker loop pulling from a DB/Redis queue; idempotency via a processed-id set / unique constraint.
- **Relations:** a worker draining a queue is an Actor (§1) with a durable bounded mailbox; consumes events from Event Sourcing (§6) and Pub-Sub (§4); the bounded queue is the backpressure §4/§5 lack.

## 8. State Machine
- **Intent:** Make legal states and transitions explicit so impossible states are unrepresentable and "what happens next" is a function, not a tangle of flags.
- **Smell it solves:** a soup of booleans (`isLoading`, `isConnected`, `hasError`, `isRetrying`) where meaningless combinations are reachable, and every method second-guesses the condition with nested `if`s.
- **How it works:** define the finite states + a `transition: State -> Event -> State` that is the *only* way state changes. In F# the states become a **DU**; the compiler forces handling every case — illegal states are uninhabited, not runtime checks.
- **Use when:** connection/session lifecycle (`Disconnected→Connecting→Connected→Reconnecting→Closed`); workflow/moderation status with strict allowed transitions; anything currently ≥3 interacting booleans; protocol/parser phases.
- **Avoid when / costs:** stateless transforms gain nothing; a combinatorial explosion of states signals wrong decomposition (consider hierarchical states / splitting); the table drifts if side effects sneak inside it — keep effects at the edges (decide → effect), not in the fold.
- **Lifecycle hazards:** entry/exit effects (open socket on `Connecting`) must be idempotent and not double-fire; persisted state must restore to a *valid* state; concurrent events racing the transition need serialization — wrap the machine in an Actor (§1).
- **Idiom (F# — illegal states unrepresentable):**
```fsharp
type Conn = Disconnected | Connecting of attempt:int | Connected of sid:string | Reconnecting of attempt:int
type Ev   = Open | Established of string | Dropped | Close
let transition state ev =
    match state, ev with
    | Disconnected,   Open           -> Connecting 1
    | Connecting _,   Established sid -> Connected sid
    | Connected _,    Dropped        -> Reconnecting 1
    | Reconnecting n, Dropped        -> Reconnecting (n + 1)
    | _,              Close          -> Disconnected
    | s, _ -> s                                  // ignore illegal event — a DELIBERATE choice; for protocols/parsers
                                                 // prefer an explicit Error state or a logged rejection so bugs aren't swallowed
```
- *.NET* `enum` + transition method, the **Stateless** library; *JS* XState (statecharts), or a reducer (`useReducer` *is* a state machine).
- **Relations:** the natural inside of an Actor (§1); the `apply`/`decide` fold of Event Sourcing (§6) is a state machine; replaces ad-hoc Strategy selection when the active strategy depends on a lifecycle phase.

---

## Quick selection guide
| If your problem is… | Reach for | Not |
|---|---|---|
| Locks everywhere around shared state | **Actor** (§1) | another `lock` |
| One domain's state + lifecycle to own | **Manager** (§2), DI singleton | a `static` god object |
| Many sources × many listeners wiring | **Event Aggregator** (§3) | direct refs everywhere |
| Fan-out notification on change | **Observer/Pub-Sub** (§4) | hard-coded callbacks |
| Composing/transforming async streams | **Reactive/Rx** (§5) | nested callbacks |
| Read & write models genuinely diverge | **CQRS** (§6, one context) | system-wide CQRS |
| Need audit / temporal / rebuild | **Event Sourcing** (§6) | for plain CRUD |
| Slow/spiky/durable/retryable work | **Queue + Worker** (§7) | inline on request thread |
| ≥3 interacting boolean flags | **State machine** (§8) | flag soup |
