# Improvement Recipes — Before/After Transformations

Concrete edit patterns for the most common rule violations. When drafting fixes in Phase 4, use these as templates.

## Contents

- Description rewrites (third person, specificity, triggers)
- Frontmatter fixes (naming, YAML, reserved words)
- Shrinking oversized SKILL.md bodies (progressive disclosure)
- Flattening nested references
- Replacing vague instructions with specific criteria
- De-railroading prescriptive steps
- Killing voodoo constants
- Fixing "punt to Claude" scripts
- Path portability

---

## Recipe 1: Third-person description

**Before:**
```yaml
description: I can help you review Python code and find security issues. You can use this to audit your modules.
```

**After:**
```yaml
description: Reviews Python code for security and performance issues including SQL injection, XSS, CSRF, hardcoded secrets, N+1 queries, and blocking calls in async code. Use when the user asks to "review code", "audit this module", "check for bugs", or wants feedback on Python code quality. Do NOT use for formatting or style questions.
```

Why: The description is injected into the system prompt. First/second person confuses the discovery heuristic. Third person, specific, includes trigger phrases, includes a negative scope.

---

## Recipe 2: Vague description → specific with triggers and scope

**Before:**
```yaml
description: Helps with documents
```

**After:**
```yaml
description: Extracts text and tables from PDF files, fills forms, and merges documents. Use when working with PDF files, when the user mentions "PDF", "forms", "extraction", or asks to pull data out of documents. Do NOT use for Word documents (use docx-processing) or for general text files.
```

Why: Vague descriptions never trigger. Specific verbs + file formats + trigger phrases + explicit scope dramatically improve selection accuracy.

---

## Recipe 3: Frontmatter casing / reserved word

**Before:**
```yaml
---
name: Claude_Helper
description: Does stuff.
---
```

**After:**
```yaml
---
name: code-helper
description: [rewritten per Recipe 2]
---
```

And rename the directory to match: `mv Claude_Helper code-helper`.

Why: `name` must be lowercase-kebab-case, must match the directory, and must not contain "claude" or "anthropic" (reserved).

---

## Recipe 4: Shrinking SKILL.md past 500 lines

When a body exceeds 500 lines, move the deepest-but-still-sometimes-needed content to `references/`.

**Before (SKILL.md, 620 lines):**
```markdown
## Process
[core workflow — 200 lines]

## Advanced patterns
[400 lines of rarely-needed detail]

## Edge cases
[20 lines]
```

**After (SKILL.md, ~240 lines):**
```markdown
## Process
[core workflow — 200 lines]

## Edge cases
[20 lines]

## Advanced patterns

For the full set of advanced patterns (batching, pipelining, custom adapters), see [references/advanced-patterns.md](references/advanced-patterns.md). Consult it when the user asks for performance tuning or integration with non-default backends.
```

And create `references/advanced-patterns.md` containing the 400 lines, with a table of contents at the top since it's >100 lines.

Why: SKILL.md body loads into context every time the skill triggers. References load only when Claude follows the link. Keeping SKILL.md ≤500 lines is an official guideline.

---

## Recipe 5: Flattening nested references

**Before:**
```
SKILL.md → references/advanced.md → references/details.md → references/internals.md
```

In SKILL.md:
```markdown
See [references/advanced.md](references/advanced.md) for advanced usage.
```

In advanced.md:
```markdown
For internals, see [details.md](details.md).
```

Problem: Claude may partial-read files via `head` when they're referenced from other referenced files. Deep chains produce incomplete information.

**After (one level deep from SKILL.md):**
```markdown
## References

- [references/basic-usage.md](references/basic-usage.md) — getting started
- [references/advanced-patterns.md](references/advanced-patterns.md) — batching, pipelining
- [references/internals.md](references/internals.md) — architecture and memory model
- [references/troubleshooting.md](references/troubleshooting.md) — common failures
```

Every reference file is linked directly from SKILL.md. No reference links to another reference.

---

## Recipe 6: "Do it well" → specific criteria

**Before:**
```markdown
Review the code for security issues.
```

**After:**
```markdown
Scan the code for:
1. SQL queries without parameterization (look for string concatenation inside `execute()` or `query()` calls)
2. `innerHTML`, `document.write`, or template interpolation receiving unsanitized user input
3. Missing CSRF tokens on state-changing HTTP handlers
4. Hardcoded secrets: API keys, passwords, tokens matching the patterns `sk-*`, `ghp_*`, `AKIA*`, private keys in PEM format
5. Overly permissive CORS: `Access-Control-Allow-Origin: *` combined with credentials
```

Why: If an instruction could be replaced with "do it well" without losing meaning, it is not specific enough. Concrete criteria turn the skill from a reminder into a useful tool.

---

## Recipe 7: De-railroading creative tasks

**Before (high-freedom task, over-constrained):**
```markdown
## Writing the summary

1. Start with the word "Summary:"
2. Write exactly three sentences
3. Begin sentence one with "This document"
4. End with "In conclusion, ..."
```

**After:**
```markdown
## Writing the summary

Produce a concise summary that answers: what is the document about, what is its most important claim, and what should the reader do next. Three to five sentences. Prefer concrete nouns and active verbs. Do not open with filler like "This document discusses..." — lead with the substance.
```

Why: For high-freedom tasks (summarizing, reviewing, explaining), goals + constraints outperform rigid scripts. Save low-freedom instructions for fragile operations like migrations.

---

## Recipe 8: Adding a low-freedom script for a fragile operation

**Before (fragile task, under-constrained):**
```markdown
## Database migration

Run the migration carefully. Make sure to back up first.
```

**After:**
```markdown
## Database migration

Run exactly this command. Do not modify the flags:

```bash
python scripts/migrate.py --verify --backup --dry-run-first
```

If `--dry-run-first` reports any `ALTER` on a table larger than 1M rows, stop and ask the user before proceeding. The script will not continue automatically in that case.
```

Why: Migrations are fragile. This is a narrow bridge with cliffs on both sides — specific guardrails beat "use judgment."

---

## Recipe 9: Killing voodoo constants

**Before (in a bundled script):**
```python
TIMEOUT = 47
MAX_RETRIES = 5
BATCH_SIZE = 128
```

**After:**
```python
# Upstream API documents a 45-second worst-case response time for complex queries;
# add a 2-second buffer to avoid spurious timeouts on the boundary.
TIMEOUT = 47

# Retries: most transient failures (DNS, 503) resolve within the first two retries.
# Beyond 5, failures are almost always persistent and retrying wastes time.
MAX_RETRIES = 5

# Batch size: balance between API quota (200/request max) and memory footprint
# when processing records client-side. 128 leaves headroom for both.
BATCH_SIZE = 128
```

Why: Ousterhout's law. If you don't know why the value was chosen, neither will Claude when it has to tune the script later.

---

## Recipe 10: Scripts that solve vs. scripts that punt

**Before (punts):**
```python
def process_file(path):
    return open(path).read()
```

**After (solves):**
```python
def process_file(path):
    """Read a file, creating an empty one at `path` if missing."""
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        print(f"File {path} not found, creating with empty content", file=sys.stderr)
        with open(path, "w") as f:
            f.write("")
        return ""
    except PermissionError as e:
        print(f"Cannot read {path}: {e}. Returning empty content.", file=sys.stderr)
        return ""
```

Why: A script that raises uncaught exceptions forces Claude to re-plan around a half-failed state. A script that handles its error cases explicitly keeps the workflow deterministic.

---

## Recipe 11: Consistent terminology

**Before:**
```markdown
Locate the form field. The box will appear at the given coordinates. Each element has a type. Elements of type "signature" need special handling. The field coordinates...
```

**After (pick one term and use it throughout):**
```markdown
Locate the form field. The field will appear at the given coordinates. Each field has a type. Fields of type "signature" need special handling. The field coordinates...
```

Why: Claude is robust to synonyms, but drift multiplies ambiguity. One term per concept reduces misunderstanding.

---

## Recipe 12: Windows paths → forward slashes

**Before:**
```markdown
Run `scripts\analyze.py` and check `references\guide.md`.
```

**After:**
```markdown
Run `scripts/analyze.py` and check `references/guide.md`.
```

Why: Forward slashes work on Windows, macOS, and Linux. Backslashes fail on Unix and are interpreted as escape characters in many contexts.

---

## Recipe 13: Too many options → one default, alternatives for edge cases

**Before:**
```markdown
To extract text, you can use pypdf, pdfplumber, PyMuPDF, pdf2image, pdftotext, or write a custom parser.
```

**After:**
```markdown
Use `pdfplumber` for text extraction:

```python
import pdfplumber
with pdfplumber.open("file.pdf") as pdf:
    text = pdf.pages[0].extract_text()
```

For scanned PDFs that need OCR, use `pdf2image` + `pytesseract` instead.
```

Why: Choice paralysis. Give a clear default and name the specific alternative for the edge case that justifies it.

---

## Recipe 14: Adding a concrete output example

**Before:**
```markdown
Report findings grouped by severity with fix suggestions.
```

**After:**
```markdown
Report findings grouped by severity, in this exact format:

```
# Security findings: auth/login.py

## Critical (1)
- Line 42: SQL injection via string concatenation in `find_user_by_email`
  Fix: Use parameterized query. `cursor.execute("SELECT ... WHERE email = ?", (email,))`

## High (0)

## Medium (2)
- Line 88: Password comparison is not constant-time; use `hmac.compare_digest`
- Line 105: Session token stored in localStorage (vulnerable to XSS); use httpOnly cookie
```

If no findings in a severity, still include the header with `(0)` so the report is complete.
```

Why: Few-shot anchoring with one concrete example beats paragraphs of format description.

---

## Recipe 15: Adding anti-hallucination guardrails to analytical skills

**Before:**
```markdown
Find bugs in the code and report them.
```

**After:**
```markdown
Find bugs in the code. A bug is a defect with a specific failing input or state and a concrete observable consequence — not a style preference.

Every finding must include: (1) file:line, (2) the input or state that triggers it, (3) the observable consequence, (4) a confidence qualifier: *definitely*, *probably*, or *might be*.

**"No bugs found" is a valid and preferred outcome.** Do not manufacture findings to appear thorough. If a section of code is clean, say so briefly and move on.
```

Why: LLMs will invent findings rather than report a clean result. Explicit permission to return empty + confidence qualifiers + concrete evidence requirements together defuse hallucination.
