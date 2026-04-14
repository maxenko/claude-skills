# claude-skills

A curated collection of **14 production-grade [Claude Code](https://claude.com/claude-code) skills** for planning, code review, refactoring, Rust workflows, and authoring skills themselves.

Each skill is a self-contained Markdown bundle that Claude Code auto-triggers when its description matches what you're doing — or that you can invoke explicitly with `/<skill-name>`.

---

## What's a skill?

A skill is a folder containing a `SKILL.md` (plus optional `references/` and `scripts/`) that teaches Claude Code a specialized workflow. The frontmatter `description` is a trigger: when your request smells like the skill's domain, Claude loads it and follows its process.

```
skill-name/
├── SKILL.md              # frontmatter + main workflow (<500 lines)
└── references/           # optional deep-dive docs, loaded on demand
    └── checklist.md
```

---

## The skills at a glance

| Skill | What it does | Trigger phrases |
|---|---|---|
| **plan-feature** | Drafts an implementation plan using parallel research agents | *"plan a feature"*, *"how should I implement…"* |
| **plan-eval** | Evaluates a plan for soundness, completeness, executability | *"review this plan"*, *"is this plan good"* |
| **plan-execute** | Implements a `.md` plan file with per-step verification | *"execute this plan"*, *"implement the plan"* |
| **code-scrutiny** | Deep review of recent changes for bugs, dupes, smells | *"review my changes"*, *"scrutinize this"* |
| **refactor-pro** | Structural refactor for readability and maintainability | *"refactor"*, *"clean up this code"* |
| **bug-hunt** | Hunts real defects through rotating expert lenses | *"find bugs"*, *"what could go wrong"* |
| **app-harden** | Audits for runtime stability, resource leaks, DoS vectors | *"harden this"*, *"pre-flight before shipping"* |
| **rust-ci-green** | Gets Rust projects to a green CI badge (fmt + clippy) | *"make CI green"*, *"fix the build"* |
| **rust-commit** | Verifies Rust code is CI-clean, then commits | *"rust commit"*, *"clean commit"* |
| **rust-warn-fix** | Fixes or silences every warning using expert judgment | *"fix warnings"*, *"zero warnings"* |
| **rust-uplift** | Finds places where idiomatic Rust reduces verbosity | *"uplift"*, *"make idiomatic"* |
| **rust-test-separate** | Splits inline `#[cfg(test)]` blocks into their own files | *"separate tests"*, *"extract tests"* |
| **skill-forge** | Researches a domain, then builds an expert-grade skill | *"forge a skill"* |
| **skill-refine** | Audits existing skills against Anthropic's best practices | *"improve this skill"*, *"refine this skill"* |

---

## Installation

Skills live in one of two places Claude Code scans on startup:

- `~/.claude/skills/` — available in every project (recommended)
- `.claude/skills/` — project-local, committed alongside your code

Clone and symlink (or copy) the skills you want:

```bash
# macOS / Linux
git clone https://github.com/<your-fork>/claude-skills.git
mkdir -p ~/.claude/skills
ln -s "$PWD/claude-skills/bug-hunt"      ~/.claude/skills/bug-hunt
ln -s "$PWD/claude-skills/rust-ci-green" ~/.claude/skills/rust-ci-green
# …and so on
```

```powershell
# Windows (PowerShell as admin — or use copy instead of symlink)
git clone https://github.com/<your-fork>/claude-skills.git
New-Item -ItemType Directory -Force "$HOME\.claude\skills" | Out-Null
New-Item -ItemType SymbolicLink -Path "$HOME\.claude\skills\bug-hunt" -Target "$PWD\claude-skills\bug-hunt"
```

Restart Claude Code (or run `/skills` to list) and the skill will appear.

---

## Quick tour by category

### Planning trio — think, critique, do

```
plan-feature   →   plan-eval   →   plan-execute
  (design)         (stress-test)       (build)
```

**Example session:**

```
You: plan a feature for adding OAuth2 login to our Express app
    → plan-feature runs parallel research agents, produces plan.md

You: review this plan
    → plan-eval stress-tests it for gaps and risky assumptions

You: execute this plan
    → plan-execute walks the plan, verifying each step before moving on
```

### Code quality — read, refactor, hunt, harden

Good for pre-commit reviews or before shipping a risky change.

```
You: scrutinize the last 3 commits
You: find bugs in src/payment/
You: refactor this — it's a mess
You: harden the /webhook endpoint for production
```

`bug-hunt` looks for **real** defects (concrete failing input + observable consequence), not style nits.
`app-harden` finds runaway logs, missing timeouts, SSRF, secret leaks, crash loops, and similar runtime issues.

### Rust toolkit — green CI every time

A pipeline for keeping Rust projects pristine:

```
rust-warn-fix     # clear compiler + clippy warnings
rust-uplift       # suggest idiomatic abstractions
rust-test-separate  # extract inline tests to their own files
rust-ci-green     # fmt + clippy until GitHub Actions passes
rust-commit       # verify CI-clean, then commit
```

**Typical flow after a feature branch:**

```
You: fix warnings
You: uplift
You: rust commit
```

### Meta-skills — skills about skills

```
You: forge a skill for auditing Terraform plans
    → skill-forge researches the domain, drafts SKILL.md, writes it to ~/.claude/skills/

You: improve my code-scrutiny skill
    → skill-refine audits it against Anthropic's best practices + fresh domain research
```

---

## How triggering works

Claude reads each skill's `description` and picks the best match for your request. Descriptions in this repo follow a deliberate pattern:

> *What it does. When to use it (trigger phrases). What it does NOT cover (negative triggers).*

You can always force a specific skill with `/skill-name` — e.g. `/bug-hunt src/auth.ts`.

---

## Authoring your own

The two meta-skills in this repo are the fastest path:

- **Start from scratch** → `forge a skill for <domain>` (uses `skill-forge`)
- **Polish an existing skill** → `refine this skill: <name>` (uses `skill-refine`)

Both encode the rules straight from Anthropic's skill authoring docs: third-person descriptions, SKILL.md under 500 lines, references one level deep, no voodoo constants, positive framing over negative constraints.

---

## License

MIT — see [LICENSE](LICENSE). Copyright (c) 2026 Max Enko.

Contributions welcome. If you add a skill, run `skill-refine` on it first.
