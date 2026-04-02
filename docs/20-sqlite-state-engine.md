# SQLite State Engine

## Overview

GSD-2 replaces its original markdown-file-based state management with atomic SQLite operations. Introduced in v2.44.0 as the tool-driven write-side transitions milestone (M001), this architecture shift brings transactional safety, eliminates filesystem race conditions, and provides a single source of truth for project state.

The SQLite state engine runs through a provider fallback chain: `node:sqlite` (built-in, Node >= 22) then `better-sqlite3` (npm). If neither is available, the engine degrades gracefully with file-based fallbacks for DB-dependent code paths.

## Motivation

The markdown-file-based state system suffered from several reliability problems documented in ADR-003:

- **TOCTOU races**: filesystem reads followed by writes could be interleaved by concurrent processes, corrupting state when multiple auto-loop workers or worktrees operated on the same `.gsd/` tree.
- **Concurrent write conflicts**: parallel slice execution and worktree-based isolation both required safe concurrent access to project state. File-level locking was fragile and platform-dependent.
- **State inconsistency between markdown files**: state was spread across ROADMAP.md checkboxes, PLAN.md task lists, SUMMARY.md files, and STATE.md renders. A crash between writing two of these files left the project in an inconsistent state.
- **Markdown parsing fragility**: the legacy parser needed to handle heading-style tasks, checkbox-style tasks, prose-format slices, and multiple ROADMAP table formats. Edge cases in parsing caused phantom tasks, missed completions, and dispatch deadlocks.
- **Context re-ingestion overhead**: ADR-003 measured approximately 208K tokens of pure waste per milestone from re-reading unchanged documents across 30 sessions in the quality profile pipeline.

## Schema Evolution

The database uses a `schema_version` table to track migrations. Each version adds columns or tables non-destructively, and `migrateSchema()` runs automatically on `openDatabase()`.

### v5: Core hierarchy tables

Initial creation of `milestones`, `slices`, `tasks`, and `verification_evidence` tables. These store the milestone/slice/task hierarchy that was previously encoded in ROADMAP.md checkboxes and PLAN.md task lists.

### v6: Slice summary and UAT columns

Added `full_summary_md` and `full_uat_md` to the `slices` table, allowing slice completion artifacts to be stored alongside their structural data.

### v7: Dependency tracking

Added `depends` (JSON array) to `slices` and `depends_on` to `milestones`, enabling the dependency resolver to query the DB directly instead of parsing markdown frontmatter.

### v8: Planning data groundwork

Major expansion for DB-backed tool guidance. Added planning columns to all three hierarchy tables:

- **milestones**: `vision`, `success_criteria`, `key_risks`, `proof_strategy`, verification columns (`verification_contract`, `verification_integration`, `verification_operational`, `verification_uat`), `definition_of_done`, `requirement_coverage`, `boundary_map_markdown`
- **slices**: `goal`, `success_criteria`, `proof_level`, `integration_closure`, `observability_impact`
- **tasks**: `description`, `estimate`, `files`, `verify`, `inputs`, `expected_output`, `observability_impact`

Also added `replan_history` and `assessments` tables for tracking slice replanning events and milestone assessments.

### v9: Sequence column

Added `sequence INTEGER DEFAULT 0` to both `slices` and `tasks` tables. Tools set this to control execution order, and queries use `ORDER BY sequence, id` to respect it.

### v10: Replan trigger column

Added `replan_triggered_at TEXT` to `slices`. The `deriveStateFromDb` function uses this column (along with `replan_history`) to detect triage-initiated replans and transition to the `replanning-slice` phase.

### v11: Task plan and replan idempotency

Added `full_plan_md` to `tasks` for storing the rendered task plan content. Added a unique index on `replan_history(milestone_id, slice_id, task_id)` to make replan insertions idempotent -- retrying the same replan silently updates the summary instead of accumulating duplicate rows.

### v12: Quality gates

Added the `quality_gates` table with columns for `milestone_id`, `slice_id`, `gate_id`, `scope`, `task_id`, `status`, `verdict`, `rationale`, `findings`, and `evaluated_at`. Gates Q3-Q8 are evaluated by parallel sub-agents before slice execution begins.

### v13: Hot-path dispatch indexes

Added performance indexes for the auto-loop's most frequent queries:

- `idx_tasks_active` on `tasks(milestone_id, slice_id, status)`
- `idx_slices_active` on `slices(milestone_id, status)`
- `idx_milestones_status` on `milestones(status)`
- `idx_quality_gates_pending` on `quality_gates(milestone_id, slice_id, status)`
- `idx_verification_evidence_task` on `verification_evidence(milestone_id, slice_id, task_id)`

### v14: Slice dependency junction table

Added `slice_dependencies` as a proper junction table replacing the JSON `depends` array for dependency lookups. The table has a composite primary key `(milestone_id, slice_id, depends_on_slice_id)` and a reverse-lookup index `idx_slice_deps_target`. The `syncSliceDependencies()` function keeps it in sync with the JSON array.

## Tool-Driven Write-Side Transitions

Starting in v2.44.0, DB tools replace direct markdown file mutation for all state-changing operations. The agent calls structured tool APIs instead of writing files, and the tool handlers perform atomic DB writes followed by filesystem rendering.

### Registered DB tools

The `registerDbTools()` function in `bootstrap/db-tools.ts` registers the following tools (with aliases for backward compatibility):

| Tool | Purpose |
|------|---------|
| `gsd_decision_save` | Record a decision to DB, regenerate DECISIONS.md |
| `gsd_requirement_save` | Record a requirement to DB, regenerate REQUIREMENTS.md |
| `gsd_requirement_update` | Update an existing requirement in DB |
| `gsd_summary_save` | Save SUMMARY/RESEARCH/CONTEXT/ASSESSMENT artifacts to DB and disk |
| `gsd_milestone_generate_id` | Generate next milestone ID (inserts a minimal DB row) |
| `gsd_plan_milestone` | Write milestone + slice planning data, render ROADMAP.md |
| `gsd_plan_slice` | Write slice + task planning data, render PLAN.md and task plans |
| `gsd_plan_task` | Write individual task planning data, render task PLAN file |
| `gsd_task_complete` | Record completed task with verification evidence |
| `gsd_slice_complete` | Record completed slice, render SUMMARY.md + UAT.md |
| `gsd_complete_milestone` | Record completed milestone, render MILESTONE-SUMMARY.md |
| `gsd_validate_milestone` | Persist validation results, render VALIDATION.md |
| `gsd_replan_slice` | Replan after blocker discovery, enforce completed task preservation |
| `gsd_reassess_roadmap` | Reassess roadmap after slice completion, enforce completed slice preservation |
| `gsd_save_gate_result` | Save quality gate evaluation result (Q3-Q8) |

### renderCall / renderResult previews (v2.45.0)

Each DB tool includes `renderCall` and `renderResult` methods that produce TUI-friendly previews. `renderCall` shows the operation being performed (e.g., tool name, scope, and key parameters). `renderResult` shows the outcome (e.g., "Decision D003 saved" or error details). These replaced opaque tool-call output with structured, color-coded feedback.

### Atomic operations via SQLite transactions

Tool handlers write to the DB inside a transaction, then perform filesystem writes (ROADMAP.md, PLAN.md, SUMMARY.md) outside the transaction. If the DB write succeeds but the disk write fails, the tool rolls back the DB row and re-throws. This ensures the DB and filesystem never diverge: either both are updated or neither is.

### Migration path: callers and auto-prompts

The v2.44.0 release migrated all state-writing callers to the DB tool API across six slices (S01-S06):

- **S01**: Planning prompts migrated to DB-backed tool guidance
- **S04**: `auto-dispatch.ts` (3 rules), `auto-verification.ts`, and `dispatch-guard.ts` migrated to DB queries with `isDbAvailable` fallback
- **S05**: 7 warm/cold callers (doctor, doctor-checks, visualizations, etc.) migrated; `migrateHierarchyToDb` extended to populate v8 planning columns; 6 remaining callers (auto-prompts, auto-recovery) migrated
- **S06**: All 16 lazy `createRequire` fallback paths stripped from the migration code

By v2.47.0, the remaining 4 prompts were migrated to use the DB-backed tool API instead of direct file writes.

## Single-Writer Engine v2

Introduced in v2.46.0 as a discipline layer on the DB architecture. The single-writer engine v2 prevents agents from bypassing the DB state machine by intercepting direct file writes to authoritative state files.

The `write-intercept.ts` module defines blocked patterns for `.gsd/STATE.md` -- the only purely engine-rendered file. When an agent attempts to write STATE.md directly (via file write tools or bash redirects/tee/cp/mv/sed), the write is blocked with an error directing the agent to use engine tool calls instead.

Blocked patterns cover:

- Direct file path matches: `/.gsd/STATE.md` (case-insensitive for macOS APFS)
- Resolved symlink paths under `~/.gsd/projects/`
- Bash command patterns: redirects (`>`/`>>`), `tee`, `cp`, `mv`, `sed -i`, `dd` targeting STATE.md

The `BLOCKED_WRITE_ERROR` message enumerates the correct tool calls for each state transition.

## Single-Writer Engine v3

Also introduced in v2.46.0 alongside v2, the v3 engine adds:

- **State machine guards**: enforcement that state transitions follow valid paths (e.g., a task cannot be completed before its slice exists, a milestone cannot be completed before all slices are done)
- **Actor identity tracking**: the `made_by` field on decisions distinguishes `human`, `agent`, and `collaborative` origins (schema v4+)
- **Reversibility of state transitions**: the `replan_history` table and `replan_triggered_at` column enable structured rollback when blockers are discovered
- **TOCTOU closure**: the v2.46.0 changelog explicitly notes "close TOCTOU, intercept bypasses, status inconsistencies" as hardening work on the single-writer engine
- **Bare-relative-path bypass fix**: the STATE.md regex was updated to match paths without a leading separator, closing a bypass where `.gsd/STATE.md` (without `/`) could sneak through

## Workflow Logger

The `workflow-logger.ts` module provides a centralized warning/error accumulator wired into the engine, tool, manifest, and reconcile paths (v2.46.0). It serves as the audit trail for state transitions.

### Architecture

- **In-memory buffer**: up to 100 `LogEntry` objects with timestamp, severity (`warn`/`error`), component tag, message, and optional structured context
- **Persistent audit log**: every entry is appended to `.gsd/audit-log.jsonl` for post-mortem analysis
- **Stderr forwarding**: all entries write immediately to stderr for terminal visibility (no disable flag -- this is operational logging, not debug logging)

### Component tags

Log entries are tagged with the originating component: `engine`, `projection`, `manifest`, `event-log`, `intercept`, `migration`, `state`, `tool`, `compaction`, `reconcile`, `db`, `dispatch`.

### Auto-loop integration

The auto-loop calls `drainAndSummarize()` at the end of each unit to surface root causes for stuck loops, silent degradation, and blocked writes. `_resetLogs()` is called at the start of each unit to prevent log bleed between units in the same process. The `setLogBasePath()` function must be called once at engine init (v2.52.0 fix: `setLogBasePath` wired into engine init to resurrect the audit log).

## DB-Backed State Derivation

The `deriveStateFromDb()` function in `state.ts` is the primary state derivation path when the database is available. It replaces the filesystem-based `_deriveStateImpl()` which is retained as a legacy fallback for unmigrated projects.

### How it works

1. Loads all milestones from `getAllMilestones()`
2. Performs incremental disk-to-DB reconciliation: milestone directories on disk that are missing from the DB are inserted with `INSERT OR IGNORE`
3. Performs slice reconciliation: slices defined in ROADMAP.md but missing from the DB are parsed and inserted, with status derived from SUMMARY file existence
4. Respects `queue-order.json` for milestone ordering (v2.51.0 fix)
5. Resolves the active milestone, slice, and task using DB queries with dependency checks
6. Reconciles stale task status from disk artifacts: when a session disconnects after writing SUMMARY but before updating the DB, tasks are reconciled from SUMMARY file existence (v2.50.0, #2514)
7. Returns a `GSDState` object identical in shape to the filesystem-based derivation

### Reconciliation: disk to DB (v2.45.0, v2.52.0)

Multiple reconciliation layers ensure the DB stays in sync with disk state:

- `deriveStateFromDb()` inserts milestones found on disk but missing from DB (#2416)
- Disk-only milestones are reconciled into the DB before the `deriveStateFromDb` guard runs (#2686, v2.52.0)
- Slices in ROADMAP.md but missing from DB are inserted to prevent permanent "No slice eligible" blocks (#2533)
- Task status is reconciled from SUMMARY files on disk when DB status is stale (#2514, v2.50.0)

### Telemetry

The module tracks `dbDeriveCount` and `markdownDeriveCount` via `getDeriveTelemetry()` for observability into which derivation path is being used.

## WAL Mode and Sidecars

### WAL (Write-Ahead Logging)

The database is opened in WAL mode for file-backed databases (`PRAGMA journal_mode=WAL`). WAL mode enables concurrent readers with a single writer, which is critical for worktree-based parallel execution where multiple processes may read state while one writes.

Additional performance pragmas set during `initSchema()`:

- `PRAGMA busy_timeout = 5000` -- wait up to 5 seconds for lock acquisition
- `PRAGMA synchronous = NORMAL` -- durability with reduced fsync overhead
- `PRAGMA auto_vacuum = INCREMENTAL` -- reclaim space without full vacuum blocking
- `PRAGMA cache_size = -8000` -- 8 MB page cache
- `PRAGMA mmap_size = 67108864` -- 64 MB memory-mapped I/O
- `PRAGMA temp_store = MEMORY` -- temporary tables in RAM
- `PRAGMA foreign_keys = ON` -- enforce referential integrity

### Sidecar exclusion (v2.44.0)

SQLite WAL mode creates sidecar files (`-wal` and `-shm`) alongside the database file. These were added to runtime exclusion lists to prevent git from tracking them and to avoid confusing file watchers (#2299).

### ATTACH corruption guard (v2.58.0)

The `reconcileWorktreeDb()` function uses `ATTACH DATABASE` to merge a worktree's database back into the main DB. When both paths resolve to the same physical file (e.g., a symlink), ATTACH corrupts the WAL. A guard using `realpathSync()` was added to bail out when both paths are identical (#2825).

## Transaction Safety

### Re-entrant transaction() support (v2.52.0)

The `transaction()` function supports re-entrancy: if called while already inside a transaction, it increments a depth counter and runs the function body without starting a new `BEGIN`/`COMMIT` pair. This prevents "cannot start a transaction within a transaction" errors when tool handlers call utility functions that also wrap their work in transactions.

### State machine guards inside transactions (v2.52.0)

Five tool handlers were updated to move state machine guards inside the transaction boundary (#2752). This ensures that the guard check and the subsequent state mutation are atomic -- closing a window where a concurrent process could change state between the guard check and the write.

### Corrupt database recovery

On `openDatabase()`, if `initSchema()` fails with a "malformed" error (corrupt freelist), the engine attempts `VACUUM` recovery before giving up (#2519). On close, a `PRAGMA wal_checkpoint(TRUNCATE)` and `PRAGMA incremental_vacuum(64)` are run as best-effort maintenance.

## Migration from Filesystem State

### Automatic migration

The `workflow-migration.ts` module handles one-time migration from legacy markdown-only projects. `needsAutoMigration()` returns true when the `milestones` table is empty but `.gsd/milestones/` exists on disk. `migrateFromMarkdown()` then:

1. Discovers milestone directories under `.gsd/milestones/`
2. Parses each ROADMAP.md for slices and each PLAN.md for tasks
3. Inserts all data atomically in a single transaction
4. Respects milestone completion: if a SUMMARY.md exists, forces all child slices and tasks to `done`
5. Validates the migration by comparing engine counts against markdown parser counts

### File-based fallbacks (v2.44.0)

DB-dependent code paths include `isDbAvailable()` guards that fall back to filesystem parsing when the database is not open. This enables hybrid operation during the transition period.

### isDbAvailable guard

The `isDbAvailable()` function returns true when a database adapter is open. It is checked before every DB query in `deriveStateFromDb()`, and tool handlers call `ensureDbOpen()` before executing. If the DB is not available, tools return a structured error message rather than crashing.

### Gate auto-mode bootstrap on SQLite (v2.45.0)

Auto-mode bootstrap was gated on SQLite availability (#2419/#2421). If no SQLite provider can be loaded (Node < 22 without `better-sqlite3`), auto-mode refuses to start rather than running in a degraded filesystem-only mode.

## Key Tables

| Table | Primary Key | Purpose |
|-------|-------------|---------|
| `schema_version` | (none, append-only) | Tracks applied schema migrations |
| `decisions` | `id` (TEXT, e.g., D001) | Project decisions with scope, choice, rationale, made_by |
| `requirements` | `id` (TEXT, e.g., R001) | Requirements with class, status, validation, ownership |
| `artifacts` | `path` (TEXT) | Cached artifact content keyed by relative `.gsd/` path |
| `memories` | `id` (TEXT) | Agent memories with category, confidence, hit count |
| `milestones` | `id` (TEXT, e.g., M001) | Milestone status, planning data, verification contracts |
| `slices` | `(milestone_id, id)` | Slice status, dependencies, planning data, sequence |
| `tasks` | `(milestone_id, slice_id, id)` | Task status, planning data, execution results, sequence |
| `verification_evidence` | `id` (INTEGER autoincrement) | Command, exit code, verdict, duration per task verification |
| `replan_history` | `id` (INTEGER autoincrement) | Blocker-triggered replan records per slice |
| `assessments` | `path` (TEXT) | Milestone/slice assessment content and status |
| `quality_gates` | `(milestone_id, slice_id, gate_id, task_id)` | Gate evaluation verdicts (Q3-Q8) |
| `slice_dependencies` | `(milestone_id, slice_id, depends_on_slice_id)` | Junction table for slice dependency lookups |
| `memory_processed_units` | `unit_key` (TEXT) | Tracks which units have been processed for memory extraction |

### Views

- `active_decisions` -- decisions where `superseded_by IS NULL`
- `active_requirements` -- requirements where `superseded_by IS NULL`
- `active_memories` -- memories where `superseded_by IS NULL`

## Data Integrity

### Comprehensive SQLite audit (v2.52.0)

A comprehensive audit addressed indexes, caching, safety, and reconciliation across the entire DB layer. Key fixes included:

- Hot-path dispatch indexes (v13) to eliminate full table scans during auto-loop
- Statement caching in the DB adapter (`stmtCache`) to avoid repeated `prepare()` calls
- Lightweight query variants (`getActiveMilestoneIdFromDb`, `getSliceStatusSummary`, `getActiveTaskIdFromDb`, `getSliceTaskCounts`) that avoid deserializing JSON planning fields on hot paths

### Prevent saveArtifactToDb from overwriting larger files (v2.45.0)

A shrinkage guard in `saveArtifactToDb()` compares the new content size against the existing file. If the new content is less than 50% of the existing file size, the richer disk file is preserved and its content is stored in the DB instead (#2442/#2447). This prevents truncated LLM output from destroying detailed artifacts.

### Delete orphaned verification_evidence on complete-task rollback (v2.52.0)

When a task completion is rolled back (e.g., the task is re-opened after a failed replan), orphaned `verification_evidence` rows are deleted via `deleteVerificationEvidence()` to maintain referential integrity (#2746).

### DB writes before disk writes in validate-milestone (v2.52.0)

The `validate-milestone` handler was updated to write the DB before disk, matching the pattern used by all other tool handlers (#2742). This ensures the DB is the first system updated, and disk writes can be rolled back if they fail.

### Worktree DB reconciliation

The `reconcileWorktreeDb()` function uses `ATTACH DATABASE` to merge a worktree's database back into the main project database after a worktree completes. It handles decisions, requirements, artifacts, milestones, slices, tasks, memories, and verification evidence. Conflicts (same ID, different content) are detected and reported. The merge runs inside a single transaction for atomicity.

### Foreign key enforcement

All tables with parent-child relationships use `FOREIGN KEY` constraints. The `DELETE` operations in `deleteTask()` and `deleteSlice()` cascade manually (evidence -> tasks -> dependencies -> slice) to respect FK ordering. `PRAGMA foreign_keys = ON` is set during schema initialization.
