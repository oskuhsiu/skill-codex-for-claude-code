---
name: codex-rescue
description: >-
  Use when the user is stuck or wants to delegate debugging, investigation, or implementation
  to Codex for a second AI perspective.
allowed-tools: Bash(codex:*)
---

# Codex Rescue

Delegate complex investigation, debugging, or implementation work to Codex when Claude is stuck,
needs a second opinion, or the task benefits from a parallel AI perspective.

## Codex CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If Codex is not installed, stop and tell the user to run `npm install -g @openai/codex && codex login`.

## When to use rescue

| Scenario | Example |
|----------|---------|
| Claude is stuck | "I can't figure out why this test fails" |
| Need a second opinion | "What would Codex do differently?" |
| Complex debugging | "This crash only happens in production" |
| Unfamiliar codebase area | "Investigate how the payment module works" |
| Implementation assistance | "Let Codex handle the database migration" |
| User explicitly asks | "Ask codex to fix this", "/codex-rescue" |

## Determine task type and sandbox level

| Task type | Sandbox | Flags | Tier |
|-----------|---------|-------|------|
| Investigation, reading, analysis | `read-only` | `--sandbox read-only` | 1 (safe) |
| Bug fix, refactoring, code edits | `workspace-write` | `--sandbox workspace-write --full-auto` | 2 (standard) |
| Needs network or system access | `danger-full-access` | `--sandbox danger-full-access` | 3 (dangerous) |

**Choose based on task intent:**
- If the user asks to "investigate", "analyze", "explain", or "find out why" → use `read-only`
- If the user asks to "fix", "refactor", "implement", or "handle" → use `workspace-write`
- When unclear, default to `read-only` and escalate if Codex needs write access

**Understanding `--full-auto`:** This flag combines `workspace-write` sandbox with `on-request` approvals, meaning Codex can execute shell commands and edit files automatically within the workspace. It is more permissive than `workspace-write` alone. Make sure the user understands this.

### Permission escalation

- **Tier 1 (read-only):** Proceed without confirmation.
- **Tier 2 (workspace-write + full-auto):** Require explicit consent. State: "Codex will run with write access and automatic execution within the workspace. It can edit files and run commands without asking. Do you want to proceed?" Wait for affirmative response.
- **Tier 3 (danger-full-access):** Ask for explicit confirmation and explain the risk: full system access including network. Wait for "yes."

## Build the rescue prompt

### Step 1: Gather context

Before calling Codex, collect relevant context to include in the prompt:

- **Error messages** — the exact error or failure output
- **What Claude tried** — summarize what Claude already attempted and why it didn't work
- **Relevant files** — list the key files involved
- **Reproduction steps** — how to trigger the issue

### Step 2: Construct the prompt

Write a clear, focused prompt for Codex. Structure it as:

```
Task: <what needs to be done>

Context:
- <error message or symptom>
- <what was already tried>
- <relevant files>

Goal: <specific desired outcome>
```

### Step 3: Parse user flags

| User flag | Codex flag | Notes |
|-----------|-----------|-------|
| `--resume` or "continue" | `resume --last` | Resume the previous Codex session (see resume rules below) |
| `--model <model>` | `-m <model>` | Override model (only when user specifies) |
| `--effort <level>` | `--config model_reasoning_effort="<level>"` | Set reasoning effort |
| `--read-only` | `--sandbox read-only` | Downgrade to investigation only |
| `--fresh` | (no resume flag) | Start a new session even if one exists |

## Execute the rescue

### Pre-flight: Prepare the worktree (workspace-write only)

Before running Codex with write access, the worktree must be in a known state for safe rollback.

**Option A: Require clean worktree (recommended)**

Check for uncommitted changes:
```bash
git status --porcelain 2>/dev/null
```

If the worktree is dirty (has uncommitted changes), inform the user:
"Your worktree has uncommitted changes. For safe rescue with write access, I recommend committing or stashing your changes first so Codex's edits can be cleanly identified and reverted if needed."

If the user agrees, help them stash:
```bash
git stash push -m "pre-codex-rescue" 2>/dev/null
```

**Option B: Proceed on dirty worktree (user accepts risk)**

If the user wants to proceed without cleaning, create a full baseline for attribution:
```bash
git diff HEAD 2>/dev/null > /tmp/codex_rescue_baseline.diff
git diff --cached 2>/dev/null >> /tmp/codex_rescue_baseline.diff
git status --porcelain 2>/dev/null > /tmp/codex_rescue_baseline_status.txt
```

Warn the user: "Proceeding on a dirty worktree. Selective revert of Codex changes may be unreliable if Codex modifies files you've already changed."

### New rescue task

For write-mode, persist stderr to a log file instead of discarding it:

```bash
codex exec --sandbox workspace-write --full-auto <<'PROMPT' 2>/tmp/codex_rescue_stderr.log
<rescue prompt here>
PROMPT
```

For read-only mode, stderr suppression is safe:
```bash
codex exec --sandbox read-only <<'PROMPT' 2>/dev/null
<rescue prompt here>
PROMPT
```

### Resume a previous rescue

**Do NOT use `resume --last` for write-capable rescues** without verifying the target session. `--last` picks the most recent Codex session in this workspace, which may belong to a different task.

For read-only resume:
```bash
codex exec resume --last <<'PROMPT' 2>/dev/null
<follow-up prompt here>
PROMPT
```

For write-capable resume, inform the user which session will be resumed before executing. If identity cannot be confirmed, start a fresh session instead.

### With model/effort override

```bash
codex exec --config model_reasoning_effort="high" --sandbox workspace-write --full-auto <<'PROMPT' 2>/tmp/codex_rescue_stderr.log
<rescue prompt here>
PROMPT
```

Only add `-m <model>` when the user explicitly provides a model name. Do not hardcode model IDs.

## Handle rescue results — Post-run gate

This is the most critical section. Follow these rules strictly.

### Understanding the two-phase flow

When Codex runs with `workspace-write`, it **directly modifies files** during execution. This is by design — rescue tasks often need Codex to actually make changes. However, Claude's job is to **review what Codex did** before the user accepts the changes.

The flow is: **Codex writes → Claude reviews → User decides what to keep or revert.**

### Hard gate: No further mutations until review is complete

After any write-mode rescue completes (success or failure):

1. **Do NOT** run further mutating commands (no additional codex runs, no commits, no file edits)
2. **Do NOT** start another rescue or Codex session
3. **Do NOT** proceed to the next task

Until the user has reviewed and approved/reverted the changes.

### Rule 1: Review before accepting

After Codex completes a write-mode rescue:

1. Run `git diff` to see exactly what changed
2. If started from a clean worktree (Option A): all changes are Codex's
3. If started from a dirty worktree (Option B): compare with `/tmp/codex_rescue_baseline.diff` to isolate Codex-only changes
4. Review the full diff for each touched file, not just `--stat` summaries
5. Present the changes to the user for approval

### Rule 2: Present results clearly

Structure the output as:

1. **What Codex found** — summary of the investigation or diagnosis
2. **What Codex changed** — full list of files modified with descriptions of each change
3. **Codex's reasoning** — why it made the choices it did
4. **Claude's assessment** — read the changed files and assess independently. Does Claude agree?

### Rule 3: Ask before finalizing

After presenting results, ask the user:
- "Which of these changes do you want to keep?"
- "Should I review any of these changes in detail?"
- "Want me to revert any of Codex's changes?"

**Selective revert** (when worktree was clean pre-rescue):
```bash
git checkout -- path/to/file  # revert a specific file
```

**Full revert** (when worktree was clean or stashed pre-rescue):
```bash
git checkout -- .  # revert all Codex changes
git stash pop      # restore user's original changes if stashed
```

### Rule 4: Compare perspectives

When relevant, compare Codex's approach with Claude's own analysis:
- Where they agree — reinforces confidence
- Where they disagree — present both sides, let user decide
- What each missed — complementary coverage

## Non-git environments

If the working directory is not a git repository:
- **Read-only rescue:** Allowed. Proceed normally.
- **Write-mode rescue:** NOT allowed. There is no rollback mechanism without git. Inform the user and suggest either initializing a git repo or restricting to read-only investigation.

## Session management

After every rescue run, inform the user:
"The Codex rescue session can be resumed. Ask to continue or provide a follow-up."

Track what Codex did in each session so follow-up context is accurate.

## Stderr handling

| Scenario | Approach |
|----------|----------|
| Read-only run | `2>/dev/null` — suppress thinking tokens |
| Write-mode run | `2>/tmp/codex_rescue_stderr.log` — persist for diagnostics |
| Non-zero exit (read-only) | Rerun without `2>/dev/null` to capture the real error |
| Non-zero exit (workspace-write) | Do NOT rerun. Check `git diff` for partial changes and read `/tmp/codex_rescue_stderr.log` for the error |
| User requests debug | Omit stderr redirection entirely |

## Error handling

| Error | Action |
|-------|--------|
| `codex: command not found` | Tell user to install: `npm install -g @openai/codex` |
| Non-zero exit (read-only) | Rerun without `2>/dev/null`, report stderr, ask for direction |
| Non-zero exit (workspace-write) | Check `git diff` for partial changes, read stderr log, present state to user. Do NOT rerun |
| Auth error / 401 | Suggest `codex login` |
| Not a git repository + write-mode | Refuse write-mode. Suggest read-only investigation or `git init` first |
| Not a git repository + read-only | Proceed; inform user that `--skip-git-repo-check` may be needed |
| No previous session to resume | Inform user; start a new session instead |
| Timeout (>120s no output) | Inform user the task is long-running; suggest checking back |
| Codex makes incorrect changes | Flag the issues, offer to revert. Present alternatives |
| Partial changes after failure | Present `git diff`, help user revert unwanted changes |
