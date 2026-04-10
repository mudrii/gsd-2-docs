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

### SQLite State Engine (v2.44.0+)

Starting in v2.44.0, GSD replaced markdown file mutation with atomic SQLite-backed tool calls for all write-side state transitions. The key changes:

- **Tool-driven writes.** State transitions (plan-task, complete-task, validate-milestone, etc.) are now DB tools that run inside SQLite transactions rather than editing markdown files on disk. All prompts were migrated to reference the DB-backed tool API instead of direct file writes.
- **Single-writer engine v2 (v2.46.0).** A discipline layer enforces that only the designated writer (the agent or the auto-loop) can mutate state at any given time. This sits on top of the DB architecture and prevents race conditions between concurrent callers.
- **Single-writer engine v3 (v2.46.0).** Adds state machine guards (run inside the DB transaction), actor identity tracking, and reversibility. TOCTOU gaps are closed and bypass attempts through bare file writes are intercepted.
- **Schema migrations v8 through v11.** v8 laid the groundwork (planning columns on tasks). v9 added a `sequence` column on slices and tasks for ordering. v10 added `replan_triggered_at`. v11 refined planning columns for milestone validation. Each migration runs automatically on DB open.
- **`deriveStateFromDb`.** State derivation reads from the database rather than scanning the filesystem. Disk-only milestones are reconciled into the DB before the guard runs, and `queue-order.json` is honored for ordering. The filesystem derivation path remains as a fallback when the DB is unavailable.
- **Workflow logger (v2.46.0).** A structured audit log wired into the engine, tool handlers, manifest updates, and reconciliation paths. Records actor, action, and before/after snapshots.
- **DB-backed tool API with renderCall/renderResult previews (v2.45.0).** DB tools expose `renderCall` and `renderResult` for TUI display, giving inline previews of what changed after each state transition.
- **Transaction safety.** `transaction()` is re-entrant (safe to nest). The `slice_dependencies` table was added to `initSchema` to track inter-slice ordering constraints.

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

### Compaction and Context Changes (v2.29.0--v2.55.0)

Several significant changes were made to how context is assembled and managed:

- **Cache-ordered prompt assembly (v2.29.0).** Prompt segments are now ordered to maximize provider cache hit rates. A dashboard widget shows the cache hit rate so users can monitor effectiveness.
- **Prompt compression subsystem removed (v2.39.0).** The entire prompt compression subsystem (~4,100 lines) was deleted. It was replaced by simpler, more predictable mechanisms.
- **30K char hard cap on prompt preamble (v2.39.0).** A hard limit prevents runaway preamble growth. Preamble content beyond 30,000 characters is truncated.
- **Queue context in milestone planning prompts (v2.55.0).** Milestone planning prompts now include queue context -- information about other queued milestones and their dependencies -- so the planner can make better sequencing decisions.

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

### New Providers and Provider Changes (v2.34.0--v2.58.0)

- **OpenRouter auto-generated registry (v2.34.0).** The OpenRouter model list is now auto-generated from the OpenRouter API instead of being manually maintained. New models appear automatically.
- **Anthropic Vertex AI provider (v2.38.0).** A dedicated `anthropic-vertex` provider enables Claude models on Google Cloud Vertex AI with proper authentication and endpoint handling.
- **Non-api-key provider support (v2.44.0).** The provider registration system now supports providers that authenticate through mechanisms other than API keys (OAuth, CLI tokens, etc.). This unblocks providers like Claude Code CLI.
- **Claude Code CLI provider (v2.47.0).** A provider extension that delegates inference to the Claude Code CLI binary, running in an external tool execution mode. Tool calls are executed externally by the CLI rather than by pi's built-in executor.
- **Capability metadata replacing model-ID pattern matching (v2.52.0).** Model routing no longer infers capabilities (reasoning, vision, context window) from model ID string patterns. Instead, each model entry carries explicit capability metadata that the router reads directly.
- **GLM-5.1 on Z.AI (v2.57.0).** GLM-5.1 was added to the Z.AI provider in the custom models registry.
- **Ollama extension for local LLMs (v2.58.0).** An Ollama provider extension enables local LLM inference through a running Ollama instance.
- **Ollama native /api/chat provider (v2.64.0).** The Ollama extension was upgraded to a native provider using the `/api/chat` endpoint with full option exposure, replacing the earlier OpenAI-compatible wrapper.
- **OAuth auth provider for MCP HTTP transport (v2.64.0).** The MCP client extension gained an OAuth authentication provider for HTTP-based MCP transports, enabling authenticated connections to remote MCP servers that require OAuth flows.

### Reacting to Model Changes

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model -- new model
  // event.previousModel -- previous model (undefined if first)
  // event.source -- "set" | "cycle" | "restore"
  ctx.ui.setStatus("model", `${event.model.provider}/${event.model.id}`);
});
```

### Intercepting Model Selection (v2.62.0)

The `before_model_select` hook fires before each model selection in the capability-aware routing pipeline (ADR-004). Extensions can inspect the proposed model, the task requirements, and the scoring results, and optionally override the selection.

```typescript
pi.on("before_model_select", async (event, ctx) => {
  // event.proposedModel -- the model the router selected
  // event.taskRequirements -- computed requirement vector for this unit
  // event.eligibleModels -- all models that passed tier filtering
  // event.scores -- scoring breakdown per eligible model
  // Return { model: alternateModel } to override, or undefined to accept
});
```

This hook is registered as a placeholder in the GSD extension (`register-hooks.ts`) and can be overridden by user extensions. The verbose scoring output (enabled via `GSD_VERBOSE_ROUTING=1`) logs the full scoring breakdown for debugging.

---

## Commands/Tools API -- RPC Protocol v2 (v2.52.0+)

### RPC Protocol v2

v2.52.0 introduced RPC protocol version 2, a structured protocol for headless and programmatic operation. The protocol uses JSON lines over stdin/stdout.

**Init handshake.** A v2 client sends an `init` command with `protocolVersion: 2` and an optional `clientId`. The server responds with `RpcInitResult` containing the negotiated protocol version, the session ID, and capability lists (supported events and commands). Clients that skip the init handshake operate in v1 (implicit) mode.

```typescript
// v2 init handshake
{ type: "init", protocolVersion: 2, clientId: "my-client" }

// Server responds with:
{
  type: "response",
  command: "init",
  success: true,
  data: {
    protocolVersion: 2,
    sessionId: "abc123",
    capabilities: {
      events: ["execution_complete", "cost_update", ...],
      commands: ["prompt", "steer", "follow_up", ...]
    }
  }
}
```

**runId generation (v2.52.0).** Every `prompt`, `steer`, and `follow_up` command response now includes a `runId` string. This ID correlates the command with its subsequent events (`execution_complete`, `cost_update`), enabling clients to track multiple concurrent runs.

**v2-only events:**

- `execution_complete` -- emitted when a prompt/steer/follow_up finishes. Carries `runId`, terminal status (`completed | error | cancelled`), optional reason, and full `SessionStats`.
- `cost_update` -- emitted per-turn with running cost data. Carries `runId`, turn cost, cumulative cost, and token breakdown (input, output, cacheRead, cacheWrite).

### Event Formatters (v2.57.0)

Ten pure functions were added that map RPC events to human-readable text. These are used by the headless text mode and can be consumed by any client that wants formatted output without implementing its own rendering.

---

## Error Handling Patterns

- **Extension errors** are logged but do not crash pi. The agent continues.
- **`tool_call` handler errors** block the tool (fail-safe behavior).
- **Tool `execute` errors** are reported to the LLM with `isError: true`, allowing it to recover.

### Unified Error Pipeline (v2.29.0--v2.53.0)

- **GSDError enforcement and unused error code activation (v2.29.0).** All error paths now throw `GSDError` with a structured error code. Previously dormant error codes were activated and wired into the classification system.
- **Auto-resume on transient server errors (v2.29.0).** The retry handler auto-resumes on transient server errors (5xx, timeouts), not just rate limits. 5xx errors are no longer treated as credential-level failures.
- **Transient error classification improvements.** Stream-truncation JSON parse errors are classified as transient (v2.51.0). Terminated/connection errors are classified as transient in the provider error handler (v2.45.0).
- **Structured error propagation through UnitResult (v2.50.0).** The auto-mode pipeline propagates errors as structured `UnitResult` values rather than throwing exceptions. This enables callers to inspect error categories and decide on retry vs. escalation.
- **Unified classify-decide-act pipeline (v2.52.0).** Three overlapping error classifiers were consolidated into a single pipeline: classify the error (transient, rate-limit, auth, infra, etc.), decide on the response (retry, fallback, pause, stop), and act. This replaced ad-hoc classification scattered across modules.
- **Rate-limit errors attempt model fallback before pausing (v2.53.0).** When a rate-limit error occurs, the system first attempts to fall back to an alternative model. Only if no fallback is available does it pause and wait for the rate limit to clear.

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

---

## Workflow MCP Tools (v2.63.0)

The `@gsd-build/mcp-server` package exposes GSD workflow operations over MCP via `workflow-tools.ts`. This enables external MCP clients (Claude Code, Cursor, etc.) to query and mutate GSD project state.

### Architecture

The tool layer is split into two files:

| File | Package | Role |
|------|---------|------|
| `workflow-tool-executors.ts` | GSD extension (`src/resources/extensions/gsd/tools/`) | Transport-neutral business logic -- pure functions that operate on the SQLite DB |
| `workflow-tools.ts` | `@gsd-build/mcp-server` (`packages/mcp-server/src/`) | MCP transport adapter -- Zod schema validation, parameter marshaling, MCP protocol response formatting |

### Available Tools (v2.63.0)

6 read-only tools for project state queries:

- `gsd_milestone_status` -- query the status of a specific milestone
- `gsd_plan_milestone` -- read the full plan for a milestone (slices, tasks, dependencies)
- `gsd_plan_slice` -- read the plan for a specific slice (tasks, criteria)
- `gsd_replan_slice` -- read replan state for a slice
- `gsd_summary_save` -- save summary artifacts (SUMMARY, RESEARCH, CONTEXT, ASSESSMENT, CONTEXT-DRAFT)
- Additional mutation tools for complete-task, complete-slice, complete-milestone, validate-milestone, reassess-roadmap

### Type Contract

All executors return a `ToolExecutionResult`:

```typescript
interface ToolExecutionResult {
  content: Array<{ type: "text"; text: string }>;
  details: Record<string, unknown>;
}
```

The MCP adapter maps this to the MCP protocol response format. Error states (DB unavailable, invalid parameters, write-gate blocks) are returned as structured error responses rather than thrown exceptions.

---

## Safety Harness API (v2.64.0)

The safety harness (`src/resources/extensions/gsd/safety/safety-harness.ts`) exports a public API for damage control during auto-mode execution.

### Configuration API

```typescript
import {
  resolveSafetyHarnessConfig,
  isHarnessEnabled,
} from "./safety/safety-harness.js";

// Resolve full config from raw preferences (missing fields use defaults)
const config = resolveSafetyHarnessConfig(preferences.safety_harness);

// Fast gate check -- used at hook registration and phase integration points
if (isHarnessEnabled(preferences.safety_harness)) {
  // register safety hooks
}
```

### Evidence Collection API

```typescript
import {
  resetEvidence,
  getEvidence,
  getBashEvidence,
  getFilePaths,
  recordToolCall,
  recordToolResult,
} from "./safety/safety-harness.js";

// Reset evidence at the start of each unit
resetEvidence();

// Record a tool invocation (called from tool_call hook)
recordToolCall(toolName, toolInput);

// Record a tool result (called from tool_result hook)
recordToolResult(toolName, toolResult);

// Query collected evidence
const allEvidence: EvidenceEntry[] = getEvidence();
const bashCmds: BashEvidence[] = getBashEvidence();
const touchedFiles: string[] = getFilePaths();
```

### Destructive Command Classification

```typescript
import { classifyCommand } from "./safety/safety-harness.js";

const result: CommandClassification = classifyCommand("rm -rf /tmp/build");
// result.level: "safe" | "warning" | "dangerous"
// result.reason: human-readable explanation
```

### File Change Validation

```typescript
import { validateFileChanges } from "./safety/safety-harness.js";

const audit: FileChangeAudit = await validateFileChanges(basePath, plannedFiles);
// audit.violations: FileViolation[] -- files changed outside the plan
// audit.unplannedChanges: string[] -- unexpected file modifications
```

### Evidence Cross-Reference

```typescript
import { crossReferenceEvidence } from "./safety/safety-harness.js";

const mismatches: EvidenceMismatch[] = crossReferenceEvidence(
  claimedEvidence,
  collectedEvidence,
);
// Each mismatch identifies a claim not backed by actual tool call records
```

---

## Notification Panel API (v2.65.0)

The notification panel provides persistent notifications via TUI overlay, widget, and web API.

### ctx.ui.notify

The existing `ctx.ui.notify()` method was extended with support for persistent notifications that survive across turns:

```typescript
// Transient notification (existing behavior)
ctx.ui.notify("Task completed", "info");

// The notification panel aggregates notifications from the store
// and renders them as a TUI overlay or widget depending on context
```

### ctx.ui.setWidget

Widgets can be placed above or below the editor to display persistent status:

```typescript
// Set a widget with rendered lines
ctx.ui.setWidget("notifications", ["Line 1", "Line 2"]);

// Remove a widget
ctx.ui.setWidget("notifications", undefined);
```

### Notification Store

The notification store (`notification-store.ts`) manages notification lifecycle:

- Notifications are stored with timestamps and severity levels.
- The overlay (`notification-overlay.ts`) renders notifications as a floating panel with backdrop dimming and content-fit sizing.
- The widget (`notification-widget.ts`) provides a compact inline view.
- The web API surfaces notifications through the existing web interface event stream.
- Lock-based ownership prevents foreign lock deletion when multiple sessions are active.
