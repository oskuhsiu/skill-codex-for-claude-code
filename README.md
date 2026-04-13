# codex-skill

A Claude Code plugin that delegates tasks to OpenAI's [Codex CLI](https://developers.openai.com/codex/cli/reference) with a safety-first design.

## Features

- **Zero-friction defaults** — uses your configured model and reasoning effort, no questions asked
- **Three-tier permission escalation** — read-only by default, explicit confirmation required for dangerous operations
- **Smart stderr handling** — suppresses thinking tokens but surfaces real errors on failure
- **Session resume** — continue Codex sessions with heredoc-safe multiline prompts
- **Structured output** — `--json`, `--output-schema`, `-o` support
- **Code review** — review uncommitted changes, branches, or commits
- **Adversarial review** — skeptical review that challenges design decisions and finds dangerous failure modes
- **Rescue / task delegation** — delegate debugging and implementation to Codex when stuck
- **Cloud tasks** — submit and manage cloud-based Codex workloads
- **Critical evaluation** — treats Codex as a peer, not an authority

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed
- [Codex CLI](https://www.npmjs.com/package/@openai/codex) v0.75.0+ installed and authenticated

```bash
npm install -g @openai/codex
codex login
```

## Installation

Clone this repo and install the plugin in Claude Code:

```bash
git clone https://github.com/your-username/codex-skill.git
```

Then in Claude Code, add the plugin directory:

```
/plugin install /path/to/codex-skill
```

## Usage

The skill activates automatically when you mention Codex, or invoke it directly:

```
Ask codex to analyze the authentication module
```

```
/codex refactor the payment service to use dependency injection
```

```
/codex-info
```

### Examples

| Task | What happens |
|------|-------------|
| "ask codex to review this code" | Runs `codex exec --sandbox read-only` (Tier 1, no confirmation) |
| "let codex refactor the auth module" | Runs `--full-auto` (Tier 2, informs you) |
| "codex needs network access for this" | Asks for explicit confirmation before `--sandbox danger-full-access` (Tier 3) |
| "resume codex" | Continues the previous session with context |
| "what would codex do differently?" | Delegates to Codex, then compares approaches |

### Dedicated Skills

| Skill | Command | Purpose |
|-------|---------|---------|
| **Code Review** | `/codex-review` | One-command code review — uncommitted changes, branch diffs, or commits |
| **Adversarial Review** | `/codex-adversarial-review` | Skeptical review challenging design decisions and failure modes |
| **Rescue** | `/codex-rescue` | Delegate debugging, investigation, or implementation to Codex |
| **Status Check** | `/codex-info` | Check Codex CLI installation and login status |

## Permission Model

| Tier | Access Level | Confirmation |
|------|-------------|-------------|
| Safe | `--sandbox read-only` | None needed |
| Standard | `--full-auto` | Informed, proceed unless objected |
| Dangerous | `--sandbox danger-full-access` | Explicit "yes" required |

`--dangerously-bypass-approvals-and-sandbox` is actively discouraged. Even when requested, the skill recommends `--sandbox danger-full-access` as a safer alternative.

## Project Structure

```
codex-skill/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── codex/
│   │   ├── SKILL.md                 # Core skill — general task delegation
│   │   └── references/
│   │       ├── flags-reference.md   # Complete CLI flag docs
│   │       ├── review-mode.md       # Code review workflows
│   │       └── cloud-tasks.md       # Cloud task management
│   ├── codex-review/
│   │   └── SKILL.md                 # Dedicated code review skill
│   ├── codex-adversarial-review/
│   │   ├── SKILL.md                 # Adversarial review skill
│   │   └── references/
│   │       ├── adversarial-prompt.md       # Adversarial prompt template
│   │       └── review-output.schema.json   # JSON Schema for structured output
│   ├── codex-rescue/
│   │   └── SKILL.md                 # Rescue / task delegation skill
│   └── codex-info/
│       └── SKILL.md                 # Quick status check
├── LICENSE
└── README.md
```

