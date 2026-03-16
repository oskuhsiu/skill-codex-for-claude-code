---
name: codex-info
description: Show Codex CLI version and login status. This skill should be used when the user asks to "check codex", "is codex installed", "codex status", "codex version", or wants to verify Codex CLI readiness before running tasks. Also use when the user asks about tool setup, available AI tools, environment readiness, or says "check my tools".
allowed-tools: Bash(codex:*)
---

## Codex CLI Status

- **Version**: !`codex --version 2>/dev/null || echo "NOT INSTALLED - run: npm install -g @openai/codex"`
- **Login**: !`codex login status 2>/dev/null && echo "Authenticated" || echo "Not logged in - run: codex login"`

Summarize Codex CLI readiness based on the status above.

If Codex is not installed, provide install instructions:
1. `npm install -g @openai/codex`
2. `codex login`

If installed but not logged in, suggest running `codex login`.
