# GSD-2 Architecture Overview

**Package**: `gsd-pi` (npm)
**Version**: 2.58.0
**License**: MIT
**Repository**: https://github.com/gsd-build/gsd-2
**Runtime**: Node.js >= 22.0.0
**Language**: TypeScript (ESM, target ES2022) + Rust (native engine)

GSD (Get Shit Done) is a TypeScript coding agent built on the Pi SDK. It embeds the Pi coding agent and extends it with a workflow engine, auto-mode state machine, and project management primitives. The CLI binary is `gsd` (aliased as `gsd-cli`), both pointing to `dist/loader.js`.

---

## 1. Monorepo Structure

GSD-2 uses npm workspaces with the following layout:

```
gsd-2/
  src/                      GSD application source (compiled to dist/)
  dist/                     Compiled output
  pkg/                      PI_PACKAGE_DIR shim (piConfig + theme assets)
  packages/
    native/                 @gsd/native      — TypeScript bindings for Rust N-API engine
    pi-agent-core/          @gsd/pi-agent-core (v0.57.1) — general-purpose agent core (vendored from pi-mono)
    pi-ai/                  @gsd/pi-ai (v0.57.1)         — unified LLM API (vendored from pi-mono)
    pi-coding-agent/        @gsd/pi-coding-agent (v2.58.0) — coding agent CLI (vendored from pi-mono)
    pi-tui/                 @gsd/pi-tui (v0.57.1)        — terminal UI library (vendored from pi-mono)
    rpc-client/             @gsd-build/rpc-client (v2.52.0) — standalone RPC protocol v2 client SDK (zero internal deps)
    mcp-server/             @gsd-build/mcp-server (v2.52.0) — MCP server for GSD orchestration (Claude Code, Cursor, etc.)
    daemon/                 @gsd-build/daemon (v0.1.0) — background daemon, Discord bot, launchd integration
  native/
    Cargo.toml              Rust workspace root
    crates/
      engine/               Main N-API addon (napi-rs) — all native modules
      grep/                 Standalone grep library crate
      ast/                  AST/structural code search crate
    npm/                    Platform-specific prebuilt binary packages
      darwin-arm64/         @gsd-build/engine-darwin-arm64
      darwin-x64/           @gsd-build/engine-darwin-x64
      linux-arm64-gnu/      @gsd-build/engine-linux-arm64-gnu
      linux-x64-gnu/        @gsd-build/engine-linux-x64-gnu
      win32-x64-msvc/       @gsd-build/engine-win32-x64-msvc
  studio/                   @gsd/studio — Electron desktop app (React 19, Zustand, electron-vite)
  vscode-extension/         GSD-2 VS Code extension (v0.3.0, publisher: FluxLabs) — sidebar dashboard, @gsd chat participant, activity feed, code lens, session forking, 33 commands
  web/                      gsd-web (v0.1.0) — Next.js 16 web interface (React 19, Radix UI, CodeMirror, xterm, Tailwind CSS 4)
  gsd-orchestrator/         Orchestrator skill — workflow templates, references
  src/resources/
    extensions/             Bundled extensions (synced to ~/.gsd/agent/extensions/)
    agents/                 Bundled agent definitions (scout, researcher, worker)
    GSD-WORKFLOW.md         Manual bootstrap protocol
    AGENTS.md               Agent routing instructions
```

### Workspace Package Roles

| Package | Version | Role |
|---------|---------|------|
| `@gsd/native` | 0.1.0 | TypeScript wrapper around Rust N-API addon; provides `grep`, `glob`, `ps`, `highlight`, `ast`, `diff`, `text`, `html`, `image`, `fd`, `clipboard`, `git`, `gsd_parser` |
| `@gsd/pi-agent-core` | 0.57.1 | General-purpose agent core: session management, tool dispatch, extension loading |
| `@gsd/pi-ai` | 0.57.1 | Unified LLM API across providers (Anthropic, OpenAI, Google, xAI, Mistral, Groq, OpenRouter, Bedrock, Anthropic Vertex, custom OpenAI-compatible) |
| `@gsd/pi-coding-agent` | 2.58.0 | Full coding agent: `InteractiveMode`, `SessionManager`, `AuthStorage`, `ModelRegistry`, `SettingsManager`, RPC/MCP modes |
| `@gsd/pi-tui` | 0.57.1 | Terminal UI primitives used by the interactive mode |
| `@gsd-build/rpc-client` | 2.52.0 | Standalone RPC client SDK — zero internal dependencies; RPC protocol v2 types, JSONL utilities, `RpcClient` class |
| `@gsd-build/mcp-server` | 2.52.0 | MCP server exposing GSD orchestration tools for Claude Code, Cursor, and other MCP clients; `SessionManager`, `createMcpServer` |
| `@gsd-build/daemon` | 0.1.0 | Background daemon: project scanning, Discord bot (discord.js v14) with orchestrator, event bridge, channel manager, launchd integration, session lifecycle management |

All four `pi-*` packages are vendored from the upstream `pi-mono` repository. They are compiled separately (`npm run build:pi`) before the main GSD build.

---

## 2. Bootstrap Flow (loader.ts -> cli.ts)

The startup uses a two-file pattern to ensure environment variables are set before any Pi SDK code evaluates.

### loader.ts (entry point: `dist/loader.js`)

1. **Fast-path exits**: `--version` and `--help` are handled immediately, reading `package.json` once, with zero SDK imports.
2. **PI_PACKAGE_DIR setup**: Points to `pkg/` (not project root). This prevents Pi's theme resolution from colliding with GSD's `src/` directory. The `pkg/` shim contains only `piConfig` and theme assets.
3. **Environment variables set**:
   - `PI_PACKAGE_DIR` = `<gsdRoot>/pkg/`
   - `PI_SKIP_VERSION_CHECK` = `1` (GSD runs its own update check)
   - `GSD_CODING_AGENT_DIR` = `~/.gsd/agent/`
   - `GSD_VERSION` = package version
   - `GSD_BIN_PATH` = path to this loader (used by subagent to spawn gsd)
   - `GSD_WORKFLOW_PATH` = path to bundled GSD-WORKFLOW.md
   - `GSD_BUNDLED_EXTENSION_PATHS` = serialized list of enabled extension entry points
   - `NODE_PATH` = prepends gsd's `node_modules` so extensions loaded via jiti can resolve their dependencies
4. **First-launch banner**: If `~/.gsd/` does not exist, prints a branded welcome logo to stderr.
5. **Extension discovery**: Scans `src/resources/extensions/` (or `dist/resources/extensions/`), remaps paths to `~/.gsd/agent/extensions/`, filters by the unified extension registry, sorts extensions in topological dependency order, and serializes to `GSD_BUNDLED_EXTENSION_PATHS`.
6. **Proxy setup**: If `HTTP_PROXY`/`HTTPS_PROXY` env vars are set, lazy-loads `undici` and configures a global proxy dispatcher.
7. **Workspace package linking**: Ensures `@gsd/*` packages are symlinked (or copied on Windows) from `packages/` into `node_modules/@gsd/`. Validates that critical packages (`@gsd/pi-coding-agent`) are resolvable.
8. **Dynamic import**: `await import('./cli.js')` — all Pi SDK imports happen here, after environment is fully configured.

### cli.ts (main application logic)

1. **Startup timings**: When `GSD_STARTUP_TIMING=1`, records per-phase timing via `startup-timings.ts` and prints a summary to stderr on completion.
2. **CLI argument parsing**: Detects `--mode`, `--print`, `--continue`, `--model`, `--extension`, `--worktree`, `--bare`, `--resume`, and subcommands.
3. **Resource skew check**: Verifies that synced resources in `~/.gsd/agent/` are not from a newer GSD version than the running binary. Uses a content fingerprint (SHA-256 computed from file paths and sizes) to detect same-version content changes.
4. **TTY gate**: Requires a terminal for interactive mode; suggests `--print`, `--mode rpc`, `--mode mcp`, or `--mode text` for non-TTY environments.
5. **Subcommand dispatch**: `config`, `update`, `sessions`, `headless`, `worktree` each have dedicated handlers.
6. **Tool bootstrap**: Provisions managed binaries (`fd`, `rg`) into `~/.gsd/agent/bin/`.
7. **Auth and credential hydration**: Creates `AuthStorage` from `~/.gsd/agent/auth.json`, loads stored env keys (Brave, Context7, Jina, Tavily, Slack, Discord, Telegram, Groq, Ollama, custom OpenAI), migrates Pi credentials.
8. **Model registry**: Loads `models.json`, validates the configured default model, auto-selects a fallback if the configured model no longer exists (preference order: Pi default, GPT-5.4, any OpenAI, Claude Opus 4-6, any Anthropic, first available).
9. **Onboarding**: On first launch (no LLM provider configured), runs a clack-based interactive wizard: provider selection (OAuth or API key), web search setup, remote questions (Discord/Slack/Telegram), optional tool keys.
10. **Resource sync**: `initResources()` copies bundled extensions, agents, and skills to `~/.gsd/agent/` (skipped when version + content fingerprint match).
11. **SQLite database initialization**: The GSD extension opens (or creates) a SQLite database (via `sql.js`) for project state. Schema migrations run on open. Auto-mode gates on SQLite availability.
12. **Startup model validation**: After `createAgentSession()` (so extension-provided models are registered), `validateConfiguredModel()` in `startup-model-validation.ts` checks that the configured default model still exists in the registry. If not, selects a fallback without overwriting valid extension-provided models.
13. **Session creation**: Per-directory session storage under `~/.gsd/sessions/--<encoded-cwd>--/`. Supports `SessionManager.create()`, `.continueRecent()`, and `.open()`.
14. **Agent session**: Creates session via `createAgentSession()` with auth, model registry, settings, session manager, and resource loader.
15. **Interactive mode**: Instantiates `InteractiveMode` and calls `run()`.

---

## 3. Key Architectural Decisions

### State Lives on Disk and in SQLite

`.gsd/` in the project root is the source of truth. Starting with v2.44.0, tool-driven write-side state transitions use atomic SQLite tool calls instead of direct markdown mutation. `deriveStateFromDb()` derives workflow state from the database, with disk-to-DB reconciliation when milestones exist on disk but are missing from the database. Disk artifacts (STATE.md, roadmaps, plans) remain the fallback when the database is unavailable.

### Two-File Loader Pattern

`loader.ts` sets all environment variables with zero SDK imports, then dynamically imports `cli.ts`. This ensures `PI_PACKAGE_DIR` is set before any Pi SDK code evaluates. Without this, Pi's `config.js` would read the wrong package directory and resolve incorrect branding/theme paths.

### `pkg/` Shim Directory

`PI_PACKAGE_DIR` points to `pkg/` (not the project root) to avoid Pi's theme resolution colliding with GSD's `src/` directory. The `pkg/` shim contains only `piConfig` (which sets `name: "gsd"` and `configDir: ".gsd"`) and theme assets.

### Always-Overwrite Resource Sync

Bundled extensions, agents, and skills are synced to `~/.gsd/agent/` on every launch, not just first run. This means `npm update -g gsd-pi` takes effect immediately. The sync is skipped when both the GSD version and a SHA-256 content fingerprint (computed from file paths and sizes) match the manifest, costing ~1ms in the common case.

### Lazy Provider Loading

LLM provider SDKs (Anthropic, OpenAI, Google, etc.) are lazy-loaded on first use rather than imported at startup. This reduces cold-start time — only the provider you connect to gets loaded.

### Fresh Session Per Unit

Every auto-mode dispatch creates a new agent session. The LLM starts with a clean context window containing only the pre-inlined artifacts it needs. This prevents quality degradation from context accumulation.

### NODE_PATH Injection

GSD prepends its own `node_modules` to `NODE_PATH` and calls `Module._initPaths()` to force Node to re-evaluate module search paths. Without this, extensions loaded via jiti (from Pi's location) cannot resolve dependencies like `playwright` that live in GSD's `node_modules`.

### Single-Writer Engine

The single-writer engine (v2/v3, introduced in v2.46.0) enforces discipline on state transitions. State machine guards prevent invalid transitions, actor identity tracks which component initiated a write, and operations are reversible. The `workflow-logger` is wired into engine, tool, manifest, and reconcile paths to provide an audit trail. All write-side DB mutations flow through typed tool calls with transaction isolation.

---

## 4. Application Paths

Defined in `src/app-paths.ts`:

| Path | Value | Purpose |
|------|-------|---------|
| `appRoot` | `~/.gsd/` | Application root — all GSD state |
| `agentDir` | `~/.gsd/agent/` | Synced extensions, agents, skills, models.json, auth.json, settings, managed tools |
| `sessionsDir` | `~/.gsd/sessions/` | Per-directory session storage (JSONL files) |
| `authFilePath` | `~/.gsd/agent/auth.json` | Credential storage (API keys, OAuth tokens) |

---

## 5. Native Module Architecture (Rust/N-API)

The `native/` directory contains a Rust workspace compiled via napi-rs into a Node.js N-API addon.

### Build Configuration

- Rust edition 2021, resolver v2
- Release profile: `opt-level = 3`, `lto = "fat"`, `codegen-units = 1`, `strip = true`, `panic = "abort"`
- Dev profile: `codegen-units = 256`, `incremental = true`

### Crate Structure

| Crate | Purpose |
|-------|---------|
| `engine` | Main N-API addon — re-exports all native modules to JS |
| `grep` | Standalone ripgrep-backed content search library |
| `ast` | Structural code search via ast-grep |

### Engine Modules (native/crates/engine/src/)

| Module | Capability |
|--------|-----------|
| `grep.rs` | Ripgrep-backed content search |
| `glob.rs` + `glob_util.rs` | Gitignore-aware file discovery |
| `ps.rs` | Cross-platform process tree management |
| `highlight.rs` | Syntect-based syntax highlighting |
| `ast.rs` | Structural code search (delegates to ast crate) |
| `diff.rs` | Fuzzy text matching and unified diff generation |
| `text.rs` | ANSI-aware text measurement and wrapping |
| `html.rs` | HTML-to-Markdown conversion |
| `image.rs` | Image decode, encode, resize |
| `fd.rs` | Fuzzy file path discovery |
| `clipboard.rs` | Native clipboard access |
| `git.rs` | libgit2-backed git read operations |
| `gsd_parser.rs` | GSD file parsing and frontmatter extraction |
| `fs_cache.rs` | Filesystem caching layer |
| `stream_process.rs` | Streaming process execution |
| `truncate.rs` | Text truncation utilities |
| `ttsr.rs` | Tool-triggered system rules processing |
| `json_parse.rs` | JSON parsing utilities |
| `xxhash.rs` | xxHash hashing |
| `task.rs` | Async task management |

### Platform Distribution

Prebuilt binaries are published as optional npm packages per platform:

- `@gsd-build/engine-darwin-arm64` (macOS Apple Silicon)
- `@gsd-build/engine-darwin-x64` (macOS Intel)
- `@gsd-build/engine-linux-arm64-gnu` (Linux ARM64)
- `@gsd-build/engine-linux-x64-gnu` (Linux x86_64)
- `@gsd-build/engine-win32-x64-msvc` (Windows x86_64)

The `@gsd/native` TypeScript package wraps the N-API addon and provides typed JS APIs.

---

## 6. Extension System

### Discovery

Extensions are discovered by `discoverExtensionEntryPaths()` in `src/extension-discovery.ts`:

1. Top-level `.ts`/`.js` files in the extensions directory are standalone entry points.
2. Subdirectories are resolved via: (a) `package.json` with `pi.extensions` array, or (b) `index.ts`/`index.js` fallback.

Extensions are filtered by the unified registry and sorted in topological dependency order before loading.

### Registry

The extension registry (`~/.gsd/extensions/registry.json`) tracks enable/disable state per extension:

- Extensions without manifests always load (backwards compatible).
- Fresh installs start with an empty registry — all extensions enabled by default.
- Extensions can be disabled via `gsd extensions disable <id>`.
- Core-tier extensions (`tier: "core"` in `extension-manifest.json`) cannot be disabled.

Each extension manifest declares:
- `id`, `name`, `version`, `description`
- `tier`: `core` | `bundled` | `community`
- `provides`: tools, commands, hooks, shortcuts
- `dependencies`: other extensions, runtime requirements

### Loading Flow

1. **loader.ts**: Scans bundled extensions directory, remaps paths to `~/.gsd/agent/extensions/`, filters by registry, sorts topologically, serializes to `GSD_BUNDLED_EXTENSION_PATHS`.
2. **resource-loader.ts**: `initResources()` syncs bundled extensions to `~/.gsd/agent/extensions/`. `buildResourceLoader()` constructs a `DefaultResourceLoader` that loads from both `~/.gsd/agent/extensions/` (GSD) and `~/.pi/agent/extensions/` (Pi), deduplicating by extension key. Pi-origin extensions that overlap with bundled GSD extensions are excluded.
3. **createAgentSession()**: The Pi SDK loads extensions from the resource loader using jiti for TypeScript transpilation.

### Bundled Extensions (v2.58.0)

| Extension | Directory | Purpose |
|-----------|-----------|---------|
| GSD | `gsd/` | Core workflow engine: auto mode, state machine, commands, dashboard |
| Browser Tools | `browser-tools/` | Playwright-based browser automation |
| Search the Web | `search-the-web/` | Brave Search, Tavily, Jina page extraction |
| Google Search | `google-search/` | Gemini-powered web search |
| Context7 | `context7/` | Up-to-date library/framework documentation |
| Background Shell | `bg-shell/` | Long-running process management |
| Subagent | `subagent/` | Delegated tasks with isolated context windows |
| Mac Tools | `mac-tools/` | macOS native app automation via Accessibility APIs |
| MCP Client | `mcp-client/` | Native MCP server integration |
| Voice | `voice/` | Real-time speech-to-text |
| Slash Commands | `slash-commands/` | Custom command creation |
| Ask User Questions | `ask-user-questions.ts` | Structured user input |
| Secure Env Collect | `get-secrets-from-user.ts` | Masked secret collection |
| Async Jobs | `async-jobs/` | Background command execution |
| Remote Questions | `remote-questions/` | Discord, Slack, Telegram integration |
| TTSR | `ttsr/` | Tool-triggered system rules |
| Universal Config | `universal-config/` | Discovery of existing AI tool configs |
| AWS Auth | `aws-auth/` | AWS authentication for Bedrock |
| Claude Code CLI | `claude-code-cli/` | Claude Code CLI provider extension |
| GitHub Sync | `github-sync/` | Auto-sync to GitHub Issues, PRs, and Milestones |
| Ollama | `ollama/` | Local Ollama model integration |
| CMux | `cmux/` | Terminal multiplexer integration |

---

## 7. Session Management

Sessions are stored as JSONL files under `~/.gsd/sessions/`, organized by working directory:

```
~/.gsd/sessions/
  --Users-mudrii-src-project--/
    <session-id>.jsonl
    <session-id>.jsonl
```

The directory name is derived from the working directory path: leading slash stripped, path separators replaced with hyphens, wrapped in `--`.

Session modes:
- `SessionManager.create(cwd, dir)` — new session
- `SessionManager.continueRecent(cwd, dir)` — resume most recent session for cwd
- `SessionManager.open(path, dir)` — open a specific session file
- `SessionManager.inMemory()` — ephemeral session (used in `--no-session` mode)

Legacy flat sessions (pre per-directory scoping) are auto-migrated on startup.

The `gsd sessions` subcommand lists past sessions for the current directory and allows interactive selection for resumption.

### DB-Backed State Derivation

`deriveStateFromDb()` reads workflow state (milestones, slices, tasks, verification evidence) from the SQLite database instead of parsing markdown files. When milestones exist on disk but are missing from the database, disk-to-DB reconciliation runs automatically. Queue ordering respects `queue-order.json` in DB-backed derivation.

### Session Lock

Session locking uses OS-level file locks (`proper-lockfile`) for exclusive access. Multi-path cleanup ensures locks are released on all exit paths (normal exit, SIGINT, SIGTERM, SIGHUP, uncaught exceptions). False positive hardening retries lock file reads before declaring compromise, and a PID self-check prevents a process from treating its own lock as foreign. Stale numbered lock files and `.gsd.lock/` directories are cleaned up automatically.

### Workflow Logger

The workflow logger (`workflow-logger.ts`) is wired into engine, tool, manifest, and reconcile paths. It records structured audit entries for all state transitions, providing observability into the single-writer engine's operation.

---

## 8. Headless Mode Architecture

`gsd headless` runs commands without a TUI by spawning a child process in RPC mode.

### Design

The headless orchestrator (`src/headless.ts`) acts as a supervisory process:

1. Resolves the CLI path (`GSD_BIN_PATH` or `process.argv[1]`).
2. Creates an `RpcClient` that spawns a child `gsd --mode rpc` process.
3. Sends `/gsd <command>` as a prompt to the RPC child.
4. Monitors events from the child, auto-responding to `extension_ui_request` events.
5. Detects completion via `execution_complete` events (RPC v2), idle timeouts, or `agent_end` events.

### RPC Protocol v2

The `@gsd-build/rpc-client` package implements RPC protocol v2 with:

- **Init handshake**: Client sends `{ type: "init", protocolVersion: 2 }`, server responds with `sessionId` and `capabilities` (available events and commands).
- **Version detection**: v1 is the implicit default; v2 requires the init handshake. The client auto-detects protocol version.
- **Execution tracking**: `runId` generation on `prompt`/`steer`/`follow_up` commands enables correlation with `execution_complete` and `cost_update` events.
- **Typed events**: `RpcExecutionCompleteEvent` (status: completed/error/cancelled with stats) and `RpcCostUpdateEvent` (per-turn and cumulative cost with token breakdown).
- **Shutdown**: Graceful shutdown command for clean process teardown.
- **Subscribe**: Event type filtering via `subscribe` command.

### Options

| Flag | Purpose |
|------|---------|
| `--timeout <ms>` | Overall timeout (default 300s, 600s for new-milestone; disabled for auto-mode) |
| `--json` | Output events as JSONL to stdout |
| `--model <id>` | Override model for the session |
| `--context <file>` | Context file for new-milestone command |
| `--context-text <text>` | Inline context text |
| `--auto` | Chain into auto-mode after milestone creation |
| `--verbose` | Show tool calls in output |
| `--max-restarts <n>` | Auto-restart on crash (default 3) |
| `--supervised` | Forward interactive requests to orchestrator via JSONL |
| `--response-timeout <ms>` | Timeout for supervised orchestrator responses (default 30s) |
| `--answers <file>` | Pre-supplied answer file for automated question responses |
| `--events <types>` | Filter JSONL output to specific event types |
| `--bare` | Skip user config (PREFERENCES.md, KNOWLEDGE.md) for clean-room execution |
| `--resume <session>` | Resume a session by ID prefix matching |

### Text Mode and Verbose Output

Text mode (`--mode text`) and headless verbose output provide observability without a full TUI:

- Colorized output with thinking content, phase transitions, cost breakdowns, and durations.
- Tool call display showing tool names, arguments, and results.
- UAT pause skipping in headless mode to avoid blocking unattended execution.

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Complete (command finished successfully) |
| 1 | Error or timeout |
| 2 | Blocked (command reported a blocker) |

### Crash Recovery

The headless orchestrator supports automatic restarts with exponential backoff (5s * attempt, capped at 30s, default max 3 restarts). SIGINT/SIGTERM interrupts are not retried.

### Answer Injection

The `--answers` flag accepts a JSON file with pre-supplied responses for extension UI requests. The `AnswerInjector` observes events for question metadata and automatically responds when a matching answer is available, enabling fully unattended CI/cron execution.

### Supervised Mode

With `--supervised`, interactive UI requests from the child are forwarded as JSONL events to stdout. The orchestrating process reads responses from stdin. If no response arrives within `--response-timeout`, falls back to auto-response. This enables external orchestrators to make decisions about questions the agent asks.

---

## 9. Onboarding and Credential Management

### First-Run Wizard (`src/onboarding.ts`)

Triggered when no LLM provider is configured and stdin is a TTY. Uses `@clack/prompts` for a branded interactive experience:

1. **LLM provider selection**: OAuth browser flow (Anthropic, GitHub Copilot, OpenAI Codex, Google Gemini CLI, Antigravity) or API key paste (Anthropic, OpenAI, Google, Groq, xAI, OpenRouter, Mistral, Ollama Cloud, custom OpenAI-compatible).
2. **Web search provider**: Anthropic built-in (no key needed), Brave Search, or Tavily.
3. **Remote questions**: Discord, Slack, or Telegram bot setup with token validation and test message delivery.
4. **Optional tool keys**: Context7, Jina AI, Groq.

All steps are skippable. All errors are recoverable. The wizard never crashes boot.

### Credential Hydration (`src/wizard.ts`)

`loadStoredEnvKeys()` runs on every launch, hydrating `process.env` from `auth.json` for:
- `BRAVE_API_KEY`, `BRAVE_ANSWERS_KEY`
- `CONTEXT7_API_KEY`, `JINA_API_KEY`, `TAVILY_API_KEY`
- `SLACK_BOT_TOKEN`, `DISCORD_BOT_TOKEN`, `TELEGRAM_BOT_TOKEN`
- `GROQ_API_KEY`, `OLLAMA_API_KEY`, `CUSTOM_OPENAI_API_KEY`

This ensures extensions see stored keys without requiring users to set environment variables manually.

---

## 10. Build System

### Build Pipeline

```
npm run build
  1. build:native-pkg  — compile @gsd/native TypeScript bindings
  2. build:pi-tui      — compile @gsd/pi-tui
  3. build:pi-ai       — compile @gsd/pi-ai
  4. build:pi-agent-core — compile @gsd/pi-agent-core
  5. build:pi-coding-agent — compile @gsd/pi-coding-agent
  6. tsc                — compile GSD src/ to dist/
  7. copy-resources     — copy src/resources/ to dist/resources/
  8. copy-themes        — copy Pi theme assets
  9. copy-export-html   — copy HTML export templates
  10. build-web-if-stale — conditionally build Next.js web UI to dist/web
```

### TypeScript Configuration

- Module: `NodeNext`
- Target: `ES2022`
- Source: `src/` (excludes `src/resources/` and `src/tests/`)
- Output: `dist/`
- Strict mode enabled

### Native Build

`npm run build:native` compiles the Rust workspace via `node native/scripts/build.js`. Platform-specific binaries are published to npm as optional dependencies.

### Published Package Contents

The `files` field in `package.json` includes:
- `dist/` — compiled JS + resources
- `dist/web` — compiled Next.js web UI
- `packages/` — workspace packages (vendored Pi SDK)
- `pkg/` — PI_PACKAGE_DIR shim
- `src/resources/` — bundled extensions, agents, workflow docs
- `scripts/postinstall.js`, `scripts/link-workspace-packages.cjs`, `scripts/ensure-workspace-builds.cjs`

### postinstall

`npm run postinstall` runs `link-workspace-packages.cjs` to symlink `@gsd/*` packages, followed by `ensure-workspace-builds.cjs` to verify workspace packages are compiled, and then `postinstall.js` for any additional setup. This handles the common case; `loader.ts` has a fallback that re-links on startup if the symlinks are missing (covers `npx --ignore-scripts`).

---

## 11. Key Dependencies

| Dependency | Role |
|------------|------|
| `@anthropic-ai/sdk` | Anthropic Claude API |
| `@anthropic-ai/vertex-sdk` | Claude on Vertex AI |
| `@anthropic-ai/claude-agent-sdk` | Claude Agent SDK (optional) |
| `openai` | OpenAI API |
| `@google/genai` | Google Gemini API |
| `@mistralai/mistralai` | Mistral API |
| `@aws-sdk/client-bedrock-runtime` | AWS Bedrock |
| `@modelcontextprotocol/sdk` | MCP protocol support |
| `@mariozechner/jiti` | Runtime TypeScript transpilation for extensions |
| `playwright` | Browser automation (browser-tools extension) |
| `sql.js` | SQLite database for project state (WebAssembly build) |
| `chalk` / `picocolors` | Terminal coloring |
| `@clack/prompts` | Interactive prompts (onboarding wizard) |
| `sharp` | Image processing |
| `chokidar` | Filesystem watching |
| `proper-lockfile` | OS-level session locking |
| `undici` | HTTP client with proxy support |
| `yaml` | YAML parsing |
| `@sinclair/typebox` | Runtime type validation |
| `ajv` + `ajv-formats` | JSON Schema validation |
| `koffi` (optional) | FFI for native platform APIs (Mac Tools) |
| `discord.js` | Discord bot integration (daemon package) |

---

## 12. Execution Modes

| Mode | Entry | Description |
|------|-------|-------------|
| Interactive | `gsd` | Full TUI session with `InteractiveMode` |
| Print | `gsd --print "msg"` or `gsd --mode text "msg"` | Single-shot prompt, text output |
| JSON | `gsd --mode json "msg"` | Single-shot prompt, JSON output |
| RPC | `gsd --mode rpc` | JSON-RPC over stdin/stdout (used by headless, VS Code extension, daemon) |
| MCP | `gsd --mode mcp` | MCP server over stdin/stdout |
| Headless | `gsd headless [command]` | Supervised auto-mode without TUI |
| Web | `gsd --web` | Browser-based web interface (Next.js) |
| Config | `gsd config` | Re-run setup wizard |
| Update | `gsd update` | Self-update via npm |
| Sessions | `gsd sessions` | List and resume past sessions |
| Worktree | `gsd worktree <list|merge|clean|remove>` | Git worktree management |

---

## 13. Bundled Agents

| Agent | Role |
|-------|------|
| Scout | Fast codebase reconnaissance — produces compressed context for handoff |
| Researcher | Web research — finds and synthesizes current information |
| Worker | General-purpose execution in an isolated context window |

Agent definitions are synced from `src/resources/agents/` to `~/.gsd/agent/agents/` alongside extensions.

---

## 14. Dispatch Pipeline (Auto Mode)

The auto-mode state machine reads `.gsd/` project state and dispatches work units:

```
1.  Read state (DB-backed via deriveStateFromDb, with disk fallback)
2.  Determine next unit type and ID
3.  Classify complexity -> select model tier
4.  Apply budget pressure adjustments
5.  Check routing history for adaptive adjustments
6.  Dynamic model routing (if enabled) -> select cheapest model for tier
7.  Resolve effective model (with fallbacks)
8.  Check pending captures -> triage if needed
9.  Build dispatch prompt (applying inline level compression)
10. Create fresh agent session
11. Inject prompt and let LLM execute
12. On completion: snapshot metrics, verify artifacts, persist state
13. Loop to step 1
```

Key auto-mode modules (in the GSD extension):

| Module | Purpose |
|--------|---------|
| `auto.ts` | State machine and orchestration |
| `auto/session.ts` | `AutoSession` class — mutable auto-mode state |
| `auto/phases.ts` | Pipeline phase definitions |
| `auto/loop-deps.ts` | Loop dependency injection |
| `auto/run-unit.ts` | Single unit execution |
| `auto-dispatch.ts` | Declarative dispatch table (phase -> unit mapping) |
| `auto-idempotency.ts` | Completed-key checks, skip loop detection |
| `auto-stuck-detection.ts` | Stuck loop recovery and retry escalation |
| `auto-start.ts` | Fresh-start bootstrap, crash lock detection, worktree setup |
| `auto-post-unit.ts` | Post-unit commit, doctor, state rebuild, hooks |
| `auto-verification.ts` | Post-unit lint/test/typecheck gate with auto-fix retries |
| `complexity-classifier.ts` | Unit complexity classification (light/standard/heavy) |
| `model-router.ts` | Dynamic model routing with cost-aware selection |
| `metrics.ts` | Token and cost tracking ledger |
| `state.ts` | State derivation from disk and DB (`deriveStateFromDb`) |
| `session-lock.ts` | OS-level exclusive session locking with multi-path cleanup |
| `db-writer.ts` | Single-writer DB mutation layer |
| `workflow-logger.ts` | Structured audit log for state transitions |
| `workflow-reconcile.ts` | Disk-to-DB reconciliation |
| `workflow-manifest.ts` | Workflow template manifest loading |

---

## 15. VS Code Extension

The VS Code extension (`vscode-extension/`, v0.3.0, publisher: FluxLabs) provides IDE integration:

- **Sidebar dashboard**: Webview-based agent panel, session tree, and activity feed
- **@gsd chat participant**: VS Code Chat integration for inline agent queries
- **Code lens**: "Ask GSD" lens above functions and classes (configurable)
- **SCM integration**: Agent changes shown as a source control provider with accept/discard per-file
- **Session management**: New session, switch session, fork session, session naming
- **33 commands**: Start/stop agent, send message, cycle model/thinking, abort, export HTML, steer, run bash, switch model, fix problems, commit agent changes, create agent branch, show diff, approval mode, and more
- **Keybindings**: Chord-based shortcuts (Cmd+Shift+G prefix on macOS)
- **Configuration**: Binary path, auto-start, auto-compaction, code lens toggle, progress notifications, context warning threshold, approval mode (auto-approve/ask/plan-only)

Communicates with the GSD CLI via RPC mode (`gsd --mode rpc`).

---

## 16. Web Interface

The web interface (`web/`, gsd-web v0.1.0) is a Next.js 16 application providing browser-based access:

- **Stack**: React 19, Radix UI component library, Tailwind CSS 4, CodeMirror editor, xterm.js terminal
- **Features**: Dark/light themes, mobile responsive layout, authentication with localStorage token persistence, project root selection
- **Launch**: `gsd --web` or `npm run gsd:web` with `--host`, `--port`, `--allowed-origins` flags
- **Deployment**: Builds to standalone mode for `node .next/standalone/web/server.js`

---

## 17. Daemon

The daemon package (`packages/daemon/`, `@gsd-build/daemon` v0.1.0) provides a background process for unattended operation:

- **Session management**: Manages headless GSD sessions across multiple projects via `@gsd-build/rpc-client`
- **Discord bot**: discord.js v14 integration with authorization guard, slash commands, channel-per-project management, event bridging, and message batching
- **Orchestrator**: Natural-language command routing via LLM (configurable model) on a control channel
- **Project scanner**: Discovers projects by marker files (git, node, gsd, rust, python, go)
- **Event formatting**: 10 pure-function formatters mapping RPC events to Discord embeds
- **Verbosity levels**: Per-channel verbosity (default/verbose/quiet) for event streaming
- **launchd integration**: macOS launch agent install/uninstall/status via plist generation
- **Configuration**: YAML-based config with Discord token, guild/owner IDs, scan roots, and log settings
- **Binary**: `gsd-daemon` CLI entry point
