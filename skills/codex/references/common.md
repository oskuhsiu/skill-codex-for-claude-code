# Codex CLI — Common Patterns

Shared patterns across all Codex skills. Each skill references this file instead of
duplicating these sections.

## CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If not installed: `npm install -g @openai/codex && codex login`.

## Permission tiers

| Tier | Sandbox | Rule |
|------|---------|------|
| 1 — Safe | `--sandbox read-only` | Proceed without asking |
| 2 — Standard | `--full-auto` | Inform user, proceed unless objected |
| 3 — Dangerous | `--sandbox danger-full-access` | Require explicit "yes" before proceeding |

Never use `--dangerously-bypass-approvals-and-sandbox`. Recommend `danger-full-access` instead. Only use `--yolo` if the user insists after understanding the risk.

## Stderr handling

Codex emits thinking tokens to stderr that bloat context.

| Mode | Stderr | On failure |
|------|--------|------------|
| Read-only | `2>/dev/null` | Rerun without `2>/dev/null` to see real error |
| Write-mode | `2>/tmp/codex_<skill>_stderr.log` | Read the log, do NOT rerun |
| Debug | No redirection | User requested verbose output |

## Pre-flight worktree check (write-mode only)

```bash
git status --porcelain 2>/dev/null
```

**Clean:** Proceed — all post-run changes are Codex's.

**Dirty:** Warn user. Recommend stash:
```bash
git stash push -m "pre-codex-<skill>" 2>/dev/null
```
If user proceeds anyway, save baseline:
```bash
git diff HEAD 2>/dev/null > /tmp/codex_<skill>_baseline.diff
git diff --cached 2>/dev/null >> /tmp/codex_<skill>_baseline.diff
```

## Post-run review gate (write-mode only)

After any write-mode run completes: **no further mutations** (no commits, no edits, no
additional Codex runs) until the user reviews and decides.

1. Show changes: `git diff --stat && git diff` and `git ls-files --others --exclude-standard`
2. If worktree was dirty, compare with baseline to isolate Codex-only changes
3. Present summary to user, ask: keep / review in detail / revert

Selective revert:
```bash
git checkout -- path/to/file    # one file
git checkout -- .               # everything
git stash pop                   # restore stashed changes
```

## Session management

After every run: "The Codex session can be resumed."

Resume:
```bash
codex exec resume --last - <<'PROMPT' 2>/dev/null
<follow-up>
PROMPT
```

Do not add flags when resuming unless user requests overrides.

## Error handling

| Error | Action |
|-------|--------|
| `codex: command not found` | Install: `npm install -g @openai/codex` |
| Non-zero exit (read-only) | Rerun without `2>/dev/null`, report stderr |
| Non-zero exit (write-mode) | Check `git diff` + stderr log, present state. Do NOT rerun |
| Auth error / 401 | `codex login` |
| Not a git repository | Read-only: proceed. Write-mode: refuse, suggest `git init` |
| Timeout (>120s) | Inform user; do not kill |
