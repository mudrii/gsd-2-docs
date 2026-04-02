# Daemon & Discord Integration

The daemon package (`packages/daemon`) provides a background process for headless GSD orchestration with Discord as the control plane. Introduced in v2.57.0 and refined in v2.58.0, it enables remote session management, real-time event streaming to Discord channels, an LLM-powered orchestrator for natural language control, and macOS launchd integration for auto-start on login.

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       gsd-daemon CLI                         │
│                                                              │
│  cli.ts ── parseArgs ──► loadConfig ──► Daemon.start()       │
│                              │                               │
│            ┌─────────────────▼──────────────────┐            │
│            │            Daemon                   │            │
│            │                                     │            │
│            │  SessionManager  (EventEmitter)      │            │
│            │       │                             │            │
│            │       ├── DiscordBot (discord.js)    │            │
│            │       │       │                     │            │
│            │       │    ChannelManager            │            │
│            │       │                             │            │
│            │       ├── EventBridge                │            │
│            │       │       │                     │            │
│            │       │    MessageBatcher            │            │
│            │       │    VerbosityManager          │            │
│            │       │                             │            │
│            │       └── Orchestrator (LLM agent)   │            │
│            │                                     │            │
│            │  ProjectScanner                      │            │
│            │  Logger (JSON-lines)                 │            │
│            └─────────────────────────────────────┘            │
│                                                              │
│  launchd.ts ── install/uninstall/status (macOS service)      │
└─────────────────────────────────────────────────────────────┘
```

---

## Daemon Configuration

### Config File Location

The config path is resolved in priority order:

1. Explicit CLI argument: `--config <path>`
2. Environment variable: `GSD_DAEMON_CONFIG`
3. Default: `~/.gsd/daemon.yaml`

Tilde (`~`) is expanded automatically in all paths.

### DaemonConfig Schema

The configuration file is YAML. Missing fields fall back to defaults. Invalid log levels fall back to `info`.

```yaml
discord:
  token: "your-discord-bot-token"       # required for Discord integration
  guild_id: "123456789"                 # required — the Discord server ID
  owner_id: "987654321"                 # required — your Discord user ID (auth guard)
  dm_on_blocker: true                   # optional — DM owner when a session blocks
  control_channel_id: "111222333"       # optional — channel for natural language orchestrator
  orchestrator:                         # optional — LLM settings for orchestrator
    model: "claude-haiku-4-5-20251001"  # default model
    max_tokens: 1024                    # default max tokens

projects:
  scan_roots:                           # directories to scan for projects (1 level deep)
    - "~/src"
    - "~/projects"

log:
  file: "~/.gsd/daemon.log"            # default log path
  level: "info"                         # debug | info | warn | error
  max_size_mb: 50                       # max log file size
```

### Environment Variable Overrides

| Variable | Effect |
|----------|--------|
| `DISCORD_BOT_TOKEN` | Overrides `discord.token` in the config file (applied even when no config file exists) |
| `GSD_DAEMON_CONFIG` | Overrides the default config file path |
| `ANTHROPIC_API_KEY` | API key for the orchestrator LLM; also forwarded into launchd environment |

If no config file exists, `loadConfig()` returns defaults and still applies environment variable overrides.

---

## Starting the Daemon

### CLI Entry Point

The daemon runs via `gsd-daemon` (the `cli.ts` entry point). It uses Node's `parseArgs` for argument parsing.

```
Usage: gsd-daemon [options]

Options:
  --config <path>  Path to YAML config file (default: ~/.gsd/daemon.yaml)
  --verbose        Print log entries to stderr in addition to the log file
  --install        Install the launchd LaunchAgent (auto-starts on login)
  --uninstall      Uninstall the launchd LaunchAgent
  --status         Show launchd agent status (registered, PID, exit code)
  --help           Show this help message and exit
```

Short flags: `-c` for `--config`, `-v` for `--verbose`, `-h` for `--help`.

### Startup Sequence

When started in normal mode (no `--install`/`--uninstall`/`--status`):

1. Resolve and load config from YAML
2. Create a `Logger` instance (JSON-lines, append mode)
3. Create `Daemon` and call `start()`
4. `Daemon.start()` creates a `SessionManager`
5. If Discord config is present and valid:
   - Create and login `DiscordBot`
   - Wait for the `ready` event (30s timeout)
   - Register slash commands on the configured guild
   - Create `EventBridge` and wire it to the session manager
   - If `control_channel_id` is set, create `Orchestrator` and listen on `messageCreate`
6. Start health heartbeat timer (every 5 minutes by default)
7. Register SIGTERM/SIGINT handlers for graceful shutdown

Discord integration is optional. If the bot login fails, the daemon logs the error and continues running without Discord.

### Shutdown Sequence

Shutdown is idempotent and ordered:

1. Remove signal handlers
2. Clear health and keepalive timers
3. Stop Orchestrator (clear history, null client)
4. Stop EventBridge (unsubscribe events, destroy batchers, clear mappings)
5. Destroy Discord bot (discord.js `client.destroy()`)
6. Cleanup all active sessions (abort + stop RPC clients)
7. Close logger (flush write stream)
8. `process.exit(0)`

### Health Heartbeat

The daemon logs a health entry every 5 minutes (configurable via constructor) containing:

- Uptime in seconds
- Active session count (running or blocked)
- Discord connection status
- Memory RSS in MB

---

## Discord Bot

### Overview

The `DiscordBot` class wraps a discord.js v14 `Client` with lifecycle management, an auth guard, slash command registration, and integration with the `SessionManager`.

### Auth Model

Authentication uses a single-user allowlist (the `owner_id` from config). The `isAuthorized()` function compares the interaction user's ID against the configured owner. All non-owner interactions are silently ignored; rejections are logged at debug level (user ID only, no PII). Both empty `userId` and empty `ownerId` fail closed (return false).

### Gateway Intents

The bot requests three intents:

- `Guilds` -- guild membership
- `GuildMessages` -- message events
- `MessageContent` -- access to message text (privileged intent)

### Event Listeners

The bot registers these event handlers on the discord.js client:

| Event | Handler |
|-------|---------|
| `ready` (once) | Log bot tag and guild names, register slash commands |
| `interactionCreate` | Route to auth guard then command dispatcher |
| `messageCreate` | Debug logging of all incoming messages |
| `shardError` | Log shard errors (added v2.58.0) |
| `shardDisconnect` | Log shard disconnect with code (added v2.58.0) |
| `shardReconnecting` | Log reconnection attempts (added v2.58.0) |
| `shardResume` | Log shard resume with replayed event count (added v2.58.0) |
| `warn` | Log discord.js warnings (added v2.58.0) |
| `error` | Log discord.js errors (added v2.58.0) |

The six shard/error/warn listeners (R027) provide structured reconnection observability.

### Login Flow

1. Create client with intents
2. Register all event listeners
3. Call `client.login(token)` -- resolves on WebSocket auth
4. Wait for the `ready` event (30s timeout) or fail on `error`/`shardDisconnect`
5. On ready: register slash commands via REST API

The ready wait uses a combined promise that races `ready`, `error`, and `shardDisconnect` against a 30-second timeout, ensuring the bot is fully initialized before `getChannelManager()` is called.

### Slash Commands

Four guild-scoped slash commands are registered via PUT on the Discord REST API:

| Command | Description |
|---------|-------------|
| `/gsd-status` | Show status of all active sessions (ephemeral reply with project name, status, duration, cost) |
| `/gsd-start` | Start a new session -- presents a select menu of scanned projects (max 25), then spawns the GSD process |
| `/gsd-stop` | Stop a running session -- presents a select menu of active sessions, then cancels the selected one |
| `/gsd-verbose` | Set event verbosity level for the current channel (default/verbose/quiet) |

The `/gsd-start` flow:
1. Defer the reply (ephemeral)
2. Scan projects via `scanForProjects()`
3. Present a `StringSelectMenu` (max 25 options, 60s timeout)
4. On selection, defer the component update (sessions take 10-30s to spawn)
5. Call `sessionManager.startSession()` and report success/failure

The `/gsd-stop` flow follows the same pattern with a select menu of active sessions filtered by status (`running`, `blocked`, or `starting`).

---

## Event Bridge

The `EventBridge` is the core wiring layer that routes `SessionManager` events to Discord channels. It handles:

- Session lifecycle -- creating Discord channels on session start, cleanup on completion
- Event streaming -- format + verbosity filter + batch into Discord messages
- Blocker resolution -- interactive buttons and text relay
- Conversation relay -- Discord messages forwarded to GSD sessions
- DM backup -- owner gets DM on blocker when `dm_on_blocker` is configured

### SessionManager Events

The bridge subscribes to five session lifecycle events:

| Event | Action |
|-------|--------|
| `session:started` | Create a project channel via `ChannelManager`, create a `MessageBatcher`, post welcome embed |
| `session:event` | Apply verbosity filter, format event, enqueue to batcher |
| `session:blocked` | Send blocker embed immediately (bypasses batching), create button collector, optionally send DM |
| `session:completed` | Post completion embed, cleanup session mappings and batcher |
| `session:error` | Post error embed, cleanup session |

### Bidirectional Mappings

The bridge maintains four maps:

- `sessionToChannel` (sessionId to channelId)
- `channelToSession` (channelId to sessionId)
- `batchers` (sessionId to MessageBatcher)
- `channels` (sessionId to TextChannel)

### Conversation Relay (Discord to GSD)

When a message arrives in a project channel from the authorized owner:

1. If the session has a pending blocker with `input` or `editor` method, resolve it with the message content
2. Otherwise, relay the message to the GSD session:
   - If session is `running`: use `client.steer()` (injects mid-turn)
   - Otherwise: use `client.prompt()` (starts a new turn)
3. React with a checkmark or envelope emoji to confirm

### Blocker Resolution

For `select` and `confirm` blockers, the bridge creates a button collector on the channel:

- Custom ID format: `blocker:{id}:{method}:{value}`
- Buttons have a 24-hour timeout (`BLOCKER_COLLECTOR_TIMEOUT_MS`)
- Auth guard on button clicks -- only the owner can resolve
- On timeout, the bridge posts a new "Blocker Expired" embed

For `input` and `editor` blockers, the user replies with plain text in the channel.

### DM Backup

When `dm_on_blocker` is enabled, the bridge fetches the owner user and sends a DM with the blocker message and a note to respond in the project channel. DM failures are non-fatal.

---

## Event Formatters

The `event-formatter.ts` module contains 10 pure functions that map RPC event types to Discord-friendly `FormattedEvent` payloads. Each formatter returns a `content` string (plain-text fallback), an optional `EmbedBuilder`, and optional `ActionRowBuilder` components.

### Color Palette

| Color | Hex | Usage |
|-------|-----|-------|
| Green | `0x2ecc71` | Success, completion |
| Red | `0xe74c3c` | Error |
| Yellow | `0xf1c40f` | Blocker (needs attention) |
| Blue | `0x3498db` | Info, session lifecycle |
| Grey | `0x95a5a6` | Tool calls, generic events |

### Formatter Functions

| Function | Event Type(s) | Output |
|----------|--------------|--------|
| `formatToolStart` | `tool_execution_start` | Grey embed with tool name and truncated input |
| `formatToolEnd` | `tool_execution_end` | Grey/red embed with tool name, truncated output, duration |
| `formatMessage` | `message_start`, `message_end`, `message` | Blue embed extracting text from content blocks, message field, or fallback |
| `formatBlocker` | (called directly) | Yellow embed with `@mention`, interactive buttons for select/confirm, text instructions for input/editor |
| `formatCompletion` | `execution_complete` | Green/red embed with status, optional reason and cost/token stats |
| `formatError` | (called directly) | Red embed with error text and session ID footer |
| `formatCostUpdate` | `cost_update` | Blue embed with cumulative cost and token breakdown |
| `formatSessionStarted` | (called directly) | Blue embed with project name |
| `formatTaskTransition` | `task_transition` | Colored embed with task/slice ID and status |
| `formatGenericEvent` | (fallback) | Grey embed with JSON preview of the event payload |

The `formatEvent` dispatch function routes by `event.type`:

| Type | Formatter |
|------|-----------|
| `tool_execution_start` | `formatToolStart` |
| `tool_execution_end` | `formatToolEnd` |
| `message_start`, `message_end`, `message` | `formatMessage` |
| `execution_complete` | `formatCompletion` |
| `cost_update` | `formatCostUpdate` |
| `task_transition` | `formatTaskTransition` |
| Any other | `formatGenericEvent` |

All string fields are truncated to safe lengths: embed descriptions to 2000 chars, field values to 1024 chars, tool names to 60 chars, button labels to 80 chars.

---

## Channel Manager

The `ChannelManager` manages per-project Discord text channels under category channels within a guild.

### Channel Naming

The `sanitizeChannelName()` function converts a project directory path into a valid Discord channel name:

1. Extract basename from the path
2. Lowercase
3. Replace non-alphanumeric (except hyphens) with hyphens
4. Collapse consecutive hyphens
5. Trim leading/trailing hyphens
6. Prefix with `gsd-`
7. Cap at 100 characters (Discord limit)

Empty or whitespace-only inputs produce `gsd-unnamed`.

### Category Structure

| Category | Purpose |
|----------|---------|
| `GSD Projects` | Active project channels (default, created on demand) |
| `GSD Archive` | Completed/archived channels (hidden from `@everyone`) |

Both categories are created on demand and cached after first resolution.

### Channel Lifecycle

- `createProjectChannel(projectDir)` -- creates a text channel under the "GSD Projects" category with a sanitized name derived from the project path
- `archiveChannel(channelId)` -- moves a channel to the "GSD Archive" category and sets a permission overwrite denying `ViewChannel` for the `@everyone` role

---

## Session Manager

The `SessionManager` extends `EventEmitter` and manages `RpcClient` lifecycle for daemon-driven GSD execution.

### Session Lifecycle

```
starting ──► running ──► completed
                │
                ├──► blocked ──► running (after blocker resolved)
                │
                └──► error
                └──► cancelled
```

### Session States

| Status | Meaning |
|--------|---------|
| `starting` | RPC client is being initialized |
| `running` | Agent is actively executing |
| `blocked` | Waiting for user response to a blocker |
| `completed` | Execution finished normally |
| `error` | An error occurred |
| `cancelled` | Stopped by user |

### Starting a Session

1. Validate `projectDir` is non-empty
2. Resolve to absolute path, derive `projectName` from basename
3. Reject if a session already exists for this directory
4. Resolve GSD CLI path (env `GSD_CLI_PATH`, then `which gsd`)
5. Create `RpcClient` with `--mode rpc` and optional `--model`/`--bare` flags
6. Insert session into the map early (keyed by dir) to prevent concurrent starts
7. `client.start()` with 30s timeout (`INIT_TIMEOUT_MS`)
8. `client.init()` for v2 handshake with 30s timeout
9. Wire event handler via `client.onEvent()`
10. Send `/gsd auto` (or custom command) via `client.prompt()`
11. Emit `session:started`

### Event Handling

Events are stored in a ring buffer capped at `MAX_EVENTS` (100). For each event:

- Push to ring buffer, splice oldest if over capacity
- Emit `session:event` for the bridge
- Track cost using cumulative-max pattern (K004): `session.cost.totalCost = Math.max(current, incoming)`
- Detect terminal notifications (`auto-mode stopped`, `step-mode stopped`)
- Detect blocking UI requests (any `extension_ui_request` with a method not in the fire-and-forget set)

Fire-and-forget methods (never treated as blockers): `notify`, `setStatus`, `setWidget`, `setTitle`, `set_editor_text`.

### Cost Tracking

The `CostAccumulator` uses cumulative-max: each field is updated only if the incoming value is greater than the current one. Fields tracked:

- `totalCost`
- `tokens.input`
- `tokens.output`
- `tokens.cacheRead`
- `tokens.cacheWrite`

### API

| Method | Description |
|--------|-------------|
| `startSession(options)` | Start a new GSD session, returns session ID |
| `getSession(sessionId)` | Look up by session ID (linear scan, <10 sessions expected) |
| `getSessionByDir(projectDir)` | Look up by project directory (direct map lookup) |
| `getAllSessions()` | Return all tracked sessions (R035) |
| `resolveBlocker(sessionId, response)` | Send UI response for pending blocker |
| `cancelSession(sessionId)` | Abort + stop the session |
| `getResult(sessionId)` | Build a result object with status, cost, recent events, blocker, error |
| `cleanup()` | Stop all active sessions |

---

## Orchestrator

The `Orchestrator` is an LLM-powered agent that listens in a designated Discord control channel and responds to natural language commands for managing GSD sessions.

### How It Works

1. A message arrives in the control channel from the owner
2. The orchestrator appends it to conversation history
3. Shows a typing indicator
4. Calls the Anthropic messages API with a system prompt, 5 tool definitions, and the history
5. Executes any tool calls (in a loop, max 10 iterations)
6. Sends the final text response to Discord

### System Prompt

The orchestrator identifies as "GSD Control" -- a concise, capable orchestrator. It is instructed to be terse and direct, use bullet points for status, confirm starts/stops, summarize failures plainly, and use Discord markdown.

### Tool Definitions

| Tool | Description |
|------|-------------|
| `list_projects` | Scan configured roots and return names, paths, markers |
| `start_session` | Start a GSD auto-mode session for a project path (optional custom command) |
| `get_status` | Show all sessions with project name, status, duration, cost |
| `stop_session` | Stop a session by ID or fuzzy project name match |
| `get_session_detail` | Detailed session info: cost breakdown, recent events, blocker, error state |

Tool inputs are validated with Zod schemas. `stop_session` supports fuzzy matching: it tries exact session ID first, then case-insensitive substring match on project name or directory.

### Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `model` | `claude-haiku-4-5-20251001` | LLM model for orchestrator requests |
| `max_tokens` | `1024` | Max tokens per response |
| `control_channel_id` | (required) | Discord channel where the orchestrator listens |

### Authentication

The orchestrator resolves an API key in this order:

1. `ANTHROPIC_API_KEY` environment variable
2. GSD's OAuth credentials at `~/.gsd/agent/auth.json` -- reads the access token, refreshes if expired using the OAuth refresh flow

On auth errors (401, missing key), the cached Anthropic client is invalidated so the next request re-resolves credentials.

### Conversation History

- History is capped at `MAX_HISTORY` (30 messages)
- Trimming removes two messages at a time from the front to keep user/assistant pairs aligned
- On errors, a synthetic `[error -- see logs]` assistant message is appended to maintain pairing

---

## Launchd Integration

The `launchd.ts` module provides macOS service management for running the daemon as a LaunchAgent.

### Service Label

`com.gsd.daemon`

### Plist Location

`~/Library/LaunchAgents/com.gsd.daemon.plist`

### Generated Plist Properties

| Property | Value |
|----------|-------|
| `Label` | `com.gsd.daemon` |
| `ProgramArguments` | `[nodePath, scriptPath, "--config", configPath]` |
| `KeepAlive.SuccessfulExit` | `false` (restart on non-zero exit) |
| `RunAtLoad` | `true` (start on login) |
| `EnvironmentVariables.PATH` | Node binary dir + system essentials |
| `EnvironmentVariables.HOME` | User home directory |
| `EnvironmentVariables.ANTHROPIC_API_KEY` | Captured from current environment at install time (if set) |
| `WorkingDirectory` | User home directory (default) |
| `StandardOutPath` | `~/.gsd/daemon-stdout.log` |
| `StandardErrorPath` | `~/.gsd/daemon-stderr.log` |

The NVM-aware PATH includes the directory containing the Node binary, ensuring the daemon can find `node` even outside a shell session.

### CLI Commands

```bash
# Install and start the LaunchAgent
gsd-daemon --install

# Uninstall (unload + remove plist)
gsd-daemon --uninstall

# Check status
gsd-daemon --status
```

### Install Behavior

- Idempotent: unloads existing agent before writing new plist
- Writes plist with `0644` permissions
- Loads via `launchctl load`
- Verifies with `launchctl list`

### Uninstall Behavior

- Graceful: does not throw if already uninstalled
- Unloads via `launchctl unload`, then deletes the plist file

### Status Parsing

The `status()` function handles two `launchctl list` output formats:

1. Tabular: `PID\tStatus\tLabel` (older macOS)
2. JSON-style dict: `"PID" = 1234;` / `"LastExitStatus" = 0;` (newer macOS)

Returns a `LaunchdStatus` with `registered`, `pid`, and `lastExitStatus` fields.

---

## Message Batching

The `MessageBatcher` accumulates `FormattedEvent` payloads and flushes them to Discord, respecting rate limits.

### Configuration

| Option | Default | Description |
|--------|---------|-------------|
| `flushIntervalMs` | `1500` | Timer-based periodic flush interval |
| `maxBatchSize` | `4` | Capacity threshold that triggers an immediate flush |

### Flush Behavior

- **Timer flush**: every 1.5s, the buffer is flushed as a single Discord message
- **Capacity flush**: when the buffer reaches `maxBatchSize`, an immediate flush fires
- **Immediate priority**: `enqueueImmediate()` flushes the pending buffer first, then sends the priority event alone (used for blockers)

### Message Combining

- Single-event sends include the rich embed for formatting
- Multi-event batches send content-only (lines joined with newlines) to avoid duplication between content text and embed descriptions, and to stay under Discord's 10-embed limit
- Component rows come from the last event in the batch that has components

### Error Handling

- Send failures are caught and logged, never crash the batcher
- On failure, the batcher retries once after a 1-second delay
- If the retry also fails, the batch is dropped (prevents infinite retry loops)
- Re-entrant flushes are prevented via a `flushing` guard
- The timer is unref'd so it does not hold the process alive

---

## Logging

The `Logger` class writes structured JSON-lines to a file in append mode.

### Log Entry Format

```json
{
  "ts": "2026-03-28T12:00:00.000Z",
  "level": "info",
  "msg": "daemon started",
  "data": { "log_level": "info", "scan_roots": 2, "discord_configured": true }
}
```

Each entry is a single line containing an ISO-8601 timestamp, level, message, and optional structured data.

### Log Levels

| Level | Numeric | Description |
|-------|---------|-------------|
| `debug` | 0 | Verbose diagnostics (raw messages, auth rejections) |
| `info` | 1 | Normal operations (session started, commands handled, health) |
| `warn` | 2 | Non-fatal issues (DM send failed, command registration failed) |
| `error` | 3 | Errors (session failures, Discord errors) |

Entries below the configured level are silently dropped.

### Verbose Mode

When `--verbose` is passed on the CLI, log entries are also written to stderr in a human-readable format: `[timestamp] LEVEL: message {data}`.

### File Management

- The logger ensures the parent directory exists (creates recursively if needed)
- The write stream is opened in append mode (`flags: 'a'`)
- `close()` returns a promise that resolves when the stream is fully flushed

---

## Verbosity

The `VerbosityManager` controls which RPC event types reach each Discord channel, with three levels.

### Levels

| Level | Events Shown |
|-------|-------------|
| `quiet` | Blockers (`extension_ui_request`), errors, execution completions only |
| `default` | Everything in quiet, plus: tool starts/ends, messages, task transitions, session started |
| `verbose` | Everything -- adds `cost_update`, `state_update`, `status`, `set_status`, `set_widget`, `set_title` |

### Event Classification

| Category | Event Types |
|----------|-------------|
| Always shown | `extension_ui_request`, `execution_complete`, `error`, `session_error` |
| Default level | `tool_execution_start`, `tool_execution_end`, `message_start`, `message_end`, `message`, `task_transition`, `session_started` |
| Verbose only | `cost_update`, `state_update`, `status`, `set_status`, `set_widget`, `set_title` |

Verbosity is set per-channel via the `/gsd-verbose` slash command and stored in a simple `Map<string, VerbosityLevel>`. Channels without an explicit setting default to `default`.

---

## Project Scanner

The `scanForProjects()` function discovers projects in the configured `scan_roots`.

### Behavior

- Reads immediate children of each root (1 level deep, not recursive)
- Skips hidden directories (starting with `.`) and `node_modules`
- Skips missing roots and permission-denied entries gracefully
- Detects markers via a predefined map; directories with no markers are excluded
- Results are sorted alphabetically by name

### Marker Detection

| File/Directory | Project Type |
|----------------|-------------|
| `.git` | `git` |
| `package.json` | `node` |
| `.gsd` | `gsd` |
| `Cargo.toml` | `rust` |
| `pyproject.toml` | `python` |
| `go.mod` | `go` |

The `lastModified` field is the most recent mtime among detected marker files.

---

## Public API (index.ts Exports)

The daemon package exports all major classes, types, and utility functions:

**Classes**: `Daemon`, `SessionManager`, `DiscordBot`, `ChannelManager`, `EventBridge`, `Orchestrator`, `MessageBatcher`, `VerbosityManager`, `Logger`

**Functions**: `loadConfig`, `validateConfig`, `resolveConfigPath`, `scanForProjects`, `isAuthorized`, `validateDiscordConfig`, `sanitizeChannelName`, `buildCommands`, `formatSessionStatus`, `registerGuildCommands`, `shouldShowAtLevel`, `escapeXml`, `generatePlist`, `getPlistPath`, `installLaunchAgent`, `uninstallLaunchAgent`, `launchAgentStatus`

**Event formatters**: `formatEvent`, `formatToolStart`, `formatToolEnd`, `formatMessage`, `formatBlocker`, `formatCompletion`, `formatError`, `formatCostUpdate`, `formatSessionStarted`, `formatTaskTransition`, `formatGenericEvent`

**Types**: `DaemonConfig`, `LogLevel`, `LogEntry`, `SessionStatus`, `ManagedSession`, `PendingBlocker`, `CostAccumulator`, `ProjectInfo`, `ProjectMarker`, `StartSessionOptions`, `FormattedEvent`, `VerbosityLevel`, `LoggerOptions`, `DiscordBotOptions`, `ChannelManagerOptions`, `BridgeClient`, `EventBridgeOptions`, `OrchestratorConfig`, `OrchestratorDeps`, `DiscordMessageLike`, `SendPayload`, `SendFn`, `BatcherLogger`, `BatcherOptions`, `PlistOptions`, `LaunchdStatus`, `RunCommandFn`

**Constants**: `MAX_EVENTS` (100), `INIT_TIMEOUT_MS` (30,000ms)

---

## Version History

| Version | Changes |
|---------|---------|
| v2.57.0 | Created `packages/daemon` workspace. `DaemonConfig`/`LogLevel`/`LogEntry` types. `DiscordBot` with auth guard and lifecycle. 10 pure-function event formatters. Extended config with `control_channel_id` and orchestrator settings. |
| v2.58.0 | Added 6 discord.js shard/error/warn event listeners for reconnection observability. Fixed launchd JSON parsing, login race condition, and interaction handling bugs. |
