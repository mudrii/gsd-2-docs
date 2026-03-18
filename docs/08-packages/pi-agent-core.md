# pi-agent-core

## Package Metadata

| Field | Value |
|---|---|
| Name | `@gsd/pi-agent-core` |
| Version | `0.57.1` |
| Description | General-purpose agent core (vendored from pi-mono) |
| License | (not specified in package.json) |
| Module type | ESM (`"type": "module"`) |
| Entry point | `./dist/index.js` |
| Types | `./dist/index.d.ts` |
| External dependencies | none (`"dependencies": {}`) |

## Purpose in the Ecosystem

`@gsd/pi-agent-core` provides the **agentic execution loop** that drives multi-turn LLM interactions with tool calls. It sits between the raw LLM streaming layer (`@gsd/pi-ai`) and higher-level applications like `@gsd/pi-coding-agent`. The package ships two primary abstractions:

- **`Agent`** — a stateful class that owns the conversation, model configuration, tool registry, and event bus. Applications interact with it through simple methods like `prompt()`, `steer()`, and `abort()`.
- **`agentLoop` / `agentLoopContinue`** — low-level generator functions that implement the turn-by-turn reasoning-and-tool-execution cycle used internally by `Agent`, but also callable directly for custom control flows.

The package has zero runtime dependencies of its own; all LLM communication goes through `@gsd/pi-ai`, which is listed as a peer/import dependency via TypeScript path aliases.

## Key Exports

```typescript
import { Agent, AgentOptions } from '@gsd/pi-agent-core';
import { agentLoop, agentLoopContinue } from '@gsd/pi-agent-core';
import type {
  AgentState, AgentEvent, AgentMessage, AgentTool, AgentToolResult,
  AgentContext, AgentLoopConfig, ThinkingLevel,
  BeforeToolCallContext, BeforeToolCallResult,
  AfterToolCallContext, AfterToolCallResult,
  StreamFn, ToolExecutionMode
} from '@gsd/pi-agent-core';
```

## Public API — `Agent` class

`Agent` is the main interface for embedding agentic behaviour in applications.

### Constructor

```typescript
const agent = new Agent({
  initialState: {
    systemPrompt: 'You are a helpful coding assistant.',
    model: getModel('anthropic', 'claude-sonnet-4-6'),
    thinkingLevel: 'medium',
    tools: [myTool],
  },
  steeringMode: 'one-at-a-time',   // default
  followUpMode: 'one-at-a-time',   // default
  sessionId: 'my-session-id',
});
```

All `AgentOptions` fields are optional:

| Option | Type | Description |
|---|---|---|
| `initialState` | `Partial<AgentState>` | Seed state; model defaults to Gemini 2.5 Flash Lite |
| `convertToLlm` | `(msgs: AgentMessage[]) => Message[]` | Custom message filter/transformer before LLM calls |
| `transformContext` | `async (msgs, signal?) => AgentMessage[]` | Context pruning or injection at the `AgentMessage` level |
| `steeringMode` | `'all' \| 'one-at-a-time'` | How steering messages are dequeued per turn |
| `followUpMode` | `'all' \| 'one-at-a-time'` | How follow-up messages are dequeued per turn |
| `streamFn` | `StreamFn` | Custom stream function (e.g. proxy backend) |
| `sessionId` | `string` | Provider session-cache token |
| `getApiKey` | `(provider) => Promise<string \| undefined>` | Dynamic API-key resolver for expiring OAuth tokens |
| `onPayload` | `(payload, model) => unknown` | Inspect / replace provider payloads before sending |
| `thinkingBudgets` | `ThinkingBudgets` | Per-level token budgets for token-budget providers |
| `transport` | `'sse' \| 'websocket' \| 'auto'` | Preferred transport; defaults to `'sse'` |
| `maxRetryDelayMs` | `number` | Cap on server-requested retry delay |

### State and configuration methods

```typescript
agent.setSystemPrompt('New system prompt');
agent.setModel(getModel('openai', 'gpt-4o'));
agent.setThinkingLevel('high');
agent.setTools([bashTool, editTool]);
agent.sessionId = 'new-session-id';
agent.thinkingBudgets = { low: 1024, medium: 4096 };
agent.setTransport('websocket');
agent.maxRetryDelayMs = 30000;

const state: AgentState = agent.state;
// state.messages, state.isStreaming, state.streamMessage, state.pendingToolCalls, state.error
```

### Sending messages

```typescript
// Send a text prompt
await agent.prompt('Refactor this function for readability.');

// Send with images
await agent.prompt('What is in this image?', [{ type: 'image', data: base64, mimeType: 'image/png' }]);

// Send pre-built message objects
await agent.prompt([
  { role: 'user', content: [{ type: 'text', text: 'Hello' }], timestamp: Date.now() }
]);
```

`prompt()` throws if the agent is already streaming. Use `steer()` or `followUp()` to queue messages for delivery during or after the current run.

### Interruption and queuing

```typescript
// Interrupt: injects message after current tool execution, skips remaining tools
agent.steer({ role: 'user', content: [{ type: 'text', text: 'Stop and summarise.' }], timestamp: Date.now() });

// Follow-up: delivers message after agent would otherwise stop
agent.followUp({ role: 'user', content: [{ type: 'text', text: 'Now run tests.' }], timestamp: Date.now() });

agent.clearSteeringQueue();
agent.clearFollowUpQueue();
agent.clearAllQueues();
agent.hasQueuedMessages(); // → boolean
```

### Aborting and waiting

```typescript
agent.abort();              // signals AbortController; current stream terminates gracefully
await agent.waitForIdle();  // resolves when no prompt is running
agent.reset();              // clears messages, error, stream state, and both queues
```

### Tool call hooks

```typescript
agent.setBeforeToolCall(async (ctx, signal) => {
  if (ctx.toolCall.name === 'delete_file' && ctx.args.path === '/etc/passwd') {
    return { block: true, reason: 'Refusing destructive system file modification.' };
  }
});

agent.setAfterToolCall(async (ctx, signal) => {
  console.log('Tool finished:', ctx.toolCall.name, ctx.result);
  // Return overrides or undefined to keep original
  return { isError: false }; // Suppress errors from non-critical tools
});
```

### Event subscription

```typescript
const unsubscribe = agent.subscribe((event) => {
  switch (event.type) {
    case 'agent_start':   console.log('Started'); break;
    case 'message_start': console.log('Message streaming:', event.message.role); break;
    case 'message_update': /* incremental delta */ break;
    case 'message_end':  console.log('Message done'); break;
    case 'tool_execution_start': console.log('Running tool:', event.toolName); break;
    case 'tool_execution_update': /* partial result */ break;
    case 'tool_execution_end': console.log('Tool done:', event.result); break;
    case 'turn_end': /* assistant response + all its tool results */ break;
    case 'agent_end': console.log('Agent idle, new messages:', event.messages.length); break;
  }
});

// Later:
unsubscribe();
```

## Public API — `AgentEvent`

```typescript
type AgentEvent =
  | { type: 'agent_start' }
  | { type: 'agent_end'; messages: AgentMessage[] }
  | { type: 'turn_start' }
  | { type: 'turn_end'; message: AgentMessage; toolResults: ToolResultMessage[] }
  | { type: 'message_start'; message: AgentMessage }
  | { type: 'message_update'; message: AgentMessage; assistantMessageEvent: AssistantMessageEvent }
  | { type: 'message_end'; message: AgentMessage }
  | { type: 'tool_execution_start'; toolCallId: string; toolName: string; args: any }
  | { type: 'tool_execution_update'; toolCallId: string; toolName: string; args: any; partialResult: any }
  | { type: 'tool_execution_end'; toolCallId: string; toolName: string; result: any; isError: boolean };
```

A **turn** is one assistant response plus all tool executions triggered by that response. The `agent_end` event carries every new `AgentMessage` added since the last `prompt()` call.

## Public API — `AgentTool<TParameters, TDetails>`

```typescript
import { Type } from '@gsd/pi-ai';

const greetTool: AgentTool<typeof params, { greeting: string }> = {
  name: 'greet',
  description: 'Returns a greeting message.',
  label: 'Greet',
  parameters: Type.Object({ name: Type.String() }),
  execute: async (toolCallId, { name }, signal, onUpdate) => {
    const greeting = `Hello, ${name}!`;
    // Stream partial results as the tool works
    onUpdate?.({ content: [{ type: 'text', text: greeting }], details: { greeting } });
    return { content: [{ type: 'text', text: greeting }], details: { greeting } };
  }
};

agent.setTools([greetTool]);
```

Tool arguments are validated by AJV against `parameters` before `execute` is called. Validation errors surface as blocked tool calls with an error `ToolResultMessage`.

## Public API — `agentLoop` and `agentLoopContinue`

These lower-level functions are useful when you need to own the event loop directly.

```typescript
import { agentLoop, agentLoopContinue } from '@gsd/pi-agent-core';
import { streamSimple } from '@gsd/pi-ai';

const controller = new AbortController();

const stream = agentLoop(
  [{ role: 'user', content: [{ type: 'text', text: 'List files in cwd.' }], timestamp: Date.now() }],
  { systemPrompt: 'You are a coding assistant.', messages: [], tools: [lsTool] },
  {
    model: getModel('anthropic', 'claude-sonnet-4-6'),
    convertToLlm: (msgs) => msgs.filter(m => m.role !== 'notification'),
  },
  controller.signal,
  streamSimple,
);

for await (const event of stream) {
  // Handle events
}

const newMessages = await stream.result();
```

`agentLoopContinue` works identically but does not add new prompt messages — it resumes from the existing context. The last message in context must convert to a `user` or `toolResult` role.

## Public API — `AgentMessage` and `CustomAgentMessages`

`AgentMessage` is the union of standard LLM `Message` types and any custom message types registered by the application:

```typescript
// Declare custom message types in your app
declare module '@gsd/pi-agent-core' {
  interface CustomAgentMessages {
    notification: { role: 'notification'; text: string };
    artifact: { role: 'artifact'; id: string; content: string };
  }
}

// Now 'notification' and 'artifact' messages can be placed in agent.state.messages
// but are filtered out by convertToLlm before LLM calls
```

## Key Internal Architecture

### The `runLoop` function

The heart of the package is the private `runLoop` function shared by `agentLoop` and `agentLoopContinue`. It runs two nested loops:

**Outer loop** — handles follow-up messages. After the agent would otherwise stop, it polls `getFollowUpMessages()`. If messages are returned, they become pending input and the outer loop continues.

**Inner loop** — processes one complete turn at a time:
1. Injects any pending steering messages into context.
2. Calls `streamAssistantResponse()`, which applies `transformContext`, calls `convertToLlm`, and streams the LLM response.
3. If the response contains tool calls, delegates to `executeToolCalls()`.
4. After each tool, polls `getSteeringMessages()`. If messages are returned, remaining tool calls are skipped.
5. Emits `turn_end`.

### Tool execution modes

Tool calls within a single assistant message can be executed in two modes, set via `AgentLoopConfig.toolExecution`:

- `"sequential"` (default-like) — each tool is prepared, `beforeToolCall`-checked, executed, `afterToolCall`-processed, and its result emitted before the next tool starts.
- `"parallel"` — all tools are prepared and `beforeToolCall`-checked sequentially, then the non-blocked tools run concurrently via `Promise` parallelism. Results are still emitted in the original source order.

### Message boundary: `AgentMessage` vs `Message`

`AgentMessage` is a superset of the LLM's `Message` type. The loop holds the full `AgentMessage[]` in `context.messages`. At the point of each LLM call, it calls `convertToLlm(messages)` which must return a filtered/transformed `Message[]` that the provider API will accept. Custom message types (notifications, artifacts, UI events) survive in `AgentMessage[]` for application use but are stripped before the wire call.

### Proxy utilities

The package exports a `proxy` module with helpers for constructing `streamFn` wrappers that communicate with a remote agent over a transport (used by Studio and RPC mode in `pi-coding-agent`).

## Dependencies on Other Packages

`@gsd/pi-agent-core` depends on `@gsd/pi-ai` for:
- `streamSimple` and `EventStream`
- `Model`, `Message`, `Context`, `Tool`, `ToolResultMessage`, `AssistantMessage`, `AssistantMessageEvent`
- `validateToolArguments`
- `getModel` (used only in `Agent` to set a default model)
