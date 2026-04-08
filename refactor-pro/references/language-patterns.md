# Language-Specific Refactoring Patterns

Quick reference for idiomatic conventions across major languages. When refactoring, match the
target language's conventions exactly. If the project already has established conventions that
differ from the standard, follow the project's conventions.

## Naming Conventions

| Language | Variables/Params | Functions/Methods | Classes/Types | Constants | Modules/Packages |
|----------|-----------------|-------------------|---------------|-----------|-----------------|
| **Python** | `snake_case` | `snake_case` | `PascalCase` | `UPPER_SNAKE` | `snake_case` |
| **JavaScript/TS** | `camelCase` | `camelCase` | `PascalCase` | `UPPER_SNAKE` | `camelCase` |
| **Java** | `camelCase` | `camelCase` | `PascalCase` | `UPPER_SNAKE` | `lowercase` |
| **C#** | `camelCase` | `PascalCase` | `PascalCase` | `PascalCase` | `PascalCase` |
| **Go** | `camelCase`/`PascalCase` | `camelCase`/`PascalCase` | `PascalCase` | `MixedCaps` | `lowercase` |
| **Rust** | `snake_case` | `snake_case` | `PascalCase` | `UPPER_SNAKE` | `snake_case` |
| **C++** | varies by guide | varies by guide | `PascalCase` | `kConstant` or `UPPER_SNAKE` | `snake_case` |
| **Ruby** | `snake_case` | `snake_case` | `PascalCase` | `UPPER_SNAKE` | `snake_case` |
| **PHP** | `$camelCase` | `camelCase` | `PascalCase` | `UPPER_SNAKE` | `PascalCase` |
| **Swift** | `camelCase` | `camelCase` | `PascalCase` | `camelCase` | `PascalCase` |
| **Kotlin** | `camelCase` | `camelCase` | `PascalCase` | `UPPER_SNAKE` | `lowercase` |

**Go-specific:** Capitalization controls visibility. `PascalCase` = exported (public),
`camelCase` = unexported (private). Go avoids `SCREAMING_SNAKE` even for constants.

**Rust-specific:** Rust docs call PascalCase "UpperCamelCase" and UPPER_SNAKE "SCREAMING_SNAKE_CASE."
Acronyms in types count as one word (`Uuid` not `UUID`).

**Python-specific:** `_private` (convention), `__mangled` (name mangling), `__dunder__` (special).

---

## Python (3.12+)

### Modern features to leverage when refactoring:
- **`match`/`case`** — Replace long `if`/`elif` chains dispatching on structure or type
- **`dataclasses` with `slots=True`, `frozen=True`** — Replace manual `__init__`/`__repr__` boilerplate
- **`StrEnum`/`IntEnum`** — Replace string/int constant groups
- **Type hints** — Add at function signatures for documentation value. Use `Protocol` for structural subtyping
- **`pathlib.Path`** — Replace `os.path` string manipulation
- **Comprehensions** — Replace simple loops that build collections (but use explicit loops when transformation involves >2 operations)
- **`contextlib.contextmanager`** — Replace try/finally resource management
- **`functools.cache`** — Replace hand-rolled memoization
- **Walrus operator `:=`** — Reduce repetition in `while` loops and comprehension filters

### Idiomatic patterns:
- Use `collections.abc` types (`Sequence`, `Mapping`) in parameter type hints, concrete types (`list`, `dict`) for return types
- Prefer EAFP (try/except) over LBYL (if/check) for duck typing
- Use `**kwargs` sparingly — prefer explicit `dataclass` parameter objects when there are >3 config options

---

## JavaScript / TypeScript (ES2024+)

### Modern features to leverage:
- **`using` keyword** (TS 5.2+) — Replace try/finally for resource cleanup
- **`Object.groupBy`/`Map.groupBy`** (ES2024) — Replace manual grouping loops
- **`satisfies` operator** (TS 4.9+) — Type-check without widening for configs/literals
- **Decorators** (TS 5.0+) — Replace higher-order function wrapping on classes
- **Template literal types** — Build precise string-based types

### TypeScript patterns:
- `interface` for object shapes that will be extended; `type` for unions, intersections, mapped types
- **Discriminated unions** over optional properties for state machines
- **`unknown`** over `any` — narrow with type guards
- **`as const`** for literal inference, **`satisfies`** for validation without widening
- **`Record<string, T>`** over `{[key: string]: T}` for readability

### React patterns (React 19+):
- Move data fetching to Server Components. `"use client"` only for event handlers/state/browser APIs
- Replace `useEffect` for data fetching with Suspense + RSC or TanStack Query
- Replace prop drilling with composition (children, render props) before Context
- Colocate state as close to where it's used as possible
- Replace `forwardRef` — React 19+ passes ref as regular prop

---

## Go

### Idiomatic patterns:
- **Accept interfaces, return structs** — Define interfaces at the consumer, not the producer
- **Small interfaces** (1-3 methods) — `io.Reader`, `io.Writer`, `fmt.Stringer` are the ideal
- **Generics** (1.18+) — Replace `interface{}` / `any` where type safety matters
- **Error wrapping** — `fmt.Errorf("context: %w", err)`, check with `errors.Is`/`errors.As`
- **Early returns** — Happy path left-aligned. Guard clauses at top
- **Table-driven tests** — Standard pattern for multiple test cases
- **`context.Context`** — First parameter for cancellation, timeouts, request-scoped values
- **Package by domain** — Not by technical layer (`models/`, `controllers/`)

### Go proverbs:
- Don't communicate by sharing memory; share memory by communicating
- A little copying is better than a little dependency
- Clear is better than clever
- The bigger the interface, the weaker the abstraction
- Make the zero value useful

---

## Rust

### Idiomatic patterns:
- **`?` operator** — Replace `unwrap()`/`expect()` chains. Use `thiserror` for library errors, `anyhow` for app errors
- **Enum + match** — For state machines. Each state is a variant carrying its data
- **Audit `.clone()`** — Most clones are unnecessary. Use references and lifetimes
- **`From`/`Into`** — For conversions instead of custom methods
- **Builder pattern** — For complex struct construction
- **NewType pattern** — `struct Meters(f64)` for type safety on primitives
- **Iterators over index access** — `for item in &collection` not `for i in 0..collection.len()`
- **`Option` combinators** — `map`, `and_then`, `unwrap_or_else` over match on every Option
- **`&str` over `String`** in params, `&[T]` over `Vec<T>`. Use `Cow<'_, str>` when sometimes allocating

### Ownership principles:
- Data flows down through ownership (tree structure)
- Question the design before reaching for `Rc`/`Arc`
- Prefer message passing (channels) over `Arc<Mutex<T>>` when possible
- Rule of thumb: if you're fighting the borrow checker, rethink your data layout

---

## Java (21+)

### Modern features to leverage:
- **Records** — Replace data-carrier classes (DTOs/POJOs) with `record Point(int x, int y) {}`
- **Sealed classes** — Replace open hierarchies when subtypes are known. Enables exhaustive matching
- **Pattern matching for `switch`** (21+) — Replace `instanceof` chains
- **Virtual threads** (21+) — Replace thread pool management for I/O-bound tasks
- **Text blocks** — Replace string concatenation for multiline strings
- **`Optional`** — Return type only (never as field/parameter). Use `map`/`flatMap`/`orElseThrow`
- **`var`** (10+) — When the type is obvious from the right-hand side

### Patterns:
- Streams for functional transformations; plain loops for mutations
- Final fields by default; mutable only when required
- Dependency injection via constructor (not field injection)

---

## C# (.NET 8+)

### Modern features to leverage:
- **Primary constructors** (C# 12) — `class Service(ILogger logger)` reduces boilerplate
- **Collection expressions** (C# 12) — `int[] arr = [1, 2, 3]`
- **Required members** (C# 11) — Replace constructor-only initialization
- **Raw string literals** (C# 11) — For JSON, XML, regex without escaping
- **`record` types** (C# 9+) — For DTOs and value objects
- **Pattern matching** (C# 7-12) — Property, relational, and list patterns
- **`Span<T>`/`Memory<T>`** — Replace array slicing that allocates
- **File-scoped namespaces** — Reduce one level of indentation
- **Nullable reference types** — Enable project-wide. Non-nullable by default

---

## C++ (C++23)

### Modern C++ features to leverage:
- **Smart pointers** — `unique_ptr` (single ownership), `shared_ptr` (shared). Raw pointers only for non-owning references
- **RAII** — Every resource acquisition in a constructor, release in a destructor
- **`std::optional`** (C++17) — Replace sentinel values and out-parameters
- **`std::variant`** (C++17) — Replace C-style unions with type tags
- **`std::expected`** (C++23) — Replace error codes for expected failures
- **`std::ranges` and views** (C++20/23) — Replace raw loops with composable pipelines
- **Concepts** (C++20) — Replace SFINAE for template constraints
- **`std::format`/`std::print`** (C++20/23) — Replace `printf`/iostream formatting
- **`std::span`** (C++20) — Replace `(pointer, size)` pairs

### Rules:
- **Rule of Zero:** If no direct resource management, don't declare special member functions
- **Rule of Five:** If you define one of destructor/copy-ctor/copy-assign/move-ctor/move-assign, define all five
- **Structured bindings** — `auto [key, value] = *map.begin()`
- **`constexpr`/`consteval`** — Move computation to compile time where possible

---

## Import Organization (Universal)

Standard grouping order (separated by blank lines):

1. **Standard library / built-in** imports
2. **Third-party / external** packages
3. **Internal / first-party** packages
4. **Local / relative** imports

Alphabetically sorted within each group. Use language-specific tools (isort, goimports, eslint
import plugin) when available.
