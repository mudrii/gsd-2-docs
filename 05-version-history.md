# GSD-2 Version History

Complete version history of all released versions, grouped by major feature milestones.

---

## Era 1: Foundation (v0.1.6 - v0.3.3) — March 11, 2026

The pre-2.0 series established GSD as an extension-based coding agent built on the Pi SDK.

### v0.1.6 — 2026-03-11

**Added:**
- GitHub extension tool suite with confirmation gate
- Bundled skills: frontend-design, swiftui, debug-like-expert
- Skills trigger table in system prompt
- Resource loader syncs bundled skills to `~/.gsd/agent/skills/`

**Fixed:**
- `~/.gsd/agent/` paths in prompt templates instead of `~/.pi/agent/`
- Guard against re-injecting discuss prompt when session already in flight

**Changed:**
- License updated to MIT

---

### v0.2.4 — 2026-03-11

**Added:**
- Branded setup wizard UI with visual hierarchy, descriptions, and status feedback
- Branded banner on first launch
- Postinstall banner with version and next-step hint

**Fixed:**
- All `.pi/` paths updated to `.gsd/`
- Default model matching, circular self-dependency removed
- Selected options stay lit when notes field is focused

---

### v0.2.5 — 2026-03-11

**Fixed:**
- Circular self-dependency removed, default model set to anthropic/claude-sonnet-4-6 with thinking off

---

### v0.2.6 — 2026-03-11

**Fixed:**
- Default model validated against full registry on every startup

---

### v0.2.8 — 2026-03-11

**Added:**
- Mac-tools extension (macOS native automation)

---

### v0.2.9 — 2026-03-11

**Fixed:**
- Idle recovery skips stuck units instead of silently stalling
- `pkg/package.json` version synced with pi-coding-agent to prevent false update banner
- Milestones with summary but no roadmap treated as complete

---

### v0.3.0 — 2026-03-11

**Added:**
- `/worktree` (`/wt`) -- git worktree lifecycle management
- `/gsd migrate` -- `.planning` to `.gsd` migration tool

**Fixed:**
- Skipped API keys persist so wizard doesn't repeat on every launch
- Scoped models restored from settings on new session startup
- Startup fallback no longer overwrites user's default model with Sonnet

---

### v0.3.1 — 2026-03-11

**Fixed:**
- Windows VT input restored after child processes exit
- Print/JSON mode in cli.js so subagents don't hang
- Bash/bg_shell hang and kill issues on Windows
- Windows backspace in masked input + custom browser path support

**Changed:**
- Renamed "Get Stuff Done" to "Get Shit Done"

---

### v0.3.3 — 2026-03-11 -- KEY RELEASE

Last pre-2.0 release. Introduced step mode and voice.

**Added:**
- `/gsd next` step mode -- walk through units one at a time with a wizard between each
- `/gsd` bare command defaults to step mode
- `/exit` command, `/clear` as alias for `/new`
- MCPorter extension for lazy on-demand MCP server integration
- `/voice` extension for real-time speech-to-text
- Post-hook bookkeeping: auto-run doctor + rebuild STATE.md after each unit

**Fixed:**
- Idle watchdog false-firing on active agents (tasks >10min no longer get skipped)
- Browser screenshots constrained to 1568px max dimension

**Removed:**
- `/gsd-run` command (replaced by `/gsd` and `/gsd next`)

---

## Era 2: Stabilization and Branding (v2.3.4 - v2.5.1) — March 11-12, 2026

Version jump from 0.x to 2.x. Focus on install reliability, onboarding, remote questions, and git safety.

### v2.3.4 — 2026-03-11

**Added:**
- CHANGELOG.md with curated history from v0.1.6 onwards
- Project-local `/publish-version` command for npm releases
- GitHub Sponsors funding configuration
- npm publish and install smoke test

---

### v2.3.5 — 2026-03-11

**Fixed:**
- Voice extension: transcription no longer lost when pausing and resuming recording

---

### v2.3.6 — 2026-03-11

**Fixed:**
- Postinstall no longer triggers hidden `sudo` prompt on Linux
- Auto-commit dirty files before branch switch to prevent lost work

---

### v2.3.7 — 2026-03-11 -- KEY RELEASE

Introduced remote questions for headless auto-mode.

**Added:**
- Remote user questions via Slack/Discord for headless auto-mode sessions

**Fixed:**
- Auto-mode model switches no longer persist as the user's global default
- Auto-mode resume now rebuilds disk state and runs doctor before dispatching
- Remote questions security hardening (SSRF prevention, input validation, rate limiting)

---

### v2.3.8 — 2026-03-11

**Fixed:**
- Worktree file operations resolve paths against active working directory
- Auto-mode merge guard handles all slice completion paths

---

### v2.3.9 — 2026-03-12

**Added:**
- Tavily as alternative web search provider alongside Brave Search
- Auto-mode progress widget shows all stats; footer hidden during auto-mode

**Fixed:**
- Auto-mode infinite loop and closeout instability (idempotent unit dispatch, retry caps)
- Migration no longer requires ROADMAP.md
- Worktree branch safety
- Multiple community-reported bugs

---

### v2.3.10 — 2026-03-12

**Added:**
- Branded postinstall experience with animated spinners and progress indicators

**Fixed:**
- Ctrl+Alt shortcuts show slash-command fallback in terminals that lack Kitty keyboard protocol

---

### v2.3.11 — 2026-03-12

**Added:**
- Branded clack-based onboarding wizard on first launch (LLM provider selection, tool API keys, setup summary)
- `gsd config` subcommand to re-run the setup wizard

**Fixed:**
- Parallel subagent results no longer truncated at 200 characters

---

### v2.4.0 — 2026-03-12

**Added:**
- Automatic migration of provider credentials from existing Pi installations
- Pi extensions from `~/.pi/agent/extensions/` discoverable in interactive mode
- GitService core implementation for programmatic git operations

**Changed:**
- System prompt compressed by 48% (360 to 187 lines)

---

### v2.5.0 — 2026-03-12

**Added:**
- Native Anthropic web search (no Brave API key required for Claude models)
- GitService fully wired into codebase
- Merge guards, snapshot support, auto-push after slice squash-merge
- Rich commit messages with structured metadata

**Fixed:**
- State machine deadlock when units fail to produce expected artifacts
- Duplicate Brave search tools, conversation replay error, Windows test globs

---

### v2.5.1 — 2026-03-12

**Added:**
- `secure_env_collect` auto-detects existing keys and provides guidance

**Changed:**
- Right-sized pipeline: single-slice milestones skip redundant sessions (9-10 reduced to 5-6)

**Fixed:**
- Squash-merge aborts cleanly on conflict instead of looping

---

## Era 3: Secret Management and Monorepo (v2.6.0 - v2.7.1) — March 12-13, 2026

Proactive secret handling, Pi SDK vendored in-tree, model fallback support.

### v2.6.0 — 2026-03-12

**Added:**
- Proactive secret management -- planning phase forecasts required API keys into a manifest
- `--continue` / `-c` CLI flag to resume the most recent session

**Fixed:**
- Doctor post-hook, `main_branch` preference, recovery OOM prevention, `.gsd/` merge conflicts

---

### v2.7.0 — 2026-03-12 -- KEY RELEASE

Pi SDK vendored into monorepo. GSD now owns its full stack.

**Changed:**
- Vendor Pi SDK source (tui, ai, agent-core, coding-agent) into workspace monorepo under `packages/`
- Replaced compiled npm dependency and patch-package workflow with direct TypeScript source

---

### v2.7.1 — 2026-03-13

**Added:**
- Model fallback support for auto-mode phases
- `/kill` command for immediate process termination

**Fixed:**
- `npm install -g gsd-pi` now works (workspace packages bundled via `bundleDependencies`)

---

## Era 4: Browser Tools and Auto-Mode Hardening (v2.8.0 - v2.9.0) — March 13, 2026

Major browser automation expansion, LSP integration, cross-platform fixes.

### v2.8.0 — 2026-03-13

**Added:**
- Browser tools: `browser_analyze_form`, `browser_fill_form`, `browser_find_best`, `browser_act`
- 108 unit and integration tests for browser tools

**Changed:**
- Browser tools decomposed from 5000-line monolith into focused modules

---

### v2.8.1 — 2026-03-13

**Added:**
- Discussion depth verification and context write-gate
- TTSR + blob/artifact storage
- Configurable `merge_strategy` preference

**Fixed:**
- `fsevents` Node 25 compatibility, UAT artifact verification, prior slice ordering

---

### v2.8.2 — 2026-03-13

**Fixed:**
- Cross-platform path operations using `node:path` instead of hardcoded forward slashes
- HTTP_PROXY/HTTPS_PROXY environment variables respected
- Windows NUL redirects sanitized

---

### v2.8.3 — 2026-03-13

**Fixed:**
- Provider-aware model resolution for per-phase preferences
- Execute-task artifact verification with self-repair
- Research phase infinite loop broken

---

### v2.9.0 — 2026-03-13 -- KEY RELEASE

Introduced LSP integration and interactive preferences wizard.

**Added:**
- LSP tool -- full Language Server Protocol integration (diagnostics, go-to-definition, references, hover, symbols, rename, code actions)
- `/thinking` slash command for toggling thinking level
- Interactive wizard mode for `/gsd prefs`
- Startup update check with 24-hour cache

**Fixed:**
- Milestone ID collision prevention, auto-mode pauses on provider errors, command injection in LSP

---

## Era 5: Native Rust Engine (v2.10.0 - v2.10.12) — March 13-14, 2026

Native Rust modules replaced JS/WASM dependencies. Major performance milestone.

### v2.10.0 — 2026-03-13 -- KEY RELEASE

Introduced the native Rust engine.

**Added:**
- Native Rust engine with N-API modules: grep (ripgrep), glob (gitignore-aware), ps (process tree), clipboard (arboard), highlight (syntect), ast (ast-grep, 38+ languages), html, text, fd, image
- Background shell `env`, `run`, and `session` process types
- Hashline edits (line-hash-anchored file editing)
- Universal config discovery extension

**Changed:**
- Find, syntax highlighting, autocomplete, text utilities, clipboard, image processing all switched to native Rust

---

### v2.10.1 — 2026-03-13

**Fixed:**
- `@gsd/native` package ships pre-compiled JavaScript (fixing startup crashes on Node.js 20, 22, 24)

---

### v2.10.2 — 2026-03-13

**Added:**
- Native Rust TTSR regex engine (one-pass DFA matching)
- Native Rust diff engine (fuzzy text matching via `similar` crate)
- Native Rust GSD file parser (frontmatter, section extraction, batch directory parsing)

---

### v2.10.4 — 2026-03-13

**Fixed:**
- Native binary distribution -- `.node` binaries were missing from npm tarball since v2.10.0

**Added:**
- Per-platform optional dependency packages (`@gsd-build/engine-*`)
- Cross-platform native binary CI build and publish workflow

---

### v2.10.5 — 2026-03-13

**Added:**
- Async background jobs extension for non-blocking task execution
- Multi-credential round-robin with rate-limit fallback
- Bash interceptor to block commands that duplicate dedicated tools
- `gsd update` subcommand, task isolation, native Rust streaming JSON parser

**Changed:**
- Simplified onboarding into two-step auth flow

---

### v2.10.6 — 2026-03-13

**Added:**
- Native Rust output truncation, xxHash32 hasher, bash stream processor
- Memory extraction pipeline
- `claude-opus-4-6` model with 1M context window

**Fixed:**
- Oversized TUI lines truncated instead of crashing, Anthropic rate limit backoff

---

### v2.10.7 — 2026-03-14

**Added:**
- GitHub Workflows skill with CI workflow template and `ci_monitor` tool
- Auto-resolve merge conflicts via LLM-powered fix-merge session
- Auto-update integration branch when user starts auto-mode from a different branch

**Fixed:**
- Unresolvable artifact paths treated as stale (preventing OOM crashes)
- Eliminated branch checkout during slice merge that caused STATE.md conflicts

---

### v2.10.8 — 2026-03-14

**Fixed:**
- Publish verification checks `dist/loader.js` is non-empty

---

### v2.10.9 — 2026-03-14

**Added:**
- Team collaboration: multiple users can work on same repo without milestone name clashes

**Changed:**
- Execute-task loop detection uses adaptive reconciliation
- Memory storage switched from better-sqlite3 to sql.js (WASM) for Node 25+ compatibility

**Fixed:**
- Node 22.22+ compatibility, infinite loop on complete-slice merge, credential handling

---

### v2.10.10 — 2026-03-14

**Added:**
- Alibaba Cloud coding-plan provider support
- Linux voice mode: Groq Whisper API backend
- Opus 4.6 1M as default model, Discord onboarding

**Fixed:**
- Broken `npm install` / `npx gsd-pi@latest` caused by unpublished workspace packages
- Pre-publish tarball install validation added to CI

---

### v2.10.11 — 2026-03-14

**Fixed:**
- Hoist workspace package dependencies into root `dependencies` for end users
- Added `undici` as root dependency to resolve startup crash
- Check `GROQ_API_KEY` before entering voice mode

---

### v2.10.12 — 2026-03-14

**Added:**
- Multi-milestone readiness flow with per-milestone discussion gate

**Fixed:**
- `npx gsd-pi@latest` failing with `ERR_MODULE_NOT_FOUND` (runtime symlink creation)

---

## Era 6: Provider Resilience and Parallel Tools (v2.11.0 - v2.12.0) — March 14-15, 2026

Cross-provider fallback, parallel tool calling, dispatch loop fixes.

### v2.11.0 — 2026-03-14

**Added:**
- Cross-provider fallback when rate or quota limits are hit
- Custom OpenAI-compatible endpoint option in onboarding wizard
- Model provider selection in preferences
- Auto-mode fallback model rotation on network errors
- Native libgit2-backed git read operations for dispatch hotpath

**Changed:**
- Dynamic extension discovery, memoized `deriveState()`, session-scoped caching

**Fixed:**
- OpenRouter model ID resolution, git-svn noise suppression, auto-mode hang prevention

---

### v2.11.1 — 2026-03-15 -- URGENT

**Fixed:**
- Auto-mode loops on research-slice and plan-slice -- stale in-process directory listing cache caused `dispatchNextUnit` to re-dispatch the same unit

---

### v2.12.0 — 2026-03-15

**Added:**
- Parallel tool calling -- tools from a single assistant message execute concurrently by default
- Ollama Cloud as model and web tool provider
- Extensible hook system for auto-mode state machine
- Event queue settlement for parallel tool execution

**Changed:**
- Inline static templates into prompt builders (eliminating ~44 READ tool calls per milestone)

---

## Era 7: Worktree Isolation and Architecture Overhaul (v2.13.0 - v2.14.4) — March 15, 2026

Worktree isolation for auto-mode, branchless architecture, dispatch loop elimination.

### v2.13.0 — 2026-03-15 -- KEY RELEASE

Introduced worktree isolation for auto-mode.

**Added:**
- Worktree isolation for auto-mode -- isolated git worktrees per milestone
- Self-healing git repair (detached HEAD, stale locks, orphaned worktrees)
- Worktree-aware doctor with git health diagnostics
- Isolation preferences (worktree vs branch modes)

**Fixed:**
- Three root causes of dispatch loops identified and fixed (parse cache, completion persistence, recovery counter)
- Non-execute-task units now verify artifacts on disk

---

### v2.13.1 — 2026-03-15

**Fixed:**
- Windows: multi-line commit messages, single-quoted git arguments, worktree path normalization

---

### v2.14.0 — 2026-03-15 -- KEY RELEASE

Branchless worktree architecture. ~2600 lines of merge/conflict code removed.

**Added:**
- Discussion manifest for multi-milestone context discussions
- Session-internal `/gsd config`, model selection UI, startup performance improvements

**Changed:**
- Branchless worktree architecture -- eliminated slice branches entirely; all work commits sequentially on `milestone/<MID>`
- `.gitignore` overhaul -- planning artifacts tracked in git naturally

**Removed:**
- `ensureSliceBranch()`, `switchToMain()`, `mergeSliceToMain()`, `mergeSliceToMilestone()`
- `git.isolation` and `git.merge_to_main` preferences (deprecated)

---

### v2.14.1 — 2026-03-15

**Fixed:**
- Quiet auto-mode warnings -- internal recovery machinery downgraded to verbose-only
- Dispatch recovery hardening (artifact fallback, TUI freeze prevention, atomic writes)

---

### v2.14.2 — 2026-03-15

**Fixed:**
- Dispatch reentrancy deadlock -- `_dispatching` flag was never reset after first dispatch
- `.gitignore` self-heal for existing projects
- Discuss depth verification rendering

---

### v2.14.3 — 2026-03-15

**Fixed:**
- Copy planning artifacts into new auto-worktrees (prevents plan-slice loops)

---

### v2.14.4 — 2026-03-15

**Fixed:**
- Session cwd update -- `newSession()` now updates the LLM's perceived working directory for auto-worktrees (root cause of complete-slice and plan-slice loops)

---

## Era 8: Commands, Budgets, and Pipeline-Aware Prompts (v2.15.0 - v2.16.0) — March 15, 2026

New commands, preferences validation, native git operations, steer command.

### v2.15.0 — 2026-03-15

**Added:**
- 8 new commands for budget enforcement, notifications, and quality-of-life
- Preferences schema validation (detects unknown/typo'd keys)
- Pipeline-aware prompts (each agent phase knows its role)
- Research depth calibration (deep/targeted/light tiers)

**Changed:**
- Auto-mode decomposed into focused modules, dispatch logic extracted to dispatch table

**Fixed:**
- Executor agents receive explicit working directory, merge loop resolution, arrow key escape sequences, YAML parser hardening, TUI resource leaks

---

### v2.15.1 — 2026-03-15

**Fixed:**
- Auto-mode worktree path resolution (artifacts written to wrong location)
- Auto-mode resource sync detection, missing import crash
- Recovery hardening (checkbox verification, corrupt roadmaps, atomic writes)
- Progress widget refreshes from disk every 5 seconds

**Added:**
- 46 new unit tests for auto-dashboard, auto-recovery, crash-recovery

---

### v2.16.0 — 2026-03-15

**Added:**
- `/gsd steer` command -- hard-steer plan documents during execution
- Native git operations via libgit2 (~70 fewer process spawns per dispatch cycle)
- Native performance optimizations for `deriveState`, JSONL parsing, path resolution
- Default model upgraded to Opus 4.6 with 1M context

---

## Era 9: Token Optimization and Smart Routing (v2.17.0 - v2.19.0) — March 15-16, 2026

Token optimization profiles, dynamic model routing, workflow visualizer, capture system.

### v2.17.0 — 2026-03-15 -- KEY RELEASE

Introduced token optimization system.

**Added:**
- Token optimization profiles: `budget`, `balanced`, `quality` (40-60% savings on budget mode)
- Complexity-based task routing (simple/standard/heavy classification with persistent learning)
- `git.commit_docs` preference for keeping `.gsd/` artifacts local-only

**Fixed:**
- Native binary hangs, auto-mode cross-terminal stop, parse cache collision, Google provider cleanup

---

### v2.18.0 — 2026-03-16

**Added:**
- Milestone queue reorder (`/gsd queue`) with dependency-aware validation
- `.gsd/KNOWLEDGE.md` -- persistent project-specific context file with `/gsd knowledge` command
- Dynamic model discovery (runtime model enumeration from provider APIs with TTL caching)
- Expanded preferences wizard (all configurable fields exposed)
- Comprehensive documentation (12 new docs)
- `resolveProjectRoot()` for worktree-safe path resolution
- 1,813 lines of new tests across 13 test files

**Fixed:**
- Heap OOM during long auto-mode sessions (four sources of unbounded memory growth)
- Stale worktree cwd after milestone completion (three-layer fix)
- Worktree integration branch resolution, milestone merge in branch mode
- Tool-aware idle detection, remote questions onboarding crash

---

### v2.19.0 — 2026-03-16 -- KEY RELEASE

Workflow visualizer and dynamic model routing.

**Added:**
- Workflow visualizer (`/gsd visualize`) -- full-screen TUI overlay with Progress, Dependencies, Metrics, and Timeline tabs
- Mid-execution capture and triage (`/gsd capture`) -- fire-and-forget thoughts during auto-mode with automatic classification
- Dynamic model routing -- complexity-based routing with budget-pressure awareness, cross-provider cost comparison, adaptive learning from routing history

**Fixed:**
- Absolute paths in auto-mode prompts (eliminated LLM path confusion in worktree contexts)
- Worktree lifecycle on mid-session milestone transitions

---

## Era 10: Team Collaboration and Quality-of-Life (v2.20.0 - v2.23.0) — March 16, 2026

Telegram integration, quick tasks, VS Code extension, headless orchestration.

### v2.20.0 — 2026-03-16

**Added:**
- Telegram remote questions (alongside Slack and Discord)
- `/gsd quick` -- quick task execution with GSD guarantees, skip planning overhead
- `/gsd mode` -- workflow mode system (solo/team presets)
- `/gsd help` -- categorized command reference
- `/gsd doctor` -- 7 runtime health checks with auto-fix
- Agent instructions injection via `agent-instructions.md`
- Skill lifecycle management (telemetry, health dashboard, heal-skill)
- SQLite context store for surgical prompt injection
- Context-window budget engine (proportional prompt sizing)
- LSP activated by default
- `gsd --debug` mode
- Worktree post-create hook

**Fixed:**
- CPU spinning from regex backtracking, model config bleed between instances
- Onboarding wizard repeats, milestone detection infinite loop
- Google Search OAuth fallback

---

### v2.21.0 — 2026-03-16

**Added:**
- Browser tools TypeScript conversion with c8 test coverage
- SSRF protection on `fetch_page`
- Stale async job cancellation

**Changed:**
- Pause/resume recovery reuses crash recovery infrastructure
- Build scripts extracted, help text deduplicated

---

### v2.22.0 — 2026-03-16

**Added:**
- `/gsd forensics` -- post-mortem investigation of auto-mode failures
- Claude marketplace import (namespaced GSD components)
- MCP server mode (`--mode mcp`)
- `/review`, `/test`, `/lint` skills
- GitHub API client with diff-aware context injection
- File watcher (chokidar-based)
- `git.isolation: "none"` option
- E2E smoke tests, subcommand help

**Fixed:**
- Background shell worktree cwd detection, auto-worktree validation
- Fractional slice IDs support, worktree state sync

---

### v2.23.0 — 2026-03-16 -- KEY RELEASE

VS Code extension and redesigned headless mode.

**Added:**
- VS Code extension with chat participant, RPC integration, marketplace publishing
- `gsd headless` -- redesigned headless mode (auto-responds, detects completion, `--json` output, `--timeout`)
- `gsd sessions` -- interactive session picker for browsing and resuming saved sessions
- 10 new browser tools (PDF save, state persistence, mocking, device emulation, visual diff, test generation, injection detection)
- Structured discussion rounds with `ask_user_questions`
- `validate-milestone` prompt
- `models.json` resolution for custom model definitions

**Fixed:**
- Forensics worktree-aware, background command pipe-open hang
- Auto mode infinite skip loop, roadmap range syntax expansion
- Anti-pattern rule prevents `bash &` usage

---

## Era 11: Parallel Orchestration and Production Pipeline (v2.24.0 - v2.26.0) — March 16-17, 2026

Parallel milestone execution, verification enforcement, production CI/CD.

### v2.24.0 — 2026-03-16 -- KEY RELEASE

Parallel milestone orchestration.

**Added:**
- Parallel milestone orchestration -- run multiple workers across phases simultaneously
- Dashboard view for parallel workers with 80% budget alert
- Headless `new-milestone` command
- Interactive update prompt on startup
- `validate-milestone` phase and dispatch

**Fixed:**
- Sync `completed-units.json` across worktree boundaries
- Auto-resume after rate limit cooldown
- Prevent stale state loop on auto-mode restart

**Changed:**
- Lazy-load LLM provider SDKs to reduce startup time

---

### v2.25.0 — 2026-03-16

**Added:**
- Native web search results rendering in TUI with `PREFER_BRAVE_SEARCH` toggle
- Meaningful commit messages generated from task summaries
- Incremental memory system for auto-mode sessions
- Visualizer enriched with stats and discussion status
- 14 new E2E smoke tests

**Fixed:**
- Phantom skip loop from stale crash recovery context
- Cache invalidation consistency

---

### v2.26.0 — 2026-03-17

**Added:**
- Model selector grouped by provider
- `require_slice_discussion` option to pause auto-mode before each slice
- Discussion status indicators in `/gsd discuss`
- Worker NDJSON monitoring and budget enforcement for parallel orchestration
- `gsd_generate_milestone_id` tool
- Alt+V clipboard image paste, hashline edit mode
- Fallback parser for prose-style roadmaps

**Fixed:**
- Windows path normalization, async bash job completion
- Native web_search limited to max 5 uses per response
- Completed milestone re-entry prevention, replan-slice loop breaking
- BMP clipboard images on WSL2

**Removed:**
- Symlink-based development workflow

---

## Era 12: Crash Recovery and HTML Reports (v2.27.0 - v2.28.0) — March 17, 2026

Crash recovery for parallel orchestrator, HTML report generation, verification gates.

### v2.27.0 — 2026-03-17 -- KEY RELEASE

Crash recovery and HTML reports.

**Added:**
- HTML report generator with progression index across milestones
- Crash recovery for parallel orchestrator (persisted state with PID liveness detection)
- Headless orchestration skill with supervised mode
- Verification enforcement gate for milestone completion
- `git.manage_gitignore` preference

**Changed:**
- Encapsulated auto.ts state into AutoSession class
- Extracted 7 focused modules from auto.ts

**Fixed:**
- Web search loop, transient network error retry, parallel worker PID tracking
- `/gsd discuss` recommends next undiscussed slice
- Context-pressure monitor wired at 70%, auto-restart headless mode on crash

---

### v2.28.0 — 2026-03-17

**Added:**
- `gsd headless query` -- instant JSON state inspection (no LLM, ~50ms)
- `/gsd update` slash command for in-session self-update
- `/gsd export --html --all` for retrospective milestone reports

**Fixed:**
- Failure recovery and resume safeguards (atomic writes, OAuth timeouts, RPC exit detection, bash cleanup, LSP retry)
- Consolidated duplicate modules (mcp-server.ts, bundled-extension-paths.ts)

---

## Era 13: Massive Feature and Refactoring Wave (v2.29.0) — March 18, 2026

Largest single release. CI pipeline, token optimization suite, skills, onboarding, and extensive refactoring.

### v2.29.0 — 2026-03-18 -- KEY RELEASE

Enormous release spanning CI pipeline, token optimization, bundled skills, and deep codebase refactoring.

**Added:**
- `searchExcludeDirs` setting for `@` file autocomplete blacklist
- Automated prod-release pipeline (version bump, changelog, tag push)
- Auto-open HTML reports in default browser on manual export
- Upgrade to Node.js 24 LTS
- `/gsd logs` command (activity, debug, metrics logs)
- Configurable screenshot resolution, format, quality for browser tools
- Pre-commit secret scanner and CI secret detection
- Per-project MCP config support (`.gsd/mcp.json`)
- API request counter for copilot/subscription users
- Per-milestone depth verification + queue-flow write-gate
- OSC 8 clickable hyperlinks for file paths
- Park/discard actions for in-progress milestones
- Three-stage promotion pipeline (Dev, Test, Prod)
- Cache-ordered prompt assembly and dashboard cache hit rate
- Comprehensive API key manager (`/gsd keys`)
- Multi-stage Dockerfile for CI
- Directory safeguards for system/home paths
- Enhanced HTML report with derived metrics, visualizations, interactivity
- Auto-extract lessons to KNOWLEDGE.md on slice/milestone completion
- Auto-create PR on milestone completion
- Semantic chunking with preferences and metrics
- Token optimization suite (prompt caching, compression, smart context selection)
- Autocomplete for `/thinking`, GSD subcommand descriptions
- `respectGitignoreInPicker` setting
- `search_provider` preference
- `--events` flag for JSONL stream filtering
- 10 bundled skills for UI, quality, and code optimization
- Group model list by provider in prefs wizard
- `--answers` flag for headless answer injection
- Project onboarding detection and init wizard

**Fixed:**
- 70+ bug fixes across CI, security, dispatch, worktree sync, verification, model resolution, and more

**Changed:**
- Major codebase decomposition: auto.ts split into 6 modules, commands.ts into 5, bg-shell god file split, headless split, doctor decomposed
- Extensive deduplication and consolidation across the entire codebase
- Performance optimizations (SSE streaming, startup I/O, makeTreeWritable calls)

---

## Era 14: Extensions, Skills, and Workflow Templates (v2.30.0 - v2.31.2) — March 18, 2026

Extension system, skill authoring, workflow templates, worktree CLI flag.

### v2.30.0 — 2026-03-18

**Added:**
- Extension manifest + registry for user-managed enable/disable
- Model health indicator in auto-mode progress widget
- Simplified auto pipeline (merged research into planning, ADR-003)
- `create-gsd-extension` skill
- Built-in skill authoring system (ADR-003)
- Two-step provider-then-model picker in preferences wizard
- Workflow templates (right-sized workflows for every task type)

**Fixed:**
- Slice progression gated on UAT verdict, cache invalidation before roadmap check
- Windows LSP `.cmd` executables, headless timeout, non-persistent bg process cleanup

**Changed:**
- `.gsd/` moved to external state directory with symlink (ADR-002)
- Replaced MCPorter with native MCP client

---

### v2.31.0 — 2026-03-18

**Added:**
- aws-auth extension for automatic Bedrock credential refresh
- `-w`/`--worktree` CLI flag for isolated worktree sessions

**Fixed:**
- Stale git-commit assertions, targeted runtime file removal, cache invalidation in discuss loop
- Robust prose slice header parsing (H1-H4, bold, dots, no-separator)
- Stranded `.gsd.lock/` directory cleanup

**Changed:**
- Removed dead `commit_docs` preference

---

### v2.31.1 — 2026-03-18

**Fixed:**
- Prevent false-positive 'Session lock lost' during auto-mode

---

### v2.31.2 — 2026-03-18

**Fixed:**
- Stop replaying completed run-uat units

---

## Era 15: Health Monitoring and Hardening (v2.32.0 - v2.33.1) — March 19, 2026

Environment health checks, live regression testing, dispatch hardening.

### v2.32.0 — 2026-03-19

**Added:**
- Always-on health widget and visualizer health tab expansion
- Environment health checks, progress score, and status integration

**Fixed:**
- Skip crash recovery when auto.lock was written by current process
- Load worktree-cli extension modules via jiti
- Prevent concurrent dispatch during skip chains
- Skip non-artifact UAT dispatch in auto-mode

**Changed:**
- Deduplication and extraction of helpers across dispatch, prompt building, and git operations

---

### v2.33.0 — 2026-03-19

**Added:**
- Live regression test harness for post-build pipeline validation

**Fixed:**
- Retry lock path aligned with primary lock settings (preventing ECOMPROMISED)
- Skip symlinks in makeTreeWritable (NixOS/nix-darwin support)
- Windows EPERM on `.gsd` migration rename (copy+delete fallback)
- Actionable recovery guidance in crash info messages
- Resolve main repo root in worktrees for stable identity hash
- Merge quick-task branch back to original after completion

**Changed:**
- Major consolidation: `tryMergeMilestone` eliminates 4 duplicate merge paths, `parseUnitId()` centralizes ID parsing, `getErrorMessage()` replaces 65 inline duplicates

---

### v2.33.1 — 2026-03-19

**Fixed:**
- Clean up stale numbered lock files and harden signal/exit handling
- Worktree sync and home-directory safety check

**Changed:**
- Remove orphaned mcporter extension manifest

---

## Summary Statistics

| Metric | Count |
|--------|-------|
| Total released versions | 62 |
| v0.x releases | 8 |
| v2.3.x releases | 8 |
| v2.4.x - v2.9.x releases | 10 |
| v2.10.x releases | 13 |
| v2.11.x - v2.33.x releases | 23 |
| Development period | March 11 - March 19, 2026 (9 days) |
| Key milestone releases | 12 |
