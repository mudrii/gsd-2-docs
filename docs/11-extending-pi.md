# Extending Pi

Extensions are TypeScript modules that hook into pi's runtime to extend its behavior. They are the primary mechanism for customizing pi — turning it from a generic coding agent into your coding agent, with your guardrails, your tools, and your workflow.

---

## What Extensions Can Do

- **Register custom tools** the LLM can call (via `pi.registerTool()`)
- **Intercept and modify events** — block dangerous tool calls, transform user input, inject context
- **Register slash commands** (`/mycommand`) for the user
- **Render custom UI** — dialogs, selectors, games, overlays, custom editors
- **Persist state** across session restarts
- **Control how tool calls and messages appear** in the TUI
- **Modify the system prompt** dynamically per-turn
- **Manage models and providers** — register custom providers, switch models
- **Override built-in tools** — wrap `read`, `bash`, `edit`, `write` with custom logic

---

## Architecture and Mental Model

```
┌─────────────────────────────────────────────────────┐
│                    Pi Runtime                        │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Session  │  │  Agent   │  │   Tool Executor  │  │
│  │  Manager  │  │  Loop    │  │                  │  │
│  └────┬─────┘  └────┬─────┘  └────────┬─────────┘  │
│       │              │                 │             │
│       └──────────────┼─────────────────┘             │
│                      │                               │
│              ┌───────▼────────┐                      │
│              │  Event System  │ ◄── All events flow  │
│              └───────┬────────┘     through here     │
│                      │                               │
│         ┌────────────┼────────────┐                  │
│         ▼            ▼            ▼                  │
│    Extension A  Extension B  Extension C             │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**Key concepts:**

- **Extensions are loaded once** when pi starts (or on `/reload`). Your default export function runs, and you subscribe to events and register tools/commands during that function call.
- **Events are the communication mechanism.** Pi emits events at every stage of its lifecycle. Your extension listens and reacts.
- **Tools are the LLM's interface to your extension.** The LLM sees tool descriptions in its system prompt and calls them when appropriate.
- **Commands are the user's interface.** Users type `/mycommand` to invoke your extension directly.
- **State lives in tool result `details`** for proper branching/forking support, or in `pi.appendEntry()` for extension-private state.

---

## Getting Started

### Minimal Extension

Create `~/.gsd/agent/extensions/my-extension.ts`:

```typescript
import type { ExtensionAPI } from "@mariozechner/pi-coding-agent";

export default function (pi: ExtensionAPI) {
  pi.on("session_start", async (_event, ctx) => {
    ctx.ui.notify("Extension loaded!", "info");
  });
}
```

### Testing

```bash
# Quick test (doesn't need to be in extensions dir)
pi -e ./my-extension.ts

# Or just place it in the extensions dir and start pi
pi
```

### Hot Reload

Extensions in auto-discovered locations can be hot-reloaded:

```
/reload
```

---

## Extension Locations and Discovery

### Auto-Discovery Paths

| Location | Scope |
|----------|-------|
| `~/.gsd/agent/extensions/*.ts` | Global (all projects) |
| `~/.gsd/agent/extensions/*/index.ts` | Global (subdirectory) |
| `.gsd/extensions/*.ts` | Project-local |
| `.gsd/extensions/*/index.ts` | Project-local (subdirectory) |

### Additional Paths (via settings.json)

```json
{
  "extensions": [
    "/path/to/local/extension.ts",
    "/path/to/local/extension/dir"
  ],
  "packages": [
    "npm:@foo/bar@1.0.0",
    "git:github.com/user/repo@v1"
  ]
}
```

### Security Warning

Extensions run with your **full system permissions**. They can execute arbitrary code, read and write any file, make network requests. Only install from sources you trust.

---

## Extension Structure Styles

### Single File (simplest)

```
~/.gsd/agent/extensions/
└── my-extension.ts
```

### Directory with index.ts (multi-file)

```
~/.gsd/agent/extensions/
└── my-extension/
    ├── index.ts        # Entry point (must export default function)
    ├── tools.ts
    └── utils.ts
```

### Package with Dependencies (when npm packages are needed)

```
~/.gsd/agent/extensions/
└── my-extension/
    ├── package.json
    ├── package-lock.json
    ├── node_modules/
    └── src/
        └── index.ts
```

```json
{
  "name": "my-extension",
  "dependencies": { "zod": "^3.0.0" },
  "pi": { "extensions": ["./src/index.ts"] }
}
```

Run `npm install` in the extension directory. Imports from `node_modules/` resolve automatically.

### Available Imports

| Package | Purpose |
|---------|---------|
| `@mariozechner/pi-coding-agent` | Extension types (`ExtensionAPI`, `ExtensionContext`, event types, utilities) |
| `@sinclair/typebox` | Schema definitions for tool parameters (`Type.Object`, `Type.String`, etc.) |
| `@mariozechner/pi-ai` | AI utilities (`StringEnum` for Google-compatible enums) |
| `@mariozechner/pi-tui` | TUI components (`Text`, `Box`, `Container`, `SelectList`, etc.) |
| Node.js built-ins | `node:fs`, `node:path`, `node:child_process`, etc. |

---

## The Extension Lifecycle

```
pi starts
  │
  └─► Extension default function runs
      ├── pi.on("event", handler)    ← Subscribe to events
      ├── pi.registerTool({...})     ← Register tools
      ├── pi.registerCommand(...)    ← Register commands
      └── pi.registerShortcut(...)   ← Register keyboard shortcuts

  └─► session_start event fires
      │
      ▼
  User types a prompt ──────────────────────────────────────┐
      │                                                      │
      ├─► Extension commands checked (bypass if match)       │
      ├─► input event (can intercept/transform)              │
      ├─► Skill/template expansion                           │
      ├─► before_agent_start (inject message, modify         │
      │   system prompt)                                     │
      ├─► agent_start                                        │
      │                                                      │
      │   ┌── Turn loop (repeats while LLM calls tools) ──┐  │
      │   │ turn_start                                     │  │
      │   │ context (can modify messages sent to LLM)      │  │
      │   │ LLM responds → may call tools:                 │  │
      │   │   tool_call (can BLOCK)                        │  │
      │   │   tool_execution_start/update/end              │  │
      │   │   tool_result (can MODIFY)                     │  │
      │   │ turn_end                                       │  │
      │   └────────────────────────────────────────────────┘  │
      │                                                      │
      └─► agent_end                                          │
                                                             │
  User types another prompt ◄──────────────────────────────┘
```

**Critical insight:** The event system is your primary mechanism for interacting with pi. Every meaningful thing that happens emits an event, and most events let you modify or block the behavior.

**Important timing:** During the default export function call (step 1), action methods like `sendMessage` and `setActiveTools` will throw. You can only register handlers and tools during the factory function. Use `session_start` for anything that needs runtime access.

---

## Events — The Nervous System

Events are the core of the extension system. They fall into six categories.

### Session Events

| Event | When | Can Return |
|-------|------|------------|
| `session_start` | Session loads | — |
| `session_before_switch` | Before `/new` or `/resume` | `{ cancel: true }` |
| `session_switch` | After session switch | — |
| `session_before_fork` | Before `/fork` | `{ cancel: true }` or `{ skipConversationRestore: true }` |
| `session_fork` | After fork | — |
| `session_before_compact` | Before compaction | `{ cancel: true }` or `{ compaction: {...} }` (custom summary) |
| `session_compact` | After compaction | — |
| `session_before_tree` | Before `/tree` navigation | `{ cancel: true }` or `{ summary: {...} }` |
| `session_tree` | After tree navigation | — |
| `session_shutdown` | On exit (Ctrl+C, Ctrl+D, SIGTERM) | — |

### Agent Events

| Event | When | Can Return |
|-------|------|------------|
| `before_agent_start` | After user prompt, before agent loop | `{ message: {...}, systemPrompt: "..." }` |
| `agent_start` | Agent loop begins | — |
| `agent_end` | Agent loop ends | — |
| `turn_start` | Each LLM turn begins | — |
| `turn_end` | Each LLM turn ends | — |
| `context` | Before each LLM call | `{ messages: [...] }` (modified copy) |
| `message_start/update/end` | Message lifecycle | — |

### Tool Events

| Event | When | Can Return |
|-------|------|------------|
| `tool_call` | Before tool executes | `{ block: true, reason: "..." }` |
| `tool_execution_start` | Tool begins executing | — |
| `tool_execution_update` | Tool sends progress | — |
| `tool_execution_end` | Tool finishes | — |
| `tool_result` | After tool executes | `{ content: [...], details: {...}, isError: bool }` (modify result) |

### Input Events

| Event | When | Can Return |
|-------|------|------------|
| `input` | User input received (before skill/template expansion) | `{ action: "transform", text: "..." }` or `{ action: "handled" }` or `{ action: "continue" }` |

### Model Events

| Event | When | Can Return |
|-------|------|------------|
| `model_select` | Model changes (`/model`, Ctrl+P, restore) | — |

### User Bash Events

| Event | When | Can Return |
|-------|------|------------|
| `user_bash` | User runs `!` or `!!` commands | `{ operations: ... }` or `{ result: {...} }` |

### Resource Events

| Event | When | Can Return |
|-------|------|------------|
| `resources_discover` | At startup and after `/reload` | `{ skillPaths: [...], promptPaths: [...], themePaths: [...] }` |

### Event Handler Signature

```typescript
pi.on("event_name", async (event, ctx: ExtensionContext) => {
  // event — typed payload for this event
  // ctx — access to UI, session, model, and control flow

  // Return undefined for no action, or a typed response object
});
```

### Type Narrowing for Tool Events

```typescript
import { isToolCallEventType, isBashToolResult } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  if (isToolCallEventType("bash", event)) {
    // event.input is typed as { command: string; timeout?: number }
  }
  if (isToolCallEventType("write", event)) {
    // event.input is typed as { path: string; content: string }
  }
});

pi.on("tool_result", async (event, ctx) => {
  if (isBashToolResult(event)) {
    // event.details is typed as BashToolDetails
  }
});
```

---

## ExtensionContext — What You Can Access

Every event handler receives `ctx: ExtensionContext`. This is your window into pi's runtime state.

### ctx.ui — User Interaction

```typescript
// Dialogs (blocking, wait for user response)
const choice = await ctx.ui.select("Pick one:", ["A", "B", "C"]);
const ok = await ctx.ui.confirm("Delete?", "This cannot be undone");
const name = await ctx.ui.input("Name:", "placeholder");
const text = await ctx.ui.editor("Edit:", "prefilled text");

// Non-blocking UI
ctx.ui.notify("Done!", "info");           // Toast notification
ctx.ui.setStatus("my-ext", "Active");     // Footer status
ctx.ui.setWidget("my-id", ["Line 1"]);    // Widget above/below editor
ctx.ui.setTitle("pi - my project");       // Terminal title
ctx.ui.setEditorText("Prefill text");     // Set editor content
ctx.ui.setWorkingMessage("Thinking...");  // Working message during streaming
```

### ctx.hasUI

`false` in print mode (`-p`) and JSON mode. `true` in interactive and RPC mode. Always check before calling dialog methods in non-interactive contexts.

### ctx.cwd

Current working directory (string).

### ctx.sessionManager — Session State

Read-only access to the session:

```typescript
ctx.sessionManager.getEntries()       // All entries in session
ctx.sessionManager.getBranch()        // Current branch entries
ctx.sessionManager.getLeafId()        // Current leaf entry ID
ctx.sessionManager.getSessionFile()   // Path to session JSONL file
ctx.sessionManager.getLabel(entryId)  // Get label on entry
```

**Important:** Always use `getBranch()` for branch-aware state — it returns only entries on the current branch path. `getEntries()` returns all entries including dead branches.

### ctx.modelRegistry / ctx.model

Access to available models and the current model.

### ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()

Control flow helpers for checking agent state.

### ctx.shutdown()

Request graceful shutdown. Deferred until agent is idle. Emits `session_shutdown` before exiting.

### ctx.getContextUsage()

Returns current context token usage. Useful for triggering compaction or showing stats.

```typescript
const usage = ctx.getContextUsage();
if (usage && usage.tokens > 100_000) {
  // Context is getting large
}
```

### ctx.compact(options?)

Trigger compaction programmatically:

```typescript
ctx.compact({
  customInstructions: "Focus on recent changes",
  onComplete: (result) => ctx.ui.notify("Compacted!", "info"),
  onError: (error) => ctx.ui.notify(`Failed: ${error.message}`, "error"),
});
```

### ctx.getSystemPrompt()

Returns the current effective system prompt (including any `before_agent_start` modifications).

---

## ExtensionAPI — What You Can Do

The `pi` object (received in your default export function) is your registration interface. It persists for the lifetime of the extension.

### Core Registration

| Method | Purpose |
|--------|---------|
| `pi.on(event, handler)` | Subscribe to events |
| `pi.registerTool(definition)` | Register a tool the LLM can call |
| `pi.registerCommand(name, options)` | Register a `/command` |
| `pi.registerShortcut(key, options)` | Register a keyboard shortcut |
| `pi.registerFlag(name, options)` | Register a CLI flag |
| `pi.registerMessageRenderer(customType, renderer)` | Custom message rendering |
| `pi.registerProvider(name, config)` | Register/override a model provider |
| `pi.unregisterProvider(name)` | Remove a provider |

### Messaging

| Method | Purpose |
|--------|---------|
| `pi.sendMessage(message, options?)` | Inject a custom message into the session |
| `pi.sendUserMessage(content, options?)` | Send a user message (triggers a turn) |

**`sendMessage` delivery modes:**
- `"steer"` (default) — Interrupts streaming. Delivered after current tool finishes, remaining tools skipped.
- `"followUp"` — Waits for agent to finish. Delivered when agent has no more tool calls.
- `"nextTurn"` — Queued for next user prompt. Does not interrupt.

### State and Session

| Method | Purpose |
|--------|---------|
| `pi.appendEntry(customType, data?)` | Persist extension state (NOT sent to LLM) |
| `pi.setSessionName(name)` | Set display name for session selector |
| `pi.getSessionName()` | Get current session name |
| `pi.setLabel(entryId, label)` | Bookmark an entry for `/tree` navigation |

### Tool Management

| Method | Purpose |
|--------|---------|
| `pi.getActiveTools()` | Get currently active tool names |
| `pi.getAllTools()` | Get all registered tools (name + description) |
| `pi.setActiveTools(names)` | Enable/disable tools at runtime |

### Model Management

| Method | Purpose |
|--------|---------|
| `pi.setModel(model)` | Switch model. Returns `false` if no API key. |
| `pi.getThinkingLevel()` | Get current thinking level |
| `pi.setThinkingLevel(level)` | Set thinking level (`"off"` through `"xhigh"`) |

### Utilities

| Method | Purpose |
|--------|---------|
| `pi.exec(command, args, options?)` | Execute a shell command |
| `pi.events` | Shared event bus for inter-extension communication |
| `pi.getFlag(name)` | Get value of a registered CLI flag |
| `pi.getCommands()` | Get all available slash commands |

---

## Custom Tools — Giving the LLM New Abilities

Tools are the most powerful extension capability. They appear in the LLM's system prompt and the LLM calls them autonomously when appropriate.

### Tool Definition

```typescript
import { Type } from "@sinclair/typebox";
import { StringEnum } from "@mariozechner/pi-ai";

pi.registerTool({
  name: "my_tool",                    // Unique identifier
  label: "My Tool",                   // Display name in TUI
  description: "What this does",      // Shown to LLM in system prompt

  // Optional: customize the one-liner in the system prompt's "Available tools" section
  promptSnippet: "List or add items to the project todo list",

  // Optional: add bullets to the system prompt's "Guidelines" section when tool is active
  promptGuidelines: [
    "Use this tool for todo planning instead of direct file edits."
  ],

  // Parameter schema (MUST use TypeBox)
  parameters: Type.Object({
    action: StringEnum(["list", "add"] as const),
    text: Type.Optional(Type.String()),
  }),

  // The execution function
  async execute(toolCallId, params, signal, onUpdate, ctx) {
    if (signal?.aborted) {
      return { content: [{ type: "text", text: "Cancelled" }] };
    }

    onUpdate?.({
      content: [{ type: "text", text: "Working..." }],
      details: { progress: 50 },
    });

    const result = await doSomething(params);

    return {
      content: [{ type: "text", text: "Done" }],  // Sent to LLM as context
      details: { data: result },                   // For rendering & state reconstruction
    };
  },
});
```

### Critical: Use StringEnum

For string enum parameters, you **must** use `StringEnum` from `@mariozechner/pi-ai`. `Type.Union([Type.Literal("a"), Type.Literal("b")])` does NOT work with Google's API.

```typescript
import { StringEnum } from "@mariozechner/pi-ai";

// Correct
action: StringEnum(["list", "add", "remove"] as const)

// Broken with Google
action: Type.Union([Type.Literal("list"), Type.Literal("add")])
```

### Dynamic Tool Registration

Tools can be registered at any time — during load, in `session_start`, in command handlers, etc. New tools are available immediately without `/reload`.

```typescript
pi.on("session_start", async (_event, ctx) => {
  pi.registerTool({ name: "dynamic_tool", /* ... */ });
});
```

### Output Truncation

Tools MUST truncate output to avoid overwhelming the LLM context. The built-in limit is 50KB / 2000 lines (whichever comes first).

```typescript
import {
  truncateHead,
  DEFAULT_MAX_BYTES, DEFAULT_MAX_LINES,
} from "@mariozechner/pi-coding-agent";

async execute(toolCallId, params, signal, onUpdate, ctx) {
  const output = await runCommand();
  const truncation = truncateHead(output, {
    maxLines: DEFAULT_MAX_LINES,
    maxBytes: DEFAULT_MAX_BYTES,
  });

  let result = truncation.content;
  if (truncation.truncated) {
    result += `\n\n[Output truncated: ${truncation.outputLines}/${truncation.totalLines} lines]`;
  }
  return { content: [{ type: "text", text: result }] };
}
```

### Overriding Built-in Tools

Register a tool with the same name as a built-in (`read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`) to override it. Your implementation **must match the exact result shape** including the `details` type.

```bash
# Start with no built-in tools, only your extensions
pi --no-tools -e ./my-extension.ts
```

### Remote Execution via Pluggable Operations

Built-in tools support pluggable operations for SSH, containers, etc.:

```typescript
import { createBashTool } from "@mariozechner/pi-coding-agent";

const remoteBash = createBashTool(cwd, {
  operations: { execute: (cmd) => sshExec(remote, cmd) }
});

// spawnHook for lighter customization (e.g., environment setup)
const bashTool = createBashTool(cwd, {
  spawnHook: ({ command, cwd, env }) => ({
    command: `source ~/.profile\n${command}`,
    cwd: `/mnt/sandbox${cwd}`,
    env: { ...env, CI: "1" },
  }),
});
```

Operations interfaces available: `ReadOperations`, `WriteOperations`, `EditOperations`, `BashOperations`, `LsOperations`, `GrepOperations`, `FindOperations`.

---

## Custom Commands — User-Facing Actions

Commands let users invoke your extension directly via `/mycommand`.

```typescript
pi.registerCommand("deploy", {
  description: "Deploy to an environment",

  // Optional: argument auto-completion
  getArgumentCompletions: (prefix: string) => {
    const envs = ["dev", "staging", "prod"];
    return envs
      .filter(e => e.startsWith(prefix))
      .map(e => ({ value: e, label: e }));
  },

  handler: async (args, ctx) => {
    // args = everything after "/deploy "
    // ctx = ExtensionCommandContext (has extra session control methods)

    await ctx.waitForIdle();  // Wait for agent to finish
    ctx.ui.notify(`Deploying to ${args}`, "info");
  },
});
```

### ExtensionCommandContext

Command handlers get `ExtensionCommandContext` which extends `ExtensionContext` with additional methods:

| Method | Purpose |
|--------|---------|
| `ctx.waitForIdle()` | Wait for agent to finish streaming |
| `ctx.newSession(options?)` | Create a new session |
| `ctx.fork(entryId)` | Fork from an entry |
| `ctx.navigateTree(targetId, options?)` | Navigate the session tree |
| `ctx.reload()` | Hot-reload extensions, skills, prompts, themes |

These methods are **only available in commands, not in event handlers**, because they would deadlock there.

---

## State Management in Extensions

### Extension-Private State (Not Visible to LLM)

Use `pi.appendEntry()` to persist data that should survive session resume but never go to the LLM:

```typescript
// Write state
pi.appendEntry("my-extension-state", { count: 42, items: ["a", "b"] });

// Restore on session start
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "custom" && entry.customType === "my-extension-state") {
      myState = entry.data;
    }
  }
});
```

### LLM-Visible State via Tool Results

Store state that the LLM should be aware of in tool result `details`. This participates properly in branching and forking:

```typescript
pi.registerTool({
  name: "learn_fact",
  // ...
  async execute(toolCallId, params) {
    facts.push(params.fact);
    return {
      content: [{ type: "text", text: `Learned: ${params.fact}` }],
      details: { facts: [...facts] }, // snapshot for branching support
    };
  },
});

// Restore during session_start
pi.on("session_start", async (_event, ctx) => {
  facts = [];
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "message" && entry.message.role === "toolResult") {
      if (entry.message.toolName === "learn_fact") {
        facts = entry.message.details?.facts ?? [];
      }
    }
  }
});
```

### Injecting State into Each Prompt

Combine `session_start` restoration with `before_agent_start` injection to keep the LLM informed:

```typescript
pi.on("before_agent_start", async () => {
  if (facts.length > 0) {
    return {
      message: {
        customType: "project-facts",
        content: `Known project facts:\n${facts.map(f => `- ${f}`).join("\n")}`,
        display: false,
      },
    };
  }
});
```

---

## Inter-Extension Communication

### The Shared Event Bus

Every extension receives the same `pi.events` instance — a simple typed pub/sub bus:

```typescript
// Extension A emits
pi.events.emit("mode-change", { mode: "plan" });

// Extension B listens
const unsub = pi.events.on("mode-change", (data) => {
  const { mode } = data as { mode: string };
  // React to mode change
});
```

**Characteristics:**
- `data` is typed as `unknown` — cast at the consumer
- No replay, no persistence — emit before subscribe and the event is lost
- Errors in handlers log but do not crash the session
- The bus is cleared on `/reload` — all subscriptions reset
- Shared across ALL extensions in the session
- Use descriptive channel names to avoid collisions (`"myext:event"` not `"update"`)

### Mode Manager Pattern

One extension acts as the mode authority, others react:

```typescript
// mode-manager.ts
export default function (pi: ExtensionAPI) {
  let currentMode: "plan" | "execute" | "review" = "execute";

  pi.registerCommand("mode", {
    handler: async (args, ctx) => {
      currentMode = args.trim() as typeof currentMode;
      pi.events.emit("mode:changed", { mode: currentMode });
      ctx.ui.notify(`Mode: ${currentMode}`);
    },
  });
}

// tool-guard.ts
export default function (pi: ExtensionAPI) {
  let currentMode = "execute";

  pi.events.on("mode:changed", (data) => {
    currentMode = (data as { mode: string }).mode;
  });

  pi.on("tool_call", async (event) => {
    if (currentMode === "plan" && ["edit", "write"].includes(event.toolName)) {
      return { block: true, reason: "Plan mode: write operations disabled" };
    }
  });
}
```

---

## Packaging and Distribution

### Creating a Pi Package

Add a `pi` manifest to `package.json`:

```json
{
  "name": "my-pi-package",
  "keywords": ["pi-package"],
  "pi": {
    "extensions": ["./extensions"],
    "skills": ["./skills"],
    "prompts": ["./prompts"],
    "themes": ["./themes"]
  }
}
```

### Installing Packages

```bash
pi install npm:@foo/bar@1.0.0
pi install git:github.com/user/repo@v1
pi install ./local/path

# Try without installing:
pi -e npm:@foo/bar
```

### Convention Directories (no manifest needed)

If no `pi` manifest exists, pi auto-discovers:
- `extensions/` → `.ts` and `.js` files
- `skills/` → `SKILL.md` folders
- `prompts/` → `.md` files
- `themes/` → `.json` files

### Dependencies

List `@mariozechner/pi-ai`, `@mariozechner/pi-coding-agent`, `@mariozechner/pi-tui`, and `@sinclair/typebox` in `peerDependencies` with `"*"` — they are bundled by pi. Other npm dependencies go in `dependencies`. Pi runs `npm install` on package installation.

---

## Key Rules and Gotchas

### Must-Follow Rules

1. **Use `StringEnum` for string enums** — `Type.Union`/`Type.Literal` breaks Google's API.
2. **Truncate tool output** — Large output causes context overflow, compaction failures, and degraded performance.
3. **Use theme from callback** — Don't import theme directly. Use the `theme` parameter from `ctx.ui.custom()` or render functions.
4. **Type the DynamicBorder color param** — Write `(s: string) => theme.fg("accent", s)`.
5. **Call `tui.requestRender()` after state changes** in `handleInput`.
6. **Return `{ render, invalidate, handleInput }`** from custom components.
7. **Lines must not exceed `width`** in `render()` — use `truncateToWidth()`.
8. **Session control methods only in commands** — `waitForIdle()`, `newSession()`, `fork()`, `navigateTree()`, `reload()` will deadlock in event handlers.
9. **Strip leading `@` from path arguments** in custom tools — some models add it.
10. **Store state in tool result `details`** for proper branching support.

### Common Patterns

- **Rebuild on `invalidate()`** when your component pre-bakes theme colors
- **Check `signal?.aborted`** in long-running tool executions
- **Use `pi.exec()` instead of spawning processes directly** for shell commands
- **Overlay components are disposed when closed** — create fresh instances each time
- **Treat `ctx.reload()` as terminal** — code after it runs from the pre-reload version
- **Use `getBranch()` not `getEntries()`** for branch-aware state reconstruction

### What the LLM Never Sees

- `pi.appendEntry()` data — stored in session file but never included in message array
- `details` on custom messages — stripped by `convertToLlm`
- `details` on tool results — stripped before LLM (only `content` reaches it)
- `!!` bash execution output — explicitly excluded from context
- Tool definitions not in the active set — if a tool is registered but not in `getActiveTools()`, the LLM does not know it exists
- `promptSnippet` and `promptGuidelines` from inactive tools — only active tools contribute to the system prompt
