---
name: arch-conformance
description: Discovers a codebase's intended architecture (from docs, ADRs, naming, package structure, and dependency direction), then deploys parallel agent teams to find code that violates it — layer skips, reverse dependencies, cycles, bypassed boundaries, leaked cross-cutting concerns, mutant duplicates. Writes findings to plan/*.md files. Use when the user asks to "find architecture violations", "check architectural conformance", "detect architectural drift", "find layer violations", "find boundary violations", "where does the code break the architecture", "audit code against the design", "find code that doesn't follow the architecture", or hands over a repo asking "what doesn't conform". Triggers even when the user does not explicitly say "conformance". Do NOT use for general architectural smells / coupling-cohesion review (use arch-audit), specific bugs (use bug-hunt), single-file or recent-diff review (use code-scrutiny), runtime or security hardening (use app-harden), or designing a new architecture from scratch.
argument-hint: "[scope: path, module name, or blank for whole repo]"
allowed-tools: "Read Glob Grep Bash Agent Write WebSearch WebFetch"
---

# Architecture Conformance

You discover a codebase's architecture, then hunt for code that violates it. Output is a set of `plan/*.md` files: one architecture summary, one prioritized findings index, and one file per violation category that produced findings.

This is **architecture conformance checking** in the academic sense (Murphy & Notkin, *Software Reflexion Models*, 1995). The technique has three primitives, and every finding you produce maps to one of them:

- **Convergence** — intended structure ∩ actual structure. Good. Don't report.
- **Divergence** — present in code but not part of the architecture. This is the **erosion** to hunt for (Perry & Wolf, 1992).
- **Absence** — in the architecture but missing from code. Worth reporting only when the missing piece causes the divergences (e.g., no repository abstraction → views call DB directly).

Your edge over a generic "find issues" pass: every finding cites a *specific architectural rule* that was discovered or stated, plus *concrete locations* where it was violated. Findings without a named rule are out of scope — they belong in `arch-audit`. ultrathink each judgment call about whether a deviation is real erosion or intentional drift.

## Context

- Scope argument: `$ARGUMENTS` (if present, restrict analysis; otherwise scan whole repo)
- Working dir: !`pwd`
- In git repo: !`git rev-parse --is-inside-work-tree 2>/dev/null || echo "not a git repo"`
- Tracked files: !`git ls-files 2>/dev/null | wc -l | tr -d ' ' || echo unknown`
- Top-level: !`ls -la 2>/dev/null | head -25`
- Existing plan dir: !`ls plan/ 2>/dev/null | head -10 || echo "(no plan/ dir yet)"`
- Existing architecture docs: !`ls ARCHITECTURE.md docs/architecture* docs/adr* docs/decisions* 2>/dev/null || echo "(none found)"`

## When NOT to use

- **Codebase has no discernible architecture** (no clear layering in dirs, no naming conventions, mixed responsibilities everywhere) — there is nothing to conform to. Tell the user this and recommend `arch-audit` to find structural problems first.
- **Tiny codebase** (<2k LOC, <30 files) — architecture has not had room to drift; conformance checking is over-engineered. Recommend `code-scrutiny`.
- **User wants general code review** — they want every kind of issue, not just architectural divergences. Recommend `code-scrutiny` or `arch-audit`.
- **Greenfield / pre-architecture** — no design exists yet to conform to. Help them define one instead.

If you decline, say so plainly and name the alternative.

## The four phases

You orchestrate four phases. Phases 1, 3, and 5 are **agent fan-outs** — multiple `Agent` calls in a single message so they run concurrently. Phase 2 and 4 are synthesis steps you do yourself with the gathered evidence.

```
Phase 1: Discover architecture     (parallel — 4–6 agents)
Phase 2: Codify rules              (synthesis — you)
Phase 3: Hunt violations            (parallel — one agent per rule)
Phase 4: Triage and dedupe          (synthesis — you)
Phase 5: Write plan/*.md            (parallel — one Write per file)
```

If the codebase is large (>50k LOC or >500 files), prefer the `Explore` subagent_type for Phase 1 and Phase 3 to keep your context clean.

---

### Phase 1 — Discover the architecture (parallel fan-out)

Launch all of these in a **single message** with multiple `Agent` tool calls:

1. **Stated architecture agent** — read `README.md`, `ARCHITECTURE.md`, `CLAUDE.md`, `docs/`, `docs/adr*`, `docs/decisions*`, `CONTRIBUTING.md`. Extract: declared style (layered, hexagonal, clean, MVC, microservices, event-driven, CQRS, DDD, modular monolith), declared layers/modules, declared dependency rules, any explicit "X must not import Y" statements, ADR titles. Quote directly. If nothing is stated, say so.

2. **Build & manifest agent** — read package manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, `pom.xml`, `build.gradle`, `*.csproj`, `*.sln`, workspace files, Nx/Turborepo configs, Bazel `BUILD` files). Identify package boundaries, declared internal vs. published APIs, workspace structure. Note any tooling that already enforces architecture (ArchUnit, dependency-cruiser, import-linter, eslint boundaries, NetArchTest, ts-arch, deptrac).

3. **Top-level structure agent** — `ls`/`tree` the top 2–3 directory levels under `src/` (or equivalent). Classify the apparent organization: by-layer (`controllers/services/repositories`), by-feature (`users/orders/billing`), by-bounded-context (DDD), hex (`domain/application/infrastructure/adapters`), clean/onion, framework-default (`pages/api/components`), or hybrid. Report the dominant pattern and any obvious exceptions.

4. **Naming convention agent** — grep for class/file suffixes that telegraph an architectural style: `*Repository*`, `*Service*`, `*Controller*`, `*UseCase*`, `*Handler*`, `*Adapter*`, `*Port*`, `*Gateway*`, `*Aggregate*`, `*Entity*`, `*ValueObject*`, `*Command*`, `*Query*`, `*Event*`, `*Saga*`, `*Reducer*`, `*Store*`, `*Provider*`. Count occurrences. The naming gives away the intended pattern even when docs are silent.

5. **Dependency-direction agent** — sample imports to map the actual dependency graph at the top level. For the 3–5 most populated top-level directories, run targeted greps: which directories import from which? Tally the directions. The dominant direction reveals the intended layering even when nothing is documented. Flag any directory pair where imports flow both ways — that is a candidate cycle for Phase 3.

6. **Test structure agent** *(if test dir exists)* — tests usually mirror the production module layout and reveal real dependencies through their mocks/fixtures. Note the structure and any modules whose tests mock `>5` collaborators (signal of high coupling, often crosses architectural boundaries).

Each agent should return ≤300 words of structured findings — facts and quotes, not opinions.

---

### Phase 2 — Codify the architectural rules (synthesis, you)

You now have evidence from up to 6 angles. Do this yourself; do not delegate synthesis.

**Decide the source of truth:**

- **Stated architecture** (docs, ADRs, ARCHITECTURE.md, CLAUDE.md mention rules) → use it directly. Quote the source.
- **Strong implicit architecture** (clear naming + consistent dirs + dependency direction enforced ≥80%) → infer rules and proceed. Label every inferred rule "INFERRED" so the user can override.
- **Weak/contradictory architecture** (inconsistent dirs, conflicting naming, mixed dependency directions) → STOP. You cannot conformance-check against an architecture that does not exist. Write only `plan/00-architecture.md` describing what you observed and what is unclear, then ask the user to confirm or specify rules before proceeding to Phase 3.

**Produce a numbered rule list.** Each rule must be:

- **Mechanically checkable** — phrasable as a forbidden import / call pattern, a forbidden directory dependency, or a structural property a script could verify. "Domain code should be clean" is not a rule. "Files under `src/domain/**` must not import from `src/infrastructure/**` or any third-party persistence/HTTP library" is.
- **Sourced** — annotated with where the rule came from: `(stated: ARCHITECTURE.md L42)`, `(stated: ADR-007)`, `(inferred from: 87/91 files in src/domain/ have no infrastructure imports)`.
- **Scoped** — names the directories/modules/layers it applies to.

A typical project yields 5–15 rules. If you have more than 20, you are over-fitting to the code instead of capturing intent — collapse them.

Consult `references/architectural-styles.md` for the canonical rule sets of common styles (layered, hexagonal, clean, DDD, MVC, CQRS) — these are ready to drop in once you've identified the style.

---

### Phase 3 — Hunt violations (parallel fan-out, one agent per rule)

This is the heaviest parallel wave. Launch **one agent per rule** in a single message. Each agent's prompt should be tightly scoped to its rule.

Agent prompt template (paraphrased — adapt per rule):

> You are checking ONE architectural rule. Rule: **[exact rule text]**. Source: **[stated/inferred citation]**. Search the codebase for violations using grep/glob — try at least 2 different search angles to avoid missing variations. For each violation, return: (a) the file path and line number, (b) the exact code excerpt (1–3 lines), (c) confidence (HIGH/MEDIUM/LOW per the calibration in `references/violation-taxonomy.md`), (d) whether the violation has any obvious justification visible nearby (a comment, an `// allow-` annotation, a lint disable). Return at most 30 violations; if there are more, return the 30 worst plus a count of the rest. If you find zero violations, return "CLEAN" — that is a valid and preferred answer.

Consult `references/violation-taxonomy.md` for the catalog of the ~15 common violation types and the grep patterns that detect each. Consult `references/parallel-orchestration.md` for guidance on batching, backpressure, and when to use `Explore` vs. `general-purpose` subagent types.

**Anti-hallucination rules — these are non-negotiable:**

- Every violation must point to a real file path and line number you can verify by Read.
- "CLEAN" is a valid finding for any rule. Do NOT manufacture borderline cases to pad a rule that came up empty.
- If a rule turns out to be unfalsifiable in this codebase (e.g., the layer it constrains does not exist), drop it from the report rather than reaching for findings.
- Spot-check 2–3 random findings per rule by Reading the cited file. If the line cited does not exist or does not say what the agent claimed, treat that agent's whole batch as suspect and re-run.

---

### Phase 4 — Triage and dedupe (synthesis, you)

Sub-agents return raw violations. You now turn them into a usable plan.

1. **Dedupe** — the same file may show up under multiple rules (one bad import can violate 3 rules). Pick the most specific rule and keep the finding there.
2. **Cluster** — if 30 files all violate the same rule for the same reason (e.g., 30 React components importing the DB client), report it as one finding with a file list, not 30 separate entries. Big lists collapse to "[N files] — see Appendix A".
3. **Prioritize** — rank by `(blast radius) × (confidence) × (fixability)`. A HIGH-confidence violation in a heavily-imported module is top priority. A LOW-confidence one-off in an isolated experiment is bottom.
4. **Drop noise** — anything with an explicit `// arch-allow` annotation or a nearby ADR exception comment is dropped, not flagged.
5. **Confidence calibration** — re-check that confidence labels follow the calibration:
   - **HIGH** — clear violation, no plausible justification, the rule is stated (not inferred).
   - **MEDIUM** — clear violation, but the rule is inferred OR there is a plausible reason it could be intentional.
   - **LOW** — pattern fragmentation or stylistic deviation; might be intentional; the user should decide.

If a category produced zero findings, do not write a file for it. Empty findings files are noise.

---

### Phase 5 — Write plan/*.md (parallel writes)

Create the `plan/` directory if absent. Then issue all `Write` calls in a single message. File set:

| File | Always written? | Contents |
|---|---|---|
| `plan/00-architecture.md` | Yes | Discovered architecture: declared style, layers/modules, full numbered rule list with sources, what was unclear. |
| `plan/99-summary.md` | Yes | Top 10 findings across all categories, ranked by priority. Each entry: 2-line summary + link to detail file. Counts per category. |
| `plan/01-layer-violations.md` | If findings | Cross-layer dependency violations (e.g., domain → infra, UI → DB). |
| `plan/02-cycles.md` | If findings | Cyclic dependencies between modules. |
| `plan/03-bypassed-boundaries.md` | If findings | Imports of internals across module/package boundaries; broken encapsulation. |
| `plan/04-leaked-concerns.md` | If findings | Cross-cutting concerns (logging, persistence, transport, auth) appearing in wrong layer. |
| `plan/05-pattern-fragmentation.md` | If findings | Same problem solved multiple ways across the codebase (multiple HTTP clients, multiple state-management approaches, etc.). |
| `plan/06-shared-state.md` | If findings | Mutable state shared across architectural boundaries without coordination. |
| `plan/07-naming-violations.md` | If findings | Files/classes whose names imply a role they don't play (e.g., `UserService` that talks to the DB directly). |
| `plan/08-mutant-duplicates.md` | If findings | Near-identical code paths with small variations — signal of copy-paste across boundaries. |

Add additional category files if the codebase has architecture-specific rules that don't fit the above (e.g., `plan/09-cqrs-violations.md` for a CQRS codebase).

If a `plan/` directory already exists with content, do NOT overwrite without asking. Either rename existing files (e.g., `plan/archive-YYYYMMDD/`) or write to `plan/conformance-YYYYMMDD/` instead.

## Output format — concrete example

This is the format every category file uses. Lead with one of these per finding:

```markdown
### V-LAY-001 — UI components import database client directly
**Confidence:** HIGH
**Rule violated:** R-3 — Files under `src/web/**` must not import from `src/infrastructure/db/**` (stated: ARCHITECTURE.md §3.2 "Persistence access goes through repositories")
**Locations:**
- `src/web/UserDashboard.tsx:42` — `import { db } from '../../infrastructure/db'`
- `src/web/admin/AuditLog.tsx:18` — `import { db } from '../../infrastructure/db'`
- `src/web/billing/InvoiceList.tsx:9` — `import { db } from '../../infrastructure/db'`
- _(plus 4 more — see Appendix A)_
**Why it matters:** UI components become coupled to the schema; ORM migrations break unrelated views. Spreads quickly because the pattern is copy-pasted.
**Suggested fix:** Add `UserRepository.findById`, `AuditRepository.list`, `InvoiceRepository.listForCustomer` to `src/application/`; have these views call those.
**Effort:** ~1 day for the 7 sites; new repositories are ~10 LOC each.
```

The fields are: `### V-<CATEGORY>-<NNN>`, **Confidence**, **Rule violated** (with source), **Locations** (concrete paths + line numbers + 1-line excerpt), **Why it matters** (1–2 sentences), **Suggested fix**, **Effort**. Skip "Effort" if you genuinely cannot estimate.

## Quality bar — read this before reporting

A useful conformance report is one a senior engineer would put on a wall and work through. Before delivering, verify:

- **Every finding names a rule from `plan/00-architecture.md`.** A finding without a rule is a smell, not a conformance violation.
- **Every finding has a verifiable file:line.** No "around line 50", no "in the user module somewhere".
- **No category file is empty.** If a rule produced zero findings, mention that under "Clean rules" in `99-summary.md` and don't create the file.
- **Inferred rules are clearly labeled.** The user must be able to scan `00-architecture.md` and override anything you guessed wrong about.
- **`99-summary.md` ranks the top 10.** Not the top 100. The point of the summary is to tell the user *where to start*.

If after deduping you have <3 findings total, that's the answer — say so plainly in `99-summary.md`. The codebase conforms. Do not pad with stylistic nitpicks to make the report look substantial.
