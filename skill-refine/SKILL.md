---
name: skill-refine
description: Audits and improves existing Claude skills against Anthropic's official best practices — third-person descriptions, concise SKILL.md bodies, progressive disclosure, appropriate degrees of freedom, no voodoo constants, one-level references. Use when the user says "improve this skill", "refine this skill", "audit this skill", "upgrade this skill", "review my skill", "check this skill against best practices", "make this skill better", or wants any existing SKILL.md revised. Also researches the skill's specific domain online to inject cutting-edge best practices. Do NOT use for creating new skills from scratch (use skill-forge) or for general code review of non-skill files.
allowed-tools: Read Write Edit Glob Grep Bash WebSearch WebFetch
argument-hint: [skill-name-or-path]
---

# Skill Refine

You improve existing Claude skills. Your edge is applying Anthropic's *official* skill authoring rules, not folklore — and then adding domain-specific research on top so the refined skill encodes current expert practice, not just better formatting.

The target skill is: `$ARGUMENTS`. If that is empty, ask the user which skill to refine before doing anything else.

## The two kinds of improvements you make

1. **Structural / authoring quality** — driven by Anthropic's official checklist (see `references/best-practices-checklist.md`). These are deterministic: a rule is either followed or it isn't.
2. **Domain depth** — driven by fresh online research into the skill's subject area. These are judgment calls: is the skill encoding expert-tier knowledge or generic advice?

Both matter. A skill with perfect frontmatter but generic domain content is mediocre. A skill with brilliant domain insights but a vague description will never trigger. Improve both.

## Process

### Phase 1: Locate and read the target

The argument may be a skill name (`code-scrutiny`), a path (`~/.claude/skills/code-scrutiny`), or a `SKILL.md` file directly. Resolve it:

1. If the argument contains `SKILL.md`, read that file directly.
2. Otherwise, check these locations in order and use the first match:
   - `~/.claude/skills/<name>/SKILL.md` (personal)
   - `.claude/skills/<name>/SKILL.md` (project-level, from cwd)
   - `$ARGUMENTS/SKILL.md` (if the argument is a directory)
3. If nothing matches, list available skills with `ls ~/.claude/skills/ .claude/skills/ 2>/dev/null` and ask the user to pick.

Once you find it:
- Read the full `SKILL.md`.
- Glob the skill directory for `references/**`, `scripts/**`, `assets/**`.
- Read every reference file (they are part of the skill you're auditing).
- Note scripts by name and purpose, but don't read their full bodies unless a finding depends on them.

Tell the user what you found in one sentence: "Refining `skill-name` — SKILL.md is N lines, with M reference files and K scripts."

### Phase 2: Audit against the official checklist

Work through `references/best-practices-checklist.md` systematically. For each item, classify the skill as:

- **Pass** — rule is satisfied, no change needed.
- **Improve** — rule is partially satisfied or could be stronger.
- **Fail** — rule is violated and must be fixed.
- **N/A** — rule doesn't apply (e.g., script rules for skills without scripts).

Critical rules that always produce a Fail if violated:

- Frontmatter has invalid YAML or missing required fields
- `name` contains uppercase, underscores, "claude", or "anthropic"
- `name` doesn't match the directory name
- `description` is in first or second person ("I can...", "You can use this to...")
- `description` is vague ("helps with projects", "does stuff with files")
- SKILL.md body exceeds 500 lines
- References are nested more than one level deep (SKILL.md → advanced.md → details.md)
- Windows-style backslash paths (`scripts\helper.py`)
- Frontmatter contains `<` or `>` (breaks parsing)
- README.md exists inside the skill folder (reserved; should not exist)

Produce findings with confidence qualifiers. Each finding must point to a specific location (line number, field, or file) and describe a concrete consequence. "Definitely violates X" and "might read as Y under Z conditions" are very different — label them differently.

**Anti-hallucination guardrail**: "No issues found" is a valid and preferred outcome for any section. If a skill already follows a rule, say so and move on — do not manufacture problems to appear thorough. Only flag issues that point to a specific line, field, or file and describe a concrete consequence. A clean audit means the skill is good, not that you didn't look hard enough.

### Phase 3: Research domain-specific best practices

This is where the skill goes from "correctly formatted" to "actually expert." Skip this phase only if WebSearch and WebFetch are unavailable — in that case, say so explicitly and note that domain improvements will come from training knowledge only, marked unverified.

1. **Identify the domain.** Read the skill's description and workflow. What is it actually about? `rust-ci-green` is about Rust CI hygiene. `bug-hunt` is about defect detection. Be specific — "programming" is not a domain, "Rust clippy and rustfmt in CI" is.

2. **State what the skill currently claims as expert knowledge.** List the specific principles, tools, and techniques it encodes. This is your baseline.

3. **Research for gaps and recency.** Search for:
   - `[domain] best practices [current year]`
   - `[domain] common mistakes` or `[domain] anti-patterns`
   - `[domain] expert workflow` or `[domain] advanced techniques`
   - Any tools/libraries the skill recommends — check they're still current, not deprecated
   - Anti-patterns the skill warns about — check they're still considered anti-patterns

   Prioritize official documentation and primary sources (expert practitioners, maintainer blogs, standards bodies) over listicles. A ratified standard outranks a recent blog post regardless of date.

4. **Identify what's missing or stale.** For each gap, record: what the skill currently says, what current expert practice says, and the source URL. If the skill's advice is still current, record that too — absence of change is useful information.

**Stopping criterion**: stop researching when you can either (a) name 3 concrete domain improvements with sources, or (b) confidently say the skill's domain content is already current. Don't research endlessly.

### Phase 4: Synthesize proposed improvements

Build a concrete change list. For each proposed change:

- **Where**: file path and line number or section heading
- **What**: the exact text or structural change
- **Why**: which best-practice rule or research finding drives it
- **Confidence**: *definitely*, *probably*, or *consider*. Reserve *definitely* for rule violations with no judgment involved. Use *consider* for domain additions that are expert opinion.

Group changes by category: `Frontmatter`, `Description`, `Structure`, `Body content`, `Domain depth`, `Scripts` (if applicable). Within each group, order by impact — highest first.

**Stay within reasonable bounds.** You are refining, not rewriting. Rules:

- Preserve the skill's existing purpose, scope, and voice. If the user wanted a different skill, they'd use skill-forge.
- Preserve user customizations — comments like "custom:" or obvious edits should stay untouched unless they directly violate a critical rule.
- Don't add features the skill doesn't have unless a specific gap demands it. "This skill would be cooler with a GUI" is out of scope.
- Don't bloat. If a change adds more than 30 lines, justify the token cost explicitly. The official guidance is that every token in SKILL.md must justify its presence.
- Don't rename the skill unless its current name violates a hard rule (reserved words, uppercase, wrong casing).

### Phase 5: Present and apply

Show the user the findings report first, before writing anything:

```
# Refinement report: <skill-name>

## Summary
<1-sentence overall assessment>
<N findings: X critical, Y improvements, Z considerations>

## Critical fixes (rule violations)
- [file:line] <what> — <why> (confidence: definitely)
- ...

## Recommended improvements
- [file:line] <what> — <why> (confidence: probably)
- ...

## Domain research additions
- <insight> — <source URL> (confidence: consider)
- ...

## Untouched by design
<rules the skill already passes, listed briefly so the user knows you checked>
```

Then ask: "Apply all changes, apply only critical fixes, or let me pick?" Wait for the user's response before editing.

When applying changes:
- Use `Edit` with precise old/new strings for targeted fixes.
- Use `Write` only when restructuring a whole file (e.g., splitting SKILL.md into SKILL.md + a new reference file).
- After edits, verify: count SKILL.md lines (must be ≤500), re-read the frontmatter, confirm it still parses as valid YAML, confirm `name` still matches the directory.
- If a proposed change would create a new reference file, create it under `references/` with a descriptive name. Never create a README.md inside the skill folder.

### Validation

After applying changes, run a final sanity check:

1. `wc -l <skill>/SKILL.md` — confirm ≤500 lines.
2. Read the new frontmatter. Confirm: valid YAML, name matches directory, no `<` or `>`, no reserved words, description in third person, description under 1024 chars.
3. Glob references one level deep only. Check that no reference file links to another reference file in a way that creates a chain >1 deep from SKILL.md.
4. Report to the user: "Applied N changes. Validation: pass." Or list any validation failures for a follow-up.

## Output format example

Here is the expected shape of the findings report for a typical refinement. Use this as the template:

```
# Refinement report: example-skill

## Summary
Solid skill with a strong workflow section, but the description is in second person and the body has grown past the 500-line budget. Found 2 critical fixes, 4 improvements, 3 domain research additions.

## Critical fixes
- SKILL.md:2 — description uses "You can use this to..." — rewrite in third person per Anthropic official guidance (confidence: definitely)
- SKILL.md (540 lines total) — body exceeds 500-line budget — move the "Advanced patterns" section to references/advanced-patterns.md (confidence: definitely)

## Recommended improvements
- SKILL.md:45 — "Review the code carefully" is non-specific — replace with concrete scan list (confidence: probably)
- ...

## Domain research additions
- Current version recommends `tool-v1` but `tool-v2` has been the default since 2025 — update example — https://tool.dev/changelog (confidence: consider)
- ...

## Untouched by design
- Frontmatter YAML is valid, name is kebab-case, matches directory
- Reference files are one level deep
- No Windows paths, no time-sensitive content
- Scripts handle errors explicitly
```

One concrete example beats pages of format description — follow this shape.

## Guardrails

- **Never touch a skill the user didn't ask about.** If the user says "improve code-scrutiny", don't also "fix" bug-hunt because you noticed something in passing.
- **Never rewrite from scratch.** Even if you'd have structured the skill differently, the user's existing structure has context you don't. Refine, don't replace.
- **Never invent sources.** If you cite a URL for a domain finding, you must have actually fetched or searched it in this session. No placeholder citations.
- **If research contradicts the skill**, present both views with sources and let the user decide. Don't silently "correct" expert judgment calls.
- **If the skill is already excellent**, say so and stop. Reporting "no meaningful improvements found" is a valid outcome and more useful than manufacturing busywork.

## References

- `references/best-practices-checklist.md` — the full Anthropic official checklist, distilled from [platform.claude.com/docs](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices). Consult this during Phase 2.
- `references/improvement-recipes.md` — before/after transformations for the most common rule violations. Consult this when drafting specific edits in Phase 4.
