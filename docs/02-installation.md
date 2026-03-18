# Installation

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| Node.js | ≥ 20.6.0 | 22+ (LTS) |
| npm | any (ships with Node) | npm 10+ |
| Git | any | any |
| LLM provider | one of 20+ supported | Anthropic API key or Claude Max subscription |

Git is initialized automatically if missing from the project directory. The `gsd-pi` package manager is `npm` — npm is the canonical package manager for this project.

> **Node.js on macOS via Homebrew:** Homebrew may install a development release of Node rather than LTS. If you encounter compatibility issues, pin Node 22 LTS. The Node.js requirement is `>=20.6.0` per `package.json`; versions below this will not work.

## Installation

Install GSD globally with npm:

```bash
npm install -g gsd-pi
```

This installs two binary aliases pointing to the same executable:

- `gsd` — primary binary name
- `gsd-cli` — alternative name (use this if `gsd` is shadowed by another tool)

### Shell Alias Conflict (zsh / oh-my-zsh)

The [oh-my-zsh git plugin](https://github.com/ohmyzsh/ohmyzsh/tree/master/plugins/git) defines `alias gsd='git svn dcommit'`, which shadows the GSD binary. Fix it with one of:

**Option 1** — Unalias in `~/.zshrc` (add after the `source $ZSH/oh-my-zsh.sh` line):

```bash
unalias gsd 2>/dev/null
```

**Option 2** — Use the alternative binary:

```bash
gsd-cli
```

Both `gsd` and `gsd-cli` point to the same binary.

### Verifying Installation

```bash
gsd --version
```

Or launch GSD and check the version displayed in the startup header:

```bash
gsd
```

## First-Run Setup

Run `gsd` in any directory:

```bash
gsd
```

On first launch, GSD runs a branded setup wizard with two stages:

### Stage 1: LLM Provider

Select your AI provider from 20+ options. The wizard presents them interactively:

- **Anthropic** (Claude) — paste your API key, or authenticate via OAuth if you have a Claude Max subscription
- **OpenAI** — paste your API key, or authenticate via OAuth if you have a ChatGPT Plus/Pro (Codex) subscription
- **Google Gemini** — paste your API key (strongly recommended over OAuth — see warning below)
- **GitHub Copilot** — OAuth flow handles authentication automatically
- **OpenRouter** — paste one API key and gain access to hundreds of models (Llama, DeepSeek, Qwen, etc.)
- **Amazon Bedrock, Azure OpenAI, Google Vertex, Groq, Cerebras, Mistral, xAI, and more**

GSD auto-selects a default model after login.

> **Warning — Google Gemini OAuth:** Using Gemini CLI or Antigravity OAuth tokens in third-party tools has resulted in Google account suspensions. This affects the entire Google account, not just the Gemini service. Use a Gemini API key instead of OAuth.

> **Claude Max and GitHub Copilot OAuth:** Usage outside the native application may be restricted by the provider's Terms of Service. API keys are the safe alternative for all providers.

### Stage 2: Tool API Keys (Optional)

The wizard then prompts for optional tool keys that extend GSD's capabilities:

| Tool | Key | Capability |
|------|-----|-----------|
| Brave Search | `BRAVE_SEARCH_API_KEY` | Web search during research phase |
| Tavily | `TAVILY_API_KEY` | Alternative web search |
| Context7 | API key | Up-to-date library and framework docs |
| Jina | API key | Page extraction |
| Slack | token | Remote questions and notifications |
| Discord | token | Remote questions and notifications |

Press Enter at any prompt to skip. All keys can be added later.

### Re-Running the Wizard

Run the wizard again at any time:

```bash
gsd config
```

All credentials are saved to `~/.gsd/agent/auth.json` and apply globally across all projects.

### Importing an Existing Pi Installation

If you have an existing Pi (`@mariozechner/pi-coding-agent`) installation, GSD automatically imports your stored provider credentials and LLM/tool keys on first launch. No re-entry is required.

## API Key Configuration

### LLM Provider Keys

After initial setup, change your LLM provider or add keys for additional providers with:

```bash
/login
```

(Inside a GSD session.)

### Tool API Keys

```bash
/gsd config
```

(Inside a GSD session — re-runs the full wizard including tool keys.)

Keys are stored globally at `~/.gsd/agent/auth.json`.

### Model Selection

GSD selects a default model automatically after login. Switch at any time:

```bash
/model
```

Or configure per-phase models in preferences (see below).

## Preferences

Global preferences live at `~/.gsd/preferences.md`. Project-level preferences live at `.gsd/preferences.md` inside the project. Manage with `/gsd prefs` inside a session.

A minimal example showing model configuration:

```yaml
---
version: 1
models:
  research: claude-sonnet-4-6
  planning:
    model: claude-opus-4-6
    fallbacks:
      - openrouter/z-ai/glm-5
  execution: claude-sonnet-4-6
  completion: claude-sonnet-4-6
budget_ceiling: 50.00
verification_commands:
  - npm run lint
  - npm run test
---
```

## Quick-Start Workflow

### Start a New Project

Open a terminal in your project directory and run:

```bash
gsd
```

Then type `/gsd` inside the session. If no `.gsd/` directory exists, GSD starts a discussion flow to capture your project vision, constraints, and preferences. This populates `PROJECT.md`, `REQUIREMENTS.md`, and queues the first milestone.

### Step Mode — Stay in the Loop

```bash
/gsd
```

GSD executes one unit of work at a time, pausing between each step with a wizard showing what completed and what is next. Use this as the on-ramp when you want to review output at each stage.

- No `.gsd/` directory → starts a discussion flow to capture your project vision
- Milestone exists, no roadmap → discuss or research the milestone
- Roadmap exists, slices pending → plan the next slice or execute a task
- Mid-task → resumes where it left off

### Auto Mode — Walk Away

```bash
/gsd auto
```

GSD autonomously researches, plans, executes, verifies, commits, and advances through every slice until the milestone is complete. Fresh context window per task. No babysitting required.

### The Two-Terminal Workflow

The recommended working pattern: auto mode in one terminal, steering from another.

**Terminal 1 — let it build:**

```bash
gsd
/gsd auto
```

**Terminal 2 — steer while it works:**

```bash
gsd
/gsd discuss    # talk through architecture decisions
/gsd status     # check progress
/gsd queue      # queue the next milestone
```

Both terminals read and write the same `.gsd/` files on disk. Decisions from terminal 2 are picked up automatically at the next phase boundary.

### Headless Mode — CI and Scripts

```bash
# Run auto mode in CI
gsd headless --timeout 600000

# Create and execute a milestone end-to-end
gsd headless new-milestone --context spec.md --auto

# One unit at a time (cron-friendly)
gsd headless next

# Instant JSON snapshot — no LLM session, ~50ms
gsd headless query
```

Exit codes: `0` complete, `1` error or timeout, `2` blocked.

### Resuming Sessions

```bash
gsd --continue    # or gsd -c — resume most recent session
gsd sessions      # interactive picker — browse all saved sessions
```

## Updates

GSD checks for updates once every 24 hours. When a new version is available, an interactive prompt appears at startup offering to update immediately or skip.

Update from within a session (added in v2.28.0):

```bash
/gsd update
```

Update from the command line:

```bash
gsd update
```

Or via npm directly:

```bash
npm update -g gsd-pi
```

Bundled extensions and agents are synced to `~/.gsd/agent/` on every launch (not just on update), so `npm update -g` takes effect immediately.

## Health Check

After installation or after a crash, run the built-in health checker:

```bash
/gsd doctor
```

This runs runtime health checks and auto-fixes common issues including roadmap checkbox integrity, missing STATE.md, and stale runtime files.

## Migrating from v1 (`.planning` Directories)

If you have projects using the original Get Shit Done v1 format (`.planning` directories), GSD-2 includes a migration tool.

> Migration works best with a `ROADMAP.md` file in the project root. Without one, milestones are inferred from the `phases/` directory.

```bash
# From within the project directory
/gsd migrate

# Or specify a path explicitly
/gsd migrate ~/projects/my-old-project
```

The migration tool:

- Parses `PROJECT.md`, `ROADMAP.md`, `REQUIREMENTS.md`, phase directories, plans, summaries, and research from the v1 layout
- Maps phases to slices, plans to tasks, milestones to milestones
- Preserves completion state — `[x]` phases remain marked done, summaries carry over
- Consolidates research files into the new `.gsd/` structure
- Shows a preview of all changes before writing anything
- Optionally runs an agent-driven review of the output for quality assurance

**Supported v1 format variations:**

- Milestone-sectioned roadmaps with `<details>` blocks
- Bold phase entries
- Bullet-format requirements
- Decimal phase numbering
- Duplicate phase numbers across milestones

**After migration, verify the output:**

```bash
/gsd doctor
```

The doctor command checks `.gsd/` structural integrity and flags any issues introduced during migration.

## Debug Mode

Start GSD with structured diagnostic logging:

```bash
gsd --debug
```

This enables JSONL diagnostic logs capturing dispatch decisions, state transitions, and timing data. Useful for troubleshooting auto-mode failures. For post-mortem analysis of failures that have already occurred:

```bash
/gsd forensics
```
