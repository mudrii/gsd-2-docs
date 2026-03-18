# Architecture

GSD (v2.28.0) is a TypeScript application built on top of the Pi SDK — a vendored, rebranded fork of pi-mono. It embeds the Pi coding agent and layers on the GSD workflow engine, auto-mode state machine, and project management primitives. This document covers the full architecture from the monorepo structure down to the data flow through the system at runtime.

---

## High-Level System Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                         gsd binary (dist/loader.js)                  │
│                                                                      │
│  loader.ts ──sets env vars──► cli.ts ──wires managers──► InteractiveMode
│                                  │                                   │
│                          ┌───────▼────────┐                         │
│                          │  Pi Runtime     │                         │
│                          │                │                         │
│                          │  ModelRegistry  │                         │
│                          │  AuthStorage    │                         │
│                          │  AgentSession   │                         │
│                          │  SessionManager │                         │
│                          │  AgentLoop      │                         │
│                          │  ToolExecutor   │                         │
│                          │  CompactionEng. │                         │
│                          │  EventSystem    │                         │
│                          │  ExtRuntime     │                         │
│                          │  ResourceLoader │                         │
│                          │  ModeLayer      │                         │
│                          └───────┬────────┘                         │
│                                  │                                   │
│                   ┌──────────────▼─────────────────┐                │
│                   │      GSD Extension Bundle        │                │
│                   │                                  │                │
│                   │  gsd/ (core: auto-mode, state    │                │
│                   │        machine, dispatch, git)   │                │
│                   │  browser-tools/  search-the-web/ │                │
│                   │  lsp/  subagent/  mac-tools/     │                │
│                   │  voice/  mcp-porter/  ...        │                │
│                   └──────────────┬─────────────────┘                │
│                                  │                                   │
│                   ┌──────────────▼─────────────────┐                │
│                   │      Native Rust Engine          │                │
│                   │  grep│glob│ps│ast│diff│text│git  │                │
│                   └─────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────────────┘

Additional modes:
  gsd headless   ── CI/cron via RPC child process
  gsd --mode mcp ── MCP server over stdin/stdout
  vscode-extension ── chat participant + RPC integration
```

---

## Monorepo Structure

The root `package.json` declares two workspace globs: `"packages/*"` and `"studio"`. This resolves to six packages total.

```
gsd-2/
├── package.json          # root — version 2.28.0, bin: gsd / gsd-cli
├── packages/
│   ├── pi-tui/           # @gsd/pi-tui     — Terminal UI library
│   ├── pi-ai/            # @gsd/pi-ai      — Unified LLM API
│   ├── pi-agent-core/    # @gsd/pi-agent-core — General-purpose agent core
│   ├── pi-coding-agent/  # @gsd/pi-coding-agent — Coding agent CLI
│   └── native/           # @gsd/native     — Rust N-API bindings
├── studio/               # @gsd/studio     — Electron dashboard (React)
├── src/                  # GSD application source (compiled to dist/)
│   ├── loader.ts         # CLI entry point
│   ├── cli.ts            # SDK wiring and startup
│   └── resources/
│       ├── extensions/   # 18 bundled extensions (gsd/, browser-tools/, ...)
│       ├── agents/       # scout, researcher, worker agent definitions
│       ├── AGENTS.md     # Agent routing instructions
│       └── GSD-WORKFLOW.md  # Manual bootstrap protocol
├── pkg/                  # Shim dir — piConfig + theme assets (no src/)
├── tsconfig.json         # Root TS config: rootDir=src, outDir=dist
└── tsconfig.extensions.json  # Extensions typecheck config (noEmit, allowJs)
```

### Workspace Package Descriptions

| Package | npm name | Description |
|---------|----------|-------------|
| `pi-tui` | `@gsd/pi-tui` | Terminal User Interface library — rendering, input, themes, ANSI layout |
| `pi-ai` | `@gsd/pi-ai` | Unified LLM API — wraps Anthropic, OpenAI, Google, Mistral, AWS Bedrock |
| `pi-agent-core` | `@gsd/pi-agent-core` | General-purpose agent core — session management, agent loop, compaction |
| `pi-coding-agent` | `@gsd/pi-coding-agent` | Coding agent CLI — extensions, tools, resource loader, RPC server |
| `native` | `@gsd/native` | Rust N-API bindings — grep, glob, diff, AST, git, image, clipboard |
| `studio` | `@gsd/studio` | Electron + React dashboard — visual project management UI |

---

## Package Dependency Graph

The packages form a strict layered stack. Runtime imports (from GSD's `src/`) confirm the dependency structure:

```
@gsd/native  (Rust, no JS deps)
     │
     ▼
@gsd/pi-tui  (TUI rendering — chalk, marked, mime-types)
     │
     ▼
@gsd/pi-ai   (LLM abstraction — anthropic-ai/sdk, openai, @google/genai, mistral)
     │
     ▼
@gsd/pi-agent-core  (agent loop, session, compaction — no @gsd/* deps declared)
     │
     ▼
@gsd/pi-coding-agent  (coding agent, extension runtime, resource loader,
                       RPC server — imports from @gsd/pi-ai, @gsd/pi-tui)
     │
     ▼
gsd root src/  (GSD CLI + extensions — imports @gsd/pi-coding-agent,
                @gsd/pi-ai, @gsd/pi-tui)
```

`@gsd/studio` (Electron) is independent of the Pi stack at the package level — it uses React, zustand, and communicates with the GSD process via RPC.

---

## The Pi Framework

The packages prefixed `pi-*` are vendored from the upstream `pi-mono` project. GSD ships them as workspace packages rather than npm dependencies so they can be built and patched as a unit.

### TUI Layer (`@gsd/pi-tui`)

Provides all terminal rendering primitives: ANSI-aware text measurement, layout engine, interactive widgets (lists, inputs, confirmations), theme resolution, and Markdown rendering. GSD's interactive mode renders through this layer.

### AI Abstraction (`@gsd/pi-ai`)

A unified interface over every supported LLM provider. A single `getModel()` call returns a normalized model object regardless of whether the underlying provider is Anthropic, OpenAI, Google Gemini, Mistral, or AWS Bedrock. API keys, OAuth tokens, and model metadata are managed by `ModelRegistry` and `AuthStorage`.

### Agent Core (`@gsd/pi-agent-core`)

Contains the agent loop engine, session manager (JSONL tree storage), and compaction engine. This is provider-agnostic — it operates on the normalized model interface from `@gsd/pi-ai`.

### Coding Agent (`@gsd/pi-coding-agent`)

Wraps the agent core with a full coding environment: built-in tools (`read`, `write`, `bash`, `grep`, `glob`, `ls`, `edit`), extension runtime, resource loader (skills, prompts, themes, context files), RPC server, and MCP server. This is the layer GSD embeds and extends.

---

## The Agent Loop

The agent loop is the runtime cycle that processes every user interaction. It lives in `@gsd/pi-agent-core` and is orchestrated by an `AgentSession`.

```
User sends prompt
        │
        ▼
┌─── TURN START ───────────────────────────────────────────────┐
│                                                               │
│  1. Assemble context                                          │
│     - System prompt (+ modifications from extension hooks)    │
│     - Previous messages (or compaction summary if compacted)  │
│     - The new user message                                    │
│                                                               │
│  2. Send to LLM (streaming)                                   │
│     - Stream response tokens to TUI                           │
│     - Parse any tool calls embedded in the response           │
│                                                               │
│  3. If tool calls present:                                    │
│     a. Fire tool_call event (extensions can block/modify)     │
│     b. Execute the tool (built-in or extension-registered)    │
│     c. Fire tool_result event (extensions can modify result)  │
│     d. Append result to message list                          │
│     e. Go back to step 1 (new turn with tool results)         │
│                                                               │
│  4. If no tool calls (LLM responded with text only):          │
│     - Save messages to session file                           │
│     - Emit agent_end event                                    │
│     - Done                                                    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

A single user prompt may trigger 1 or 50 LLM turns depending on task complexity. The loop continues until the LLM produces a `stop` reason (normal completion) rather than a `toolUse` reason. The `aborted` stop reason is produced when the user presses Escape.

### Event System

Every action in the agent loop emits a typed event. Extensions subscribe to these events to observe, gate, or modify behavior:

```
session_start → input → before_agent_start → agent_start
→ turn_start → context → tool_call → tool_result
→ turn_end → agent_end → session_shutdown
```

---

## Session / Memory Model

Sessions are the persistence mechanism for conversation history. They are stored as JSONL files:

```
~/.gsd/agent/sessions/--path--to--project--/<timestamp>_<uuid>.jsonl
```

### Tree Structure

Session entries form a **tree**, not a linear list. Every entry carries an `id` and `parentId`. When the user navigates to a previous point with `/tree` and continues, a new branch is created from that entry. All branches coexist in the same file — nothing is deleted.

```
                    ┌─ [user] ─ [assistant] ─ [tool] ─ [assistant]  ← Branch A
[header] ─ [user] ─┤
                    └─ [user] ─ [assistant]                          ← Branch B
```

### Entry Types

| Type | Sent to LLM | Purpose |
|------|-------------|---------|
| `session` | No | File header — metadata, version, working directory |
| `message` | Yes | Conversation message (user, assistant, toolResult) |
| `compaction` | Summary only | Pointer to where summarization cut off |
| `branch_summary` | Yes | Summary of an abandoned branch |
| `model_change` | No | Records model switch |
| `custom` | No | Extension state (not sent to LLM) |
| `custom_message` | Yes | Extension-injected message |
| `label` | No | User bookmark |

### Context Building

When a turn starts, the session manager walks the entry tree from the current leaf to the root. If a `compaction` entry is on the path, the LLM sees: system prompt → compaction summary → messages from `firstKeptEntryId` onward. This is how sessions survive context limit resets without losing continuity.

---

## Compaction

LLMs have finite context windows. The compaction engine keeps conversations alive beyond those limits.

### Trigger Conditions

- **Automatic:** When `contextTokens > contextWindow - reserveTokens`. Default reserve: 16,384 tokens.
- **Proactive:** Triggers as the limit approaches, not just when hit.
- **Manual:** `/compact [custom instructions]`

### How It Works

```
Before compaction:
  [user][assistant][tool][user][assistant][tool][tool][assistant]
  └─────────── summarize these ──────────┘└──── keep these (20k tokens) ────┘

After compaction:
  What the LLM sees: [system prompt] [summary] [kept messages...]
```

The LLM generates a structured summary of the older messages covering: goal, constraints, progress (done / in-progress / blocked), key decisions, next steps, critical context, and file lists (`<read-files>`, `<modified-files>`). A `CompactionEntry` is appended to the JSONL tree pointing to the `firstKeptEntryId`. The full history remains in the file — `/tree` can revisit the pre-compaction state at any time.

Default settings: `reserveTokens: 16384`, `keepRecentTokens: 20000`.

---

## The Branchless Worktree Architecture (ADR-001)

**Status:** Proposed (2026-03-15). Targets replacement of the slice-branch model shipped in v2.13.0.

### Background

Prior to ADR-001, GSD created a worktree per milestone with separate slice branches inside (`gsd/M001/S01`, `gsd/M001/S02`) that merged back to a milestone branch via `--no-ff`. This architecture produced persistent failures:

- Planning artifacts written to a slice branch were invisible to the milestone branch, causing `verifyExpectedArtifact()` to retry indefinitely and trigger hard stops.
- `.gsd/` was fully gitignored, creating a contradictory workaround where `smartStage()` force-added durable paths at runtime.
- ~582 lines of merge/branch/conflict code across `auto-worktree.ts`, `git-service.ts`, and `git-self-heal.ts` existed solely to manage slice branch complexity.

### The Decision

Eliminate slice branches entirely. All work within a milestone worktree commits sequentially on a single branch (`milestone/<MID>`). No branch creation, no branch switching, no slice merges, no conflict resolution within a worktree.

```
main ──────────────────────────────────────────────── main
  │                                                     ↑
  └─ worktree (milestone/M001)                          │
       │                                                │
       commit: feat(M001): context + roadmap            │
       commit: feat(M001/S01): research                 │
       commit: feat(M001/S01): plan                     │
       commit: feat(M001/S01/T01): impl                 │
       commit: feat(M001/S01/T02): impl                 │
       commit: feat(M001/S01): summary + UAT            │
       commit: feat(M001/S02): research                 │
       commit: ...                                      │
       commit: feat(M001): milestone complete           │
       │                                                │
       └─────────────── squash merge ───────────────────┘
```

### Git Primitives

| Used | Purpose |
|------|---------|
| Worktrees | One per active milestone — filesystem isolation at `.gsd/worktrees/<MID>/` |
| Sequential commits | Granular history of every action on the single branch |
| Squash merge | One clean commit on `main` per milestone completion |
| `milestone/<MID>` branch | The only branch created per milestone (plus `main`) |

Slice branches, `--no-ff` merges, branch switching within a worktree, and intra-worktree conflict resolution are all eliminated.

### .gsd/ Tracking Model

Under ADR-001, `.gsd/` is split into tracked and untracked zones:

**Tracked in git (travel with the branch):**
- `.gsd/milestones/` — roadmaps, plans, summaries, research, task plans
- `.gsd/PROJECT.md`, `.gsd/DECISIONS.md`, `.gsd/REQUIREMENTS.md`, `.gsd/QUEUE.md`

**Gitignored (ephemeral/runtime):**
- `.gsd/runtime/`, `.gsd/activity/`, `.gsd/worktrees/`, `.gsd/auto.lock`
- `.gsd/metrics.json`, `.gsd/completed-units.json`, `.gsd/STATE.md`, `.gsd/gsd.db`

This eliminates the contradiction where `smartStage()` force-added paths that `.gitignore` claimed were untracked. After ADR-001, `git add -A` picks up planning artifacts naturally, and `smartStage()` reduces to staging all files then unstaging the runtime paths.

### Code Impact

ADR-001 deletes approximately 770+ lines across 6 files and 11 test files, including `mergeSliceToMilestone()`, `mergeSliceToMain()`, `git-self-heal.ts` merge recovery functions, the `fix-merge` dispatch unit, and branch-mode isolation support.

---

## GSD Extension: Plugging into Pi

GSD's workflow engine is implemented as an extension loaded by the Pi extension runtime. The core GSD extension lives at `src/resources/extensions/gsd/` and is synced to `~/.gsd/agent/extensions/` on every launch.

### Key Modules

| Module | Purpose |
|--------|---------|
| `auto.ts` | Auto-mode state machine and orchestration loop |
| `auto-dispatch.ts` | Declarative dispatch table mapping phases to unit types |
| `auto-prompts.ts` | Prompt builders with inline-level compression |
| `auto-worktree.ts` | Worktree lifecycle — create, enter, merge, teardown |
| `complexity-classifier.ts` | Classify unit complexity (light/standard/heavy) for model routing |
| `model-router.ts` | Cost-aware dynamic model selection |
| `routing-history.ts` | Adaptive learning from past routing outcomes |
| `captures.ts` | Fire-and-forget thought capture and triage |
| `state.ts` | Derive GSD state from disk artifacts |
| `git-service.ts` | Commit, merge, worktree sync, libgit2-backed read operations |
| `metrics.ts` | Token and cost tracking ledger |
| `memory-extractor.ts` | Extract reusable knowledge from session transcripts |

The extension registers its tools and commands with the Pi extension runtime via the event system, making them available in every agent session GSD starts.

### Always-Overwrite Sync

Bundled extensions and agents are copied from `dist/resources/` (or `src/resources/` in dev) to `~/.gsd/agent/` on every launch. This means `npm update -g gsd-pi` takes effect immediately without a separate setup step.

---

## CLI Entry Point: loader.ts

The binary entry point is `dist/loader.js`, which compiles from `src/loader.ts`. It implements a strict two-phase startup to guarantee environment variables are set before any Pi SDK code evaluates.

### Phase 1 — Environment Setup (no SDK imports)

1. Read `package.json` for the version string.
2. Handle `--version` / `--help` fast paths without loading any heavy dependencies.
3. Set `PI_PACKAGE_DIR` to the `pkg/` shim directory. This must happen before any Pi SDK module evaluates, because Pi reads this env var at module initialization time to determine `APP_NAME` and `CONFIG_DIR_NAME`.
4. Set `GSD_CODING_AGENT_DIR`, `GSD_VERSION`, `GSD_BIN_PATH`, `GSD_WORKFLOW_PATH`, `GSD_BUNDLED_EXTENSION_PATHS`.
5. Prepend GSD's `node_modules` to `NODE_PATH` and call `Module._initPaths()` synchronously so extension module resolution works correctly.
6. Lazily configure a proxy dispatcher via `undici` only if `HTTP_PROXY` / `HTTPS_PROXY` env vars are present.
7. Ensure workspace packages are symlinked into `node_modules/@gsd/` (handles `npx --ignore-scripts` edge case; falls back to `cpSync` on Windows without Developer Mode).

### Phase 2 — SDK Startup (dynamic import)

```typescript
await import('./cli.js')
```

The `await import()` is a dynamic import, which defers ESM module evaluation. By the time `cli.ts` loads and executes its static imports of the Pi SDK, `PI_PACKAGE_DIR` is already in `process.env`. `cli.ts` wires the SDK managers, loads extensions, and starts the interactive mode.

### The pkg/ Shim

`PI_PACKAGE_DIR` points to `pkg/` rather than the project root. This prevents Pi's theme resolution from colliding with GSD's `src/` directory. The `pkg/` directory contains only the `piConfig` entry in `package.json` (name: `"gsd"`, configDir: `".gsd"`) and the theme assets under `dist/`.

---

## Build System

### TypeScript Compilation

The root `tsconfig.json` compiles `src/` (excluding `src/resources/` and `src/tests/`) to `dist/` with:
- `module: "NodeNext"`, `moduleResolution: "NodeNext"` — full ESM with explicit extensions
- `target: "ES2022"` — modern JS output
- `strict: true`, `declaration: true`

Extensions in `src/resources/extensions/` are NOT compiled — they are copied as-is and loaded at runtime via jiti (TypeScript transpiler). `tsconfig.extensions.json` is used only for type-checking extensions (`noEmit: true`, `allowImportingTsExtensions: true`).

### Workspace Build Order

```
build:native-pkg          # Rust → N-API .node file
  → build:pi-tui          # @gsd/pi-tui
  → build:pi-ai           # @gsd/pi-ai
  → build:pi-agent-core   # @gsd/pi-agent-core
  → build:pi-coding-agent # @gsd/pi-coding-agent
  → tsc                   # root src/ → dist/
  → copy-resources        # src/resources/ → dist/resources/
  → copy-themes           # theme assets
  → copy-export-html      # HTML export template
```

The full pipeline is triggered by `npm run build`. Before publishing, `prepublishOnly` runs `sync-pkg-version`, `sync-platform-versions`, `build`, `typecheck:extensions`, and `validate-pack` in sequence.

### Platform-Specific Native Packages

The Rust engine ships as separate optional npm packages per platform:

```
@gsd-build/engine-darwin-arm64
@gsd-build/engine-darwin-x64
@gsd-build/engine-linux-arm64-gnu
@gsd-build/engine-linux-x64-gnu
@gsd-build/engine-win32-x64-msvc
```

npm installs only the matching optional dependency for the current platform. The `@gsd/native` package resolves the correct binary at runtime.

---

## RPC and SDK Embedding Model

Pi (and by extension GSD) is designed to be embedded in other applications via two mechanisms.

### TypeScript SDK

Node.js applications import `@gsd/pi-coding-agent` directly:

```typescript
import { AuthStorage, createAgentSession, ModelRegistry, SessionManager } from "@mariozechner/pi-coding-agent";

const { session } = await createAgentSession({
  sessionManager: SessionManager.inMemory(),
  authStorage: AuthStorage.create(),
  modelRegistry: new ModelRegistry(authStorage),
});

session.subscribe((event) => { /* stream events */ });
await session.prompt("What files are in the current directory?");
```

This gives full control over tools, resource loading, session management, model selection, and event streaming.

### RPC Mode (Any Language)

Non-Node.js applications spawn GSD as a subprocess and communicate via newline-delimited JSON over stdin/stdout:

```
gsd --mode rpc --provider anthropic
```

Commands sent to stdin:
```json
{"type": "prompt", "message": "..."}
{"type": "steer", "message": "..."}
{"type": "follow_up", "message": "..."}
```

Events received from stdout:
```json
{"type": "event", "event": {"type": "message_update", ...}}
{"type": "response", "command": "prompt", "success": true}
```

GSD's headless mode uses this same RPC mechanism — it spawns a Pi child process and communicates with it to orchestrate CI/cron workflows.

---

## Auto-Mode Dispatch Pipeline

When auto mode runs, each work unit goes through a 13-step dispatch pipeline:

```
1.  Read disk state (STATE.md, roadmap, plans)
2.  Determine next unit type and ID
3.  Classify complexity → select model tier (light/standard/heavy)
4.  Apply budget pressure adjustments
5.  Check routing history for adaptive model adjustments
6.  Dynamic model routing → select cheapest model for the resolved tier
7.  Resolve effective model (with fallbacks)
8.  Check pending captures → triage if needed
9.  Build dispatch prompt (applying inline-level compression)
10. Create fresh agent session (clean context window)
11. Inject prompt and let LLM execute the agent loop
12. On completion: snapshot metrics, verify artifacts, persist state
13. Loop to step 1
```

Every dispatch creates a **new** agent session. The LLM starts with a clean context window containing only the pre-inlined artifacts it needs. This prevents quality degradation from accumulated context — each unit is isolated from the execution noise of previous units.

Phase skipping (configured via the token profile) gates steps 2-3: if a phase is marked skipped, its unit type is never dispatched.

---

## Data Flow: End to End

```
User types input in terminal
        │
        ▼
┌── @gsd/pi-tui ──────────────────────────────────┐
│   Interactive mode captures keystrokes           │
│   Renders streaming output to terminal           │
└──────────────────┬──────────────────────────────┘
                   │ user message
                   ▼
┌── AgentSession (@gsd/pi-agent-core) ────────────┐
│   Session manager appends entry to JSONL tree    │
│   Agent loop assembles context (or summary)      │
└──────────────────┬──────────────────────────────┘
                   │ assembled messages array
                   ▼
┌── @gsd/pi-ai ───────────────────────────────────┐
│   Routes to correct provider (Anthropic/OpenAI/  │
│   Google/Mistral/Bedrock)                        │
│   Streams response tokens back                   │
└──────────────────┬──────────────────────────────┘
                   │ tool calls or text response
                   ▼
┌── ToolExecutor + ExtensionRuntime ──────────────┐
│   Built-in tools: read, write, bash, grep, ls   │
│   Extension tools: browser, search, LSP, etc.   │
│   GSD auto-mode tools: dispatch, git, state      │
│   Each tool fires tool_call and tool_result      │
│   events that extensions can intercept           │
└──────────────────┬──────────────────────────────┘
                   │ tool results → loop back to LLM
                   │ or final text response
                   ▼
┌── SessionManager ───────────────────────────────┐
│   Appends all messages to JSONL tree             │
│   Triggers compaction if context limit approached│
│   Emits agent_end event                          │
└──────────────────┬──────────────────────────────┘
                   │
                   ▼
        Rendered to terminal via @gsd/pi-tui
```

---

## Key Design Decisions Summary

| Decision | Rationale |
|----------|-----------|
| Two-file loader pattern | `PI_PACKAGE_DIR` must be set before any Pi SDK module evaluates. Dynamic import defers evaluation. |
| `pkg/` shim directory | Prevents Pi's theme resolution from colliding with GSD's `src/` directory. |
| Always-overwrite extension sync | `npm update -g` takes effect immediately — no separate setup step. |
| Lazy provider loading | LLM provider SDKs load on first use, reducing cold-start time significantly. |
| Fresh session per dispatch unit | Prevents context accumulation from degrading LLM quality across units. |
| State lives on disk (`.gsd/`) | Enables crash recovery, multi-terminal steering, session resumption. |
| Branchless worktree (ADR-001) | Eliminates ~582 lines of merge/conflict code and the artifact-visibility failures caused by slice branches. |
| libgit2 for read operations | Eliminates ~70 process spawns per dispatch cycle (since v2.16). |
| JSONL tree for sessions | All branches coexist in one file; `/tree` can revisit any historical point. |
