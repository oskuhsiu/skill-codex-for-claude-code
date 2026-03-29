---
name: codex
description: >-
  Delegate tasks to Codex CLI. Use when user says "run codex", "ask codex", "codex review",
  "codex cloud", "let codex handle it", wants a second opinion from another AI, or asks to
  use GPT/OpenAI for a code task.
allowed-tools: Bash(codex:*)
---

# Codex CLI Skill

Delegate code tasks to OpenAI's Codex CLI and manage the results.

## Codex CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If Codex is not installed, stop and tell the user to run `npm install -g @openai/codex && codex login`.

## Running a task

### 1. Determine sandbox level from task intent

| Task type | Sandbox | Flags |
|-----------|---------|-------|
| Analysis, review, reading code | `read-only` | `--sandbox read-only` |
| Apply edits, refactoring, file creation | `workspace-write` | `--sandbox workspace-write --full-auto` |
| Network access, system-wide changes | `danger-full-access` | `--sandbox danger-full-access --full-auto` |

Default to `read-only`. Escalate only when the task clearly requires writes.

### 2. Build the command

Use the user's configured defaults — do NOT hardcode model names or reasoning effort levels. Omit `-m` and `--config model_reasoning_effort` unless the user explicitly requests an override.

**Basic read-only task:**
```bash
codex exec --sandbox read-only "analyze the authentication flow" 2>/dev/null
```

**Task requiring edits:**
```bash
codex exec --sandbox workspace-write --full-auto "refactor the login module to use async/await" 2>/dev/null
```

**With user-specified overrides:**
```bash
codex exec -m o4-mini --config model_reasoning_effort="high" --sandbox read-only "explain this codebase" 2>/dev/null
```

### 3. Handle the working directory

Run Codex in the current working directory by default. Use `-C <dir>` only when the user specifies a different directory. Validate the directory exists before passing it.

Do NOT use `--skip-git-repo-check` by default. If Codex fails because the directory is not a git repo, inform the user and ask whether to proceed with `--skip-git-repo-check`.

### 4. Execute and report

Run the command. See [Stderr handling strategy](#stderr-handling-strategy) for how to manage stderr.

**On success (exit 0):** summarize the output for the user. Mention that the session can be resumed with a follow-up request.

**On failure (non-zero exit):** rerun the same command WITHOUT `2>/dev/null` to capture the actual error. Report the error and ask the user for direction. Do not retry blindly.

## Permission escalation

Three tiers of safety. Never skip a tier.

### Tier 1: Safe (no permission needed)
- `--sandbox read-only`
- Default approval policy
- `--ephemeral` (no persistence)

### Tier 2: Standard (inform user, proceed unless objected)
- `--sandbox workspace-write` with `--full-auto`
- `-C <dir>` to a different directory
- `--add-dir <path>` for additional directory access
- Resuming a previous session

Before proceeding, state: "Codex will run with write access to the workspace. Proceeding unless you object."

### Tier 3: Dangerous (MUST ask user and get explicit confirmation)
- `--sandbox danger-full-access`
- `--dangerously-bypass-approvals-and-sandbox` / `--yolo`
- `--skip-git-repo-check`
- `-a never` (never ask for approval)

Before using any Tier 3 flag, ask the user for explicit confirmation and explain the specific risk. Example: "This grants Codex full system access including network. Codex could read/write any file and make network requests. Do you want to proceed?"

Wait for a clear "yes" before continuing. Do not proceed on ambiguous responses.

**Absolute rule:** Never use `--dangerously-bypass-approvals-and-sandbox`. Even when the user requests it, recommend `--sandbox danger-full-access` instead, which provides full access while preserving Codex's own approval prompts. Only use `--yolo` if the user insists after understanding the difference.

## Session management

### Resume a session

Use heredoc for multiline safety:
```bash
codex exec resume --last <<'PROMPT' 2>/dev/null
Your follow-up prompt here.
Can span multiple lines safely.
PROMPT
```

For simple single-line prompts:
```bash
echo "follow-up prompt" | codex exec resume --last 2>/dev/null
```

When resuming, do not add flags (`-m`, `--sandbox`, etc.) unless the user explicitly requests them — the resumed session inherits the original configuration.

Track what was done in each session so follow-up context is accurate.

### After every Codex run

Inform the user: "The Codex session can be resumed. Ask to continue or provide a follow-up instruction."

## Stderr handling strategy

Codex emits thinking tokens to stderr, which bloat Claude's context window.

| Scenario | Approach |
|----------|----------|
| Normal run | `2>/dev/null` — suppress stderr |
| Non-zero exit | Rerun without `2>/dev/null` to see the real error |
| User requests debug | Omit `2>/dev/null` entirely |

This is better than blindly suppressing all stderr because real errors are never lost — they surface on failure.

## Structured output

When programmatic output is needed:

- **JSONL event stream:** `codex exec --json --sandbox read-only "task" 2>/dev/null`
- **Save final message to file:** `codex exec -o /tmp/codex_result.txt --sandbox read-only "task" 2>/dev/null`
- **Validated output shape:** `codex exec --output-schema schema.json --sandbox read-only "task" 2>/dev/null`

## Advanced features

For detailed flag documentation, consult `references/flags-reference.md` in the skill directory.

| Feature | Flag | Notes |
|---------|------|-------|
| Code review | See `references/review-mode.md` | Review uncommitted changes, branches, commits |
| Cloud tasks | See `references/cloud-tasks.md` | Submit and manage cloud-based Codex tasks |
| Image attachment | `-i <path>` | Attach screenshots or diagrams to the prompt |
| Additional directories | `--add-dir <path>` | Grant Codex access to directories outside workspace |
| Config profiles | `-p <profile>` | Load named profiles from config.toml |
| Feature flags | `--enable <flag>` / `--disable <flag>` | Toggle experimental Codex features |
| Ephemeral mode | `--ephemeral` | No disk persistence for the session |
| Local/OSS models | `--oss` | Use local Ollama models instead of OpenAI |

## Critical evaluation of Codex output

Codex is a peer, not an authority. Its models have their own knowledge cutoffs and blind spots.

**When confident Codex is wrong:**
1. State the disagreement clearly with evidence
2. Optionally resume the session to discuss, identifying as Claude:
   ```bash
   cat <<'PROMPT' | codex exec resume --last 2>/dev/null
   This is Claude following up. I disagree with your suggestion
   about X because [evidence]. What's your reasoning?
   PROMPT
   ```
3. Let the user decide when there is genuine ambiguity

**Do not defer blindly.** Evaluate Codex suggestions critically, especially regarding recent API changes, library versions, and evolving best practices that may post-date Codex's training data.

## Error handling

| Error | Action |
|-------|--------|
| `codex: command not found` | Tell user to install: `npm install -g @openai/codex` |
| Non-zero exit | Rerun without `2>/dev/null`, report stderr, ask for direction |
| Auth error / 401 | Suggest `codex login` |
| "not a git repository" | Ask user whether to use `--skip-git-repo-check` (Tier 3) |
| Timeout (>120s no output) | Inform user the task is long-running; do not kill it |
| Partial output with warnings | Summarize warnings, ask how to adjust |
