# codex-skill

A Claude Code plugin that delegates tasks to OpenAI's [Codex CLI](https://developers.openai.com/codex/cli/reference) with a safety-first design.

## Features

- **Zero-friction defaults** — uses your configured model and reasoning effort, no questions asked
- **Three-tier permission escalation** — read-only by default, explicit confirmation required for dangerous operations
- **Smart stderr handling** — suppresses thinking tokens but surfaces real errors on failure
- **Session resume** — continue Codex sessions with heredoc-safe multiline prompts
- **Structured output** — `--json`, `--output-schema`, `-o` support
- **Code review** — review uncommitted changes, branches, or commits
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
| "let codex refactor the auth module" | Runs `--sandbox workspace-write --full-auto` (Tier 2, informs you) |
| "codex needs network access for this" | Asks for explicit confirmation before `--sandbox danger-full-access` (Tier 3) |
| "resume codex" | Continues the previous session with context |
| "what would codex do differently?" | Delegates to Codex, then compares approaches |

## Permission Model

| Tier | Access Level | Confirmation |
|------|-------------|-------------|
| Safe | `--sandbox read-only` | None needed |
| Standard | `--sandbox workspace-write --full-auto` | Informed, proceed unless objected |
| Dangerous | `--sandbox danger-full-access` | Explicit "yes" required |

`--dangerously-bypass-approvals-and-sandbox` is actively discouraged. Even when requested, the skill recommends `--sandbox danger-full-access` as a safer alternative.

## Project Structure

```
codex-skill/
├── .claude-plugin/
│   └── plugin.json
├── skills/
│   ├── codex/
│   │   ├── SKILL.md                 # Core skill (185 lines)
│   │   └── references/
│   │       ├── flags-reference.md   # Complete CLI flag docs
│   │       ├── review-mode.md       # Code review workflows
│   │       └── cloud-tasks.md       # Cloud task management
│   └── codex-info/
│       └── SKILL.md                 # Quick status check
├── LICENSE
└── README.md
```

## Comparison with skill-codex

| Capability | [skill-codex](https://github.com/skills-directory/skill-codex) | codex-skill (this) |
|---|---|---|
| Model selection | Hardcoded 3 models, asks every time | User's config default, zero questions |
| Reasoning effort | Asks every time | User's config default, override on request |
| `--skip-git-repo-check` | Always on | Only when needed + user confirms |
| Stderr handling | `2>/dev/null` hides ALL errors | Suppresses thinking tokens; reruns on failure to show real errors |
| Resume | Pipe-only (`echo \| codex exec resume`) | Heredoc for multiline safety |
| Structured output | Not supported | `--json`, `--output-schema`, `-o` |
| Code review | Not supported | Full review workflow |
| Cloud tasks | Not supported | Full cloud workflow |
| Image attachment | Not supported | `-i <path>` |
| Additional directories | Not supported | `--add-dir <path>` |
| Profiles | Not supported | `-p <profile>` |
| Ephemeral mode | Not supported | `--ephemeral` |
| Permission model | Ad-hoc asks | Three-tier escalation |
| Dynamic context | Not used | `!`codex --version`` via `!` syntax |
| Architecture | Monolithic SKILL.md | Progressive disclosure with references/ |
| `--yolo` safety | Asks permission | Actively discourages, recommends safer alternative |
| Model future-proofing | Hardcoded names go stale | Relies on user config |

