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
| v2.49.0 | Mar 25 | `--yolo` flag for non-interactive project init; GSD metadata moved to git trailers |
| v2.50.0 | Mar 26 | 8-question quality gates at planning and completion; parallel gate evaluation |
| v2.58.0 | Mar 28 | Concurrent invocation guard on `startAuto()` (#2923) |
| v2.59.0 | Apr 3 | Dynamic routing enabled by default (#3120); codebase map for fresh agent contexts |
| v2.61.0 | Apr 4 | Stop/backtrack capture classifications for milestone regression (#3488); context optimization with model routing and context masking |
| v2.65.0 | Apr 7 | Pre-execution plan verification checks; post-execution cross-task consistency checks |
| v2.66.0 | Apr 8 | Fast path for queued milestone discussion; parallel research slices and parallel milestone validation |
| v2.67.0 | Apr 9 | Session timeout increased to 120s with recoverable pause (#3767); resilient transient error recovery |

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
| v2.41.0 | Mar 21 | Merge anchor verification before worktree teardown (#1829) |
| v2.42.0 | Mar 22 | Pre-merge cleanup: SQUASH_MSG cleanup, guard teardown against uncommitted changes (#1868) |
| v2.59.0 | Apr 3 | Comprehensive pre-merge cleanup adopted from PR #3322; clean stale MERGE_HEAD before squash merge (#2912) |
| v2.62.1 | Apr 5 | Gate steer worktree routing on active session; resolve steer overrides to worktree path |
| v2.66.0 | Apr 8 | Skip current milestone in `syncWorktreeStateBack` to prevent merge conflicts; auto-checkout to main when `isolation:none` finds stale milestone branch |

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
| v2.66.0 | Apr 8 | Recover from stale lockfile after crash or SIGKILL; add `createdAt` timestamp and 30s age guard to staleness check |
| v2.66.1 | Apr 8 | Add escalation and unit-detach guards to finalize timeout handlers; timeout guard around `postUnitPreVerification` |
| v2.67.0 | Apr 9 | Harden auto merge recovery and session safety; resilient transient error recovery -- defer to Core RetryHandler |

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
| v2.54.0 | Mar 27 | Real-time TUI monitor dashboard with self-healing (#2799) |
| v2.56.0 | Mar 27 | `/gsd parallel watch` native TUI overlay for worker monitoring (#2806) |
| v2.59.0 | Apr 3 | Scope commits to milestone boundaries in parallel mode (#3047) |
| v2.64.0 | Apr 6 | **Slice-level parallelism** with dependency-aware dispatch (#3315) |
| v2.66.0 | Apr 8 | Parallel research slices and parallel milestone validation; worker model override for parallel milestone workers |

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
| v2.61.0 | Apr 4 | GSD context optimization with model routing and context masking |
| v2.66.0 | Apr 8 | Trim `promptGuidelines` to 1 line to reduce per-turn token cost |
| v2.67.0 | Apr 9 | **M005: Tiered Context Injection** -- relevance-scoped context with 65%+ reduction |

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
| v2.38.0 | Mar 20 | ADR-004 -- derived-graph reactive task execution (#1546) |
| v2.52.0 | Mar 27 | Replace model-ID pattern matching with capability metadata (#2548) |
| v2.59.0 | Apr 3 | Enable dynamic routing by default (#3120); resolve bare model IDs to anthropic over claude-code provider |
| v2.62.0 | Apr 4 | **Capability-aware model routing:** fire `before_model_select` hook, STEP 2 capability scoring, `taskMetadata` extraction, capability types/data tables/scoring functions |
| v2.64.0 | Apr 6 | Harden flat-rate routing guard against alias/resolution gaps; disable dynamic model routing for flat-rate providers |
| v2.66.0 | Apr 8 | Reclassify planning phases from standard to heavy tier |

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
| v2.65.0 | Apr 7 | Wire blocking behavior and strict mode for enhanced verification; pre-execution plan verification checks; post-execution cross-task consistency checks |
| v2.66.0 | Apr 8 | Validate depth verification answer before unlocking write-gate; add verification gate to `complete-slice` tool; tighten `verifyExpectedArtifact` to prevent rogue-write false positives |

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
| v2.40.0 | Mar 20 | Skill tool resolution for automatic skill activation (#1661) |
| v2.51.0 | Mar 26 | 30 skill packs with 40+ curated skills; `~/.agents/skills/` as primary directory |
| v2.60.0 | Apr 4 | `/btw` skill -- ephemeral side questions from conversation context |
| v2.64.0 | Apr 6 | Add Claude Code official skill directories to skill resolution |

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
| v2.59.0 | Apr 3 | Widget: last commit display and dashboard layout improvements (#3226) |
| v2.66.0 | Apr 8 | `/gsd show-config` command; reactive graph diagnostics |

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
| v2.50.0 | Mar 26 | Disable overall timeout for auto-mode; lock-guard auto-select (#2586) |
| v2.52.0 | Mar 27 | `--bare` mode wired across headless/agent/resource pipeline; RPC protocol v2 with version handshake |
| v2.54.0 | Mar 27 | Headless integration hardening and release (#2811) |
| v2.55.0 | Mar 27 | Colorized verbose output with thinking, phases, cost, and durations (#2886); text mode observability |
| v2.57.0 | Mar 28 | `--resume` flag for session ID prefix matching; text mode shows tool calls |
| v2.59.0 | Apr 3 | Stream full text and thinking output in headless verbose mode (#2934) |
| v2.65.0 | Apr 7 | Treat discuss and plan as multi-turn commands in headless |

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
| v2.62.0 | Apr 4 | Fire configured channels in interactive mode |
| v2.67.0 | Apr 9 | Cancel local TUI when remote answer wins the race; race local TUI against remote channel instead of remote-only routing |

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
| v2.52.0 | Mar 27 | Phase 1: status bar, file decorations, bash terminal, session tree, conversation history, code lens (#2651) |
| v2.53.0 | Mar 27 | Phase 2: activity feed, workflow controls, session forking, enhanced code lens (#2656) |
| v2.59.0 | Apr 3 | Phase 3: sidebar redesign, SCM provider, checkpoints, diagnostics |

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
| v2.45.0 | Mar 25 | `/gsd mcp` command for MCP server status and connectivity (#2362) |
| v2.63.0 | Apr 5 | **MCP-server: 6 read-only tools** for project state queries (#3515) |
| v2.64.0 | Apr 6 | MCP-client: add OAuth auth provider for HTTP transport (#3295) |
| v2.66.0 | Apr 8 | Use `createRequire` to resolve SDK wildcard subpath imports |

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
| v2.39.0 | Mar 20 | 13 new doctor enhancements (#1583) |
| v2.40.0 | Mar 20 | Health check phase 2: real-time doctor issue visibility across widget, visualizer, and HTML reports (#1644) |
| v2.41.0 | Mar 21 | Worktree lifecycle checks, cleanup consolidation, enhanced `/worktree list` (#1814) |
| v2.59.0 | Apr 3 | Stale commit safety check with GSD snapshot and auto-cleanup; harden codebase-map |
| v2.62.0 | Apr 4 | Add codebase validation in `validatePreferences`; harden audit log persistence -- errors-only, sanitized |
| v2.66.1 | Apr 8 | Orphaned milestone branch audit at auto-mode bootstrap |

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

## Web Interface

Browser-based UI for GSD.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.41.0 | Mar 21 | **Introduced:** browser-based web interface (#1717); Next.js-based |
| v2.42.0 | Mar 22 | `--host`, `--port`, `--allowed-origins` flags for web mode (#1873); auth token persisted in sessionStorage (#1877) |
| v2.44.0 | Mar 24 | "Change project root" button in web UI (#2355) |
| v2.45.0 | Mar 25 | Mobile responsive layout (#2354) |
| v2.52.0 | Mar 27 | Dark mode contrast improvements (#2734); auth token gate with synthetic 401, unauthenticated boot state, and recovery screen (#2740) |
| v2.56.0 | Mar 27 | Light theme terminal contrast improvements (#2819) |
| v2.58.0 | Mar 28 | Fall back to project totals when dashboard metrics are zero (#2847); skip shutdown in daemon mode (#2842) |
| v2.64.0 | Apr 6 | Remove 200-column cap on welcome screen width; use `safePackageRootFromImportUrl` for cross-platform package root |
| v2.65.0 | Apr 7 | Persistent notification panel with TUI overlay, widget, and web API |

---

## SQLite State Engine

Tool-driven write-side state transitions backed by SQLite.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.44.0 | Mar 24 | **Introduced:** tool-driven write-side state transitions -- replace markdown mutation with atomic SQLite tool calls (#2141); schema v8 groundwork |
| v2.44.0 | Mar 24 | Schema v9 migration with sequence column on slices/tasks; schema v10 adds `replan_triggered_at` |
| v2.46.0 | Mar 25 | Single-writer state engine v2 -- discipline layer on DB architecture; v3 -- state machine guards, actor identity, reversibility |
| v2.46.0 | Mar 25 | Workflow-logger wired into engine, tool, manifest, and reconcile paths (#2494) |
| v2.52.0 | Mar 27 | Comprehensive SQLite audit fixes -- indexes, caching, safety, reconciliation; schema v11 |
| v2.59.0 | Apr 3 | Migrate unit ownership from JSON to SQLite to eliminate read-modify-write race (#3061) |
| v2.63.0 | Apr 5 | Delete orphaned WAL/SHM files alongside empty `gsd.db` (#2478); wrap saves in transaction to prevent ID races |
| v2.66.0 | Apr 8 | WAL-safe migration backup; critical state machine data integrity fixes (5 waves); 9 resilience fixes + 86 regression tests (#3161) |

---

## Declarative Workflow Engine

YAML-defined workflows through the auto-loop.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.42.0 | Mar 22 | **Introduced:** declarative workflow engine -- YAML-defined workflows through the auto-loop (#2024) |

---

## Daemon and Discord

Background daemon process and Discord bot integration.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.57.0 | Mar 28 | **Introduced:** `packages/daemon` workspace package with DaemonConfig/LogLevel; discord.js v14 DiscordBot class with auth guard and lifecycle |
| v2.57.0 | Mar 28 | Pure-function event formatters (10 functions) mapping RPC events to Discord messages |
| v2.58.0 | Mar 28 | 6 discord.js shard/error/warn event listeners for reconnect handling; launchd fixes |

---

## Capability-Aware Model Routing

Task-capability based model selection (ADR-004).

| Version | Date | Evolution |
|---------|------|-----------|
| v2.38.0 | Mar 20 | ADR-004 -- derived-graph reactive task execution (#1546) |
| v2.52.0 | Mar 27 | **Introduced:** replace model-ID pattern matching with capability metadata (#2548) |
| v2.62.0 | Apr 4 | **Full system:** capability types, data tables, scoring functions; `BeforeModelSelectEvent` extension API; `taskMetadata` extraction and STEP 2 capability scoring; verbose scoring output and capability overrides |
| v2.66.0 | Apr 8 | `subagent_model` config for reactive graph; worker model override for parallel milestone workers |

---

## Quality Gates

Structured evaluation gates at planning and completion.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.50.0 | Mar 26 | **Introduced:** 8-question quality gates to planning and completion templates; parallel gate evaluation with `evaluating-gates` phase |

---

## AI-Powered Triage

Automated issue and PR classification using Claude Haiku.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.36.0 | Mar 20 | **Introduced:** AI-powered issue and PR triage via Claude Haiku (#1510) |

---

## GitHub Sync Extension

Auto-sync project state to GitHub Issues, PRs, and Milestones.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.39.0 | Mar 20 | **Introduced:** GitHub sync extension -- auto-sync to Issues, PRs, Milestones (#1603) |

---

## External State Directory

Moving `.gsd/` out of the repo root into an external directory.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.30.0 | Mar 18 | Move `.gsd/` to external state directory with symlink (ADR-002) (#1242) |
| v2.34.0 | Mar 19 | Keep external GSD state stable in worktrees (#1334) |
| v2.36.0 | Mar 20 | Smarter `.gsd` root discovery -- git-root anchor + walk-up replaces symlink hack (#1386) |

---

## Parallel TUI Monitor

Real-time dashboard for monitoring parallel workers.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.54.0 | Mar 27 | **Introduced:** real-time TUI monitor dashboard with self-healing for parallel workers (#2799) |
| v2.56.0 | Mar 27 | `/gsd parallel watch` -- native TUI overlay for worker monitoring (#2806) |

---

## Curated Skill Ecosystem

Managed skill packs and ecosystem directory.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.43.0 | Mar 23 | Extension resource management gates skills on Skill tool (#2235) |
| v2.51.0 | Mar 26 | **Introduced:** 30 skill packs with 40+ curated skills; `~/.agents/skills/` as primary skills directory |
| v2.51.0 | Mar 26 | SQLite/SQL, Redis, Prisma, Supabase/Postgres database packs; cloud platform packs (Firebase, Azure, AWS) |
| v2.51.0 | Mar 26 | SDKROOT parsing from pbxproj for platform-aware iOS skill matching; migration from `~/.gsd/agent/skills/` |

---

## Offline Mode

Complete offline support for GSD operation.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.45.0 | Mar 25 | **Introduced:** complete offline mode support (#2429) |

---

## Git Trailers

GSD metadata stored in git trailers instead of commit subjects.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.49.0 | Mar 25 | **Introduced:** move GSD metadata from commit subject scopes to git trailers |

---

## Docker Sandbox

Official Docker template for isolated auto-mode execution.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.44.0 | Mar 24 | **Introduced:** official Docker sandbox template for isolated GSD auto mode (#2360) |
| v2.52.0 | Mar 27 | Docker overhaul -- adopt proven container patterns; add missing runtime stage name (#2716, #2765) |

---

## Event Journal and Rule Registry

Unified rule registry and event journal for workflow automation.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.42.0 | Mar 22 | **Introduced:** unified rule registry, event journal, journal query tool, and tool naming convention (#1928) |
| v2.48.0 | Mar 25 | Enhanced `/gsd forensics` with journal and activity log awareness |

---

## Claude Code CLI Provider

External tool execution mode via Claude Code CLI.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.44.0 | Mar 24 | Support for non-API-key provider extensions like Claude Code CLI (#2382) |
| v2.47.0 | Mar 25 | **Introduced:** `externalToolExecution` mode for external providers; Claude Code CLI provider extension |
| v2.64.0 | Apr 6 | Make claude-code provider stateful with full context and sidechain events (#2859) (#3254) |
| v2.67.0 | Apr 9 | Use native Windows claude lookup; guard claude-code fallback to anthropic provider only; route Anthropic subscription users through Claude Code CLI (#3772) |

---

## Managed RTK Integration

Opt-in RTK (Real-Time Kit) support with preference and web UI toggle.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.51.0 | Mar 26 | **Introduced:** managed RTK integration with opt-in preference and web UI toggle (#2620) |

---

## Safety Mechanisms

Snapshots and pre-merge checks for auto-mode safety.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.53.0 | Mar 27 | **Introduced:** enable safety mechanisms by default -- snapshots and pre-merge checks (#2678) |

---

## Search

Web search integration and budget management.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.3.9 | Mar 12 | Tavily search integration |
| v2.5.0 | Mar 12 | Native Anthropic web search |
| v2.37.0 | Mar 20 | Session-level search budget to prevent unbounded native web search (#1529) |
| v2.50.0 | Mar 26 | Enforce hard search budget and survive context compaction |
| v2.67.0 | Apr 9 | Prompts: harden non-bypassable gates and exclude dot-folders from scanning |

---

## CI/CD

Continuous integration and delivery pipeline.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.29.0 | Mar 18 | Three-stage promotion pipeline (Dev/Test/Prod) (#1098); automated prod-release (#1194) |
| v2.38.0 | Mar 20 | Reduce GitHub Actions minutes ~60-70% (~10k to ~3-4k/month) (#1552) |
| v2.41.0 | Mar 21 | Skip build/test for docs-only PRs; prompt injection scan (#1699) |
| v2.42.0 | Mar 22 | PR risk checker -- classify changed files by system and surface risk level (#1930) |

---

## LLM Safety Harness

Damage control mechanisms for auto-mode LLM execution.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.64.0 | Apr 6 | **Introduced:** LLM safety harness for auto-mode damage control |
| v2.66.0 | Apr 8 | Adversarial review waves 1-3 hardening; write safety -- atomic writes and randomized tmp paths; session and recovery robustness |
| v2.67.0 | Apr 9 | Fail closed for discussion gate enforcement; harden non-bypassable gates |

---

## Ollama / Local LLM Support

First-class support for local LLM inference via Ollama.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.59.0 | Apr 3 | **Introduced:** Ollama extension for first-class local LLM support (#3371) |
| v2.64.0 | Apr 6 | Native `/api/chat` provider with full option exposure; register `models.json` providers and await Ollama probe in headless mode; use apiKey auth mode to avoid `streamSimple` crash |

---

## MCP Server (Read-Only Tools)

GSD as an MCP server exposing project state queries.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.22.0 | Mar 16 | MCP server mode (`--mode mcp`) |
| v2.63.0 | Apr 5 | **Introduced:** 6 read-only tools for project state queries (#3515) |
| v2.64.0 | Apr 6 | OAuth auth provider for MCP HTTP transport (#3295) |

---

## Notification System

Persistent notifications across TUI, widget, and web surfaces.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.65.0 | Apr 7 | **Introduced:** persistent notification panel with TUI overlay, widget, and web API |
| v2.65.0 | Apr 7 | Wrap long messages and fit overlay to content; backdrop dimming and viewport padding; prevent foreign lock deletion |

---

## Context Engineering

Optimized context injection and scope management.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.59.0 | Apr 3 | Codebase map -- structural orientation for fresh agent contexts |
| v2.61.0 | Apr 4 | GSD context optimization with model routing and context masking |
| v2.64.0 | Apr 6 | Inject `S##-CONTEXT.md` from slice discussion into all prompt builders |
| v2.67.0 | Apr 9 | **R005:** decision scope cascade -- derive scope from slice metadata |
| v2.67.0 | Apr 9 | **M005: Tiered Context Injection** -- relevance-scoped context with 65%+ reduction |

---

## Write Gate and Verification

Depth verification, write-gate enforcement, and artifact verification.

| Version | Date | Evolution |
|---------|------|-----------|
| v2.29.0 | Mar 18 | Per-milestone depth verification + queue-flow write-gate |
| v2.65.0 | Apr 7 | Wire blocking behavior and strict mode for enhanced verification; `enhanced_verification` preferences added to `mergePreferences` |
| v2.66.0 | Apr 8 | Validate depth verification answer before unlocking write-gate; add verification gate to `complete-slice` tool |
| v2.67.0 | Apr 9 | Mechanical enforcement for discussion question gates |

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
| External state directory (ADR-002) | v2.30.0 | Mar 18 |
| AWS auth extension | v2.31.0 | Mar 18 |
| Environment health checks | v2.32.0 | Mar 19 |
| Live regression test harness | v2.33.0 | Mar 19 |
| OpenRouter model auto-generation | v2.34.0 | Mar 19 |
| AI-powered triage (Claude Haiku) | v2.36.0 | Mar 20 |
| Session-level search budget | v2.37.0 | Mar 20 |
| ADR-004 reactive task execution | v2.38.0 | Mar 20 |
| CI Actions ~60-70% reduction | v2.38.0 | Mar 20 |
| Doctor 13 enhancements | v2.39.0 | Mar 20 |
| GitHub sync extension | v2.39.0 | Mar 20 |
| Skill tool resolution | v2.40.0 | Mar 20 |
| Doctor health phase 2 | v2.40.0 | Mar 20 |
| Web interface | v2.41.0 | Mar 21 |
| Doctor worktree lifecycle | v2.41.0 | Mar 21 |
| Declarative workflow engine | v2.42.0 | Mar 22 |
| Event journal and rule registry | v2.42.0 | Mar 22 |
| PR risk checker | v2.42.0 | Mar 22 |
| SQLite state engine | v2.44.0 | Mar 24 |
| Docker sandbox | v2.44.0 | Mar 24 |
| Offline mode | v2.45.0 | Mar 25 |
| Mobile responsive web UI | v2.45.0 | Mar 25 |
| `/gsd mcp` command | v2.45.0 | Mar 25 |
| Single-writer engine v2/v3 | v2.46.0 | Mar 25 |
| Claude Code CLI provider | v2.47.0 | Mar 25 |
| Git trailers | v2.49.0 | Mar 25 |
| Quality gates (8-question) | v2.50.0 | Mar 26 |
| Hard search budget enforcement | v2.50.0 | Mar 26 |
| Curated skill ecosystem (30 packs) | v2.51.0 | Mar 26 |
| Managed RTK integration | v2.51.0 | Mar 26 |
| VS Code extension phase 1 | v2.52.0 | Mar 27 |
| Capability metadata model routing | v2.52.0 | Mar 27 |
| Dark mode (web) | v2.52.0 | Mar 27 |
| RPC protocol v2 | v2.52.0 | Mar 27 |
| Safety mechanisms (default) | v2.53.0 | Mar 27 |
| VS Code extension phase 2 | v2.53.0 | Mar 27 |
| Parallel TUI monitor | v2.54.0 | Mar 27 |
| Colorized headless output | v2.55.0 | Mar 27 |
| `/gsd parallel watch` | v2.56.0 | Mar 27 |
| Daemon and Discord bot | v2.57.0 | Mar 28 |
| Concurrent invocation guard | v2.58.0 | Mar 28 |
| Ollama local LLM extension | v2.59.0 | Apr 3 |
| Codebase map | v2.59.0 | Apr 3 |
| Dynamic routing default | v2.59.0 | Apr 3 |
| VS Code extension phase 3 | v2.59.0 | Apr 3 |
| Stale commit doctor check | v2.59.0 | Apr 3 |
| `/btw` skill | v2.60.0 | Apr 4 |
| Stop/backtrack capture | v2.61.0 | Apr 4 |
| Context optimization | v2.61.0 | Apr 4 |
| Capability-aware model routing (full) | v2.62.0 | Apr 4 |
| `/gsd codebase` enhancements | v2.62.0 | Apr 4 |
| MCP server read-only tools | v2.63.0 | Apr 5 |
| LLM safety harness | v2.64.0 | Apr 6 |
| Native Ollama provider | v2.64.0 | Apr 6 |
| Slice-level parallelism | v2.64.0 | Apr 6 |
| MCP OAuth auth provider | v2.64.0 | Apr 6 |
| Notification system | v2.65.0 | Apr 7 |
| Enhanced verification (strict mode) | v2.65.0 | Apr 7 |
| Pre/post-execution checks | v2.65.0 | Apr 7 |
| Parallel research slices | v2.66.0 | Apr 8 |
| `/gsd show-config` | v2.66.0 | Apr 8 |
| Graph diagnostics | v2.66.0 | Apr 8 |
| Adversarial review hardening (5 waves) | v2.66.0 | Apr 8 |
| R005 decision scope cascade | v2.67.0 | Apr 9 |
| M005 tiered context injection | v2.67.0 | Apr 9 |
