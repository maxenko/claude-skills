# Code Smells Taxonomy

Quick-reference for identifying refactoring opportunities. Based on Martin Fowler's catalog with modern additions.

## Bloaters (grew too large)

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| Long Method | Function > 20 lines or does more than one thing | Extract Method |
| Large Class | Class with > 5 responsibilities or > 300 lines | Extract Class, Extract Interface |
| Long Parameter List | > 3-4 parameters | Introduce Parameter Object, use builder |
| Primitive Obsession | Raw strings/ints for domain concepts (email, money, ID) | Replace with Value Object |
| Data Clumps | Same group of variables passed together repeatedly | Extract into a struct/class |

## Object-Orientation Abusers

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| Switch Statements | Repeated switch/if-else on the same type field | Replace with Polymorphism |
| Refused Bequest | Subclass ignores inherited methods | Replace Inheritance with Delegation |
| Temporary Field | Fields only populated in some scenarios | Extract Class, Introduce Null Object |

## Change Preventers (high modification cost)

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| Divergent Change | One class changed for many different reasons | Extract Class by responsibility |
| Shotgun Surgery | One change touches many classes | Move Method, Inline Class |
| Parallel Inheritance | Subclass in hierarchy A forces subclass in B | Merge hierarchies or use composition |

## Dispensables (remove to improve)

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| Duplicate Code | Identical or near-identical blocks | Extract Method, Pull Up |
| Dead Code | Unreachable branches, unused imports/vars | Delete it |
| Lazy Class | Class that doesn't justify its existence | Inline Class |
| Speculative Generality | Abstractions for imagined future needs | Collapse Hierarchy, Inline |
| Data Class | Only fields + getters/setters, no behavior | Move behavior into the class |

## Couplers (excessive interdependence)

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| Feature Envy | Method uses another class's data more than its own | Move Method |
| Inappropriate Intimacy | Classes know too much about each other | Move Method, Extract Class |
| Message Chains | a.getB().getC().getD() | Hide Delegate, Extract Method |
| Middle Man | Class only delegates to another | Remove Middle Man, Inline |

## Modern Additions

| Smell | Signal | Refactoring |
|-------|--------|-------------|
| Callback Hell | Deeply nested async callbacks/promises | async/await, extract named functions |
| Copy-Paste-Modify | Duplicated code with small variations | Extract with parameters, generics, or strategy |
| Stringly-Typed | Strings where enums/types would be safer | Replace with enum/union type |
| God Config | One massive config object for everything | Split by concern/module |
| Boolean Blindness | Functions taking multiple booleans | Replace with enum, options object, or separate functions |
| Leaky Abstraction | Implementation details exposed through interface | Redesign interface boundary |

## When NOT to Refactor

- The code is being deleted soon
- The code works, is tested, and nobody needs to modify it
- The refactoring would require touching many unrelated files (high blast radius for low value)
- Three similar lines is fine; premature abstraction is worse than mild duplication
