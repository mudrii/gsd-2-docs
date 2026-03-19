# GSD-2 User Guide

Comprehensive reference for GSD-2, the autonomous coding agent built on the Pi SDK. This guide covers released functionality as of v2.33.x.

## Installation and Requirements

### System Requirements

- Node.js >= 22.0.0 (24 LTS recommended)
- Git
- Supported platforms: darwin-arm64, darwin-x64, linux-x64, linux-arm64, win32-x64

### Install via npm

```bash
npm install -g gsd-pi
```

GSD checks for updates once every 24 hours. When a new version is available, an interactive prompt appears at startup with the option to update immediately or skip. Update from within a session with `/gsd update`.

### VS Code Extension

Available on the VS Code Marketplace, published by FluxLabs. Search for "GSD" in VS Code extensions or install from:
https://marketplace.visualstudio.com/items?itemName=FluxLabs.gsd-2

The extension provides:
- `@gsd` chat participant in VS Code Chat
- Sidebar dashboard with connection status, model info, token usage, and quick actions
- Full command palette for start/stop, model switching, and session export

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

### Headless Mode (CI)

`gsd headless auto` runs auto mode without interactive prompts:
- Auto-responds to interactive prompts
- Detects completion and exits with structured codes: 0 complete, 1 error/timeout, 2 blocked
- Auto-restarts on crash with exponential backoff (default 3 attempts)
- Combined with provider error auto-resume, this enables true overnight unattended execution

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

## Git Strategy

### Isolation Modes

GSD supports three isolation modes configured via `git.isolation`:

| Mode | Working Directory | Branch | Best For |
|------|-------------------|--------|----------|
| `worktree` (default) | `.gsd/worktrees/<MID>/` | `milestone/<MID>` | Full file isolation between milestones |
| `branch` | Project root | `milestone/<MID>` | Submodule-heavy repos |
| `none` | Project root | Current branch | Hot-reload workflows |

### Worktree Mode (Default)

Each milestone gets its own git worktree. All execution happens inside the worktree. On completion, the worktree is squash-merged to main as one clean commit. The worktree and branch are then cleaned up.

### Branch Mode

Work happens in the project root on a `milestone/<MID>` branch. No worktree is created. On completion, the branch is merged to main (squash or regular merge, per `merge_strategy`).

### None Mode

Work happens directly on the current branch. No worktree, no milestone branch. GSD still commits sequentially with conventional commit messages, but there is no branch isolation.

### Commit Format

Commits use conventional commit format with scope:

```
feat(S01/T01): core type definitions
feat(S01/T02): markdown parser for plan files
fix(M001/S03): bug fixes and doc corrections
docs(M001/S04): workflow documentation
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

Run `/gsd doctor` to check git health manually.

### Native Git Operations

Since v2.16, GSD uses libgit2 via native bindings for read-heavy operations in the dispatch hot path, eliminating approximately 70 process spawns per dispatch cycle.

## Working in Teams

### Setup

Set `mode: team` in project preferences:

```yaml
# .gsd/preferences.md (project-level, committed to git)
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
- `.gsd/preferences.md` -- project preferences
- `.gsd/PROJECT.md` -- living project description
- `.gsd/REQUIREMENTS.md` -- requirement contract
- `.gsd/DECISIONS.md` -- architectural decisions
- `.gsd/milestones/` -- roadmaps, plans, summaries, research

**What stays local** (gitignored):
- Lock files, metrics, state cache, runtime records, worktrees, activity logs

### Unique Milestone IDs

In team mode, milestones get unique IDs (e.g., `M001-abc123`) to prevent collisions when multiple developers create milestones concurrently.

### commit_docs: false

For teams where only some members use GSD, or when company policy requires a clean repo:

```yaml
git:
  commit_docs: false
```

This adds `.gsd/` to `.gitignore` entirely.

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

### Merge Reconciliation

- `.gsd/` state files: auto-resolved (accept milestone branch version)
- Code conflicts: stop and report, manual resolution required

## Cost Management and Budgets

### Cost Tracking

Every unit's metrics are captured automatically: token counts, USD cost, duration, tool calls, message counts. Data is stored in `.gsd/metrics.json`.

View costs via `Ctrl+Alt+G` or `/gsd status`. Aggregations are available by phase, by slice, by model, and project totals.

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

### Prompt Compression (v2.29.0)

Two strategies:
- `truncate`: drop entire sections at boundaries (pre-v2.29 behavior, default for `quality`)
- `compress`: apply heuristic text compression first, then truncate (default for `budget` and `balanced`)

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
- Git worktree health
- Stale lock files and orphaned runtime records

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
- Discord: https://discord.gg/gsd (approximately 3,200 members)
- Dashboard: `Ctrl+Alt+G` or `/gsd status`
- Forensics: `/gsd forensics` for post-mortem analysis
- Session logs: `.gsd/activity/` contains JSONL session dumps

### Platform-Specific Notes

**macOS/Linux**: the `gsd` command may conflict with the oh-my-zsh git plugin alias (`git svn dcommit`). Either `unalias gsd` in `.zshrc` or use the `gsd-cli` binary name.

**Windows**: MSYS2/Git Bash LSP issues with POSIX paths were fixed in v2.29+ (now uses `where.exe`). EBUSY errors during WXT/extension builds are caused by browser locks on output directories.

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
compression_strategy: compress  # "truncate" or "compress"
context_selection: full         # "full" or "smart"

models:
  planning: claude-opus-4-6
  execution: claude-sonnet-4-6
  execution_simple: claude-haiku-4-5
  completion: claude-haiku-4-5

git:
  isolation: worktree           # "worktree", "branch", or "none"
  auto_push: false
  push_branches: false
  pre_merge_check: false
  merge_strategy: squash
  main_branch: main
  commit_docs: true
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
