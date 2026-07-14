# Pattern Relationships, Disambiguation & Over-Engineering Guardrails

Three catalogues for applying patterns **surgically**: which patterns reinforce each other,
which are confused for each other, and — most important — when NOT to pattern at all.

---

## 1. Patterns that combine well (synergies)

| Combination | Why they work together |
|---|---|
| **Abstract Factory + Factory Method (+ Singleton)** | Abstract Factory is implemented *with* Factory Methods; the concrete factory is often one shared instance. One swappable family-creator, one allocation point. |
| **Composite + Iterator** | A recursive tree (Composite) + uniform traversal without exposing leaf-vs-branch. |
| **Composite + Visitor** | Add new *operations* over a Composite's node types without editing every node class — the AST answer. |
| **Composite + Iterator + Visitor (triad)** | Structure + uniform walk + open-ended operations: the interpreter/AST toolkit. |
| **Command + Memento (+ Composite)** | Command encapsulates the action; Memento captures pre-state to undo it; a Composite of Commands = macros. The canonical undo stack. |
| **Strategy + Factory Method** | The variation is *which* Strategy; a Factory Method decides and instantiates it, keeping selection in one place. |
| **State + Strategy** | Near-identical structure; State objects often *are* internal strategies that also trigger transitions. |
| **Decorator + Composite** | Both recursive same-interface compositions; decorators add behavior to a node, composites aggregate children — a tree of individually-augmented nodes. |
| **Decorator + Strategy** | Decorator changes the skin (wrap), Strategy the guts (swap algorithm) — layered behavior + pluggable core. |
| **Observer + Mediator** | A Mediator centralizes many-to-many coupling and frequently *uses* Observer to broadcast — the event-bus / pub-sub hub. |
| **Facade + Adapter** | At a subsystem boundary: Facade simplifies the whole subsystem; Adapters reshape the incompatible pieces underneath. |
| **Builder + Composite** | A Builder assembles a Composite tree step-by-step, hiding the recursion from the client. |
| **Flyweight + Factory (Pool)** | The factory maintains the shared pool and returns cached intrinsic-state instances. |
| **Template Method + Factory Method** | Factory Method is a specialization; a template's fixed skeleton calls factory hooks for the variable creation step. |
| **Chain of Responsibility + Command** | Each handler processes/forwards a request reified as a Command — the middleware/interceptor pipeline. |
| **Chain of Responsibility + Composite** | A handler is a node in a tree; a request bubbles up parent links until handled (event bubbling). |
| **Bridge + Abstract Factory** | Bridge separates abstraction from implementation; an Abstract Factory creates and pairs the right implementor per platform. |
| **Manager + Strategy + Observer + Facade** | A well-built **Stateful Manager** typically composes all three: pluggable policy (Strategy), notifies subscribers (Observer), one API over collaborators (Facade). |
| **Actor + State machine** | The actor serializes events; the state machine decides transitions — the disciplined stateful-entity shape. |

## 2. Patterns frequently confused (disambiguation)

**Strategy vs State** — same delegation skeleton, different intent. **Strategy** when the *client* picks one of several interchangeable algorithms that don't know about each other. **State** when behavior changes with lifecycle and the *states themselves* decide the next transition.

**Decorator vs Proxy vs Adapter vs Facade** — all wrap something:
- **Adapter** *changes* the interface of an existing object (you don't control it).
- **Decorator** *keeps* the interface and *adds* responsibilities; designed for recursive stacking.
- **Proxy** keeps the interface but *controls access* (lazy/remote/cache/auth); manages the subject's lifecycle.
- **Facade** defines a *new, simpler* interface over a whole subsystem; not recursive.

**Factory Method vs Abstract Factory** — Factory Method: one overridable method, one product, via inheritance. Abstract Factory: an object with several creation methods producing a *family*, via composition. Reach for Abstract Factory only when products must be made in consistent sets.

**Mediator vs Observer vs Event Aggregator** — Observer: one-to-many dependency (subject knows it's observed). Mediator: centralizes *many-to-many* peer coordination with *behavior* in the hub. Event Aggregator: a dumb pub/sub channel — routing only, no coordination logic. Behavior in the hub → Mediator; routing only → Aggregator.

**Adapter vs Bridge** — Adapter is *remedial* (after the fact, reconcile two existing interfaces). Bridge is *preventive* (designed up front so abstraction and implementation vary on two axes).

**Template Method vs Strategy** — both vary an algorithm. Template Method: inheritance, override fixed hook steps (compile-time). Strategy: composition, swap the whole algorithm (runtime). Prefer Strategy for runtime swap / to avoid inheritance coupling.

**Chain of Responsibility vs Decorator** — near-identical structure. CoR lets a handler *stop* (exactly one or zero consume the request). Decorator never breaks the flow — *every* wrapper executes.

**Composite vs Decorator** — both recursive same-interface trees. Composite aggregates *many children* (structure). Decorator wraps *exactly one* component (enhancement).

**Strategy vs Command** — Command reifies a *request/action* (queue/log/undo), carries receiver + params. Strategy reifies an *algorithm* plugged into a host. Command = "do/undo later"; Strategy = "compute this way."

---

## 3. Over-engineering guardrails — when NOT to pattern

**Every pattern has a price.** A pattern buys flexibility along *one* axis and charges interest in: added indirection (control flow hops through interfaces), more files/types to navigate, steeper onboarding, harder debugging (the stack trace stops reading top-to-bottom). The flexibility is worth it *only if the axis you made flexible actually changes.* **Flexibility you don't use is pure cost.**

**Ousterhout's deepness test** — the right question for any abstraction: **"Is the interface simpler than the implementation it hides?"** A *deep* module hides substantial complexity behind a small interface (Unix file I/O: 5 calls). Premature patterns produce *shallow* modules — `IFooFactory`, `AbstractFooProviderStrategy`, a one-line interface per one-line class — where learning the interface costs more than the complexity hidden. A pattern that adds interface surface without hiding real complexity makes the system **worse**.

**The Sandi Metz / YAGNI axis:**
- **Duplication is far cheaper than the wrong abstraction.** A premature abstraction gets a parameter, then a flag, then a conditional, and becomes a tangle no one can safely change. *"When you have the wrong abstraction, the fastest way forward is back"* — re-inline and re-extract from real data.
- **Rule of three.** Wait for the *third* occurrence before extracting. Two points define a line through anything; three reveal the true axis of variation.
- **YAGNI.** Don't pattern for an imagined future. Speculative generality is a named smell, not foresight.

**JUSTIFIED vs SPECULATIVE — the discriminator:**

| JUSTIFIED | SPECULATIVE |
|---|---|
| A real, named smell exists *today* (shotgun surgery, a giant conditional that grows every feature) | "We might need to support X someday" |
| The variation point has actually varied — a concrete 2nd/3rd case is in hand | Only one implementation; the interface has one caller |
| The same change keeps touching many files (measured) | The flexibility is hypothetical / unrequested |
| The pattern *removes* branching/coupling | The pattern *adds* indirection while keeping the same coupling |
| Hides real complexity behind a smaller interface (deep) | One-line interface over one-line class (shallow) |

**Pre-flight checklist — introduce a pattern only if the change passes ALL four gates:**
1. **(a) Removes a real, named smell.** Point at the specific smell (not "cleaner"/"more flexible"). If you can't name it, stop.
2. **(b) Reduces *total* complexity, not just relocates it.** Count the system, not the file. Indirection that moves an `if` into a class hierarchy without shrinking the whole isn't a win.
3. **(c) Has ≥2–3 concrete call sites or a real second case.** No imagined future use; the variation must already exist or be committed and imminent.
4. **(d) Is reversible.** Prefer the change you can cheaply inline back. If backing out is expensive, raise the bar on (a)–(c).

**Default posture:** start with the simplest thing that works (a plain function, a conditional, a duplicated block). Let the code show you the seam *under real change pressure*, then apply the pattern surgically at the seam that actually moved. **The pattern is the result of observed variation, never the down payment on it.**

**Complement, don't churn.** Code that is already well-factored (a clean Strategy, a disciplined Manager, a DU state machine) should be *recognized and left alone* — or extended in its own idiom. Rewriting a good design into a different-but-equivalent one is negative-value work. Name what's good and why, so it isn't accidentally "refactored" away later.

**The underlying coupling test (connascence).** When two coupling smells compete, the deciding question is: does the refactor convert a *stronger* connascence into a *weaker* one and/or shrink its scope? Strength order (weak→strong): Name → Type → Meaning → Position → Algorithm → Execution-order → Timing → Value → Identity. A value object turns Data Clumps' connascence of *position* (argument order) into connascence of *name* (fields); that's why it's an improvement. The pattern is justified when it lowers connascence strength or moves it into a smaller scope — not when it just renames the coupling.

**The inheritance contract (LSP/DIP).** Before recommending an inheritance-based pattern (Template Method, Factory Method via subclassing, class Strategy/State hierarchies), check **Liskov**: a subtype must accept everything the base accepts (no strengthened preconditions), promise everything the base promises (no weakened postconditions), preserve invariants and history, and never throw where the base wouldn't. The classic trap is Rectangle/Square — if a subclass can't honor the base's contract, prefer **composition/delegation** over inheritance. And follow **DIP**: depend on abstractions, point dependencies *inward* toward the domain — this is the real justification for the DI and Repository entries, not "interfaces are tidy."

## 4. Big refactors: stage them, don't cut them in one go
Gate (d) prefers reversible changes, but some Justified findings are genuinely large (replacing a god Manager, swapping a persistence layer, splitting a tangled module). Don't do those in one cut:
- **Strangler-fig** — stand the new structure up *beside* the old, route call sites to it incrementally, and delete the old path only when its traffic is zero. Each step is small and reversible.
- **Anti-corruption layer (ACL)** — when integrating a model you don't control (legacy/external), put a translation boundary at the seam so its shapes don't leak into the new design.
- **Expand → contract (parallel change)** — for an interface/schema change: add the new form, migrate callers, then remove the old. Never break-and-fix in one commit.
- **Branch by abstraction** — introduce a seam (an abstraction over the thing being replaced), swap implementations behind it, retire the seam if it was scaffolding.
The throughline: a large refactor is a *sequence of small green-to-green steps* behind a seam, each pinned by tests (Phase 7), never a big-bang rewrite.
