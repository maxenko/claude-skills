# Anthropic Official Skill Best Practices — Audit Checklist

Distilled from the official Anthropic documentation at `https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices` and the Claude Code skills reference at `https://code.claude.com/docs/en/skills`. When auditing a skill, walk through every item. Pass, Improve, Fail, or N/A.

## 1. Frontmatter rules (hard constraints)

These are validation-level requirements. A violation is always a Fail.

- [ ] Frontmatter is enclosed between `---` markers at the very top of SKILL.md
- [ ] YAML parses cleanly (no tabs, no unmatched quotes, no unescaped colons)
- [ ] `name` field present and matches the parent directory name exactly
- [ ] `name` uses lowercase letters, numbers, and hyphens only (≤64 chars)
- [ ] `name` does NOT contain the reserved words `anthropic` or `claude`
- [ ] `description` field present and non-empty (≤1024 chars)
- [ ] Neither `name` nor `description` contains `<` or `>` characters (breaks frontmatter parsing)
- [ ] No XML tags inside any frontmatter value

## 2. Description quality

The description is the single most important field — Claude uses it to decide whether to load the skill at all.

- [ ] **Written in third person.** "Processes Excel files..." not "I can help..." or "You can use this..."
- [ ] **States WHAT the skill does** in specific, concrete terms
- [ ] **States WHEN to use it** with explicit trigger phrases the user might naturally say
- [ ] **States SCOPE limits** — at least one negative trigger ("Do NOT use for...") to prevent false activation
- [ ] **Front-loads the key use case** in the first ~250 characters (truncation point in skill listings)
- [ ] **Includes domain-specific keywords** a user would actually type (e.g., "PDF", "forms", "extraction")
- [ ] Not vague. Fails: "helps with documents", "processes data", "does stuff with files"
- [ ] Slightly "pushy" — lean into activating when relevant rather than being conservative. Claude tends to undertrigger skills.

## 3. Body structure and size

- [ ] SKILL.md body is **under 500 lines** (hard guideline from official docs)
- [ ] Critical instructions are at the top, not buried mid-document
- [ ] Section headings are descriptive, not generic (`## Process` is OK; `## Stuff` is not)
- [ ] Workflow steps are numbered when order matters
- [ ] For multi-step workflows >5 steps: includes a checklist Claude can copy and tick off

## 4. Conciseness — "Claude is already smart"

Every token in SKILL.md competes with conversation history. Challenge each piece of information.

- [ ] No explanations of fundamentals Claude already knows (what a PDF is, what HTTP means, what a function is)
- [ ] No generic advice that applies to all programming ("write clean code", "handle errors")
- [ ] No restating of tool descriptions Claude already has
- [ ] Instructions focus on pushing Claude *out of its defaults* — the things a non-expert would miss
- [ ] No padding or throat-clearing — every paragraph justifies its token cost

## 5. Degrees of freedom

Match the specificity of instructions to how fragile and variable the task is.

- [ ] **High-freedom tasks** (many valid approaches) use text-based guidance, not rigid steps
- [ ] **Medium-freedom tasks** (a preferred pattern exists) use templates or pseudocode
- [ ] **Low-freedom tasks** (fragile, must be exact) use specific scripts or exact commands
- [ ] The skill does not over-constrain high-freedom tasks (don't railroad Claude on creative/judgment work)
- [ ] The skill does not under-constrain low-freedom tasks (don't say "use judgment" when a specific sequence is required)

## 6. Specificity — the "could be replaced with 'do it well'" test

If an instruction could be swapped for "do it well" without losing meaning, it is not specific enough.

- [ ] Instructions include **concrete criteria, thresholds, or examples** — not just "make it good"
- [ ] For analytical skills: findings must point to a specific location and concrete consequence
- [ ] For output-producing skills: at least one concrete example of the desired output format
- [ ] For decision points: a decision framework (not "use judgment")

## 7. Progressive disclosure and references

- [ ] Content loaded only when needed lives in `references/`, not inline in SKILL.md
- [ ] **References are one level deep from SKILL.md.** SKILL.md → file.md. NOT SKILL.md → file.md → other.md.
- [ ] Every reference file is explicitly linked from SKILL.md with a description of its contents and when to load it
- [ ] Reference files longer than 100 lines begin with a table of contents
- [ ] File names are descriptive (`security-checklist.md`, not `doc2.md`)
- [ ] Directories are organized by domain or feature (`reference/finance.md`, not `docs/file1.md`)

## 8. Consistency and terminology

- [ ] One term per concept throughout the skill. Not "field" in one place and "box" or "element" elsewhere for the same thing.
- [ ] Not mixing "endpoint" / "URL" / "route" / "path" for the same concept
- [ ] Not mixing "extract" / "pull" / "get" / "retrieve" for the same action

## 9. No time-sensitive or decaying content

- [ ] No date-keyed advice like "Before August 2025, use X" that will rot
- [ ] Legacy content lives in a clearly marked "Old patterns" section, ideally collapsed with `<details>`
- [ ] No references to specific model versions or API versions that will change

## 10. Path and platform portability

- [ ] All file paths use **forward slashes** (`scripts/helper.py`), even on Windows. Backslashes fail on Unix.
- [ ] No hardcoded absolute paths where a relative path or `${CLAUDE_SKILL_DIR}` would work
- [ ] No shell syntax that silently breaks across bash/zsh/PowerShell unless `shell:` is pinned in frontmatter

## 11. Default-with-escape-hatch — no option overload

- [ ] Does not present 4+ alternative tools/libraries without a clear default. Pick one, mention alternatives only for specific edge cases.
- [ ] Recommended default is current (not deprecated), supported, and widely available

## 12. Scripts (only if the skill bundles executable scripts)

- [ ] Scripts **solve problems explicitly** rather than punting to Claude with uncaught exceptions
- [ ] Error handling produces actionable messages ("Field 'X' not found. Available: ...") not generic failures
- [ ] No **voodoo constants** — every magic number is justified in a comment (Ousterhout's law)
- [ ] Required packages are listed in SKILL.md with install commands, not assumed
- [ ] Scripts use `${CLAUDE_SKILL_DIR}` or relative paths, not hardcoded absolute paths
- [ ] SKILL.md makes clear whether Claude should **execute** vs **read** each script
- [ ] For destructive or high-stakes operations: a plan-validate-execute pattern with an intermediate verifiable artifact

## 13. Invocation control and visibility

- [ ] Skills with side effects (deploy, commit, send, delete) set `disable-model-invocation: true`
- [ ] Background-knowledge skills (reference material, not actionable commands) set `user-invocable: false`
- [ ] `context: fork` is ONLY used when the skill contains an explicit, actionable task — not for pure guidelines (would return empty)
- [ ] If `allowed-tools` is set, it only includes tools the skill actually uses
- [ ] MCP tools are referenced by fully qualified name (`ServerName:tool_name`), not bare name

## 14. Structural hygiene

- [ ] Directory name is kebab-case
- [ ] Filename is exactly `SKILL.md` (not `SKILL.MD`, `skill.md`, `Skill.md`)
- [ ] No `README.md` inside the skill folder (reserved; causes conflicts)
- [ ] `scripts/`, `references/`, `assets/` are the standard subdirectory names — avoid inventing new ones

## 15. Anti-patterns specifically called out in Anthropic guidance

- [ ] Does not "state the obvious" — focuses on pushing Claude out of default behavior
- [ ] Does not "railroad" Claude with prescriptive steps when goals and constraints would suffice
- [ ] Does not offer too many options without a default
- [ ] Does not include Windows-style paths
- [ ] Does not assume packages are installed without stating the dependency
- [ ] Scripts do not "punt to Claude" — they handle their own error conditions

## 16. Tone

- [ ] Conversational, explains the *why* behind instructions when non-obvious
- [ ] Not a wall of ALL CAPS "ALWAYS" / "NEVER" — reasoning over shouting
- [ ] Not a compliance document tone — reads like expert mentorship

## Source

All items above are derived directly from:
- `https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices`
- `https://code.claude.com/docs/en/skills`
- `https://claude.com/blog/equipping-agents-for-the-real-world-with-agent-skills`

If Anthropic publishes updated guidance, re-fetch those URLs and update this file.
