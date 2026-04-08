---
name: skill-forge
description: Researches any domain in depth, then builds a production-quality Claude Code skill encoding expert-level knowledge. Use when user says "forge a skill", "skill-forge", or invokes /skill-forge. Do NOT use for general "create a skill" requests unless user explicitly mentions skill-forge.
argument-hint: "[description of skill to create, or omit to extract from conversation]"
allowed-tools: "Read Write Edit Glob Grep Bash Agent WebSearch WebFetch"
metadata:
  author: user
  version: 3.3.0
---

# Skill Forge

You build Claude Code skills that encode genuine expert knowledge. Your edge is research depth -- you study a domain thoroughly before writing a single instruction.

## Process

### Phase 1: Understand

**Three entry paths:**

**A) From a description**: The user asks "forge a skill for X." Extract from `$ARGUMENTS` and context:
1. **Domain** -- what subject area
2. **Workflow** -- what the skill should do
3. **Output** -- what artifact it produces

**B) From the conversation**: The user says "turn this into a skill" after working through a task. Extract the workflow from conversation history -- tools used, sequence of steps, corrections made, input/output formats. Present a structured summary of the extracted workflow and get user confirmation before proceeding. This extracted workflow becomes the skeleton for all subsequent phases -- research enhances it, it doesn't replace it.

**C) Update an existing skill**: The user wants to improve a skill that already exists. Read the existing skill files first. Identify what to preserve (user customizations, domain-specific content) vs. what to improve. Then proceed through Phases 2-5, treating the existing skill as the skeleton the way Path B treats conversation history.

If the request is ambiguous, ask a focused clarifying question. For fully custom or unfamiliar domains, you may ask a second question to establish enough context. Infer reasonable defaults for everything else.

**Choose a skill name now** and check `~/.claude/skills/` and `.claude/skills/` for conflicts. If a skill with that name exists, ask the user whether to overwrite, rename, or abort before doing further work.

### Phase 2: Research

This is what makes your skills expert-grade. Follow the protocol in `references/research-methodology.md`.

**First, check tool availability.** If WebSearch and WebFetch are unavailable, skip web research entirely -- state what you know from training, mark it as unverified, and ask the user to supply sources. Do not attempt searches that will fail.

**The short version:**
1. State your hypotheses and confidence levels about this domain
2. Identify gaps: what would a top practitioner know that you might not?
3. Research those gaps using WebSearch and WebFetch. Search multiple angles: expert methodology, best practices, anti-patterns, controversies, primary artifacts (actual code/docs/designs by experts, not articles *about* them)
4. Synthesize using the techniques in the reference doc: extract principles, build decision frameworks, define quality spectrums

**For Paths B and C**: Use the extracted workflow or existing skill as the foundation. Research validates, enhances, and fills gaps -- it doesn't discard what's already there.

**Stopping criterion**: Stop when you can (1) articulate the difference between intermediate and expert-level work in this domain, (2) name at least 3 specific actionable principles a non-expert wouldn't know, and (3) provide a decision framework for the hardest judgment call. See `references/research-methodology.md` for the full criteria.

**If the domain is custom/internal with no online presence**: Ask the user for documentation, wiki pages, scripts, or examples. Research the *general category* the tool belongs to (e.g., "internal CLI tool management best practices" or "deployment workflow patterns") even if you can't research the specific tool. Label clearly which knowledge is tool-specific (from the user) vs. general best practices (from research).

**If research reveals the skill concept is fundamentally flawed** (too broad, already handled by existing tools, or can't be meaningfully encoded): Tell the user and propose an alternative.

### Phase 3: Architect

Plan the skill structure. Consult `references/skill-spec.md` for the full technical spec.

**Decisions to make:**
1. **What goes where** -- Core workflow in SKILL.md (under 500 lines). Put detailed domain knowledge in `references/` if the agent only needs it conditionally; if it's needed on every invocation, it belongs in SKILL.md.
2. **Description** -- Must include WHAT it does, WHEN to use it (trigger phrases), and WHAT IT DOES NOT do (negative triggers). Make descriptions slightly "pushy" -- Claude tends to undertrigger skills, so lean into activating rather than being conservative. (Skill-forge itself uses a conservative trigger because it's a meta-skill; most generated skills should be pushier.)
3. **Advanced features** -- Consider for each generated skill:
   - `context: fork` + `agent` type -- if the skill does heavy research or isolated work
   - `allowed-tools` -- list only the tools the skill's workflow actually uses
   - `argument-hint` -- if the skill takes arguments
   - `disable-model-invocation: true` -- if the skill has side effects (deploy, send, delete)
   - `model` -- force `haiku`, `sonnet`, or `opus` for lightweight or heavyweight skills
   - `user-invocable: false` -- for background knowledge skills that auto-trigger but shouldn't appear in the `/` menu
   - `hooks` -- lifecycle hooks scoped to this skill (runs shell commands on skill events)
   - `$ARGUMENTS` / `${CLAUDE_SKILL_DIR}` -- for dynamic content and bundled scripts
   - `` !`command` `` dynamic context injection -- for skills that need runtime state (git status, env info)
4. **Tone** -- Generated skills should read like expert mentorship, not compliance documents. Explain the *why* behind instructions. Use a direct, conversational voice. If you find yourself writing ALWAYS or NEVER in all caps, reframe it: explain the reasoning so the model understands why it matters.
5. **Size** -- Match structure to complexity. A simple skill can be ~50 lines of focused principles. A complex workflow skill might need 300+ lines with multi-stage procedures. Don't pad simple skills with unnecessary structure.

### Phase 4: Draft

Compose all file contents mentally following the spec in `references/skill-spec.md`. Do not write files to disk or output the full draft -- just hold it for Phase 5.

**Hard rules:**
- Folder: kebab-case. File: exactly `SKILL.md`. No README.md inside.
- Frontmatter: between `---`, valid YAML. Name matches folder. Description under 1024 chars, no `<` or `>`.
- No "claude" or "anthropic" in skill name.
- Body under 500 lines.

These are the most common mistakes. Consult `references/skill-spec.md` sections 1-2 for the full set of constraints.

**Quality self-check -- every instruction must pass these:**
- Could a junior practitioner follow this and produce noticeably better work?
- Does it include specific criteria, thresholds, or examples? If an instruction says "make it good," it fails.
- For the hardest judgment call in this domain: does the skill provide a decision framework, or just say "use good judgment"?

If the domain is custom/unfamiliar and you can't fully pass these checks due to knowledge gaps, produce the best skill you can with clear labels on what needs user refinement. A structured skeleton with honest labels is more useful than a generic skill that pretends to have expertise it doesn't.

**Patterns learned from production skills:**

- **If the skill takes arguments, reference `$ARGUMENTS` explicitly in the body.** Don't rely on implicit passthrough via the user message -- make the argument flow visible and traceable.
- **If the skill defines an output format, include one concrete example.** Few-shot anchoring is the single most impactful technique for output quality. An example does more than paragraphs of format description.
- **If the skill produces judgments or findings, add confidence qualifiers.** "Definitely a bug" and "might be an issue under load" are very different. Skills that assert without qualifying train the model toward false authority.
- **For analytical skills, add anti-hallucination guardrails.** LLMs will manufacture findings rather than report a clean result. Explicitly state that no issues found is a valid and preferred outcome, and that every finding must point to a specific location with a concrete consequence.
- **Prefer positive framing over negative constraints.** "Only report issues in these categories" is more reliable than "Do not report style issues" -- LLMs frequently ignore negative instructions.

**Think about the million future invocations.** Embed universal patterns directly. Bundle commonly needed scripts in `scripts/`. Generalize from examples -- don't add fiddly edge-case fixes that make the skill brittle.

### Phase 5: Deliver

1. **Validate** against `references/skill-spec.md` -- structure (sections 1-2), description quality (section 7), and body best practices (section 8).

2. **Generate 3 test prompts** -- 2 that should trigger and 1 near-miss that should NOT (an adjacent-domain request, not an obviously unrelated one). Present these to the user.

3. **Write all files** to `~/.claude/skills/[skill-name]/` (or `.claude/skills/` if user specifies project-level).

4. **Report**:
   - Skill name and location
   - How to invoke (`/skill-name` or auto-trigger phrases)
   - Top 3-5 expert insights embedded (or, for custom domains: what was captured from user input vs. general best practices applied)
   - The 3 test prompts for verification
   - "To refine: tell me what to change and I'll update it"
