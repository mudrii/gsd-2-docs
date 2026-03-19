# API and Internals

A detailed reference for the Pi extension API surface: `ExtensionContext`, `ExtensionAPI`, custom tools, custom commands, state management, system prompt modification, compaction, model management, error handling, and key rules.

---

## ExtensionContext (ctx)

Every event handler receives `ctx: ExtensionContext`. This is the read-only window into pi's runtime state.

### ctx.ui -- User Interaction

```typescript
// Blocking dialogs (wait for user response)
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

Returns `false` in print mode (`-p`) and JSON mode. Returns `true` in interactive and RPC mode. Always check before calling dialog methods.

### ctx.cwd

Current working directory (string).

### ctx.sessionManager -- Session State

```typescript
ctx.sessionManager.getEntries()       // All entries in session
ctx.sessionManager.getBranch()        // Current branch entries
ctx.sessionManager.getLeafId()        // Current leaf entry ID
ctx.sessionManager.getSessionFile()   // Path to session JSONL file
ctx.sessionManager.getLabel(entryId)  // Get label on entry
```

### ctx.modelRegistry / ctx.model

Access to the model registry (all available models across providers) and the currently active model.

### ctx.isIdle() / ctx.abort() / ctx.hasPendingMessages()

Control flow helpers. `isIdle()` checks whether the agent is waiting for input. `abort()` interrupts the current operation. `hasPendingMessages()` checks for queued messages.

### ctx.shutdown()

Request graceful shutdown. Deferred until agent is idle. Emits `session_shutdown` before exiting.

### ctx.getContextUsage()

Returns current context token usage:

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

## ExtensionAPI (pi)

The `pi` object received in the default export function is the registration interface. It persists for the lifetime of the extension.

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
- `"steer"` (default) -- interrupts streaming. Delivered after current tool finishes, remaining tools skipped.
- `"followUp"` -- waits for agent to finish. Delivered when agent has no more tool calls.
- `"nextTurn"` -- queued for next user prompt. Does not interrupt.

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
| `pi.exec(command, args, options?)` | Run a shell command |
| `pi.events` | Shared event bus for inter-extension communication |
| `pi.getFlag(name)` | Get value of a registered CLI flag |
| `pi.getCommands()` | Get all available slash commands |

### ExtensionCommandContext (commands only)

Command handlers receive `ExtensionCommandContext`, which extends `ExtensionContext` with session control methods. These methods are only available in commands -- using them in event handlers would cause deadlocks.

| Method | Purpose |
|--------|---------|
| `ctx.waitForIdle()` | Wait for agent to finish streaming |
| `ctx.newSession(options?)` | Create a new session |
| `ctx.fork(entryId)` | Fork from an entry |
| `ctx.navigateTree(targetId, options?)` | Navigate the session tree |
| `ctx.reload()` | Hot-reload extensions, skills, prompts, themes |

---

## Custom Tools -- Giving the LLM New Abilities

Tools are the most powerful extension capability. They appear in the LLM's system prompt and the LLM calls them autonomously.

### Tool Definition

```typescript
import { Type } from "@sinclair/typebox";
import { StringEnum } from "@mariozechner/pi-ai";

pi.registerTool({
  name: "my_tool",                    // Unique identifier
  label: "My Tool",                   // Display name in TUI
  description: "What this does",      // Shown to LLM in system prompt

  // Optional: customize the one-liner in "Available tools" section
  promptSnippet: "List or add items to the project todo list",

  // Optional: add bullets to the "Guidelines" section when tool is active
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

    // Stream progress updates to the UI
    onUpdate?.({
      content: [{ type: "text", text: "Working..." }],
      details: { progress: 50 },
    });

    const result = await doSomething(params);

    return {
      content: [{ type: "text", text: "Done" }],  // Sent to LLM as context
      details: { data: result },                   // For rendering and state reconstruction
    };
  },

  // Optional: Custom TUI rendering
  renderCall(args, theme) { ... },
  renderResult(result, options, theme) { ... },
});
```

### StringEnum Requirement

For string enum parameters, you must use `StringEnum` from `@mariozechner/pi-ai`. `Type.Union([Type.Literal("a"), Type.Literal("b")])` does NOT work with Google's API.

```typescript
import { StringEnum } from "@mariozechner/pi-ai";

// Correct
action: StringEnum(["list", "add", "remove"] as const)

// Broken with Google
action: Type.Union([Type.Literal("list"), Type.Literal("add")])
```

### Dynamic Tool Registration

Tools can be registered at any time -- during load, in `session_start`, in command handlers. New tools are available immediately without `/reload`.

### Output Truncation

Tools MUST truncate output to avoid overwhelming the LLM context. The built-in limit is 50KB / 2000 lines (whichever is reached first).

```typescript
import {
  truncateHead, truncateTail, formatSize,
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

Register a tool with the same name as a built-in (`read`, `bash`, `edit`, `write`, `grep`, `find`, `ls`) to override it. Your implementation must match the exact result shape including the `details` type.

```bash
# Start with no built-in tools, only your extensions
pi --no-tools -e ./my-extension.ts
```

### Remote Execution via Pluggable Operations

Built-in tools support pluggable operations for SSH, containers, and other remote scenarios:

```typescript
import { createReadTool, createBashTool } from "@mariozechner/pi-coding-agent";

const remoteBash = createBashTool(cwd, {
  operations: { execute: (cmd) => sshExec(remote, cmd) }
});
```

The bash tool also supports a `spawnHook` for modifying the command, working directory, or environment before running.

**Operations interfaces:** `ReadOperations`, `WriteOperations`, `EditOperations`, `BashOperations`, `LsOperations`, `GrepOperations`, `FindOperations`

---

## Custom Commands -- User-Facing Actions

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
    // ctx = ExtensionCommandContext (has session control methods)
    await ctx.waitForIdle();
    ctx.ui.notify(`Deploying to ${args}`, "info");
  },
});
```

Command handlers receive `ExtensionCommandContext` which extends `ExtensionContext` with session control methods: `waitForIdle()`, `newSession()`, `fork()`, `navigateTree()`, `reload()`.

These methods are only available in commands. They will deadlock if called from event handlers.

---

## State Management and Persistence

### Pattern: State in Tool Result Details

The recommended approach for stateful tools. State lives in `details` so it works correctly with branching and forking.

```typescript
export default function (pi: ExtensionAPI) {
  let items: string[] = [];

  // Reconstruct from session on load
  pi.on("session_start", async (_event, ctx) => {
    items = [];
    for (const entry of ctx.sessionManager.getBranch()) {
      if (entry.type === "message" && entry.message.role === "toolResult") {
        if (entry.message.toolName === "my_tool") {
          items = entry.message.details?.items ?? [];
        }
      }
    }
  });

  pi.registerTool({
    name: "my_tool",
    // ...
    async execute(toolCallId, params, signal, onUpdate, ctx) {
      items.push(params.text);
      return {
        content: [{ type: "text", text: "Added" }],
        details: { items: [...items] },  // Snapshot state here
      };
    },
  });
}
```

### Pattern: Extension-Private State (appendEntry)

For state that does not participate in LLM context but needs to survive restarts:

```typescript
pi.appendEntry("my-state", { count: 42, lastRun: Date.now() });

// Restore on reload
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      const data = entry.data;  // { count: 42, lastRun: ... }
    }
  }
});
```

---

## System Prompt Modification

### Per-Turn Modification (before_agent_start)

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  return {
    // Inject a persistent message (stored in session, visible to LLM)
    message: {
      customType: "my-extension",
      content: "Additional context for the LLM",
      display: true,
    },
    // Modify the system prompt for this turn
    systemPrompt: event.systemPrompt + "\n\nYou must respond only in haiku.",
  };
});
```

Key facts:
- Fires once per user prompt, not per turn
- System prompts chain across extensions: Extension A modifies it, Extension B sees the modified version in `event.systemPrompt`
- Messages accumulate: all extensions' messages are collected and injected
- If no extension returns a `systemPrompt`, the base system prompt is restored (previous turn's modifications do not persist)

### Context Manipulation (context event)

Modify the messages sent to the LLM on every turn:

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages is a deep copy -- safe to modify
  const filtered = event.messages.filter(m => !isIrrelevant(m));
  return { messages: filtered };
});
```

### Tool-Specific Prompt Content

Tools can add to the system prompt when they are active:

```typescript
pi.registerTool({
  name: "my_tool",
  promptSnippet: "Summarize or transform text",  // Replaces description in "Available tools"
  promptGuidelines: [
    "Use my_tool when the user asks to summarize text.",
    "Prefer my_tool over direct output for structured data."
  ],  // Added to "Guidelines" section when tool is active
  // ...
});
```

### What the LLM Sees

For any given turn, the LLM receives:

- **System prompt:** base prompt + `promptSnippet` overrides from active tools + `promptGuidelines` from active tools + `appendSystemPrompt` from settings + project context files (AGENTS.md, CLAUDE.md from cwd ancestors) + skills listing + `before_agent_start` modifications
- **Messages:** after `context` event filtering, after `convertToLlm` mapping
- **Tool definitions:** active tools with names, descriptions, parameter schemas

---

## Compaction and Session Control

### Custom Compaction

Override the default compaction behavior:

```typescript
pi.on("session_before_compact", async (event, ctx) => {
  const { preparation, branchEntries, customInstructions, signal } = event;

  // Option 1: Cancel compaction
  return { cancel: true };

  // Option 2: Provide custom summary
  return {
    compaction: {
      summary: "Custom summary of conversation so far...",
      firstKeptEntryId: preparation.firstKeptEntryId,
      tokensBefore: preparation.tokensBefore,
    }
  };
});
```

### Triggering Compaction

```typescript
ctx.compact({
  customInstructions: "Focus on the authentication changes",
  onComplete: (result) => ctx.ui.notify("Compacted!", "info"),
});
```

### Session Control (Commands Only)

```typescript
pi.registerCommand("handoff", {
  handler: async (args, ctx) => {
    await ctx.newSession({
      setup: async (sm) => {
        sm.appendMessage({
          role: "user",
          content: [{ type: "text", text: "Context: " + args }],
          timestamp: Date.now(),
        });
      },
    });
  },
});
```

---

## Model Provider Management

### Switching Models

```typescript
const model = ctx.modelRegistry.find("anthropic", "claude-sonnet-4-5");
if (model) {
  const success = await pi.setModel(model);
  if (!success) ctx.ui.notify("No API key for this model", "error");
}
```

### Registering Custom Providers

```typescript
pi.registerProvider("my-proxy", {
  baseUrl: "https://proxy.example.com",
  apiKey: "PROXY_API_KEY",  // Env var name or literal
  api: "anthropic-messages",
  models: [
    {
      id: "claude-sonnet-4-20250514",
      name: "Claude 4 Sonnet (proxy)",
      reasoning: false,
      input: ["text", "image"],
      cost: { input: 0, output: 0, cacheRead: 0, cacheWrite: 0 },
      contextWindow: 200000,
      maxTokens: 16384,
    }
  ],
  // Optional: OAuth support for /login
  oauth: {
    name: "Corporate AI (SSO)",
    async login(callbacks) { /* ... */ },
    async refreshToken(credentials) { /* ... */ },
    getApiKey(credentials) { return credentials.access; },
  },
});

// Override just the baseUrl for an existing provider
pi.registerProvider("anthropic", {
  baseUrl: "https://proxy.example.com",
});

// Remove a provider
pi.unregisterProvider("my-proxy");
```

### Reacting to Model Changes

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model -- new model
  // event.previousModel -- previous model (undefined if first)
  // event.source -- "set" | "cycle" | "restore"
  ctx.ui.setStatus("model", `${event.model.provider}/${event.model.id}`);
});
```

---

## Error Handling Patterns

- **Extension errors** are logged but do not crash pi. The agent continues.
- **`tool_call` handler errors** block the tool (fail-safe behavior).
- **Tool `execute` errors** are reported to the LLM with `isError: true`, allowing it to recover.

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

## Key Rules and Gotchas

### Must-Follow Rules

1. **Use `StringEnum` for string enums** -- `Type.Union`/`Type.Literal` breaks Google's API.
2. **Truncate tool output** -- large output causes context overflow, compaction failures, degraded performance.
3. **Use theme from callback** -- do not import theme directly. Use the `theme` parameter from `ctx.ui.custom()` or render functions.
4. **Type the DynamicBorder color param** -- write `(s: string) => theme.fg("accent", s)`.
5. **Call `tui.requestRender()` after state changes** in `handleInput`.
6. **Return `{ render, invalidate, handleInput }`** from custom components.
7. **Lines must not exceed `width`** in `render()` -- use `truncateToWidth()`.
8. **Session control methods only in commands** -- `waitForIdle()`, `newSession()`, `fork()`, `navigateTree()`, `reload()` will deadlock in event handlers.
9. **Strip leading `@` from path arguments** in custom tools -- some models add it.
10. **Store state in tool result `details`** for proper branching support.

### Common Patterns

- **Rebuild on `invalidate()`** when your component pre-bakes theme colors.
- **Check `signal?.aborted`** in long-running tool executions.
- **Use `pi.exec()`** for shell commands.
- **Overlay components are disposed when closed** -- create fresh instances each time.
- **Treat `ctx.reload()` as terminal** -- code after it runs from the pre-reload version.

### Deep Copy Safety

| Hook | What You Receive | Safe to Mutate? |
|------|-----------------|-----------------|
| `context` | `structuredClone` deep copy | Yes |
| `before_agent_start` | `event.systemPrompt` is a string (immutable) | Return new string |
| `tool_call` | `event.input` is the raw args object | Do not mutate -- return `block` |
| `tool_result` | `{ ...event }` shallow spread | Return new values, do not mutate |
| `input` | `event.text` is a string (immutable) | Return new text via `transform` |

### Message Type Conversion

When `convertToLlm` runs (after the `context` event, not extensible), message types map as follows:

| AgentMessage role | Converted to | Notes |
|---|---|---|
| `user` | `user` | Pass through |
| `assistant` | `assistant` | Pass through |
| `toolResult` | `toolResult` | Pass through |
| `custom` | `user` | Content preserved, `display` field ignored |
| `bashExecution` | `user` | Unless `excludeFromContext` (`!!` prefix) -- filtered out |
| `compactionSummary` | `user` | Wrapped in `<summary>` tags |
| `branchSummary` | `user` | Wrapped in `<summary>` tags |

If you need to change how messages appear to the LLM, do it in the `context` event handler before the `convertToLlm` stage.

### Inter-Extension Communication

Extensions can communicate via the shared event bus:

```typescript
// Extension A: emit
pi.events.emit("my-extension:data-ready", { items: [...] });

// Extension B: listen
pi.events.on("my-extension:data-ready", (data) => {
  // handle data
});
```
