---
name: codex-info
description: >-
  When the user asks about Codex CLI status, version, login state, or troubleshoots
  "codex not working / not found / auth error" — e.g., "codex 壞了", "codex 能用嗎",
  "check codex", "codex version", "codex 登入了嗎", "codex login 過期".
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
