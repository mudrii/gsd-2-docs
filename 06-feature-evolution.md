# GSD-2 Feature Evolution

Tracks when major features were introduced and how they evolved across versions.

---

## Auto Mode

The core autonomous execution engine.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.3.3 | Mar 11 | `/gsd next` step mode introduced; `/gsd` defaults to step mode |
| v2.3.7 | Mar 11 | Auto-mode model switches no longer persist as user's global default; resume rebuilds disk state |
| v2.3.9 | Mar 12 | Idempotent unit dispatch, retry caps, atomic closeout (loop/instability fixes) |
| v2.5.1 | Mar 12 | Right-sized pipeline: single-slice milestones skip redundant sessions (9-10 to 5-6) |
| v2.11.1 | Mar 15 | URGENT fix: stale directory listing cache caused research/plan-slice loops |
| v2.12.0 | Mar 15 | Hook system for auto-mode state machine; post-unit hooks |
| v2.13.0 | Mar 15 | Dispatch loop root causes fixed (parse cache, completion persistence, recovery counter) |
| v2.14.0 | Mar 15 | Branchless worktree architecture; ~2600 lines of merge code removed |
| v2.15.0 | Mar 15 | Pipeline-aware prompts (each phase knows its role); dispatch table replaces if-else chain |
| v2.17.0 | Mar 15 | Token optimization profiles (budget/balanced/quality, 40-60% savings) |
| v2.19.0 | Mar 16 | Mid-execution capture and triage (`/gsd capture`) |
| v2.24.0 | Mar 16 | Parallel milestone orchestration (multi-worker) |
| v2.29.0 | Mar 18 | Three-stage promotion pipeline (Dev/Test/Prod) |
| v2.30.0 | Mar 18 | Simplified pipeline: merged research into planning (ADR-003) |
| v2.33.0 | Mar 19 | Live regression test harness for post-build pipeline validation |

---

## Worktree Isolation

Git worktree-based isolation for auto-mode execution.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.3.0 | Mar 11 | `/worktree` (`/wt`) git worktree lifecycle management |
| v2.3.8 | Mar 11 | Worktree file operations resolve paths against active working directory |
| v2.13.0 | Mar 15 | **Introduced:** worktree isolation for auto-mode (isolated worktrees per milestone) |
| v2.13.1 | Mar 15 | Windows worktree path normalization |
| v2.14.0 | Mar 15 | **Branchless architecture:** eliminated slice branches; sequential commits on `milestone/<MID>` |
| v2.14.3 | Mar 15 | Copy planning artifacts into new auto-worktrees |
| v2.14.4 | Mar 15 | Session cwd update for worktree-aware LLM prompts |
| v2.18.0 | Mar 16 | Stale worktree cwd fix (three-layer); worktree integration branch resolution |
| v2.19.0 | Mar 16 | Absolute paths in auto-mode prompts for worktree contexts |
| v2.22.0 | Mar 16 | `git.isolation: "none"` option; worktree state synced to project root |
| v2.29.0 | Mar 18 | Worktree sync, branch-mode merge fallback, living docs sync |
| v2.31.0 | Mar 18 | `-w`/`--worktree` CLI flag for isolated worktree sessions |
| v2.33.0 | Mar 19 | Resolve main repo root in worktrees for stable identity hash |

---

## Crash Recovery

Session survival and recovery after interruptions.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.2.9 | Mar 11 | Idle recovery skips stuck units instead of stalling |
| v2.14.1 | Mar 15 | Dispatch recovery hardening (artifact fallback, TUI freeze prevention, atomic writes) |
| v2.15.1 | Mar 15 | Recovery hardened: checkbox verification, corrupt roadmaps, atomic writes |
| v2.25.0 | Mar 16 | Phantom skip loop from stale crash recovery context fixed |
| v2.27.0 | Mar 17 | **Major:** crash recovery for parallel orchestrator (persisted state, PID liveness detection) |
| v2.28.0 | Mar 17 | Failure recovery safeguards (atomic writes, OAuth timeouts, RPC exit detection) |
| v2.32.0 | Mar 19 | Skip crash recovery when auto.lock was written by current process |
| v2.33.0 | Mar 19 | Actionable recovery guidance in crash info messages |

---

## Parallel Orchestration

Running multiple workers across milestones simultaneously.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.12.0 | Mar 15 | Parallel tool calling (tools from single message execute concurrently) |
| v2.24.0 | Mar 16 | **Introduced:** parallel milestone orchestration with multi-worker support |
| v2.24.0 | Mar 16 | Dashboard view for parallel workers with 80% budget alert |
| v2.26.0 | Mar 17 | Worker NDJSON monitoring and budget enforcement |
| v2.27.0 | Mar 17 | Crash recovery for parallel orchestrator (persisted state, PID liveness) |

---

## Token Optimization

Reducing token consumption and cost management.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.4.0 | Mar 12 | System prompt compressed by 48% (360 to 187 lines) |
| v2.5.1 | Mar 12 | Heavyweight plan sections conditional (omitted for simple slices) |
| v2.12.0 | Mar 15 | Inline static templates into prompt builders (~44 fewer READ calls per milestone) |
| v2.15.0 | Mar 15 | Pipeline-aware prompts; research depth calibration (deep/targeted/light) |
| v2.17.0 | Mar 15 | **Introduced:** token optimization profiles (budget/balanced/quality, 40-60% savings) |
| v2.17.0 | Mar 15 | Complexity-based task routing with persistent learning |
| v2.19.0 | Mar 16 | Dynamic model routing with budget-pressure awareness and adaptive learning |
| v2.20.0 | Mar 16 | Context-window budget engine (proportional prompt sizing by relevance) |
| v2.29.0 | Mar 18 | Token optimization suite (prompt caching, compression, smart context selection); semantic chunking |
| v2.29.0 | Mar 18 | Cache-ordered prompt assembly and dashboard cache hit rate |

---

## Model Routing

Selecting and routing to different models based on context.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.7.1 | Mar 13 | Model fallback support for auto-mode phases |
| v2.8.3 | Mar 13 | Provider-aware model resolution for per-phase preferences |
| v2.10.10 | Mar 14 | Alibaba Cloud coding-plan provider support |
| v2.11.0 | Mar 14 | Cross-provider fallback on rate/quota limits; fallback model rotation |
| v2.17.0 | Mar 15 | Complexity-based routing (simple/standard/heavy); persistent routing history |
| v2.18.0 | Mar 16 | Dynamic model discovery (runtime enumeration from provider APIs with TTL caching) |
| v2.19.0 | Mar 16 | **Full system:** dynamic model routing with budget-pressure awareness, cross-provider cost comparison, adaptive learning |
| v2.26.0 | Mar 17 | Model selector grouped by provider with type and API docs fields |
| v2.29.0 | Mar 18 | Model list grouped by provider in prefs wizard |
| v2.30.0 | Mar 18 | Two-step provider-then-model picker in preferences wizard |

---

## Verification System

Automated verification of task outputs.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.5.0 | Mar 12 | Merge guards prevent slice completion with uncommitted changes |
| v2.8.1 | Mar 13 | UAT artifact verified before marking complete-slice done |
| v2.8.3 | Mar 13 | Execute-task artifact verification with self-repair |
| v2.13.0 | Mar 15 | Non-execute-task units now verify artifacts on disk |
| v2.15.0 | Mar 15 | Preferences schema validation; pipeline-aware prompts |
| v2.24.0 | Mar 16 | `validate-milestone` phase and dispatch |
| v2.26.0 | Mar 17 | Replan-slice artifact verification breaks infinite replanning loop |
| v2.27.0 | Mar 17 | **Verification enforcement gate** for milestone completion |
| v2.29.0 | Mar 18 | Per-milestone depth verification + queue-flow write-gate |
| v2.29.0 | Mar 18 | Non-blocking verification gate for auto-discovered commands |

---

## Skill System

Discovery, loading, and authoring of skills.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.1.6 | Mar 11 | Bundled skills (frontend-design, swiftui, debug-like-expert); skills trigger table |
| v2.20.0 | Mar 16 | Skill lifecycle management (telemetry, health dashboard, heal-skill) |
| v2.22.0 | Mar 16 | `/review`, `/test`, `/lint` skills |
| v2.29.0 | Mar 18 | 10 bundled skills for UI, quality, and code optimization |
| v2.30.0 | Mar 18 | **Built-in skill authoring system** (ADR-003); `create-gsd-extension` skill |

---

## Dashboard

Progress visualization and monitoring.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.3.9 | Mar 12 | Auto-mode progress widget shows all stats; footer hidden during auto-mode |
| v2.15.1 | Mar 15 | Progress widget refreshes from disk every 5 seconds |
| v2.19.0 | Mar 16 | **Workflow visualizer:** full-screen TUI overlay (Progress, Dependencies, Metrics, Timeline tabs) |
| v2.20.0 | Mar 16 | `/gsd status` dashboard; `Ctrl+Alt+G` toggle |
| v2.24.0 | Mar 16 | Dashboard view for parallel workers with 80% budget alert |
| v2.25.0 | Mar 16 | Visualizer enriched with stats and discussion status |
| v2.26.0 | Mar 17 | Discussion status indicators in `/gsd discuss` |
| v2.30.0 | Mar 18 | Model health indicator in auto-mode progress widget |
| v2.32.0 | Mar 19 | Always-on health widget; visualizer health tab expansion |

---

## Headless Mode

Non-interactive execution for CI, scripts, and automation.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.23.0 | Mar 16 | **Introduced:** redesigned headless mode (auto-responds, detects completion, `--json`, `--timeout`) |
| v2.24.0 | Mar 16 | Headless `new-milestone` command for programmatic milestone creation |
| v2.27.0 | Mar 17 | Headless orchestration skill with supervised mode; auto-restart on crash with exponential backoff |
| v2.28.0 | Mar 17 | `gsd headless query` for instant JSON state inspection (~50ms, no LLM) |
| v2.29.0 | Mar 18 | `--answers` flag for headless answer injection; `--events` flag for JSONL stream filtering |

---

## HTML Reports

Self-contained HTML report generation.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.27.0 | Mar 17 | **Introduced:** HTML report generator with progression index across milestones |
| v2.28.0 | Mar 17 | `/gsd export --html --all` for retrospective milestone reports |
| v2.29.0 | Mar 18 | Enhanced HTML report with derived metrics, visualizations, and interactivity |
| v2.29.0 | Mar 18 | Auto-open HTML reports in default browser on manual export; OSC 8 clickable hyperlinks |

---

## Remote Questions

Routing decisions to humans via messaging platforms.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.3.7 | Mar 11 | **Introduced:** remote user questions via Slack/Discord for headless auto-mode |
| v2.3.7 | Mar 11 | Security hardening (SSRF prevention, input validation, rate limiting) |
| v2.18.0 | Mar 16 | Remote questions onboarding crash fix |
| v2.19.0 | Mar 16 | Discord integration parity with Slack |
| v2.20.0 | Mar 16 | **Telegram remote questions** added alongside Slack/Discord |

---

## Browser Tools

Playwright-based browser automation.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.3.3 | Mar 11 | Browser screenshots constrained to 1568px max |
| v2.8.0 | Mar 13 | **Major expansion:** `browser_analyze_form`, `browser_fill_form`, `browser_find_best`, `browser_act`; 108 tests; decomposed from 5000-line monolith |
| v2.21.0 | Mar 16 | Browser tools converted to TypeScript with c8 coverage; SSRF protection |
| v2.23.0 | Mar 16 | 10 new browser tools (PDF save, state persistence, mocking, device emulation, visual diff, test generation, injection detection) |
| v2.26.0 | Mar 17 | Screenshot constraining uses independent width/height caps |
| v2.29.0 | Mar 18 | Configurable screenshot resolution, format, and quality |

---

## VS Code Extension

Integration with Visual Studio Code.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.23.0 | Mar 16 | **Introduced:** full VS Code extension with chat participant, RPC integration, marketplace publishing under FluxLabs publisher |

---

## Voice

Real-time speech-to-text transcription.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.3.3 | Mar 11 | **Introduced:** `/voice` extension for real-time speech-to-text |
| v2.3.5 | Mar 11 | Transcription no longer lost when pausing and resuming recording |
| v2.10.10 | Mar 14 | Linux voice mode: Groq Whisper API backend |

---

## Native Rust Engine

High-performance native modules replacing JS/WASM.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.10.0 | Mar 13 | **Introduced:** native Rust N-API modules (grep, glob, ps, clipboard, highlight, ast, html, text, fd, image) |
| v2.10.1 | Mar 13 | Pre-compiled JavaScript for Node.js 20/22/24 compatibility |
| v2.10.2 | Mar 13 | TTSR regex engine, diff engine (similar crate), GSD file parser |
| v2.10.4 | Mar 13 | Per-platform native binary distribution packages |
| v2.10.5 | Mar 13 | Streaming JSON parser |
| v2.10.6 | Mar 13 | Output truncation, xxHash32 hasher, bash stream processor |
| v2.16.0 | Mar 15 | Native git operations via libgit2 (~70 fewer process spawns per dispatch) |

---

## LSP Integration

Language Server Protocol support.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.9.0 | Mar 13 | **Introduced:** LSP tool (diagnostics, go-to-definition, references, hover, symbols, rename, code actions) |
| v2.20.0 | Mar 16 | LSP activated by default; call hierarchy, formatting, signature help, synchronized edits |
| v2.27.0 | Mar 17 | LSP command resolution fix for Windows/MSYS |

---

## MCP (Model Context Protocol)

MCP server and client integration.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.3.3 | Mar 11 | MCPorter extension for lazy on-demand MCP server integration |
| v2.22.0 | Mar 16 | MCP server mode (`--mode mcp`); MCP server discovery from `.mcp.json` |
| v2.29.0 | Mar 18 | Per-project MCP config support (`.gsd/mcp.json`) |
| v2.30.0 | Mar 18 | Replaced MCPorter with native MCP client |

---

## Doctor and Diagnostics

Runtime health checks and debugging.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.3.3 | Mar 11 | Post-hook bookkeeping: auto-run doctor + rebuild STATE.md after each unit |
| v2.13.0 | Mar 15 | Worktree-aware doctor with git health diagnostics |
| v2.20.0 | Mar 16 | **`/gsd doctor`:** 7 runtime health checks with auto-fix |
| v2.20.0 | Mar 16 | `gsd --debug` mode (structured JSONL diagnostic logging) |
| v2.22.0 | Mar 16 | `/gsd forensics` post-mortem investigation of auto-mode failures |
| v2.29.0 | Mar 18 | `/gsd logs` command (activity, debug, metrics logs) |
| v2.32.0 | Mar 19 | Environment health checks, progress score, always-on health widget |

---

## Extension System

Extension discovery, management, and authoring.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.4.0 | Mar 12 | Pi extensions from `~/.pi/agent/extensions/` discoverable |
| v2.10.5 | Mar 13 | Async background jobs extension |
| v2.11.0 | Mar 14 | Dynamic extension discovery replaces hardcoded list |
| v2.22.0 | Mar 16 | Claude marketplace import as namespaced GSD components |
| v2.29.0 | Mar 18 | Pre-commit secret scanner; 10 bundled skills |
| v2.30.0 | Mar 18 | **Extension manifest + registry** for user-managed enable/disable |
| v2.30.0 | Mar 18 | Built-in skill authoring system; `create-gsd-extension` skill |
| v2.31.0 | Mar 18 | aws-auth extension for automatic Bedrock credential refresh |

---

## Team Collaboration

Multi-user workflow support.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.10.9 | Mar 14 | **Introduced:** team collaboration with unique milestone names and checked-in `.gsd/` artifacts |
| v2.20.0 | Mar 16 | `/gsd mode` workflow mode system (solo/team presets) |
| v2.26.0 | Mar 17 | `gsd_generate_milestone_id` tool for multi-milestone unique ID generation |

---

## Onboarding and Setup

First-run experience and configuration.

| Version | Date | Evolution |
|---------|------|-----------|
| v0.2.4 | Mar 11 | Branded setup wizard UI |
| v2.3.10 | Mar 12 | Branded postinstall experience with animated spinners |
| v2.3.11 | Mar 12 | Clack-based onboarding wizard (LLM provider + tool keys) |
| v2.4.0 | Mar 12 | Automatic credential migration from Pi installations |
| v2.10.5 | Mar 13 | Simplified two-step auth flow |
| v2.11.0 | Mar 14 | Custom OpenAI-compatible endpoint in onboarding wizard |
| v2.29.0 | Mar 18 | Project onboarding detection and init wizard |

---

## Feature Introduction Timeline

Compact view of when each major feature first appeared:

| Feature | Version | Date |
|---------|---------|------|
| Bundled skills | v0.1.6 | Mar 11 |
| Setup wizard | v0.2.4 | Mar 11 |
| Mac tools | v0.2.8 | Mar 11 |
| Worktree management | v0.3.0 | Mar 11 |
| Step mode | v0.3.3 | Mar 11 |
| Voice | v0.3.3 | Mar 11 |
| MCP (MCPorter) | v0.3.3 | Mar 11 |
| Remote questions (Slack/Discord) | v2.3.7 | Mar 11 |
| Tavily search | v2.3.9 | Mar 12 |
| Native Anthropic web search | v2.5.0 | Mar 12 |
| Secret management | v2.6.0 | Mar 12 |
| Pi SDK vendored | v2.7.0 | Mar 12 |
| Model fallback | v2.7.1 | Mar 13 |
| Browser tools expansion | v2.8.0 | Mar 13 |
| LSP integration | v2.9.0 | Mar 13 |
| Native Rust engine | v2.10.0 | Mar 13 |
| Team collaboration | v2.10.9 | Mar 14 |
| Cross-provider fallback | v2.11.0 | Mar 14 |
| Parallel tool calling | v2.12.0 | Mar 15 |
| Worktree isolation (auto-mode) | v2.13.0 | Mar 15 |
| Branchless architecture | v2.14.0 | Mar 15 |
| Pipeline-aware prompts | v2.15.0 | Mar 15 |
| Steer command | v2.16.0 | Mar 15 |
| Token optimization profiles | v2.17.0 | Mar 15 |
| Knowledge base (KNOWLEDGE.md) | v2.18.0 | Mar 16 |
| Dynamic model discovery | v2.18.0 | Mar 16 |
| Workflow visualizer | v2.19.0 | Mar 16 |
| Capture and triage | v2.19.0 | Mar 16 |
| Dynamic model routing | v2.19.0 | Mar 16 |
| Telegram remote questions | v2.20.0 | Mar 16 |
| Quick tasks | v2.20.0 | Mar 16 |
| Doctor command | v2.20.0 | Mar 16 |
| SQLite context store | v2.20.0 | Mar 16 |
| Debug mode | v2.20.0 | Mar 16 |
| SSRF protection | v2.21.0 | Mar 16 |
| Forensics | v2.22.0 | Mar 16 |
| MCP server mode | v2.22.0 | Mar 16 |
| VS Code extension | v2.23.0 | Mar 16 |
| Headless mode (redesigned) | v2.23.0 | Mar 16 |
| Session picker | v2.23.0 | Mar 16 |
| Parallel orchestration | v2.24.0 | Mar 16 |
| Validate-milestone phase | v2.24.0 | Mar 16 |
| Verification enforcement gate | v2.27.0 | Mar 17 |
| HTML reports | v2.27.0 | Mar 17 |
| Headless query (no LLM) | v2.28.0 | Mar 17 |
| CI/CD three-stage pipeline | v2.29.0 | Mar 18 |
| API key manager | v2.29.0 | Mar 18 |
| Auto-create PR | v2.29.0 | Mar 18 |
| Semantic chunking | v2.29.0 | Mar 18 |
| Extension manifest/registry | v2.30.0 | Mar 18 |
| Skill authoring system | v2.30.0 | Mar 18 |
| Workflow templates | v2.30.0 | Mar 18 |
| AWS auth extension | v2.31.0 | Mar 18 |
| Environment health checks | v2.32.0 | Mar 19 |
| Live regression test harness | v2.33.0 | Mar 19 |
