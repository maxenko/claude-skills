# Community Defaults — Boring-Technology Baseline

When a refactor prescription names a concrete library, runtime, or tool (not just a pattern), default to the ecosystem's dominant choice. "Boring technology" (Dan McKinley) carries real leverage: larger hiring pool, better docs, mature tooling, AI-assistant familiarity, composable ecosystem.

Obscure alternatives are fine *if they fit the problem* — but only when the finding names a concrete reason the default falls short (license constraint, platform target, runtime footprint, measured performance gap, genuinely better domain fit). When off-default, add a **Why-not-default** line in the prescription.

**Important**: this list is a snapshot as of writing. Communities shift. When unsure whether a default is still current, run a web search (`WebSearch` for "[language/ecosystem] default [concern] 2026" or similar) and prefer the primary source — framework docs, maintainer blogs, published benchmarks — over listicles.

## Defaults by ecosystem

### Backend languages

| Ecosystem | Concern | Community default | Notes |
|---|---|---|---|
| **Rust** | async runtime | **Tokio** | not async-std (unmaintained) or smol (niche) |
| Rust | HTTP server | Axum | Actix-web still widely used |
| Rust | error handling | `thiserror` (libraries) + `anyhow` (apps) | |
| Rust | embedded async | Embassy | Tokio doesn't target `no_std` |
| **Go** | concurrency | stdlib goroutines + channels + `errgroup` | no framework by convention |
| Go | HTTP | stdlib `net/http` + chi or gin | |
| Go | tasks / queues | stdlib + DB-backed queue (river, pgmq) | |
| **Python** | async | asyncio + httpx + FastAPI | |
| Python | sync web | Django / Flask | |
| Python | tasks | Celery (mature) / RQ / Arq | |
| **Node.js** | backend | Express / Fastify / NestJS | |
| Node.js | client state | Zustand / Redux Toolkit / TanStack Query | |
| **JVM** | actors | Akka or Pekko (post-license-change fork) | |
| JVM | reactive streams | Project Reactor | |
| JVM | web | Spring Boot; Kotlin + Ktor for Kotlin | |
| **.NET** | actors | Akka.NET or Orleans (virtual actors) | |
| .NET | messaging | MassTransit / NServiceBus | |
| .NET | web | ASP.NET Core | |
| **Elixir / Erlang** | everything | stdlib OTP (GenServer, Supervisor, Task, Agent) | Phoenix + LiveView for web |
| **F#** | actor / agent | stdlib `MailboxProcessor` | |
| F# | web | Giraffe / Saturn | |

### Frontend

| Concern | Community default |
|---|---|
| Meta-framework | Next.js / Remix / SvelteKit / Nuxt |
| Client state | Zustand (simple) / Redux Toolkit (large apps) |
| Server state | TanStack Query (formerly React Query) |
| State machines | XState |
| Forms | React Hook Form + Zod; similar combos per framework |

### Infrastructure / integration

| Concern | Community default | Notes |
|---|---|---|
| Workflow orchestration (general) | **Temporal** | Cadence was predecessor |
| Workflow orchestration (AWS-native) | AWS Step Functions | |
| Workflow orchestration (data) | Airflow / Dagster / Prefect | |
| Messaging — high-throughput durable log | **Kafka** | Redpanda for Kafka-compatible alternative |
| Messaging — classic work-queue | **RabbitMQ** | |
| Messaging — lightweight pub/sub | NATS | |
| Messaging — already-have-Redis | Redis Streams | |
| CDC / outbox relay | **Debezium** | |
| Observability (traces/metrics/logs) | **OpenTelemetry** | vendor-neutral instrumentation |
| Metrics + dashboards | Prometheus + Grafana | |

## When to deviate

A finding that picks off-default must include a **Why-not-default** line. Examples of valid deviations:

- "Why not Tokio? The project targets `no_std` embedded — Embassy is the community default there."
- "Why not Kafka? We already run NATS for other flows and message volume is <1k/s — adding Kafka ops is overhead without payoff."
- "Why not Redux Toolkit? The app is <10 screens with no complex cross-component state — Zustand is lighter and equally idiomatic."

Every deviation is itself an architectural decision worth attributing.
