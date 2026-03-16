# Cloud Tasks with Codex CLI

> **Note:** Cloud tasks are an EXPERIMENTAL feature. Commands and behavior may change between Codex versions.

Cloud tasks run Codex workloads on OpenAI's infrastructure instead of locally. Useful for long-running tasks, parallel work, or when local resources are limited.

## Task lifecycle

### 1. Submit a cloud task
```bash
codex cloud exec --env <ENV_ID> "your task description" 2>/dev/null
```

The `--env` flag is required and specifies the target environment (configured via Codex Cloud settings).

**With multiple attempts (best-of-N):**
```bash
codex cloud exec --env <ENV_ID> --attempts 3 "refactor the payment module" 2>/dev/null
```

### 2. List cloud tasks
```bash
codex cloud list 2>/dev/null
```

**With JSON output:**
```bash
codex cloud list --json 2>/dev/null
```

**With pagination:**
```bash
codex cloud list --limit 10 --cursor <cursor> 2>/dev/null
```

### 3. Check task status
Monitor a specific task by its ID (returned from `codex cloud exec`).

### 4. Apply results locally
```bash
codex apply <TASK_ID>
```

This applies the diff from the cloud task to the local workspace.

## Key flags

| Flag | Values | Description |
|------|--------|-------------|
| `--env <ENV_ID>` | environment ID | Target environment (required) |
| `--attempts <N>` | 1-4 | Number of parallel attempts (best-of-N) |
| `--limit <N>` | 1-20 | Number of tasks to list |
| `--cursor <TOKEN>` | pagination token | Continue listing from cursor |
| `--json` | boolean | JSON output for list command |

## When to use cloud tasks

| Scenario | Local (`codex exec`) | Cloud (`codex cloud exec`) |
|----------|---------------------|---------------------------|
| Quick analysis | Preferred | Overkill |
| Large refactoring | May timeout | Preferred |
| Multiple attempts needed | Manual retry | Built-in with `--attempts` |
| Need local file access | Yes | Limited to repo snapshot |
| Interactive follow-up | Yes (resume) | No |

## Important limitations

1. Cloud tasks work on a snapshot of the repository — they cannot access uncommitted local changes unless pushed
2. Results must be applied locally with `codex apply`
3. The `--env` flag is required and must be configured beforehand
4. This feature is experimental and may change
