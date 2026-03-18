# pi-coding-agent

## Package Metadata

| Field | Value |
|---|---|
| Name | `@gsd/pi-coding-agent` |
| Version | `0.57.1` |
| Description | Coding agent CLI (vendored from pi-mono) |
| License | (not specified in package.json) |
| Module type | ESM (`"type": "module"`) |
| Entry point | `./dist/index.js` |
| Types | `./dist/index.d.ts` |
| CLI name | `pi` (from `piConfig.name`) |
| Config directory | `.pi` (from `piConfig.configDir`) |

## Purpose in the Ecosystem

`@gsd/pi-coding-agent` is the top-level application package. It brings together all other packages in the GSD monorepo into a usable product: a full-featured **AI coding assistant** that runs in the terminal. It owns everything above the agent loop abstraction:

- The interactive TUI (built on `@gsd/pi-tui`)
- Session management, compaction, and branching
- The extension system (loading, discovery, lifecycle)
- A built-in set of filesystem tools (`bash`, `read`, `write`, `edit`, `find`, `grep`, `ls`)
- Authentication and credential storage
- Model registry and discovery
- RPC and print run modes for non-interactive use
- The `pi` CLI entry point

## Dependencies

| Package | Purpose |
|---|---|
| `@mariozechner/jiti` ^2.6.2 | TypeScript/ESM extension loading at runtime |
| `@silvia-odwyer/photon-node` ^0.3.4 | Image processing and auto-resize |
| `chalk` ^5.5.0 | CLI output formatting |
| `diff` ^8.0.2 | File diff computation for edit tool |
| `extract-zip` ^2.0.1 | Unzipping binary downloads (fd, rg) |
| `file-type` ^21.1.1 | MIME detection for file attachments |
| `glob` ^13.0.1 | Pattern-based file discovery |
| `hosted-git-info` ^9.0.2 | Parse git URLs for package installation |
| `ignore` ^7.0.5 | `.gitignore`-style filtering |
| `marked` ^15.0.12 | Markdown parsing |
| `minimatch` ^10.2.3 | Glob matching |
| `proper-lockfile` ^4.1.2 | File-based locking for concurrent access |
| `sql.js` ^1.14.1 | SQLite for conversation storage |
| `strip-ansi` ^7.1.0 | ANSI stripping for output truncation |
| `undici` ^7.24.2 | HTTP client |
| `yaml` ^2.8.2 | YAML parsing for skill frontmatter |

## Key Exports

The package exports a large public surface used both by the CLI entry point and by integrators who want to embed the coding agent programmatically.

### Entry points

```typescript
import { main } from '@gsd/pi-coding-agent';
// Main CLI function — takes process.argv slice
await main(process.argv.slice(2));
```

### SDK — `createAgentSession`

The primary embedding API:

```typescript
import { createAgentSession, type CreateAgentSessionOptions, type CreateAgentSessionResult } from '@gsd/pi-coding-agent';

const { session, modelFallbackMessage } = await createAgentSession({
  model: getModel('anthropic', 'claude-sonnet-4-6'),
  thinkingLevel: 'medium',
  sessionManager,
  authStorage,
  modelRegistry,
  resourceLoader,
  tools: [bashTool, editTool],
});
```

`AgentSession` is the object returned inside `session`. It manages the full lifecycle of one coding conversation.

### Run modes

```typescript
import { InteractiveMode, runPrintMode, runRpcMode } from '@gsd/pi-coding-agent';
import type { InteractiveModeOptions, PrintModeOptions } from '@gsd/pi-coding-agent';

// Interactive TUI mode
const mode = new InteractiveMode(session, options);
await mode.run();

// Non-interactive print mode (streaming text output)
await runPrintMode(session, { mode: 'text', messages: ['Hello'] });

// JSON-RPC mode (machine-readable stdin/stdout protocol)
await runRpcMode(session);
```

### RPC client

```typescript
import { RpcClient, type RpcClientOptions, type RpcCommand, type RpcResponse, type RpcSessionState } from '@gsd/pi-coding-agent';
```

`RpcClient` connects to a running `pi` process over a socket or pipe and sends JSON-RPC commands.

### Session management

```typescript
import {
  SessionManager, SessionInfo, SessionHeader, SessionEntry,
  SessionMessageEntry, CompactionEntry, BranchSummaryEntry,
  parseSessionEntries, migrateSessionEntries, buildSessionContext,
  CURRENT_SESSION_VERSION
} from '@gsd/pi-coding-agent';

// Create a new session
const manager = SessionManager.create(process.cwd());

// Open an existing session by path
const manager = SessionManager.open('/path/to/session.jsonl');

// Continue the most recent session
const manager = SessionManager.continueRecent(process.cwd());

// List all sessions
const sessions: SessionInfo[] = await SessionManager.list(process.cwd());
const allSessions: SessionInfo[] = await SessionManager.listAll();
```

### Authentication and credential storage

```typescript
import {
  AuthStorage, FileAuthStorageBackend, InMemoryAuthStorageBackend,
  type ApiKeyCredential, type OAuthCredential, type AuthCredential, type AuthStorageBackend
} from '@gsd/pi-coding-agent';

const auth = AuthStorage.create();
auth.setApiKey('anthropic', 'sk-ant-...');
const key = auth.getApiKey('anthropic');
```

### Tool factories

```typescript
import {
  bashTool, editTool, readTool, writeTool, findTool, grepTool, lsTool,
  codingTools,              // all tools combined
  readOnlyTools,            // bash + read + find + grep + ls
  createBashTool,           // factory with custom cwd
  createEditTool,
  createCodingTools,
  createReadOnlyTools,
  hashlineEditTool,         // line-number mode edit tool
  hashlineReadTool,         // line-number mode read tool
  hashlineCodingTools,
} from '@gsd/pi-coding-agent';
```

### Compaction

```typescript
import {
  compact, shouldCompact, generateSummary, generateBranchSummary,
  calculateContextTokens, estimateTokens,
  DEFAULT_COMPACTION_SETTINGS, type CompactionSettings
} from '@gsd/pi-coding-agent';
```

### Extension system

```typescript
import {
  createExtensionRuntime, discoverAndLoadExtensions, ExtensionRunner,
  wrapToolWithExtensions, wrapToolsWithExtensions,
  type Extension, type ExtensionFactory, type ExtensionAPI,
  type ExtensionContext, type ExtensionRuntime,
  type LoadExtensionsResult,
} from '@gsd/pi-coding-agent';
```

### Skills

```typescript
import { loadSkills, loadSkillsFromDir, formatSkillsForPrompt, type Skill } from '@gsd/pi-coding-agent';
```

### Model registry and discovery

```typescript
import {
  ModelRegistry, ModelDiscoveryCache,
  getDiscoverableProviders, getDiscoveryAdapter,
  type DiscoveredModel, type DiscoveryResult, type ProviderDiscoveryAdapter
} from '@gsd/pi-coding-agent';
```

### UI components (for extensions)

```typescript
import {
  AssistantMessageComponent, UserMessageComponent, ToolExecutionComponent,
  FooterComponent, ModelSelectorComponent, SessionSelectorComponent,
  LoginDialogComponent, SettingsSelectorComponent, ThemeSelectorComponent,
  BashExecutionComponent, Markdown as MarkdownComponent,
  renderDiff, type RenderDiffOptions,
} from '@gsd/pi-coding-agent';
```

### Theme

```typescript
import {
  initTheme, Theme, getMarkdownTheme, getSelectListTheme, getSettingsListTheme,
  highlightCode, getLanguageFromPath, type ThemeColor
} from '@gsd/pi-coding-agent';
```

### Config paths

```typescript
import { getAgentDir, VERSION } from '@gsd/pi-coding-agent';
// getAgentDir() → ~/.pi/agent  (or $PI_CODING_AGENT_DIR)
// VERSION       → "0.57.1"
```

## Config Path Functions (from `config.ts`)

| Function | Returns |
|---|---|
| `getAgentDir()` | `~/.pi/agent/` (or `$PI_CODING_AGENT_DIR`) |
| `getModelsPath()` | `~/.pi/agent/models.json` |
| `getAuthPath()` | `~/.pi/agent/auth.json` |
| `getSettingsPath()` | `~/.pi/agent/settings.json` |
| `getSessionsDir()` | `~/.pi/agent/sessions/` |
| `getBlobsDir()` | `~/.pi/agent/blobs/` |
| `getThemesDir()` | Built-in themes directory |
| `getPackageDir()` | Base dir for shipped assets (handles Bun binary vs Node.js) |
| `detectInstallMethod()` | `'bun-binary' \| 'npm' \| 'pnpm' \| 'yarn' \| 'bun' \| 'unknown'` |

The `APP_NAME` constant (`"pi"`) and `CONFIG_DIR_NAME` (`".pi"`) are read from `package.json`'s `piConfig` field at startup.

## CLI Interface

The `main(args)` function processes `process.argv.slice(2)` through a two-pass argument parser:

**First pass** — picks up `--extension`, `--skills`, `--prompt-templates`, `--themes`, and `--no-*` flags to early-load resources before the second parse.

**Second pass** — after extensions register their custom flags, runs the full parse.

Key flags include:

| Flag | Description |
|---|---|
| `--model <pattern>` | Select model by pattern (e.g. `claude`, `openai/gpt-4o`) |
| `--provider <name>` | Restrict model lookup to one provider |
| `--models <patterns>` | Set a model scope for Ctrl+P cycling |
| `--thinking <level>` | Override thinking level |
| `--print` / `-p` | Non-interactive print mode |
| `--mode rpc` | JSON-RPC stdin/stdout mode |
| `--session <id\|path>` | Resume or open a specific session |
| `--continue` / `-c` | Continue the most recent session |
| `--resume` | Show session picker TUI |
| `--no-session` | Run without persisting session |
| `--no-tools` | Start with no built-in tools |
| `--api-key <key>` | Runtime API key (not persisted) |
| `--offline` | Disable network calls except LLM |
| `--version` / `-v` | Print version and exit |
| `--help` / `-h` | Print help and exit |
| `install/remove/update/list` | Package management subcommands |
| `config` | Open interactive config selector |

File arguments prefixed with `@` are expanded into the initial message context:

```bash
pi "Explain this file" @src/main.ts
```

Piped stdin is automatically detected (non-TTY) and forces print mode:

```bash
echo "Summarise this" | pi
```

## Key Internal Architecture

### `AgentSession` (in `core/agent-session.ts`)

`AgentSession` is the central stateful object. It wires together:
- `Agent` (from `@gsd/pi-agent-core`) for LLM interaction
- `SessionManager` for JSONL-based conversation persistence
- `ModelRegistry` for model lookup and API key resolution
- `SettingsManager` for user and project configuration
- The extension runner for event routing to extensions
- Compaction logic triggered when context window nears capacity

### Extension system

Extensions are Node.js/TypeScript modules that export an `ExtensionFactory`. They are loaded via `jiti` (TypeScript-aware module loader) and receive an `ExtensionAPI` object that exposes hooks for:
- Tool registration and wrapping
- Slash commands
- Custom UI widgets and editors
- Message rendering overrides
- Session lifecycle events
- Footer data contributions

Extensions are discovered from:
1. `~/.pi/agent/extensions/` (user-global)
2. `.pi/extensions/` (project-local)
3. Paths provided via `--extension`

### Session persistence

Conversations are stored as JSONL files in `~/.pi/agent/sessions/<cwd-hash>/`. Each line is a typed `SessionEntry`:
- `SessionInfoEntry` — header with metadata
- `SessionMessageEntry` — a single message (user, assistant, or tool result)
- `CompactionEntry` — context compaction summary
- `BranchSummaryEntry` — branch/fork summary
- `ModelChangeEntry`, `ThinkingLevelChangeEntry` — mid-session model changes

### Compaction

When `calculateContextTokens(messages) > threshold`, `shouldCompact()` returns `true` and the session triggers `compact()`. This:
1. Finds a `cutPoint` before the oldest messages that can be dropped.
2. Calls the LLM to generate a `generateSummary()` of the dropped portion.
3. Inserts a `CompactionEntry` that replaces the dropped messages.
4. Continues from the summary.

### Tools

The six built-in coding tools are:

| Tool | Description |
|---|---|
| `bashTool` | Executes shell commands with configurable interceptor rules |
| `readTool` | Reads files with line range support and output truncation |
| `writeTool` | Creates or overwrites files |
| `editTool` | Applies targeted string-replacement edits (old_string → new_string) |
| `findTool` | Recursive file discovery matching patterns |
| `grepTool` | Content search using ripgrep |
| `lsTool` | Directory listing with size and type info |

All tools accept an `operations` parameter for dependency injection (used in tests), and support `TruncationOptions` to limit output size.

`hashlineEditTool` and `hashlineReadTool` are alternative edit/read tools that use line-number prefixes in their format for higher edit precision with long files.

## How to Use / Import (Programmatic SDK)

```typescript
import { createAgentSession, createCodingTools, AuthStorage, ModelRegistry,
         DefaultResourceLoader, SettingsManager } from '@gsd/pi-coding-agent';
import { getAgentDir } from '@gsd/pi-coding-agent';
import { getModel } from '@gsd/pi-ai';

const cwd = process.cwd();
const agentDir = getAgentDir();
const authStorage = AuthStorage.create();
const settingsManager = SettingsManager.create(cwd, agentDir);
const modelRegistry = new ModelRegistry(authStorage, `${agentDir}/models.json`);
const resourceLoader = new DefaultResourceLoader({ cwd, agentDir, settingsManager });

await resourceLoader.reload();

const { session } = await createAgentSession({
  model: getModel('anthropic', 'claude-sonnet-4-6'),
  thinkingLevel: 'medium',
  authStorage,
  modelRegistry,
  resourceLoader,
  tools: createCodingTools({ cwd }),
});

// Subscribe to agent events
session.agent.subscribe((event) => {
  if (event.type === 'message_update') {
    // stream content to your UI
  }
});

// Send a prompt
await session.prompt('Refactor the authentication module to use async/await.');
```

## Dependencies on Other Packages

`@gsd/pi-coding-agent` depends on both `@gsd/pi-ai` (for `getModel`, `streamSimple`, all LLM types) and `@gsd/pi-agent-core` (for `Agent`, `agentLoop`, `AgentTool`, all event types). It also depends on `@gsd/pi-tui` for the interactive TUI (the `@gsd/pi-tui` dependency is not in `package.json` as a workspace dep — it is resolved by the monorepo build, and its types are used throughout).
