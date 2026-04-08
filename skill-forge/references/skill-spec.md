# Claude Code Skills - Technical Specification

Authoritative reference for building skills. Consult when constructing any skill file.

## 1. File Structure

### Required
```
skill-name/
└── SKILL.md           # Exact casing. Not SKILL.MD, skill.md, Skill.md.
```

### Full structure
```
skill-name/
├── SKILL.md             # Required - main instructions
├── scripts/             # Optional - executable code (NOT loaded into context)
├── references/          # Optional - docs Claude reads on demand
└── assets/              # Optional - templates, schemas, static resources
```

### Naming
- Folder: **kebab-case only**. Valid: `code-reviewer`. Invalid: `Code_Reviewer`, `codeReviewer`.
- No README.md inside the skill folder.
- No "claude" or "anthropic" in skill name (reserved).

### Directory conventions
- `references/`: Markdown files Claude reads on demand. Name descriptively (e.g., `api-guide.md`, `security-checklist.md`).
- `scripts/`: Executable code invoked via Bash. Include shebangs. These are NOT loaded into context -- Claude must run them explicitly.
- `assets/`: Templates, schemas, and static resources. Any format. Referenced from SKILL.md or scripts.

## 2. YAML Frontmatter

Between `---` markers at top of SKILL.md. No XML angle brackets (`<` `>`) anywhere in frontmatter.

### Recommended fields

| Field | Rules | Limit |
|-------|-------|-------|
| `name` | kebab-case, must match folder name | 64 chars |
| `description` | WHAT + WHEN (trigger phrases) + SCOPE (negative triggers) | 1024 chars |

Claude Code treats both as optional -- `name` defaults to the folder name, `description` falls back to the first paragraph of markdown. However, always set both explicitly for reliable triggering and clarity.

### Optional standard fields

| Field | Description |
|-------|-------------|
| `license` | MIT, Apache-2.0, etc. |
| `compatibility` | Environment requirements (max 500 chars) |
| `metadata` | Key-value map: author, version, mcp-server, etc. |
| `allowed-tools` | Space-delimited tool list: `"Read Write Bash(python:*) WebFetch"` |

### Claude Code extended fields

| Field | Description |
|-------|-------------|
| `argument-hint` | Autocomplete hint: `[issue-number]`, `[filename]` |
| `disable-model-invocation` | `true` = description hidden from Claude, only user can invoke via `/`. Use for skills with side effects. |
| `user-invocable` | `false` = hidden from `/` menu. Background knowledge only. |
| `model` | Force model: `haiku`, `sonnet`, `opus` |
| `context` | `fork` = run in isolated subagent |
| `agent` | Subagent type with `context: fork`: `Explore`, `Plan`, `general-purpose` |
| `hooks` | Lifecycle hooks scoped to this skill (same format as global hooks in settings.json -- runs shell commands on skill events) |

**`context: fork` requires a task.** If the skill contains only guidelines (e.g., "use these conventions") without an actionable prompt, the subagent receives the guidelines but has nothing to do and returns empty. Only use `context: fork` for skills that perform concrete work.

### Invocation control

| Settings | User invokes via `/`? | Description in Claude's context? | Claude auto-triggers? |
|----------|----------------------|--------------------------------|---------------------|
| (defaults) | Yes | Yes | Yes |
| `disable-model-invocation: true` | Yes | No (invisible to Claude) | No |
| `user-invocable: false` | No | Yes | Yes |

## 3. String Substitutions

| Variable | Description |
|----------|-------------|
| `$ARGUMENTS` | All args passed when invoking |
| `$ARGUMENTS[0]`, `$0` | First argument |
| `$ARGUMENTS[1]`, `$1` | Second argument |
| `${CLAUDE_SESSION_ID}` | Current session ID |
| `${CLAUDE_SKILL_DIR}` | Skill directory path -- use for referencing bundled scripts |

## 4. Dynamic Context Injection

`` !`command` `` runs a shell command before the skill loads. Output replaces the expression.

```markdown
## Context
- Branch: !`git branch --show-current`
- Status: !`git status --short`
- Recent: !`git log --oneline -5`
```

## 5. Extended Thinking

Include the word `ultrathink` anywhere in a skill's content to enable extended thinking (deeper reasoning) when the skill is active. Place it in a comment or instruction where it reads naturally.

## 6. Progressive Disclosure

| Level | What | When loaded | Budget |
|-------|------|-------------|--------|
| 1. Metadata | name + description | Always (system prompt) | ~100 tokens per skill. Total budget: 2% of context window |
| 2. Instructions | SKILL.md body | When skill is activated | Keep under 500 lines |
| 3. Resources | references/, scripts/, assets/ | On demand, when referenced | No fixed limit |

Reference resources from SKILL.md so Claude knows they exist:
```markdown
Consult `references/api-guide.md` for rate limiting details.
```

## 7. Description Engineering

Formula: `[What it does] + [When to use it] + [What it does NOT do]`

Claude tends to undertrigger skills -- it won't use them when it should. Combat this by making descriptions slightly "pushy": lean into the conditions where the skill should activate rather than being conservative.

```yaml
# Good: specific, pushy triggers, negative scope
description: Code review for Python focusing on security and performance. Use when user asks to "review code", "check for bugs", "audit this module", or wants feedback on Python code quality, even if they don't explicitly ask for a "review". Do NOT use for syntax questions or formatting.

# Bad: vague, no triggers
description: Helps with projects.
```

## 8. Body Best Practices

- Put critical instructions at the top
- Be specific: `"Scan for SQL injection, XSS, CSRF"` not `"Check for security issues"`
- Use conversational tone -- explain the *why*, not just the *what*
- For critical validations, bundle a script in `scripts/` -- code is deterministic, language isn't
- Reference bundled resources explicitly so Claude can find them

### Turning Research into Instructions

Research findings must be transformed into specific, actionable instructions:

| Research finding | Bad instruction | Good instruction |
|---|---|---|
| "Headlines matter" | "Write a good headline" | "Write 5 variants using: [Number]+[Benefit]+[Timeframe], How to [X] Without [Pain Point], The [Adj] Guide to [Topic] That [Outcome]. Pick the one with the strongest hook and most specific promise." |
| "Check for security" | "Review security" | "Scan for: 1) SQL without parameterization, 2) innerHTML with unsanitized data, 3) Missing CSRF tokens, 4) Secrets in code, 5) Overly permissive CORS" |
| "Tell a story" | "Make it a story" | "Use SCR: Situation (current state), Complication (the tension), Resolution (your solution). Every slide answers: why should the audience care RIGHT NOW?" |

If an instruction could be replaced with "do it well" without losing meaning, it is not specific enough.

## 9. Complete Example

A minimal, correct skill:

````markdown
---
name: code-reviewer
description: Reviews Python code for security and performance. Use when user asks to "review code", "check for bugs", or wants feedback on code quality. Do NOT use for formatting or syntax questions.
allowed-tools: "Read Glob Grep"
---

# Code Reviewer

You review Python code with a focus on security vulnerabilities and performance bottlenecks.

## Process

1. Read the target file(s)
2. Scan for security issues: SQL injection, XSS, CSRF, hardcoded secrets, overly permissive CORS
3. Check for performance issues: N+1 queries, unnecessary allocations, blocking calls in async code
4. Report findings grouped by severity (critical → minor), with fix suggestions

Consult `references/security-checklist.md` for the full OWASP-based checklist.
````
