---
name: codex-rescue
description: >-
  When the user is stuck and wants Codex to debug, investigate, or provide a second AI
  perspective on a problem.
allowed-tools: Bash(codex:*)
---

# Codex Rescue

Delegate debugging or investigation to Codex when Claude is stuck or a second perspective
helps. For shared patterns (permissions, stderr, sessions, errors), see
`skills/codex/references/common.md`.

## CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If not installed: `npm install -g @openai/codex && codex login`.

## Choose sandbox level

- "investigate", "analyze", "explain", "find out why" → `read-only` (Tier 1)
- "fix", "refactor", "implement", "handle" → `--full-auto` (Tier 2)
- Unclear → default to `read-only`, escalate if needed

For write-mode: run the pre-flight worktree check from `common.md`.

## Build the rescue prompt

Gather context before calling Codex:
- Exact error message or symptom
- What Claude already tried and why it failed
- Relevant files
- Reproduction steps

```bash
codex exec --sandbox <level> <<'PROMPT' 2>/dev/null
Task: <what needs to be done>

Context:
- <error or symptom>
- <what was tried>
- <relevant files>

Goal: <desired outcome>
PROMPT
```

Write-mode: use `2>/tmp/codex_rescue_stderr.log` instead of `2>/dev/null`.

### User flag mapping

| User says | Codex flag |
|-----------|-----------|
| "continue" / `--resume` | `resume --last` |
| `--model <model>` | `-m <model>` |
| `--effort <level>` | `--config model_reasoning_effort="<level>"` |
| `--read-only` | `--sandbox read-only` |
| `--fresh` | Start new session |

## Post-run review gate (write-mode)

Follow the post-run review gate from `common.md`. Present results as:

1. **What Codex found** — diagnosis summary
2. **What Codex changed** — files modified with descriptions
3. **Claude's assessment** — independent evaluation, flag agreements and disagreements

Ask user what to keep or revert. Compare Claude and Codex perspectives where relevant.

## Non-git environments

Read-only: allowed. Write-mode: refused — no rollback without git.
