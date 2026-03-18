# GSD-2 Overview

## What Is GSD-2?

GSD-2 ("Get Shit Done 2") is a standalone coding agent CLI. It is the second-generation evolution of the original [Get Shit Done](https://github.com/gsd-build/get-shit-done) project, which was a prompt framework for Claude Code that went viral for its practical, opinionated approach to AI-assisted development.

Where v1 was a collection of markdown prompts injected into Claude Code's slash command system, v2 is a full TypeScript application. It does not rely on the LLM to follow instructions about context management, branching, crash recovery, or task sequencing — it controls those things directly through the Pi SDK. The result is an agent that can autonomously research, plan, execute, verify, commit, and advance through an entire milestone without human intervention.

The one-line pitch: **one command, walk away, come back to a built project with clean git history.**

## Current Release

| Field | Value |
|-------|-------|
| Package name | `gsd-pi` |
| Current version | **v2.28.0** |
| Release date | 2026-03-17 |
| License | MIT |
| Binary names | `gsd`, `gsd-cli` |
| Repository | [github.com/gsd-build/gsd-2](https://github.com/gsd-build/gsd-2) |

## Core Value Proposition vs. Other Agents

The fundamental difference between GSD-2 and most coding agents is one of architecture, not features.

Other agents are typically prompt frameworks or thin wrappers that ask the LLM to manage its own context, track progress, and recover from failure. GSD-2 is a TypeScript state machine that controls the agent session from the outside. This distinction matters because:

- **Context management is programmatic.** Every task gets a fresh 200k-token context window. There is no accumulated session garbage, no "I'll be more concise" degradation.
- **State lives on disk.** `.gsd/` files are the source of truth. Auto mode reads disk state to determine what to do next. A crashed session can be resumed because the state survived.
- **Automation is real.** "Auto mode" in v1 was the LLM calling itself in a loop. In v2, it is a state machine that dispatches fresh agent sessions, each scoped to one unit of work.
- **Verification is enforced.** Shell commands (`npm run lint`, `npm run test`) run automatically after task execution, with configurable retry on failure.

| Capability | v1 (Prompt Framework) | v2 (Agent Application) |
|---|---|---|
| Context management | Hope the LLM doesn't fill up | Fresh session per task, programmatic |
| Auto mode | LLM self-loop | State machine reading `.gsd/` files |
| Crash recovery | None | Lock files + session forensics |
| Git strategy | LLM writes git commands | Worktree isolation, sequential commits, squash merge |
| Cost tracking | None | Per-unit token/cost ledger with dashboard |
| Stuck detection | None | Retry once, then stop with diagnostics |
| Verification | Manual | Automated with auto-fix retries |
| Reporting | None | Self-contained HTML reports with metrics |
| Parallel execution | None | Multi-worker parallel milestone orchestration |

## Key Capabilities

### Auto Mode — Fully Autonomous Execution

`/gsd auto` is the headline feature. GSD researches, plans, executes, verifies, commits, and advances through every slice of a milestone with no human intervention. Each task gets a clean context window with exactly the right files pre-loaded. The state machine detects completion artifacts on disk to advance, handles stuck loops with diagnostics, supervises against soft/idle/hard timeouts, and can be interrupted at any time with Escape.

### Parallel Orchestration

Since v2.24.0, GSD supports running multiple workers across milestones simultaneously. A parallel orchestration layer manages worker processes via file-based IPC in `.gsd/parallel/`, tracks budget per worker, and persists orchestrator state with PID liveness detection so multi-worker sessions survive crashes.

### Terminal UI (TUI)

GSD ships with a full terminal interface built on the Pi SDK's TUI layer. The interface includes real-time response streaming, tool call visualization, session management, and a dashboard overlay (`Ctrl+Alt+G`) showing milestone/slice/task progress, per-unit cost and token breakdowns, cost projections, and parallel worker status.

### Multi-Provider AI

GSD is not locked to any single AI provider. It runs on the Pi SDK's model registry, which supports 20+ providers out of the box:

- Anthropic, OpenAI, Google Gemini, Google Vertex, Amazon Bedrock
- Azure OpenAI, GitHub Copilot, Groq, Cerebras, Mistral, xAI
- OpenRouter (access to hundreds of models with one key), Vercel AI Gateway, Hugging Face, and more

Different models can be assigned to different phases — Opus for planning, Sonnet for execution, a fast model for research. Each phase also supports a fallback chain: if the primary model fails due to a rate limit or provider outage, GSD automatically tries the next fallback.

### Extension System

GSD ships 14 bundled extensions that are loaded automatically:

| Extension | Provides |
|-----------|----------|
| GSD | Core workflow engine, auto mode, commands, dashboard |
| Browser Tools | Playwright browser with form intelligence, device emulation, visual diffing, test generation, PDF export |
| Search the Web | Brave Search, Tavily, or Jina page extraction |
| Google Search | Gemini-powered web search |
| Context7 | Up-to-date library and framework documentation |
| Background Shell | Long-running process management |
| Subagent | Delegated tasks with isolated context windows |
| Mac Tools | macOS native app automation via Accessibility APIs |
| MCPorter | Lazy on-demand MCP server integration |
| Voice | Real-time speech-to-text (macOS and Ubuntu 22.04+) |
| Slash Commands | Custom command creation |
| LSP | Language Server Protocol — diagnostics, go-to-definition, references, rename, code actions |
| Ask User Questions | Structured user input with single/multi-select |
| Secure Env Collect | Masked secret collection without manual `.env` editing |

### Studio

GSD's `package.json` includes a `studio` workspace, indicating a Studio mode is part of the package ecosystem.

### VS Code Extension

A VS Code extension (publisher: FluxLabs) connects to the GSD CLI via RPC, providing a `@gsd` chat participant, sidebar dashboard with connection status and token usage, and a command palette for starting/stopping the agent, switching models, and exporting sessions. Requires the `gsd-pi` CLI to be installed separately.

## The Pi Framework

GSD is built on the [Pi SDK](https://github.com/badlogic/pi-mono) (`@mariozechner/pi-coding-agent`). Pi is a terminal-native coding agent harness that sits between the user and an LLM, providing:

- A full session system with branching, compaction, and JSONL event persistence
- An event architecture that extensions can observe and modify at every stage
- A model registry covering 20+ providers with runtime model switching
- Four runtime modes (Interactive, Print, JSON, RPC)

Pi's design philosophy is "extend, don't fork." The core is deliberately minimal. Features like sub-agents, plan mode, permission gates, and MCP support are not baked in — they are built via extensions. An extension can override any built-in tool, replace the system prompt, modify every message sent to the LLM, replace the input editor, or register new model providers. GSD is itself a Pi extension (the largest one) layered on top of this platform.

### The Agent Loop

At the heart of every Pi-based session is an agent loop:

1. Assemble context (system prompt, previous messages or compaction summary, new user message)
2. Send to the LLM and stream the response
3. If the LLM emits tool calls: execute each tool, fire events, append results, and loop back to step 1
4. If the LLM stops calling tools: save messages to session and exit the loop

A single user prompt may trigger 1 turn or 50 turns depending on task complexity. GSD's auto mode layers a state machine on top of this loop, spawning a fresh agent loop per unit of work and reading disk state between loops.

## Four Modes of Operation

GSD inherits Pi's four runtime modes:

| Mode | Invocation | Use case |
|------|-----------|----------|
| **Interactive** | `gsd` | Full TUI — streaming responses, real-time tool calls, session management |
| **Print** | `pi -p "prompt"` | Non-interactive, sends a prompt, prints response, exits — for scripting |
| **JSON** | `--mode json` | Streams all events as JSON lines to stdout — for consuming output programmatically |
| **RPC** | `--mode rpc` | Full bidirectional JSON protocol over stdin/stdout — for IDE embedding (VS Code extension uses this) |

GSD adds a fifth operational layer: **Headless mode** (`gsd headless`), which wraps the Interactive mode but auto-responds to prompts, detects completion, and exits with structured exit codes (`0` complete, `1` error/timeout, `2` blocked). Designed for CI pipelines and scheduled automation.

## Work Hierarchy

GSD structures all projects into a three-level hierarchy:

```
Milestone  →  a shippable version (4-10 slices)
  Slice    →  one demoable vertical capability (1-7 tasks)
    Task   →  one context-window-sized unit of work
```

The iron rule: a task must fit in one context window. If it cannot, it becomes two tasks.

State lives entirely on disk in `.gsd/`:

```
.gsd/
  PROJECT.md          — what the project is right now
  REQUIREMENTS.md     — requirement contract (active/validated/deferred)
  DECISIONS.md        — append-only architectural decisions
  STATE.md            — quick-glance status
  milestones/
    M001/
      M001-ROADMAP.md
      M001-CONTEXT.md
      slices/
        S01/
          S01-PLAN.md
          S01-SUMMARY.md
          S01-UAT.md
          tasks/
            T01-PLAN.md
            T01-SUMMARY.md
```

## Package Ecosystem

GSD uses npm workspaces. The top-level `gsd-pi` package bundles:

- `@gsd/pi-tui` — terminal UI layer
- `@gsd/pi-ai` — AI provider integrations
- `@gsd/pi-agent-core` — core agent loop
- `@gsd/pi-coding-agent` — coding agent harness
- `@gsd/native` — platform-native binaries (grep acceleration, etc.)
- `studio` — Studio workspace

Platform-specific native engine packages are distributed as optional dependencies for macOS (arm64, x64), Linux (arm64, x64), and Windows (x64).

## Who GSD-2 Is For

GSD-2 is designed for developers who:

- Want to delegate entire milestones or features to an AI agent and return to working code
- Are comfortable with a terminal-first workflow
- Want full control over context, cost, and model selection without being locked into one provider
- Need CI/headless automation as a first-class use case
- Work in teams and need reproducible, file-based state shared through git

It is not a chat interface for asking coding questions. It is a build system for agentic software development.
