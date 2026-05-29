# Architectural Styles — Identification and Rule Sets

Drop-in rule sets for the most common architectural styles. Once Phase 1 has identified the style, lift the relevant rules below into Phase 2 and adapt them to the codebase's actual directory names.

The rules are written in checkable form: `[scope] must / must not [import|call|implement] [target]`. Replace `<dir>` placeholders with the codebase's real paths.

---

## 1. Layered (n-tier)

**Telltale signs:** Top-level dirs like `controllers/`, `services/`, `repositories/`, `models/`, `dao/`. README mentions "n-tier" or "MVC" (in the n-tier sense, not the GUI sense). Imports flow downward more than 90% of the time.

**Standard rules:**
- R-L1: `<controllers>/**` may import `<services>/**` and `<models>/**`. Must not import `<repositories>/**` directly.
- R-L2: `<services>/**` may import `<repositories>/**` and `<models>/**`. Must not import `<controllers>/**`.
- R-L3: `<repositories>/**` may import `<models>/**` and the persistence library. Must not import `<services>/**` or `<controllers>/**`.
- R-L4: `<models>/**` must not import any other layer.
- R-L5: No layer may skip a layer below it. Controllers do not call repositories directly.

**Common violations:** Layer skipping (controller → repo), reverse flow (model imports service for "convenience"), service classes that hold HTTP request objects.

---

## 2. Hexagonal / Ports & Adapters (Cockburn)

**Telltale signs:** Directories named `domain/`, `application/`, `adapters/`, `ports/`, `infrastructure/`. Class names ending in `Port`, `Adapter`, `Driver`. README cites Alistair Cockburn or "ports and adapters".

**Standard rules:**
- R-H1: `<domain>/**` must not import `<adapters>/**`, `<infrastructure>/**`, or any third-party library other than language stdlib.
- R-H2: `<application>/**` may import `<domain>/**` and `<ports>/**`. Must not import `<adapters>/**` or `<infrastructure>/**`.
- R-H3: `<ports>/**` are interfaces only. No concrete classes, no I/O, no third-party imports.
- R-H4: `<adapters>/**` implement `<ports>/**`. Adapters may import `<infrastructure>/**` and third-party libs.
- R-H5: Wiring (DI registration, composition root) is the *only* place adapters and ports meet by name. Typically `main.*` or `composition_root.*`.

**Common violations:** Domain classes annotated with ORM annotations (`@Entity`), domain entities throwing HTTP exceptions, application code instantiating adapters directly instead of receiving ports.

---

## 3. Clean / Onion (Martin / Palermo)

**Telltale signs:** Concentric rings: `entities/` → `usecases/` (or `interactors/`) → `interface_adapters/` → `frameworks_drivers/`. Strict dependency rule cited.

**Standard rules:**
- R-C1: Source-code dependencies point only inward. An outer ring may import an inner ring; the reverse is forbidden.
- R-C2: `<entities>/**` (innermost) must not import any framework, library, or other ring.
- R-C3: `<usecases>/**` may import `<entities>/**` only. Repository contracts live here as interfaces.
- R-C4: `<interface_adapters>/**` (controllers, presenters, gateways) may import `<usecases>/**` and `<entities>/**`. Must not import `<frameworks_drivers>/**`.
- R-C5: `<frameworks_drivers>/**` (DB, web, UI) may import everything. This is where wiring lives.

**Common violations:** Use cases returning framework types (e.g., `Response` objects), entities depending on the ORM, controllers calling other controllers.

---

## 4. Domain-Driven Design (DDD) bounded contexts

**Telltale signs:** Directories named after business contexts (`billing/`, `shipping/`, `catalog/`) rather than technical layers. Files named `*Aggregate*`, `*ValueObject*`, `*DomainEvent*`, `*Repository*`. ADRs mention "bounded context" or "context map".

**Standard rules:**
- R-D1: Each bounded context (`<context>/**`) is independent. No direct imports between contexts; communication is via published events or anti-corruption layers.
- R-D2: Aggregates may be referenced from outside only by ID, never by direct object reference.
- R-D3: Cross-context queries go through a dedicated read model or query service, not by reaching into another context's repositories.
- R-D4: Domain events crossing contexts are typed in a shared `<contracts>/` or `<shared-kernel>/` directory and treated as a contract.
- R-D5: No context's domain layer imports another context's domain layer.

**Common violations:** "Shared" directory that drifts into a god module, contexts importing each other's repositories, leaking aggregates as DTOs across context boundaries.

---

## 5. MVC (frontend / GUI sense)

**Telltale signs:** `models/`, `views/`, `controllers/` (or `components/`, `containers/` for React-era variants). State management library visible (Redux, Vuex, MobX, Pinia).

**Standard rules:**
- R-M1: Views render data and emit events. They do not perform I/O, do not mutate state directly.
- R-M2: Controllers (or reducers / actions / commands) own state mutation.
- R-M3: Models are passive data holders; in TypeScript codebases, often type-only.
- R-M4: No view component imports HTTP clients, websockets, or storage APIs directly.
- R-M5: Cross-feature shared state lives in a defined store; ad-hoc globals or context-providers nested deep in the tree are forbidden.

**Common violations:** Views calling `fetch()` directly in `useEffect`, components mutating shared state without going through the store, business logic embedded in JSX render paths.

---

## 6. CQRS / Event-sourced

**Telltale signs:** Directories like `commands/`, `queries/`, `events/`, `projections/`, `handlers/`. Files named `*Command*`, `*Query*`, `*CommandHandler*`, `*Projection*`.

**Standard rules:**
- R-Q1: Command handlers must not return data. They mutate state and (optionally) emit events.
- R-Q2: Query handlers must not mutate state.
- R-Q3: Read models (`<projections>/**`) are derived only from events. They do not call command handlers, they do not write to the write model.
- R-Q4: Events are immutable, named in past tense (`OrderShipped`, not `ShipOrder`), and versioned when in production.
- R-Q5: The write model (commands + aggregates) and the read model (projections + queries) do not share storage.

**Common violations:** Command handlers returning DTOs, query handlers issuing UPDATEs (often hidden in lazy-loaded ORM relations), projections calling other projections.

---

## 7. Microservices / modular monolith

**Telltale signs:** A monorepo with workspace packages, each a deployable or potentially-deployable unit. `packages/`, `services/`, or `apps/` at top level. Each unit has its own manifest.

**Standard rules:**
- R-S1: Services may not import each other's source code directly. Communication is via HTTP, gRPC, or queue.
- R-S2: Shared code lives in published packages (`packages/<lib>`), not via relative imports across service boundaries.
- R-S3: Each service owns its data store. No cross-service DB access; no foreign keys spanning services' tables.
- R-S4: Service contracts (OpenAPI, protobuf, schema files) are versioned and live in a contracts directory.
- R-S5: Migrations for a service are owned by that service alone.

**Common violations:** Service A importing `../../service-b/src/...`, two services connecting to the same DB schema, contract files duplicated across services and drifted.

---

## Architectural style not in this list?

If the codebase clearly uses a style not listed (e.g., actor model, Elm architecture, pipes-and-filters, microkernel, plugin architecture, ECS), do a quick web search for the canonical rule set and translate it into the same `R-X1: <scope> must / must not <action>` form. The list above is not exhaustive — it is a starting kit for the most common cases.

## Hybrid codebases

Most real codebases mix styles: a hexagonal core with MVC controllers, or DDD bounded contexts each internally layered. That is fine. Identify the *outer* style first (what governs the top-level dirs), then apply the *inner* style's rules within each top-level unit. Note the hybrid in `plan/00-architecture.md` so the user can validate.
