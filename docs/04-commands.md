# Commands Reference

This document covers every command available in GSD: CLI flags, slash commands, keyboard shortcuts, and headless mode. Verified against source code through v2.58.0.

---

## Quick Reference Tables

### CLI Commands

| Command | Description |
|---------|-------------|
| `gsd` | Start a new interactive session |
| `gsd --continue` / `gsd -c` | Resume the most recent session for the current directory |
| `gsd --model <id>` | Override the default model for this session |
| `gsd --print "msg"` / `gsd -p "msg"` | Single-shot prompt mode (no TUI) |
| `gsd --mode <text\|json\|rpc\|mcp>` | Output mode for non-interactive use |
| `gsd --list-models [search]` | List available models and exit |
| `gsd --version` / `gsd -v` | Print version and exit |
| `gsd --help` / `gsd -h` | Print help and exit |
| `gsd --debug` | Enable structured JSONL diagnostic logging |
| `gsd sessions` | Interactive session picker for current directory |
| `gsd config` | Set up global API keys for search and docs tools |
| `gsd update` | Update GSD to the latest version |
| `gsd headless` | Run auto mode without a TUI |
| `gsd headless --bare` | Minimal resource loading for fast startup |
| `gsd headless --events` | Stream execution events as JSONL |
| `gsd headless --answers` | Pre-supply answers to interactive prompts |
| `gsd headless --resume` | Resume a previous headless session by ID prefix |
| `gsd headless query` | Instant JSON project snapshot (~50ms, no LLM) |
| `gsd headless new-milestone` | Create a new milestone from a context file |
| `gsd --web` / `gsd web` | Start web mode (browser-based UI) |
| `gsd daemon` | Start background daemon process |
| `gsd --mode mcp` | Run GSD as an MCP server over stdin/stdout |

### Slash Command Quick Reference

| Command | Category | Description |
|---------|----------|-------------|
| `/gsd` | Workflow | Step mode — execute one unit, then pause |
| `/gsd next` | Workflow | Explicit step mode (same as `/gsd`) |
| `/gsd auto` | Workflow | Autonomous mode — loop until done |
| `/gsd stop` | Workflow | Stop auto mode gracefully |
| `/gsd pause` | Workflow | Pause auto mode (preserves state) |
| `/gsd discuss` | Workflow | Guided milestone/slice discussion (targets queued milestones since v2.48) |
| `/gsd quick` | Workflow | Execute a quick task without full planning overhead |
| `/gsd rethink` | Workflow | Conversational project reorganization (v2.45) |
| `/gsd fast` | Workflow | Toggle fast service tier for supported models (v2.42) |
| `/gsd rate` | Visibility | Show rate limit status and token profile defaults (v2.36) |
| `/gsd changelog` | Visibility | LLM-summarized release notes (v2.35) |
| `/gsd logs` | Visibility | Browse activity, debug, and metrics logs (v2.29) |
| `/gsd mcp` | Diagnostics | MCP server status and connectivity (v2.45) |
| `/terminal` | Shell | Direct shell execution (v2.51) |
| `/gsd parallel watch` | Orchestration | Native TUI overlay for worker monitoring (v2.56) |
| `/gsd status` | Visibility | Progress dashboard |
| `/gsd visualize` | Visibility | 10-tab interactive workflow visualizer |
| `/gsd queue` | Visibility | Show queued/dispatched units and order |
| `/gsd history` | Visibility | View execution history |
| `/gsd steer <desc>` | Course correction | Hard-steer plan documents |
| `/gsd capture <text>` | Course correction | Fire-and-forget thought capture |
| `/gsd triage` | Course correction | Classify and route pending captures |
| `/gsd skip <unit>` | Course correction | Prevent a unit from auto-mode dispatch |
| `/gsd undo` | Course correction | Revert last completed unit |
| `/gsd park [id]` | Course correction | Park a milestone — skip without deleting |
| `/gsd unpark [id]` | Course correction | Reactivate a parked milestone |
| `/gsd knowledge` | Project knowledge | Add rule, pattern, or lesson to KNOWLEDGE.md |
| `/gsd init` | Setup | Project init wizard |
| `/gsd setup` | Setup | Global setup status and sub-configuration |
| `/gsd mode` | Setup | Switch workflow mode (solo/team) |
| `/gsd prefs` | Setup | Manage preferences |
| `/gsd config` | Setup | Set API keys for external tools |
| `/gsd keys` | Setup | Comprehensive API key manager (v2.29) |
| `/gsd hooks` | Setup | Show hook configuration |
| `/gsd doctor` | Maintenance | Diagnose and repair `.gsd/` state |
| `/gsd forensics` | Maintenance | Full-access GSD debugger with journal and activity log awareness (v2.40+) |
| `/gsd export` | Maintenance | Export milestone/slice results |
| `/gsd cleanup` | Maintenance | Remove merged branches or snapshots |
| `/gsd migrate` | Maintenance | Migrate v1 `.planning/` to `.gsd/` format |
| `/gsd remote` | Maintenance | Control remote auto-mode (Slack/Discord/Telegram) |
| `/gsd inspect` | Maintenance | Show SQLite DB diagnostics |
| `/gsd update` | Maintenance | Update GSD to latest version via npm |
| `/gsd skill-health` | Diagnostics | Skill lifecycle dashboard |
| `/gsd run-hook` | Diagnostics | Manually trigger a specific hook |
| `/gsd parallel` | Orchestration | Parallel milestone orchestration |
| `/worktree` / `/wt` | Git | Git worktree lifecycle |

### Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+Alt+G` | Toggle dashboard overlay |
| `Ctrl+Alt+V` | Toggle voice transcription |
| `Ctrl+Alt+B` | Show background shell processes |
| `Ctrl+V` / `Alt+V` | Paste image from clipboard (screenshot to vision input) |
| `Escape` | Pause auto mode (preserves conversation) |

> In terminals without Kitty keyboard protocol support (macOS Terminal.app, JetBrains IDEs), slash-command fallbacks are shown instead of `Ctrl+Alt` shortcuts. If `Ctrl+V` is intercepted by your terminal (e.g. Warp), use `Alt+V` for clipboard image paste.

---

## CLI Commands

### `gsd` — Start a session

```bash
gsd
```

Starts a new interactive TUI session in the current directory. Runs onboarding if no LLM provider is configured.

### `gsd --continue` / `gsd -c`

```bash
gsd --continue
gsd -c
```

Resumes the most recent saved session for the current directory.

### `gsd sessions`

```bash
gsd sessions
```

Interactive session picker. Lists all saved sessions for the current directory (up to 20), shows date, message count, and first message preview, then prompts for a number to resume.

### `gsd --model <id>`

```bash
gsd --model claude-opus-4-6
gsd --model anthropic/claude-opus-4-6
```

Override the default model for the session. Accepts a bare model ID or `provider/model` format.

### `gsd --print` / `gsd -p`

```bash
gsd --print "Explain this codebase"
gsd -p "What does main.ts do?"
```

Single-shot prompt mode. Sends one message, prints the response, and exits. Does not open a TUI. Useful for scripting.

### `gsd --mode`

```bash
gsd --mode text "Your message"
gsd --mode json "Your message"
gsd --mode rpc
gsd --mode mcp
```

Non-interactive output modes:

| Mode | Description |
|------|-------------|
| `text` | Plain text output to stdout |
| `json` | JSON-structured output to stdout |
| `rpc` | JSON-RPC server over stdin/stdout (used by headless) |
| `mcp` | Model Context Protocol server over stdin/stdout |

### `gsd --list-models [search]`

```bash
gsd --list-models
gsd --list-models claude
gsd --list-models sonnet
```

Lists available models in a table showing provider, model ID, name, context window, max output, and thinking support. Accepts an optional search filter. Exits after printing.

### `gsd config`

```bash
gsd config
```

Opens the global API key setup wizard. Saves keys to `~/.gsd/agent/auth.json`. Applies to all projects. The supported tools are:

| Tool | Environment Variable |
|------|---------------------|
| Tavily Search | `TAVILY_API_KEY` |
| Brave Search | `BRAVE_API_KEY` |
| Context7 Docs | `CONTEXT7_API_KEY` |
| Jina Page Extract | `JINA_API_KEY` |
| Groq Voice | `GROQ_API_KEY` |

Keys set via environment variables take precedence over saved keys. Anthropic models have built-in web search and do not require Brave or Tavily.

### `gsd update`

```bash
gsd update
```

Checks npm for the latest version of `gsd-pi` and installs it. Exits after completing.

### `gsd --web` / `gsd web` (v2.41)

```bash
gsd --web
gsd web
gsd web --host 0.0.0.0 --port 3000 --allowed-origins "http://localhost:3000"
```

Starts GSD in web mode with a browser-based interface. Opens a local web server and launches the UI in your default browser. Supports `--host`, `--port`, and `--allowed-origins` flags for network configuration (v2.42). The web UI includes dark mode, light theme, auth token management, and project root switching.

### `gsd daemon` (v2.57)

```bash
gsd daemon
```

Starts GSD as a background daemon process. The daemon runs persistently and survives terminal/tab close. Useful for long-running auto-mode sessions where you want GSD to continue after disconnecting. Configured via `DaemonConfig` with `control_channel_id` and orchestrator settings.

### `gsd --debug`

```bash
gsd --debug
```

Enables structured JSONL diagnostic logging for troubleshooting dispatch and state issues.

---

## Headless Mode

`gsd headless` runs GSD commands without a TUI. It spawns a child process in RPC mode, auto-responds to interactive prompts, detects completion, and exits with meaningful exit codes.

**Exit codes:** `0` = complete, `1` = error or timeout, `2` = blocked.

```bash
# Run auto mode (default)
gsd headless

# Run a single unit (step mode)
gsd headless next

# Instant JSON snapshot — no LLM, ~50ms
gsd headless query

# With timeout for CI (milliseconds)
gsd headless --timeout 600000 auto

# Force a specific phase
gsd headless dispatch plan

# Create a milestone from a context file and start auto mode
gsd headless new-milestone --context brief.md --auto

# Create a milestone from inline text
gsd headless new-milestone --context-text "Build a REST API with auth"

# Pipe context from stdin
echo "Build a CLI tool" | gsd headless new-milestone --context -
```

Any `/gsd` subcommand works as a positional argument:

```bash
gsd headless status
gsd headless doctor
gsd headless dispatch execute
gsd headless dispatch research
```

### Headless Flags

| Flag | Description |
|------|-------------|
| `--timeout N` | Overall timeout in milliseconds (default: 300000 / 5 min). Disabled for auto-mode (v2.50) |
| `--max-restarts N` | Auto-restart on crash with exponential backoff (default: 3). Set `0` to disable |
| `--json` | Stream all events as JSONL to stdout |
| `--model ID` | Override the model for the headless session |
| `--context <file>` | Context file for `new-milestone` (use `-` for stdin) |
| `--context-text <text>` | Inline context text for `new-milestone` |
| `--auto` | Chain into auto mode after milestone creation |
| `--bare` | Minimal resource loading for fast startup (v2.52) |
| `--events` | Stream execution events as JSONL (v2.57) |
| `--answers` | Pre-supply answers to interactive prompts (v2.57) |
| `--resume <id>` | Resume a previous headless session by ID prefix match (v2.57) |

### `gsd headless query`

Returns a single JSON object with the full project snapshot. No LLM session, no RPC child — instant response (~50ms). Recommended for orchestrators and scripts that need to inspect GSD state.

```bash
gsd headless query | jq '.state.phase'
# "executing"

gsd headless query | jq '.next'
# {"action":"dispatch","unitType":"execute-task","unitId":"M001/S01/T03"}

gsd headless query | jq '.cost.total'
# 4.25
```

**Output schema:**

```json
{
  "state": {
    "phase": "executing",
    "activeMilestone": { "id": "M001", "title": "..." },
    "activeSlice": { "id": "S01", "title": "..." },
    "activeTask": { "id": "T01", "title": "..." },
    "registry": [{ "id": "M001", "status": "active" }],
    "progress": { "milestones": { "done": 0, "total": 2 }, "slices": { "done": 1, "total": 3 } },
    "blockers": []
  },
  "next": {
    "action": "dispatch",
    "unitType": "execute-task",
    "unitId": "M001/S01/T01"
  },
  "cost": {
    "workers": [{ "milestoneId": "M001", "cost": 1.50, "state": "running" }],
    "total": 1.50
  }
}
```

---

## MCP Server Mode

```bash
gsd --mode mcp
```

Runs GSD as a Model Context Protocol server over stdin/stdout. Exposes all GSD tools (read, write, edit, bash, etc.) to external AI clients — Claude Desktop, VS Code Copilot, and any MCP-compatible host. The server registers all tools from the agent session and maps MCP `tools/list` and `tools/call` requests to GSD tool definitions. Runs until the transport closes.

---

## Slash Commands — Workflow

### `/gsd`

```
/gsd
```

Step mode. Executes one unit of work, then pauses. Equivalent to `/gsd next`. Use this to move through the queue one unit at a time with full review between each step.

### `/gsd next`

```
/gsd next
/gsd next --verbose
/gsd next --dry-run
```

Explicit step mode. Executes the next queued unit, then pauses.

| Flag | Description |
|------|-------------|
| `--verbose` | Show detailed step output |
| `--dry-run` | Preview next step without executing |

**`--dry-run` output includes:** next unit type and ID, milestone, current phase, estimated cost (from historical averages), estimated duration, budget spent so far, and remaining budget.

### `/gsd auto`

```
/gsd auto
/gsd auto --verbose
/gsd auto --debug
```

Autonomous mode. Runs all queued units continuously — research, plan, execute, commit, repeat — until done, blocked, or budget ceiling reached. Stops automatically when all milestones are complete.

| Flag | Description |
|------|-------------|
| `--verbose` | Show detailed execution output |
| `--debug` | Enable debug logging for this session |

### `/gsd stop`

```
/gsd stop
```

Stops auto mode gracefully. If auto mode is running in a remote process (different terminal session), sends a stop signal to the remote PID. The session will complete the current unit before halting.

### `/gsd pause`

```
/gsd pause
```

Pauses auto mode while preserving all state. Resume with `/gsd auto`. The `Escape` key also pauses auto mode.

### `/gsd discuss`

```
/gsd discuss
/gsd discuss <milestone-id>
```

Opens a guided milestone/slice discussion. Works alongside auto mode for architectural conversations without disrupting the queue. Since v2.48.0, `/gsd discuss` can target queued (not yet active) milestones, allowing you to discuss upcoming work before it begins.

### `/gsd quick`

```
/gsd quick
/gsd quick <task description>
```

Execute a quick, self-contained task with GSD guarantees (atomic commits, state tracking) but without the full research-plan-execute-complete pipeline. Best for small, well-understood changes.

### `/gsd rethink` (v2.45)

```
/gsd rethink
```

Conversational project reorganization. Starts a guided session that lets you restructure milestones, reorder slices, merge or split work items, and update dependency relationships through dialogue rather than manual file editing.

### `/gsd fast` (v2.42)

```
/gsd fast
/gsd fast on
/gsd fast off
```

Toggle the fast service tier for supported models. When enabled, requests use the provider's fast/priority tier (where available) for lower latency. The service tier icon in the footer indicates when fast mode is active. Works both inside and outside auto mode.

---

## Slash Commands — Visibility

### `/gsd status`

```
/gsd status
```

Opens the progress dashboard overlay. Shows milestones, slices, tasks, cost breakdown, and current phase. Also accessible via `Ctrl+Alt+G`.

### `/gsd visualize`

```
/gsd visualize
```

Opens the 10-tab interactive workflow visualizer:

1. **Progress** — milestone and slice completion
2. **Timeline** — chronological execution history
3. **Dependencies** — dependency graph
4. **Metrics** — token and cost breakdown by phase, model, slice
5. **Health** — skill health dashboard
6. **Agent** — active agent state
7. **Changes** — recent file changes
8. **Knowledge** — KNOWLEDGE.md entries
9. **Captures** — pending captures
10. **Export** — export controls

Requires an interactive terminal. Not available in headless or print mode.

### `/gsd queue`

```
/gsd queue
```

Shows queued and dispatched units with their execution order. Safe to run during auto mode.

### `/gsd history`

```
/gsd history
/gsd history 10
/gsd history 20
/gsd history 50
/gsd history --cost
/gsd history --phase <phase>
/gsd history --model <model>
```

View execution history from `.gsd/metrics.json`.

| Flag/Arg | Description |
|----------|-------------|
| `N` (number) | Show last N entries |
| `--cost` | Show cost breakdown per entry |
| `--phase` | Filter by phase type |
| `--model` | Filter by model used |

### `/gsd changelog` (v2.35)

```
/gsd changelog
```

Displays LLM-summarized release notes for the current GSD version. The changelog is generated from the raw CHANGELOG.md using an LLM to produce a concise, user-friendly summary of what changed.

### `/gsd logs` (v2.29)

```
/gsd logs
/gsd logs activity
/gsd logs debug
/gsd logs metrics
```

Browse session activity logs, debug output, and metrics data. Provides a filterable view into `.gsd/activity/` log files for troubleshooting and auditing.

### `/gsd rate` (v2.36)

```
/gsd rate
```

Shows rate limit status and token profile defaults for the current provider. Wires dead token-profile defaults into a visible display.

---

## Slash Commands — Course Correction

### `/gsd steer`

```
/gsd steer <description of change>
```

Hard-steers plan documents during execution. The override is saved to `.gsd/OVERRIDES.md` and injected into all future task prompts. If auto mode is active, a document rewrite unit runs before the next task to propagate the change across all active plan documents.

**Example:**

```
/gsd steer Use Postgres instead of SQLite
/gsd steer Add rate limiting to all API endpoints
```

### `/gsd capture`

```
/gsd capture "your thought here"
```

Fire-and-forget thought capture. Saves the text to `.gsd/CAPTURES.md` without interrupting the current task. Works during auto mode. Use `/gsd triage` to classify and route accumulated captures.

**Example:**

```
/gsd capture "Consider caching the user lookup in the auth middleware"
```

### `/gsd triage`

```
/gsd triage
```

Manually triggers triage of all pending captures in `.gsd/CAPTURES.md`. Classifies each capture and routes it to the appropriate plan document (active slice, roadmap, backlog, etc.).

### `/gsd skip`

```
/gsd skip <unit-id>
```

Prevents a specific unit from being dispatched in auto mode. Writes the unit key to `.gsd/completed-units.json`.

**Accepted formats:**

| Format | Example | Resolves to |
|--------|---------|-------------|
| Full key | `execute-task/M001/S01/T03` | Used as-is |
| Path | `M001/S01/T03` | `execute-task/M001/S01/T03` |
| Task shorthand | `T03` | `execute-task/<active-mid>/<active-sid>/T03` |
| Slice shorthand | `S02` | `plan-slice/<active-mid>/S02` |

**Example:**

```
/gsd skip T03
/gsd skip M001/S01/T03
/gsd skip execute-task/M001/S01/T03
```

### `/gsd undo`

```
/gsd undo
/gsd undo --force
```

Reverts the last completed unit. Shows a confirmation prompt unless `--force` is passed.

### `/gsd park`

```
/gsd park
/gsd park <milestone-id>
/gsd park <milestone-id> "reason"
```

Parks a milestone — marks it as skipped without deleting it. Parked milestones are excluded from auto-mode dispatch. If no ID is given, parks the current active milestone.

**Example:**

```
/gsd park M002 "Waiting for design approval"
```

### `/gsd unpark`

```
/gsd unpark
/gsd unpark <milestone-id>
```

Reactivates a parked milestone. If no ID is given and only one milestone is parked, it is unparked automatically. If multiple are parked, lists them and asks for an ID.

---

## Slash Commands — Project Knowledge

### `/gsd knowledge`

```
/gsd knowledge rule <description>
/gsd knowledge pattern <description>
/gsd knowledge lesson <description>
```

Adds a persistent entry to `.gsd/KNOWLEDGE.md`. The file is injected into every agent prompt automatically.

| Type | Use for |
|------|---------|
| `rule` | Always/never rules: "Always use TypeScript strict mode" |
| `pattern` | Code patterns to follow: "Use repository pattern for data access" |
| `lesson` | Lessons learned: "Avoid N+1 queries in the user listing endpoint" |

**Example:**

```
/gsd knowledge rule Use real DB for integration tests, not mocks
/gsd knowledge pattern Prefer functional patterns over classes
/gsd knowledge lesson The auth middleware must run before rate limiting
```

---

## Slash Commands — Setup and Configuration

### `/gsd init`

```
/gsd init
```

Project init wizard. Detects the project state, configures settings, and bootstraps the `.gsd/` directory. If the project is already initialized, offers a re-init option.

### `/gsd setup`

```
/gsd setup
/gsd setup llm
/gsd setup search
/gsd setup remote
/gsd setup keys
/gsd setup prefs
```

Shows global setup status (global preferences state, project state, detected language) and provides entry points for sub-configuration.

| Subcommand | Description |
|------------|-------------|
| `llm` | LLM authentication (redirects to `/login`) |
| `search` | Web search provider configuration |
| `remote` | Remote questions (Discord/Slack/Telegram) |
| `keys` | Tool API key manager |
| `prefs` | Global preferences wizard |

### `/gsd mode`

```
/gsd mode
/gsd mode global
/gsd mode project
```

Switches the workflow mode between `solo` and `team`. Coordinates milestone ID format, git commit behavior, and documentation settings.

| Mode | Unique Milestone IDs | Auto Push | Push Branches | Pre-Merge Check |
|------|---------------------|-----------|---------------|-----------------|
| `solo` | No | Yes | No | No |
| `team` | Yes | No | Yes | Yes |

### `/gsd prefs`

```
/gsd prefs
/gsd prefs global
/gsd prefs project
/gsd prefs status
/gsd prefs wizard
/gsd prefs setup
/gsd prefs import-claude
/gsd prefs import-claude global
/gsd prefs import-claude project
```

Manages preferences files interactively.

| Subcommand | Description |
|------------|-------------|
| (bare) | Open the global preferences wizard |
| `global` | Interactive wizard for `~/.gsd/preferences.md` |
| `project` | Interactive wizard for `.gsd/preferences.md` |
| `status` | Show current preference files, merged values, and skill resolution status |
| `wizard` | Alias for `global` |
| `setup` | Alias for `wizard` — creates preferences file if missing |
| `import-claude` | Import Claude marketplace plugins and skills as namespaced GSD components |
| `import-claude global` | Import to global scope |
| `import-claude project` | Import to project scope |

### `/gsd config`

```
/gsd config
```

Opens the tool API key configuration wizard. Shows which keys are configured and which are missing. Saves keys to `~/.gsd/agent/auth.json`.

### `/gsd keys`

```
/gsd keys
/gsd keys list
/gsd keys add
/gsd keys remove
/gsd keys test
/gsd keys rotate
/gsd keys doctor
```

API key manager with full CRUD operations and health checks.

| Subcommand | Description |
|------------|-------------|
| `list` | Show key status dashboard |
| `add` | Add a key for a provider |
| `remove` | Remove a key |
| `test` | Validate key(s) with an API call |
| `rotate` | Replace an existing key |
| `doctor` | Health check all keys |

### `/gsd hooks`

```
/gsd hooks
```

Displays all configured `post_unit_hooks` and `pre_dispatch_hooks` from the active preferences, with their trigger conditions and status.

---

## Slash Commands — Maintenance

### `/gsd doctor`

```
/gsd doctor
/gsd doctor fix
/gsd doctor heal
/gsd doctor audit
/gsd doctor fix <scope>
/gsd doctor heal <scope>
```

Runs health checks on the `.gsd/` state directory. Detects and optionally fixes common state corruption issues.

| Mode | Description |
|------|-------------|
| (bare) | Run checks, show report |
| `fix` | Run checks and auto-fix detected issues |
| `heal` | Run checks, fix what can be fixed, then dispatch remaining issues to the LLM for AI-driven deep healing |
| `audit` | Run checks without fixing — shows all warnings, up to 50 issues |

The `heal` mode dispatches a structured prompt to the LLM with the list of unresolved errors and context from the GSD workflow protocol, enabling the agent to repair complex state problems.

### `/gsd forensics` (upgraded v2.40)

```
/gsd forensics
/gsd forensics <args>
```

Full-access GSD debugger. Upgraded in v2.40.0 from a post-mortem tool to a comprehensive debugger with access to the event journal and activity logs. Inspects execution logs and runs structured root-cause analysis. Use this when auto mode crashes or produces unexpected results. In v2.43.0+, includes opt-in duplicate detection before issue creation. In v2.48.0+, has journal and activity log awareness for richer diagnostics.

### `/gsd export`

```
/gsd export --html
/gsd export --html --all
/gsd export --json
/gsd export --markdown
```

Generates reports of milestone work.

| Flag | Description |
|------|-------------|
| `--html` | Generate self-contained HTML report for the current or most recent milestone |
| `--html --all` | Generate retrospective HTML reports for all milestones |
| `--json` | Export as JSON |
| `--markdown` | Export as Markdown |

Reports are saved to `.gsd/reports/` with a browseable `index.html` that links to all generated snapshots.

### `/gsd cleanup`

```
/gsd cleanup
/gsd cleanup branches
/gsd cleanup snapshots
```

Removes merged branches and old snapshot refs.

| Subcommand | Description |
|------------|-------------|
| (bare) | Run both `branches` and `snapshots` cleanup |
| `branches` | Delete GSD milestone branches (`gsd/*`) that are already merged into the main branch |
| `snapshots` | Prune old `refs/gsd/snapshots/` refs, keeping the 5 most recent per label |

### `/gsd migrate`

```
/gsd migrate
```

Migrates a v1 `.planning/` directory to the current `.gsd/` format. Run this once when upgrading from GSD v1.

### `/gsd remote`

```
/gsd remote
/gsd remote slack
/gsd remote discord
/gsd remote status
/gsd remote disconnect
```

Configures remote question routing so GSD can ask for user input via Slack, Discord, or Telegram during headless auto mode.

| Subcommand | Description |
|------------|-------------|
| (bare) | Show remote questions menu and current status |
| `slack` | Interactive Slack integration setup wizard |
| `discord` | Interactive Discord integration setup wizard |
| `status` | Show current configuration and last prompt status |
| `disconnect` | Remove remote questions configuration |

See the remote questions configuration section for setup details.

### `/gsd inspect`

```
/gsd inspect
```

Shows SQLite database diagnostics — schema, row counts, and recent entries. Useful for debugging state persistence issues.

### `/gsd update`

```
/gsd update
```

Checks npm for a newer version of `gsd-pi` and installs it without leaving the session. Reports the current version and the version it updated to.

```
Current version: v2.28.0
Checking npm registry...
Updated to v2.29.0. Restart GSD to use the new version.
```

If already up to date, reports so and takes no action.

---

## Slash Commands — Diagnostics

### `/gsd skill-health`

```
/gsd skill-health
/gsd skill-health <skill-name>
/gsd skill-health --stale N
/gsd skill-health --declining
```

Skill lifecycle dashboard. Shows usage stats, success rates, token trends, and staleness warnings.

| Invocation | Description |
|------------|-------------|
| (bare) | Overview table: name, uses, success%, tokens, trend, last used |
| `<skill-name>` | Detailed view for a single skill |
| `--stale N` | Show skills unused for N or more days |
| `--declining` | Show only skills flagged for declining performance (success rate below 70% over last 10 uses) |

Flagging thresholds:
- Success rate below 70% over the last 10 uses
- Token usage rising 20% or more compared to the previous window
- Unused beyond the configured `skill_staleness_days` threshold

### `/gsd mcp` (v2.45)

```
/gsd mcp
/gsd mcp status
```

Shows MCP (Model Context Protocol) server status and connectivity. Displays which MCP servers are configured, their connection state, and available tools. Useful for diagnosing MCP integration issues.

### `/gsd run-hook`

```
/gsd run-hook <hook-name> <unit-type> <unit-id>
```

Manually triggers a named post-unit hook outside of auto mode.

**Accepted unit types:**

| Unit Type | Unit ID Format |
|-----------|---------------|
| `execute-task` | `M001/S01/T01` |
| `plan-slice` | `M001/S01` |
| `research-milestone` | `M001` |
| `complete-slice` | `M001/S01` |
| `complete-milestone` | `M001` |

**Examples:**

```
/gsd run-hook code-review execute-task M001/S01/T01
/gsd run-hook lint-check plan-slice M001/S01
```

---

## Slash Commands — Parallel Orchestration

### `/gsd parallel start`

```
/gsd parallel start
```

Analyzes milestone eligibility for parallel execution, confirms candidates, and starts worker processes. Requires `parallel.enabled: true` in preferences.

### `/gsd parallel status`

```
/gsd parallel status
```

Shows all active workers with their state, completed units, and cost.

### `/gsd parallel stop`

```
/gsd parallel stop
/gsd parallel stop <milestone-id>
```

Stops all workers or a specific milestone's worker.

### `/gsd parallel pause`

```
/gsd parallel pause
/gsd parallel pause <milestone-id>
```

Pauses all workers or a specific one.

### `/gsd parallel resume`

```
/gsd parallel resume
/gsd parallel resume <milestone-id>
```

Resumes paused workers.

### `/gsd parallel watch` (v2.56)

```
/gsd parallel watch
```

Native TUI overlay for real-time worker monitoring. Displays a live dashboard showing all parallel workers, their current phase, completed units, cost, and health status. Complements `/gsd parallel status` with a persistent, auto-refreshing view.

### `/gsd parallel merge`

```
/gsd parallel merge
/gsd parallel merge <milestone-id>
```

Merges completed milestones back to main. With no argument, merges all completed milestones at once.

---

## Slash Commands — Advanced

### `/gsd dispatch`

```
/gsd dispatch <phase>
```

Directly dispatches a specific workflow phase without going through the normal sequencing logic. Intended for debugging and recovery from interrupted sessions.

| Phase | Description |
|-------|-------------|
| `research` | Run milestone or slice research |
| `plan` | Run planning phase |
| `execute` | Run execution phase |
| `complete` | Run completion phase |
| `reassess` | Reassess current roadmap progress |
| `uat` | Run user acceptance testing |
| `replan` | Replan the current slice |

### `/gsd new-milestone`

```
/gsd new-milestone
```

Creates a milestone from a headless context file (`.gsd/runtime/headless-context.md`). If no headless context exists, falls back to the interactive smart entry flow. Used internally by `gsd headless new-milestone`.

### `/gsd help`

```
/gsd help
```

Prints a categorized command reference with descriptions for all GSD subcommands directly in the session.

---

## Shell Commands

### `/terminal` (v2.51)

```
/terminal
/terminal <command>
```

Direct shell execution from within a GSD session. Opens an inline shell or executes a single command and returns the output. Useful for quick system commands without leaving the session context.

---

## Git Commands

### `/worktree` / `/wt`

```
/worktree
/wt
```

Git worktree lifecycle management — create, switch, merge, and remove worktrees. GSD uses worktrees to isolate milestone work from the main branch when `git.isolation` is set to `worktree`. Note: the default isolation mode changed from `worktree` to `none` in v2.46.0.

---

## Session Management Commands

These are built-in session commands, not GSD-specific:

| Command | Description |
|---------|-------------|
| `/clear` | Start a new session |
| `/exit` | Graceful shutdown — saves session state before exiting |
| `/kill` | Kill GSD process immediately |
| `/model` | Switch the active model |
| `/login` | Log in to an LLM provider |
| `/thinking` | Toggle thinking level during sessions |
| `/voice` | Toggle real-time speech-to-text (macOS, Linux) |
