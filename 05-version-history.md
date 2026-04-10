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

## Era 16: Hardening & Deduplication (v2.34.0 - v2.35.0) — March 19, 2026

Crash lock hardening, OpenRouter auto-generation, massive code deduplication (~25 consolidation PRs), changelog command, JS fallbacks for native addon.

### v2.34.0 — 2026-03-19

**Added:**
- Auto-generate OpenRouter model registry from API + add missing models (#1407) (#1426)

**Fixed:**
- Release stranded bootstrap locks and handle re-entrant reacquire (#1352)
- Add JS fallbacks for `wrapTextWithAnsi` and `visibleWidth` when native addon unavailable (#1418) (#1428)
- Emit `agent_end` after abort during tool execution (#1414) (#1417)
- Auto-discard bootstrap crash locks and clean `auto.lock` on exit (#1397)
- Harden quick-task branch lifecycle -- disk recovery + integration branch guard (#1342)
- Skip verification retry on spawn infra errors (ETIMEDOUT, ENOENT) (#1340)
- Keep external GSD state stable in worktrees (#1334)
- Stop excluding all `.gsd/` from commits -- only exclude runtime files (#1326) (#1328)
- Handle ECOMPROMISED in uncaughtException guard and align retry onCompromised (#1322) (#1332)

---

### v2.35.0 — 2026-03-19

**Added:**
- `/gsd changelog` command with LLM-summarized release notes (#1465)

**Fixed:**
- Restore LSP single-server selector export
- Preserve `args` for `mcp_call` tool invocations (#1354)
- Accumulate session cost independently of message array (#1423)
- Resolve CI failures -- scope provider check, fix Windows path, correct severity
- Close 5 doctor coverage gaps -- providers, lock dir, integration branch, orphaned worktrees
- Add PID self-check to guided-flow crash lock detection (#1398)
- Close merge, validation, serialization, and docs gaps in preferences

**Changed:**
- Deduplicate error emission and message patterns in agent-core (#1444)
- Simplify settings manager with generic setter helpers (#1461)
- Consolidate theme files and remove manual schema (#1478)
- Extract overlay layout and compositing from TUI into separate module (#1482)
- Extract slash command handlers from interactive-mode (#1485)
- Remove dead code (unused exports) (#1486)
- Extract retry handler and compaction orchestrator from agent-session
- Deduplicate rendering patterns in markdown and keys
- Consolidate shared code between OpenAI providers
- Deduplicate RPC mode shared patterns
- Extract shared tree rendering utilities
- Consolidate OAuth callback server and helper utilities
- Extract shared file lock utilities
- Consolidate resource loader with generic update/dedupe methods
- Consolidate model switching logic in agent-session
- Deduplicate `toPosixPath`, `ZERO_USAGE`, and `shortenPath` utilities
- Consolidate 9 emit methods in extension runner into shared `invokeHandlers`
- Consolidate duplicate patterns in LSP module

---

## Era 17: External State & Automation (v2.36.0 - v2.39.0) — March 20, 2026

`.gsd/` moved external with symlink, AI triage (Claude Haiku), AGENTS.md, GitHub sync extension, `/gsd doctor` 13 new checks, `/gsd rethink`, `/gsd changelog`, auto-PR, browser/runtime UAT types, `GSD_HOME`/`GSD_PROJECT_ID` env vars, prompt compression removal (4100 lines), sliding-window stuck detection.

### v2.36.0 — 2026-03-20

**Added:**
- Deprecate `agent-instructions.md` in favor of `AGENTS.md` / `CLAUDE.md` (#1492) (#1514)
- AI-powered issue and PR triage via Claude Haiku (#1510)

**Fixed:**
- Preserve user messages during abort with origin-aware queue clearing (#1439) (#1521)
- Remove broken SwiftUI skill and add CI reference check (#1476) (#1520)
- Wire `escalateTier` into auto-loop retry path (#1505) (#1519)
- Prevent bare `/gsd` from stealing session lock from running auto-mode (#1507) (#1517)
- Wire dead token-profile defaults and add `/gsd rate` command (#1505) (#1516)
- Prevent false-positive session lock loss during sleep/event loop stalls (#1512) (#1513)
- Filter non-milestone directories from `findMilestoneIds` (#1494) (#1508)
- Accept `passed` as terminal validation verdict (#1429) (#1509)
- Prevent `ensureGitignore` from adding `.gsd` when tracked in git (#1364) (#1367)
- Check project root `.env` when secrets gate runs in worktree (#1387) (#1470)
- Realign cwd before dispatch + clean stale merge state on failure (#1389) (#1400)
- Create `milestones/` directory in worktree when missing (#1374)
- Verify symlink after migration + fix test failures (#1377) (#1404)
- Validate CWD instead of project root when running from a GSD worktree (#1317) (#1504)
- Smarter `.gsd` root discovery -- git-root anchor + walk-up replaces symlink hack (#1386)

---

### v2.37.0 — 2026-03-20

**Added:**
- Two-column dashboard layout with redesigned widget (#1530)
- Integrate cmux with GSD runtime (#1532)

**Fixed:**
- Add session-level search budget to prevent unbounded native web search (#1309) (#1529)

---

### v2.37.1 — 2026-03-20

**Fixed:**
- Interactive guard menu for remote auto-mode sessions (#1507) (#1524)
- Use `pull_request_target` so AI triage has secret access on PRs
- cmux library directory incorrectly loaded as extension (#1537)
- Separate pi-tui-dependent layout utils to fix report generation (#1527)
- Clarify session lock loss diagnostics (#1535)
- Auto-mode worktree commits land on main instead of milestone branch (#1526) (#1534)

---

### v2.38.0 — 2026-03-20

**Added:**
- ADR-004 -- derived-graph reactive task execution (#1546)
- Anthropic-vertex provider for Claude on Vertex AI (#1533)

**Fixed:**
- Reduce GitHub Actions minutes ~60-70% (~10k to ~3-4k/month) (#1552)
- Reactive batch verification + dependency-based carry-forward (#1549)
- Enforce backtick file paths in task plan IO sections (#1548)

---

### v2.39.0 — 2026-03-20

**Added:**
- Activate matching skills in dispatched prompts (#1630)
- `.gsd/RUNTIME.md` template for declared runtime context (#1626)
- Create draft PR on milestone completion when `git.auto_pr` enabled (#1627)
- Browser-executable and runtime-executable UAT types (#1620)
- Apply model preferences in guided flow for milestone planning (#1614)
- GitHub sync extension -- auto-sync to Issues, PRs, Milestones (#1603)
- `GSD_PROJECT_ID` env var to override project hash (#1600)
- `GSD_HOME` env var to override global `~/.gsd` directory (#1566)
- 13 enhancements to `/gsd doctor` (#1583)
- Minimal GSD welcome screen on startup (#1584)

**Fixed:**
- Recover + prevent #1364 `.gsd/` data-loss (v2.30.0-v2.38.0) (#1635)
- Treat summary as terminal artifact even when roadmap slices are unchecked (#1632)
- Close residual #1364 data-loss vectors on v2.36.0+ (#1637)
- Auto-resolve npm subpath exports in extension loader (#1624)
- Create `node_modules` symlink for dynamic import resolution in extensions (#1623)
- Filter cross-milestone errors from health tracker escalation (#1621)
- Move unit closeout to run immediately after completion (#1612)
- Use pathspec exclusions in `smartStage` to prevent hanging on large repos (#1613)
- Add auto-fix for premature slice completion deadlock in doctor (#1611)
- Resolve `${VAR}` env references in MCP client `.mcp.json` configs (#1609)
- Return "dispatched" after doctor heal to prevent session race (#1580) (#1610)
- Update Anthropic OAuth endpoints to `platform.claude.com` (#1608)
- Lazy-open GSD database on first tool call in manual sessions (#1606)

**Changed:**
- Unify sidecar mini-loop into main dispatch path (#1617)
- Add 30K char hard cap on prompt preamble (#1619)
- Replace stuck counter with sliding-window detection (#1618)
- Remove prompt compression subsystem (~4,100 lines) (#1597)
- Crashproof `stopAuto` with independent try/catch per cleanup step (#1596)
- Replace session-scoped promise bridge with per-unit one-shot (#1595)

---

## Era 18: Web UI, Skill Tool & Health (v2.40.0 - v2.43.0) — March 20-23, 2026

Browser-based web interface, Skill tool resolution, health check phase 2, declarative workflow engine (YAML), rule registry + event journal, PR risk checker, ADR attribution, `/gsd fast`, `--host`/`--port`/`--allowed-origins`, startup optimizations, forensics duplicate detection.

### v2.40.0 — 2026-03-20

**Added:**
- Skill tool resolution (#1661)
- Health check phase 2 -- real-time doctor issue visibility across widget, visualizer, and HTML reports (#1644)
- Upgrade forensics prompt to full-access GSD debugger (#1660)

**Fixed:**
- Prune stale `env-utils.js` from extensions root, preventing startup load error (#1655)
- Replace box corners with full-width bars for visual unity with auto-mode widget (#1654)
- Add runtime paths to forensics prompt to prevent path hallucination (#1657)
- Guard TUI render during session transitions to prevent freeze (#1658)
- Default UAT type to artifact-driven to prevent unnecessary auto-mode pauses (#1651)
- Cancel trailing async jobs on session switch to prevent wasted LLM turns (#1643)

**Changed:**
- Decompose `autoLoop` into pipeline phases (#1615) (#1659)

---

### v2.41.0 — 2026-03-21 -- KEY RELEASE

Introduced browser-based web interface.

**Added:**
- Browser-based web interface (#1717)
- Worktree lifecycle checks, cleanup consolidation, enhanced `/worktree list` (#1814)
- Skip build/test for docs-only PRs and add prompt injection scan (#1699)
- Custom Models guide and updated documentation (#1670)
- Surface doctor issue details in progress score widget and health views (#1667)
- `~/.gsd/projects/` orphan detection and pruning (#1686)

**Fixed:**
- Fall through to prose slice parser when checkbox parser yields empty under `## Slices` (#1744)
- Verify merge anchored before worktree teardown (#1829)
- Reject `execute-task` with zero tool calls as hallucinated (#1838)
- Read `depends_on` from `CONTEXT-DRAFT.md` when `CONTEXT.md` is absent (#1743)
- Detect completion marker in prose slice headers (#1816)
- Reverse-sync root-level `.gsd` files on worktree teardown (#1831)
- Prevent TUI freeze when using `@` file finder (#1832)
- Prevent silent data loss when milestone merge fails due to dirty working tree (#1752)
- Avoid DEP0190 by passing command to shell explicitly (#1827)
- Treat zero-slice roadmap as pre-planning instead of blocked (#1826)
- Process depth verification in queue mode (#1823)
- Register SIGHUP/SIGINT handlers to clean lock files on crash (#1821)
- Auto-dispatch discussion instead of hard-stopping on `needs-discussion` phase (#1820)
- Fix roadmap checkbox and UAT stub immediately instead of deferring (#1819)
- Resolve pending `unitPromise` in `stopAuto` to prevent hang (#1818)
- Handle unborn branch in `nativeBranchExists` to prevent dispatch deadlock (#1815)
- Prevent cleanup from deleting user work files (#1825)
- Tool-call loop guard + TUI stack overflow prevention (#1801)
- 40+ additional bug fixes across worktree, dispatch, parallel, roadmap, session lock, and metrics

**Changed:**
- Split `shared/mod.ts` into pure and TUI-dependent barrels (#1807)
- Replace hardcoded `/tmp` paths with `os.tmpdir()`/`homedir()` (#1708)
- Reduce pipeline minutes with shallow clones, npm caching, and exponential backoff (#1700)
- Split `auto-loop.ts` monolith into `auto/` directory modules (#1682)

---

### v2.42.0 — 2026-03-22

**Added:**
- Declarative workflow engine -- YAML-defined workflows through the auto-loop (#2024)
- Unified rule registry, event journal, journal query tool, and tool naming convention (#1928)
- PR risk checker -- classify changed files by system and surface risk level (#1930)
- ADR attribution -- distinguish human vs agent vs collaborative decisions (#1830)
- `/gsd fast` command and gate service tier icon to supported models (#1848) (#1862)
- `--host`, `--port`, `--allowed-origins` flags for web mode (#1847) (#1873)

**Fixed:**
- Recursive key sorting in tool-call loop guard hash function (#1962)
- Prevent SIGTSTP crash on Windows (#2018)
- Repo-identity: use native realpath on Windows to resolve 8.3 short paths (#1960)
- Doctor: gate roadmap checkbox on summary existing on disk, not issue detection (#1915)
- Warn when milestone merge contains only metadata and no code (#1906) (#1927)
- Worktree: resolve 8.3 short paths and use shell mode for `.bat` hooks on Windows (#1956)
- Web: persist auth token in `sessionStorage` to survive page refreshes (#1877)
- Clean up `SQUASH_MSG` after squash-merge and guard worktree teardown against uncommitted changes (#1868)
- Populate `RecoveryContext` in hook unit supervision to prevent crash on stalled tool recovery (#1867)
- Resolve worktree path from git registry when `.gsd/` symlink is shadowed (#1866)
- Resolve Node v24 web boot failure -- `ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING` (#1864)
- Prevent cross-project state leak in brand-new directories (#1639) (#1861)
- Reconcile worktree HEAD with milestone branch ref before squash merge (#1846) (#1859)
- Normalize Windows backslash paths in bash command strings (#1436) (#1863)

**Changed:**
- Startup optimizations -- pre-compiled extensions, compile cache, batch discovery (#2125)

---

### v2.43.0 — 2026-03-23

**Added:**
- Forensics opt-in duplicate detection before issue creation (#2105)

**Fixed:**
- Prevent banner from printing twice on first run (#2251)
- Suppress duplicate follow-up for awaited job results (#2248) (#2250)
- Remove force-staging of `.gsd/milestones/` through symlinks (#2247) (#2249)
- Remove over-broad skill activation heuristic (#2239) (#2244)
- Auth: fall through to env/fallback when OAuth credential has no registered provider (#2097)
- LSP: bound message buffer and clean up stale client state (#2171)
- Clean up macOS numbered `.gsd` collision variants (#2205) (#2210)
- Keep duplicate-search loop guard armed (#2117)
- Clean up extension error listener on session dispose (#2165)
- Apply fast service tier outside auto-mode (#2126)
- Clean up leaked SIGINT and extension selector listeners (#2172)
- Standardize GitHub Actions and Node.js versions (#2169)
- Resolve memory leaks in glob, ttsr, and image overflow (#2170)
- Extension resource management -- prune stale dirs, fix `isBuiltIn`, gate skills on Skill tool (#2235)
- Kill stale server process before launch to prevent EADDRINUSE (#1934) (#2034)
- Force `LC_ALL=C` in `GIT_NO_PROMPT_ENV` to support non-English locales (#2035)
- Force gh CLI for issue creation to prevent misrouting (#2067) (#2094)
- Display active inference model during execution (#1982)
- Correct Copilot context window and output token limits (#2118)

**Changed:**
- Startup optimizations -- pre-compiled extensions, compile cache, batch discovery (#2125)

---

## Era 19: SQLite State Engine (v2.44.0 - v2.46.1) — March 24-25, 2026

Tool-driven write-side state transitions (SQLite replaces markdown), single-writer engine v2/v3, schema v8-v11 migrations, `show_token_cost`, Docker sandbox, non-api-key provider support, mobile web, `/gsd rethink`, `/gsd mcp`, offline mode, KNOWLEDGE.md injection, timestamps.

### v2.44.0 — 2026-03-24 -- KEY RELEASE

Introduced tool-driven SQLite state engine.

**Added:**
- Tool-driven write-side state transitions -- replace markdown mutation with atomic SQLite tool calls (#2141)
- Support for non-api-key provider extensions like Claude Code CLI (#2382)
- Official Docker sandbox template for isolated GSD auto mode (#2360)
- Show per-prompt token cost in footer behind `show_token_cost` preference (#2357)
- "Change project root" button in web UI (#2355)
- Schema v8-v11 migrations: planning columns, sequence on slices/tasks, `replan_triggered_at`

**Fixed:**
- Post-migration cleanup -- pragmas, rollbacks, tool gaps, stale code (#2410)
- Prevent planning data loss from destructive upsert and post-unit re-import (#2370)
- Use correct notify severity type ("warning" not "warn")
- Resolve compiled `.js` modules for all subprocess calls under `node_modules` (#2320)
- Add missing SQLite WAL sidecars and journal to runtime exclusion lists (#2299)
- Fix memory and resource leaks across TUI, LSP, DB, and automation (#2314)
- Preserve freeform `DECISIONS.md` content on decision save (#2319)
- Restore alibaba-coding-plan provider via `models.custom.ts` (#2350)
- Skip false `env_dependencies` error in auto-worktrees (#2318)
- Auto-stash dirty files before squash merge and surface dirty filenames in error (#2298)
- Detect TypeScript syntax in `.js` extension files and suggest renaming to `.ts` (#2386)

**Changed:**
- Add CODEOWNERS and team workflow docs (#2286)

---

### v2.45.0 — 2026-03-25

**Added:**
- Mobile responsive web UI (#2354)
- `/gsd rethink` command for conversational project reorganization (#2459)
- `renderCall`/`renderResult` previews on DB tools (#2273)
- Timestamps on user and assistant messages (#2368)
- `/gsd mcp` command for MCP server status and connectivity (#2362)
- Complete offline mode support (#2429)
- Inject global `~/.gsd/agent/KNOWLEDGE.md` into system prompt (#2331)

**Fixed:**
- Handle `retentionDays=0` on Windows + run windows-portability on PRs (#2460)
- `isInheritedRepo` conflates `~/.gsd` with project `.gsd` when git root is `$HOME` (#2398)
- Reconcile disk milestones missing from DB in `deriveStateFromDb` (#2416) (#2422)
- Reset `recoveryAttempts` on unit re-dispatch (#2322) (#2424)
- Detect and preserve submodule state during worktree teardown (#2337) (#2425)
- Handle survivor branch recovery in `phase=complete` (#2358) (#2427)
- Preserve rich task plans on DB roundtrip (#2450) (#2453)
- Merge worktree back to main when `stopAuto` is called after milestone completion (#2317) (#2430)
- Skip doctor directory checks for pending slices (#2446)
- Migrate completion/validation prompts to DB-backed tools (#2449)
- Prevent `saveArtifactToDb` from overwriting larger files with truncated content (#2442) (#2447)
- Stop auto loop on real code merge conflicts (#2330) (#2428)
- Classify terminated/connection errors as transient in provider error handler (#2309) (#2432)
- `auto_pr: true` now actually creates PRs -- fix 3 interacting bugs (#2302) (#2433)
- Gate auto-mode bootstrap on SQLite availability (#2419) (#2421)
- Block `/gsd quick` when auto-mode is active (#2420)

**Changed:**
- Migrate test files from custom harness to `node:test` (multiple batches)

---

### v2.46.0 — 2026-03-25

**Added:**
- Single-writer engine v3 -- state machine guards, actor identity, reversibility
- Single-writer state engine v2 -- discipline layer on DB architecture
- Workflow-logger wired into engine, tool, manifest, reconcile paths (#2494)

**Fixed:**
- Align prompts with single-writer tool API
- Integration-proof -- check DB state not roadmap projection after reset
- Block milestone completion when verification fails (#2500)
- Add `typecheck:extensions` to pretest to prevent silent type drift
- Update test assertions for schema v11, prompt changes, and removed `completedUnits`
- Harden single-writer engine -- close TOCTOU, intercept bypasses, status inconsistencies
- Close bare-relative-path bypass in `STATE.md` regex
- Fix misleading portaudio error on PEP 668 Linux systems (#2403) (#2407)
- Address PR review feedback for non-apikey provider support (#2452)
- Change default isolation mode from worktree to none (#2481)
- Add startup checks for Node version and git availability (#2463)
- Add worktree lifecycle events to journal (#2486)

---

### v2.46.1 — 2026-03-25

**Fixed:**
- Prevent windows-portability from blocking pipeline
- Prevent pipeline race condition on release push
- Create empty DB for fresh projects with empty `.gsd/` (#2510)
- Hydrate remote channel tokens from `auth.json` on startup

---

## Era 20: VS Code Extension, Headless & Daemon (v2.47.0 - v2.58.0) — March 25-28, 2026

Claude Code CLI provider, VS Code extension (3 phases), RPC v2, parallel TUI monitor, headless hardening (`--bare`, colorized), daemon/Discord, quality gates, `--yolo`, git trailers, ADR-004 capability routing, 30 skill packs, `/terminal`, managed RTK, safety mechanisms default.

### v2.47.0 — 2026-03-25

**Added:**
- Claude Code CLI provider extension (#2452)
- External tool execution mode for external providers

**Fixed:**
- Render tool calls above text response for Claude Code CLI
- Update `FILE-SYSTEM-MAP.md` path after docs-internal move
- `isInheritedRepo` false negative when parent has stale `.gsd`; defense-in-depth local `.git` check in bootstrap
- Resolve SDK executable path and update model IDs
- Migrate remaining 4 prompts to use DB-backed tool API instead of direct write
- Make workflow event hash platform-deterministic
- Reconcile stale task DB status from disk artifacts (#2514)

---

### v2.48.0 — 2026-03-25

**Added:**
- `/gsd discuss` can target queued milestones
- Enhanced `/gsd forensics` with journal and activity log awareness

**Fixed:**
- Make journal scanning intelligent -- limit parsed files, line-count older ones
- Scope custom provider stream handlers to prevent clobbering built-in API handlers
- Filter benign bash exit-code-1 and user skips from error traces in forensics
- Clear stale milestone ID reservations at session start
- Render tool calls above text response for external providers
- Skip `CONTEXT-DRAFT` warning for completed/parked milestones

**Changed:**
- Extract `RAPID_ITERATION_THRESHOLD_MS`, simplify data access

**Removed:**
- Remove `insertChildBefore` usage in chat-controller

---

### v2.49.0 — 2026-03-25

**Added:**
- `--yolo` flag to `/gsd auto` for non-interactive project init

**Fixed:**
- Use full git log in merge tests to match trailer-based milestone IDs
- Verdict gate accepts `PARTIAL` for mixed/human-experience/live-runtime UATs

**Changed:**
- Move GSD metadata from commit subject scopes to git trailers

---

### v2.50.0 — 2026-03-26

**Added:**
- Parallel quality gate evaluation with `evaluating-gates` phase
- 8-question quality gates in planning and completion templates
- Wire structured error propagation through `UnitResult`

**Fixed:**
- Reconcile stale task status in filesystem-based state derivation (#2514)
- Handle `session_switch` event so `/resume` restores GSD state (#2587)
- Use GitHub Issue Types via GraphQL instead of classification labels
- Disable overall timeout for auto-mode in headless, fix lock-guard auto-select (#2586)
- Align UAT artifact suffix with `gsd_slice_complete` output (#2592)
- Stop treating 5xx server errors as credential-level failures
- Retry lock file reads before declaring compromise
- Prevent `ensureGsdSymlink` from creating subdirectory `.gsd` when git-root `.gsd` exists
- Add EAGAIN to `INFRA_ERROR_CODES` to stop budget-burning retries
- Enforce hard search budget and survive context compaction
- Add `SAFE_SKILL_NAME` guard to reject prompt-injection via crafted skill names
- Downgrade isolation mode when worktree creation fails
- Resolve race conditions in blob-store, discovery-cache, and agent-loop
- Resolve WebSocket listener leaks and bound session cache
- Resolve double-set race, missing error ID, and stream handler in RPC

**Changed:**
- Adopt `parseUnitId` utility across all auto-* modules
- Deduplicate artifact path functions into single module
- Consolidate branch name patterns into single module
- Split doctor-checks into focused modules
- Remove dead worktree code and unused methods

---

### v2.51.0 — 2026-03-26 -- KEY RELEASE

Introduced 30 skill packs, `/terminal` command, and managed RTK integration.

**Added:**
- `/terminal` slash command for direct shell execution (#2349)
- Check verification class compliance before milestone completion (#2623)
- Managed RTK integration with opt-in preference and web UI toggle (#2620)
- Inject verification classes into milestone validation prompt (#2621)
- 19 wshobson/agents packs with 40 curated skills
- 11 new skill packs covering major frameworks and languages
- SQLite/SQL detection, SQL optimization pack, and Redis pack
- Prisma and Supabase/Postgres database packs
- Cloud platform packs (Firebase, Azure, AWS)
- Curate catalog -- add top ecosystem skills, drop low-quality bundled ones
- Parse SDKROOT from pbxproj for platform-aware iOS skill matching
- Use `~/.agents/skills/` as primary skills directory with curated catalog
- Detect initialized projects in health widget

**Fixed:**
- Honor explicit model config when model is not in known tier map (#2643)
- Exclude `lastReasoning` from retry diagnostic to prevent hallucination loops (#2663)
- Persist `rewrite-docs` attempt counter to disk for session restart survival (#2671)
- Prefer `terminal-notifier` over `osascript` on macOS (#2633)
- Classify stream-truncation JSON parse errors as transient (#2636)
- Call `ensureDbOpen()` before slice queries in `/gsd discuss` (#2640)
- Use `--body-file` for forensics issue creation (#2641)
- `isLockProcessAlive` should return true for own PID (#2642)
- Check ASSESSMENT file for UAT verdict in `checkNeedsRunUat` (#2646)
- Use `pauseAuto` instead of `stopAuto` for warning-level dispatch stops (#2666)
- Prevent double `mergeAndExit` on milestone completion (#2648)
- Respect `queue-order.json` in DB-backed state derivation (#2649)
- VS Code: support Remote SSH by adding `extensionKind` and error handler (#2650)
- Update DB task status in `writeBlockerPlaceholder` for execute-task (#2657)

**Changed:**
- Consolidate docs, remove stale artifacts, and repo hygiene (#2665)
- Extract `runSafely` helper for try-catch-debug-continue pattern (#2611)

---

### v2.52.0 — 2026-03-27 -- KEY RELEASE

Introduced VS Code extension phase 1 and RPC v2.

**Added:**
- VS Code extension: status bar, file decorations, bash terminal, session tree, conversation history, code lens [1/2] (#2651)
- Dark mode contrast -- raise token floor and flatten opacity tier system (#2734)
- `--bare` mode wired across headless, pi-coding-agent, resource-loader
- `runId` generation on prompt/steer/follow_up commands with event routing
- RPC protocol v2 types, init handshake with version detection

**Fixed:**
- Auto-mode stops after provider errors (#2762) (#2764)
- Add missing runtime stage name to Dockerfile (#2765)
- Make `transaction()` re-entrant and add `slice_dependencies` to `initSchema`
- Remove `preferences.md` from `ROOT_STATE_FILES` to prevent back-sync overwrite
- Wire tool handlers through DB port layer, remove `_getAdapter` from all tools
- Move state machine guards inside transaction in 5 tool handlers (#2752)
- Reconcile disk milestones into empty DB before `deriveStateFromDb` guard (#2686)
- Seed `preferences.md` into auto-mode worktrees (#2693)
- Discover marketplace plugins nested inside container directories (#2718)
- Exempt interactive tools from idle watchdog stall detection (#2676)
- Guard `allSlicesDone` against vacuous truth on empty slice array (#2679)
- Block `complete-milestone` dispatch when VALIDATION is `needs-remediation` (#2682)
- Auth token gate -- synthetic 401 on missing token, unauthenticated boot state, and recovery screen (#2740)
- Docker: overhaul fragile setup, adopt proven container patterns (#2716)
- Windows: prevent EINVAL by disabling detached process groups on Win32 (#2744)
- Delete orphaned `verification_evidence` rows on `complete-task` rollback (#2746)

**Changed:**
- Replace model-ID pattern matching with capability metadata (#2548)
- Comprehensive SQLite audit fixes -- indexes, caching, safety, reconciliation
- Rename `preferences.md` to `PREFERENCES.md` for consistency (#2700) (#2738)
- Unify three overlapping error classifiers into single classify-decide-act pipeline

---

### v2.53.0 — 2026-03-27

**Added:**
- VS Code extension phase 2: activity feed, workflow controls, session forking, enhanced code lens [2/3] (#2656)
- Enable safety mechanisms by default (snapshots, pre-merge checks) (#2678)

**Fixed:**
- Hydrate collected secrets for current session (#2788)
- Resolve stash pop conflicts and stop swallowing merge errors (#2780)
- Treat any extracted verdict as terminal in `isValidationTerminal` (#2774)
- Use `localStorage` for auth token to enable multi-tab usage (#2785)
- Guard `activeMilestone.id` access in discuss and headless paths (#2776)
- Clean up zombie parallel workers stuck in error state (#2782)
- Relax milestone validation gate to accept prose evidence (#2779)
- Write milestone reports to project root instead of worktree (#2778)
- Auto-resolve build artifact conflicts in milestone merge (#2777)
- Let rate-limit errors attempt model fallback before pausing (#2775)
- Prevent `gsd next` from self-killing via stale crash lock (#2784)
- Add shell flag for Windows spawn in VS Code extension (#2781)

**Changed:**
- Extract duplicated status guards and validation helpers (#2767)

---

### v2.54.0 — 2026-03-27

**Added:**
- Headless integration hardening and release (M002) (#2811)
- Parallel: real-time TUI monitor dashboard with self-healing (#2799)

---

### v2.55.0 — 2026-03-27

**Added:**
- Colorized headless verbose output with thinking, phases, cost, and durations (#2886)
- Headless text mode observability + skip UAT pause (#2867)

**Fixed:**
- Let `gsd update` bypass version mismatch gate (#2845)
- Add `isWorkspaceEvent` guard + close `routeLiveInteractionEvent` exhaustiveness gap (#2878)
- Use project root for prior-slice dispatch guard (#2863)
- Include queue context in milestone planning prompts (#2846)
- Detect monorepo roots in project discovery to prevent workspace fragmentation (#2849)
- Recover from deleted cwd in bg-shell timers (#2850)
- Enable dynamic routing without `models` section (#2851)
- Fully remove providers from `/providers` (#2852)

---

### v2.56.0 — 2026-03-27

**Added:**
- `/gsd parallel watch` -- native TUI overlay for worker monitoring (#2806)

**Fixed:**
- Copy `web/components` to dist-test for xterm-theme test (#2891)
- Prefer `PREFERENCES.MD` in worktrees (#2796)
- Resume auto-mode after transient provider pause (#2822)
- Resolve session lock contention and 3 related parallel-mode bugs (#2184) (#2800)
- Improve light theme terminal contrast (#2819)
- Preserve auto start model through discuss (#2837)

**Changed:**
- Compile unit tests with esbuild, reclassify integration tests, fix `node_modules` symlink (#2809)

---

### v2.57.0 — 2026-03-28

**Added:**
- Extended `DaemonConfig` with `control_channel_id` and orchestrator settings
- Pure-function event formatters (10 functions) mapping RPC events to Discord
- GLM-5.1 added to Z.AI provider in custom models
- discord.js v14 integration, `DiscordBot` class with auth guard and lifecycle
- `packages/daemon` workspace package with `DaemonConfig`/`LogLevel` types
- Headless text mode shows tool calls + skip UAT pause in headless
- `--resume` flag resolves session IDs via prefix matching
- Migrated headless orchestrator to use `execution_complete` events

**Fixed:**
- Match "completed" status from RPC v2 in exit code mapper
- Show external drives in directory browser on Linux
- Resume cold auto bootstrap from DB
- Preserve first auto unit model after session reset
- Accept flags after positional command in headless arg parser
- Discover project subagents in `.gsd`
- Use honest `unitTypes` for discuss dispatches and map all auto-dispatch phases

**Changed:**
- Auto-commit after `complete-milestone`

---

### v2.58.0 — 2026-03-28

**Added:**
- 6 discord.js shard/error/warn event listeners for reconnect handling

**Fixed:**
- Guard `startAuto()` against concurrent invocation (#2923)
- Widen operational verification gate regex (fixes #2866) (#2898)
- Three bugs preventing reliable parallel worker execution (#2801)
- Fall back to project totals when dashboard metrics are zero (#2847)
- Parse raw YAML under preference headings (#2794)
- Persist verification classes in milestone validation (#2820)
- Guard `reconcileWorktreeDb` against same-file ATTACH corruption (#2825)
- Skip shutdown in daemon mode so server survives tab close (#2842)
- Skip `execution_complete` for multi-turn commands (auto/next)
- Fixed launchd JSON parsing, login race condition, interaction bugs

---

## Era 21: Safety, Parallelism, and Context Engineering (v2.59.0 - v2.67.0) — April 3-9, 2026

LLM safety harness, Ollama/local LLM support, MCP server read-only tools, capability-aware model routing, notification system, context engineering with tiered injection, write gate and verification hardening, parallel research slices, /btw skill, codebase map, stale commit doctor check, sidebar redesign (VS Code), persistent notification panel, adversarial review waves.

### v2.59.0 — 2026-04-03 -- KEY RELEASE

Introduced Ollama extension, codebase map, stale commit doctor check, and dynamic routing enabled by default.

**Added:**
- Ollama extension for first-class local LLM support (#3371)
- Doctor: stale commit safety check with GSD snapshot and auto-cleanup
- Extensions: wire up topological sort and unified registry filtering (#3152)
- Widget: last commit display and dashboard layout improvements (#3226)
- Model-routing: enable dynamic routing by default (#3120)
- VS Code: sidebar redesign, SCM provider, checkpoints, diagnostics [3/3]
- Splash: add remote channel indicator to welcome screen tools row
- Stream full text and thinking output in headless verbose mode (#2934)
- Codebase map -- structural orientation for fresh agent contexts

**Fixed:**
- Worktree: resolve merge conflict for PR #3322 -- adopt comprehensive pre-merge cleanup
- Merge: clean stale MERGE_HEAD before squash merge (#2912)
- State: always run disk-to-DB reconciliation when DB is available (#2631)
- Git-service: fix merge-base ancestry check and `.gsd/` leakage in snapshot absorption
- Extensions: update `provides.hooks` in 7 extension manifests to match actual registrations (#3157)
- Surface `nativeCommit` errors in `reconcileMergeState` instead of silently swallowing (#3052)
- Parallel: scope commits to milestone boundaries in parallel mode (#3047)
- Add `windowsHide` to all web-mode subprocess spawns (#2628) (#3046)
- Skip auto-mode pause on empty-content aborted messages (#2695) (#3045)
- Detect and remove nested `.git` dirs in worktree cleanup to prevent data loss (#3044)
- Prevent data loss when git isolation default changes (#2625) (#3043)
- Read-tool: clamp offset to file bounds instead of throwing (#3007) (#3042)
- Preserve queued milestones with worktrees in ghost detection (#3041)
- Compaction: add chunked fallback when messages exceed model context window (#3038)
- Preserve interactive terminal across tab switches and project changes (#3055)
- Call `cleanupQuickBranch` on `turn_end` to squash-merge quick branch back (#3054)
- Align `run-uat` artifact path to ASSESSMENT, preventing false stuck retries (#3053)
- Replace invalid Discord invite links with canonical URL (#3056)
- Add Windows shell guard to remaining spawn sites (#3058)
- Route `gsd auto` to headless runner to prevent hang on piped stdin/stdout (#3057)
- Respect `.gitignore` for `.gsd/` in rethink prompt (#3059)
- Migrate unit ownership from JSON to SQLite to eliminate read-modify-write race (#3061)
- Roadmap: handle numbered, bracketed, and indented prose H3 headers in slice parser (#3063)
- Add `worktree-merge` to `resolveModelWithFallbacksForUnit` switch and update `KNOWN_UNIT_TYPES` (#3066)
- Clean up MERGE_HEAD on all error paths in `mergeMilestoneToMain` (#2912) (#3068)
- Prevent LLM from confusing background task output with user input (#3069)
- Add openai-codex provider and modern OpenAI models to `MODEL_CAPABILITY_TIER` and cost tables (#3070)
- Preserve active tab when switching projects (#3071)
- Include project name in desktop notifications (#3072)
- Recover from many-image dimension overflow by stripping older images (#3075)
- Resolve bare model IDs to anthropic over claude-code provider (#3076)
- Auto: move `selectAndApplyModel` before `updateProgressWidget` (#3079)
- Detect project relocation and recover state without data loss (#3080)
- Add free-text input to `ask-user-questions` when "None of the above" is selected (#3081)
- Block work execution during `/gsd queue` mode (#2545) (#3082)
- Detect worktree basePath in `gsdRoot()` to prevent escaping to project root (#3083)
- Invalidate stale quick-task captures across milestone boundaries (#3084)
- Defer model validation until after extensions register (#3089)
- Repair YAML bullet lists in malformed tool-call JSON (#3090)
- Unify `SUMMARY.md` render paths for projection fidelity (#3091)
- Chat mode misrepresents terminal output, looks stuck, omits user messages (#3092)
- Resolve 4 state corruption bugs in milestone/slice completion (#2945) (#3093)
- Isolate guided-flow session state and key discussion milestone queries (#2985) (#3094)
- Guided-flow: route `dispatchWorkflow` through dynamic routing pipeline (#3153)
- Skip external state migration inside git worktrees (#2970) (#3227)
- Coerce non-numeric strings in DB columns during manifest serialization (#2962) (#3229)
- Route `allDiscussed` and zero-slices paths to queued milestone discussion (#3150) (#3230)
- Use loose equality for null checks in `secure_env_collect` (#2997) (#3231)
- Prevent prompt explosion from `$'` in template replacement values (#2968) (#3232)
- Resolve OAuth API key in `buildMemoryLLMCall` via `modelRegistry` (#2959) (#3233)
- Forensics: read completion status from DB instead of legacy file (#3129) (#3234)
- Use camelCase parameter names in execute-task and complete-slice prompts (#2933) (#3236)
- Check bootstrap completeness in init wizard gate, not just `.gsd/` existence (#2942) (#3237)
- Specify write tool for `PROJECT.md` in milestone/slice prompts (#3238)
- Widen completing-milestone gate to accept "None required" and similar phrasings (#2931) (#3239)
- Prevent `ask_user_questions` from poisoning auto-mode dispatch (#2936) (#3240)
- Guard null `s.currentUnit` in `runUnitPhase` closeout after `stopAuto` race (#2939) (#3241)
- Replace `web_search` with `search-the-web` in prompts and agent frontmatter (#2920) (#3245)
- Preserve milestone title in `upsertMilestonePlanning` when DB row pre-exists (#2879) (#3247)
- Invalidate stale milestone validation on roadmap reassessment (#2957) (#3242)
- Discuss: add roadmap fallback when DB is open but empty (#2892) (#3244)
- Integrate Codex and Gemini CLI into provider routes and rate-limit handling (#2922) (#3246)
- Error-classifier: widen `STREAM_RE` to cover all 7 V8 JSON parse error variants (#2916) (#3243)
- Prevent git stash from destroying queued milestone CONTEXT files (#2505) (#3273)
- Skip staleness rebuild in npm tarball installs (#2877) (#3250)
- Parallel: check worktree DB for milestone completion in merge (#2812) (#3256)
- Make claude-code provider stateful with full context and sidechain events (#2859) (#3254)
- Worktree: preserve non-empty `gsd.db` during sync to prevent truncation (#2815) (#3255)
- Align `@gsd/native` module type with compiled output (#3253)
- Parse `hook/*` completed-unit keys correctly in forensics + doctor (#2826) (#3252)
- Copy `mcp.json` into auto-mode worktrees (#2791) (#3251)
- Add `gsd_requirement_save` and upsert path for requirement updates (#3249)
- Handle `pause_turn` stop reason to prevent 400 errors with native web search (#2869) (#3248)
- Use authoritative milestone status in web roadmap (#2807) (#3258)
- Classify long-context entitlement 429 as `quota_exhausted`, not `rate_limit` (#2803) (#3257)
- Docs: use `~/.pi/agent/extensions/` for community extension install path (#3131) (#3259)
- Add disk-to-DB slice reconciliation in `deriveStateFromDb` (#2533) (#3262)
- Run forensics duplicate detection before investigation (#2704) (#3260)
- Skip TUI render loop on non-TTY stdout to prevent CPU burn (#3095) (#3263)
- Persist forensics report context across follow-up turns (#2941) (#3261)
- Invalidate workspace state on `turn_end` so milestones list stays current (#2706) (#3266)
- Eliminate 3 recurring doctor audit false positives (#3105) (#3264)
- Web: reconcile auto-mode state with on-disk lock in dashboard (#2705) (#3265)
- Treat ghost milestones as ineligible for parallel execution (#2501) (#3268)
- Redirect auto-mode to headless when stdout is piped (#2732) (#3269)
- Attempt VACUUM recovery when `initSchema` fails with corrupt freelist (#2519) (#3270)
- Resolve `db_unavailable` loop in worktree/symlink layouts (#2517) (#3271)
- Correct OAuth fallback request shape for `google_search` (#2963) (#3272)
- Prevent UAT stuck-loop and orphaned worktree after milestone completion (#3065)
- MCP: handle server names with spaces in `mcp_discover` (#3037)
- Detect markdown body verdicts and guard `plan-milestone` against completed slices (#2960) (#3035)
- Error-classifier: replace `STREAM_RE` whack-a-mole with catch-all V8 JSON.parse pattern
- Type `_borderColorKey` as `'dim' | 'bashMode'` to match ThemeColor
- TUI: comprehensive review -- layout, flow, rendering, and state fixes
- Harden codebase-map -- bug fixes, UX polish, and expanded tests

**Changed:**
- State: centralize pipeline logging through workflow logger (#3282)
- Gitignore: exclude `src/` build artifacts, scratch files, and `.plans/`
- Complexity: reclassify planning phases from standard to heavy tier

---

### v2.60.0 — 2026-04-04

**Added:**
- `/btw` skill -- ephemeral side questions from conversation context

**Fixed:**
- btw: remove LLM-specific references from skill description

---

### v2.61.0 — 2026-04-04

**Added:**
- Stop/backtrack capture classifications for milestone regression (#3488)
- GSD context optimization with model routing and context masking

---

### v2.62.0 — 2026-04-04

**Added:**
- Enhance `/gsd codebase` with preferences, `--collapse-threshold`, and auto-init
- Fire `before_model_select` hook, add verbose scoring output, load capability overrides
- Register `before_model_select` placeholder handler in GSD hooks
- Add `BeforeModelSelectEvent` to extension API and wire emission
- Wire `taskMetadata` from `selectAndApplyModel` to `resolveModelForComplexity`
- Insert STEP 2 capability scoring into `resolveModelForComplexity`
- Add `taskMetadata` to `ClassificationResult` and export `extractTaskMetadata`
- Add capability types, data tables, and scoring functions to model-router

**Fixed:**
- Add codebase validation in `validatePreferences` so preferences are not silently dropped
- Update `db-path-worktree-symlink` test for simplified diagnostic logging
- Update tests for errors-only audit persistence, fix empty catch blocks
- Harden audit log persistence -- errors-only, sanitized, demote probe warnings
- Address adversarial review findings on workflow-logger migration
- Fail-closed stop guard, harden backtrack parsing, fix prompt params
- Add diagnostic logging to empty catch blocks in auto-mode
- LSP: add legacy alias for renamed `kotlin-language-server` key
- Break infinite notes loop when selecting "None of the above"
- Align `defaultRoutingConfig` `capability_routing` to `true`
- Upgrade Kotlin LSP to official `Kotlin/kotlin-lsp`
- Use correct `RequirementCounts` type fields in edge case tests
- Remote-questions: fire configured channels in interactive mode

**Changed:**
- Migrate all catch blocks to centralized workflow-logger
- Init GSD

---

### v2.62.1 — 2026-04-05

**Fixed:**
- Gate steer worktree routing on active session, fix messaging
- Resolve steer overrides to worktree path when worktree is active

---

### v2.63.0 — 2026-04-05

**Added:**
- MCP-server: add 6 read-only tools for project state queries (#3515)

**Fixed:**
- Enrich vague diagnostic messages with root-cause context
- Reset dedup cache between `ask-user-freetext` tests
- DB: delete orphaned WAL/SHM files alongside empty `gsd.db` (#2478)
- Prevent auto-wrapup from interrupting in-flight tool calls (#3512)
- Handle bare model IDs in `resolveDefaultSessionModel` (#3517)
- Wrap decision and requirement saves in transaction to prevent ID races
- Prefer `PREFERENCES.md` over `settings.json` for session bootstrap model (#3517)
- Add Claude Code official skill directories to skill resolution
- Dedup: hash full question payload, not just IDs
- Prevent duplicate `ask_user_questions` dispatches with per-turn dedup cache
- pi-ai: extend `repairToolJson` to handle XML tags and truncated numbers
- pi-coding-agent: cancel stale retries after model switch

**Changed:**
- Untrack `.repowise/` and add to `.gitignore`

---

### v2.64.0 — 2026-04-06 -- KEY RELEASE

Introduced LLM safety harness, native Ollama provider, slice-level parallelism, and MCP OAuth.

**Added:**
- LLM safety harness for auto-mode damage control
- Ollama: native `/api/chat` provider with full option exposure
- Parallel: slice-level parallelism with dependency-aware dispatch (#3315)
- MCP-client: add OAuth auth provider for HTTP transport (#3295)

**Fixed:**
- UI: remove 200-column cap on welcome screen width
- Address adversarial review findings for #3576
- Replace hardcoded agent skill paths with dynamic resolution (#3575)
- Headless: sync resources and use agent dir for query
- CLI: show latest version and bypass npm cache in update check
- Follow CONTRIBUTING standards for #3565
- Address Codex adversarial review findings for #3565
- Coerce string arrays to objects in `complete-slice`/`task` tools (#3565)
- Harden flat-rate routing guard against alias/resolution gaps
- pi-coding-agent: register `models.json` providers and await Ollama probe in headless mode
- Ollama: use apiKey auth mode to avoid `streamSimple` crash
- Disable dynamic model routing for flat-rate providers
- Address Codex adversarial review findings
- Prevent LLM from querying `gsd.db` directly via bash (#3541)
- Seed requirements table from `REQUIREMENTS.md` on first update
- Inject `S##-CONTEXT.md` from slice discussion into all prompt builders
- CLI: guard model re-apply against session restore and async rejection
- pi-coding-agent: resolve model fallback race that ignores configured provider (#3534)
- Detection: add xcodegen and Xcode bundle support to project detection (#1882)
- Perf: share jiti module cache across extension loads (#3308)
- Resource-sync: prune removed bundled subdirectory extensions on upgrade (#1972)
- Recognize U+2705 checkmark emoji as completion marker in prose roadmaps (#1897)
- Web: use `safePackageRootFromImportUrl` for cross-platform package root (#1881) (#1893)
- Isolate `CmuxClient` stdio to prevent TUI hangs in CMUX (#3306)
- Worktree health check walks parent dirs for monorepo support (#3313)
- Promote milestone status from `queued` to `active` in `plan-milestone` (#3317)
- Worktree: correct merge failure notification command from `/complete-milestone` to `/gsd dispatch complete-milestone` (#1901)
- Detect and block Gemini CLI OAuth tokens used as API keys (#3296)
- Auto: break retry loop on tool invocation errors (malformed JSON) (#3298)
- Git: use `git add -u` in symlink `.gsd` fallback to prevent hang (#3299)
- Handle `complete-slice` context exhaustion to unblock downstream slices (#3300)
- Cap consecutive tool validation failures to prevent stuck-loop (#3301)
- Make enrichment tool params optional for limited-toolcall models (#3302)
- Add filesystem safety guard to `complete-slice.md` (#3304)
- Extensions: use `bundledExtensionKeys` for conflict detection instead of broken path heuristic (#3305)
- Scope tools during discuss flows to prevent grammar overflow (#3307)
- Preferences: warn on silent parse failure for non-frontmatter files (#3310)
- Track remote-questions in managed-resources manifest (#3312)
- Auto: add timeout guard for `postUnitPostVerification` in `runFinalize` (#3314)
- Handle large markdown parameters in `complete-milestone` JSON parsing (#3316)
- Metrics: deduplicate idle-watchdog entries and fix forensics false-positives (#1973)
- Prevent milestone/slice artifact rendering corruption (#3293)
- Doctor: strip `--fix` flag before positional parse (#1919) (#1926)
- Resolve external-state worktree DB path (#2952) (#3303)
- Worktree teardown path validation prevents data loss (#3311)
- Prevent auto-mode from dispatching deferred slices (#3309)
- Preserve completed slice status on `plan-milestone` re-plan (#3318)
- Reopen DB on cold resume, recognize heavy check mark (#3319)
- Dashboard model label shows dispatched model, not stale previous unit (#3320)

**Changed:**
- Remove copyright line from test file
- Trim `promptGuidelines` to 1 line to reduce per-turn token cost
- Web: consolidate subprocess boilerplate into shared runner (#1899)

---

### v2.65.0 — 2026-04-07

**Added:**
- Persistent notification panel with TUI overlay, widget, and web API
- Wire blocking behavior and strict mode for enhanced verification
- Add post-execution cross-task consistency checks
- Add pre-execution plan verification checks

**Fixed:**
- Wrap long notification messages and fit overlay to content
- Remove background color from backdrop, fix message truncation
- Restore consistent overlay height to prevent ghost artifacts
- Improve notification overlay backdrop and content-fit sizing
- Only unlink notification lock when owned, prevent foreign lock deletion
- Add backdrop dimming and viewport padding to notification overlay
- Add intent + phase guards to resume context fallback (#3615)
- Inject task context for unstructured resume prompts (#3615)
- pi-coding-agent: restore extension tools after session switch (#3616)
- Agent-loop: schema overload cap ignores bash execution errors (#3618)
- bg-shell: prevent signal handler accumulation + cap alert queue
- Coerce plain-string `provides` field to array in `complete-slice` (#3585)
- Address PR #3468 review findings
- Persist `autoStartTime` across session resume so elapsed timer survives `/exit`
- Add `enhanced_verification` preferences to `mergePreferences`
- Headless: treat discuss and plan as multi-turn commands

**Changed:**
- Interactive: cap rendered chat components + kill orphan descendants
- TUI: render-skip, frame isolation, Text cache guard, dispose

---

### v2.66.0 — 2026-04-08 -- KEY RELEASE

Introduced parallel research slices, graph diagnostics, and five waves of adversarial review hardening.

**Added:**
- Fast path for queued milestone discussion
- `/gsd show-config` command
- Reactive: graph diagnostics and `subagent_model` config
- Dispatch: parallel research slices and parallel milestone validation
- Parallel: worker model override for parallel milestone workers

**Fixed:**
- Validate depth verification answer before unlocking write-gate
- Revert unknown artifact check to warn-and-proceed
- Add missing `cmd` field to test base `WorkflowEvent`
- Address remaining adversarial review findings for wave 3
- Detect concurrent event log growth during reconcile
- Address adversarial review findings for wave 3
- Address adversarial review findings for wave 2
- Address adversarial review findings for wave 1
- WAL-safe migration backup + stronger regression tests
- Consistency and cleanup (wave 5/5)
- Write safety -- atomic writes and randomized tmp paths (wave 4/5)
- Session and recovery robustness (wave 3/5)
- Event log and reconciliation robustness (wave 2/5)
- Critical state machine data integrity fixes (wave 1/5)
- Remove ecosystem research stub and address adversarial review
- Suppress model change notification in auto-mode unless verbose
- Exclude `task.files` from `checkTaskOrdering` to prevent false positives
- State: skip ghost check for queued milestones in registry build
- CI: replace empty catch blocks and raw stderr with `logWarning`
- Logging: add `debugLog` to empty catch in reopen-milestone
- State-machine: 9 resilience fixes + 86 regression tests (#3161)
- Add incremental persistence to discuss prompts
- Replace empty catch with `logWarning` for silent-catch-diagnostics test
- Test: escape regex metacharacters in skip-by-preference pattern test
- Test: search for numbered step definitions in prompt ordering test
- Test: update notes loop test for `notesVisible` guard behavior
- Test: update action count for note captures now included in results
- Test: remove extraneous test file from wrong branch
- Test: update worktree sync tests to use separate milestone IDs
- Use valid `LogComponent` type for stale branch guard warning
- Test: update rogue detection test for auto-remediation behavior
- Test: update stuck-planning test to expect executing after reconciliation
- Test: update file path consistency tests for inputs-only checking
- Test: add CONTEXT file to queued milestone ghost detection test
- Test: update needs-remediation test to expect `validating-milestone` phase
- Import all-done milestones as complete during DB migration
- Allow milestone completion when validation skipped by preference
- Set slice sequence at all three insertion sites
- Four prompt/runtime fixes for completion and session stability
- Default `insertMilestone` status to `queued` instead of `active`
- Suppress repeated frontmatter YAML parse warnings
- Normalize list inputs in `complete-task` + fix roadmap dep parsing
- Open DB before status derivation + respect `isolation:none` in quick
- Add `.bg-shell/` to baseline gitignore patterns
- TUI: prevent Enter key infinite loop in interview notes mode
- Provider: handle Enter key to initiate auth setup in provider manager
- MCP: use `createRequire` to resolve SDK wildcard subpath imports
- Mark note captures as executed in `executeTriageResolutions`
- Validate `main_branch` preference exists before using in merge
- Handle deleted cwd in `projectRoot` to prevent ENOENT crash
- Skip current milestone in `syncWorktreeStateBack` to prevent merge conflicts
- Add `structuredQuestionsAvailable` conditional to slice discuss
- Restore full tool set after discuss flow scoping
- Tighten `verifyExpectedArtifact` to prevent rogue-write false positives
- Add verification gate to `complete-slice` tool
- Fix pre-execution-checks false positives from backticks and `task.files`
- Stop `renderAllProjections` from overwriting authoritative `PLAN.md`
- Auto-checkout to main when `isolation:none` finds stale milestone branch
- Auto-remediate stale slice DB status when SUMMARY exists on disk
- Open DB on demand in `gsd_milestone_status` for non-auto sessions
- Detect phantom milestones from abandoned `gsd_milestone_generate_id`
- Force re-validation when verdict is `needs-remediation`
- Exclude closed slices from `findMissingSummaries` check
- Recover from stale lockfile after crash or SIGKILL
- Add `createdAt` timestamp and 30s age guard to staleness check
- Clear stale `pendingAutoStart` after `/clear` interrupts discussion
- Suppress misleading warnings for expected ENOENT/EISDIR conditions
- Extract real error from message content when `errorMessage` is useless
- Show accurate pause message for queued-user-message skip
- Treat queued-user-message skip as non-retryable interruption
- Recognize "Not provided." default in `isVerificationNotApplicable`
- `discoverManifests` skips symlinked extension directories
- Reconcile plan-file tasks into DB when planner skips persistence (#3600)
- Use `isClosedStatus()` in dispatch guard instead of raw complete check
- Browser-tools: make `sharp` an optional lazy dependency
- Pass required arguments in defer-milestone-stamp test
- Replace remaining empty catch with `logWarning`
- Use `logWarning` instead of raw stderr in catch blocks
- Log error instead of empty catch in `STATE.md` rebuild
- Log error instead of empty catch in `skip_slice`
- Cast milestone classification to string for type safety
- Treat zero-slice roadmap as pre-planning in guided flow
- Rebuild `STATE.md` after skip-slice and strengthen rethink prompt
- Use `main_branch` preference in worktree creation
- Stamp defer and milestone captures as executed after triage
- TUI: treat absolute file paths as plain text, not commands
- TUI: break infinite re-render loop for images in cmux
- Rebuild `STATE.md` before guided-flow dispatch
- Defer queued shells in active milestone selection
- Retry: prevent 429 quota cascade and 30-min lockout
- Add `fastPathInstruction` to `buildDiscussMilestonePrompt` `loadPrompt` call

**Changed:**
- Auto-commit after quick-task

---

### v2.66.1 — 2026-04-08

**Fixed:**
- pi-tui: revert `contentCursorRow`, use `hardwareCursorRow` as movement baseline
- pi-tui: use `contentCursorRow` for render movement baseline instead of `cursorRow`
- Add `logWarning` to empty catch block in orphaned worktree cleanup
- Add `consecutiveFinalizeTimeouts` to `LoopState` in journal tests
- Add escalation and unit-detach guards to finalize timeout handlers
- Add timeout guard around `postUnitPreVerification` to prevent auto-loop hang
- OS-specific keyboard shortcut hints via `formatShortcut` helper
- Subagent: support list-style tools frontmatter
- Clear autocomplete rows from content bottom
- Parse annotated pre-exec file paths
- Add orphaned milestone branch audit at auto-mode bootstrap

---

### v2.67.0 — 2026-04-09 -- KEY RELEASE

Introduced tiered context injection (M005) with 65%+ token reduction and decision scope cascade (R005).

**Added:**
- Context: implement R005 decision scope cascade and derive scope from slice metadata
- M005: Tiered Context Injection -- relevance-scoped context with 65%+ reduction

**Fixed:**
- Test: align auto-loop test timers with updated session timeout
- Repair CI after branch split (3 occurrences)
- Fail closed for discussion gate enforcement
- Harden auto merge recovery and session safety
- Repair overlay, shortcut, and widget surfaces
- Prevent stale workflow reconcile state writes
- Align prompt contracts and validation flow
- pi-tui: harden input parsing and editor focus behavior
- Remote-questions: cancel local TUI when remote answer wins the race
- Auto: increase session timeout to 120s and treat timeout as recoverable pause (#3767)
- UI: apply `anthropic-api` display name to all model/provider UI surfaces
- UI: display `anthropic-api` in GSD preferences wizard provider list
- Remote-questions: race local TUI against remote channel instead of remote-only routing
- UI: display `anthropic-api` in model selector to distinguish from claude-code
- Gates: add mechanical enforcement for discussion question gates
- Prompts: harden non-bypassable gates and exclude dot-folders from scanning
- Ignore filename headings in `parsePlan`
- Providers: match 'out of extra usage' error and respect claude-code provider in model resolution (#3772)
- pi-ai: recover XML parameters trapped in JSON strings
- Retry: guard claude-code fallback to anthropic provider only
- Providers: route Anthropic subscription users through Claude Code CLI (#3772)
- claude-code: use native Windows claude lookup
- Suppress repeated preferences section warnings
- Normalize described expected output paths
- Auto: resilient transient error recovery -- defer to Core RetryHandler and fix `cmdCtx` race

---

## Summary Statistics

| Metric | Count |
|--------|-------|
| Total released versions | 111 |
| v0.x releases | 9 |
| v2.3.x releases | 8 |
| v2.4.x - v2.9.x releases | 11 |
| v2.10.x releases | 12 |
| v2.11.x - v2.33.x releases | 33 |
| v2.34.x - v2.58.x releases | 27 |
| v2.59.x - v2.67.x releases | 11 |
| Development period | March 11 - April 9, 2026 (30 days) |
| Key milestone releases | 20 |
| Total eras | 21 |
