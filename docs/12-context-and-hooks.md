# Context and Hooks

A deep reference for how context flows through pi and how to intercept and shape it. Read the extending-pi guide first for fundamentals, then use this for precision work.

---

## The Context Pipeline

The full journey of a user prompt from keypress to LLM input, through every transformation stage.

```
User types prompt and hits Enter
│
├─► Extension command check (/command)
│   If match → run handler, skip everything below
│
├─► input event
│   Extensions can: transform text/images, intercept entirely, or pass through
│
├─► Skill expansion (/skill:name)
│   Skill file content injected into prompt text
│
├─► Prompt template expansion (/template)
│   Template file content merged into prompt text
│
├─► before_agent_start event [ONCE per user prompt]
│   Extensions can:
│     • Inject custom messages (appended after user message)
│     • Modify the system prompt (chained across extensions)
│
├─► Agent.prompt(messages)
│   Messages array: [user message, ...nextTurn messages, ...extension messages]
│
│   ┌── Turn loop (repeats while LLM calls tools) ──────────────┐
│   │                                                            │
│   │  context event [EVERY turn]                                │
│   │    Extensions receive AgentMessage[] deep copy             │
│   │    Can filter, reorder, inject, or replace messages        │
│   │    Multiple handlers chain: each sees previous output      │
│   │                                                            │
│   │  convertToLlm [EVERY turn, AFTER context event]           │
│   │    AgentMessage[] → Message[]                              │
│   │    Custom types mapped to user role                        │
│   │    Not extensible — hardcoded in messages.ts               │
│   │                                                            │
│   │  LLM call                                                  │
│   │    System prompt + converted messages + tool definitions   │
│   │                                                            │
│   │  Tool execution (if LLM calls tools)                       │
│   │    tool_call event → can block                             │
│   │    execute runs                                            │
│   │    tool_result event → can modify result                   │
│   │    Steering check → may skip remaining tools               │
│   │                                                            │
│   │  Follow-up check (if no more tool calls)                   │
│   │    Queued follow-up messages become next turn input         │
│   │                                                            │
│   └────────────────────────────────────────────────────────────┘
│
└─► agent_end event
```

### Key Timing Distinctions

| Hook | When | How often | Can modify |
|------|------|-----------|-----------|
| `input` | Before expansion | Once per user input | Input text |
| `before_agent_start` | After expansion, before agent loop | Once per user prompt | System prompt + inject messages |
| `context` | Before each LLM call | Every turn in agent loop | Message array |
| `tool_call` | Before each tool execution | Per tool call | Block execution |
| `tool_result` | After each tool execution | Per tool call | Result content/details |

### The Deep Copy Question

| Hook | What you receive | Safe to mutate? |
|------|-----------------|-----------------|
| `context` | `structuredClone` deep copy | Yes |
| `before_agent_start` | `event.systemPrompt` is a string | Return new string |
| `tool_call` | `event.input` is the raw args object | Do not mutate — return `block` |
| `tool_result` | Shallow spread | Return new values, don't mutate |
| `input` | `event.text` is a string | Return new text via `transform` |

---

## Hook Reference

### 1. Input Hooks

#### `input`

**When:** User submits text (Enter in editor, RPC message, or `pi.sendUserMessage` from an extension with `source: "extension"`).

**Important edge case:** Extension commands (`/mycommand`) are checked **before** `input` fires. Built-in commands (`/new`, `/model`, etc.) are checked **after** `input` transforms — so `input` can transform text into a built-in command.

**Chaining:** Sequential through all extensions. Each handler sees the text output of the previous handler's `transform`. First `handled` stops the chain and the pipeline.

```typescript
pi.on("input", async (event, ctx) => {
  // event.text: string — current text (possibly transformed by earlier handler)
  // event.images: ImageContent[] | undefined
  // event.source: "interactive" | "rpc" | "extension"

  // Option 1: Pass through
  return { action: "continue" };
  // or return nothing (undefined) — same as continue

  // Option 2: Transform
  return { action: "transform", text: "rewritten", images: newImages };

  // Option 3: Swallow (no LLM call, no further handlers)
  return { action: "handled" };
});
```

---

### 2. Agent Lifecycle Hooks

#### `before_agent_start`

**When:** After input processing, skill/template expansion, and the user message is constructed — but before `agent.prompt()` is called.

**Fires:** Once per user prompt. Does NOT fire on subsequent turns within the same agent run.

**Chaining:**
- **System prompt chains.** Extension A modifies `event.systemPrompt`, Extension B sees that modified version. If no extension returns a `systemPrompt`, the base prompt is used (resetting any previous turn's modifications). This means `before_agent_start` modifications are per-prompt — the change must be re-applied every turn.
- **Messages accumulate.** All `message` results are collected and injected as separate `CustomMessage` entries with `role: "custom"` after the user message.

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  // event.prompt: string — expanded user prompt text
  // event.images: ImageContent[] | undefined
  // event.systemPrompt: string — current system prompt (may be chained from earlier extension)

  return {
    // Optional: inject a custom message into the session
    message: {
      customType: "my-extension",  // identifies the message type
      content: "Text the LLM sees",
      display: true,                // controls UI rendering only — LLM ALWAYS sees it
      details: { any: "data" },     // for rendering and state reconstruction
    },

    // Optional: modify the system prompt for this agent run
    systemPrompt: event.systemPrompt + "\nNew instructions",
  };
});
```

**Critical detail:** The `display` field controls whether the message shows in the TUI chat log. The LLM **always** sees the message content regardless of `display`.

**Error handling:** If a handler throws, the error is captured and reported. Other handlers still run.

#### `agent_start`

**When:** The agent loop begins (after `before_agent_start`, after `agent.prompt()` is called). Fires once per agent run. Informational only — no return value.

```typescript
pi.on("agent_start", async (event, ctx) => {
  // event: { type: "agent_start" }
  // Useful for: starting timers, resetting per-run state
});
```

#### `agent_end`

**When:** The agent loop finishes (all turns complete, no more tool calls, no queued messages). Fires once per agent run.

```typescript
pi.on("agent_end", async (event, ctx) => {
  // event.messages: AgentMessage[] — NEW messages from this run only
  // Use ctx.sessionManager.getBranch() for full conversation history
});
```

---

### 3. Per-Turn Hooks

#### `context`

**When:** Before each LLM call, after the turn starts. This is the last chance to modify what the LLM sees.

**Fires:** Every turn. If the LLM calls 3 tools and loops back, `context` fires 4 times (once for initial call + once per loop-back).

**Chaining:** Sequential. Each handler receives the output of the previous. First handler gets a `structuredClone` deep copy.

```typescript
pi.on("context", async (event, ctx) => {
  // event.messages: AgentMessage[] — deep copy, safe to mutate

  // Filter out messages
  const filtered = event.messages.filter(m => !isIrrelevant(m));
  return { messages: filtered };

  // Or inject messages
  return { messages: [...event.messages, syntheticMessage] };

  // Or return nothing to pass through unchanged
});
```

**What `event.messages` contains:**
- All roles: `user`, `assistant`, `toolResult`, `custom`, `bashExecution`, `compactionSummary`, `branchSummary`
- Current user prompt and custom messages injected by `before_agent_start`
- Tool results from earlier turns in this agent run
- Historical messages from the session (including compaction summaries)

**What it does NOT contain:**
- The system prompt (use `before_agent_start` for that)
- Tool definitions (use `pi.setActiveTools()` for that)

**Important:** Messages injected in `context` are NOT persisted to the session. They exist only for the LLM call. This is actually useful — it means the context is always fresh.

#### `turn_start` / `turn_end`

```typescript
pi.on("turn_start", async (event, ctx) => {
  // event.turnIndex: number — 0-based index of this turn within the agent run
  // event.timestamp: number
});

pi.on("turn_end", async (event, ctx) => {
  // event.turnIndex: number
  // event.message: AgentMessage — the assistant's response message
  // event.toolResults: ToolResultMessage[] — results from tools called this turn
});
```

#### `message_start` / `message_update` / `message_end`

```typescript
pi.on("message_start", async (event, ctx) => {
  // event.message: AgentMessage — user, assistant, toolResult, or custom
});

pi.on("message_update", async (event, ctx) => {
  // event.message: AgentMessage — partial assistant message (streaming)
  // event.assistantMessageEvent: AssistantMessageEvent — the specific token event
});

pi.on("message_end", async (event, ctx) => {
  // event.message: AgentMessage — final message
  // Messages are persisted to the session file at this point
});
```

---

### 4. Tool Hooks

#### `tool_call`

**When:** After the LLM requests a tool call, before it executes.

**Chaining:** Sequential. If any handler returns `{ block: true }`, execution stops immediately. The block reason becomes an Error returned as the tool result with `isError: true`.

```typescript
pi.on("tool_call", async (event, ctx) => {
  // event.toolCallId: string
  // event.toolName: string
  // event.input: typed based on tool (use isToolCallEventType for narrowing)

  // Block execution
  return { block: true, reason: "Not allowed in read-only mode" };

  // Allow execution (return nothing or undefined)
});
```

**Type narrowing:**
```typescript
import { isToolCallEventType } from "@mariozechner/pi-coding-agent";

pi.on("tool_call", async (event, ctx) => {
  if (isToolCallEventType("bash", event)) {
    event.input.command; // string — typed!
  }
  if (isToolCallEventType("write", event)) {
    event.input.path;    // string
    event.input.content; // string
  }
  // Custom tools need explicit type params:
  if (isToolCallEventType<"my_tool", { action: string }>("my_tool", event)) {
    event.input.action;  // string
  }
});
```

#### `tool_execution_start` / `tool_execution_update` / `tool_execution_end`

Informational events during tool execution. No return values.

```typescript
pi.on("tool_execution_start", async (event) => {
  // event.toolCallId, event.toolName, event.args
});

pi.on("tool_execution_update", async (event) => {
  // event.partialResult — streaming progress from onUpdate callback
});

pi.on("tool_execution_end", async (event) => {
  // event.result, event.isError
});
```

#### `tool_result`

**When:** After a tool finishes executing, before the result is returned to the agent loop.

**Chaining:** Sequential. Each handler can modify the result. All handlers see the evolving `currentEvent` with content/details/isError updated by previous handlers.

**Also fires for errors:** If tool execution throws, `tool_result` still fires with `isError: true`.

```typescript
pi.on("tool_result", async (event, ctx) => {
  // event.toolCallId: string
  // event.toolName: string
  // event.input: Record<string, unknown>
  // event.content: (TextContent | ImageContent)[]
  // event.details: unknown
  // event.isError: boolean

  // Modify the result
  return {
    content: [...event.content, { type: "text", text: "\n\nAudit: logged" }],
    isError: false, // can flip error state
  };

  // Return nothing to pass through unchanged
});
```

---

### 5. Session Hooks

#### `session_start`

**When:** Initial session load (startup) and after session switch/fork. Also fires after `/reload`.

**Use for:** State restoration from session entries, initial setup.

```typescript
pi.on("session_start", async (_event, ctx) => {
  // Restore state from session
  for (const entry of ctx.sessionManager.getBranch()) {
    if (entry.type === "custom" && entry.customType === "my-state") {
      myState = entry.data;
    }
  }
});
```

#### `session_before_switch` / `session_switch`

```typescript
pi.on("session_before_switch", async (event) => {
  // event.reason: "new" | "resume"
  // event.targetSessionFile?: string (only for resume)
  return { cancel: true }; // prevent the switch
});
```

#### `session_before_fork` / `session_fork`

```typescript
pi.on("session_before_fork", async (event) => {
  // event.entryId: string — the entry being forked from
  return { cancel: true };
  // or
  return { skipConversationRestore: true }; // fork without restoring messages
});
```

#### `session_before_compact` / `session_compact`

```typescript
pi.on("session_before_compact", async (event) => {
  // event.preparation: CompactionPreparation
  // event.branchEntries: SessionEntry[]
  // event.customInstructions?: string
  // event.signal: AbortSignal

  return { cancel: true };
  // or provide custom compaction:
  return {
    compaction: {
      summary: "My custom summary",
      firstKeptEntryId: event.preparation.firstKeptEntryId,
      tokensBefore: event.preparation.tokensBefore,
    }
  };
});
```

#### `session_shutdown`

```typescript
pi.on("session_shutdown", async (_event, ctx) => {
  // Last chance to persist state
  // Keep it fast — process is exiting
});
```

---

### 6. Model Hooks

#### `model_select`

**When:** Model changes via `/model`, Ctrl+P cycling, or session restore.

```typescript
pi.on("model_select", async (event, ctx) => {
  // event.model: Model — the new model
  // event.previousModel: Model | undefined
  // event.source: "set" | "cycle" | "restore"
});
```

---

### 7. Resource Hooks

#### `resources_discover`

**When:** At startup and after `/reload`. Lets extensions provide additional skill, prompt template, and theme paths. Not documented in the main extending-pi docs.

```typescript
pi.on("resources_discover", async (event, ctx) => {
  // event.cwd: string
  // event.reason: "startup" | "reload"

  return {
    skillPaths: [join(__dirname, "skills", "SKILL.md")],
    promptPaths: [join(__dirname, "prompts", "my-template.md")],
    themePaths: [join(__dirname, "themes", "dark.json")],
  };
});
```

Returned paths are loaded and integrated into the system prompt (skills) and available commands (prompts/themes). The system prompt is rebuilt after resources are extended.

---

### 8. User Bash Hooks

#### `user_bash`

**When:** User executes a command via `!` or `!!` prefix in the editor.

```typescript
pi.on("user_bash", async (event, ctx) => {
  // event.command: string
  // event.excludeFromContext: boolean (true if !! prefix)
  // event.cwd: string

  // Provide custom execution (e.g., SSH)
  return {
    operations: { execute: (cmd) => sshExec(remote, cmd) },
  };

  // Or provide a full replacement result
  return {
    result: { output: "custom output", exitCode: 0, cancelled: false, truncated: false },
  };
});
```

---

### Execution Order Across Extensions

All hooks iterate through extensions in **load order** (project-local first, then global, then explicitly configured via `-e`). Within each extension, handlers for the same event run in registration order.

For hooks that chain (`context`, `before_agent_start.systemPrompt`, `input`, `tool_result`): Extension A's handler runs first, Extension B sees A's output.

For hooks that short-circuit (`tool_call` with `block`, `input` with `handled`, session `cancel`): First extension to return the short-circuit value wins — remaining handlers are skipped.

---

## Context Injection Patterns

### Pattern 1: Per-Prompt System Prompt Modification

**Use when:** You want to change the LLM's behavior for the entire agent run based on some condition. Use `before_agent_start` because `context` cannot modify the system prompt.

```typescript
let debugMode = false;

pi.registerCommand("debug", {
  handler: async (_args, ctx) => {
    debugMode = !debugMode;
    ctx.ui.notify(debugMode ? "Debug mode ON" : "Debug mode OFF");
  },
});

pi.on("before_agent_start", async (event) => {
  if (debugMode) {
    return {
      systemPrompt: event.systemPrompt + `

## Debug Mode
- Show your reasoning for each decision
- Before executing any tool, explain what you expect to happen
- After each tool result, explain what you learned`,
    };
  }
});
```

### Pattern 2: Invisible Context Injection

**Use when:** You need the LLM to know something without the user seeing it in the chat.

```typescript
pi.on("before_agent_start", async (event, ctx) => {
  const gitBranch = await getBranch();
  const recentCommits = await getRecentCommits(5);

  return {
    message: {
      customType: "git-context",
      content: `[Git Context] Branch: ${gitBranch}\nRecent commits:\n${recentCommits}`,
      display: false,  // User doesn't see this in chat
      // But the LLM DOES see it — display only controls UI rendering
    },
  };
});
```

**Important:** `display: false` hides from UI only. There is no way to inject UI-only messages that participate in the conversation flow — `display: false` only hides from the TUI, not from the LLM.

### Pattern 3: Conditional Context Filtering

**Use when:** Some messages in the history are no longer relevant and waste context tokens.

```typescript
pi.on("context", async (event) => {
  return {
    messages: event.messages.filter(m => {
      // Remove custom messages from a previous mode
      if (m.role === "custom" && m.customType === "plan-mode-context") {
        return currentMode === "plan";
      }
      return true;
    }),
  };
});
```

### Pattern 4: Dynamic Context Injection Per Turn

**Use when:** You want to add context that changes between turns (e.g., current file state, running process output).

```typescript
pi.on("context", async (event, ctx) => {
  const liveStatus = await getProcessStatus();

  const contextMessage = {
    role: "user" as const,
    content: [{ type: "text" as const, text: `[Live Status] ${liveStatus}` }],
    timestamp: Date.now(),
  };

  return {
    messages: [...event.messages, contextMessage],
  };
});
```

Messages injected in `context` are NOT persisted to the session — they are regenerated fresh each turn.

### Pattern 5: Deferred Context (Next Turn)

**Use when:** You want to attach context to the user's next prompt without interrupting the current conversation.

```typescript
pi.sendMessage(
  {
    customType: "deferred-context",
    content: "The test suite passed with 47/47 tests",
    display: false,
  },
  { deliverAs: "nextTurn" }
);
```

Unlike `context` hook injection, these messages ARE persisted to the session.

### Pattern 6: Context Window Management

**Use when:** You are approaching the context limit and need to intelligently prune.

```typescript
pi.on("context", async (event, ctx) => {
  const usage = ctx.getContextUsage();
  if (!usage || usage.percent === null || usage.percent < 70) {
    return; // plenty of room
  }

  // Aggressive pruning: remove tool results beyond the last 20
  let toolResultCount = 0;
  const total = event.messages.filter(m => m.role === "toolResult").length;

  return {
    messages: event.messages.filter(m => {
      if (m.role === "toolResult") {
        toolResultCount++;
        return toolResultCount > total - 20;
      }
      return true;
    }),
  };
});
```

### Pattern 7: Steering with Context

**Use when:** You want to redirect the agent mid-run with additional context.

```typescript
pi.sendMessage(
  {
    customType: "user-feedback",
    content: "IMPORTANT: The user just updated the config file. Re-read config.json before continuing.",
    display: true,
  },
  { deliverAs: "steer" }
);
```

The current tool call finishes, remaining queued tool calls are skipped, and the steering message becomes input for the next turn.

### Pattern 8: Follow-Up Context After Completion

**Use when:** You want to trigger another LLM turn after the agent finishes, with additional context.

```typescript
pi.on("agent_end", async (event, ctx) => {
  const hasEdits = event.messages.some(m =>
    m.role === "toolResult" && m.toolName === "edit"
  );

  if (hasEdits) {
    pi.sendMessage(
      {
        customType: "auto-verify",
        content: "You just made edits. Please verify them by running the test suite.",
        display: true,
      },
      { deliverAs: "followUp", triggerTurn: true }
    );
  }
});
```

### Pattern 9: Tool-Scoped Context via promptGuidelines

**Use when:** You want context that only appears when specific tools are active.

```typescript
pi.registerTool({
  name: "deploy",
  label: "Deploy",
  description: "Deploy the application",
  promptSnippet: "Deploy the application to staging or production",
  promptGuidelines: [
    "Always run tests before deploying",
    "Never deploy to production without explicit user confirmation",
    "After deploying, verify the health check endpoint",
  ],
  parameters: Type.Object({ /* ... */ }),
  async execute(toolCallId, params, signal, onUpdate, ctx) { /* ... */ },
});
```

The `promptGuidelines` are added to the "Guidelines" section ONLY when the `deploy` tool is in the active tool set.

### Pattern 10: Input Preprocessing / Macros

**Use when:** You want custom syntax that expands before the LLM sees it.

```typescript
import { readFileSync } from "node:fs";

pi.on("input", async (event) => {
  // Expand @file references to file contents
  const expanded = event.text.replace(/@(\S+)/g, (match, filePath) => {
    try {
      const content = readFileSync(filePath, "utf-8");
      return `\`\`\`${filePath}\n${content}\n\`\`\``;
    } catch {
      return match; // leave unchanged if can't read
    }
  });

  if (expanded !== event.text) {
    return { action: "transform", text: expanded };
  }
  return { action: "continue" };
});
```

### Pattern 11: Context-Aware Tool Blocking

**Use when:** You want to prevent certain tool usage based on conversation context.

```typescript
let inPlanMode = false;

pi.on("tool_call", async (event, ctx) => {
  if (!inPlanMode) return;

  const destructiveTools = ["edit", "write", "bash"];

  if (event.toolName === "bash" && isToolCallEventType("bash", event)) {
    if (isSafeCommand(event.input.command)) return;
  }

  if (destructiveTools.includes(event.toolName)) {
    return {
      block: true,
      reason: `Plan mode active: ${event.toolName} is not allowed. Use /plan to exit plan mode.`,
    };
  }
});
```

### Anti-Patterns

**Do not modify system prompt in `context`** — the `context` event can only modify messages, not the system prompt. `return { systemPrompt: "new prompt" }` is not a valid return field.

**Do not rely on `display: false` for security** — `display: false` only hides from UI, the LLM still sees the content.

**Do not use `context` for one-time injection** — `context` fires every turn. Use `before_agent_start` with `message` for one-time per-prompt injection instead.

**Do not use `getEntries()` for branch-aware state** — `getEntries()` returns ALL entries including dead branches. Use `getBranch()` instead.

---

## Message Types and LLM Visibility

### The AgentMessage Type Hierarchy

Pi uses `AgentMessage` as its internal message type:

```typescript
// Standard LLM messages
type Message = UserMessage | AssistantMessage | ToolResultMessage;

// Custom messages added by pi's coding agent
interface CustomAgentMessages {
  bashExecution: BashExecutionMessage;
  custom: CustomMessage;
  branchSummary: BranchSummaryMessage;
  compactionSummary: CompactionSummaryMessage;
}

type AgentMessage = Message | CustomAgentMessages[keyof CustomAgentMessages];
```

### Message Type → LLM Conversion Table

| AgentMessage type | Role seen by LLM | Content transformation | When excluded |
|---|---|---|---|
| `user` | `user` | Pass through unchanged | Never |
| `assistant` | `assistant` | Pass through unchanged | Never |
| `toolResult` | `toolResult` | Pass through unchanged | Never |
| `custom` | `user` | `content` preserved as-is | Never — ALL custom messages reach the LLM |
| `bashExecution` | `user` | Formatted: `` Ran `cmd`\n```\noutput\n``` `` | When `excludeFromContext: true` (`!!` prefix) |
| `compactionSummary` | `user` | Wrapped in `<summary>` tags | Never |
| `branchSummary` | `user` | Wrapped in `<summary>` tags | Never |

`convertToLlm` is not extensible — it is a hardcoded function in `messages.ts`. If you need to change how messages appear to the LLM, do it in the `context` event handler before this stage.

### The `display` Field Misconception

```typescript
pi.sendMessage({
  customType: "my-context",
  content: "This text goes to the LLM",
  display: false,  // ONLY controls UI rendering
});
```

- `display: true`: Message appears in the TUI chat log
- `display: false`: Message is hidden from the TUI chat log
- The LLM ALWAYS receives the content as a `user` role message regardless of `display`
- The message is ALWAYS persisted to the session file regardless of `display`

### How Custom Messages Become User Messages

In `convertToLlm` (messages.ts), custom messages are converted to:

```typescript
case "custom": {
  const content = typeof m.content === "string"
    ? [{ type: "text", text: m.content }]
    : m.content;
  return {
    role: "user",
    content,
    timestamp: m.timestamp,
  };
}
```

The `customType`, `display`, and `details` fields are all stripped. The LLM sees a plain user message with the content.

### Bash Execution Messages

`!` commands (included in context):

```typescript
// User types: !ls -la
// LLM sees:
{
  role: "user",
  content: [{ type: "text", text: "Ran `ls -la`\n```\n<output>\n```" }]
}
```

`!!` commands (excluded from context): The `excludeFromContext` flag causes `convertToLlm` to return `undefined` for this message, effectively removing it from the LLM's view entirely.

### Compaction and Branch Summary Messages

**Compaction Summary** — When the context is compacted, older messages are replaced with:

```
The conversation history before this point was compacted into the following summary:

<summary>
[LLM-generated summary]
</summary>
```

**Branch Summary** — When navigating away from a branch and back:

```
The following is a summary of a branch that this conversation came back from:

<summary>
[summary of the branch]
</summary>
```

### What the LLM Never Sees

1. **`appendEntry` data** — Extension-private entries are stored in the session file but NEVER included in the message array.
2. **`details` on custom messages** — Stripped by `convertToLlm`.
3. **`details` on tool results** — Only `content` reaches the LLM.
4. **`!!` bash execution output** — Explicitly excluded from context.
5. **Tool definitions not in the active set** — If a tool is not in `getActiveTools()`, the LLM doesn't know it exists.
6. **`promptSnippet` and `promptGuidelines` from inactive tools** — Only active tools contribute to the system prompt.

### The Message Array Order

For a typical conversation, the LLM sees (after `context` event and `convertToLlm`):

```
1. [compactionSummary → user]  (if compaction happened)
2. [branchSummary → user]      (if navigated back from a branch)
3. [user]                       (first user message after compaction)
4. [assistant]                  (LLM response)
5. [toolResult]                 (tool results)
6. [user]                       (next user message)
7. [custom → user]              (extension-injected message)
...
N. [user]                       (current prompt)
N+1. [custom → user]            (before_agent_start injected messages)
N+2. [custom → user]            (nextTurn queued messages)
```

---

## Inter-Extension Communication

### pi.events — The Shared Event Bus

```typescript
// Emit an event on a channel
pi.events.emit("my-channel", { action: "started", id: 123 });

// Subscribe to a channel — returns an unsubscribe function
const unsub = pi.events.on("my-channel", (data) => {
  const payload = data as { action: string; id: number };
});

// Stop listening
unsub();
```

| Property | Behavior |
|---|---|
| **Typing** | `data` is `unknown`. No generics. Cast at the consumer. |
| **Error handling** | Handlers are wrapped in async try/catch. Errors log but don't propagate. |
| **Ordering** | Handlers fire in subscription order. |
| **Persistence** | No replay, no persistence. Emit before subscribe → event is lost. |
| **Scope** | Shared across ALL extensions in the session. |
| **Lifecycle** | Cleared on `/reload`. Subscriptions from old extension instances are gone. |

### Extension A Signals Extension B

```typescript
// Extension A: plan-mode.ts
export default function (pi: ExtensionAPI) {
  pi.registerCommand("plan", {
    handler: async (_args, ctx) => {
      planEnabled = !planEnabled;
      pi.events.emit("mode-change", { mode: planEnabled ? "plan" : "normal" });
    },
  });
}

// Extension B: status-display.ts
export default function (pi: ExtensionAPI) {
  pi.events.on("mode-change", (data) => {
    const { mode } = data as { mode: string };
    // React to mode change
  });
}
```

### Shared Module State

If two extensions are loaded from the same package, they can share state through module-level variables in a shared file. On `/reload`, everything is re-imported from scratch — shared state resets.

### Session Entries as Coordination Points

Extensions can read each other's `appendEntry` data from the session during `session_start`:

```typescript
// Extension A writes:
pi.appendEntry("ext-a-config", { theme: "dark", verbose: true });

// Extension B reads during session_start:
pi.on("session_start", async (_event, ctx) => {
  for (const entry of ctx.sessionManager.getEntries()) {
    if (entry.type === "custom" && entry.customType === "ext-a-config") {
      const config = entry.data as { theme: string; verbose: boolean };
    }
  }
});
```

This only works after `session_start` — not suitable for real-time coordination during a turn.

---

## System Prompt Anatomy

### The Final Prompt Structure

When `buildSystemPrompt()` runs, it assembles sections in this exact order:

```
1. Base prompt (default or SYSTEM.md override)
   ├── Identity statement
   ├── Available tools list
   ├── Custom tools note
   ├── Guidelines
   └── Pi documentation pointers

2. Append system prompt (APPEND_SYSTEM.md)

3. Project context files
   ├── ~/.gsd/agent/AGENTS.md (global)
   ├── Ancestor AGENTS.md / CLAUDE.md files
   └── cwd AGENTS.md / CLAUDE.md

4. Skills listing
   └── <available_skills> XML block

5. Date/time and working directory
```

After `buildSystemPrompt()`, extensions can further modify via `before_agent_start`.

### Section 1: The Base Prompt

**Default Base Prompt (no SYSTEM.md):**

```
You are an expert coding assistant operating inside pi, a coding agent harness.
You help users by reading files, executing commands, editing code, and writing new files.

Available tools:
- read: Read file contents
- bash: Execute bash commands (ls, grep, find, etc.)
- edit: Make surgical edits to files (find exact text and replace)
- write: Create or overwrite files
- my_custom_tool: [promptSnippet or description]

Guidelines:
- Use bash for file operations like ls, rg, find
- Use read to examine files before editing.
- Use edit for precise changes (old text must match exactly)
- Use write only for new files or complete rewrites
- [extension tool promptGuidelines inserted here]
- Be concise in your responses
- Show file paths clearly when working with files

Pi documentation:
- Main documentation: [path]
```

**SYSTEM.md Override:** If `.gsd/SYSTEM.md` (project) or `~/.gsd/agent/SYSTEM.md` (global) exists, its contents **completely replace** the default base prompt. Project takes precedence over global. Only one SYSTEM.md is used.

What still gets appended even with a custom SYSTEM.md:
- APPEND_SYSTEM.md content
- Project context files (AGENTS.md / CLAUDE.md)
- Skills listing (if the `read` tool is active)
- Date/time and cwd

**How Tool Descriptions Appear:** Each active tool gets a line in "Available tools". The description priority is: `promptSnippet` → built-in description → tool's `name` as fallback.

**How Guidelines Are Built:**

| Condition | Guideline |
|---|---|
| bash active, no grep/find/ls | "Use bash for file operations like ls, rg, find" |
| bash active + grep/find/ls | "Prefer grep/find/ls tools over bash for file exploration" |
| read + edit active | "Use read to examine files before editing" |
| edit active | "Use edit for precise changes (old text must match exactly)" |
| write active | "Use write only for new files or complete rewrites" |
| Always | "Be concise in your responses" |
| Always | "Show file paths clearly when working with files" |

Extension tool `promptGuidelines` are appended after the built-in guidelines and deduplicated.

### Section 3: Project Context Files

Pi walks the filesystem collecting context files:

```
1. ~/.gsd/agent/AGENTS.md (global)
2. Walk from cwd upward to root:
   - Each directory: check for AGENTS.md, then CLAUDE.md (first found wins per directory)
   - Files collected root-down (ancestors first, cwd last)
```

All found files are concatenated under a "# Project Context" header.

**AGENTS.md vs CLAUDE.md:** Both are treated identically. Per directory, AGENTS.md is checked first. If it exists, CLAUDE.md in the same directory is skipped.

### Section 4: Skills Listing

If the `read` tool is active and skills are loaded:

```xml
<available_skills>
  <skill>
    <name>commit-outstanding</name>
    <description>Commit all uncommitted files in logical groups</description>
    <location>/Users/you/.gsd/agent/skills/commit-outstanding/SKILL.md</location>
  </skill>
</available_skills>
```

Only names, descriptions, and file paths go into the system prompt. The full skill content is NOT loaded. The agent uses the `read` tool to load specific skills on demand. This keeps the system prompt small even with many skills.

### When the System Prompt Is Rebuilt

| Trigger | What happens |
|---|---|
| Startup | Full rebuild with initial tool set |
| `setActiveToolsByName()` | Rebuild with new tool set |
| `reload()` (`/reload`) | Full rebuild — reloads SYSTEM.md, context files, skills, extensions |
| `extendResourcesFromExtensions()` | Rebuild after `resources_discover` adds new skills |
| `_refreshToolRegistry()` | Rebuild when extension tools change dynamically |

**Per-Prompt Modifications:** On each user prompt, `before_agent_start` can modify the system prompt. This modification is NOT persisted — the base prompt is restored if no extension modifies it on the next prompt.

### Every Lever for Shaping the System Prompt

**Static (file-based, loaded at startup):**

| Mechanism | Scope | Effect |
|---|---|---|
| `SYSTEM.md` | Replace base prompt entirely | Nuclear option — you own everything |
| `APPEND_SYSTEM.md` | Append to base prompt | Safe additive instructions |
| `AGENTS.md` / `CLAUDE.md` | Project context section | Per-project conventions and rules |
| Skill `SKILL.md` files | Skills listing | On-demand capability descriptions |

**Dynamic (extension-based, runtime):**

| Mechanism | Scope | Timing | Effect |
|---|---|---|---|
| `before_agent_start` → `systemPrompt` | Full prompt | Per user prompt | Modify/append/replace system prompt |
| `promptSnippet` on tools | Tool description line | When tool set changes | Custom one-liner in "Available tools" |
| `promptGuidelines` on tools | Guidelines section | When tool set changes | Add behavioral bullets |
| `pi.setActiveTools()` | Tool list + guidelines | Immediate, next prompt | Add/remove tools (rebuilds prompt) |
| `resources_discover` event | Skills listing | Startup + reload | Inject additional skills from extensions |

**Per-Turn (message-based, not system prompt):**

| Mechanism | Timing | Effect |
|---|---|---|
| `before_agent_start` → `message` | Per user prompt | Inject custom message (becomes user role) |
| `context` event | Per LLM turn | Filter/inject/transform message array |
| `pi.sendMessage()` | Anytime | Inject custom message into conversation |

---

## Advanced Patterns

### Mode-Aware Tool Sets with Context Injection

The built-in plan mode uses defense in depth with three layers:

```typescript
// Layer 1: Tool set — remove write tools entirely (LLM doesn't know they exist)
if (planModeEnabled) {
  pi.setActiveTools(["read", "bash", "grep", "find", "ls"]);
}

// Layer 2: Bash guard — block destructive commands even within allowed tools
pi.on("tool_call", async (event) => {
  if (!planModeEnabled || event.toolName !== "bash") return;
  if (!isSafeCommand(event.input.command)) {
    return { block: true, reason: "Plan mode: command blocked" };
  }
});

// Layer 3: Context cleanup on mode exit — remove stale plan context messages
pi.on("context", async (event) => {
  if (planModeEnabled) return;
  return {
    messages: event.messages.filter(m => {
      if (m.customType === "plan-mode-context") return false;
      return true;
    }),
  };
});
```

A naive implementation would just change the tool set. But `bash` with `rm -rf` is technically a "read-only" tool by name, and stale context messages from a previous mode can confuse the LLM.

### Deferred System Prompt Application (Preset Pattern)

Store the active state and let `before_agent_start` read it, rather than setting the system prompt directly:

```typescript
// On apply — just store
activePresetName = name;
activePreset = preset;

// On each prompt — inject
pi.on("before_agent_start", async (event) => {
  if (activePreset?.instructions) {
    return {
      systemPrompt: `${event.systemPrompt}\n\n${activePreset.instructions}`,
    };
  }
});
```

This is better than calling any direct set method because `before_agent_start` fires on every prompt keeping the system prompt current, and other extensions can see and further modify the prompt in the chain.

### Compaction-Aware State

If your extension stores state in messages that might get compacted away, detect compaction and use a fallback:

```typescript
pi.on("session_start", async (_event, ctx) => {
  const entries = ctx.sessionManager.getBranch();
  const hasCompaction = entries.some(e => e.type === "compaction");

  if (hasCompaction) {
    // State before compaction is gone from messages
    // Fall back to appendEntry data or re-derive from remaining messages
    restoreFromAppendEntries(entries);
  } else {
    // Full message history available
    restoreFromToolResults(entries);
  }
});
```

### Dynamic Resource Injection

Extensions can ship their own skills, prompts, and themes:

```typescript
import { dirname, join } from "node:path";
import { fileURLToPath } from "node:url";

const baseDir = dirname(fileURLToPath(import.meta.url));

export default function (pi: ExtensionAPI) {
  pi.on("resources_discover", () => {
    return {
      skillPaths: [join(baseDir, "SKILL.md")],
      promptPaths: [join(baseDir, "dynamic.md")],
      themePaths: [join(baseDir, "dynamic.json")],
    };
  });
}
```

After `session_start`, the runner calls `emitResourcesDiscover()`. The system prompt is rebuilt after resources are extended, so new skills appear in the same prompt turn.

### The Complete Extension Initialization Sequence

```
1. Extension factory function runs
   ├─► pi.on() — register event handlers
   ├─► pi.registerTool() — register tools
   ├─► pi.registerCommand() — register commands
   ├─► pi.registerShortcut() — register shortcuts
   ├─► pi.registerFlag() — register CLI flags
   └─► pi.registerProvider() — queued (not yet applied)

2. ExtensionRunner created with all extensions

3. bindCore() — action methods become live
   ├─► pi.sendMessage, pi.setActiveTools, etc. now work
   └─► Queued provider registrations flushed to ModelRegistry

4. bindExtensions() — UI context and command context connected

5. session_start event fires
   └─► Extensions restore state from session

6. resources_discover event fires
   └─► Extensions provide additional skill/prompt/theme paths

7. System prompt rebuilt with new resources

8. Ready for first user prompt
```

**Critical:** During step 1, action methods (`sendMessage`, `setActiveTools`, etc.) will throw. You can only register handlers and tools during the factory function. Use `session_start` for anything that needs runtime access.

### The Full Context Surface Area

Everything the LLM sees on a given turn:

```
System prompt (built from all sources + before_agent_start mods)
  +
Message array (after context event filtering + convertToLlm):
  - Compaction summaries (user role)
  - Branch summaries (user role)
  - Historical user/assistant/toolResult messages
  - Bash execution results (user role, unless !! excluded)
  - Custom messages from extensions (user role)
  - Current prompt + before_agent_start injected messages
  +
Tool definitions:
  - name, description, parameter JSON schema
  - Only for active tools (pi.getActiveTools())
```

Understanding this complete surface area — and which levers control which parts — is the foundation of effective context engineering in pi.
