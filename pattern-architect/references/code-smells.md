# Code Smells → Pattern Map

The five smell families (Fowler / refactoring.guru). Each entry: how to **spot** it (concrete
signals), why it **hurts**, and what **refactorings + patterns** resolve it. The smell is the
*evidence* that justifies a pattern — never recommend a pattern without naming the smell it kills.

**Contents:** 1. Bloaters · 2. OO Abusers · 3. Change Preventers · 4. Dispensables · 5. Couplers ·
the **SMELL → PATTERN MAP** lookup table · **concurrency/stateful smells → pattern**.

---

## 1. BLOATERS — code grown to gargantuan size

**Long Method** — *Signs (semantic first):* you must insert a comment to explain a block; multiple levels of abstraction in one method; deep nesting. Line count (~10 is Fowler's *provocation*, not a limit) is a smell only when comprehension actually suffers — don't flag a clean 30-line method. *Hurts:* hard to comprehend, hides duplicate code. *Fix:* Extract Method (primary), Replace Temp with Query, Decompose Conditional, Replace Method with Method Object, Introduce Parameter Object.

**Large Class** — *Signs:* many fields/methods/lines; mixed responsibilities; field clusters used by only some methods. *Hurts:* high cognitive load, breeds duplication. *Fix:* Extract Class, Extract Subclass, Extract Interface. *Pattern:* **Strategy / Facade** to redistribute responsibilities.

**Primitive Obsession** — *Signs:* primitives standing in for domain concepts (money/phone/ZIP as `string`/`int`); constants encoding meaning (`USER_ADMIN_ROLE = 1`); string keys indexing a pseudo-field array. *Hurts:* validation/logic scattered instead of in one type. *Fix:* Replace Data Value with Object; Replace Type Code with Class/Subclasses/State-Strategy; Introduce Parameter Object. *Pattern:* **State / Strategy** for coded behavioral variants.

**Long Parameter List** — *Signs:* more than three or four parameters. *Hurts:* hard to read, easy to mis-order. *Fix:* Replace Parameter with Method Call, Preserve Whole Object, Introduce Parameter Object. *Pattern:* **Builder** for many optional construction params.

**Data Clumps** — *Signs:* the same group of variables recurring as fields in several classes or as the same parameter cluster (host/port/user/password). *Hurts:* related data that should be one concept stays scattered. *Fix:* Extract Class, Introduce Parameter Object, Preserve Whole Object.

## 2. OBJECT-ORIENTATION ABUSERS — OO misapplied

**Switch Statements** — *Signs:* a complex `switch`/`case` or long `if-else if`, especially branching on a type code and reappearing in several places. *Hurts:* the same switch scatters; every new case forces edits in every copy. *Fix:* Replace Conditional with Polymorphism (primary), Replace Type Code with Subclasses or State/Strategy. *Pattern:* **State / Strategy / Factory Method** (a switch inside a factory is acceptable).

**Temporary Field** — *Signs:* fields populated only during one algorithm, empty otherwise. *Hurts:* conditionally-set fields obscure intent, invite null bugs. *Fix:* Extract Class (method object), Introduce Null Object.

**Refused Bequest** — *Signs:* a subclass uses only a fraction of inherited members; inheritance overridden to throw. *Hurts:* inheritance-for-reuse signals a wrong "is-a". *Fix:* Replace Inheritance with Delegation; Extract Superclass when there *is* genuine shared structure.

**Alternative Classes w/ Different Interfaces** — *Signs:* two classes do the same job with differently named/shaped methods. *Hurts:* duplication + confusion over which to use. *Fix:* Rename/Add Parameter/Parameterize Method to align; Extract Superclass. *Pattern:* **Adapter** to reconcile interfaces you can't change.

## 3. CHANGE PREVENTERS — one change forces many

**Divergent Change** — *Signs:* editing one class for unrelated reasons (a new product type makes you touch find, display, *and* order methods in the same class). *Hurts:* one class, multiple responsibilities (violates SRP). *Fix:* Extract Class, Extract Superclass/Subclass. One class, one axis of change.

**Shotgun Surgery** — *Signs:* one conceptual change forces many small edits across many classes. *Hurts:* a single responsibility smeared across the codebase; updates easy to miss (inverse of Divergent Change). *Fix:* Move Method + Move Field to gather it into one class; Inline Class to collapse remnants.

**Parallel Inheritance Hierarchies** — *Signs:* adding a subclass to one hierarchy forces a matching subclass in another. *Hurts:* mirrored hierarchies duplicate structure and drift. *Fix:* reference one hierarchy from the other, then Move Method/Field to collapse the redundant one. (Leave it if removal makes the code uglier.)

## 4. DISPENSABLES — pointless things to remove

**Comments** — *Signs:* a method larded with comments narrating *what* the code does. *Hurts:* comments paper over code that should be self-explanatory; they drift. *Fix:* Extract Method, Rename Method, Extract Variable, Introduce Assertion. Comments explaining *why* or a genuinely complex algorithm are legitimate.

**Duplicate Code** — *Signs:* near-identical fragments, or fragments differing superficially while doing the same thing. *Hurts:* every fix must hit n places; one gets missed. *Fix:* Extract Method; Pull Up Method; Extract Superclass/Class; Consolidate Duplicate Conditional Fragments. *Pattern:* **Template Method** (shared skeleton, varying steps). *(But heed "the wrong abstraction" — see pattern-relationships.md §3.)*

**Lazy Class** — *Signs:* a class not doing enough to justify its existence. *Hurts:* every class costs comprehension. *Fix:* Inline Class, Collapse Hierarchy. (A deliberate placeholder may stay.)

**Data Class** — *Signs:* only fields + getters/setters; all behavior lives in *other* classes manipulating its data. *Hurts:* forfeits the core power of objects (behavior with data). *Fix:* Move Method, Encapsulate Field/Collection, Remove Setting Method.

**Dead Code** — *Signs:* an unreferenced variable/param/field/method/class; branches that can't execute. *Hurts:* clutters reading. *Fix:* delete it; Remove Parameter; Inline Class. (Use dead-code analyzers / coverage.)

**Speculative Generality** — *Signs:* unused abstract classes, hooks, parameters, fields built "just in case." *Hurts:* hypothetical flexibility, real complexity now (YAGNI). *Fix:* Collapse Hierarchy, Inline Class/Method, Remove Parameter. (Exception: framework code with external consumers.)

## 5. COUPLERS — excessive coupling

**Feature Envy** — *Signs:* a method uses another object's data more than its own. *Hurts:* behavior lives apart from its data; they change out of sync. *Fix:* Move Method (to the data's class); Extract Method then move. *Exception:* **Strategy / Visitor** deliberately separate behavior from data.

**Inappropriate Intimacy** — *Signs:* one class reaches into another's internals; tightly bound bidirectional relationships. *Hurts:* mutual knowledge of internals blocks independent maintenance/reuse. *Fix:* Move Method/Field, Extract Class + Hide Delegate, Change Bidirectional Association to Unidirectional.

**Message Chains** — *Signs:* `a.getB().getC().getD()`. *Hurts:* the client is welded to the whole object graph's shape (violates Law of Demeter). *Fix:* Hide Delegate, Extract + Move Method. (Don't over-apply, or you create a Middle Man.)

**Middle Man** — *Signs:* a class whose methods mostly just delegate to another — a hollow pass-through. *Hurts:* needless indirection. *Fix:* Remove Middle Man, Inline Method. *Exception:* keep it when it's an intentional **Proxy / Decorator** or a dependency-reducing layer.

*(Also: Incomplete Library Class — fix with Introduce Foreign Method / Local Extension; pattern: **Adapter**.)*

---

## SMELL → PATTERN MAP (the lookup table)

**Read left-to-right and stop at the first thing that works.** The simplest refactoring is listed
first on every row for a reason — most smells are fixed by Extract/Move/Inline/rename, *not* a pattern.
A GoF pattern is the *escalation* (after Rule of Three, when variation is real), never the default.
Reaching straight for the pattern column is how a healthy codebase gets pattern-bombed.

| Smell | Simplest fix first → … → pattern (escalation) |
|---|---|
| **Long Method** | Extract Method · Replace Temp with Query · Method Object |
| **Large Class** | Extract Class/Subclass/Interface · **Strategy / Facade** |
| **Primitive Obsession** | Replace Data Value with Object (a value type — the usual fix) · Replace Type Code · State/Strategy *only for type codes that drive behavior* |
| **Long Parameter List** | Introduce Parameter Object · Preserve Whole Object · **Builder** |
| **Data Clumps** | Extract Class · Introduce Parameter Object |
| **Switch Statements** | Replace Conditional with Polymorphism · **State / Strategy / Factory Method** |
| **Temporary Field** | Extract Class (Method Object) · Null Object |
| **Refused Bequest** | Replace Inheritance with Delegation · Extract Superclass |
| **Alt. Classes / Diff. Interfaces** | Extract Superclass · **Adapter** |
| **Divergent Change** | Extract Class (SRP split) |
| **Shotgun Surgery** | Move Method/Field · Inline Class · (often a **Manager**/Facade to own the scattered concern) |
| **Parallel Inheritance** | Move Method/Field (cross-reference then collapse) |
| **Comments** | Extract Method · Rename · Extract Variable |
| **Duplicate Code** | Extract Method · Pull Up Method · **Template Method** |
| **Lazy Class** | Inline Class · Collapse Hierarchy |
| **Data Class** | Move Method · Encapsulate Field/Collection |
| **Dead Code** | Delete · Remove Parameter |
| **Speculative Generality** | Collapse Hierarchy · Inline Class/Method (it *is* the over-pattern smell) |
| **Feature Envy** | Move Method (Strategy/Visitor as deliberate exception) |
| **Inappropriate Intimacy** | Move Method/Field · Hide Delegate |
| **Message Chains** | Hide Delegate · Extract + Move Method |
| **Middle Man** | Remove Middle Man (keep if **Proxy/Decorator**) |

## Concurrency / stateful smells → pattern (see concurrency-patterns.md)
| Smell | Pattern |
|---|---|
| `lock`/`Mutex` scattered around shared state | **Actor / Mailbox Processor** |
| state + logic smeared across call sites, invariants enforced nowhere | **Stateful Manager** (DI singleton) |
| a "Manager" that owns unrelated domains; everything routes through it | split by domain (god-object cure), not more patterns |
| O(sources×clients) subscription wiring | **Event Aggregator / Bus** |
| producer hard-coding calls to each consumer | **Observer / Pub-Sub** |
| nested async callbacks, manual subscription bookkeeping | **Reactive / Rx** |
| slow/risky work inline on the request thread | **Queue + Worker** |
| ≥3 interacting boolean flags; reachable meaningless combinations | **State machine** (DU states) |
| subscriptions that never release (handlers fire after teardown) | the **unsubscribe/disposer** discipline, not a new pattern |
