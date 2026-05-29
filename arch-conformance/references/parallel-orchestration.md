# Parallel Orchestration — Maximizing Throughput

This skill is bottlenecked by latency, not by token cost or compute. The faster you fan out, the faster the user gets results. Use this reference to size waves correctly and avoid the common parallelism mistakes.

---

## The mental model

There are three waves. Each wave is **one message that contains many `Agent` tool calls**. Within a wave, agents run in parallel. Between waves, the main agent (you) does synthesis serially.

```
Wave 1 — Discovery       (4–6 agents in parallel)     ~30–60s wall clock
   ↓
You synthesize Phase 2   (no agents)                    ~10–30s
   ↓
Wave 2 — Conformance     (1 agent per rule, 5–15)     ~60–120s wall clock
   ↓
You synthesize Phase 4   (no agents)                    ~10–30s
   ↓
Wave 3 — Write           (one Write per file, parallel) ~5–15s
```

Total wall clock: **2–4 minutes** for a medium repo. A naive sequential approach would take 15–25 minutes for the same work.

The cost of an extra parallel agent is small. The cost of a sequential pass is large. **Err on the side of more parallelism, smaller agent prompts.**

---

## Rules of fan-out

### 1. One message, multiple Agent calls
This is the only way agents actually run in parallel. If you split them across messages, they serialize. Always batch into a single message:

```
GOOD:  one message containing 6 Agent({...}) tool calls
BAD:   six messages each containing one Agent({...}) call (these are serial)
```

### 2. Cap each wave at ~12 agents
Above 12 you start hitting overhead and your synthesis step becomes hard to manage. If you have 18 rules to check, batch them: 12 in wave 2a, 6 in wave 2b, then synthesize once.

### 3. Each agent prompt should be ≤500 words
Long prompts slow agent startup. Keep them tight: the rule, the search hints, the return format, the anti-hallucination clause. Move shared context (architectural style, output format) into the prompt only by reference (`see references/violation-taxonomy.md §V-LAY`) — but remember sub-agents may not auto-load your references, so quote the critical bits inline.

### 4. Each agent returns ≤300 words of structured output
Long agent responses defeat the point of fanning out. Demand structured findings, not narrative. A return of 30 violations × 4 lines each = 120 lines is the upper bound. If a sub-agent wants to return more, it should return the top N + a count.

### 5. Pick the right subagent_type
- `Explore` — read-only fast search across the codebase. **Default for Phase 1 and Phase 3.** Optimized for finding code; cannot edit or write.
- `general-purpose` — when the agent needs to read more selectively, run scripts, or do moderate analysis. Slower than Explore but more capable.
- `Plan` — for architectural design / implementation plans. Not used in this skill.

### 6. Independence is mandatory within a wave
Agents in the same wave must NOT depend on each other's output. If agent B needs agent A's result, they belong in different waves. The whole point of parallel fan-out is independence.

---

## Wave 1 templates — Discovery agents

Drop these into individual `Agent` calls in a single message. Each is `subagent_type: "Explore"` (or `general-purpose` if it needs more than read-only).

### Agent 1.1 — Stated architecture
```
Read these files if they exist (skip silently if not):
  README.md, README.*, ARCHITECTURE.md, ARCHITECTURE.*, CLAUDE.md,
  CONTRIBUTING.md, docs/architecture*, docs/adr*, docs/decisions*, docs/design*
Extract:
  - Declared architectural style (layered, hexagonal, clean, MVC, microservices, DDD, CQRS, event-driven, etc.)
  - Declared layers/modules and their responsibilities
  - Any explicit "X must not Y" or "X depends on Y" rules
  - ADR titles (just the titles, list them)
Return structured: STYLE, LAYERS, RULES, ADRS, NOTES.
If nothing is stated anywhere, return "NO STATED ARCHITECTURE" and stop.
≤300 words.
```

### Agent 1.2 — Build & manifest
```
Find and read all manifest files: package.json, pyproject.toml, Cargo.toml,
go.mod, pom.xml, build.gradle*, *.csproj, *.sln, deno.json, bun.lockb,
nx.json, turbo.json, lerna.json, pnpm-workspace.yaml, BUILD, BUILD.bazel.
Extract:
  - Workspace structure (monorepo? single package? polyrepo seen via gitmodules?)
  - Internal vs published packages
  - Declared package "exports", "main", "files" boundaries
  - Architecture-enforcement tools already configured: ArchUnit, dependency-cruiser,
    import-linter, eslint-plugin-boundaries, NetArchTest, ts-arch, deptrac, jdeps
Return: WORKSPACE_TYPE, PACKAGES, BOUNDARIES, ENFORCEMENT_TOOLS.
≤300 words.
```

### Agent 1.3 — Top-level structure
```
List directories at depth 1, 2, and 3 under the source root (src/, lib/, app/,
or repo root if no source dir exists). Use ls/find, not full file listings.
Classify the dominant organization:
  - by-layer (controllers/services/repositories)
  - by-feature (users/orders/billing)
  - by-bounded-context (DDD)
  - hex/clean (domain/application/infrastructure)
  - framework-default (Next.js pages/api/components, Rails app/controllers/models)
  - hybrid (describe)
Note any directory that is an obvious exception to the dominant pattern.
Return: ROOT, LAYOUT_TREE (depth 2), DOMINANT_PATTERN, EXCEPTIONS.
≤300 words.
```

### Agent 1.4 — Naming conventions
```
Grep the codebase for class/file/struct/interface name suffixes that telegraph
architectural intent. Count occurrences for each:
  Repository, Service, Controller, UseCase, Interactor, Handler, Adapter, Port,
  Gateway, Aggregate, Entity, ValueObject, Command, Query, Event, Saga,
  Reducer, Store, Provider, Factory, Manager, Worker.
Use grep with appropriate file extensions.
Report only suffixes with count >= 3.
The dominant naming convention often reveals the intended style even when docs
are silent.
Return: SUFFIX_COUNTS (table), INFERRED_STYLE_SIGNAL (one sentence).
≤200 words.
```

### Agent 1.5 — Dependency direction
```
For the 3-5 most populated top-level source directories, sample imports to
determine which direction code flows:
  - For each pair (A, B) of top-level dirs, count files in A that import from B.
  - Use grep -rn for "from <B>", "import <B>", "use <B>::", "import \"<B>" etc.
    depending on language.
Report a coarse adjacency table: rows=source dir, cols=imported dir, cells=count.
Flag any pair where imports flow both ways (candidate cycle).
The dominant direction reveals the intended layering even when nothing is documented.
Return: ADJACENCY_TABLE, DOMINANT_DIRECTIONS (top 5), BIDIRECTIONAL_PAIRS.
≤300 words.
```

### Agent 1.6 — Test structure (only if test dir exists)
```
List the test directory structure. Compare to production source layout — does it
mirror? If a test file mocks more than 5 distinct collaborators, note the
production module under test as "high-coupling-suspect".
Return: TEST_LAYOUT, MIRRORS_PROD (yes/partial/no), HIGH_COUPLING_SUSPECTS (list).
≤200 words.
```

---

## Wave 2 template — Conformance agents

This is the heaviest wave. One agent per rule from Phase 2. Use `subagent_type: "Explore"`.

```
You are checking ONE architectural conformance rule.

RULE: <exact rule text from Phase 2>
SOURCE: <stated/inferred citation>

Search the codebase for violations.
Try at least 2 different search angles to avoid missing variations
(e.g., for "domain must not import infrastructure" try both:
  grep "from .*infrastructure" src/domain/
  grep "import .*infrastructure" src/domain/
and check for re-exports through barrel files).

Detection hints for this category: <copy from violation-taxonomy.md>

Exclude these paths from results: tests, generated code, vendor/,
node_modules/, dist/, build/, target/.
Excluded composition-root files: <list main.*, wire.go, etc.>

For each violation return:
  PATH:LINE — <code excerpt 1-3 lines>
  CONFIDENCE — HIGH | MEDIUM | LOW (per calibration)
  JUSTIFICATION_NEARBY — none | <quote of nearby comment/annotation>

Return at most 30 violations. If more exist, return the 30 worst plus the count.
If you find ZERO violations, return exactly: "CLEAN"
That is a valid and preferred answer. Do not invent borderline cases to fill quota.

≤300 words.
```

---

## Backpressure and retries

### When an agent returns junk
If a sub-agent returns vague findings without file:line, or invents files, treat the whole batch as suspect. Re-dispatch that one rule with a more constrained prompt (cite the exact grep command to start with).

### When an agent times out / errors
Re-dispatch in the next wave. Do not block other agents.

### When the same finding shows up under multiple rules
Normal. Phase 4 (your synthesis) is responsible for deduping. Sub-agents need not coordinate.

### When a wave produces too many findings to triage
Emit `plan/00-architecture.md` with the rules and a `plan/INTERIM-volume-warning.md` explaining the codebase has substantial drift, then ask the user whether they want to (a) narrow scope, (b) raise the confidence floor to HIGH only, or (c) proceed with the full report.

---

## Anti-patterns of fan-out

- **Spinning up an agent that needs another agent's result first.** Move it to the next wave.
- **One mega-agent that does "everything for layer X".** Defeats the point. Split per rule.
- **Sub-agents that write files.** No. Synthesis is yours; only Wave 3 writes.
- **Sub-agents that recurse into more sub-agents.** Forbidden — recursion explodes wall-clock and context. Sub-agents in this skill are leaves.
- **Re-running the same wave with minor tweaks instead of synthesizing.** If wave 2 returned mediocre results, the fix is usually a better rule list (back to Phase 2), not a re-run with the same rules.

---

## Wave 3 — Write

After Phase 4 deduping, issue all `Write` calls in a single message. They run in parallel. Typical wave: 4–10 file writes (`plan/00-architecture.md`, `plan/99-summary.md`, plus one per non-empty category).

Do not try to write all categories sequentially. Even though Write is fast, the latency adds up over 8 files; in parallel it's one round-trip.
