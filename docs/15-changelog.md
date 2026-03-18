# Changelog

This document covers released versions v2.24.0 through v2.28.0. All versions were released in March 2026. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

---

## v2.28.0 — 2026-03-17

### Added

- **`gsd headless query`** — instant, read-only state inspection that returns the current phase, cost, progress percentage, and next scheduled unit as parseable JSON. Unlike `gsd headless auto`, this command does not spawn an LLM session and returns immediately. Useful for polling GSD state from external scripts, dashboards, or CI status checks.
- **`/gsd update`** — in-session self-update slash command. Allows updating GSD to the latest version without leaving the active session.
- **`/gsd export --html --all`** — retrospective HTML reports covering all milestones, not just the current one. Generates a self-contained HTML file with the full project history for archiving or handoff.

### Fixed

- **Failure recovery and resume safeguards** — a broad set of hardening improvements: atomic file writes (prevents partial writes on crash), OAuth fetch timeouts (30-second cap), RPC subprocess exit detection, extension command context guards to prevent use-after-free errors, bash temp file cleanup, settings write queue flush on exit, LSP initialization retry with exponential backoff, crash detection on session resume, and blob garbage collection.
- Consolidated the duplicate `mcp-server.ts` implementation into a single canonical file, removing an ambiguity that could cause inconsistent behavior depending on which copy was loaded.
- Consolidated the duplicate `bundled-extension-paths.ts` module for the same reason.
- Removed the duplicate `marketplace-discovery.test.ts` test file that was causing false positives in CI.
- Established npm as the canonical package manager, replacing inconsistent tooling references across build scripts.
- Exported RPC utilities from the `pi-coding-agent` public API surface, making them available to external integrations.
- Prompt system now requires grammatical narration for agent output, producing cleaner and more readable step-by-step descriptions.

### Changed

- Documentation updated to cover v2.26 and v2.27.0 features.
- All `preferences.md` fields are now documented in both the reference guide and the template file.
- Removed stale `.pi/agents/` files that were superseded by built-in agent definitions in earlier versions.

---

## v2.27.0 — 2026-03-17

### Added

- **HTML report generator** — generates a self-contained HTML report in `.gsd/reports/` with a progression index across milestones. Includes progress tree, dependency graph (SVG DAG), cost/token bar charts, execution timeline, changelog, and knowledge base. All assets inlined; printable to PDF. An auto-generated `index.html` shows all reports with progression metrics. Configurable with `auto_report: true` to generate automatically after milestone completion.
- **Crash recovery for parallel orchestrator** — persisted state with PID liveness detection. If the orchestrator process dies, state is recovered on restart and workers that were orphaned are detected via PID checks.
- **Headless orchestration skill with supervised mode** — extends `gsd headless` with a supervised execution mode suitable for unattended overnight runs. Combined with the existing auto-restart on crash (exponential backoff, 3 attempts), this enables true unattended execution.
- **Verification enforcement gate for milestone completion** — milestones must now pass a verification step before being marked complete. This prevents partially-complete milestones from being closed prematurely.
- **`git.manage_gitignore` preference** — opt out of automatic `.gitignore` changes. When set to `false`, GSD will not modify `.gitignore` during team setup or worktree creation.

### Fixed

- Single `ENTER` now correctly submits slash command argument autocomplete instead of requiring a second keypress.
- Web search loop broken by a consecutive-duplicate guard that prevents the same query from being issued twice in a row.
- Transient network errors are now retried before triggering model fallback, reducing unnecessary fallback churn.
- Parallel worker PID tracking, spawn-status race, and exit persistence corrected to prevent ghost workers.
- `/gsd discuss` now recommends the next undiscussed slice rather than re-suggesting an already-discussed one.
- Roadmap parser now allows suffix text after the `## Slices` heading (previously caused a parse failure).
- User's model choice is no longer overwritten when the API key is temporarily unavailable (e.g., network blip during key validation).
- Reassess-roadmap skip loop broken by preventing re-persistence of evicted cache keys.
- LSP command resolution and `ENOENT` crash on Windows/MSYS path environments corrected.
- Dispatch plan-slice when task plan files are missing rather than silently skipping.
- Reduced CPU usage on long auto-mode sessions (previously, event loop polling accumulated over time).
- Orphan-prone child processes reaped across session churn to prevent zombie processes accumulating.
- Symlinked skill directories now resolved correctly in `always_use_skills` and preferences lookups.
- Replan-slice infinite loop and non-standard `finish_reason` crash fixed.
- Slice plan commit skipped correctly when `commit_docs` is `false`.
- Context-pressure monitor wired to send wrap-up signal at 70% context window consumption.
- Missing `STATE.md` in a fresh worktree no longer deadlocks the pre-dispatch health gate.
- Stale runtime unit files for completed milestones are cleaned on startup.
- Broken install detection improved with Windows symlink fallback.
- Auto-restart for headless mode on crash with exponential backoff implemented.
- Generic type now preserved on `resolveModelId` through the full resolution chain.

### Changed

- Auto-mode state encapsulated into `AutoSession` class, replacing module-level mutable state. This is a significant internal refactor that improves session lifecycle management and testability. The `auto-session-encapsulation.test.ts` CI gate enforces this constraint going forward.
- Extracted 7 focused modules from the previously monolithic `auto.ts`: `auto-worktree-sync`, resource staleness tracking, and stale-escape logic among them.
- TUI dashboard cleaned up, deduplicated, and improved with new features.
- Visualizer tabs and HTML report sections reordered into more logical groupings.

---

## v2.26.0 — 2026-03-17

### Added

- **Model selector grouped by provider** — the model picker now groups entries by provider and shows model type, provider name, and a link to API documentation for each option.
- **`require_slice_discussion`** — new preference that pauses auto-mode before each slice and waits for a human discussion step. Designed for teams that want to review requirements at the slice level before automated execution begins.
- **Discussion status indicators in `/gsd discuss`** — the slice picker now shows which slices have been discussed and what state the discussion left off in.
- **Worker NDJSON monitoring and budget enforcement for parallel orchestration** — each parallel worker now streams NDJSON status, and the orchestrator enforces budget limits across workers. A dashboard alert fires at 80% budget consumption.
- **`gsd_generate_milestone_id` tool** — generates unique IDs for multi-milestone workflows, ensuring no ID collisions when multiple milestones are created in a single operation.
- **Alt+V clipboard image paste** on macOS for pasting images directly into GSD prompts.
- **Hashline edit mode** integrated into active workflow.
- **Fallback parser for prose-style roadmaps** — roadmaps that do not have a `## Slices` section are now parsed using a fallback prose parser rather than failing.

### Fixed

- Windows path normalization in LLM-visible text prevents bash failures caused by backslash path separators.
- Async bash job completion no longer triggers spurious extra LLM turns.
- Native web search limited to a maximum of 5 uses per response to prevent runaway search loops.
- Completed milestones with a summary file are no longer re-entered as active on session resume.
- Replan-slice artifact verification breaks the infinite replanning-slice loop that could occur when verification was not enforced.
- `STATE.md` missing in pre-dispatch health gate is now auto-healed rather than causing a hard failure.
- Worktree artifact copy now includes `STATE.md`, `KNOWLEDGE.md`, and `OVERRIDES.md` — previously these were omitted, causing incomplete worktree state.
- Transient network errors marked as retriable in the Anthropic provider (previously fell through to permanent-error handling).
- `needs-remediation` validation verdict now treated as terminal to prevent a hard looping condition.
- Post-hook doctor uses `fixLevel: 'all'` to fix roadmap checkboxes correctly.
- `task_done_missing_summary` now fixable in doctor, preventing the validate-milestone skip loop.
- BMP clipboard images on WSL2 handled via `wl-paste` PNG conversion.
- Extended idle timeout for headless `new-milestone` to prevent premature timeout on slow LLM responses.
- EPIPE handling in LSP `sendNotification` with proper process exit wait.
- Debug logging added for silent early-return paths in `dispatchNextUnit`.
- Untracked `.gsd/` state files removed before milestone merge checkout to prevent checkout conflicts.
- Crash prevention when cancelling the OAuth provider login dialog.
- Resource staleness check now compares `gsdVersion` instead of `syncedAt` for more accurate staleness detection.
- Unique temp paths in `saveFile()` to prevent parallel write collisions.
- Validation and summary file generation for completed milestones during migration from older versions.
- Cache invalidation before initial state derivation in `startAuto` to prevent stale-state startup issues.
- Headless mode no longer exits early when a progress notification string contains the word `'complete'`.

### Removed

- Symlink-based development workflow (reverted PR #744).

### Changed

- Explicit Gemini OAuth terms-of-service warning added to README — recommends API keys over OAuth for production use.
- Documentation updated for v2.24 release features.
- Bug report template updated with model type, provider, and API documentation fields.

---

## v2.25.0 — 2026-03-16

### Added

- **Native web search results rendering in TUI** — search results are displayed directly in the terminal interface rather than as raw text. The `PREFER_BRAVE_SEARCH` environment variable selects Brave Search as the backend when set.
- **Meaningful commit messages generated from task summaries** — instead of generic automated commit messages, GSD now generates a commit message derived from the task's actual summary, producing a more useful git history.
- **Incremental memory system for auto-mode sessions** — auto-mode now maintains incremental context between tasks within a session, improving coherence across long-running executions.
- **Visualizer enriched with stats and discussion status** — the `/gsd visualize` overlay now shows discussion status indicators and additional statistics alongside the existing progress, dependencies, metrics, and timeline tabs.
- **14 new E2E smoke tests** — CLI verification coverage expanded with 14 additional end-to-end tests.

### Fixed

- Phantom skip loop caused by stale crash recovery context — previously, a crash during a skip sequence could leave recovery context that caused future sessions to enter a skip loop immediately on startup.
- Skip-loop now interruptible and counts toward the lifetime cap, preventing an infinite skip loop from running forever undetected.
- Cache invalidation consistency — orphaned `invalidateStateCache()` calls replaced with coordinated `invalidateAllCaches()`, and the DB artifact cache is now included in the full cache invalidation pass.
- Plan checkbox reconciliation on worktree re-attach after crash — previously, checkboxes could be out of sync if a crash occurred mid-task in a worktree.

### Changed

- Removed unnecessary `as any` casts, dead exports, and duplicate code across the codebase.
- Documentation updated for v2.22 and v2.23 release features.

---

## v2.24.0 — 2026-03-16

### Added

- **Parallel milestone orchestration** — run multiple auto-mode workers simultaneously across phases. Each worker operates on an independent milestone with its own worktree and branch. A dashboard view shows all parallel workers with an 80% budget alert. This is the foundation for multi-developer GSD workflows at scale.
- **Dashboard view for parallel workers** — the `Ctrl+Alt+G` dashboard now has a panel showing the status of all active parallel workers, including per-worker budget consumption.
- **Headless `new-milestone` command** — `gsd headless new-milestone` creates a milestone programmatically without interactive prompts. Designed for use in CI pipelines or automation scripts.
- **Interactive update prompt on startup** — when a newer version of GSD is available, the startup sequence now shows an interactive prompt offering to update immediately.
- **Symlink-based development workflow for `src/resources/`** — simplifies local development by symlinking resource directories instead of copying them. (Note: this was subsequently reverted in v2.26.0, PR #744.)
- **Descriptions added to `/gsd` autocomplete commands** — each slash command in the autocomplete list now shows a short description.
- **`validate-milestone` phase and dispatch** — a new phase that runs validation checks on a completed milestone before it is closed. Ensures all tasks are complete, all summaries are present, and the milestone is in a consistent state.

### Fixed

- Sync `completed-units.json` across worktree boundaries — previously, units completed in a worktree were not reflected in the root's `completed-units.json`.
- Worktree artifact verification now uses the correct base path.
- Auto-resume after rate limit cooldown — auto-mode now resumes automatically after a rate limit delay rather than requiring manual intervention.
- Raised `maxDelayMs` default from 60 seconds to 300 seconds for better handling of prolonged rate-limit windows.
- Downgraded `missing_tasks_dir` from error to warning for completed slices — a completed slice without a tasks directory is a normal state, not an error.
- Prevented stale-state loop on auto-mode restart when an existing worktree is present.
- Always sync bundled resources and clean stale files on startup to ensure the installed version's resources match what is on disk.
- Added stop reason to every auto-mode stop event for better diagnostics.
- Skipped redundant checkout in worktree merge when `main` is already the current branch.
- Prevented runaway `execute-task` dispatch when the task plan is missing after a failed research phase.
- Fixed read-only file permissions after `cpSync` from the Nix store.
- Fixed parallel `sendMessage` calls that were missing required fields, causing silent failures in multi-worker sessions.
- Stripped the clack UI from the `postinstall` script while keeping the silent Playwright browser download.

### Changed

- Lazy-load LLM provider SDKs — provider-specific SDK modules (Anthropic, OpenAI, etc.) are now loaded on demand rather than at startup, reducing startup time.

---

## Migration Notes

### v2.24.0 → v2.25.0

No breaking changes. The cache invalidation improvements in v2.25.0 may cause a slightly longer first startup as stale caches are rebuilt, but this is a one-time cost.

### v2.25.0 → v2.26.0

The symlink-based development workflow introduced in v2.24.0 was reverted in v2.26.0 (PR #744). If you were relying on symlinked `src/resources/` for local development, revert to the copy-based approach.

The `require_slice_discussion` preference introduced in v2.26.0 defaults to `false` (auto-mode behavior unchanged). No action needed unless you want to adopt per-slice discussion gates.

### v2.26.0 → v2.27.0

The `AutoSession` class encapsulation in v2.27.0 is an internal refactor with no user-facing API changes. However, if you have any custom extensions or scripts that access internal auto-mode state directly (module-level exports from `auto.ts`), these may need to be updated to use the `AutoSession` class interface. The CI gate `auto-session-encapsulation.test.ts` enforces this constraint for the core codebase.

The `git.manage_gitignore` preference was added in v2.27.0. If you previously relied on GSD modifying `.gitignore` automatically and want to retain that behavior, no change is needed — it defaults to `true`. Set to `false` only if you want to manage `.gitignore` manually.

### v2.27.0 → v2.28.0

No breaking changes. The new `gsd headless query` command is additive. The prompt system change (grammatical narration requirement) affects agent output text only and does not change command behavior or file formats.
