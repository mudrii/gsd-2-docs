# Extensions and Bundled Tools

GSD-2 (pi) ships with a modular extension system. Each extension registers tools (LLM-callable), commands (slash-commands), hooks (lifecycle events), and shortcuts. Extensions are lazy-loaded in interactive mode to keep startup fast.

This document covers every bundled extension, the agent definitions, and the skill system.

---

## Table of Contents

1. [Extension System](#extension-system)
2. [GSD Core Extension](#gsd-core-extension)
3. [Ollama](#ollama)
4. [Claude Code CLI Provider](#claude-code-cli-provider)
5. [Browser Tools](#browser-tools)
6. [Search the Web](#search-the-web)
7. [Google Search](#google-search)
8. [Context7](#context7)
9. [Background Shell](#background-shell)
10. [Async Jobs](#async-jobs)
11. [Subagent](#subagent)
12. [Bundled Agents](#bundled-agents)
13. [Mac Tools](#mac-tools)
14. [MCP Client](#mcp-client)
15. [MCP Server](#mcp-server)
16. [Voice](#voice)
17. [GitHub Sync](#github-sync)
18. [Slash Commands](#slash-commands)
19. [Ask User Questions](#ask-user-questions)
20. [Secure Env Collect](#secure-env-collect)
21. [Remote Questions](#remote-questions)
22. [Universal Config](#universal-config)
23. [AWS Auth](#aws-auth)
24. [TTSR](#ttsr)
25. [Skill System](#skill-system)
26. [BTW Skill](#btw-skill)
27. [Dashboard and Visualizer](#dashboard-and-visualizer)

---

## Extension System

### Manifest and Registry (v2.30.0)

Every extension can declare a `package.json` manifest with a `pi` field containing metadata: extension ID, display name, description, and `provides` (tools, commands, hooks). The **extension registry** (`~/.gsd/extension-registry.json`) stores per-extension enable/disable state, allowing users to manage which extensions load at startup.

Extensions without a manifest are always loaded (backwards-compatible). Extensions with a manifest are filtered through the registry before loading.

### Topological Sort and Unified Filtering (v2.58.0, wired v2.59.0)

Extension loading now performs a topological sort on declared dependencies so that libraries (e.g., `cmux`) initialize before the extensions that depend on them. The unified registry filtering pipeline runs after discovery: read manifests, check enabled state, sort by dependency order, and return diagnostics for any unresolvable dependency warnings. As of v2.59.0, topological sort and unified registry filtering are fully wired into the extension loading pipeline.

### Extension Discovery with Pi Manifest Opt-Out (v2.39.0)

The extension discovery system (`extension-discovery.ts`) scans `src/resources/extensions/` for entry points. Directories that declare `"pi": {}` (empty provides) in their `package.json` are treated as shared libraries, not extensions, and are excluded from the loading pipeline. This allows utility packages like `cmux` to live alongside extensions without being loaded as one.

### `provides.hooks` in Extension Manifests (v2.58.0)

Seven extension manifests now declare `provides.hooks` arrays listing which lifecycle hooks they handle (e.g., `session_start`, `agent_end`, `turn_start`). The `/gsd extensions` command displays these hooks in the extension detail view, giving users visibility into which extensions participate in each lifecycle phase.

### TypeScript Syntax Detection in .js Extension Files (v2.44.0)

The extension loader detects TypeScript syntax inside `.js` files and surfaces a warning suggesting the file be renamed to `.ts`. This prevents silent compilation failures when user-authored extensions use TypeScript features but have a `.js` extension.

### Shared Jiti Module Cache (v2.64.0)

Extension loading now shares a single jiti module cache across all extension `import()` calls. This eliminates redundant re-compilation of shared dependencies during startup, improving load performance for projects with many extensions.

### Pruning Removed Bundled Extensions on Upgrade (v2.64.0)

When GSD is upgraded, the resource sync pipeline detects bundled subdirectory extensions that have been removed from the new version and prunes them from the user's local extension directory. This prevents stale extensions from lingering after an upgrade and causing conflicts or errors.

---

## GSD Core Extension

**Path:** `src/resources/extensions/gsd/`

The core extension powers the entire GSD workflow system. It is the largest extension by far, spanning 150+ source files.

### Commands

| Command | Description |
|---------|-------------|
| `/gsd` or `/gsd next` | Contextual wizard -- reads project state from disk and shows the next logical action |
| `/gsd auto` | Start auto-mode -- loops fresh sessions per unit until milestone is complete |
| `/gsd stop` | Stop auto-mode gracefully |
| `/gsd pause` | Pause auto-mode (preserves state) |
| `/gsd status` | Progress dashboard overlay |
| `/gsd visualize` | Full-screen 10-tab workflow visualizer |
| `/gsd queue` | Queue and reorder future milestones |
| `/gsd quick <task>` | Execute a task without full milestone/slice ceremony |
| `/gsd discuss` | Architecture/decision discussion phase |
| `/gsd capture` | Fire-and-forget thought capture |
| `/gsd triage` | Manually triage pending captures |
| `/gsd dispatch` | Dispatch a specific phase directly |
| `/gsd history` | View execution history |
| `/gsd undo` | Revert last completed unit |
| `/gsd skip` | Prevent a unit from auto-mode dispatch |
| `/gsd export` | Export milestone/slice results (includes `--html` for HTML reports) |
| `/gsd cleanup` | Remove merged branches or snapshots |
| `/gsd mode` | Switch workflow mode (solo/team) |
| `/gsd prefs` | Manage preferences (model selection, timeouts, etc.) |
| `/gsd config` | Set API keys for external tools |
| `/gsd keys` | API key manager -- list, add, remove, test, rotate, doctor |
| `/gsd hooks` | Show configured post-unit and pre-dispatch hooks |
| `/gsd run-hook` | Manually trigger a specific hook |
| `/gsd skill-health` | Skill performance dashboard |
| `/gsd doctor` | Environment health checks |
| `/gsd remote` | Remote questions setup (Slack/Discord/Telegram) |
| `/gsd steer` | Inject guidance into running auto-mode |
| `/gsd knowledge` | View project knowledge base |
| `/gsd parallel` | Parallel orchestration across slices |
| `/gsd start` | Start from a workflow template |
| `/gsd templates` | List available workflow templates |
| `/gsd new-milestone` | Create a new milestone |
| `/gsd update` | Update GSD |
| `/gsd mcp` | MCP server status and connectivity (v2.45.0) |
| `/gsd rethink` | Conversational project reorganization (v2.45.0) |
| `/gsd codebase` | Generate and manage codebase map; supports preferences, `--collapse-threshold`, and auto-init (v2.58.0, enhanced v2.62.0) |
| `/gsd changelog` | LLM-summarized release notes (v2.35.0) |
| `/gsd forensics` | Full-access GSD debugger with journal awareness (v2.40.0) |
| `/gsd fast` | Enable fast service tier for supported models (v2.42.0) |
| `/gsd logs` | Browse activity, debug, and metrics logs (v2.29.0) |
| `/gsd parallel watch` | Native TUI overlay for worker monitoring (v2.56.0) |

### Key Subsystems

**Guided Flow** (`guided-flow.ts`): The smart entry wizard. Reads project state from disk (STATE.md, ROADMAP.md, plans, tasks), detects what phase the project is in, and presents contextual options. Dispatches work through GSD-WORKFLOW.md.

**Quick Mode** (`quick.ts`): Lightweight task execution with GSD guarantees (atomic commits, state tracking) but without milestone/slice ceremony. Quick tasks live in `.gsd/quick/` and are tracked in STATE.md.

**Key Manager** (`key-manager.ts`): Comprehensive CLI for managing API keys across 15+ LLM providers (Anthropic, OpenAI, Google, Groq, xAI, OpenRouter, Mistral, Cerebras, Azure, Ollama Cloud, custom OpenAI-compat), tool keys (Context7, Jina), search providers (Tavily, Brave), and remote integrations (Slack, Discord, Telegram). Supports list, add, remove, test, rotate, and doctor operations.

**Skill Discovery** (`skill-discovery.ts`): Detects skills installed during auto-mode by comparing the current skills directory against a snapshot taken at auto-mode start. New skills are injected into the system prompt via `before_agent_start`.

**Reports** (`reports.ts`): Manages `.gsd/reports/` -- the persistent progression log of HTML snapshots. Auto-triggered after milestone completion when `auto_report: true`. Generates self-contained HTML files with progress trees, dependency graphs (SVG DAG), cost/token bar charts, execution timeline, changelog, and knowledge base. An auto-generated `index.html` shows all reports with progression metrics.

**Auto-Mode** (`auto.ts` and related `auto-*.ts` files): The autonomous execution loop. Fresh session per unit, with stuck detection, timeout recovery, budget management, model selection, observability, verification gates, and worktree isolation.

**Parallel Orchestrator** (`parallel-orchestrator.ts`): Runs multiple slices in parallel across worktrees with concurrency management, worker status tracking, and merge conflict resolution.

**Dashboard Overlay** (`dashboard-overlay.ts`): Full-screen TUI overlay showing auto-mode progress -- milestone/slice/task breakdown, current unit, completed units, timing, activity log, and cost metrics. Toggled with `Ctrl+Alt+G`.

**Preferences** (`preferences.ts`, `preferences-*.ts`): YAML-based preferences in `~/.gsd/PREFERENCES.md` and project-level `.gsd/PREFERENCES.md`. Controls model selection, timeouts, skill rules, auto-mode behavior, and more.

**Doctor** (`doctor.ts`, `doctor-*.ts`): Environment health checks covering providers, tools, git configuration, permissions, and file integrity.

**Codebase Map** (`codebase-generator.ts`, `commands-codebase.ts`) (v2.58.0, enhanced v2.59.0 + v2.62.0): Generates `.gsd/CODEBASE.md` -- a structured map of the project's source files with file descriptions, coverage stats, and generation timestamps. Supports incremental updates that preserve existing descriptions. Subcommands: `generate [--max-files N]`, `update`, `stats`. As of v2.59.0, the codebase map provides structural orientation for fresh agent contexts. In v2.62.0, `/gsd codebase` gained preferences integration, a `--collapse-threshold` flag for controlling directory expansion, and auto-init behavior.

**Stale Commit Safety Check** (`doctor-git-checks.ts`) (v2.58.0): Doctor detects uncommitted changes that have sat idle beyond a configurable threshold. When auto-fix is enabled, it creates a snapshot commit (`gsd snapshot: uncommitted changes after Nm inactivity`) to prevent data loss. Stale snapshot commits are auto-cleaned during `/gsd cleanup`.

**Single-Writer Engine** (v2.46.0): State machine guards, actor identity, and reversibility layer on the SQLite-backed state engine. All write-side state transitions go through atomic SQLite tool calls instead of direct markdown mutation (v2.44.0).

---

## Ollama

**Path:** `src/resources/extensions/ollama/`

First-class local LLM support via Ollama (v2.58.0, native provider v2.59.0 + v2.64.0). Auto-detects a running Ollama instance, discovers locally pulled models, and registers them as a provider. No configuration required -- if Ollama is running, models appear automatically.

### How It Works

1. Probes `http://localhost:11434/api/tags` (or `OLLAMA_HOST` env var) on session start
2. Auto-discovers all locally pulled models via `/api/tags`
3. Capability detection per model: vision, reasoning, and context window size
4. Zero-cost model registration (local inference, no API key)
5. Re-probes on session switch to pick up newly pulled models
6. Unregisters provider automatically if Ollama stops running

### Native `/api/chat` Provider (v2.64.0)

In v2.64.0, Ollama switched from the OpenAI-compatible endpoint (`/v1/chat/completions`) to a native `/api/chat` stream provider (`ollama-chat-provider.ts`). This exposes the full set of Ollama generation options directly:

- **Temperature**, **top_p**, **top_k**, **repeat_penalty**, and other sampling parameters
- Full option passthrough to the Ollama API without translation loss
- Uses `apiKey` auth mode to avoid a `streamSimple` crash that occurred with the `none` auth mode

### Tool: `ollama_manage`

LLM-callable tool for managing local models:

| Action | Description |
|--------|-------------|
| `status` | Check if Ollama is running, show version |
| `list` | List available local models with sizes |
| `pull` | Download a model with progress tracking |
| `ps` | Show running models and VRAM usage |

### Commands

| Command | Description |
|---------|-------------|
| `/ollama` | Show Ollama status (running, version, loaded models) |
| `/ollama list` | List all available local models with sizes |
| `/ollama pull <model>` | Pull a model with progress |
| `/ollama remove <model>` | Delete a local model |
| `/ollama ps` | Show running models and resource usage |

### Provider Registration

- **Auth mode:** `apiKey` (v2.64.0; previously `none`)
- **API:** Native Ollama `/api/chat` (v2.64.0; previously OpenAI-compat `/v1/chat/completions`)
- **Compat flags:** No developer role, no reasoning effort, no usage in streaming
- **Cost:** Zero (local inference)

---

## Claude Code CLI Provider

**Path:** `src/resources/extensions/claude-code-cli/`

External tool execution mode for CLI providers (v2.47.0, stateful v2.59.0). Registers a model provider that delegates inference to the user's locally-installed Claude Code CLI via the official Agent SDK.

Users with a Claude Code subscription (Pro/Max/Team) get subsidized inference through GSD's UI -- no API key required. Uses Anthropic's official `@anthropic-ai/claude-agent-sdk`, never touches credentials, never offers a login flow.

### Architecture

- **Auth mode:** `externalCli` -- a new provider category added in v2.44.0 for non-API-key providers
- **API:** `anthropic-messages`
- **Readiness check:** Detects whether `claude` CLI is installed and authenticated
- **Streaming:** Custom stream adapter translates Agent SDK events into GSD's internal stream format
- **Partial builder:** Accumulates streaming deltas into complete message structures
- Tool calls are rendered above text responses in the TUI

### Stateful Context (v2.59.0)

The Claude Code CLI provider is now stateful with full context and sidechain events. This means the provider maintains conversation context across turns within a session, matching the behavior of direct API providers.

### Anthropic Subscription Routing (v2.67.0)

Users with active Anthropic subscriptions are automatically routed through the Claude Code CLI provider instead of the direct API. This avoids API billing for users who already have a subscription plan. The UI distinguishes between `anthropic-api` (direct API) and `claude-code` (CLI provider) in the model selector, preferences wizard, and all model/provider surfaces. Fallback guards ensure that when the CLI provider fails, fallback is restricted to the `anthropic` provider only.

### Native Windows Support (v2.67.0)

The provider uses native Windows `claude` CLI lookup on Win32, removing the dependency on Unix-style PATH resolution.

### Models

Exposes the Claude models available through the user's Claude Code subscription, identified by their standard Anthropic model IDs.

---

## Browser Tools

**Path:** `src/resources/extensions/browser-tools/`

Full browser interaction via Playwright. Provides comprehensive web automation capabilities through a modular tool system.

### Tool Categories

| Category | Module | Capabilities |
|----------|--------|-------------|
| **Navigation** | `navigation.ts` | Navigate to URLs, go back, manage browser lifecycle |
| **Screenshot** | `screenshot.ts` | Capture page screenshots for visual analysis |
| **Interaction** | `interaction.ts` | Click, type, drag elements on the page |
| **Inspection** | `inspection.ts` | Read page state, DOM inspection, accessibility tree |
| **Session** | `session.ts` | Browser session management (open, close, context) |
| **Assertions** | `assertions.ts` | Verify page state, element presence, content checks |
| **Refs** | `refs.ts` | Element reference system with versioned snapshots |
| **Wait** | `wait.ts` | Wait for elements, navigation, network idle |
| **Pages** | `pages.ts` | Multi-tab/page management |
| **Forms** | `forms.ts` | Form filling, select options, file upload |
| **Intent** | `intent.ts` | High-level intent-based actions |
| **PDF** | `pdf.ts` | PDF interaction and extraction |
| **State Persistence** | `state-persistence.ts` | Save and restore browser state |
| **Network Mock** | `network-mock.ts` | Mock/intercept network requests |
| **Device** | `device.ts` | Device emulation (mobile, tablet) |
| **Extract** | `extract.ts` | Extract structured data from pages |
| **Visual Diff** | `visual-diff.ts` | Compare visual snapshots |
| **Zoom** | `zoom.ts` | Page zoom control |
| **Codegen** | `codegen.ts` | Generate test code from interactions |
| **Action Cache** | `action-cache.ts` | Cache and replay action sequences |
| **Injection Detection** | `injection-detect.ts` | Detect injection vulnerabilities |

### Architecture

- Lazy-loads on `session_start` -- browser is only launched when tools are first called
- Cleans up browser on `session_shutdown`
- Supports adaptive settling after actions (mutation detection)
- Element reference system with ref snapshots for stable targeting
- Session artifact management for screenshots and captures
- Post-action summaries with compact page state capture

---

## Search the Web

**Path:** `src/resources/extensions/search-the-web/`

Web search with multiple provider backends. Provides three tools:

### Tools

**`web_search`** -- Rich web search with support for:
- **Brave Search**: Full API with web results, summarizer, spell correction, freshness filters
- **Tavily Search**: AI-optimized search with relevance scoring
- **Ollama**: Local search backend
- Automatic provider selection based on available API keys
- LRU + TTL caching, retry with backoff, rate limit tracking
- Structured error taxonomy (auth, rate limit, network, etc.)

**`fetch_page`** -- Extract clean markdown from any URL:
- Primary: Jina Reader API (converts pages to markdown)
- Fallback: Direct fetch with content-type awareness
- Offset parameter for continuation reading
- Selector parameter for targeted extraction (Jina's X-Target-Selector)
- JSON passthrough, PDF detection

**`search_and_read`** -- Combined search + content extraction in one call:
- Tavily backend: POST-based with client-side token budgeting
- Brave backend: LLM Context API with server-side budgeting
- Returns pre-extracted, relevance-scored page content
- Best for "I need to know about X" queries

### Search Provider Resolution

Provider preference is stored in `~/.gsd/agent/auth.json` and `~/.gsd/PREFERENCES.md`. Resolution order:

1. Override preference (from tool parameter)
2. `PREFERENCES.md` setting
3. `auth.json` stored preference
4. Auto-detect: Tavily first, then Brave, then Ollama

**API keys:** `TAVILY_API_KEY`, `BRAVE_API_KEY`, `OLLAMA_API_KEY`

### Commands

**`/search-provider`** -- Switch the active search provider.

---

## Google Search

**Path:** `src/resources/extensions/google-search/`

Web search via Gemini's Google Search grounding feature.

### Tool: `google_search`

Sends queries to Gemini Flash with `googleSearch: {}` enabled. Gemini internally performs Google searches, synthesizes an answer, and returns it with source URLs from grounding metadata.

**Features:**
- AI-synthesized answers grounded in Google Search results
- Source URLs with domain information
- In-session result caching
- Dual authentication: `GEMINI_API_KEY` or Google OAuth (via Cloud Code Assist API)
- 30-second timeout per request
- Retry with backoff for 429/5xx errors (OAuth path)
- Configurable model via `GEMINI_SEARCH_MODEL` env var (default: `gemini-2.5-flash`)

---

## Context7

**Path:** `src/resources/extensions/context7/`

Native pi extension replacing the Context7 MCP server. Provides up-to-date library documentation.

### Tools

**`resolve_library`** -- Search the Context7 library catalogue by name:
- Returns candidates with metadata (trust score, benchmark score, snippet count, token count)
- In-session caching of search results
- Optional query parameter improves result ranking

**`get_library_docs`** -- Fetch documentation for a specific library:
- Scoped to an optional query/topic for focused results
- Smart token budgeting (default 5000, configurable per call, max 10000)
- Truncation guard to prevent context overflow
- In-session caching per library+query+tokens

**API:** Context7 v2 REST API (`context7.com/api/v2`)
**Auth:** Optional `CONTEXT7_API_KEY` env var (increases rate limits from free tier)

---

## Background Shell

**Path:** `src/resources/extensions/bg-shell/`

Run shell commands in the background with lifecycle management.

### Tool: Background process management

- Spawn background processes with a TUI widget showing status
- Lifecycle hooks for process start/stop
- Widget refresh on state changes

### Command: `/bg-shell`

Interact with background processes.

---

## Async Jobs

**Path:** `src/resources/extensions/async-jobs/`

Allows bash commands to run in the background. The agent gets a job ID immediately and can continue working.

### Tools

| Tool | Description |
|------|-------------|
| `async_bash` | Run a command in the background, get a job ID |
| `await_job` | Wait for background jobs to complete, get results |
| `cancel_job` | Cancel a running background job |

### Command: `/jobs`

Show running and recent background jobs.

### Architecture

- Results delivered via follow-up messages when jobs complete
- Truncated output in follow-ups (2000 char limit), full output via `await_job`
- Job manager tracks status, timing, and output for each background process

---

## Subagent

**Path:** `src/resources/extensions/subagent/`

Delegate tasks to specialized agents with isolated context windows. Each subagent is a separate `pi` process with its own tools, model, and system prompt.

### Tool: `subagent`

Three execution modes:

| Mode | Parameters | Description |
|------|-----------|-------------|
| **Single** | `{ agent, task }` | One agent, one task |
| **Parallel** | `{ tasks: [{agent, task}, ...] }` | Up to 8 tasks, 4 concurrent |
| **Chain** | `{ chain: [{agent, task}, ...] }` | Sequential with `{previous}` placeholder |

**Key features:**
- Isolated filesystem via git worktree (optional `isolated: true`)
- Delta patch capture and merge for isolated runs
- Automatic retry of failed parallel tasks (1 retry)
- Worker registry for batch tracking
- Streaming updates with per-task progress
- Rich TUI rendering: collapsed/expanded views, tool call previews, usage stats
- Agent scope control: `user`, `project`, or `both`
- Confirmation prompt for project-local agents (security)

### Agent Discovery

Agents are `.md` files with YAML frontmatter, discovered from:
- **User agents:** `~/.gsd/agent/agents/`
- **Project agents:** `.pi/agents/` or `.gsd/agents/` (nearest ancestor)

Frontmatter fields:
```yaml
---
name: scout
description: Fast codebase recon
tools: read, grep, find, ls, bash
model: sonnet
---
```

As of v2.66.1, the `tools` field also supports list-style syntax:
```yaml
---
tools:
  - read
  - grep
  - bash
---
```

### Command: `/subagent`

List available subagents with their source and description.

---

## Bundled Agents

**Path:** `src/resources/agents/`

Five agent definitions ship with GSD-2:

### scout

Fast codebase reconnaissance. Returns compressed, structured context for handoff to other agents.

- **Tools:** read, grep, find, ls, bash
- **Output format:** Files Retrieved, Key Code, Architecture, Start Here
- **Thoroughness levels:** Quick, Medium, Thorough

### researcher

Web researcher that finds and synthesizes current information using web search.

- **Tools:** web_search, bash
- **Strategy:** 2-3 targeted search queries, synthesize, cite sources
- **Output format:** Summary, Key Findings (with source URLs), Sources

### worker

General-purpose subagent with full capabilities and isolated context.

- **Tools:** All available
- **Constraint:** Does not spawn subagents or act as orchestrator unless explicitly instructed
- **Output format:** Completed, Files Changed, Notes

### javascript-pro

Modern JavaScript specialist (ES2023+, Node.js 20+) for browser, backend, and full-stack applications.

- **Model:** sonnet
- **Memory:** project-scoped persistent memory
- **Capabilities:** Async patterns, performance optimization, module design, testing, security

### typescript-pro

TypeScript specialist for advanced type system patterns, complex generics, and end-to-end type safety.

- **Model:** sonnet
- **Memory:** project-scoped persistent memory
- **Capabilities:** Type-first development, strict mode, branded types, discriminated unions, build tooling

### Codebase Map (v2.58.0)

The `/gsd codebase generate` command produces `.gsd/CODEBASE.md`, a structured inventory of source files with directory groupings, file descriptions, and coverage statistics. Agents can reference this map for faster orientation. Incremental updates (`/gsd codebase update`) preserve existing descriptions and add only new files.

### Stale Commit Safety Check (v2.58.0)

Doctor's git checks detect uncommitted changes that have been idle beyond a configurable threshold. When auto-fix is enabled, a snapshot commit is created to prevent data loss. These snapshot commits are auto-cleaned during `/gsd cleanup` or worktree teardown, so they never pollute the final history.

---

## Mac Tools

**Path:** `src/resources/extensions/mac-tools/`

macOS automation via a Swift CLI that interfaces with Accessibility APIs, NSWorkspace, and CGWindowList.

### Architecture

- Swift CLI (`swift-cli/`) handles all macOS API calls
- JSON protocol: stdin `{ command, params }` -> stdout `{ success, data?, error? }`
- TypeScript extension invokes CLI per-command via `execFileSync`
- Mtime-based compilation caching: recompiles only when source files change

### Tools

| Tool | Description |
|------|-------------|
| `mac_check_permissions` | Check Accessibility and Screen Recording permissions |
| `mac_list_apps` | List running macOS applications |
| `mac_launch_app` | Launch an app by name or bundle ID |
| `mac_activate_app` | Bring a running app to the front |
| `mac_quit_app` | Quit a running application |
| `mac_list_windows` | List on-screen windows for an app (returns window IDs) |
| `mac_find` | Find UI elements by role/title/value/identifier (search, tree, focused modes) |
| `mac_get_tree` | Compact accessibility tree of an app's UI structure |
| `mac_click` | Click a UI element via AXPress |
| `mac_type` | Type text into a UI element by setting AXValue |
| `mac_screenshot` | Screenshot a window by ID (JPEG/PNG, optional retina) |
| `mac_read` | Read accessibility attributes from a UI element |

### Interaction Pattern

1. **Discover:** `mac_find` (search) or `mac_get_tree` (structure overview)
2. **Act:** `mac_click` (press buttons/menus) or `mac_type` (enter text)
3. **Verify:** `mac_read` (check attributes) or `mac_screenshot` (visual confirmation)

**Distinction from browser-tools:** Mac tools control native macOS apps (Finder, TextEdit, Xcode). Browser tools control web page content inside browsers.

---

## MCP Client

**Path:** `src/resources/extensions/mcp-client/`

Native MCP (Model Context Protocol) server integration, replacing the former MCPorter external dependency (v2.30.0). Connects to external MCP servers configured in project files using the `@modelcontextprotocol/sdk` Client directly -- no external CLI required.

### Tools

| Tool | Description |
|------|-------------|
| `mcp_servers` | List available MCP servers from `.mcp.json` or `.gsd/mcp.json` |
| `mcp_discover` | Get tool signatures for a specific server (lazy connect) |
| `mcp_call` | Call a tool on an MCP server (lazy connect) |

### Configuration

MCP servers are configured in project files:
- `.mcp.json` (project root)
- `.gsd/mcp.json` (per-project, v2.29.0)

Supports two transport types:
- **stdio:** Local process with command + args + env
- **http:** Remote server via Streamable HTTP transport with OAuth auth provider (v2.64.0)

### Environment Variable Resolution (v2.39.0)

`${VAR}` references in `.mcp.json` `env` fields are resolved from `process.env` at connection time. This allows secrets to be stored in shell profile or `.env` files rather than committed to the MCP config.

### `/gsd mcp` Command (v2.45.0)

Dedicated command for MCP server status and connectivity diagnostics:
- Lists all configured servers with transport type and connection state
- Tests connectivity to each server
- Shows discovered tools per server

### Connection Management

- Lazy connection: servers are only connected when first used
- Connection pooling per server name
- Automatic cleanup on session shutdown
- Config cache invalidation on session switch
- Output truncation for large results

---

## MCP Server

**Path:** `packages/mcp-server/`

GSD exposes its orchestration, project-state, and workflow capabilities as an MCP server (v2.63.0, workflow tools v2.67.0). External agents (Claude Code, Cursor, etc.) can connect to GSD over MCP and drive workflow operations without using the TUI.

### Session Management Tools (6)

| Tool | Description |
|------|-------------|
| `gsd_execute` | Start a GSD auto-mode session for a project directory; returns a `sessionId` |
| `gsd_status` | Poll session status including progress, recent events, pending blockers, and cost |
| `gsd_result` | Get accumulated session result (partial if still running) |
| `gsd_cancel` | Cancel a running session and abort the current operation |
| `gsd_query` | Read project state from filesystem (STATE.md, PROJECT.md, requirements, milestones); no session required |
| `gsd_resolve_blocker` | Resolve a pending blocker by sending a response |

### Read-Only Project State Tools (6, v2.63.0)

Pure filesystem reads that require no active session:

| Tool | Description |
|------|-------------|
| `gsd_progress` | Structured progress metrics: active milestone/slice/task, phase, completion counts, blockers, next action |
| `gsd_roadmap` | Full project roadmap structure with milestones, slices, tasks, status, risk, and dependencies; optional milestone filter |
| `gsd_history` | Execution history with cost, token usage, model, and duration per unit |
| `gsd_doctor` | Lightweight structural health check on `.gsd/` (missing files, status inconsistencies, orphaned state) |
| `gsd_captures` | Captured ideas and thoughts from CAPTURES.md with triage status filtering |
| `gsd_knowledge` | Project knowledge base: rules, patterns, and lessons learned |

### Workflow Management Tools (17, v2.67.0)

Mutation tools that execute GSD workflow operations. Each tool validates `projectDir` against `GSD_WORKFLOW_PROJECT_ROOT` for path safety. Write-gate enforcement blocks operations that violate discussion or queue phase constraints. Tools are registered with both canonical and alias names for ergonomic access:

| Tool | Alias | Description |
|------|-------|-------------|
| `gsd_plan_milestone` | -- | Plan a new milestone with slices, risks, proof strategy, and verification contract |
| `gsd_plan_slice` | -- | Plan a slice within a milestone with tasks, success criteria, and proof level |
| `gsd_replan_slice` | `gsd_slice_replan` | Replan a slice after discovering a blocker |
| `gsd_slice_complete` | `gsd_complete_slice` | Mark a slice as complete with summary, verification, and requirement outcomes |
| `gsd_complete_milestone` | `gsd_milestone_complete` | Mark a milestone as complete with narrative, verification, and lessons learned |
| `gsd_validate_milestone` | `gsd_milestone_validate` | Run milestone validation with verdict, remediation plan, and verification classes |
| `gsd_reassess_roadmap` | `gsd_roadmap_reassess` | Reassess the roadmap after a slice completes (modify, add, or remove slices) |
| `gsd_save_gate_result` | -- | Save a quality gate result (pass/flag/omitted) with rationale |
| `gsd_summary_save` | -- | Save a summary artifact (SUMMARY, CONTEXT, PLAN, etc.) |
| `gsd_task_complete` | `gsd_complete_task` | Mark a task as complete with verification evidence |
| `gsd_milestone_status` | -- | Get detailed status for a specific milestone |

### OAuth Auth Provider (v2.64.0)

The MCP client's HTTP transport now supports an OAuth auth provider, enabling authenticated connections to remote MCP servers that use OAuth-based access control.

### Architecture

- Dynamic imports for `@modelcontextprotocol/sdk` to work around TS Node16 subpath export resolution
- Serialized workflow execution queue prevents concurrent mutations
- Write-gate enforcement loaded from GSD's bootstrap module
- `GSD_WORKFLOW_PROJECT_ROOT` env var restricts operations to a specific directory tree
- `GSD_WORKFLOW_EXECUTORS_MODULE` env var allows custom executor module injection

---

## Voice

**Path:** `src/resources/extensions/voice/`

Voice transcription input for the TUI.

### Platforms

| Platform | Backend | Requirements |
|----------|---------|-------------|
| **macOS** | Swift speech recognizer (compiled on first use) | Xcode CLI tools |
| **Linux** | Python script with Groq API | `GROQ_API_KEY`, `python3`, `libportaudio2` |

### Usage

- **Command:** `/voice`
- **Shortcut:** `Ctrl+Alt+V`
- Toggle on/off; transcribed text fills the editor
- Visual indicator: flashing red dot + "transcribing" in footer
- End with Escape or Enter

### Architecture

- macOS: Swift binary using Speech and AVFoundation frameworks
- Linux: Python script with `sounddevice` for recording, Groq Whisper API for transcription
- Auto-installs Python dependencies in `~/.gsd/voice-venv/` on first run (Linux)
- Accumulates text across pause-induced transcription resets

---

## GitHub Sync

**Path:** `src/resources/extensions/github-sync/`

Opt-in extension that syncs GSD lifecycle events to GitHub (v2.39.0). Maps GSD's internal workflow hierarchy to native GitHub objects for team visibility and project tracking.

### Sync Mapping

| GSD Concept | GitHub Object |
|-------------|---------------|
| Milestone | GitHub Milestone + tracking issue |
| Slice | Draft PR |
| Task | Sub-issue with auto-close on commit |

### How It Works

1. Integration hooks into `auto-post-unit.ts` via dynamic import
2. On milestone creation, creates a GitHub Milestone and a tracking issue
3. On slice creation, creates a draft PR linked to the milestone
4. On task completion, closes the corresponding sub-issue
5. Mapping state is persisted in `.gsd/github-sync-mapping.json`

### Commands

| Command | Description |
|---------|-------------|
| `/github-sync` or `/github-sync bootstrap` | Initialize sync -- creates GitHub objects for existing milestones/slices/tasks |
| `/github-sync status` | Show sync mapping counts and open/closed state |

### Requirements

- `gh` CLI installed and authenticated
- Repository must be a GitHub remote

---

## Slash Commands

**Path:** `src/resources/extensions/slash-commands/`

Utility slash commands for project scaffolding and auditing.

### Commands

| Command | Description |
|---------|-------------|
| `/create-slash-command` | Generate a new slash command extension from a plain-English description. Uses an interview wizard to gather purpose, trigger type, and behavior. |
| `/create-extension` | Scaffold a new pi extension |
| `/audit` | Audit the current codebase against a specific goal. Writes a structured report to `.gsd/audits/`. Read-only exploration -- does not modify code. |
| `/clear` | Clear session state |

---

## Ask User Questions

**Path:** `src/resources/extensions/ask-user-questions.ts`

### Tool: `ask_user_questions`

LLM-callable tool for presenting structured questions to the user. Presents 1-3 questions with multiple-choice options in a TUI interview widget.

**Features:**
- Single-select (2-3 options) with automatic "None of the above" free-form option
- Multi-select (any number of options, user toggles with SPACE)
- Option descriptions for explaining impact/tradeoffs
- Recommended option highlighting (first option with "(Recommended)" suffix)
- Falls back to remote questions (Slack/Discord/Telegram) in headless mode
- Falls back to sequential `ctx.ui.select()` in RPC mode

---

## Secure Env Collect

**Path:** `src/resources/extensions/get-secrets-from-user.ts`

### Tool: `secure_env_collect`

Secure collection of environment variables through a paged masked-input TUI.

**Features:**
- One key per page with masked display (e.g., `sk-ir***dgdh`)
- Per-key guidance steps (numbered instructions for finding the key)
- Format hints shown to user
- Three write destinations: `.env` (local), Vercel, Convex
- Auto-detection of destination based on project files (`vercel.json`, `convex/` directory)
- Values never echoed in tool output
- Skip with Ctrl+S or Escape

**Manifest orchestration:** Can read from a secrets manifest file (per-milestone), check existing keys against `.env` and `process.env`, show a summary screen, and update manifest status after collection.

---

## Remote Questions

**Path:** `src/resources/extensions/remote-questions/`

Route `ask_user_questions` prompts to Slack, Discord, or Telegram when running in headless auto-mode.

### Local TUI Racing (v2.67.0)

In interactive mode with a remote channel configured, the system now races the local TUI prompt against the remote channel instead of routing exclusively to the remote. If the remote answer arrives first, the local TUI is automatically cancelled and the remote response is used. This means users can answer from either the terminal or their phone -- whichever is faster wins.

### Supported Channels

| Channel | Message Format | Response Detection |
|---------|---------------|-------------------|
| **Discord** | Rich embeds with fields | Reaction emojis (1-5) or message replies |
| **Slack** | Block Kit messages | Reaction emojis or thread replies |
| **Telegram** | Formatted messages | Message replies |

### Setup

- `/gsd remote discord` -- Interactive setup wizard (bot token, server, channel, test message)
- `/gsd remote slack` -- Interactive setup wizard (bot token, channel, test message)
- `/gsd remote telegram` -- Interactive setup wizard (bot token, chat ID, test message)
- `/gsd remote status` -- Show current configuration
- `/gsd remote disconnect` -- Remove configuration

### Configuration

Stored in `~/.gsd/PREFERENCES.md`:
```yaml
remote_questions:
  channel: discord
  channel_id: "1234567890123456789"
  timeout_minutes: 5
  poll_interval_seconds: 5
```

### Response Formats

- **Single question:** React with number emoji or reply with a number
- **Multiple questions:** Reply with semicolons (`1;2;custom text`) or newlines
- **Free text:** Captured as user notes
- Timeout after `timeout_minutes` -- LLM handles timeouts contextually

### Exported API

```typescript
export { handleRemote } from "./remote-command.js";
export { createPromptRecord, writePromptRecord } from "./store.js";
export { getLatestPromptSummary } from "./status.js";
export { parseSlackReply, parseDiscordResponse, formatForDiscord, formatForSlack,
         parseSlackReactionResponse, formatForTelegram, parseTelegramResponse } from "./format.js";
export { resolveRemoteConfig, isValidChannelId } from "./config.js";
export { sendRemoteNotification } from "./notify.js";
```

---

## Universal Config

**Path:** `src/resources/extensions/universal-config/`

Auto-detects and displays configuration from 8 AI coding tools. Read-only -- never modifies other tools' config files.

### Scanned Tools

Claude Code, Cursor, Windsurf, Gemini CLI, Codex, Cline, GitHub Copilot, VS Code.

### Discovered Items

- MCP servers (reusable across tools)
- Rules/instructions
- Context files
- Settings
- Claude skills
- Claude plugins

### Tool: `discover_configs`

LLM-callable tool that scans both user-level (`~/`) and project-level (`./`) config directories.

- Filter to a specific tool with the `tool` parameter
- In-session caching with optional `refresh` parameter

### Command: `/configs`

Show discovered AI tool configurations in the TUI.

---

## AWS Auth

**Path:** `src/resources/extensions/aws-auth/`

Automatically refreshes AWS credentials when Bedrock API requests fail with authentication/token errors (v2.31.0).

### How It Works

1. Hooks into `agent_end` to check if the last assistant message failed with an AWS auth error
2. Runs the configured `awsAuthRefresh` command (e.g., `aws sso login`)
3. Streams the SSO auth URL and verification code to the TUI
4. Calls `retryLastTurn()` to re-run from the user's original message

### Activation

Completely inert unless BOTH conditions are met:
- A Bedrock API request fails with a recognized AWS auth error
- `awsAuthRefresh` is configured in `settings.json`

### Setup

Add to `~/.gsd/agent/settings.json`:
```json
{ "awsAuthRefresh": "aws sso login --profile my-profile" }
```

### Matched Error Patterns

ExpiredTokenException, expired security tokens, SSO session expired, unable to locate credentials, UnrecognizedClientException, SSOTokenProviderFailure, and more.

---

## TTSR

**Path:** `src/resources/extensions/ttsr/`

**Time Traveling Stream Rules** -- zero-context-cost guardrails that monitor streaming output against regex patterns.

### How It Works

1. Rules are loaded from disk on `session_start`
2. During streaming, each text/thinking/toolcall delta is checked against rule patterns
3. On match: abort stream, set pending violation
4. On `agent_end`: inject the matched rule as a system reminder and trigger a new turn
5. The LLM retries with the rule now visible in context

### Key Properties

- **Zero context cost until fired:** Rules are regex patterns checked against stream deltas, not injected into every prompt
- **Per-stream buffering:** Separate buffers for text, thinking, and each tool call
- **Tool-aware:** Can match against tool call arguments including file paths
- **Rule deduplication:** Rules are marked as injected after firing to prevent re-injection
- **Message count tracking:** Rules can be configured to fire only after N messages

### Hook Lifecycle

| Hook | Action |
|------|--------|
| `session_start` | Load rules, populate manager |
| `turn_start` | Reset buffers |
| `message_update` | Check delta against rules, abort on match |
| `turn_end` | Increment message count |
| `agent_end` | If pending violation, inject rule via `sendMessage` |

---

## Skill System

Skills are specialized instruction sets that GSD loads when the task matches. They provide domain-specific guidance -- coding patterns, framework idioms, testing strategies, and tool usage.

### Skill Locations

- **Primary user skills:** `~/.agents/skills/<skill-name>/SKILL.md` (v2.51.0)
- **Legacy user skills:** `~/.gsd/agent/skills/<skill-name>/SKILL.md` (migrated automatically)
- **Project skills:** `.pi/agent/skills/<skill-name>/SKILL.md`
- User skills take precedence over project skills

### Migration from Legacy Path (v2.51.0)

Skills previously stored in `~/.gsd/agent/skills/` are auto-migrated to `~/.agents/skills/` on first access. The migration copies skill directories, prioritizes the ecosystem (curated) catalog over legacy copies, and skips already-migrated skills. After migration, the legacy directory is deprioritized in discovery.

### Curated Catalog (v2.51.0)

GSD ships with a curated skill catalog containing 30 skill packs with 40+ curated skills. The catalog is organized by ecosystem detection -- project dependencies, file patterns, and framework markers determine which skills are surfaced.

| Category | Packs |
|----------|-------|
| **Web frameworks** | React, Next.js, Vue, Svelte, Angular, Remix |
| **Backend** | FastAPI, Django, Express, NestJS, Rails |
| **Languages** | Rust, Go, Python, TypeScript, Swift |
| **Mobile** | SwiftUI, React Native, Flutter |
| **Databases** | SQLite/SQL, Prisma, Supabase/Postgres, Redis |
| **Cloud** | Firebase, Azure, AWS |
| **Quality** | Testing, security audit, code review, linting |
| **DevOps** | GitHub Actions, Docker security |
| **Desktop** | Tauri, Electron |

Skills are matched by scanning `package.json`, `Cargo.toml`, `go.mod`, `requirements.txt`, `Podfile`, directory structures, and other ecosystem markers.

### Platform-Aware iOS Skill Matching (v2.51.0)

For iOS/macOS projects, the skill matcher parses `SDKROOT` from `.pbxproj` files to distinguish between iOS, macOS, watchOS, and tvOS targets. This prevents SwiftUI-iOS skills from activating on macOS-only projects and vice versa.

### Bundled Skills

| Skill | Trigger | Description |
|-------|---------|-------------|
| `frontend-design` | Web UI work | Production-grade frontend with high design quality |
| `swiftui` | macOS/iOS apps | Full lifecycle from creation to shipping |
| `debug-like-expert` | Complex debugging | Methodical investigation with evidence gathering |
| `rust-core` | Rust code | Idiomatic, safe, performant Rust patterns |
| `axum-web-framework` | Axum web apps | Complete Axum development guide |
| `axum-tests` | Testing Axum apps | Test patterns for Axum applications |
| `tauri` | Tauri v2 desktop apps | Cross-platform desktop app development |
| `tauri-ipc-developer` | Tauri IPC | React-Rust type-safe communication |
| `tauri-devtools` | Tauri debugging | CrabNebula DevTools integration |
| `github-workflows` | GitHub Actions | CI/CD, workflow debugging, live syntax |
| `security-audit` | Security auditing | Dependency scanning, OWASP |
| `security-review` | Code security review | Injection, XSS, auth flaws |
| `security-docker` | Docker security | Dockerfile, runtime hardening |
| `review` | Code review | Diff-aware code review with quality analysis |
| `test` | Test generation/execution | Auto-detects frameworks, failure analysis |
| `lint` | Linting and formatting | ESLint, Biome, Prettier auto-detect |

### Skill Discovery Modes

| Mode | Behavior |
|------|----------|
| `auto` | Skills found and applied automatically |
| `suggest` | Skills identified but require confirmation (default) |
| `off` | No skill discovery |

### Skill Tool Resolution (v2.40.0)

The Skill tool (`pi.registerTool("Skill", ...)`) is the LLM-callable interface for activating skills during a session. When the agent calls the Skill tool, it resolves the skill name against the catalog, loads the SKILL.md, and injects it into the system prompt. This replaces the earlier heuristic-based activation that triggered skills based on keyword matching in task descriptions.

### `create-gsd-extension` Skill (v2.30.0)

A built-in skill that guides the agent through creating a new GSD extension from scratch -- scaffolding the directory, writing the `package.json` manifest, creating the `index.ts` entry point with tool/command/hook registrations, and testing the extension loads correctly.

### SAFE_SKILL_NAME Guard (v2.50.0)

Skill names are validated against a strict allowlist pattern before activation. This prevents prompt injection via crafted skill names that could contain system prompt overrides or instruction fragments. Names must match `^[a-zA-Z0-9_-]+$`.

### Custom Skills

Create a directory with a `SKILL.md` file:
```
~/.agents/skills/my-skill/
  SKILL.md           -- instructions for the LLM
  references/        -- optional reference files
```

### Skill Preferences

Control via `~/.gsd/PREFERENCES.md`:
```yaml
always_use_skills:
  - debug-like-expert
prefer_skills:
  - frontend-design
avoid_skills:
  - security-docker
skill_rules:
  - when: task involves Clerk authentication
    use: [clerk]
```

### Skill Health

**`/gsd skill-health`** shows:
- Usage count, success rate, token consumption, trend
- Flags: success rate below 70%, token usage rising 20%+, stale (unused beyond threshold)
- Configurable staleness (`skill_staleness_days`, default 60)

### Heal-Skill

Post-unit hook that analyzes whether the agent deviated from a skill's instructions. Writes proposed fixes to `.gsd/skill-review-queue.md` for human review. Skills are never auto-modified.

---

## BTW Skill

**Command:** `/btw`

Ephemeral side-question skill (v2.60.0). Allows users to ask quick tangential questions without derailing the current conversation context. The `/btw` command opens a lightweight interaction that draws on the existing conversation for context but does not persist its output into the main session history. Useful for clarifying a concept, checking syntax, or getting a quick answer mid-task.

---

## Dashboard and Visualizer

### Dashboard Overlay

**Shortcut:** `Ctrl+Alt+G`
**Command:** `/gsd status`

Full-screen TUI overlay showing:
- Milestone/slice/task breakdown with completion status
- Current unit and completed units
- Timing and cost metrics
- Activity log
- Worker batch status (from parallel orchestration)
- Environment health checks
- Progress score

### Workflow Visualizer

**Command:** `/gsd visualize`

10-tab full-screen TUI overlay:

| Tab | Content |
|-----|---------|
| **Progress** | Tree view of milestones, slices, tasks with completion status |
| **Dependencies** | ASCII dependency graph showing slice relationships |
| **Metrics** | Bar charts: cost by phase, by slice, by model |
| **Timeline** | Chronological execution history with unit type, duration, model, tokens |
| Additional tabs | Health, agent, changes, knowledge, captures, export |

**Controls:**
- `Tab` / `Shift+Tab` / `1`-`4` for tab navigation
- Arrow keys for scrolling
- `Escape` / `q` to close
- Auto-refresh every 2 seconds

### HTML Export

**Command:** `/gsd export --html`

Generates self-contained HTML reports in `.gsd/reports/`:
- Progress tree, dependency graph (SVG DAG), cost/token bar charts
- Execution timeline, changelog, knowledge base
- All CSS and JS inlined -- no external dependencies
- Auto-generated `index.html` with progression metrics across milestones
- Printable to PDF from any browser

**Auto-generation:** `auto_report: true` (default) generates after milestone completion.

### Persistent Notification Panel (v2.65.0)

A persistent notification system with three presentation surfaces:

- **TUI overlay:** Full notification overlay with backdrop dimming and viewport padding. Content-fit sizing wraps long messages and adapts overlay height to content. Notifications display as a stack with most recent on top.
- **Widget:** Compact notification indicator in the dashboard footer area for unread notifications.
- **Web API:** Notification state exposed through the web dashboard API for external consumption.

**Lock ownership:** Notification locks include ownership metadata. The system only unlinks locks owned by the current process, preventing foreign lock deletion in multi-session scenarios.

---

## Anthropic Vertex Provider (v2.38.0)

Claude on Vertex AI. Registered as the `anthropic-vertex` provider, giving users access to Claude models through Google Cloud's Vertex AI platform. Detected automatically by the provider doctor.

Requires Google Cloud credentials configured for Vertex AI access. Uses the Anthropic Messages API format routed through the Vertex AI endpoint.
