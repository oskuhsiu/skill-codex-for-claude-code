---
name: codex-code
description: >-
  When the user asks Codex to write, generate, or implement code — e.g., "用 codex 寫...",
  "let codex implement this", "請 codex 實作", "codex code this". Not for reviews
  (codex-review), debugging (codex-rescue), or general tasks (codex).
allowed-tools: Bash(codex:*)
---

# Codex Code — Write Code with Codex

Codex writes code, Claude reviews it, user decides what to keep.
For shared patterns (permissions, stderr, sessions, errors), see
`skills/codex/references/common.md`.

## CLI readiness

- Version: !`codex --version 2>/dev/null || echo "NOT INSTALLED"`

If not installed: `npm install -g @openai/codex && codex login`.

## Pre-flight

Run the worktree check from `common.md`. Default sandbox is Tier 2
(`--full-auto`) since writing code always requires file writes.

## Build the coding prompt

The key difference from the general `codex` skill: wrap the user's request in a structured
prompt that steers Codex toward clean, idiomatic code.

### Step 1: Gather project context

```bash
ls .eslintrc* .prettierrc* pyproject.toml setup.cfg .editorconfig 2>/dev/null
ls -d test/ tests/ __tests__/ spec/ 2>/dev/null
ls package.json Cargo.toml go.mod requirements.txt Gemfile build.gradle pom.xml 2>/dev/null
```

### Step 2: Construct the prompt

```bash
codex exec --full-auto <<'PROMPT' 2>/tmp/codex_code_stderr.log
Task: <user's request>

Coding guidelines:
- Read existing code first. Match naming conventions, file structure, and patterns.
- Write minimal, production-ready code. No "// TODO" placeholders.
- Error handling only at system boundaries (user input, network, file I/O).
- Comments only where logic is genuinely non-obvious.
- No extra type annotations, docstrings, or defensive checks beyond project norms.
- No unnecessary abstractions for one-time operations.
- If tests exist (<test info>), write tests using the same framework and patterns.

<project context from Step 1>
PROMPT
```

Remove the testing line if no test directory was found. Add `-m` or `--config
model_reasoning_effort` only when user requests overrides.

If the task needs network access, escalate to `danger-full-access` (Tier 3 — requires
explicit "yes").

## Post-run review gate

Follow the post-run review gate from `common.md`. Additionally, review each changed file for:

1. **Correctness** — Does it do what was asked?
2. **Conventions** — Matches project style?
3. **Security** — OWASP top 10 concerns?
4. **Minimality** — Over-engineered or bloated?
5. **Tests** — If written, do they test the right things?

Present: what Codex built, files changed, Claude's assessment, then ask the user to
keep / review in detail / revert.
