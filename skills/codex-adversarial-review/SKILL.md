---
name: codex-adversarial-review
description: >-
  Use when the user wants a skeptical, adversarial review that challenges design decisions,
  assumptions, and failure modes — beyond standard code review.
allowed-tools: Bash(codex:*)
---

# Codex Adversarial Review

A skeptical, adversarial review that goes beyond standard code review. Codex acts as a hostile
reviewer challenging every assumption, design choice, and failure mode.

## Codex CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If Codex is not installed, stop and tell the user to run `npm install -g @openai/codex && codex login`.

## How it differs from standard review

| Standard review (`codex review`) | Adversarial review |
|----------------------------------|-------------------|
| Finds bugs and style issues | Challenges design decisions and assumptions |
| Accepts stated intent at face value | Questions whether the intent is correct |
| Reports what's wrong | Asks "what happens when this fails?" |
| Focuses on code quality | Focuses on expensive, dangerous failures |
| Helpful tone | Skeptical tone — default to doubt |

## Build the adversarial prompt

### Step 1: Collect git context

First check the diff size:
```bash
git diff HEAD --stat 2>/dev/null
```

Then collect the diff. Use the **same scope** for both `--stat` and the actual diff:

**Default (uncommitted changes — staged + unstaged):**
```bash
git diff HEAD 2>/dev/null
```

**Branch comparison (`--base <branch>` — skill-level shorthand):**
```bash
git diff <branch>...HEAD 2>/dev/null
```

**Important scope note for `--base` mode:** `git diff <branch>...HEAD` compares committed history only — it excludes uncommitted local changes. If the user has both committed and uncommitted work, warn them that only the committed portion is reviewed. To include everything, suggest they commit or stash first.

**Truncation:** If the diff exceeds 500 lines, truncate it. Inform Codex and the user that truncation occurred and list the omitted files:
```bash
git diff HEAD --stat 2>/dev/null  # show full file list
git diff HEAD 2>/dev/null | head -500  # truncated diff for prompt
```

**Untracked files:** `git diff HEAD` excludes untracked files. Always check for them:
```bash
git ls-files --others --exclude-standard 2>/dev/null
```
If untracked files exist, inform the user they are not covered by the adversarial review.

### Step 2: Construct the prompt

Load the adversarial prompt template from `references/adversarial-prompt.md` in this skill's directory.

Interpolate the following into the template:
- `{{DIFF}}` — the git diff output from Step 1
- `{{FOCUS}}` — the user's optional focus text, or "general" if none provided
- `{{REPO_CONTEXT}}` — brief repo description (from README first line or directory name)

When interpolating, check that the diff content does not contain the heredoc delimiter string. If it does, use a different delimiter (e.g., `ADVERSARIAL_REVIEW_EOF` or a UUID-based delimiter).

### Step 3: Execute with structured output

**Always use `--sandbox read-only`.** Adversarial reviews must never modify code.

```bash
codex exec --sandbox read-only --output-schema /path/to/skills/codex-adversarial-review/references/review-output.schema.json -o /tmp/codex_adversarial_review.json <<'PROMPT' 2>/dev/null
<paste the interpolated adversarial prompt here>
PROMPT
```

If `--output-schema` is not supported by the installed Codex version, fall back to the prompt-only approach below.

### Fallback (no --output-schema support)

```bash
codex exec --sandbox read-only -o /tmp/codex_adversarial_review.txt <<'PROMPT' 2>/dev/null
<paste the interpolated adversarial prompt here>

Respond ONLY with valid JSON matching this exact structure:
{
  "verdict": "needs-attention",
  "summary": "One paragraph overall assessment of the changes.",
  "findings": [
    {
      "severity": "high",
      "title": "Example title",
      "file": "path/to/file",
      "line_range": "42-58",
      "confidence": 0.85,
      "description": "What is wrong and why.",
      "recommendation": "Concrete fix."
    }
  ],
  "next_steps": ["Actionable item"]
}

Valid values for verdict: "approve", "needs-attention", "insufficient-context"
Valid values for severity: "critical", "high", "medium", "low"
PROMPT
```

**Fallback validation:** After reading the output file, attempt to parse it as JSON. If parsing fails:
1. Check if the output contains a JSON block within markdown code fences — extract it
2. If extraction succeeds, use the extracted JSON
3. If extraction fails, treat the entire output as plain-text findings and present them without structured formatting
4. Inform the user that structured output was unavailable and findings are presented as-is

### Step 4: Parse and present results

Read the output file and present findings:

1. **Show verdict** — `approve`, `needs-attention`, or `insufficient-context`
   - If `insufficient-context`: explain what context was missing and suggest how to get a more complete review (e.g., smaller scope, provide more files)
2. **Show summary** — the overall assessment paragraph
3. **List findings** ordered by severity (critical > high > medium > low), each with:
   - Severity badge
   - Title and description
   - File and line range
   - Confidence score (0.0-1.0)
   - Concrete recommendation
4. **Show next steps** — actionable items the user should consider
5. **Claude's verification** — for any finding Claude wants to challenge, **first read the actual source files** to verify independently. Only then state the disagreement with specific evidence from the code.

## What the adversarial review focuses on

From the prompt template, the review targets:

- **Auth & access control** — privilege escalation, token handling, session management
- **Data integrity** — data loss paths, partial writes, constraint violations
- **Race conditions** — TOCTOU, concurrent access, lock ordering
- **Rollback safety** — can this change be safely reverted?
- **Schema drift** — migration compatibility, backward/forward compatibility
- **Observability gaps** — silent failures, missing logs, unmonitored error paths
- **Dependency risk** — version pinning, supply chain, deprecation exposure

It explicitly **skips**: naming conventions, formatting, style preferences, speculative concerns without evidence.

## Steering the review

Users can provide optional focus text to steer the adversarial review:

```
/codex-adversarial-review focus on auth and data loss scenarios
```

```
/codex-adversarial-review what happens if the database goes down mid-transaction?
```

The focus text is injected into the `{{FOCUS}}` slot of the prompt template.

## Critical rules

1. **NEVER auto-apply fixes.** Present findings only. Ask the user which issues to address.
2. **NEVER modify code.** This skill is strictly read-only.
3. **Verify before disagreeing.** Read the actual source files before challenging a Codex finding.
4. **Respect confidence scores.** Low-confidence findings should be flagged as "worth investigating" not "must fix."
5. **Distinguish verdicts.** `insufficient-context` is NOT the same as `approve` — never treat an incomplete review as a clean pass.

## Error handling

| Error | Action |
|-------|--------|
| `codex: command not found` | Tell user to install: `npm install -g @openai/codex` |
| Non-zero exit | Rerun without `2>/dev/null`, report stderr |
| Auth error / 401 | Suggest `codex login` |
| Not a git repository | Inform user; this skill requires a git repo for diff context |
| No diff available | Inform user; suggest specifying `--base <branch>` or a commit range |
| Diff is empty (scope mismatch) | Verify the chosen scope has actual changes; suggest alternative scope |
| JSON parse failure | Follow the fallback validation steps above |
| Empty findings | Report "Codex found no adversarial concerns" — this is a valid outcome |
