# pi-ai

## Package Metadata

| Field | Value |
|---|---|
| Name | `@gsd/pi-ai` |
| Version | `0.57.1` |
| Description | Unified LLM API (vendored from pi-mono) |
| License | (not specified in package.json) |
| Module type | ESM (`"type": "module"`) |
| Entry point | `./dist/index.js` |
| Types | `./dist/index.d.ts` |
| Additional exports | `./oauth`, `./bedrock-provider` |

## Purpose in the Ecosystem

`@gsd/pi-ai` is the single place where all LLM communication happens in the GSD monorepo. It presents a **unified streaming API** across more than a dozen providers — Anthropic, OpenAI, Google Gemini, Mistral, AWS Bedrock, Azure OpenAI, and many OpenAI-compatible endpoints — behind a common `stream()` / `complete()` surface. Callers never need to know which SDK or wire format a provider uses; they work with the same `Model`, `Context`, and `AssistantMessageEventStream` types regardless.

The package is consumed by `@gsd/pi-agent-core` (which wraps it in an event-driven agent loop) and by `@gsd/pi-coding-agent` (which builds the full CLI and session management on top of it). It is vendored from `pi-mono` and carries no `@gsd/*` sibling dependencies.

## Dependencies

| Package | Purpose |
|---|---|
| `@anthropic-ai/sdk` ^0.73.0 | Anthropic Messages API |
| `@aws-sdk/client-bedrock-runtime` ^3.983.0 | AWS Bedrock Converse API |
| `@google/genai` ^1.40.0 | Google Gemini generative AI |
| `@mistralai/mistralai` ^1.14.1 | Mistral AI conversations API |
| `@sinclair/typebox` ^0.34.41 | JSON Schema type system (re-exported) |
| `ajv` + `ajv-formats` | Runtime tool-argument validation |
| `openai` ^6.26.0 | OpenAI completions and responses APIs |
| `proxy-agent` ^6.5.0 | HTTP(S) proxy support |
| `undici` ^7.24.2 | Fetch implementation for Node.js |
| `zod-to-json-schema` ^3.24.6 | Converts Zod schemas to JSON Schema for tool definitions |

## Key Exports

The package exposes everything from its main entry point (`@gsd/pi-ai`). The most important groups are:

### Streaming and completion functions

```typescript
import { stream, complete, streamSimple, completeSimple } from '@gsd/pi-ai';
```

| Function | Signature | Description |
|---|---|---|
| `stream` | `(model, context, options?) => AssistantMessageEventStream` | Low-level streaming; uses provider-specific options |
| `complete` | `(model, context, options?) => Promise<AssistantMessage>` | Awaits the full response |
| `streamSimple` | `(model, context, options?) => AssistantMessageEventStream` | Streaming with thinking-level support |
| `completeSimple` | `(model, context, options?) => Promise<AssistantMessage>` | Non-streaming with thinking-level support |

### Model registry

```typescript
import { getModel, getModels, getProviders, calculateCost, modelsAreEqual, supportsXhigh } from '@gsd/pi-ai';
```

| Function | Description |
|---|---|
| `getModel(provider, modelId)` | Typed lookup — returns `Model<TApi>` for the given provider/model pair |
| `getModels(provider)` | All models registered for a provider |
| `getProviders()` | All known provider names |
| `calculateCost(model, usage)` | Fills in the `cost` fields of a `Usage` object |
| `modelsAreEqual(a, b)` | Compares by `id` and `provider` |
| `supportsXhigh(model)` | `true` for GPT-5.2/5.3/5.4 and Anthropic Opus 4.6 models |

### Types (core message system)

```typescript
import type {
  Model, Api, KnownApi, Provider, KnownProvider,
  Context, Message, UserMessage, AssistantMessage, ToolResultMessage,
  TextContent, ThinkingContent, ImageContent, ToolCall,
  Usage, StopReason,
  StreamOptions, SimpleStreamOptions,
  AssistantMessageEvent, AssistantMessageEventStream,
  Tool
} from '@gsd/pi-ai';
```

### Type builder (re-exported from TypeBox)

```typescript
import { Type, type Static, type TSchema } from '@gsd/pi-ai';

const MyParams = Type.Object({
  query: Type.String(),
  limit: Type.Optional(Type.Number())
});

type MyParamsType = Static<typeof MyParams>;
```

### Provider registration and API registry

```typescript
import { registerApiProvider, getApiProvider } from '@gsd/pi-ai';
```

Allows third-party code to register custom provider implementations that plug into the `stream()` dispatch table.

### OAuth types

```typescript
import type { OAuthAuthInfo, OAuthCredentials, OAuthLoginCallbacks,
              OAuthPrompt, OAuthProviderId, OAuthProviderInterface } from '@gsd/pi-ai/oauth';
```

### Utilities

```typescript
import { EventStream, validateToolArguments, parseJsonStream } from '@gsd/pi-ai';
import { getEnvApiKey } from '@gsd/pi-ai';
```

## Public API — Core Types

### `Model<TApi>`

```typescript
interface Model<TApi extends Api> {
  id: string;
  name: string;
  api: TApi;
  provider: Provider;
  baseUrl: string;
  reasoning: boolean;
  input: ('text' | 'image')[];
  cost: {
    input: number;   // USD per million input tokens
    output: number;  // USD per million output tokens
    cacheRead: number;
    cacheWrite: number;
  };
  contextWindow: number;
  maxTokens: number;
  headers?: Record<string, string>;
  compat?: OpenAICompletionsCompat | OpenAIResponsesCompat;
}
```

The type parameter `TApi` flows through to `StreamFunction` so TypeScript can enforce that the correct options type is used for each API.

### `Context`

```typescript
interface Context {
  systemPrompt?: string;
  messages: Message[];
  tools?: Tool[];
}
```

### `Message` union

```typescript
type Message = UserMessage | AssistantMessage | ToolResultMessage;
```

`UserMessage` carries `role: "user"` and `content: string | (TextContent | ImageContent)[]`.

`AssistantMessage` carries `role: "assistant"` and `content: (TextContent | ThinkingContent | ToolCall | ServerToolUseContent | WebSearchResultContent)[]`, plus `api`, `provider`, `model`, `usage`, `stopReason`, and `timestamp`.

`ToolResultMessage` carries `role: "toolResult"`, `toolCallId`, `toolName`, `content`, `details`, and `isError`.

### `AssistantMessageEvent`

The streaming interface emits a discriminated union:

```typescript
type AssistantMessageEvent =
  | { type: 'start'; partial: AssistantMessage }
  | { type: 'text_delta'; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: 'text_end'; contentIndex: number; content: string; partial: AssistantMessage }
  | { type: 'thinking_delta'; contentIndex: number; delta: string; partial: AssistantMessage }
  | { type: 'toolcall_end'; contentIndex: number; toolCall: ToolCall; partial: AssistantMessage }
  | { type: 'done'; reason: 'stop' | 'length' | 'toolUse'; message: AssistantMessage }
  | { type: 'error'; reason: 'aborted' | 'error'; error: AssistantMessage }
  // ... and several other granular events
```

### `StreamOptions` / `SimpleStreamOptions`

```typescript
interface StreamOptions {
  temperature?: number;
  maxTokens?: number;
  signal?: AbortSignal;
  apiKey?: string;
  transport?: 'sse' | 'websocket' | 'auto';
  cacheRetention?: 'none' | 'short' | 'long';
  sessionId?: string;
  onPayload?: (payload: unknown, model: Model<Api>) => unknown | undefined | Promise<unknown | undefined>;
  headers?: Record<string, string>;
  maxRetryDelayMs?: number;
  metadata?: Record<string, unknown>;
}

interface SimpleStreamOptions extends StreamOptions {
  reasoning?: ThinkingLevel;  // 'minimal' | 'low' | 'medium' | 'high' | 'xhigh'
  thinkingBudgets?: ThinkingBudgets;
}
```

### `Tool`

```typescript
interface Tool<TParameters extends TSchema = TSchema> {
  name: string;
  description: string;
  parameters: TParameters;
}
```

Tool schemas are expressed with TypeBox (`Type.Object(...)`) and validated at runtime via AJV.

## Known Providers

The `KnownProvider` type lists every provider the package ships built-in support for:

`amazon-bedrock`, `anthropic`, `google`, `google-gemini-cli`, `google-antigravity`, `google-vertex`, `openai`, `azure-openai-responses`, `openai-codex`, `github-copilot`, `xai`, `groq`, `cerebras`, `openrouter`, `vercel-ai-gateway`, `zai`, `mistral`, `minimax`, `minimax-cn`, `huggingface`, `opencode`, `opencode-go`, `kimi-coding`, `alibaba-coding-plan`, `ollama-cloud`.

## How to Use / Import

### Streaming a response

```typescript
import { getModel, streamSimple, type Context } from '@gsd/pi-ai';

const model = getModel('anthropic', 'claude-sonnet-4-6');

const context: Context = {
  systemPrompt: 'You are a helpful assistant.',
  messages: [{ role: 'user', content: 'Hello!', timestamp: Date.now() }]
};

const stream = streamSimple(model, context, { reasoning: 'medium' });

for await (const event of stream) {
  if (event.type === 'text_delta') {
    process.stdout.write(event.delta);
  }
  if (event.type === 'done') {
    console.log('\nUsage:', event.message.usage);
  }
}
```

### Awaiting a complete response

```typescript
import { getModel, completeSimple } from '@gsd/pi-ai';

const model = getModel('openai', 'gpt-4o');
const message = await completeSimple(model, context);
console.log(message.content);
```

### Defining and using tools

```typescript
import { Type, getModel, streamSimple } from '@gsd/pi-ai';

const searchTool = {
  name: 'web_search',
  description: 'Search the web for information.',
  parameters: Type.Object({
    query: Type.String({ description: 'The search query' })
  })
};

const context: Context = {
  messages: [{ role: 'user', content: 'Find the latest news.', timestamp: Date.now() }],
  tools: [searchTool]
};

const stream = streamSimple(getModel('anthropic', 'claude-sonnet-4-6'), context);
for await (const event of stream) {
  if (event.type === 'toolcall_end') {
    console.log('Tool called:', event.toolCall.name, event.toolCall.arguments);
  }
}
```

### Aborting a request

```typescript
const controller = new AbortController();
const stream = streamSimple(model, context, { signal: controller.signal });

// Later:
controller.abort();
```

## Key Internal Architecture

### API registry

`registerApiProvider(api, provider)` and `getApiProvider(api)` form a simple registry that maps API identifiers (e.g. `"anthropic-messages"`, `"openai-completions"`) to implementations. The built-in providers are registered via `register-builtins.ts`, which is imported side-effectfully at the top of `stream.ts`. This means the registry is populated automatically when any streaming function is called.

### Provider implementations (in `src/providers/`)

Each provider file exports a `StreamFunction` and a `streamSimple` wrapper. The main provider files are:

| File | Covers |
|---|---|
| `anthropic.ts` | `anthropic-messages` API |
| `openai-completions.ts` | `openai-completions` API (OpenAI, OpenRouter, Ollama, …) |
| `openai-responses.ts` | `openai-responses` API (newer OpenAI Responses endpoint) |
| `azure-openai-responses.ts` | Azure-hosted OpenAI Responses |
| `google.ts` | Google Gemini via `@google/genai` |
| `google-vertex.ts` | Google Vertex AI |
| `google-gemini-cli.ts` | Google Gemini CLI OAuth flow |
| `mistral.ts` | Mistral Conversations API |
| `amazon-bedrock.ts` | AWS Bedrock Converse stream |

### `EventStream<T, R>`

The streaming return value is an `EventStream` which is both an async iterable (for `for await`) and has a `.result()` method that returns a `Promise<AssistantMessage>` resolving to the complete response once the stream ends. This dual interface lets callers choose between incremental and batch consumption.

### `SimpleStreamOptions` and thinking levels

`streamSimple` translates the `reasoning` field (`'minimal' | 'low' | 'medium' | 'high' | 'xhigh'`) to provider-specific parameters. For Anthropic this maps to `thinking: { type: 'enabled', budget_tokens }`, for OpenAI it maps to `reasoning_effort`, and so on. The mapping is provider-specific and handled inside each provider's `streamSimple` wrapper via `simple-options.ts`.

### OpenAI compatibility layer

`OpenAICompletionsCompat` provides fine-grained overrides for the many providers that expose OpenAI-compatible endpoints but differ in small details — which field to use for max tokens (`max_tokens` vs `max_completion_tokens`), whether the `store` field is supported, whether tool results need a `name` field, etc. Most of these settings are auto-detected from `baseUrl`, but can be overridden per-model.
