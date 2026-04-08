# Plan Evaluation Criteria

Detailed reference for each evaluation dimension. Agents should consult the section relevant to their assigned dimension. Plan sizes (Small, Medium, Large) are defined in SKILL.md Step 2.

**Scope filter — architecture and design only**: This evaluation operates at the plan/design level. Do NOT flag implementation-level details that are resolved during coding: missing imports, missing derive macros or attributes, incorrect function signatures, type mismatches, missing trait implementations, wrong generic bounds, lifetime annotations, syntax issues, missing semicolons, variable naming typos, or any other issue that a compiler, linter, or IDE would catch. These are not plan defects. Focus on: architectural decisions, dependency ordering, design pattern choices, idiomatic alignment with the language/framework ecosystem, risk coverage, cross-cutting concerns, and structural completeness.

**Deduplication rule**: If you encounter the same issue from multiple angles, report it once under the most severe framing. Do not report the same finding multiple times.

## Quick Evaluation Checklist (Small Plans)

For plans under 1 page, check only these four items and stop:
1. **Is the "why" clear?** (Section 1.2)
2. **Does the solution match the problem?** (Section 2.1)
3. **Does it reference real code?** (Section 2.2)
4. **Are there obvious risks unaddressed?** (Section 5.1)

Everything else is optional for small plans.

---

## 1. Completeness & Structure

A complete plan enables implementation without guesswork. Only flag what's missing AND relevant to the plan's scale.

### 1.1 Goals and Non-Goals
- **Goals** state what the plan accomplishes in concrete terms
- **Non-goals** explicitly exclude things that could reasonably be in scope but aren't. "Non-goals are not negated goals" (Google design docs) — "system shouldn't crash" is not a non-goal; "ACID compliance" deliberately excluded from a cache layer is
- **Why it matters**: Without non-goals, scope creep is inevitable; implementers build speculative features
- **Hallucination guardrail**: Only flag missing non-goals if you can identify a specific scope-creep risk an implementer would likely pursue without the exclusion. "The plan doesn't mention it won't rebuild the auth system" is not a valid finding unless the work borders auth.

### 1.2 Context and Motivation
- Why this work is needed — the problem, not just the solution
- Who benefits and how
- What triggers this work now (incident, customer request, technical debt threshold)
- **Red flag**: A plan that jumps straight to "how" without establishing "why"
- **Guardrail**: If the plan's purpose is self-evident from its title and the problem is well-known (e.g., a bug fix for a reported issue, a feature with a linked ticket), a brief one-line motivation is sufficient. Do not demand a multi-paragraph "why" when the "why" is obvious.

### 1.3 Alternatives Considered
- At least 2-3 alternatives with trade-off analysis for significant decisions
- Why the chosen approach was selected over alternatives
- **Why it matters**: Without alternatives, reviewers can't assess whether the author explored the solution space
- **Scale gate**: Required when the choice is hard to reverse once implemented and at least two credible options exist (e.g., REST vs. GraphQL, monolith vs. microservice, build vs. buy). Not required for conventional choices where the project already has an established pattern (e.g., "we used PostgreSQL because the project uses PostgreSQL").

### 1.4 Cross-Cutting Concerns
First determine which of these concerns are relevant to this specific plan. Report only the relevant ones — listing irrelevant concerns as "N/A" is noise, not thoroughness. For each relevant concern, state whether it is addressed, partially addressed, or missing.

| Concern | When relevant |
|---------|--------------|
| Security | Any user-facing feature, data handling, auth changes, API endpoints |
| Error handling | Any feature that can fail (most of them) |
| Testing strategy | Any plan with more than 1 slice |
| Observability | Production services, data pipelines. "We'll add logging" alone is insufficient — look for specific metrics, alerting thresholds, or dashboard/runbook updates |
| Migration/rollback | Database changes, breaking API changes, infrastructure (rollback specifics covered in 5.2) |
| Performance | High-traffic paths, data-intensive operations |
| Backwards compatibility | Existing APIs, shared libraries, data formats |
| Accessibility | User-facing web/mobile features |
| Data lifecycle | Plans that create, store, or process user/customer data — retention, classification, compliance |
| Regulatory/compliance | Systems handling PII, financial data, health records, or operating in regulated industries |
| Internationalization | User-facing apps with multi-language audiences |

Don't flag missing concerns that genuinely don't apply. A CLI internal tool doesn't need internationalization.

### 1.5 Document Structure
- Is the plan navigable? Can a developer unfamiliar with it find what they need?
- Is it self-contained or does it rely on unlinked external context?
- For large plans involving multi-system interaction: is there a diagram or prose description showing how the new work fits into the broader system?
- **Red flag**: Wall-of-text plans with no headers, or plans that are all headers with no substance

## 2. Technical Soundness

**Scale gate**: For small plans, focus on 2.1 and 2.2 only.

The plan's technical decisions should be logically consistent, grounded in the actual codebase, and free of contradictions.

### 2.1 Logical Consistency
- Do the proposed solutions actually solve the stated problems?
- Do design decisions contradict each other? (e.g., claiming "loose coupling" while designing tightly coupled components)
- Are the stated benefits actually achievable with the proposed approach?
- Do the risk mitigations actually address the identified risks?
- Are decisions presented without trade-off discussion? Every significant choice involves trade-offs — a plan that lists only benefits is hiding something.

### 2.2 Codebase Alignment
Verify against the actual code when possible:
- Do referenced file paths exist?
- Do referenced patterns, functions, or modules actually work the way the plan claims?
- Does the plan follow the project's existing architectural conventions (module structure, layering, error handling strategy)?
- Are claimed integration points real?
- **Red flag**: A plan that could apply to any project — it probably wasn't grounded in actual code analysis
- **Hallucination guardrail**: Distinguish between files that should exist today (and don't) versus files the plan intends to create. Only flag missing paths for files the plan claims already exist. When uncertain, mark confidence as "low."
- **Scope reminder**: This section is about structural alignment (does the plan fit the project's architecture, module boundaries, and conventions?), NOT code-level correctness (missing attributes, wrong function signatures, type errors). The latter is resolved during implementation.

### 2.3 Architecture Decisions
For each significant technical decision in the plan:
- Is the rationale documented (context, decision, consequences)?
- Are trade-offs acknowledged, not just benefits?
- Is the confidence level appropriate?
- Is the decision reversible? If not, is extra care warranted? (Rollback specifics covered in 5.2)
- If the plan proposes replacing an existing approach, does it acknowledge the prior decision and explain why circumstances changed?
- **Y-Statement check**: "In the context of [X], facing [Y], we decided [Z] to achieve [Q], accepting [D]" — can you fill in all five parts? Example: "In the context of **10k orders/sec at peak**, facing **PostgreSQL write bottleneck**, we decided **to add a Redis write-ahead queue** to achieve **sub-100ms p99 write latency**, accepting **eventual consistency with a 2-second window**."

### 2.4 Anti-Pattern Detection

**Scale gate**: For small plans, skip this section. For medium+ plans, evaluate only the patterns relevant to the plan's domain.

**Over-engineering**
- Abstractions built for imagined future needs (YAGNI violation)
- Generic frameworks where a specific solution would suffice
- **Signal**: Plan mentions "extensibility" or "future-proofing" frequently without concrete requirements driving it

**Under-engineering**
- No consideration of concurrent access when it's clearly possible
- **Signal**: Plan focuses only on the happy path

**Distributed System Fallacies** (only when the plan involves network communication between services, external API calls, or distributed data)
- Assumes network is reliable (no retry/timeout strategy)
- Assumes latency is zero (synchronous calls in critical paths)

**Big Bang Delivery**
- Everything ships at once with no incremental path
- No way to test or validate until all slices are complete
- **Signal**: Plan has one milestone: "done"

This list is not exhaustive. Flag any anti-pattern you can ground in specific plan text with a concrete consequence, even if not listed above.

**Hallucination guardrail**: Only flag an anti-pattern if you can quote specific plan text that exhibits it and explain concretely why it causes harm in this context. "This feels over-engineered" is not a finding. "The plan introduces an AbstractProcessorFactory with three levels of indirection to handle a single known use case (X)" is a finding.

## 3. Best Practices Verification

**Scale gate**: Optional for small plans. For medium+ plans, research only the top 2-3 consequential decisions.

### 3.1 What to Research
A decision is consequential if it is (a) hard to reverse once implemented, (b) affects multiple components, or (c) involves a technology the team hasn't used before. Focus on the single most consequential decision. If the plan uses well-established patterns and you can evaluate against known best practices directly, skip web research — state your assessment with confidence level instead.

Key areas:
- The chosen architecture pattern (is it appropriate for this scale?)
- Specific technology choices (are there known pitfalls?)
- The migration/deployment strategy (does it follow established patterns?)

### 3.2 How to Compare
For each researched decision, structure your analysis:
1. **What the plan proposes**
2. **What best practice recommends** (with source)
3. **Alignment**: Do they match, partially diverge, or contradict?
4. **If divergent**: Does the plan document a rationale for the divergence?
5. **Verdict**: Acceptable divergence (with rationale) vs. potential gap (no rationale)

Plans that deviate knowingly (with rationale) are different from plans that deviate unknowingly. Only flag unknowing divergence.

### 3.3 Source Quality Rules

When performing web research for best practices:

**Trusted sources** (cite freely):
- Official documentation (language, framework, cloud provider docs)
- Engineering blogs from established companies (Google, Meta, Microsoft, Stripe, Shopify, Netflix, Uber, Cloudflare)
- Well-known practitioners (Martin Fowler, Kent Beck, Charity Majors, Kelsey Hightower, etc.)
- Research institutions (CMU SEI, IEEE, ACM)
- Well-maintained GitHub repos (1000+ stars, active maintenance)
- Thoughtworks Technology Radar
- Conference talks from major conferences (QCon, Strange Loop, KubeCon)
- Stack Overflow answers with high votes AND explanations of reasoning

**Skip entirely**:
- Reddit posts (any subreddit)
- Anonymous or uncredentialed blog posts
- SEO listicles ("Top 10 Ways to...")
- AI-generated content farms
- Quora answers
- Content older than 5 years unless it's a foundational reference (e.g., Fowler's Refactoring, Nygard's ADRs)

When citing a best practice, always note the source so the user can verify. If a trusted source is inaccessible (e.g., paywalled), note the citation at low confidence and seek an alternative from the trusted list.

### 3.4 Technical Debt Assessment
If the plan knowingly takes shortcuts, classify using Fowler's Technical Debt Quadrant:

| | Deliberate | Inadvertent |
|---|---|---|
| **Prudent** | "We'll use polling for v1 and migrate to webhooks in Q3 — tracked in JIRA-1234" | "Now we know how we should have done it" |
| **Reckless** | "We'll skip input validation to hit the deadline" | Plan proposes storing sessions in local memory for a horizontally-scaled service |

Prudent-deliberate debt with a payback plan is acceptable. Reckless debt of either kind is a finding.

## 4. Executability & Parallelization

**Scale gate**: Skip entirely for plans with fewer than 3 tasks or slices.

Can this plan actually be implemented efficiently? Is it structured for a team (or agent team) to execute?

### 4.1 Task Granularity
- Are tasks small enough to be completed independently but large enough to be meaningful?
- Does each task have clear acceptance criteria ("done when...")?
- Can a developer pick up any non-blocked task and start working without extensive context from other tasks?
- **Too granular**: "Add import statement to line 5" — creates synchronization overhead
- **Too coarse**: "Implement the backend" — impossible to estimate, review, or parallelize
- **Guardrail**: Do not prescribe a specific task size. Flag granularity only at the extremes — tasks too large to estimate or review (multiple person-weeks with no decomposition), or too small to justify coordination overhead (single-line changes as separate work items). The space between is team preference, not a defect.

### 4.2 Vertical Slicing
- Does each slice deliver end-to-end functionality (not just one layer)?
- Can each slice be tested independently?
- Do slices leave the system in a working state?

Bad: "Slice 1: Create all DB tables. Slice 2: Build all API endpoints. Slice 3: Build all UI."
Good: "Slice 1: User registration (DB + API + UI). Slice 2: User login (DB + API + UI)."

### 4.3 Dependency Accuracy & Chain Analysis
- Are inter-task dependencies correct? (Does Slice 3 actually need Slice 2 to be complete?)
- Are there hidden dependencies the plan doesn't acknowledge?
- Are external dependencies (libraries, services, APIs) available and suitable?
- Are there circular dependencies between slices?
- Can the longest chain of sequential tasks be identified from the plan?
- Are there tasks serialized unnecessarily that could run in parallel?
- **Red flag**: All tasks in a linear sequence when some could clearly run concurrently
- **Hallucination guardrail**: Only flag a missing or incorrect dependency if you can name both tasks and the specific shared artifact or state that creates the dependency. Do not invent plausible-sounding dependencies based on general knowledge.

### 4.4 Integration Planning
- When parallel work streams converge, is the integration approach planned?
- Are there explicit sync points or checkpoints?
- **Why it matters**: Teams that parallelize but don't plan integration spend disproportionate time merging

### 4.5 Feasibility Signals
- Are unknowns or risky assumptions validated with a spike before dependent work begins?
- Does the plan include go/no-go gates for high-risk items?
- Are estimates grounded in something (prior work, benchmarks, team velocity)?
- Does the plan account for realistic overhead (code review, testing, deployment)?

## 5. Risk Awareness

**Scale gate**: For small plans, a single "what could go wrong?" check suffices.

### 5.1 Risk Identification
- Are the real risks called out? (Not imagined risks, not obvious-but-harmless risks)
- Failure modes: what happens when components fail? Is there graceful degradation?
- Data risks: could the plan cause data loss, corruption, or inconsistency?
- Security risks: does the plan introduce new attack surfaces?
- Operational risks: could the plan cause outages during deployment?

### 5.2 Risk Mitigation
- Does each identified risk have a mitigation strategy?
- Are mitigations realistic and proportional to the risk?
- Are there rollback procedures for the riskiest changes?
- Are there monitoring/alerting additions to detect problems early?

### 5.3 Unknown Unknowns
- Has the plan author acknowledged areas of uncertainty?
- Are there time-boxed spikes for high-uncertainty items?
- Are there explicit fallback plans if key assumptions prove wrong?
- **Hallucination guardrail**: Do not invent unknown unknowns. Only flag this if the plan explicitly claims comprehensive risk coverage while omitting an obvious category (e.g., a data migration plan that identifies code risks but never mentions data integrity). If you cannot point to a specific omission, report "no findings."
