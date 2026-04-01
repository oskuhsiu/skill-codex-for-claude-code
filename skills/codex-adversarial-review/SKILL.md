---
name: codex-adversarial-review
description: >-
  When the user wants a hostile, skeptical review that challenges design decisions,
  assumptions, and failure modes — beyond what codex-review covers.
allowed-tools: Bash(codex:*)
---

# Codex Adversarial Review

Skeptical review that challenges assumptions, design choices, and failure modes.
Always read-only. For shared patterns, see `skills/codex/references/common.md`.

## CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If not installed: `npm install -g @openai/codex && codex login`.

## Build the adversarial prompt

### Step 1: Collect diff

```bash
git diff HEAD --stat 2>/dev/null
git diff HEAD 2>/dev/null
```

For branch comparison: `git diff <branch>...HEAD` (committed only — warn user if they
have uncommitted work). Truncate if >500 lines. Check for untracked files:
```bash
git ls-files --others --exclude-standard 2>/dev/null
```

### Step 2: Construct prompt

Load template from `references/adversarial-prompt.md`. Interpolate:
- `{{DIFF}}` — git diff output
- `{{FOCUS}}` — user's focus text or "general"
- `{{REPO_CONTEXT}}` — from README first line or directory name

Users steer the review by providing focus text (e.g., "focus on auth and data loss").

### Step 3: Execute

Always `--sandbox read-only`. Try structured output first:

```bash
codex exec --sandbox read-only --output-schema references/review-output.schema.json \
  -o /tmp/codex_adversarial_review.json <<'PROMPT' 2>/dev/null
<interpolated prompt>
PROMPT
```

If `--output-schema` unsupported, embed the JSON schema in the prompt and request JSON
output. Parse the result; if JSON parsing fails, extract from code fences or present as
plain text.

### Step 4: Present results

1. **Verdict** — `approve`, `needs-attention`, or `insufficient-context`
2. **Summary** — overall assessment
3. **Findings** by severity, each with: title, file, line range, confidence, recommendation
4. **Next steps** — actionable items
5. **Claude's verification** — read source files before challenging any finding

## Review focus areas

Auth & access control, data integrity, race conditions, rollback safety, schema drift,
observability gaps, dependency risk. Skips: naming, formatting, style, speculative concerns.

## Rules

- Never auto-apply fixes or modify code
- Verify source files before disagreeing with a finding
- Low-confidence findings are "worth investigating", not "must fix"
- `insufficient-context` is not `approve`

## Error handling

See `common.md` for standard errors. Additional:

| Error | Action |
|-------|--------|
| No diff available | Suggest `--base <branch>` or a commit range |
| Empty findings | Valid outcome — report "no adversarial concerns found" |
