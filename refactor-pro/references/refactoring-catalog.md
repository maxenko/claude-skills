# Refactoring Catalog and Code Smell Taxonomy

Reference guide organized by problem → solution. Consult when diagnosing code or choosing
which refactoring to apply.

## Code Smell Taxonomy

### Category 1: Bloaters (Things That Grew Too Large)

| Smell | How to Detect | Refactoring to Apply |
|-------|--------------|---------------------|
| **Long Function** | >25 lines, needs comments to explain sections, multiple abstraction levels | Extract Function, Replace Temp with Query |
| **God Class** | >500 lines, >15 methods, does unrelated things, low cohesion | Extract Class, Move Function |
| **Primitive Obsession** | Domain concepts as bare strings/ints (`string email`, `float price`) | Introduce Value Object, Replace Primitive with Object |
| **Long Parameter List** | >3 parameters (4+) | Introduce Parameter Object, Preserve Whole Object |
| **Data Clumps** | Same 3+ fields appear together in multiple places | Extract Class, Introduce Parameter Object |

### Category 2: Structural Problems

| Smell | How to Detect | Refactoring to Apply |
|-------|--------------|---------------------|
| **Feature Envy** | Function uses data from another class more than its own | Move Function to the class whose data it uses |
| **Divergent Change** | One class changes for multiple unrelated reasons | Extract Class (split by responsibility) |
| **Shotgun Surgery** | One change requires editing many files | Move Function, Inline Class (consolidate) |
| **Inappropriate Intimacy** | Classes access each other's internals excessively | Move Function, Extract Class, Hide Delegate |
| **Message Chains** | `a.getB().getC().getD().doThing()` | Hide Delegate, Extract Function |
| **Middle Man** | Class does nothing but delegate | Remove Middle Man, Inline Class |
| **Parallel Inheritance** | Adding a subclass in one hierarchy requires one in another | Move Function, Collapse Hierarchy |

### Category 3: Unnecessary Complexity (Dead Weight)

| Smell | How to Detect | Refactoring to Apply |
|-------|--------------|---------------------|
| **Speculative Generality** | Interfaces with 1 impl, unused parameters, abstract classes with 1 subclass | Collapse Hierarchy, Inline Class, Remove Parameter |
| **Dead Code** | Unreachable code, unused variables, unread writes | Delete it. Version control remembers. |
| **Lazy Class** | Class that doesn't do enough to justify its existence | Inline Class |
| **Refused Bequest** | Subclass doesn't use most inherited interface | Replace Inheritance with Delegation |

### Category 4: Missed Modernization

| Smell | How to Detect | Refactoring to Apply |
|-------|--------------|---------------------|
| **Loops** (Fowler 2nd ed.) | Complex loops with filtering, mapping, accumulating | Replace Loop with Pipeline (map/filter/reduce) |
| **Temporary Field** | Object fields only set/used in certain methods, null/empty otherwise | Extract Class, Introduce Special Case |

### Category 5: Control Flow Problems

| Smell | How to Detect | Refactoring to Apply |
|-------|--------------|---------------------|
| **Deep Nesting** | >3 levels of indentation in conditions/loops | Replace with Guard Clauses, Extract Function |
| **Switch on Type** | Repeated switch/if-else on same type discriminator | Replace Conditional with Polymorphism |
| **Complex Boolean** | Boolean expressions with >2 operators or negation chains | Extract to Named Boolean Variables |
| **Flag Arguments** | Boolean parameter that changes function behavior | Split into Separate Functions |

### Category 6: Naming and Clarity

| Smell | How to Detect | Refactoring to Apply |
|-------|--------------|---------------------|
| **Mysterious Name** | Must read implementation to understand purpose | Rename (function, variable, class) |
| **Comments Explaining What** | Comment describes what code does (not why) | Extract Function named after the comment |
| **Inconsistent Naming** | Same concept called different things in different places | Rename consistently |
| **Encoding in Names** | Type info in name (`strName`, `iCount`, `btnSubmit`) | Rename to describe purpose, not type |

---

## Refactoring Catalog by Situation

### "I need to understand this code"
1. **Rename Variable/Function/Class** — Make names express intent
2. **Extract Variable** — Name complex expressions
3. **Extract Function** — Name code blocks after their purpose
4. **Decompose Conditional** — Make conditions self-documenting

### "This function is too long"
1. **Extract Function** — Pull out blocks that serve a sub-purpose
2. **Replace Temp with Query** — Eliminate temporaries that force sequential reading
3. **Split Phase** — If the function does two distinct things, split at the boundary
4. **Replace Loop with Pipeline** — Map/filter/reduce when transforming collections

### "This class does too much"
1. **Extract Class** — Split by responsibility (ask: "what are its reasons to change?")
2. **Move Function** — Move functions to the class whose data they primarily use
3. **Extract Interface** — Define the contract if different callers use different subsets

### "I keep changing the same thing in multiple places"
1. **Move Function** — Consolidate into one location
2. **Extract Superclass/Interface** — Unify shared behavior
3. **Introduce Parameter Object** — Group commonly-passed parameters
4. **Replace Conditional with Polymorphism** — Centralize type-based dispatch

### "I can't test this code"
1. **Extract Interface** — Create a seam for mocking
2. **Dependency Injection** — Pass dependencies in rather than creating them internally
3. **Extract Pure Function** — Isolate logic from side effects
4. **Replace Global with Injection** — Make hidden dependencies explicit

### "Adding a feature is unnecessarily hard"
This is Fowler's "preparatory refactoring" — restructure first, then make the easy change.
1. **Move Function** — Put code where the new feature needs it
2. **Extract Interface** — Create an extension point
3. **Split Phase** — Separate what exists from what you're adding
4. **Parameterize Function** — Generalize an existing function to handle the new case

### "The inheritance hierarchy is wrong"
1. **Replace Subclass with Delegate** — Switch from inheritance to composition
2. **Replace Superclass with Delegate** — When the parent-child relationship is forced
3. **Collapse Hierarchy** — When a subclass adds nothing meaningful
4. **Extract Interface** — Decouple from concrete hierarchy

---

## Metric Thresholds Quick Reference

| Metric | Good | Acceptable | Investigate | Refactor |
|--------|------|------------|-------------|----------|
| Function length (lines) | 1-15 | 16-25 | 26-40 | >40 |
| Cyclomatic complexity | 1-5 | 6-8 | 9-12 | >12 |
| Cognitive complexity | 1-8 | 9-12 | 13-15 | >15 |
| Parameters per function | 0-2 | 3 | 4-5 | >5 |
| Nesting depth | 1-2 | 3 | 4 | >4 |
| Class/file length (lines) | 50-200 | 201-300 | 301-500 | >500 |
| Methods per class | 3-7 | 8-10 | 11-15 | >15 |
| Inheritance depth | 1-2 | 3 | 4-5 | >5 |

These are guidelines, not laws. A 50-line function that does one thing at one abstraction level
may be fine. A 10-line function that does three things at different levels needs splitting. The
metric is a signal to investigate, not an automatic verdict.

---

## Anti-Patterns to Avoid

### Big Bang Refactoring
Rewriting large sections at once. No incremental feedback, impossible to bisect bugs, often
never finishes. Instead: Strangler Fig pattern — incrementally replace while the system runs.

### Premature Abstraction
Creating abstractions before you have 3+ concrete examples. "Three strikes and you refactor"
(Fowler). One implementation behind an interface is over-engineering.

### Over-DRYing
Coupling unrelated code that happens to look similar. Ask: "Would these uses evolve together?"
If no, duplication is correct. "Duplication is far cheaper than the wrong abstraction" (Sandi Metz).

### Refactoring Without Tests
Changing structure without a safety net. If no tests exist, write characterization tests first
(record actual outputs, assert on them). Only then refactor.

### Refactoring + Feature Change Simultaneously
If something breaks, you can't tell which caused it. Fowler's "Two Hats" rule: you're either
refactoring or adding behavior, never both in the same commit.

### Ignoring Hot Paths
Refactoring performance-critical loops for readability without benchmarking. Profile first.
Refactor non-hot paths freely. For hot paths, document the optimization rationale.

### Inconsistent Abstraction Levels
A function that mixes high-level orchestration with low-level details. Every line in a function
should operate at the same level of abstraction. Extract the low-level bits.

### Gold Plating
Adding layers beyond what requirements demand. "What if we need to swap databases?" when you
never will. Strategy pattern with one strategy. Factory for one product. YAGNI.

---

## Incremental Refactoring Strategies

For large codebases where you can't refactor everything at once:

### Strangler Fig
1. Create a facade/proxy intercepting calls to legacy code
2. Route features one-by-one to new implementations
3. Old and new coexist during transition
4. Decommission legacy when migration complete

### Branch by Abstraction
1. Create an interface representing the component to replace
2. Change clients to use the interface (still backed by old code)
3. Build new implementation alongside old
4. Swap implementations (can use feature toggles)
5. Remove old implementation

### Parallel Change (Expand-Contract)
1. **Expand**: Add new API alongside old (both work)
2. **Migrate**: Switch callers one-by-one to new API
3. **Contract**: Remove old API when all callers migrated
