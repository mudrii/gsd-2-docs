# Developer Guide

This guide covers extension development, the Pi SDK, the context pipeline, the TUI component system, agent design principles, architecture decisions, the companion apps (VS Code extension and Studio), the daemon/Discord integration, and testing infrastructure.

---

## Pi SDK Overview

Pi is a terminal-native coding agent harness. It sits between the user and an LLM, providing tools for reading, writing, editing, and executing code, while offering a rich terminal UI with session management, branching, and a deep extension system.

**Repository:** [github.com/badlogic/pi-mono](https://github.com/badlogic/pi-mono)
**Package:** `@mariozechner/pi-coding-agent`

### Key Characteristics

- Not a thin API wrapper -- it has a full session system, branching, compaction, and event architecture
- Not locked down -- nearly everything can be replaced, extended, or overridden via TypeScript extensions
- Not tied to one model -- supports 20+ providers with mid-conversation model switching

### The Four Modes of Operation

| Mode | Invocation | Purpose |
|------|-----------|---------|
| **Interactive (TUI)** | `pi` | Full terminal UI with editor, tool rendering, keyboard shortcuts |
| **Print** | `pi -p "prompt"` | One-shot mode, prints response and exits |
| **JSON** | `pi --mode json` | Structured JSON output for scripting |
| **RPC** | `pi --mode rpc` | JSON-RPC over stdin/stdout for embedding in editors and GUIs |

### Architecture

The runtime is composed of these subsystems:

| Subsystem | Role |
|-----------|------|
| **Model Registry** | Tracks all available models across all providers |
| **Auth Storage** | Stores API keys and OAuth tokens |
| **Agent Session** | Main orchestrator -- agent loop, session, tools, and events |
| **Session Manager** | Reads/writes JSONL session files, manages the entry tree, handles branching |
| **Agent Loop** | Core cycle: send messages to LLM, execute tool calls, repeat until LLM stops |
| **Tool Executor** | Runs built-in and custom tools with cancellation support |
| **Compaction Engine** | Summarizes old messages when context gets too large |
| **Event System** | Every action emits events that extensions can observe and modify |
| **Extension Runtime** | Loads/manages extensions, routes events, handles registration |
| **Resource Loader** | Discovers and loads skills, prompts, themes, and context files |
| **Mode Layer** | Handles I/O for the current mode (TUI, RPC, JSON, print) |

---

## Extension Development

Extensions are TypeScript modules that hook into pi's runtime. They are the primary mechanism for customizing pi.

### What Extensions Can Do

- Register custom tools the LLM can call (`pi.registerTool()`)
- Intercept and modify events -- block dangerous tool calls, transform user input, inject context
- Register slash commands (`/mycommand`) for the user
- Render custom UI -- dialogs, selectors, overlays, custom editors
- Persist state across session restarts
- Control how tool calls and messages appear in the TUI
- Modify the system prompt dynamically per-turn
- Manage models and providers -- register custom providers, switch models
- Override built-in tools -- wrap `read`, `bash`, `edit`, `write` with custom logic

### Creating a Minimal Extension

Create a file at `~/.gsd/agent/extensions/my-extension.ts`:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });
}
```

### Running and Testing

```bash
# Quick test (doesn't need to be in extensions dir)
pi -e ./my-extension.ts

# Or place it in the extensions dir and start pi
pi
```

Extensions in auto-discovered locations (`~/.gsd/agent/extensions/` or `.gsd/extensions/`) can be hot-reloaded with `/reload`.

### Extension Lifecycle

```
pi starts
  |
  +-> Extension default function runs
      |-- pi.on("event", handler)      <- Subscribe to events
      |-- pi.registerTool({...})       <- Register tools
      |-- pi.registerCommand(...)      <- Register commands
      +-- pi.registerShortcut(...)     <- Register keyboard shortcuts

  +-> session_start event fires

  User types a prompt
      |-- Extension commands checked (bypass if match)
      |-- input event (can intercept/transform)
      |-- Skill/template expansion
      |-- before_agent_start (inject message, modify system prompt)
      |-- agent_start
      |
      |   Turn loop (repeats while LLM calls tools)
      |     context event (can modify messages sent to LLM)
      |     LLM responds -> may call tools:
      |       tool_call event (can BLOCK)
      |       tool_execution_start/update/end
      |       tool_result event (can MODIFY)
      |
      +-> agent_end
```

### Extension Locations and Discovery

Extensions are discovered from these locations:

1. **Global:** `~/.gsd/agent/extensions/` -- always loaded
2. **Project:** `.gsd/extensions/` -- loaded when in that project directory
3. **CLI flag:** `pi -e ./my-extension.ts` -- loaded explicitly

### Extension Manifest and Registry (v2.30.0+)

Extensions can declare an `extension-manifest.json` in their directory with metadata that the registry uses for enable/disable control:

```json
{
  "id": "my-extension",
  "name": "My Extension",
  "version": "1.0.0",
  "description": "Does something useful",
  "tier": "community",
  "requires": { "platform": "any" },
  "provides": {
    "tools": ["my_tool"],
    "commands": ["/my-command"],
    "hooks": ["tool_call"],
    "shortcuts": ["ctrl+shift+m"]
  },
  "dependencies": {
    "extensions": ["other-extension"],
    "runtime": ["node >= 22"]
  }
}
```

**Registry system:** Persisted at `~/.gsd/extensions/registry.json`. Extensions without manifests always load (backwards compatible). A fresh install has an empty registry -- all extensions are enabled by default. The only way to stop an extension from loading is an explicit `gsd extensions disable <id>`. Core-tier extensions cannot be disabled.

**Topological sort for load order:** After filtering disabled extensions via the registry, extension paths are sorted in topological dependency order using `sortExtensionPaths()`. Extensions declaring `dependencies.extensions` in their manifest are loaded after their dependencies. Circular or missing dependencies emit warnings but do not block loading.

**`provides.hooks` matching:** The `provides.hooks` field in the manifest declares which event types the extension hooks into (e.g., `tool_call`, `input`, `context`). This metadata is surfaced in the `/gsd extensions list` output and used for introspection.

**TypeScript syntax detection in `.js` files (v2.44.0):** The extension loader detects TypeScript syntax in `.js` extension files and suggests renaming to `.ts` for proper compilation support.

**Pi manifest opt-out for discovery:** Directories containing a `package.json` with a `pi` manifest object use it authoritatively. If `pi.extensions` is an array, those entries are resolved as extension entry points. If `pi: {}` is declared with no extensions array, the directory is treated as a library opt-out (e.g., cmux) and no extensions are loaded from it. Only when no `pi` manifest exists does discovery fall back to `index.ts` / `index.js`.

---

## Context Pipeline and Hooks System

The context pipeline is the full journey of a user prompt through every transformation stage to the LLM.

### Pipeline Stages

1. **Extension command check** -- if `/command` matches, handler runs and pipeline stops
2. **`input` event** -- extensions can transform text, intercept entirely, or pass through
3. **Skill/template expansion** -- deterministic text substitution
4. **`before_agent_start` event** (once per user prompt) -- inject messages, modify system prompt
5. **Turn loop** (repeats while LLM calls tools):
   - `context` event -- extensions receive a deep copy of `AgentMessage[]`, can filter/reorder/inject/replace
   - `convertToLlm` -- maps `AgentMessage[]` to `Message[]` (not extensible)
   - LLM call with system prompt + converted messages + tool definitions
   - Tool execution with `tool_call` and `tool_result` events
6. **`agent_end` event**

### Key Timing Distinctions

| Hook | When | Frequency | Can Modify |
|------|------|-----------|-----------|
| `input` | Before expansion | Once per user input | Input text |
| `before_agent_start` | After expansion, before loop | Once per prompt | System prompt + inject messages |
| `context` | Before each LLM call | Every turn | Message array (deep copy) |
| `tool_call` | Before each tool execution | Per tool call | Block execution |
| `tool_result` | After each tool execution | Per tool call | Result content/details |

### Event Categories

**Session events:** `session_start`, `session_before_switch`, `session_switch`, `session_before_fork`, `session_fork`, `session_before_compact`, `session_compact`, `session_before_tree`, `session_tree`, `session_shutdown`

**Agent events:** `before_agent_start`, `agent_start`, `agent_end`, `turn_start`, `turn_end`, `context`, `message_start/update/end`

**Tool events:** `tool_call`, `tool_execution_start`, `tool_execution_update`, `tool_execution_end`, `tool_result`

**Input events:** `input`

**Model events:** `model_select`

**User bash events:** `user_bash`

### Event Chaining

Multiple handlers for the same event chain their results. If handler A transforms input, handler B sees the transformed version. If any `input` handler returns `{ action: "handled" }`, the pipeline stops.

System prompt modifications via `before_agent_start` also chain: Extension A modifies it, Extension B sees the modified version.

---

## UI/TUI Component System

Pi's TUI is a custom terminal rendering system built on two packages:

| Package | Provides |
|---------|---------|
| `@mariozechner/pi-tui` | Core components (`Text`, `Box`, `Container`, `SelectList`), keyboard handling, text utilities |
| `@mariozechner/pi-coding-agent` | Higher-level components (`DynamicBorder`, `BorderedLoader`, `CustomEditor`), theming helpers, code highlighting |

### TUI Layout

```
Terminal Window
  Custom Header (ctx.ui.setHeader)
  Message Area (user messages, assistant responses, tool calls/results, notifications)
  Widgets above editor (ctx.ui.setWidget)
  Editor (can be replaced by ctx.ui.custom() or ctx.ui.setEditorComponent())
  Widgets below editor (ctx.ui.setWidget)
  Footer (ctx.ui.setFooter / ctx.ui.setStatus)
  Overlay (floating, rendered on top of everything via ctx.ui.custom({ overlay }))
```

### Rendering Principles

- Everything renders as arrays of strings (one per line)
- Each line must not exceed the `width` parameter -- this is enforced
- ANSI escape codes are used for styling and do not count toward visible width
- Styles do NOT carry across lines -- the TUI resets SGR at the end of each line
- All state changes require explicit invalidation followed by a render request
- Theme is always passed via callbacks -- never import it directly

### UI Entry Points

- `ctx.ui.select()`, `ctx.ui.confirm()`, `ctx.ui.input()`, `ctx.ui.editor()` -- blocking dialog methods
- `ctx.ui.notify()` -- toast notifications
- `ctx.ui.setStatus()`, `ctx.ui.setWidget()`, `ctx.ui.setHeader()`, `ctx.ui.setFooter()` -- persistent UI
- `ctx.ui.custom()` -- full custom components (temporary or overlay)
- `ctx.ui.setEditorComponent()` -- permanent editor replacement
- `pi.registerMessageRenderer()` -- custom message type rendering
- `renderCall()` / `renderResult()` on tool definitions -- custom tool display

---

## Building Coding Agents -- Key Principles

Research compiled from parallel deep-dive conversations with Claude, Gemini, GPT, and Grok on autonomous AI coding agent architecture.

### Work Decomposition

Use progressive decomposition: Vision -> Capabilities -> Systems/Architecture -> Features -> Tasks.

Core principles:
- **Start with outcomes, not features.** Define "done" before anything else.
- **Vertical slices over horizontal layers.** Build thin end-to-end slices (UI -> API -> DB).
- **The 1-Day Rule.** If a task takes longer than a day, it is not a task -- it is a milestone.
- **Risk-first exploration.** Identify the hardest parts first.
- **Interface-first design.** Define contracts between components before building them.
- **MECE decomposition.** Tasks should be Mutually Exclusive and Collectively Exhaustive.

### Top 10 Tips for a World-Class Agent

1. **The orchestrator is the product, not the model.** Invest 70% of effort in orchestration, 30% in prompt engineering.
2. **Context assembly is a craft.** Profile context like code. Measure which elements correlate with first-attempt success.
3. **Make the feedback loop the fastest thing.** Treat latency like a game engine treats frame rate.
4. **Build first-class error recovery into every layer.** Retry with variation, automatic rollback, structured escalation.
5. **Verify through execution, not self-assessment.** Run the code and observe results.
6. **Return structured, actionable data from every tool.** Remove cognitive load from the model.
7. **Use a DAG, not a flat list.** Explicit inputs, outputs, dependencies, acceptance criteria per task.
8. **Keep the manifest small and always current.** One file, under 1000 tokens, updated automatically.
9. **Build observability from day one.** Log every LLM call, track failure rates, first-attempt success.
10. **Make human touchpoints high-leverage and low-friction.** Present specific questions with context, not walls of text.

---

## Architecture Decision Records

### ADR-001: Branchless Worktree Architecture

**Status:** Proposed (2026-03-15)

**Problem:** The existing architecture used slice branches inside worktrees, causing:
- Planning artifact invisibility (loop detection failures when artifacts are on slice branches but verification checks the milestone branch)
- `.gsd/` state clobbering across branches (shared gitignored directory)
- 582+ lines of merge/conflict/branch-switching code
- Dual isolation modes with parallel implementations

**Decision:** Eliminate slice branches entirely. All work within a milestone worktree commits sequentially on a single branch (`milestone/<MID>`). Track `.gsd/` planning artifacts in git; gitignore only runtime/ephemeral state.

**Architecture:**
```
main ----------------------------------------- main
  |                                              ^
  +-- worktree (milestone/M001)                  |
       commit: feat(M001/S01): research          |
       commit: feat(M001/S01/T01): impl          |
       commit: feat(M001): milestone complete    |
       +-------------- squash merge -------------+
```

**Impact:** Deletes ~770+ lines of merge/branch/conflict code. Simplifies `mergeMilestoneToMain()` and `smartStage()`. Eliminates the `fix-merge` dispatch unit.

### ADR-003: Auto-Mode Pipeline Simplification

**Status:** Proposed (2026-03-18)

**Problem:** The auto-mode pipeline for a 4-slice, 3-task milestone runs 30 sessions under the quality profile, but only 12 are task execution. The remaining 18 are pipeline ceremony (research, completion, reassessment, validation). Each fresh session re-ingests static context at ~208K tokens of overhead per milestone.

**Decision:** Collapse from 13 unit types to 5 in the default path:
- Merge research into planning (research-milestone into plan-milestone, research-slice into plan-slice)
- Fold slice completion into mechanical post-unit processing
- Eliminate reassessment by default (opt-in via preference)
- Replace LLM-driven milestone validation with mechanical verification aggregation
- Replace LLM-driven milestone completion with mechanical summary generation

**Impact:** Reduces a 4-slice milestone from 30 sessions to 16 (47% reduction). Saves ~156-481K tokens per milestone. All eliminated unit types are retained as fallbacks for explicit dispatch.

### ADR-004: Capability-Aware Model Routing

**Status:** Proposed (Revised, 2026-03-26)

**Related:** ADR-003 (pipeline simplification)

**Problem:** The existing dynamic model router is fundamentally complexity-tier and cost based, not task-capability based. It optimizes for task difficulty vs model cost, when the real problem is task requirements vs model strengths. Three structural issues:

1. **Wrong optimization target** -- the router selects the cheapest model in the eligible tier, ignoring that models within the same tier have different strengths (Claude excels at greenfield architecture, Codex at debugging/refactoring, Gemini at long-context synthesis).
2. **Poor behavior with heterogeneous pools** -- different users have different subscriptions. A fixed mapping like "research always uses Gemini" does not generalize across varied provider configurations.
3. **Capability knowledge trapped in user intuition** -- experienced users know which models perform better at coding, debugging, research, or instruction following. GSD has no representation for that knowledge.

**Decision:** Extend dynamic routing from a one-dimensional tier system to a two-dimensional system combining complexity classification ("how hard") with capability scoring ("what kind"), while preserving downgrade-only semantics, budget controls, and user overrideability.

**Design principles:**
- Downgrade-only invariant preserved -- the user's configured model is always the ceiling
- Complexity classification unchanged -- `classifyUnitComplexity()` still determines tier eligibility
- Cost is a constraint, not a score dimension -- budget pressure limits eligibility, not capability profiles
- Requirement vectors are dynamic -- computed from `(unitType, TaskMetadata)`, not from unit type alone

**Revised selection pipeline:**

```
unit dispatch
  -> classifyUnitComplexity(unitType, unitId, basePath, budgetPct)
      [unchanged -- determines tier eligibility and budget filtering]
  -> resolveModelForComplexity(classification, phaseConfig, routingConfig, availableModelIds)
      -> STEP 1: filter to tier-eligible models (downgrade-only from user ceiling)
      -> STEP 2: if capability routing enabled AND >1 eligible model:
          -> computeTaskRequirements(unitType, taskMetadata)
          -> scoreEligibleModels(eligible, taskRequirements)
          -> select highest-scoring model (deterministic tie-break by cost, then ID)
      -> STEP 3: assemble fallback chain
  -> resolveModelId() -> pi.setModel()
```

Each model gains an optional capability profile with numeric scores (0.0-1.0) across dimensions like coding, debugging, research, long-context, and instruction-following. Task requirements are computed dynamically from unit type and `TaskMetadata` (file counts, dependency counts, tags, complexity keywords). The selection pipeline scores eligible models against the requirement vector and picks the best match.

**Relationship to ADR-003:** ADR-003 reduced the number of dispatch units per milestone. ADR-004 makes each remaining unit smarter by routing to the most capable model for that specific kind of work, rather than defaulting to the cheapest in the tier.

---

## VS Code Extension Development

The VS Code extension (`gsd-2`) provides full IDE integration for the GSD-2 coding agent. It was built in a 3-phase rollout across v2.52.0 through v2.58.0.

**Publisher:** FluxLabs
**Version:** 0.3.0
**Engine requirement:** VS Code >= 1.95.0

### Phase 1 (v2.52.0): Foundation

Introduced the core extension infrastructure:
- **Status bar** -- connection status indicator in the VS Code status bar
- **File decorations** -- "G" badge on agent-modified files in the Explorer
- **Bash terminal** -- dedicated terminal for agent shell output
- **Session tree** -- sidebar panel listing all past sessions for the current workspace, with click-to-switch
- **Conversation history** -- full message viewer with tool calls, thinking blocks, search, and fork-from-here
- **Code lens** -- inline actions above functions and classes (Ask GSD, Refactor, Find Bugs, Tests) for TS/JS/Python/Go/Rust

### Phase 2 (v2.53.0): Interactivity

Added workflow and session management features:
- **Activity feed** -- real-time log of every tool the agent executes (Read, Write, Edit, Bash, Grep, Glob) with status icons, duration, and click-to-open
- **Workflow controls** -- one-click buttons (Auto, Next, Quick, Capture) routing through the Chat panel
- **Session forking** -- fork from any previous message to create a new branch of conversation
- **Enhanced code lens** -- expanded language coverage and action reliability

### Phase 3 (v2.58.0): Change Management

Completed the full change review and approval workflow:
- **Change tracker** -- captures file state before modifications for SCM diffs and rollback
- **Checkpoints** -- automatic checkpoints at the start of each agent turn for safe rollback
- **Diagnostics** -- Fix Errors button reads active file diagnostics; Fix All Problems collects workspace-wide errors/warnings; chat auto-includes diagnostics when "fix" or "error" mentioned
- **Git integration** -- Commit Agent Changes, Create Branch, Show Diff commands
- **Line decorations** -- green background on added lines, yellow on modified lines, left border gutter indicator, hover tooltips
- **Permissions** -- approval modes (auto-approve, ask, plan-only) controlling agent autonomy
- **Plan viewer** -- visualization of agent planning state
- **SCM provider** -- dedicated "GSD Agent" section in Source Control with per-file accept/discard, Accept All/Discard All, and native diff editor integration

### Source Files

| File | Role |
|------|------|
| `extension.ts` | Main entry point, activation, command registration |
| `gsd-client.ts` | RPC client -- spawns `gsd --mode rpc`, manages JSON-RPC communication |
| `sidebar.ts` | Sidebar dashboard -- connection status, model, tokens, workflow buttons, settings |
| `chat-participant.ts` | `@gsd` chat participant in VS Code Chat with streaming, file/selection/diagnostic context |
| `activity-feed.ts` | Real-time tool execution log with status icons and click-to-open |
| `session-tree.ts` | Session list panel with click-to-switch and current session highlighting |
| `conversation-history.ts` | Full message viewer with search, tool call display, and fork-from-here |
| `code-lens.ts` | Inline actions (Ask, Refactor, Find Bugs, Tests) above functions/classes |
| `change-tracker.ts` | Captures pre-modification file state for diff computation and rollback |
| `checkpoints.ts` | Automatic per-turn checkpoints for safe rollback |
| `scm-provider.ts` | Source Control integration -- GSD Agent group, per-file accept/discard |
| `file-decorations.ts` | "G" badge on agent-modified files in Explorer |
| `line-decorations.ts` | Green/yellow line backgrounds and gutter indicators on agent-touched lines |
| `diagnostics.ts` | Reads VS Code Problems panel, sends diagnostics to agent |
| `git-integration.ts` | Commit, branch, and diff commands for agent changes |
| `permissions.ts` | Approval mode management (auto-approve / ask / plan-only) |
| `plan-viewer.ts` | Agent planning state visualization |
| `bash-terminal.ts` | Dedicated terminal instance for agent shell output |
| `slash-completion.ts` | Auto-complete for `/gsd` slash commands |

### Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `gsd.binaryPath` | `"gsd"` | Path to the GSD binary |
| `gsd.autoStart` | `false` | Start agent on extension activation |
| `gsd.autoCompaction` | `true` | Automatic context compaction |
| `gsd.codeLens` | `true` | Code lens above functions/classes |
| `gsd.showProgressNotifications` | `false` | Progress notification (off -- Chat shows progress) |
| `gsd.activityFeedMaxItems` | `100` | Max items in Activity feed |
| `gsd.showContextWarning` | `true` | Warn when context exceeds threshold |
| `gsd.contextWarningThreshold` | `80` | Context % that triggers warning |
| `gsd.approvalMode` | `"auto-approve"` | Agent permission mode |

### How It Works

The extension spawns `gsd --mode rpc` and communicates over JSON-RPC via stdin/stdout. Agent events stream in real-time. The change tracker captures file state before modifications for SCM diffs and rollback. UI requests from the agent (questions, confirmations) are handled via VS Code dialogs.

---

## Daemon and Discord Integration (v2.57.0+)

The `packages/daemon` workspace package provides a long-running background service that manages headless GSD sessions and bridges them to Discord channels.

### Architecture

```
DaemonConfig (YAML)
  |
  Daemon
    +-- SessionManager      (manages ManagedSession lifecycle, RpcClient instances)
    +-- DiscordBot           (discord.js Client, auth guard, slash commands)
    +-- EventBridge          (RPC events -> channel messages via formatter + batcher)
    +-- Orchestrator         (LLM-powered control channel agent with 5 tool definitions)
    +-- ChannelManager       (per-project Discord text channels under "GSD Projects" category)
```

### DaemonConfig

Top-level configuration loaded from YAML:

```typescript
interface DaemonConfig {
  discord?: {
    token: string;
    guild_id: string;
    owner_id: string;
    dm_on_blocker?: boolean;
    control_channel_id?: string;
    orchestrator?: { model?: string; max_tokens?: number };
  };
  projects: { scan_roots: string[] };
  log: { file: string; level: LogLevel; max_size_mb: number };
}
```

### DiscordBot

Wraps `discord.js` Client with login/destroy lifecycle and an auth guard. Auth model: single Discord user ID allowlist (`owner_id`). All non-owner interactions are silently ignored; rejections are logged at debug level (userId only, no PII). The `isAuthorized()` function fails closed on empty or missing ownerId.

### Event Bridge

Orchestrates the flow from SessionManager events through the formatter, verbosity filter, and message batcher into Discord channels. Handles:
- Session lifecycle -- Discord channel creation and cleanup
- Event streaming -- format + verbosity filter + batcher
- Blocker resolution -- interactive buttons + text relay
- Conversation relay -- Discord messages forwarded to GSD sessions
- DM backup -- owner gets DM on blocker when `dm_on_blocker` is configured

### Pure-Function Event Formatters

10 exported formatter functions in `event-formatter.ts` mapping RPC event types to Discord embeds with a color palette (green=success, red=error, yellow=blocker, blue=info, grey=tool):

| Function | Maps |
|----------|------|
| `formatToolStart` | Tool execution begin |
| `formatToolEnd` | Tool execution completion |
| `formatMessage` | Agent text messages |
| `formatBlocker` | Blocker events requiring user input |
| `formatCompletion` | Task/milestone completion |
| `formatError` | Session errors |
| `formatCostUpdate` | Cost/token tracking updates |
| `formatSessionStarted` | Session start notification |
| `formatTaskTransition` | Task state changes |
| `formatGenericEvent` | Catch-all for unmapped event types |

The top-level `formatEvent()` dispatches to the appropriate specific formatter based on event type.

### Launchd Integration

`launchd.ts` manages macOS service registration. Generates a `com.gsd.daemon.plist` file and provides `LaunchdStatus` queries (registered, PID, last exit status). Accepts `PlistOptions` for configuring Node.js binary path, script path, config path, working directory, and log paths.

### Channel Manager

`channel-manager.ts` manages per-project Discord text channels under a "GSD Projects" category with archive support. Project directory paths are sanitized into valid Discord channel names (lowercased, non-alphanumeric replaced with hyphens, prefixed with `gsd-`, capped at 100 characters).

### Orchestrator

`orchestrator.ts` is an LLM-powered agent for the `#gsd-control` Discord channel. It receives Discord messages, maintains conversation history, calls the Anthropic messages API with 5 tool definitions (`list_projects`, `start_session`, `get_status`, `stop_session`, `get_session_detail`), and sends the LLM's response back to Discord. Uses standard `messages.create()` tool-use loop with Zod schemas for input validation.

### Source Files

| File | Role |
|------|------|
| `types.ts` | All shared types -- DaemonConfig, ManagedSession, SessionStatus, PendingBlocker, CostAccumulator, ProjectInfo, FormattedEvent |
| `daemon.ts` | Core daemon class -- config + logger + lifecycle, signal handlers, keepalive |
| `discord-bot.ts` | Discord.js client wrapper with auth guard and slash commands |
| `event-bridge.ts` | RPC event -> Discord channel message pipeline |
| `event-formatter.ts` | Pure-function formatters (10 functions) mapping events to embeds |
| `session-manager.ts` | Manages ManagedSession instances and RpcClient processes |
| `channel-manager.ts` | Per-project Discord channel lifecycle and archival |
| `orchestrator.ts` | LLM-powered control channel agent (5 tools) |
| `launchd.ts` | macOS launchd plist generation and status queries |
| `message-batcher.ts` | Batches Discord messages to respect rate limits |
| `verbosity.ts` | Per-channel verbosity levels (default/verbose/quiet) |
| `config.ts` | YAML config loading and validation |
| `logger.ts` | JSON-lines structured logging |
| `cli.ts` | CLI entry point for daemon process |
| `project-scanner.ts` | Scans configured roots for GSD projects |
| `index.ts` | Package barrel export |

---

## Studio (Electron App)

`@gsd/studio` is the desktop GUI shell for the GSD toolkit. It is an Electron application at an early scaffold stage (`version: 0.0.0`, private, not published to npm).

### Current State

- Electron host window (1400x900, dark theme, macOS traffic-light buttons)
- React 19 renderer with Tailwind CSS v4 and Zustand state management
- Design system with token-based colors (deep black palette with warm amber accent), fonts (Inter for sans, JetBrains Mono for mono), and font sizes
- Stub IPC bridge (`StudioBridge`) with `onEvent`, `sendCommand`, `spawn`, and `getStatus` methods -- all currently no-ops
- `getStatus()` always returns `{ connected: false }`

### Architecture

Standard Electron three-process model:
- **Main process** -- `src/main/index.ts`, creates BrowserWindow with `contextIsolation: true` and `nodeIntegration: false`
- **Preload script** -- `src/preload/index.ts`, exposes `window.studio` via `contextBridge`
- **Renderer** -- `src/renderer/src/App.tsx`, React 19 with design system proof-of-concept

### Running

```bash
pnpm run dev      # electron-vite dev server with hot reload
pnpm run build    # production build
pnpm run preview  # preview the production build
```

---

## Testing Infrastructure

### node:test Migration (v2.45.0)

Starting with v2.45.0, all tests were migrated from the custom test harness (`createTestContext`) to Node.js built-in `node:test`. The migration was done in waves:

- `src/tests` a-n and o-z: replaced `try/finally` with `t.after()` for cleanup
- `gsd/tests` a-c, i-n, o-r, s-z: migrated from custom harness to `node:test`
- `packages` tests: replaced `try/finally` with `beforeEach`/`afterEach`

This removed the dependency on custom test utilities and aligned with the Node.js standard test runner.

### esbuild Compilation for Unit Tests (v2.56.0)

In v2.56.0, unit tests were switched to compile via esbuild rather than running TypeScript directly through the Node.js loader. This provides faster test startup and catches compilation errors earlier.

### Integration Test Reclassification (v2.56.0)

Also in v2.56.0, tests were reclassified: tests requiring filesystem setup, network access, or process spawning were moved from the unit test suite to the integration test suite. This fixed issues with `node_modules` symlinks in the unit test environment and made the unit test pass more reliable in CI.

---

## How to Contribute

### Extension Development Workflow

1. Create a `.ts` file in `~/.gsd/agent/extensions/` or `.gsd/extensions/`
2. Export a default function that receives `pi: ExtensionAPI`
3. Register tools, commands, shortcuts, and event handlers in that function
4. Test with `pi -e ./my-extension.ts`
5. Hot-reload with `/reload` during development

### Key Packages

| Package | Purpose |
|---------|---------|
| `@mariozechner/pi-coding-agent` | Core coding agent, extension API, tools, UI components |
| `@mariozechner/pi-tui` | Terminal UI framework, core components, keyboard handling |
| `@mariozechner/pi-ai` | LLM provider abstraction, model registry, `StringEnum` utility |
| `@mariozechner/pi-agent-core` | Agent loop, session management, message types |
| `@gsd-build/rpc-client` | RPC protocol types, client for embedding GSD in editors/GUIs |
| `packages/daemon` | Background daemon service with Discord integration |

### Development Guidelines

- Use `StringEnum` from `@mariozechner/pi-ai` for string enum parameters (not `Type.Union`/`Type.Literal`)
- Truncate tool output to avoid context overflow (50KB / 2000 lines limit)
- Use theme from callback parameters, never import directly
- Store state in tool result `details` for proper branching support
- Session control methods (`waitForIdle()`, `newSession()`, `fork()`) are only available in command handlers, not event handlers
- Check `signal?.aborted` in long-running tool executions
- Use `pi.exec()` for shell commands
