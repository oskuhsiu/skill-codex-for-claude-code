---
name: codex-review
description: >-
  When the user wants Codex to review code — uncommitted changes, a branch diff, or a
  specific commit.
allowed-tools: Bash(codex:*)
---

# Codex Code Review

Code review via `codex review`. Always read-only. For shared patterns, see
`skills/codex/references/common.md`.

## CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If not installed: `npm install -g @openai/codex && codex login`.

## Determine scope

| User intent | Command |
|-------------|---------|
| Default (no specific target) | `codex review --uncommitted` |
| "review against main" | `codex review --base <branch>` |
| "review commit abc123" | `codex review --commit abc123` |

Default to `--uncommitted`. Resolve ambiguous branch names:
```bash
git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'
```

## Focus instructions

Pass optional focus as a heredoc prompt:

```bash
codex review --uncommitted - <<'PROMPT' 2>/dev/null
Focus on security: injection vulnerabilities, auth issues, data exposure.
PROMPT
```

Common focuses: security, performance, API compatibility, or the user's custom text.
Omit the prompt for a general review.

## Present results

1. Summarize findings by severity (critical > warning > info)
2. If Claude also reviewed the same code, highlight agreements and disagreements
3. Verify before disagreeing — read source files first
4. Do not auto-apply fixes — present findings only

## Dual-review option

Suggest running both Claude and Codex reviews for maximum coverage. Mention it, don't
force it.

## Session note

`codex review` is non-interactive — no resumable session. For deeper discussion of
findings, start a new session via the general `codex` skill.

## Error handling

See `common.md` for standard errors. Additional:

| Error | Action |
|-------|--------|
| No changes to review | Suggest `--base <branch>` or `--commit <sha>` |
| Branch not found | Suggest `git branch -a` |
