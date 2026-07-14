# F# Uplift — Pattern Catalog

Detailed before/after examples for each category. Reference this when building report findings. Every "After" assumes it passes the uplift test in SKILL.md — these show *what good looks like*, not a mandate to apply everywhere.

## Contents

1. Pipeline & Composition — forward pipe, eta-reduction, value-restriction footgun, partial application
2. Collection HOFs — map/filter, comprehensions, choose, fold, find, collect, groupBy, iter, List-vs-Seq
3. Option Combinators — map/bind/defaultValue, eager-default trap (defaultWith), ofObj, option CE
4. Result & Railway-Oriented — result CE, error-arm caveat, traverse/sequence, exceptions→Result, asyncResult, validation
5. Discriminated Unions — stringly-typed, boolean-soup, single-case smart constructor, units of measure
6. Pattern Matching & Active Patterns — if-ladder→match, `function`, wildcard exhaustiveness, active patterns
7. Records & Immutability — data class→record, copy-update, anonymous records
8. Async / Task — StartChild vs Parallel, RunSynchronously, async-vs-task
9. OO Residue — single-method class→function, object expressions, modules, use/using, null→Option
10. Ecosystem & Misc — Argu, interpolation, nameof

Catalog sections group the same patterns as the SKILL.md checklist (units of measure live under §5, async under §8); the in-prose "see catalog" pointers reference sections by name, not number.

## 1. Pipeline & Composition

### Nested calls → forward pipe
```fsharp
// Before: reads inside-out
let result = List.sum (List.map square (List.filter isEven numbers))

// After: reads in execution order
let result =
    numbers
    |> List.filter isEven
    |> List.map square
    |> List.sum
```

### Lambda wrapper → eta-reduction / composition
```fsharp
// Before
let names = users |> List.map (fun u -> getName u)
let process = fun x -> validate (normalize x)

// After
let names = users |> List.map getName
let process = normalize >> validate
```

### When NOT to go point-free
```fsharp
// Too clever — what does this even take?
let f = List.filter (fst >> (<) 0) >> List.map snd >> List.sumBy id

// Clearer with names — keep this
let totalOfPositive pairs =
    pairs
    |> List.filter (fun (qty, _) -> qty > 0)
    |> List.sumBy (fun (_, price) -> price)
```

### Value restriction — the eta-reduction footgun
```fsharp
// Compiles: the parameter pins the type
let mapAll = fun xs -> List.map id xs

// FS0030 value restriction: generic, not applied, can't generalize — DOES NOT COMPILE
let mapAll = List.map id

// Safe to eta-reduce because the type is concrete (a concrete fn, or fixed by use):
let upper = List.map (fun (s: string) -> s.ToUpper())
```
The trap only bites when the eta-reduced top-level `let` is still **generic**. If the inner function is concrete (e.g. `getName : User -> string`), `let getNames = List.map getName` compiles fine. Eta-reduction in *argument position* (`xs |> List.map id`) is always safe.

### Partial application to capture config (F#'s constructor injection)
```fsharp
// Before: a class whose only job is to hold a dependency and expose one method
type Validator(rules: Rule list) =
    member _.Validate(input) = checkAll rules input

// After: partial application captures the config; no class needed
let validate rules input = checkAll rules input
let validate' = validate myRules        // pre-applied; pass around like a service
let result = validate' input
```

---

## 2. Collection HOFs

### Imperative loop → map/filter
```fsharp
// Before
let upperNames = ResizeArray()
for u in users do
    if u.IsActive then
        upperNames.Add(u.Name.ToUpper())

// After
let upperNames =
    users
    |> List.filter (fun u -> u.IsActive)
    |> List.map (fun u -> u.Name.ToUpper())
```

### Loop with interleaved logic → comprehension
```fsharp
// When the body has multiple yields or branching, a comprehension can beat a HOF chain
let cells =
    [ for row in 0 .. rows - 1 do
        for col in 0 .. cols - 1 do
            if isLive row col then
                yield { Row = row; Col = col } ]
```

### filter + map → choose (one pass, option-driven)
```fsharp
// Before
let parsed = ResizeArray()
for s in lines do
    match Int32.TryParse s with
    | true, n -> parsed.Add(n)
    | _ -> ()

// After
let parsed =
    lines |> List.choose (fun s ->
        match Int32.TryParse s with
        | true, n -> Some n
        | _ -> None)
```

### Mutable accumulator → fold / sumBy
```fsharp
// Before
let mutable total = 0m
for line in order.Lines do
    total <- total + line.Price * decimal line.Qty

// After
let total = order.Lines |> List.sumBy (fun l -> l.Price * decimal l.Qty)
```

### Manual find with break → tryFind / tryPick
```fsharp
// Before
let mutable found = None
let mutable i = 0
while found.IsNone && i < users.Length do
    if users.[i].Id = targetId then found <- Some users.[i]
    i <- i + 1

// After
let found = users |> List.tryFind (fun u -> u.Id = targetId)
```

### Nested loops → collect (flat-map)
```fsharp
// Before
let allItems = ResizeArray()
for order in orders do
    for item in order.Items do
        allItems.Add(item)

// After
let allItems = orders |> List.collect (fun o -> o.Items)
```

### Manual grouping → groupBy / countBy
```fsharp
// Before
let byCategory = Dictionary<string, ResizeArray<Product>>()
for p in products do
    match byCategory.TryGetValue p.Category with
    | true, xs -> xs.Add p
    | _ -> let xs = ResizeArray() in xs.Add p; byCategory.[p.Category] <- xs

// After
let byCategory = products |> List.groupBy (fun p -> p.Category)   // (string * Product list) list
```

### Effect-only loop → iter (not map |> ignore)
```fsharp
// Before
for u in users do
    sendEmail u

// After: iter, no throwaway list. (map |> ignore would allocate a result list to discard.)
users |> List.iter sendEmail
```

### List vs Seq — laziness is behavior, not style
```fsharp
// This re-runs the side effect for EACH consumer, and recomputes every enumeration:
let logged = items |> Seq.map (fun x -> printfn "seen %A" x; transform x)
logged |> Seq.length |> ignore   // prints
logged |> Seq.head  |> ignore    // prints AGAIN

// If you want it computed once, eagerly, with effects firing once: use List/Array
let logged = items |> List.map (fun x -> printfn "seen %A" x; transform x)
```

---

## 3. Option Combinators

### Match → map / bind / defaultValue
```fsharp
// Before
let displayName =
    match user.Nickname with
    | Some nick -> Some (nick.ToUpper())
    | None -> None

// After
let displayName = user.Nickname |> Option.map (fun n -> n.ToUpper())
```

```fsharp
// Before: nested option pyramid
let city =
    match user.Address with
    | Some addr ->
        match addr.City with
        | Some c -> Some c
        | None -> None
    | None -> None

// After
let city = user.Address |> Option.bind (fun a -> a.City)
```

```fsharp
// Before
let name = match config.Name with Some n -> n | None -> "default"
// After
let name = config.Name |> Option.defaultValue "default"
```

### Eager-default trap: defaultValue vs defaultWith
```fsharp
// WRONG: openConnection() runs even when the cache HAS the value — defaultValue is eager
let conn = cache.TryGet key |> Option.defaultValue (openConnection ())

// RIGHT: defaultWith defers the fallback to the None case only
let conn = cache.TryGet key |> Option.defaultWith (fun () -> openConnection ())
```
Use `defaultValue` only for literals/already-bound values; any function call, allocation, throw, or log on the fallback path means `defaultWith`.

### Unsafe access → safe combinators
```fsharp
// Before: .IsSome/.Value bypasses exhaustiveness — throws if the assumption breaks
let port = if config.Port.IsSome then config.Port.Value else 8080

// After
let port = config.Port |> Option.defaultValue 8080
```

### .NET null boundary → Option.ofObj
```fsharp
// Before
let header = if isNull (req.Headers.["X-Trace"]) then None else Some req.Headers.["X-Trace"]
// After
let header = req.Headers.["X-Trace"] |> Option.ofObj
```

### Nested options → option CE (FsToolkit.ErrorHandling)
```fsharp
open FsToolkit.ErrorHandling

let fullAddress =
    option {
        let! user    = tryFindUser id
        let! addr    = user.Address
        let! zip     = addr.Zip
        return formatAddress addr zip
    }
```

---

## 4. Result & Railway-Oriented Programming

> Every CE in this section requires `FsToolkit.ErrorHandling` to be **already referenced** in the project. If it isn't, this is a "requires new dependency" finding (low confidence), not a free win.

### Hand-threaded Result match → result CE
```fsharp
// Before
let register input =
    match validateName input.Name with
    | Error e -> Error e
    | Ok name ->
        match validateEmail input.Email with
        | Error e -> Error e
        | Ok email ->
            match saveUser name email with
            | Error e -> Error e
            | Ok id -> Ok id

// After (FsToolkit.ErrorHandling)
open FsToolkit.ErrorHandling

let register input =
    result {
        let! name  = validateName input.Name
        let! email = validateEmail input.Email
        let! id    = saveUser name email
        return id
    }
```

**Caveat — error-arm effects:** the `result` CE only reproduces the "before" when each error arm *just returns the error*. If the original did `| Error e -> log e; Error e`, the CE silently drops the `log e`. Flag that as a behavior change rather than a clean swap.

### List of Results → traverse / sequence
```fsharp
// Before: hand-fold a list of Results, short-circuiting on the first Error
let parseAll lines =
    (Ok [], lines) ||> List.fold (fun acc line ->
        match acc, parseLine line with
        | Ok xs, Ok x -> Ok (x :: xs)
        | Error e, _  -> Error e
        | _, Error e  -> Error e)
    |> Result.map List.rev

// After (FsToolkit): traverse applies a Result-returning fn and collapses the list
let parseAll lines = lines |> List.traverseResultM parseLine   // Result<_ list, _>
// If you already have a `Result list`, use List.sequenceResultM to flip it.
```

### Exceptions for expected failures → Result + DU error
```fsharp
// Before: control flow via exceptions
let parseAge (s: string) =
    let n = Int32.Parse s          // throws on bad input
    if n < 0 then failwith "negative age"
    n

// After: failures are values, callers must handle them
type AgeError = NotANumber of string | Negative of int

let parseAge (s: string) : Result<int, AgeError> =
    match Int32.TryParse s with
    | false, _      -> Error (NotANumber s)
    | true, n when n < 0 -> Error (Negative n)
    | true, n       -> Ok n
```

### Async + Result → asyncResult / taskResult
```fsharp
open FsToolkit.ErrorHandling

let placeOrder cmd =
    asyncResult {
        let! customer = loadCustomer cmd.CustomerId      // Async<Result<_,_>>
        let! _        = checkCredit customer cmd.Total
        let! orderId  = persistOrder cmd                  // also Async<Result<_,_>>
        return orderId
    }
```

### Accumulate ALL errors → validation CE (applicative)
```fsharp
open FsToolkit.ErrorHandling

// result { } stops at the FIRST error. For form validation you usually want them all.
let validateForm (f: RawForm) : Result<Customer, string list> =
    validation {
        let! name  = validateName f.Name        // Result<_, string>
        and! email = validateEmail f.Email       // 'and!' runs independently and collects
        and! age   = validateAge f.Age
        return { Name = name; Email = email; Age = age }
    }
// On failure → Error [ "name too long"; "email invalid"; "age must be >= 0" ]
```

---

## 5. Discriminated Unions — make illegal states unrepresentable

### Stringly-typed → DU
```fsharp
// Before: any string compiles; typos are runtime bugs
type Order = { Status: string }   // "pending"? "Pending"? "PENDING"?
let isOpen o = o.Status = "pending" || o.Status = "shipped"

// After: only valid states exist; match is exhaustive
type OrderStatus = Pending | Shipped | Delivered | Cancelled
type Order = { Status: OrderStatus }
let isOpen o = match o.Status with Pending | Shipped -> true | _ -> false
```

### Boolean-soup → DU carrying state-specific data
```fsharp
// Before: which combinations are legal? Can a user be both verified and pending?
type User =
    { Email: string
      IsVerified: bool
      VerifiedDate: DateTime option
      PendingToken: string option }

// After: each case carries exactly — and only — the data that state needs
type Registration =
    | Pending of token: string
    | Verified of on: DateTime
type User = { Email: string; Registration: Registration }
```

### Bare primitive → single-case DU with smart constructor
```fsharp
// Before: a string is a string; nothing stops you passing a name where an email goes
let sendWelcome (email: string) = ...

// After: validated at the only door in, can't be confused with other strings
type Email = private Email of string
module Email =
    let create (s: string) =
        if s.Contains "@" then Ok (Email s) else Error "invalid email"
    let value (Email s) = s

let sendWelcome (e: Email) = ...   // callers must hold a real Email; use Email.value e internally
// Note: don't deconstruct a `private` case at an external call site (`Email _ as e`) —
// the private modifier makes the pattern inaccessible outside the defining module.
```

### Units of measure for numeric primitives (F#-unique)
```fsharp
[<Measure>] type m
[<Measure>] type s

let distance = 100.0<m>
let time     = 9.58<s>
let speed    = distance / time          // float<m/s>
// let bug   = distance + time          // compile ERROR: can't add metres to seconds
```

---

## 6. Pattern Matching & Active Patterns

### if/elif ladder → match
```fsharp
// Before
let describe n =
    if n < 0 then "negative"
    elif n = 0 then "zero"
    elif n < 10 then "small"
    else "large"

// After
let describe n =
    match n with
    | n when n < 0  -> "negative"
    | 0             -> "zero"
    | n when n < 10 -> "small"
    | _             -> "large"
```

### Lambda doing a match → `function`
```fsharp
// Before
let label = fun status ->
    match status with
    | Pending -> "…"
    | Shipped -> "✓"
    | _ -> "?"

// After
let label =
    function
    | Pending -> "…"
    | Shipped -> "✓"
    | _ -> "?"
```

### Repeated parse-and-test → active pattern
```fsharp
// Before: this TryParse dance recurs in many matches
let route (s: string) =
    match Int32.TryParse s with
    | true, n -> handleId n
    | _ -> match s with "all" -> handleAll() | _ -> notFound()

// After: name the concept once, reuse it
let (|Int|_|) (s: string) =
    match Int32.TryParse s with
    | true, n -> Some n
    | _ -> None

let route = function
    | Int n -> handleId n
    | "all" -> handleAll()
    | _     -> notFound()
```
**Don't** introduce an active pattern used in exactly one place where a `when` guard reads just as well — it adds indirection for no reuse.

### Wildcard defeats DU evolution → enumerate cases
```fsharp
type Event = Created | Updated | Deleted | Archived

// Before: add a 5th case later and this SILENTLY keeps returning "other" — no compiler help
let label e =
    match e with
    | Created -> "new"
    | _       -> "other"

// After: every case is named, so a new case is a COMPILE ERROR here until you handle it
let label e =
    match e with
    | Created             -> "new"
    | Updated             -> "changed"
    | Deleted | Archived  -> "gone"
```
Use `[<RequireQualifiedAccess>]` on DUs whose case names (`Created`, `Open`, `None`…) would collide with other DUs or shadow library names.

---

## 7. Records & Immutability

### Data class → record
```fsharp
// Before (C#-style)
type Point(x: float, y: float) =
    member val X = x with get, set
    member val Y = y with get, set

// After: structural equality, immutable, with-copy, one line
type Point = { X: float; Y: float }
```

### Clone-then-mutate → copy-and-update
```fsharp
// Before
let updated = Point(p.X, p.Y)
updated.Y <- p.Y + 1.0

// After
let updated = { p with Y = p.Y + 1.0 }
```

### Opaque tuple → anonymous record (local shapes)
```fsharp
// Before: what are .Item1/.Item2 at the call site?
let stats xs = (List.length xs, List.sum xs, List.average xs)

// After: self-documenting, no new named type needed
let stats xs = {| Count = List.length xs; Sum = List.sum xs; Avg = List.average xs |}
```

---

## 8. Async / Task

### Sequential independent awaits → concurrent fan-out
```fsharp
// Before: three round-trips back to back
let load () = async {
    let! users     = fetchUsers ()
    let! orders    = fetchOrders ()       // doesn't depend on users
    let! inventory = fetchInventory ()    // doesn't depend on either
    return users, orders, inventory
}

// After (HETEROGENEOUS types): Async.StartChild preserves distinct types AND the tuple.
// Do NOT use Async.Parallel here — it requires all elements to be Async<'T> of the SAME 'T
// and returns a 'T[], not a tuple, so it neither type-checks nor drops in.
let load () = async {
    let! usersChild     = Async.StartChild (fetchUsers ())
    let! ordersChild    = Async.StartChild (fetchOrders ())
    let! inventoryChild = Async.StartChild (fetchInventory ())
    let! users     = usersChild
    let! orders    = ordersChild
    let! inventory = inventoryChild
    return users, orders, inventory
}

// Async.Parallel is the right tool only for SAME-TYPED work over a collection:
let loadAll (ids: int list) = async {
    let! results =                              // results : User[]
        ids |> List.map fetchUser |> Async.Parallel
    return results
}
// Caveat: Async.Parallel launches everything at once. For large fan-out, bound it:
//   Async.Parallel(computations, maxDegreeOfParallelism = 8)
// and note failure semantics differ (runs all, aggregates) vs let! (stops at first).
```

### RunSynchronously mid-flow → let!/do!
```fsharp
// Before: blocks a thread inside an async workflow
let handler () = async {
    let user = loadUser () |> Async.RunSynchronously   // anti-pattern
    return render user
}

// After: stay async; run-sync only at the real entry point
let handler () = async {
    let! user = loadUser ()
    return render user
}
```

### async vs task — pick deliberately
```fsharp
// Prefer async for F#-native code: cold, compositional, supports async try/finally.
let work () = async { ... }

// Use task when interop with Task-returning .NET APIs dominates, or for hot-start/perf.
// Caveat: task does NOT do tail calls — avoid it for deep recursive async loops.
let dbCall () = task {
    let! rows = cmd.ExecuteReaderAsync()   // .NET API returns Task
    return rows
}
```

---

## 9. OO Residue → Functional

### Single-method class → function
```fsharp
// Before
type Greeter(greeting: string) =
    member _.Greet(name: string) = sprintf "%s, %s!" greeting name

// After: a curried function captures the same "config + call"
let greet greeting name = $"{greeting}, {name}!"
let hello = greet "Hello"      // partial application replaces the constructor
```

### One-off interface impl → object expression
```fsharp
// Before: a whole named type used once
type ConsoleLogger() =
    interface ILogger with
        member _.Log msg = printfn "%s" msg

// After
let logger =
    { new ILogger with
        member _.Log msg = printfn "%s" msg }
```

### Static utility class → module
```fsharp
// Before
type MathUtils =
    static member Square x = x * x
    static member Cube x = x * x * x

// After
module MathUtils =
    let square x = x * x
    let cube x = x * x * x
```

### Manual dispose → use / use!
```fsharp
// Before
let readAll path =
    let reader = new StreamReader(path: string)
    try reader.ReadToEnd()
    finally reader.Dispose()

// After: `use` disposes at scope exit; `use!` does the same inside async/task CEs
let readAll path =
    use reader = new StreamReader(path: string)
    reader.ReadToEnd()
```

### null at the boundary → Option
```fsharp
// Before: F# code passing null around
let tryHeader (req: Request) : string = req.Headers.["X-Trace"]   // may be null

// After: convert once, work in Option-land internally
let tryHeader (req: Request) : string option = req.Headers.["X-Trace"] |> Option.ofObj
```

---

## 10. Ecosystem & Misc

### Hand-parsed argv → Argu
```fsharp
// Before
let argv = Environment.GetCommandLineArgs()
let input = argv |> Array.tryItem 1
let verbose = argv |> Array.contains "--verbose"

// After
open Argu
type Args =
    | [<MainCommand; ExactlyOnce>] Input of path: string
    | Verbose
    interface IArgParserTemplate with
        member this.Usage =
            match this with
            | Input _ -> "input file path"
            | Verbose -> "enable verbose output"

let results = ArgumentParser.Create<Args>().Parse(argv)
let input = results.GetResult Input
```

### sprintf concatenation → interpolated strings
```fsharp
// Before
let msg = sprintf "user %s has %d points" name points
// After
let msg = $"user {name} has {points} points"
```

### Magic name strings → nameof
```fsharp
// Before
if propName = "Email" then ...
// After
if propName = nameof user.Email then ...
```

---

## Expert References

Authoritative sources behind these patterns:

- **F# for Fun and Profit** (Scott Wlaschin) — "Designing with types" series (make illegal states unrepresentable, single-case DUs), "Choosing between collection functions," Railway-Oriented Programming.
- **Domain Modeling Made Functional** (Scott Wlaschin) — DUs/records for domain design; the canonical "types over runtime checks" treatment.
- **F# Language Reference** (Microsoft Learn) — Active Patterns; Async vs Task expressions (async preferred; task for .NET interop; task lacks tail calls); Collection Types (Seq laziness/re-enumeration pitfalls).
- **FsToolkit.ErrorHandling** (demystifyfp) — `result`/`option`/`asyncResult`/`taskResult` and the applicative `validation` CE for error accumulation.
- **Argu** documentation — declarative, type-safe CLI parsing.
- **F# Style Guide / component design guidelines** (Microsoft Learn) — when to keep annotations on public APIs; module vs class conventions.
