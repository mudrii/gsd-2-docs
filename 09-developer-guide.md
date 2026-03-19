# Developer Guide

This guide covers extension development, the Pi SDK, the context pipeline, the TUI component system, agent design principles, architecture decisions, and the companion apps (VS Code extension and Studio).

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

---

## VS Code Extension

The VS Code extension (`gsd-2`) provides IDE integration for the GSD-2 coding agent.

**Publisher:** FluxLabs
**Version:** 0.1.0

### Features

- **Sidebar dashboard** -- connection status, active model, thinking level, token usage, quick action buttons
- **Chat integration** -- `@gsd` chat participant in VS Code Chat for sending messages to the agent
- **15 commands** -- accessible via `Ctrl+Shift+P` (Start Agent, Stop Agent, New Session, Send Message, Abort, Steer, Switch Model, Cycle Model, Set Thinking, Cycle Thinking, Compact, Export HTML, Session Stats, Run Bash, List Commands)
- **Keyboard shortcuts** -- `Cmd+Shift+G Cmd+Shift+N` (New Session), `Cmd+Shift+G Cmd+Shift+M` (Cycle Model), `Cmd+Shift+G Cmd+Shift+T` (Cycle Thinking)

### How It Works

The extension spawns `gsd --mode rpc` in the background and communicates over JSON-RPC via stdin/stdout. All 25 RPC commands are supported, including streaming events for real-time sidebar updates.

### Configuration

| Setting | Default | Description |
|---------|---------|-------------|
| `gsd.binaryPath` | `"gsd"` | Path to the GSD binary if not on PATH |
| `gsd.autoStart` | `false` | Start the agent automatically on activation |
| `gsd.autoCompaction` | `true` | Enable automatic context compaction |

### Requirements

GSD must be installed: `npm install -g gsd-pi`. Node.js >= 20.6.0 and Git are required.

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

### Development Guidelines

- Use `StringEnum` from `@mariozechner/pi-ai` for string enum parameters (not `Type.Union`/`Type.Literal`)
- Truncate tool output to avoid context overflow (50KB / 2000 lines limit)
- Use theme from callback parameters, never import directly
- Store state in tool result `details` for proper branching support
- Session control methods (`waitForIdle()`, `newSession()`, `fork()`) are only available in command handlers, not event handlers
- Check `signal?.aborted` in long-running tool executions
- Use `pi.exec()` for shell commands
