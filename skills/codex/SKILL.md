---
name: codex
description: >-
  When the user says "run codex", "ask codex", "let codex handle it", or wants to use
  GPT/OpenAI for a general code task. Not for code writing (codex-code), reviews
  (codex-review), or debugging (codex-rescue).
allowed-tools: Bash(codex:*)
---

# Codex CLI Skill

General-purpose Codex CLI delegation. For shared patterns (permissions, stderr, sessions,
errors), see `references/common.md`.

## CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If not installed: `npm install -g @openai/codex && codex login`.

## Running a task

### 1. Choose sandbox level

| Task type | Sandbox |
|-----------|---------|
| Analysis, reading code | `read-only` (default) |
| Edits, refactoring, file creation | `workspace-write --full-auto` |
| Network access, system-wide changes | `danger-full-access` |

Default to `read-only`. Escalate only when the task clearly requires writes.
See `references/common.md` for permission tier rules.

### 2. Build and run

Do not hardcode model names. Omit `-m` and `--config model_reasoning_effort` unless the
user requests an override.

```bash
codex exec --sandbox read-only "analyze the authentication flow" 2>/dev/null
```

```bash
codex exec --sandbox workspace-write --full-auto "refactor the login module" 2>/dev/null
```

Run in cwd by default. Use `-C <dir>` only when user specifies a different directory.

**On success:** Summarize output. Mention session can be resumed.
**On failure:** Rerun without `2>/dev/null` to capture the error. Report and ask for direction.

## Structured output

- **JSONL:** `codex exec --json --sandbox read-only "task" 2>/dev/null`
- **Save to file:** `codex exec -o /tmp/result.txt --sandbox read-only "task" 2>/dev/null`
- **Schema validation:** `codex exec --output-schema schema.json --sandbox read-only "task" 2>/dev/null`

## Advanced features

See `references/flags-reference.md` for full flag documentation.

| Feature | Flag |
|---------|------|
| Code review | See `references/review-mode.md` |
| Cloud tasks | See `references/cloud-tasks.md` |
| Image attachment | `-i <path>` |
| Additional directories | `--add-dir <path>` |
| Config profiles | `-p <profile>` |
| Feature flags | `--enable <flag>` / `--disable <flag>` |
| Ephemeral mode | `--ephemeral` |
| Local/OSS models | `--oss` |

## Critical evaluation

Codex is a peer, not an authority. When confident Codex is wrong, state the disagreement
with evidence. Optionally resume the session to discuss. Let the user decide on genuine
ambiguity.
