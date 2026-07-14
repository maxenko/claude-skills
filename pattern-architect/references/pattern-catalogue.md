# Design Pattern Catalogue (GoF + Extended)

The full Gang of Four catalogue plus the extended/architectural patterns from common
references (GeeksforGeeks, Fowler). Each entry: **Intent**, the **smell** that motivates it,
**use when**, **avoid when / costs** (the over-engineering trap), **relations**, the
**before→after signal**, and **language idioms** (C#/.NET, F#, JS/TS) — because in functional
and modern languages many patterns collapse to a function, a module, or a discriminated union.

**Contents:**
- **Creational:** Factory Method · Abstract Factory · Builder · Prototype · Singleton · Object Pool · Lazy Initialization
- **Structural:** Adapter · Bridge · Composite · Decorator · Facade · Flyweight · Proxy
- **Behavioral:** Chain of Responsibility · Command · Iterator · Mediator · Memento · Observer · State · Strategy · Template Method · Visitor
- **Extended / architectural:** Dependency Injection · Repository · MVC / MV\*
- **Functional collapse cheat-sheet** (which GoF patterns dissolve into a function/module/DU)
- *Stateful, messaging & concurrency patterns live in `concurrency-patterns.md`.*

---

> Recurring meta-smell behind all CREATIONAL patterns: the `new ConcreteType()` literal
> hard-wired into business logic, binding callers to concretes they shouldn't know about.

---

# CREATIONAL

## Factory Method
- **Intent:** Define an interface for creating an object, but let subclasses/overrides decide which concrete class to instantiate.
- **Smell:** `new X()` calls and `switch`/`if` ladders on a type discriminator scattered across call sites; every new variant forces edits everywhere (Open/Closed violation).
- **Use when:** the exact product types aren't known up front; you want a library extension seam; you want to pool/cache expensive objects behind creation; the "what to create" decision varies by subclass.
- **Avoid when / costs:** needing a parallel creator hierarchy just to vary one product is heavy; for a fixed small set a `switch` or a dict-of-constructors is clearer; inheritance-based variation is rigid vs. passing a factory function.
- **Relations:** evolves into Abstract Factory / Prototype / Builder; a specialization of Template Method; Abstract Factory is built from Factory Methods.
- **Before→after:** `Transport t = type=="sea" ? new Ship() : new Truck()` repeated everywhere → `createTransport()` overridden per subclass.
- **Idioms:** *.NET* `Func<IProduct>` delegate or DI registration usually replaces it. *F#* pass a `unit -> Product` constructor function, or branch on a DU. *JS/TS* a factory function / `registry[type]()`.

## Abstract Factory
- **Intent:** Produce families of related objects without naming their concrete classes.
- **Smell:** related products (Button+Checkbox+Menu) come in matched variants (Win/Mac/Web) and you risk mixing incompatible members; concrete instantiation is scattered.
- **Use when:** code must stay independent of concrete classes across multiple product families; you must guarantee products used together share a variant; families are added over time.
- **Avoid when / costs:** interface + class explosion (one method per product × one factory per variant); adding a new *product type* forces changes to the factory interface and every implementation; overkill for one family.
- **Relations:** built from Factory Methods or Prototype; contrast Builder (returns step-by-step, later); concrete factories often Singletons; pairs with Bridge.
- **Before→after:** OS checks duplicated everywhere with latent mismatch bugs → `factory.createButton()` where `factory` is chosen once at boot.
- **Idioms:** *.NET* `IWidgetFactory` via DI, or a record of delegates. *F#* a record of functions `{ CreateButton: unit -> Button; ... }`. *JS/TS* an object literal of factory fns per variant.

## Builder
- **Intent:** Construct complex objects step by step; the same process can produce different representations.
- **Smell:** telescoping constructor / long optional parameter list; `new Pizza(size, cheese, pepperoni, …)` with most args null in any call.
- **Use when:** escaping a telescoping constructor; different representations share construction steps; building Composite trees; you want to validate/finalize only when fully assembled (immutability via `build()`).
- **Avoid when / costs:** extra classes not worth it for simple objects; if mostly required fields, a plain constructor or named params is simpler; the Director is often needless indirection.
- **Relations:** grows out of Factory Method; alternative to Abstract Factory for complex assembly; great for Composite; Director can be a Bridge abstraction.
- **Before→after:** `new House(4,2,true,false,null,…)` → `new HouseBuilder().walls(4).withGarage().build()`.
- **Idioms:** *.NET* fluent `Build()`, or object initializers / `with` on records. *F#* immutable records + `{ cfg with Walls = 4 }` and computation expressions (the canonical CE builder). *JS/TS* an options object usually replaces it; fluent chaining for query builders.

## Prototype
- **Intent:** Copy existing objects without depending on their concrete classes.
- **Smell:** you hold only an interface reference and can't call the constructor; or a type-checking ladder breaks encapsulation to copy private fields; or fresh-object init duplicates a lot of setup.
- **Use when:** code shouldn't depend on concrete types it copies; avoid a subclass-per-configuration explosion (clone a configured prototype); expensive init you can pre-build once; runtime-registerable presets via a registry.
- **Avoid when / costs:** circular references make cloning tricky; deep-vs-shallow bugs leak shared mutable refs; native structural copy often suffices.
- **Relations:** flexible alternative to Factory Method (no inheritance); can implement Abstract Factory; lighter than Memento for simple snapshots; registries often Singletons.
- **Before→after:** `if (s is Circle c) { var n = new Circle(); n.r = c.r; }` → `Shape copy = original.Clone()`.
- **Idioms:** *.NET* explicit `Clone()`, records give value-copy + `with`. *F#* mostly redundant — records/DUs are immutable with structural equality and `{ x with … }`. *JS/TS* `structuredClone()`, spread (shallow), `Object.create(proto)`.

## Singleton
- **Intent:** Ensure a class has exactly one instance with a global access point.
- **Smell:** a shared resource must exist once and a bare global is unsafe (anyone can overwrite/re-create it).
- **Use when:** exactly one instance is a genuine domain invariant (not convenience); you need stricter control than a bare global; lazy first-use init.
- **Avoid when / costs:** **widely an anti-pattern** — violates SRP (controls lifetime *and* access), is hidden global mutable state, masks excessive coupling, and is hostile to unit testing (resists mocking, leaks state across tests). Multithreading needs careful guarding.
- **Relations:** Facades/factories often Singletons; resembles Flyweight (but Flyweight allows many immutable shared instances). **Prefer DI** with a singleton-lifetime registration.
- **Before→after:** `static Config CONFIG` mutated from anywhere → an `IConfig` registered singleton-lifetime in DI and injected (single instance, but mockable + explicit). **Caveat:** DI singleton-lifetime fixes testability and explicitness, NOT mutability — a *mutable* singleton-scoped service is still shared state and still needs synchronization (or make it immutable, or an Actor).
- **Idioms:** *.NET* `services.AddSingleton<IFoo,Foo>()`; `Lazy<T>` / static readonly if truly needed. *F#* a `module` with `let`-bound values is a thread-safe lazy singleton; `lazy` for deferred init. *JS/TS* an ES module is a singleton by spec — `export const instance = …`.

## Object Pool *(extended — GfG)*
- **Intent:** Reuse a fixed set of expensive-to-create objects instead of allocating/freeing each time.
- **Smell:** hot-path code repeatedly creating and discarding costly objects (DB connections, threads, large buffers, sockets), thrashing allocation/GC.
- **Use when:** object creation is genuinely expensive *and* high-frequency; the object is reusable after a reset; you must cap concurrent live instances (connection limit).
- **Avoid when / costs:** premature without a measured allocation problem; pooled objects carrying stale state cause subtle bugs (reset discipline required); pool tuning (min/max/idle eviction) adds operational surface; in GC languages the allocator is often fast enough.
- **Relations:** a specialized Factory (the pool *is* the creation point); the Flyweight factory is a sibling (shares vs. lends out); the borrow/return shape resembles a Proxy guarding a resource.
- **Before→after:** `using var c = new SqlConnection(...)` per call → borrow from a managed pool, return on dispose.
- **Idioms:** *.NET* `ObjectPool<T>` (Microsoft.Extensions.ObjectPool), `ArrayPool<T>`, ADO.NET connection pooling (built in). *F#* same .NET pools; a `MailboxProcessor` lending resources. *JS/TS* a free-list array with `acquire`/`release`.

## Lazy Initialization *(extended — GfG)*
- **Intent:** Defer creating/computing a value until its first actual use.
- **Smell:** expensive fields built eagerly at construction that many code paths never touch; slow startup paying for things rarely used.
- **Use when:** the value is costly and not always needed; you want to break a construction-order dependency; caching a first-computed result (a memoized getter).
- **Avoid when / costs:** thread-safety required around first init (double-checked locking / a once-guard); hides latency to first use; "always eventually needed" values gain nothing.
- **Relations:** the mechanism behind a *virtual Proxy*; pairs with Flyweight/Object Pool factories; memoization is lazy init of a function result.
- **Before→after:** `_cache = BuildHugeIndex()` in the ctor → `_cache ??= BuildHugeIndex()` / `Lazy<T>` on first read.
- **Idioms:** *.NET* `Lazy<T>` (thread-safe), `??=`. *F#* `lazy expr` + `.Force()`; module `let` is already lazy-on-access at module load. *JS/TS* a getter that caches, or `let v; const get = () => v ??= compute()`.

---

# STRUCTURAL

## Adapter
- **Intent:** Let incompatible interfaces collaborate by wrapping one in a translation layer exposing the interface the client expects.
- **Smell:** working code speaks interface A; a third-party/legacy class you can't modify speaks interface B; call sites are littered with inline conversion glue.
- **Use when:** using an existing class whose interface doesn't match; integrating third-party/legacy code you can't change; converting data formats at a boundary.
- **Avoid when / costs:** adds an interface + class layer; if you control the service and can fix its interface directly, an adapter is over-engineering.
- **Relations:** **Bridge** is designed up front; Adapter retrofits. **Decorator** keeps the interface (and recurses); **Proxy** keeps the same interface; **Facade** defines a *new* simplified interface — Adapter *changes* one existing interface to another.
- **Before→after:** inline format-juggling at every call site → one `XmlToJsonAdapter` at the seam.
- **Idioms:** *.NET* thin wrapper class implementing the target interface. *F#* a module of mapping functions or an object expression `{ new ITarget with … }`. *JS/TS* a wrapper module re-exposing a foreign SDK under your shape.

## Bridge
- **Intent:** Split a class into two independent hierarchies — abstraction and implementation — so each varies separately.
- **Smell:** extending along two orthogonal axes with inheritance (Shape × Color) → combinatorial m×n subclass explosion.
- **Use when:** several orthogonal variation axes; you want to switch implementations at runtime; a platform-independent abstraction shielded from platform details; both dimensions will grow.
- **Avoid when / costs:** needless indirection on a class with only one real variation axis; premature if the second dimension never materializes.
- **Relations:** confused with Adapter (Bridge = planned, Adapter = retrofit); shares composition shape with State/Strategy; pairs with Abstract Factory and Builder.
- **Before→after:** `RemoteControl` subclassed per device brand → `RemoteControl` holds an `IDevice`; the two hierarchies evolve independently (m+n).
- **Idioms:** *.NET* abstraction takes `IImplementor` via ctor injection. *F#* a record/function carrying an implementation record-of-functions; partial application binds the implementation. *JS/TS* abstraction composed with an injected implementation object (renderer/transport splits).

## Composite
- **Intent:** Compose objects into trees and treat individual objects and whole compositions uniformly through one interface.
- **Smell:** nested part-whole hierarchies where an operation forces type-checks and explicit recursion that know concrete types and depth.
- **Use when:** the model is naturally a tree; clients should treat leaves and containers the same with no type-checking; add new element types without changing clients.
- **Avoid when / costs:** forcing one interface onto genuinely different leaves/containers overgeneralizes; if behavior diverges a lot, the shared interface gets awkward (no-op methods on leaves).
- **Relations:** Builder constructs the tree; Chain of Responsibility propagates through it; Iterator/Visitor traverse it; Flyweight shares leaf state; **Decorator** has a similar recursive shape but augments one child vs. aggregating many.
- **Before→after:** `getTotal()` with `instanceof` branches and manual recursion → `component.GetTotal()` recursing internally.
- **Idioms:** *.NET* `IComponent` with `Leaf` and `Composite : IComponent` holding `List<IComponent>`. *F#* a recursive DU `type Node = Leaf of int | Branch of Node list` + recursive fold — the canonical functional Composite. *JS/TS* nodes with a shared method, containers reducing over `children[]` (AST / virtual DOM).

## Decorator
- **Intent:** Attach behaviors to an object dynamically by wrapping it in decorators that share its interface.
- **Smell:** inheritance forces a compile-time combinatorial subclass explosion for every feature blend; runtime mix-and-match impossible.
- **Use when:** add/remove responsibilities at runtime without breaking clients; subclassing is impractical (sealed classes); arbitrary stackable optional behaviors; naturally layered logic (compress/encrypt/log around a stream).
- **Avoid when / costs:** removing a decorator from the middle of a stack is hard; behavior becomes order-dependent; many layers make wiring verbose.
- **Relations:** Adapter changes interface, Decorator keeps it; **Proxy** has the same structure but manages the service's lifecycle itself; **Chain of Responsibility** can stop the flow, a decorator never does; **Strategy** changes the guts, Decorator changes the skin.
- **Before→after:** a subclass per feature blend → `new CompressionDecorator(new EncryptionDecorator(stream))`.
- **Idioms:** *.NET* wrapper classes; idiomatically **middleware** (`app.Use`) and `Stream` wrappers (`GZipStream`). *F#* a **same-type wrapper** — `let logged next = fun req -> log req; next req` (type `Handler -> Handler`), with decorators chained as combinators (`(logged >> auth) baseHandler`). Note `f >> g` over the *payload* functions is pipeline composition, not decoration — the decorator preserves the wrapped type. *JS/TS* HOFs / middleware (Express, Redux), ES `@decorator`.

## Facade
- **Intent:** Provide a simplified, unified interface to a complex subsystem.
- **Smell:** using a subsystem requires initializing many objects and calling methods in a precise order, coupling business logic to implementation details.
- **Use when:** a simple entry point into a complex/legacy subsystem; reduce boilerplate at call sites; organize the system into layers with a facade per boundary.
- **Avoid when / costs:** can drift into a **god object** coupled to too much; may hide functionality some clients legitimately need (leaky bypasses).
- **Relations:** Adapter adapts *one* interface, Facade fronts a *whole subsystem* with a new interface; **Mediator** is bidirectional coordination, Facade is one-way simplification; often a Singleton.
- **Before→after:** client wiring `Codec`, `BitrateReader`, `AudioMixer` in sequence → `new VideoConverter().Convert("vid.ogg","mp4")`.
- **Idioms:** *.NET* an application-service / manager class wrapping internal services. *F#* a **module** exposing a few public functions over many private helpers — the module *is* the facade. *JS/TS* an index/barrel module or SDK client over internal plumbing.

## Flyweight
- **Intent:** Fit more objects in RAM by sharing the common (intrinsic) parts of state across many objects.
- **Smell:** an enormous number of similar objects each duplicating shared data (sprite/texture/color), exhausting memory.
- **Use when:** a massive number of similar objects *measurably* exhausts RAM; duplicate state can be cleanly extracted and shared; the memory win justifies the complexity.
- **Avoid when / costs:** **only an optimization** — premature without a measured RAM problem; trades RAM for CPU; the intrinsic/extrinsic split confuses newcomers.
- **Relations:** shared leaves inside a Composite; resembles Singleton (but many, distinguished by intrinsic state); factory often an Object Pool / cache.
- **Before→after:** `Tree { x,y,name,color,texture }` ×1M → shared `TreeType` via factory + `Tree { x,y,type→ }`.
- **Idioms:** *.NET* `Dictionary<key,Flyweight>` cache; string interning is a built-in flyweight. *F#* immutable records + a memoizing factory keyed on intrinsic fields. *JS/TS* a `Map`-backed factory of frozen shared objects.

## Proxy
- **Intent:** A substitute/placeholder controlling access to another object, running logic before/after requests.
- **Smell:** a heavyweight object you need intermittently; scattered lazy-init / auth / caching code duplicated across clients.
- **Types:** Virtual (lazy-init), Protection (auth), Remote (hide network), Logging, Caching, Smart-reference (release when unused).
- **Use when:** lazy init of a costly object; access control at the boundary; hiding remote-call plumbing; transparent caching/logging/ref-counting.
- **Avoid when / costs:** adds classes + indirection; the extra hop adds latency.
- **Relations:** Adapter changes the interface, Proxy keeps it identical; Decorator adds behavior (client-controlled) vs. Proxy controls access (manages lifecycle itself); Facade exposes a *new* simpler interface.
- **Before→after:** every caller does `if (service==null) service=new Expensive()` + inline auth → clients use `IService`; a `CachingServiceProxy` centralizes it.
- **Idioms:** *.NET* `DispatchProxy` / Castle DynamicProxy, EF Core lazy-loading proxies, `Lazy<T>`. *F#* a function/object-expression wrapping the real call (`memoize fetch`); `lazy`. *JS/TS* the built-in **`Proxy`** object with traps (Vue 3 / MobX reactivity), RPC client stubs.

---

# BEHAVIORAL

## Chain of Responsibility
- **Intent:** Pass a request along a chain of handlers; each handles it or forwards to the next.
- **Smell:** a bloated procedure of sequential duplicated `if`-checks (auth→validate→cache→throttle) growing brittler with every new check.
- **Use when:** varied request kinds whose types/sequence aren't known up front; the handler set/order must change at runtime; each step should be independently testable/reorderable.
- **Avoid when / costs:** a request can fall off the end unhandled (need a terminal handler); for a fixed short sequence a plain function is simpler; control flow hops across objects.
- **Relations:** handlers are often Commands and traverse Composites; shares "decouple sender from receiver" with Command/Mediator/Observer; structurally like Decorator but a handler may halt.
- **Before→after:** a giant method of stacked guard clauses → a list of small handlers assembled (and reordered) at the call site.
- **Idioms:** *.NET* ASP.NET Core middleware (`app.Use(next => …)`), `DelegatingHandler`. *F#* `Request -> Request option` functions folded together; Giraffe `HttpHandler`. *JS/TS* Express/Koa/Redux middleware.

## Command
- **Intent:** Turn a request into a standalone object with all its info — enabling parameterization, queuing, logging, undo.
- **Smell:** UI tightly coupled to business logic; near-identical control subclasses when the same operation is reached from toolbar+menu+shortcut.
- **Use when:** parameterize objects with an operation; queue/schedule/execute remotely; reversible operations (`undo()`); one operation from several entry points; audit/replay log.
- **Avoid when / costs:** extra layer between sender and receiver — overkill for a direct one-off with no undo/queue/log; many tiny command classes become boilerplate.
- **Relations:** pair with **Memento** for undo; CoR handlers can be Commands; **Strategy** swaps algorithms, Command reifies an operation+receiver; Visitor is a more powerful Command across a structure.
- **Before→after:** copy-pasted `onClick` business calls across triggers → a `CutCommand` instance wired to all three, pushed on an undo stack.
- **Idioms:** *.NET* `ICommand` (WPF `RelayCommand`), `Action`/`Func`, MediatR `IRequest`. *F#* a DU message type interpreted by one function — "command as data"; or a closure. *JS/TS* Redux **action objects**; `{ execute, undo }` records; thunks/sagas.

## Iterator
- **Intent:** Traverse a collection's elements without exposing its underlying representation.
- **Smell:** traversal stuffed into the collection class blurs its responsibility; clients coupled to one structure and one order.
- **Use when:** hide a complex internal structure; reduce duplicated traversal code; traverse different structures uniformly; multiple simultaneous independent traversals; alternate orders as swappable iterators.
- **Avoid when / costs:** overkill where a built-in loop suffices; can be less efficient than direct indexed access.
- **Relations:** traverses Composite trees; a Factory Method returns the matching iterator; pairs with Memento (snapshot iteration state) and Visitor (operate per element).
- **Before→after:** `collection.internalArray[i]` poking / reimplemented tree-walks → `foreach (var x in collection)`.
- **Idioms:** *.NET* `IEnumerable<T>`/`yield return`, LINQ, `IAsyncEnumerable<T>`. *F#* `seq { … }` / `Seq` (lazy). *JS/TS* iterator protocol `[Symbol.iterator]`, `function*`, `for…of`.

## Mediator
- **Intent:** Reduce chaotic many-to-many dependencies by making objects collaborate only through a mediator.
- **Smell:** tightly coupled peers calling each other directly form a tangled web; a component can't be reused because its coordination logic is bound to specific siblings.
- **Use when:** classes are hard to change due to tight coupling to many others; a component can't be reused (too many peer deps); interaction logic is complex and worth centralizing.
- **Avoid when / costs:** the mediator can swell into a **God Object** — the central risk; over-engineering for few stable interactions; a central bottleneck.
- **Relations:** **vs Observer** — Mediator centralizes peer coordination; Observer sets dynamic one-way subscriptions (Mediator may be *implemented* with Observer). **vs Facade** — Mediator replaces bidirectional peer comms; Facade is one-way simplification.
- **Before→after:** every dialog control toggling other controls → controls fire events at a `DialogMediator` owning the cross-control rules.
- **Idioms:** *.NET* **MediatR**, an in-process event aggregator. *F#* a central `MailboxProcessor` receiving a message DU and coordinating. *JS/TS* an event bus / Redux store / a parent React component lifting state.

## Memento
- **Intent:** Capture and restore an object's state without revealing its implementation.
- **Smell:** implementing undo tempts you to expose private fields; keeping them private blocks any external snapshot.
- **Use when:** snapshots to roll back (undo/history, transactions); direct field access would break encapsulation; checkpoints / rollback on failure.
- **Avoid when / costs:** frequent/large mementos eat RAM; caretakers must manage lifecycles or leak; for a simple object, Prototype (clone) may suffice.
- **Relations:** pair with **Command** for undo (caretaker = undo stack); combine with Iterator to snapshot iteration state.
- **Before→after:** `getInternalBuffer()/setInternalBuffer()` for an external `History` → `editor.save()` returns an opaque memento; internals stay private.
- **Idioms:** *.NET* an immutable snapshot `record` in a `Stack<T>`; nested private class for opacity. *F#* records/DUs *are* mementos — undo history = a list of past states (persistent data structures make it nearly free). *JS/TS* frozen state pushed on an undo stack; Redux time-travel / Immer.

## Observer
- **Intent:** A subscription mechanism notifying multiple objects automatically of events on the object they observe.
- **Smell:** a state change must reach an unknown dynamic set of dependents — naive code either polls or hard-wires the subject to every dependent.
- **Use when:** a change to one object must update an unknown number of others; the dependent set changes at runtime; GUI/event systems; broadcast without the source depending on receiver types.
- **Avoid when / costs:** notification order is unpredictable; hidden update cascades obscure control flow; **memory leaks (lapsed-listener)** if you forget to detach; overkill for one fixed dependent.
- **Relations:** Mediator centralizes (many-to-many hub); Observer is dynamic one-way. The substrate under Event Aggregator and Reactive streams (see concurrency-patterns.md §3–5).
- **Before→after:** polling loops / a subject manually calling each dependent → a generic subscriber list + `notify()`.
- **Idioms:** *.NET* `event`/`EventHandler<T>`, `IObservable<T>` + Rx, `INotifyPropertyChanged`. *F#* `IEvent<_>` / `Observable` (`subscribe`/`map`/`filter`) as first-class values. *JS/TS* `EventTarget`, Node `EventEmitter`, RxJS `Subject`, store subscriptions.

## State
- **Intent:** Let an object alter its behavior when its internal state changes — it appears to change class.
- **Smell:** bulky `switch`/`if` on a status enum smeared across many methods; states multiply; transition logic duplicated.
- **Use when:** very different behavior across many states that changes often; a class polluted with state conditionals; duplicated transition code; a genuine finite state machine (workflow, connection lifecycle, document status).
- **Avoid when / costs:** a handful of stable states — an enum + switch is clearer; **state-class explosion**; transitions scattered across state classes obscure the machine (a transition table may read better).
- **Relations:** **State ≅ Strategy structurally** (composition + delegation) but State objects know about and trigger transitions to each other; shares the skeleton of Bridge/Adapter.
- **Before→after:** a giant `switch (this.status)` repeated across methods → a `currentState` object; transition = reassign the reference.
- **Idioms:** *.NET* `IState` classes, or enum + switch-expression for simple machines. *F#* the **idiomatic fit** — a DU for states + a pure `transition: State -> Event -> State`; pattern matching replaces the class hierarchy. *JS/TS* a TS discriminated union + reducer (XState formalizes it).

## Strategy
- **Intent:** Define a family of interchangeable algorithms, encapsulate each, make them swappable at runtime.
- **Smell:** a class bloated with conditionals choosing among algorithm variants; every variant grows the class.
- **Use when:** switch algorithm variants at runtime; many similar classes differing only in one behavior; isolate algorithm from plumbing; a massive conditional selecting variants of one operation.
- **Avoid when / costs:** a couple of stable algorithms — the extra interface is ceremony; **in modern languages a plain lambda usually suffices**; clients must understand variants to choose.
- **Relations:** State has the same structure but its objects transition; Decorator changes the skin, Strategy the guts; **Template Method** does the same via inheritance (compile-time) vs. Strategy via composition (runtime).
- **Before→after:** `if (mode==X) sortByX() else sortByY()` → `context.setStrategy(s); context.execute()`.
- **Idioms:** *.NET* an `IStrategy`, or idiomatically a **`Func<TIn,TOut>`** (LINQ comparers, DI delegates). *F#* **collapses to a first-class function** passed as a parameter; a partially-applied function *is* the strategy. *JS/TS* pass a function / a map of named functions.

## Template Method
- **Intent:** Define an algorithm skeleton in a base class, letting subclasses override specific steps without changing its structure.
- **Smell:** several classes implement nearly the same algorithm with minor differences (duplicated code); clients need conditionals to pick the class.
- **Use when:** clients should extend only particular *steps*, not the structure; multiple classes share an almost-identical algorithm; consolidate duplicated skeletons; replace type-switching with polymorphism.
- **Avoid when / costs:** inheritance locks clients into the skeleton; can **violate LSP** if a subclass suppresses a step; if steps must vary *at runtime*, prefer Strategy.
- **Relations:** Factory Method is a specialization and often a *step* inside one; **vs Strategy** — inheritance/static (override steps) vs. composition/runtime (swap whole algorithm).
- **Before→after:** copy-pasted near-duplicate methods + client `switch`-by-type → one base `templateMethod()` calling overridable steps.
- **Idioms:** *.NET* `abstract` base with a non-virtual template calling `protected abstract` steps + `virtual` hooks. *F#* a **higher-order function taking the varying steps as function params** — the functional dual. *JS/TS* `abstract class`, or a function accepting step callbacks / an options object of hooks.

## Visitor
- **Intent:** Separate algorithms from the object structure they operate on.
- **Smell:** you must add new operations across an existing (stable) class hierarchy without modifying those classes; auxiliary behaviors pollute element classes.
- **How (double dispatch):** each element's `accept(visitor)` calls back `visitor.visitConcreteElement(this)`; dispatch on both element and visitor type → no `switch` on type.
- **Use when:** an operation over all elements of a complex structure (AST); extract unrelated behaviors out of element classes (SRP); you'll add *many* operations over a *stable* element set.
- **Avoid when / costs:** **every visitor must change when an element class is added** — wrong trade if element types change often; visitors may lack access to private members; accept/visit boilerplate.
- **Relations:** traverses Composite; pair with Iterator (walk) + Visitor (operate); a more powerful Command across many element classes.
- **Before→after:** a sprawling `if (e is A)… else if (e is B)…` repeated per operation → `accept()` + a visitor per operation.
- **Idioms:** *.NET* classic `IVisitor`, or modern pattern matching / `switch` over a sealed hierarchy. *F#* **subsumed by DUs + exhaustive `match`** — a new operation is just a new function; the compiler enforces exhaustiveness. **The expression-problem trade (pick by which axis varies):** DU+match makes adding *operations* easy but a new *variant* forces editing every existing match (and external code can't add variants); class-based Visitor is the dual — easy to add element types, every new operation touches every visitor. *JS/TS* explicit `accept`/`visit`, or a discriminated union + `switch` on `kind` with a `never` exhaustiveness check.

---

# EXTENDED / ARCHITECTURAL *(GfG — not GoF)*

> These operate at a larger granularity than the GoF object patterns. Recognize them as
> existing structure to *complement*, and recommend them only when a whole layer's smell
> (data access tangled into domain logic, hidden dependencies, mixed UI/logic) justifies it.

## Dependency Injection
- **Intent:** Provide a component's dependencies from outside rather than having it construct or locate them.
- **Smell:** classes `new`-ing their collaborators or reaching for global singletons → untestable, hidden coupling, rigid wiring.
- **Use when:** you want testable seams (inject fakes), swappable implementations, and lifecycle management (singleton/scoped/transient) owned in one composition root.
- **Avoid when / costs:** a container is overkill for a tiny app (poor-man's DI — constructor params — is enough); over-abstracting every concrete into an interface produces shallow indirection; runtime-resolution magic hides the object graph.
- **Idioms:** *.NET* the built-in `IServiceCollection` / `IServiceProvider`. *F#* partial application *is* DI — pass dependencies as function arguments; the "Reader" pattern / composition root wiring. *JS/TS* constructor params, factory functions, or a container (InversifyJS).

## Repository
- **Intent:** Mediate between the domain and data-mapping layers, exposing a collection-like interface over persistence.
- **Smell:** SQL / ORM queries scattered through domain and UI code; business logic coupled to a specific database/ORM; no seam to test domain logic without a DB.
- **Use when:** you want to centralize data-access logic, swap the persistence backend behind an interface (the IStore seam), and unit-test domain logic against an in-memory fake.
- **Avoid when / costs:** a thin pass-through over an already-abstracted ORM (e.g. EF `DbSet` is already a repository) adds a **Middle Man**; generic `IRepository<T>` often leaks query concerns and becomes a Lazy Class.
- **Idioms:** *.NET* an `IFooRepository` over EF/Dapper. *F#* a record of data-access functions or an interface (the `IStore` seam pattern). *JS/TS* a data-access module exposing domain-shaped queries.

## MVC / MV* (MVVM, MVP)
- **Intent:** Separate presentation into Model (state/rules), View (rendering), and Controller/Presenter/ViewModel (input + coordination).
- **Smell:** UI components holding business rules and data access inline (fat controllers / god components) — untestable logic welded to rendering.
- **Use when:** a UI app needs testable logic independent of rendering; clear ownership of state vs. view vs. coordination.
- **Avoid when / costs:** ceremony for trivial UIs; "fat controller" / "fat view-model" reintroduces the smell one layer over; anemic models (Data Class smell) when all behavior migrates to controllers.
- **Idioms:** *.NET* ASP.NET MVC, WPF/MAUI MVVM (`INotifyPropertyChanged` + binding). *JS/TS* React = roughly View + ViewModel via hooks/stores; keep domain logic in stores/managers, not components.

---

## Functional collapse cheat-sheet
In F#, modern C#, and JS, many GoF patterns dissolve into language features — recognize these so you don't recommend ceremony the language already gives for free:

| GoF pattern | Functional / modern form |
|---|---|
| Strategy | a higher-order function (lambda) |
| State / Visitor | discriminated union + exhaustive `match` |
| Template Method | a function taking step callbacks |
| Command | a DU "message" interpreted by one function, or a closure |
| Observer | `IObservable`/Rx / RxJS streams, events |
| Factory Method | a `unit -> T` constructor function / DI |
| Singleton | a module (ES module / F# `module`) |
| Decorator | function composition `f >> g` / middleware |
| Memento | an immutable snapshot value (records are mementos) |
| Prototype | `{ x with … }` copy-update / `structuredClone` |
| Iterator | `seq`/`yield`/generators |
