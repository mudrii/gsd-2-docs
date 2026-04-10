# GSD-2 User Guide

Comprehensive reference for GSD-2, the autonomous coding agent built on the Pi SDK. This guide covers released functionality as of v2.67.0.

## Installation and Requirements

### System Requirements

- Node.js >= 22.0.0 (24 LTS recommended)
- Git
- Supported platforms: darwin-arm64, darwin-x64, linux-x64, linux-arm64, win32-x64

At startup, GSD runs checks to verify that the Node.js version meets the minimum requirement and that `git` is available on `$PATH`. If either check fails, a clear error message is shown before the process exits (v2.46.0).

### Install via npm

```bash
npm install -g gsd-pi
```

GSD checks for updates once every 24 hours. When a new version is available, an interactive prompt appears at startup with the option to update immediately or skip. Update from within a session with `/gsd update`.

### Docker Sandbox

An official Docker sandbox template is available for running GSD auto mode in an isolated container (v2.44.0):

```bash
# Build and run the official GSD Docker image
docker build -t gsd-sandbox .
docker run -it gsd-sandbox
```

The Dockerfile uses a multi-stage build with proven container patterns. This is useful for CI environments or when you want full filesystem isolation for auto-mode execution.

### VS Code Extension

Available on the VS Code Marketplace, published by FluxLabs. Search for "GSD" in VS Code extensions or install from:
https://marketplace.visualstudio.com/items?itemName=FluxLabs.gsd-2

The extension (v0.3.0, 1,578 installs) provides:
- `@gsd` chat participant in VS Code Chat
- Sidebar redesign with SCM provider integration, checkpoints, and diagnostics (v2.59.0 phase 3)
- Full command palette for start/stop, model switching, and session export
- Activity feed, workflow controls, session forking, and enhanced code lens (v2.53.0)
- Status bar, file decorations, bash terminal, session tree, and conversation history (v2.52.0)
- Remote SSH support via `extensionKind` (v2.51.0)

The CLI (`gsd-pi`) must be installed first -- the extension connects to it via RPC.

### API Key Setup

Run `/gsd config` inside any session to set API keys globally. Keys are saved to `~/.gsd/agent/auth.json` and apply to all projects.

If using a non-Anthropic model, a search API key (Brave Search, Jina, etc.) is required for web search.

## First Run and Onboarding Wizard

Run `gsd` in any directory. On first launch, a setup wizard runs:

1. **LLM Provider** -- select from 20+ providers (Anthropic, OpenAI, Google, OpenRouter, GitHub Copilot, Amazon Bedrock, Azure, and more). OAuth flows handle Claude Max and Copilot subscriptions automatically; otherwise paste an API key.
2. **Tool API Keys** (optional) -- Brave Search, Context7, Jina, Slack, Discord. Press Enter to skip any.

Existing Pi installations have provider credentials imported automatically. Re-run the wizard anytime with `gsd config`.

## Step Mode vs Auto Mode

### Step Mode (`/gsd`)

Type `/gsd` inside a session. GSD executes one unit of work at a time, pausing between each with a wizard showing what completed and what comes next.

The behavior adapts to project state:
- No `.gsd/` directory: starts a discussion flow to capture the project vision
- Milestone exists, no roadmap: discuss or research the milestone
- Roadmap exists, slices pending: plan the next slice or execute a task
- Mid-task: resume where you left off

Step mode keeps you in the loop, reviewing output between each step.

### Auto Mode (`/gsd auto`)

Type `/gsd auto` and walk away. GSD autonomously researches, plans, executes, verifies, commits, and advances through every slice until the milestone is complete.

Auto mode reads `.gsd/STATE.md`, determines the next unit of work, creates a fresh agent session, and injects a focused prompt with all relevant context pre-inlined. Every task gets a clean context window.

Use `--yolo` to skip the interactive project init prompt when launching auto mode in a new project (v2.49.0).

## Web Mode

GSD includes a browser-based interface for visual monitoring and interaction.

### Starting Web Mode

```bash
gsd --web     # start the web server
gsd web       # alternative syntax
```

### Configuration Flags (v2.42.0)

| Flag | Description | Default |
|------|-------------|---------|
| `--host <addr>` | Bind address | `localhost` |
| `--port <num>` | Listen port | auto-assigned |
| `--allowed-origins <list>` | CORS allowed origins | same-origin |

### Features

- **Mobile responsive** -- the web UI adapts to phone and tablet screens (v2.45.0)
- **Auth token gate** -- sessions require an auth token; a synthetic 401 is returned on missing tokens, with an unauthenticated boot state and recovery screen (v2.52.0). Tokens are stored in `localStorage` for multi-tab usage.
- **Dark mode** -- a dedicated dark theme with raised token floor and flattened opacity tiers for improved terminal contrast (v2.52.0). Light theme also has improved contrast (v2.56.0).
- **Change project root** -- a button in the web UI lets you switch the project directory without restarting the server (v2.44.0)
- **Dashboard metrics** -- falls back to project totals when dashboard metrics are zero (v2.58.0)
- **Persistent notification panel** -- non-modal notification panel for background events and status updates (v2.65.0)

## Headless Mode

Run GSD commands without the interactive TUI, suitable for CI pipelines, scripted workflows, and orchestrator integration.

### Basic Usage

```bash
gsd headless "prompt"        # run a prompt
gsd headless auto            # run auto mode (default)
gsd headless next            # run one unit
gsd headless status          # show progress dashboard
gsd headless query           # instant JSON state snapshot (no LLM)
gsd headless new-milestone --context spec.md   # create milestone from file
```

### Flags

| Flag | Description |
|------|-------------|
| `--bare` | Minimal context: skip CLAUDE.md, AGENTS.md, user settings, user skills |
| `--events <types>` | Filter JSONL output to specific event types (comma-separated) |
| `--answers <path>` | Pre-supply answers and secrets from a JSON file |
| `--resume <id>` | Resume a prior headless session by ID (prefix matching supported, v2.57.0) |
| `--verbose` | Show tool calls in progress output |
| `--timeout N` | Overall timeout in ms (default: 300000; disabled for auto-mode, v2.50.0) |
| `--json` | JSONL event stream to stdout |
| `--output-format <fmt>` | `text` (default), `json` (structured result), `stream-json` (JSONL events) |
| `--model ID` | Override model |
| `--supervised` | Forward interactive UI requests to orchestrator via stdout/stdin |
| `--response-timeout N` | Timeout (ms) for orchestrator response (default: 30000) |

### Output

Headless text mode provides colorized output with thinking indicators, phase names, cost summaries, and durations (v2.55.0). Tool calls are shown when `--verbose` is enabled.

### Auto-Mode Integration

Headless mode integrates with auto mode seamlessly:
- Auto-responds to interactive prompts
- Detects completion and exits with structured codes
- Auto-restarts on crash with exponential backoff (default 3 attempts)
- Combined with provider error auto-resume, this enables true overnight unattended execution
- UAT pauses are skipped in headless mode (v2.55.0)

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Error or timeout |
| 10 | Blocked |
| 11 | Cancelled |

## Two-Terminal Workflow

The recommended workflow: auto mode in one terminal, steering from another.

**Terminal 1 -- let it build:**

```bash
gsd
/gsd auto
```

**Terminal 2 -- steer while it works:**

```bash
gsd
/gsd discuss    # talk through architecture decisions
/gsd status     # check progress
/gsd queue      # queue the next milestone
```

Both terminals read and write the same `.gsd/` files. Decisions in terminal 2 are picked up at the next phase boundary automatically.

## Project Structure

GSD organizes work into a hierarchy:

```
Milestone  ->  a shippable version (4-10 slices)
  Slice    ->  one demoable vertical capability (1-7 tasks)
    Task   ->  one context-window-sized unit of work
```

The iron rule: a task must fit in one context window. If it cannot, it is two tasks.

All state lives on disk in `.gsd/`:

```
.gsd/
  PROJECT.md          -- what the project is right now
  REQUIREMENTS.md     -- requirement contract (active/validated/deferred)
  DECISIONS.md        -- append-only architectural decisions
  KNOWLEDGE.md        -- cross-session rules, patterns, and lessons
  RUNTIME.md          -- runtime context: API endpoints, env vars, services (v2.39)
  STATE.md            -- quick-glance status
  milestones/
    M001/
      M001-ROADMAP.md -- slice plan with risk levels and dependencies
      M001-CONTEXT.md -- scope and goals from discussion
      slices/
        S01/
          S01-PLAN.md     -- task decomposition
          S01-SUMMARY.md  -- what happened
          S01-UAT.md      -- human test script
          tasks/
            T01-PLAN.md
            T01-SUMMARY.md
```

## Commands Reference

### Core Workflow

| Command | Description |
|---------|-------------|
| `/gsd` | Step mode -- execute one unit at a time |
| `/gsd auto` | Auto mode -- autonomous execution to milestone completion |
| `/gsd status` | Progress dashboard with cost, phase, and health info |
| `/gsd discuss` | Discuss architecture decisions; can target queued milestones (v2.48.0) |
| `/gsd queue` | Queue the next milestone |
| `/gsd next` | Run one unit and stop |
| `/gsd show-config` | Display resolved configuration from all sources (v2.66.0) |
| `/btw` | Ephemeral side question -- ask something without affecting the current task context (v2.60.0) |

### Project Management

| Command | Description |
|---------|-------------|
| `/gsd rethink` | Conversational project reorganization -- restructure milestones, slices, and priorities through discussion (v2.45.0) |
| `/gsd mcp` | Show MCP server status and connectivity for configured servers (v2.45.0) |
| `/gsd changelog` | LLM-summarized release notes for GSD versions (v2.35.0) |
| `/gsd logs` | Browse activity, debug, and metrics logs (v2.29.0) |
| `/gsd keys` | Comprehensive API key manager -- add, remove, and verify keys (v2.29.0) |
| `/gsd fast` | Switch to fast service tier for supported models (v2.42.0) |
| `/gsd rate` | Show current rate limit status and token profile info (v2.36.0) |

### Diagnostics and Debugging

| Command | Description |
|---------|-------------|
| `/gsd doctor` | Validate `.gsd/` integrity, git health, and environment |
| `/gsd forensics` | Full-access GSD debugger -- post-mortem analysis with journal and activity log awareness (upgraded v2.40.0, enhanced v2.48.0) |

### Shell and Execution

| Command | Description |
|---------|-------------|
| `/terminal` | Direct shell execution from within a GSD session (v2.51.0) |

### Ephemeral Side Questions

| Command | Description |
|---------|-------------|
| `/btw` | Ask an ephemeral side question without polluting the current task context (v2.60.0) |

### Parallel Orchestration

| Command | Description |
|---------|-------------|
| `/gsd parallel start` | Analyze eligibility and start workers |
| `/gsd parallel status` | Show worker state, units, cost |
| `/gsd parallel stop [MID]` | Stop all or specific worker |
| `/gsd parallel pause [MID]` | Pause all or specific worker |
| `/gsd parallel resume [MID]` | Resume paused workers |
| `/gsd parallel merge [MID]` | Merge completed milestones |
| `/gsd parallel watch` | Native TUI overlay for real-time worker monitoring (v2.56.0) |

### Other

| Command | Description |
|---------|-------------|
| `/model` | Switch model within a session |
| `/gsd config` | Re-run the setup wizard |
| `/gsd update` | Update GSD to latest version |
| `/gsd migrate` | Migrate from v1 `.planning` directories |
| `/worktree` (or `/wt`) | Manage git worktrees (list, merge, clean, remove) |

## OS-Specific Keyboard Shortcut Hints (v2.66.1)

The TUI adapts keyboard shortcut hints to the current platform. On macOS, shortcuts display with `Cmd` and `Opt` modifiers; on Linux and Windows, `Ctrl` and `Alt` are shown instead. This applies to the status bar, help overlays, and all inline shortcut references throughout the interface.

## Ollama and Local LLM Support (v2.59.0+)

GSD supports running local models via Ollama for offline or privacy-sensitive workflows.

### Setup

Install Ollama and pull a model:

```bash
ollama pull llama3
```

GSD auto-discovers running Ollama instances on the default port. No additional configuration is required -- available models appear in the model picker automatically.

### Commands

| Command | Description |
|---------|-------------|
| `/ollama status` | Show Ollama connection status and available models |
| `/ollama models` | List all locally available models |
| `/ollama pull <model>` | Pull a model from the Ollama registry |

Local models can be assigned to specific phases via the standard model configuration in preferences. The dynamic routing system treats Ollama models as a local provider alongside cloud providers.

## LLM Safety Harness (v2.64.0)

The safety harness prevents destructive operations during autonomous execution. Before executing potentially irreversible actions (file deletions, database writes, force pushes), GSD creates a checkpoint of the current project state. If the operation fails or produces unexpected results, the harness can roll back to the checkpoint automatically.

### Behavior

- **Checkpoint creation:** automatic before destructive tool calls in auto mode
- **Rollback:** triggered on task failure or verification gate rejection
- **Manual rollback:** available via `/gsd doctor` recovery procedures
- **Scope:** applies to filesystem changes and git operations within the project directory

The harness is enabled by default in auto mode and does not require configuration.

## Slice-Level Parallelism (v2.64.0)

Within a single milestone, multiple slices can now execute in parallel when their dependency graphs allow it. This is distinct from milestone-level parallel orchestration -- slice-level parallelism runs concurrent slices inside one milestone worker.

GSD analyzes the slice dependency declarations in the roadmap and identifies slices that have no mutual dependencies. Eligible slices are dispatched to concurrent task runners sharing the same worktree, with file-level locking to prevent conflicts.

Enable via preferences:

```yaml
parallel:
  slice_parallelism: true
```

## Multi-Runtime Support

GSD supports 9 runtimes for agent execution:

1. Claude Code
2. OpenCode
3. Gemini CLI
4. Codex
5. GitHub Copilot
6. Antigravity
7. Cursor
8. Windsurf
9. Augment

### RTK Integration

Shell output compression uses RTK (Runtime Toolkit) integration to reduce token consumption from verbose command output. Long shell outputs are automatically summarized before being fed back into the context window, preserving relevant information while minimizing token waste.

## Git Strategy

### Isolation Modes

GSD supports three isolation modes configured via `git.isolation`:

| Mode | Working Directory | Branch | Best For |
|------|-------------------|--------|----------|
| `worktree` | `.gsd/worktrees/<MID>/` | `milestone/<MID>` | Full file isolation between milestones |
| `branch` | Project root | `milestone/<MID>` | Submodule-heavy repos |
| `none` (default since v2.46.0) | Project root | Current branch | Hot-reload workflows |

### Worktree Mode

Each milestone gets its own git worktree. All execution happens inside the worktree. On completion, the worktree is squash-merged to main as one clean commit. The worktree and branch are then cleaned up.

### Branch Mode

Work happens in the project root on a `milestone/<MID>` branch. No worktree is created. On completion, the branch is merged to main (squash or regular merge, per `merge_strategy`).

### None Mode

Work happens directly on the current branch. No worktree, no milestone branch. GSD still commits sequentially with conventional commit messages, but there is no branch isolation.

### Commit Format

GSD metadata is stored in git trailers rather than commit subject scopes (v2.49.0):

```
feat: core type definitions

GSD-Milestone: M001
GSD-Slice: S01
GSD-Task: T01
```

### Merge Strategies

In worktree and branch modes, all commits are squashed into one clean commit on main (configurable via `merge_strategy`).

### Worktree Management Commands

```
/worktree create
/worktree switch
/worktree merge
/worktree remove
```

Also available as `/wt`.

### Workflow Modes

Set `mode` in preferences for sensible defaults:

```yaml
mode: solo    # personal projects -- auto-push, squash, simple IDs
mode: team    # shared repos -- unique IDs, push branches, pre-merge checks
```

### Self-Healing

GSD includes automatic recovery for common git issues:
- Detached HEAD -- automatically reattaches to the correct branch
- Stale lock files -- removes `index.lock` files from crashed processes
- Orphaned worktrees -- detects and offers to clean up abandoned worktrees
- Dirty working tree on merge -- auto-stashes dirty files before squash merge (v2.44.0)
- Build artifact conflicts -- auto-resolved during milestone merge (v2.53.0)

Run `/gsd doctor` to check git health manually.

### Native Git Operations

Since v2.16, GSD uses libgit2 via native bindings for read-heavy operations in the dispatch hot path, eliminating approximately 70 process spawns per dispatch cycle.

## Working in Teams

### Setup

Set `mode: team` in project preferences:

```yaml
# .gsd/PREFERENCES.md (project-level, committed to git)
---
version: 1
mode: team
---
```

This enables unique milestone IDs, push branches, and pre-merge checks in one setting.

### .gitignore Configuration

Share planning artifacts while keeping runtime files local:

```bash
# Runtime / Ephemeral (per-developer, per-session)
.gsd/auto.lock
.gsd/completed-units.json
.gsd/STATE.md
.gsd/metrics.json
.gsd/activity/
.gsd/runtime/
.gsd/worktrees/
.gsd/milestones/**/continue.md
.gsd/milestones/**/*-CONTINUE.md
```

**What gets shared** (committed to git):
- `.gsd/PREFERENCES.md` -- project preferences
- `.gsd/PROJECT.md` -- living project description
- `.gsd/REQUIREMENTS.md` -- requirement contract
- `.gsd/DECISIONS.md` -- architectural decisions
- `.gsd/KNOWLEDGE.md` -- cross-session rules and patterns
- `.gsd/milestones/` -- roadmaps, plans, summaries, research

**What stays local** (gitignored):
- Lock files, metrics, state cache, runtime records, worktrees, activity logs

### Unique Milestone IDs

In team mode, milestones get unique IDs (e.g., `M001-abc123`) to prevent collisions when multiple developers create milestones concurrently.

### Automatic Pull Requests

GSD can create a PR when a milestone completes:

```yaml
git:
  auto_push: true
  auto_pr: true
  pr_target_branch: develop
```

Requires `gh` CLI installed and authenticated.

### Parallel Development

Multiple developers can run auto mode simultaneously on different milestones. Each developer gets their own worktree and works on a unique `milestone/<MID>` branch.

## Parallel Orchestration

Run multiple milestones simultaneously in isolated git worktrees. Behind `parallel.enabled: false` by default.

### Quick Start

```yaml
---
parallel:
  enabled: true
  max_workers: 2
---
```

```
/gsd parallel start    # analyze eligibility, confirm, spawn workers
/gsd parallel status   # monitor progress
/gsd parallel watch    # native TUI overlay (v2.56.0)
/gsd parallel stop     # stop all workers
```

### Architecture

Each worker is a separate `gsd` process with complete isolation:
- Filesystem: git worktree per worker
- Git branch: `milestone/<MID>` per milestone
- State derivation: `GSD_MILESTONE_LOCK` env var restricts visibility
- Context window: separate process per worker
- Metrics: per-worktree `.gsd/metrics.json`
- Crash recovery: per-worktree `.gsd/auto.lock`

Workers and the coordinator communicate through file-based IPC using atomic writes (write-to-temp + rename).

### TUI Monitor Dashboard (v2.54.0)

A real-time TUI dashboard shows the state of all parallel workers, including:
- Per-worker status, current unit, and cost
- Aggregated progress across milestones
- Self-healing worker recovery -- automatically restarts workers that fail due to transient errors

### /gsd parallel watch (v2.56.0)

A native TUI overlay dedicated to worker monitoring. Provides a focused view of parallel execution without interfering with other GSD commands running in the same terminal.

### Self-Healing Worker Recovery (v2.54.0)

The parallel orchestrator detects workers stuck in error states and automatically cleans them up or restarts them. Zombie workers are identified and terminated (v2.53.0).

### Session Lock Contention (v2.56.0)

Resolved multiple bugs around session lock contention in parallel mode. Workers no longer interfere with each other's lock files, and stale locks from crashed workers are cleaned up reliably.

### Eligibility Analysis

Before starting, GSD checks:
1. Milestone is not already complete
2. All `dependsOn` entries have status `complete`
3. File overlap check (warning only, not blocking)

### Configuration

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | Master toggle |
| `max_workers` | number (1-4) | `2` | Maximum concurrent workers |
| `budget_ceiling` | number | none | Aggregate cost ceiling in USD |
| `merge_strategy` | `"per-slice"` or `"per-milestone"` | `"per-milestone"` | When to merge back |
| `auto_merge` | `"auto"`, `"confirm"`, `"manual"` | `"confirm"` | How merge-back is handled |

### Commands

| Command | Description |
|---------|-------------|
| `/gsd parallel start` | Analyze eligibility and start workers |
| `/gsd parallel status` | Show worker state, units, cost |
| `/gsd parallel stop [MID]` | Stop all or specific worker |
| `/gsd parallel pause [MID]` | Pause all or specific worker |
| `/gsd parallel resume [MID]` | Resume paused workers |
| `/gsd parallel merge [MID]` | Merge completed milestones |
| `/gsd parallel watch` | Native TUI overlay for worker monitoring (v2.56.0) |

### Merge Reconciliation

- `.gsd/` state files: auto-resolved (accept milestone branch version)
- Code conflicts: stop and report, manual resolution required

## Cost Management and Budgets

### Cost Tracking

Every unit's metrics are captured automatically: token counts, USD cost, duration, tool calls, message counts. Data is stored in `.gsd/metrics.json`.

View costs via `Ctrl+Alt+G` or `/gsd status`. Aggregations are available by phase, by slice, by model, and project totals.

### Per-Prompt Cost Display (v2.44.0)

Enable the `show_token_cost` preference to see per-prompt token cost in the TUI footer:

```yaml
show_token_cost: true
```

This shows the cost of each individual prompt/response cycle as it completes.

### Session Cost Tracking (v2.35.0)

Session cost is accumulated independently of the message array. This means cost tracking survives context compaction and message pruning -- the running total is always accurate even when older messages are dropped to fit the context window.

### API Request Counter (v2.29.0)

For Copilot and subscription users who are billed by request count rather than tokens, the metrics system tracks API request counts alongside token usage.

### Budget Ceiling

```yaml
---
version: 1
budget_ceiling: 50.00
---
```

### Enforcement Modes

| Mode | Behavior |
|------|----------|
| `warn` | Log a warning, continue |
| `pause` | Pause auto mode, wait for user action (default) |
| `halt` | Stop auto mode entirely |

### Cost Projections

After at least two slices complete, GSD projects remaining cost using per-slice averages.

### Budget Pressure

When approaching the ceiling, the complexity router automatically downgrades model assignments:
- < 50% used: no adjustment
- 50-75% used: standard tasks downgrade to light
- 75-90%: more aggressive downgrading
- > 90%: nearly everything downgrades; only heavy tasks stay at standard

Budget pressure is factored into model routing decisions when `dynamic_routing.budget_pressure` is enabled. Rate-limit errors also attempt model fallback before pausing (v2.53.0).

## Token Optimization Profiles

Set via `token_profile` preference. Introduced in v2.17.0.

### Budget Profile (40-60% reduction)

Uses cheaper models, skips optional phases (milestone research, slice research, roadmap reassessment), compresses dispatch context to the minimum needed.

### Balanced Profile (default, 10-20% reduction)

Keeps important phases, skips slice research, uses standard context compression. The subagent model defaults to Sonnet.

### Quality Profile (no compression)

Every phase runs. Every context artifact is inlined. No shortcuts.

### Context Compression

Each profile maps to an inline level (minimal, standard, full) that controls how much context is pre-loaded into dispatch prompts.

### Context Selection

- `full`: inline entire files (default for `balanced` and `quality`)
- `smart`: use TF-IDF semantic chunking for large files (default for `budget`)

## Dynamic Model Routing (v2.19.0)

Automatically selects cheaper models for simple work and reserves expensive models for complex tasks. Off by default.

```yaml
dynamic_routing:
  enabled: true
```

### Complexity Tiers

| Tier | Typical Work | Default Model Level |
|------|-------------|-------------------|
| Light | Slice completion, UAT, hooks | Haiku-class |
| Standard | Research, planning, execution | Sonnet-class |
| Heavy | Replanning, roadmap reassessment | Opus-class |

Key rule: downgrade-only semantics. The user's configured model is always the ceiling.

### Escalation on Failure

When a task fails at a given tier, the router escalates: Light -> Standard -> Heavy.

### Adaptive Learning

Routing history (`.gsd/routing-history.json`) tracks success/failure per tier per unit type. If failure rate exceeds 20%, future classifications are bumped up. User feedback (`over`/`under`/`ok`) is weighted 2x.

### Cross-Provider Routing

When enabled, the router may select models from other configured providers based on the built-in cost table.

## Troubleshooting and Recovery

### /gsd doctor

Validates `.gsd/` integrity:
- File structure and naming conventions
- Roadmap, slice, and task referential integrity
- Completion state consistency
- Git worktree health and lifecycle checks
- Stale lock files and orphaned runtime records
- Environment health checks (Node version, git, providers)
- Real-time issue visibility across the progress widget and health views

### Common Issues

**Auto mode loops on the same unit**: run `/gsd doctor` to repair state, then resume with `/gsd auto`.

**"Loop detected"**: a unit failed to produce its expected artifact twice. Check the task plan for clarity and refine it manually.

**npm install fails**: ensure Node.js >= 22.0.0. Known issues with `postinstall` on Linux were fixed in v2.3.6+.

**Provider errors during auto mode (v2.26)**: rate limits and server errors auto-resume with backoff. Auth/billing errors require manual intervention. Configure fallback models for resilience.

**Budget ceiling reached**: increase `budget_ceiling` or switch to `budget` token profile.

**Stale lock file**: GSD automatically detects stale locks via PID checks. If automatic recovery fails, delete `.gsd/auto.lock` and `.gsd.lock/` manually.

**Git merge conflicts**: GSD auto-resolves `.gsd/` state conflicts. Code conflicts get an LLM-driven fix-merge attempt; if that fails, manual resolution is needed.

### Recovery Procedures

Reset auto mode state:
```bash
rm .gsd/auto.lock
rm .gsd/completed-units.json
```

Reset routing history:
```bash
rm .gsd/routing-history.json
```

Full state rebuild:
```
/gsd doctor
```

### Getting Help

- GitHub Issues: https://github.com/gsd-build/GSD-2/issues
- Discord: https://discord.gg/gsd
- Dashboard: `Ctrl+Alt+G` or `/gsd status`
- Forensics: `/gsd forensics` for full-access debugging and post-mortem analysis
- Session logs: `.gsd/activity/` contains JSONL session dumps

### Platform-Specific Notes

**macOS/Linux**: the `gsd` command may conflict with the oh-my-zsh git plugin alias (`git svn dcommit`). Either `unalias gsd` in `.zshrc` or use the `gsd-cli` binary name.

**Windows**: MSYS2/Git Bash LSP issues with POSIX paths were fixed in v2.29+ (now uses `where.exe`). EBUSY errors during WXT/extension builds are caused by browser locks on output directories. Detached process groups are disabled on Win32 to prevent EINVAL (v2.52.0).

## Migration from v1

If you have projects with `.planning` directories from the original Get Shit Done (v1):

```
/gsd migrate
```

The migration tool:
- Parses old `PROJECT.md`, `ROADMAP.md`, `REQUIREMENTS.md`, phase directories, plans, summaries, and research
- Maps phases to slices, plans to tasks, milestones to milestones
- Preserves completion state
- Consolidates research files
- Shows a preview before writing
- Optionally runs an agent-driven review for quality assurance

After migrating, verify with `/gsd doctor`.

## Session Management

Resume the most recent session:
```bash
gsd --continue    # or gsd -c
```

Browse and pick from saved sessions:
```bash
gsd sessions
```

## Model Selection

Switch models within a session:
```
/model
```

Configure per-phase models in preferences for fine-grained control (planning model, execution model, completion model, subagent model, simple task model).

## Key Preferences Reference

```yaml
---
version: 1
mode: solo                      # or "team"
token_profile: balanced         # "budget", "balanced", or "quality"
budget_ceiling: 50.00           # USD limit
budget_enforcement: pause       # "warn", "pause", or "halt"
context_selection: full         # "full" or "smart"
show_token_cost: false          # show per-prompt cost in footer (v2.44.0)

models:
  planning: claude-opus-4-6
  execution: claude-sonnet-4-6
  execution_simple: claude-haiku-4-5
  completion: claude-haiku-4-5

git:
  isolation: none               # "worktree", "branch", or "none" (default changed v2.46.0)
  auto_push: false
  push_branches: false
  pre_merge_check: false
  merge_strategy: squash
  main_branch: main
  auto_pr: false
  pr_target_branch: develop

parallel:
  enabled: false
  max_workers: 2
  budget_ceiling: 50.00
  merge_strategy: per-milestone
  auto_merge: confirm

dynamic_routing:
  enabled: false
  escalate_on_failure: true
  budget_pressure: true
  cross_provider: true

phases:
  skip_research: false
---
```
