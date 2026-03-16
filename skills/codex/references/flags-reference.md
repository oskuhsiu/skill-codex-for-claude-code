# Codex CLI Flags Reference

Complete reference for `codex exec` flags, organized by category.

## Table of Contents
- [Model & Reasoning](#model--reasoning)
- [Sandbox & Safety](#sandbox--safety)
- [Input & Output](#input--output)
- [Context & Working Directory](#context--working-directory)
- [Feature Flags](#feature-flags)
- [Resume & Sessions](#resume--sessions)

## Model & Reasoning

| Flag | Values | Description |
|------|--------|-------------|
| `-m, --model <MODEL>` | Any supported model ID | Override the configured model. Omit to use user's default from config.toml |
| `--config model_reasoning_effort="<LEVEL>"` | `low`, `medium`, `high`, `xhigh` | Set reasoning effort. Omit to use user's default |
| `--oss` | boolean | Use local open source model via Ollama instead of OpenAI API |
| `-p, --profile <NAME>` | profile name | Load a named configuration profile from config.toml |

**Best practice:** Never hardcode model names. Models change frequently. Always rely on the user's configured defaults unless they explicitly request an override.

## Sandbox & Safety

| Flag | Values | Description |
|------|--------|-------------|
| `-s, --sandbox <MODE>` | `read-only`, `workspace-write`, `danger-full-access` | Sandbox policy for shell commands |
| `-a, --ask-for-approval <MODE>` | `untrusted`, `on-request`, `never` | When to ask before executing commands |
| `--full-auto` | boolean | Shortcut: `workspace-write` sandbox + `on-request` approvals |
| `--dangerously-bypass-approvals-and-sandbox` / `--yolo` | boolean | No approvals, no sandbox. Only for externally hardened environments |
| `--skip-git-repo-check` | boolean | Allow running outside a git repository |

### Sandbox modes explained

| Mode | File access | Network | Risk |
|------|------------|---------|------|
| `read-only` | Read only | No | Minimal |
| `workspace-write` | Read + write in workspace | No | Moderate |
| `danger-full-access` | Full system access | Yes | High |

### Safety combinations to avoid

- `--full-auto` + `--dangerously-bypass-approvals-and-sandbox` — removes all safety guardrails
- `--sandbox danger-full-access` + `-a never` — full access with no approval prompts
- `--skip-git-repo-check` without understanding why — may run in unintended directories

## Input & Output

| Flag | Values | Description |
|------|--------|-------------|
| `--json` | boolean | Output newline-delimited JSON events instead of text |
| `-o, --output-last-message <FILE>` | file path | Write the assistant's final message to a file |
| `--output-schema <FILE>` | JSON Schema file | Validate response shape against a JSON Schema |
| `--color` | `always`, `never`, `auto` | Color output control |
| `--ephemeral` | boolean | No disk persistence for this session |
| `-i, --image <PATH>` | file path(s) | Attach image files to the initial prompt. Comma-separated for multiple |

### Structured output patterns

**Save result to file for later processing:**
```bash
codex exec -o /tmp/result.txt --sandbox read-only "analyze this code" 2>/dev/null
```

**JSONL event stream for programmatic consumption:**
```bash
codex exec --json --sandbox read-only "list all functions" 2>/dev/null
```

**Validated output with schema:**
```bash
codex exec --output-schema schema.json --sandbox read-only "extract API endpoints" 2>/dev/null
```

## Context & Working Directory

| Flag | Values | Description |
|------|--------|-------------|
| `-C, --cd <DIR>` | directory path | Set working directory before the agent starts |
| `--add-dir <PATH>` | directory path | Grant additional directory write access alongside the workspace |
| `--search` | boolean | Enable live web search instead of cached |

### Working directory best practices

- Default to the current working directory (no `-C` flag)
- Validate directory exists before passing `-C`
- Use `--add-dir` when Codex needs access to dependencies or shared code outside the workspace
- Prefer `--add-dir` over `--sandbox danger-full-access` when only additional directory access is needed

## Feature Flags

| Flag | Values | Description |
|------|--------|-------------|
| `--enable <FEATURE>` | feature name | Force-enable a feature flag (repeatable) |
| `--disable <FEATURE>` | feature name | Force-disable a feature flag (repeatable) |

Manage feature flags with `codex features list`, `codex features enable <feature>`, `codex features disable <feature>`.

## Resume & Sessions

| Flag | Values | Description |
|------|--------|-------------|
| `resume` | subcommand | Resume a previous session |
| `--last` | boolean | Resume the most recent session |
| `--all` | boolean | Show all sessions to pick from |

### Resume syntax

**Simple follow-up:**
```bash
echo "add error handling to the function" | codex exec resume --last 2>/dev/null
```

**Multiline follow-up (heredoc for safety):**
```bash
codex exec resume --last <<'PROMPT' 2>/dev/null
Please also:
1. Add input validation
2. Write unit tests
3. Update the docstring
PROMPT
```

**Resume with flag override (only when user requests):**
```bash
echo "retry with more reasoning" | codex exec --config model_reasoning_effort="high" resume --last 2>/dev/null
```

When resuming, the session inherits the original model, sandbox, and reasoning settings. Only add flags if the user explicitly requests changes.
