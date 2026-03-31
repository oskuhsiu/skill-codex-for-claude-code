---
name: codex-review
description: >-
  Use when the user wants a Codex-powered code review on uncommitted changes, a branch diff,
  or a specific commit.
allowed-tools: Bash(codex:*)
---

# Codex Code Review

One-command code review via `codex review` — Codex CLI's native review subcommand. Always read-only, always safe.

## Codex CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If Codex is not installed, stop and tell the user to run `npm install -g @openai/codex && codex login`.

## Determine review scope

`codex review` supports three scope flags. Choose based on user intent:

| User intent | Flag | Command |
|-------------|------|---------|
| No specific target (default) | `--uncommitted` | `codex review --uncommitted` |
| "review against main" / a branch | `--base <branch>` | `codex review --base <branch>` |
| "review commit abc123" | `--commit <sha>` | `codex review --commit abc123` |

**Default to `--uncommitted`** which covers staged, unstaged, and untracked changes.

When the user says "review against main" or similar, resolve the actual base branch name if ambiguous:
```bash
git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}'
```
Use the resolved name. If resolution fails (no remote, detached HEAD), ask the user to specify.

## Add focus instructions

`codex review` accepts an optional `[PROMPT]` argument for custom review instructions. Use it to specify focus:

| Focus | Prompt |
|-------|--------|
| General (default) | (omit prompt — Codex's default review is general) |
| Security | `"Focus on security: injection vulnerabilities, auth issues, data exposure, OWASP top 10."` |
| Performance | `"Focus on performance: unnecessary allocations, N+1 queries, missing indexes, hot paths."` |
| API compatibility | `"Focus on API compatibility: breaking changes, backward compatibility, contract violations."` |
| Custom focus | Pass the user's focus text as the prompt argument |

**Always use heredoc** when passing custom review instructions to prevent shell injection:

```bash
codex review --uncommitted <<'PROMPT' 2>/dev/null
Focus on security: injection vulnerabilities, auth issues, data exposure.
PROMPT
```

## Build and execute the command

### Examples

**Review uncommitted changes (most common — default scope):**
```bash
codex review --uncommitted 2>/dev/null
```

**Review uncommitted changes with security focus:**
```bash
codex review --uncommitted <<'PROMPT' 2>/dev/null
Focus on security: injection vulnerabilities, auth issues, data exposure, OWASP top 10.
PROMPT
```

**Review against a branch:**
```bash
codex review --base main 2>/dev/null
```

**Review against a branch with custom focus:**
```bash
codex review --base develop <<'PROMPT' 2>/dev/null
Focus on breaking changes and test coverage.
PROMPT
```

**Review a specific commit:**
```bash
codex review --commit abc1234 2>/dev/null
```

**With optional title for context:**
```bash
codex review --uncommitted --title "Add user authentication" 2>/dev/null
```

## Present results

After receiving Codex's review output:

1. **Summarize findings** grouped by severity (critical > warning > info)
2. **Highlight agreements** if Claude has also reviewed the same code
3. **Flag disagreements** — if Claude disagrees with a Codex finding, first read the relevant source files to verify, then state the disagreement with evidence
4. **Do NOT auto-apply any fixes** — present findings only, let the user decide next steps

## Dual-review pattern

For maximum coverage, suggest running both Claude and Codex reviews:

1. Claude reviews directly (using Read/Grep tools on the changed files)
2. Codex reviews via this skill
3. Compare findings, highlight where they agree and disagree
4. User decides on disputed items

Mention this option when the user runs a review, but do not force it.

## Session management

`codex review` runs non-interactively and does not create a resumable session. If the user wants to discuss specific findings in depth, use the general `codex` skill to start a new interactive session:

```bash
codex exec --sandbox read-only <<'PROMPT' 2>/dev/null
I just received a code review with this finding: <finding details>.
Explain this issue in more detail and suggest a fix.
PROMPT
```

## Error handling

| Error | Action |
|-------|--------|
| `codex: command not found` | Tell user to install: `npm install -g @openai/codex` |
| Non-zero exit | Rerun without `2>/dev/null`, report stderr, ask for direction |
| Auth error / 401 | Suggest `codex login` |
| No changes to review (`--uncommitted`) | Inform user there are no uncommitted changes; suggest `--base <branch>` or `--commit <sha>` |
| Not a git repository | Inform user; `codex review` requires a git repository |
| Branch not found (`--base`) | Check branch name; suggest listing branches with `git branch -a` |
