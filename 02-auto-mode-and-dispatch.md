# GSD Auto Mode and Dispatch System

## Overview

Auto mode is GSD's autonomous execution engine. It reads `.gsd/` state from disk, determines the next unit of work, creates a fresh agent session with a focused prompt, lets the LLM execute, then reads disk state again and dispatches the next unit. The loop runs until all milestones are complete, a budget ceiling is hit, or the user intervenes.

Entry point: `/gsd auto` (autonomous) or `/gsd next` (step mode -- one unit at a time with pause between each). The `--yolo` flag (v2.49.0) allows non-interactive project initialization, skipping the guided setup wizard when launching auto mode on an uninitialized project.

All mutable auto-mode state lives in the `AutoSession` class (`auto/session.ts`). The single module-level instance `s` is the only mutable binding. This invariant is enforced by tests.

`startAuto()` has a concurrent invocation guard (v2.58.0) that rejects overlapping calls, preventing subtle state corruption when multiple code paths attempt to start the auto loop simultaneously.

---

## State Machine: deriveState

**File:** `src/resources/extensions/gsd/state.ts`

`deriveState(basePath)` is the source of truth. It reconstructs the full project state by scanning `.gsd/` files on disk. The return type is `GSDState`:

```
GSDState {
  activeMilestone: { id, title } | null
  activeSlice:     { id, title } | null
  activeTask:      { id, title } | null
  phase:           Phase
  registry:        MilestoneRegistryEntry[]
  blockers:        string[]
  progress:        { milestones, slices?, tasks? }
}
```

### Phase Values

| Phase | Meaning |
|-------|---------|
| `pre-planning` | Active milestone has no roadmap yet |
| `needs-discussion` | Milestone has CONTEXT-DRAFT.md but no CONTEXT.md |
| `planning` | Slice needs a plan (no S##-PLAN.md) |
| `evaluating-gates` | Quality gates are being evaluated in parallel (v2.50.0) |
| `executing` | Active task exists in the slice plan |
| `summarizing` | All tasks done but slice not marked complete in roadmap |
| `replanning-slice` | A completed task set `blocker_discovered: true` and no REPLAN.md exists |
| `validating-milestone` | All slices done, no terminal VALIDATION file |
| `completing-milestone` | Validation terminal, but no milestone SUMMARY file |
| `blocked` | No eligible slice (unmet dependencies) or unmet milestone dependencies |
| `complete` | All milestones complete |

### Milestone Scanning Algorithm

1. Find all milestone IDs on disk (sorted).
2. If `GSD_MILESTONE_LOCK` env var is set (parallel worker), filter to that single milestone.
3. Build a roadmap cache: parse each milestone's ROADMAP once, determine completeness.
4. Build registry: iterate milestones in order. The first incomplete, non-parked, non-dep-blocked milestone becomes the active milestone.
5. For the active milestone: find the first incomplete slice whose `depends` are all satisfied.
6. For the active slice: check for a plan file, then find the first incomplete task.

### Caching

`deriveState` results are memoized for 100ms (`CACHE_TTL_MS`). Within a single dispatch cycle, repeated calls return the cached value. Call `invalidateStateCache()` after file mutations.

When the native Rust parser is available, a single batch call reads all `.md` files under `.gsd/` into an in-memory content map, eliminating O(N) individual filesystem reads.

---

## Dispatch Pipeline: resolveDispatch

**File:** `src/resources/extensions/gsd/auto-dispatch.ts`

The dispatch table is a declarative array of rules (`DISPATCH_RULES`). Each rule has a `name` and a `match(ctx)` function. Rules are evaluated in order; the first match wins. The result is one of:

- `{ action: "dispatch", unitType, unitId, prompt }` -- dispatch a unit
- `{ action: "stop", reason }` -- stop auto mode
- `{ action: "skip" }` -- skip and re-derive state

### Dispatch Rules (in evaluation order)

| Rule Name | Trigger Phase | Unit Type |
|-----------|--------------|-----------|
| rewrite-docs (override gate) | Any (active overrides) | `rewrite-docs` |
| summarizing | `summarizing` | `complete-slice` |
| uat-verdict-gate | Any (failed UAT blocks) | stop |
| run-uat (post-completion) | Any (completed slice needs UAT) | `run-uat` |
| reassess-roadmap | Any (opt-in: `reassess_after_slice`) | `reassess-roadmap` |
| needs-discussion | `needs-discussion` | dispatch to interactive flow |
| pre-planning (no context) | `pre-planning`, no CONTEXT.md | stop |
| pre-planning (has context) | `pre-planning` | `plan-milestone` |
| evaluating-gates | `evaluating-gates` | parallel gate evaluation |
| planning | `planning` | `plan-slice` |
| replanning-slice | `replanning-slice` | `replan-slice` |
| executing (missing task plan) | `executing`, no task PLAN file | `plan-slice` (recovery) |
| executing | `executing` | `execute-task` |
| validating-milestone | `validating-milestone` | `validate-milestone` |
| completing-milestone | `completing-milestone` | `complete-milestone` |

If no rule matches, auto mode stops with "Unhandled phase."

**needs-discussion routing (v2.41.0, backported to v2.29.0):** Instead of hard-stopping auto mode, the `needs-discussion` phase now dispatches to `showSmartEntry`, routing the user into an interactive discussion flow. This prevents infinite `/gsd` loops when a milestone has a CONTEXT-DRAFT but no finalized CONTEXT.

**Dispatch guard dependency awareness (v2.41.0):** The dispatch guard uses explicit dependency declarations (`depends_on` in CONTEXT.md / CONTEXT-DRAFT.md) instead of positional ordering to determine milestone eligibility. Completed milestones with a SUMMARY file are always skipped. This allows non-linear milestone execution when dependencies permit.

**Hallucination guard (v2.41.0):** After `execute-task` completes, the dispatch pipeline rejects units that produced zero tool calls, classifying them as hallucinated execution. This prevents the LLM from claiming task completion without actually performing any work.

**Merge anchor verification (v2.41.0):** Before worktree teardown, GSD verifies that the merge target (main branch) is anchored correctly. This prevents data loss when the worktree branch has diverged in ways that would make the squash merge silently discard commits.

### Pipeline Simplification (ADR-003, v2.30.0)

ADR-003 introduced three structural changes to the dispatch pipeline:

1. **Research merged into planning:** Separate `research-milestone` and `research-slice` phases were eliminated. Research is now folded into the planning phase prompts. The dispatch rules for `pre-planning (no research)` and `planning (no research, not S01)` no longer exist; planning units handle research inline.

2. **Mechanical completion:** Slice and milestone completion units use streamlined prompts that focus on artifact generation (summaries, validation verdicts) without re-analyzing code.

3. **Workflow templates:** Right-sized workflows are selected at project init based on task type (e.g., bug fix, feature, refactor). Templates control which phases run and which are skipped, replacing the per-phase skip preferences.

### Quality Gates (v2.50.0)

Planning and completion templates now include an 8-question quality gate evaluation. When a planning or completion unit finishes, the gate questions are evaluated in parallel (the `evaluating-gates` phase). Gate failures block progression and feed structured feedback into a retry prompt.

### Phase Skip Behavior by Token Profile

| Phase | `budget` | `balanced` | `quality` |
|-------|----------|------------|-----------|
| Milestone Research | Merged into planning | Merged into planning | Merged into planning |
| Slice Research | Merged into planning | Merged into planning | Merged into planning |
| Reassess Roadmap | Opt-in only (`reassess_after_slice`) | Opt-in only | Opt-in only |
| Milestone Validation | Runs | Runs | Runs |

### Git Trailers for Metadata (v2.49.0)

GSD moved commit metadata from the subject line (e.g., `[M001/S01]`) to git trailers. Commits now use clean subject lines with structured trailers like `GSD-Milestone: M001` and `GSD-Slice: S01`. This avoids polluting commit history and allows standard git tooling to parse metadata.

### Auto-Commit on Milestone Completion (v2.49.0-v2.57.0)

After `complete-milestone`, GSD auto-commits the milestone summary and related artifacts. This was introduced progressively: first for individual slice phases (v2.44.0), then unified for milestone completion (v2.49.0), and stabilized in v2.57.0.

---

## Phase Lifecycle

The typical flow for a single milestone (post-ADR-003):

```
pre-planning
  |
  +-- plan-milestone --> creates ROADMAP.md (research folded in)
  |   +-- [quality gates evaluate]
  |
planning (per slice)
  |
  +-- plan-slice --> creates S##-PLAN.md + task plans (research folded in)
  |   +-- [quality gates evaluate]
  |
executing (per task)
  |
  +-- execute-task --> marks task done in plan, commits code
  |   +-- [verification gate runs after each execute-task]
  |   +-- [hallucination guard: reject zero-tool-call units]
  |   +-- [post-unit hooks fire if configured]
  |
summarizing (all tasks done)
  |
  +-- complete-slice --> writes summary, marks slice done in roadmap, commits
  |   +-- [run-uat if uat_dispatch enabled]
  |   +-- [reassess-roadmap if reassess_after_slice enabled]
  |
  (next slice... or if all slices done:)
  |
validating-milestone
  |
  +-- validate-milestone --> writes VALIDATION.md with verdict
  |   +-- [verification class compliance check (v2.51.0)]
  |
completing-milestone
  |
  +-- complete-milestone --> writes milestone SUMMARY.md
  |   +-- [auto-commit artifacts]
  |   +-- [merge anchor verification before worktree teardown]
  |
  (next milestone... or "All milestones complete")
```

---

## Fresh Session Per Unit

Every dispatched unit gets a clean agent session via `ctx.newSession()`. The dispatch prompt contains all necessary context pre-inlined:

| Inlined Artifact | Purpose |
|------------------|---------|
| Task plan | What to build (authoritative local execution contract) |
| Slice plan excerpt | Goal, demo, verification criteria |
| Prior task summaries | What previous tasks in this slice accomplished |
| Continue file | Resume state from interrupted work |
| Active overrides | Steering instructions injected mid-execution |
| Decisions register | Architectural context (from DB) |
| Knowledge base | Cross-session project-specific rules |
| Agent instructions | User-provided always-follow instructions |

The system prompt is assembled in `before_agent_start` from the GSD system context, preferences, knowledge, memories, worktree context, and agent instructions.

Session creation has a timeout (`NEW_SESSION_TIMEOUT_MS`, 120s as of v2.67.0, increased from the previous value) to prevent permanent hangs. The timeout is now treated as a recoverable pause rather than a hard failure -- auto mode pauses and can be resumed, instead of stopping entirely.

---

## Crash Recovery

### Lock File (`auto.lock`)

**File:** `src/resources/extensions/gsd/crash-recovery.ts`

A lock file at `.gsd/auto.lock` tracks the current unit. It contains:

```json
{
  "pid": 12345,
  "startedAt": "...",
  "unitType": "execute-task",
  "unitId": "M001/S01/T03",
  "completedUnits": 7,
  "sessionFile": "/path/to/session"
}
```

If the process dies, the next `/gsd auto` reads the surviving session file, calls `synthesizeCrashRecovery()` to extract a recovery briefing from every tool call that made it to disk, and prepends this context to the next dispatch prompt.

### Session Forensics

`synthesizeCrashRecovery()` scans the crashed session's persisted entries for tool calls, extracting a timeline of what the agent accomplished before the crash. This briefing is capped to 50,000 characters and injected as a preamble to the resumed unit's prompt.

For retry scenarios (unit dispatched twice), `getDeepDiagnostic()` provides a diagnostic from the previous attempt.

### Headless Auto-Restart (v2.26)

When running `gsd headless auto`, crashes trigger automatic restart with exponential backoff (5s, 10s, 30s cap, default 3 attempts). SIGINT/SIGTERM bypasses restart. Configure with `--max-restarts N`.

### Signal Handlers and Lock Cleanup (v2.41.0)

SIGHUP and SIGINT handlers are registered to clean up lock files on crash. This prevents stale locks from blocking subsequent auto-mode launches. The handlers complement the existing `process.once("exit")` cleanup.

### Self-Heal Runtime Records (v2.41.0)

Before entering the auto loop, `selfHealRuntimeRecords()` clears orphaned "dispatched" records that were never completed (e.g., from a crash mid-unit). This prevents the dispatch pipeline from misinterpreting stale records as in-progress work.

### Closeout on Pause, Heal on Resume (v2.41.0)

When auto mode pauses, the current unit is properly closed out (artifacts flushed, metrics recorded). On resume, runtime records are healed before re-entering the loop, ensuring clean state.

### Infrastructure Error Halt (v2.41.0)

Auto mode stops immediately on infrastructure errors (`ENOSPC`, `ENOMEM`, `EAGAIN`) instead of burning budget on retries that cannot succeed. These are tagged as `infraError` and bypass the normal retry/escalation path.

### DB Reconciliation for Stale Task Status (v2.50.0)

Before state derivation, GSD reconciles stale task status between the filesystem and the SQLite database. Tasks marked as "dispatched" in the DB but completed on disk are corrected, and vice versa. This prevents ghost tasks from blocking slice progression after crashes.

### Stale Lockfile Recovery (v2.66.0)

Lock files now include a `createdAt` timestamp. When GSD encounters a lock file from a dead process, a 30-second age guard prevents premature cleanup of locks that were just created by a concurrent launch. The staleness check uses both PID liveness (`process.kill(pid, 0)`) and the age guard before removing a lock, reducing the risk of two processes both deciding the lock is stale and racing to acquire it.

### Orphaned Milestone Branch Audit (v2.66.1)

At auto-mode bootstrap, GSD audits for orphaned milestone branches -- branches that exist in the git repo but have no corresponding milestone on disk or in the database. Orphaned branches are logged as warnings to surface leftover state from interrupted milestone deletions or manual git operations.

---

## Session Lock (OS-Level Exclusion)

**File:** `src/resources/extensions/gsd/session-lock.ts`

Prevents multiple GSD processes from running auto mode on the same project simultaneously. Uses `proper-lockfile` for OS-level file locking (flock). Falls back to PID-based checking if the library is unavailable.

### Lifecycle

| Function | When Called |
|----------|------------|
| `acquireSessionLock()` | Start of `bootstrapAutoSession` |
| `validateSessionLock()` | Each dispatch cycle (detects takeover) |
| `updateSessionLock()` | Each unit dispatch (updates metadata) |
| `releaseSessionLock()` | Clean stop/pause |

### Lock Compromise Detection

`proper-lockfile` fires `onCompromised` when it detects mtime drift (system sleep, heavy event loop stalls). Instead of crashing, GSD sets a flag. On the next dispatch cycle, `validateSessionLock()` returns false, and auto mode stops gracefully.

### Stale Lock Cleanup

On acquisition, GSD cleans up:
- Numbered lock file variants from cloud sync conflicts (e.g., "auto 2.lock")
- Stray `proper-lockfile` directories (`.gsd.lock/`)
- Lock files from dead processes (PID check via `process.kill(pid, 0)`)

A `process.once("exit")` handler ensures locks are cleaned up on normal exit.

---

## Stuck Detection

**File:** `src/resources/extensions/gsd/auto-stuck-detection.ts`

### Sliding-Window Detection (v2.39.0)

The original stuck counter was replaced with a sliding-window approach. Instead of a simple dispatch count per unit, GSD tracks a rolling window of recent dispatch outcomes. This reduces false positives from legitimate retries (e.g., verification fix cycles) while still catching genuine infinite loops.

If the same unit is dispatched repeatedly without progress, GSD applies escalating recovery:

1. **First retry**: Dispatches with a deep diagnostic prompt (what went wrong in the previous attempt).
2. **Second retry**: If still stuck, auto mode stops with the exact file path it expected.

### Idempotency Checking

**File:** `src/resources/extensions/gsd/auto-idempotency.ts`

Before dispatching, the idempotency check verifies whether a unit's expected artifact already exists on disk (completed by a previous session). If so, the unit is skipped without dispatching. This prevents re-executing already-completed work after crashes or resumes.

Constants governing stuck detection:

| Constant | Value | Purpose |
|----------|-------|---------|
| `MAX_UNIT_DISPATCHES` | (from session.ts) | Max dispatches of same unit before stopping |
| `MAX_LIFETIME_DISPATCHES` | (from session.ts) | Absolute cap across all skip cycles |
| `MAX_CONSECUTIVE_SKIPS` | (from session.ts) | Max completed-unit skips before yielding to UI |
| `MAX_SKIP_DEPTH` | (from session.ts) | Recursion depth guard for skip chains |

---

## Timeout Supervision

**File:** `src/resources/extensions/gsd/auto-timers.ts`

Three timeout tiers prevent runaway sessions:

| Timeout | Default | Behavior |
|---------|---------|----------|
| Soft | 20 min | Sends a wrap-up message to the LLM ("finish durable output") |
| Idle | 10 min | Detects stalls (no tool calls); intervenes with a nudge |
| Hard | 30 min | Pauses auto mode |

Configured via preferences:

```yaml
auto_supervisor:
  soft_timeout_minutes: 20
  idle_timeout_minutes: 10
  hard_timeout_minutes: 30
```

### Context Pressure Monitor (v2.26)

At 70% context usage, GSD sends a wrap-up signal nudging the agent to finish durable output (commit, write summaries) before the context window fills.

### Dispatch Gap Watchdog

After `handleAgentEnd` completes, a watchdog fires if no new unit is dispatched within `DISPATCH_GAP_TIMEOUT_MS`. This catches silent dispatch chain breaks (unhandled exceptions in `dispatchNextUnit`).

### Dispatch Hang Guard

A pre-dispatch watchdog runs before `dispatchNextUnit`. If dispatch itself hangs (newSession deadlock, model selection timeout), this guard starts the gap watchdog for recovery.

---

## Cost Tracking

**File:** `src/resources/extensions/gsd/metrics.ts`

Every unit's token usage and cost is captured in a `MetricsLedger` persisted to `.gsd/metrics.json`.

### Data Flow

1. Before `newSession()` wipes context, `snapshotUnitMetrics()` scans session entries for `AssistantMessage` usage data.
2. The unit record (type, id, model, tokens, cost, tool calls, duration) is appended to the in-memory ledger and flushed to disk.
3. The dashboard and progress widget read from the in-memory ledger.

### Token Breakdown

Each `UnitMetrics` record contains:
- `tokens.input` / `tokens.output` / `tokens.cacheRead` / `tokens.cacheWrite` / `tokens.total`
- `cost` (USD)
- `toolCalls`, `assistantMessages`, `userMessages`, `apiRequests`
- `tier` (light/standard/heavy if dynamic routing is active)
- `cacheHitRate` (percentage 0-100)
- `promptCharCount` / `baselineCharCount` (prompt measurement)

### Aggregation

Metrics can be aggregated by:
- **Phase**: research, planning, execution, completion, reassessment
- **Slice**: per milestone/slice pair
- **Model**: per model ID
- **Tier**: per complexity tier (light/standard/heavy)

### Budget Ceiling

When `budget_ceiling` is set in preferences, GSD checks total spend before each dispatch:

| Threshold | Action |
|-----------|--------|
| 75% | Info notification |
| 80% | Warning notification |
| 90% | Warning notification + desktop notification |
| 100% | Enforced per `budget_enforcement` (warn/pause/halt) |

Budget enforcement modes:
- `warn` -- log a warning, continue
- `pause` (default) -- pause auto mode, `/gsd auto` to override
- `halt` -- stop auto mode entirely

### Cost Projection

`formatCostProjection()` estimates remaining cost based on completed slice averages. Requires 2+ completed slice-level entries for a reliable projection.

---

## Verification Enforcement

**File:** `src/resources/extensions/gsd/verification-gate.ts`

After every `execute-task` unit, the verification gate discovers and runs shell commands.

### Command Discovery (first non-empty source wins)

1. `verification_commands` from preferences (blocking)
2. Task plan `verify` field, split on `&&` (blocking)
3. `package.json` scripts: `typecheck`, `lint`, `test` (advisory, non-blocking)
4. None found -- gate passes

### Execution

- Commands run sequentially via `spawnSync` with a 2-minute default timeout.
- stdout/stderr truncated to 10 KB per command.
- Infrastructure errors (ETIMEDOUT, ENOENT, ENOMEM) are tagged as `infraError` and do not trigger auto-fix retries.
- Gate passes when all blocking checks exit 0.

### Wider Operational Regex (v2.58.0)

The verification gate regex that identifies operational commands was broadened to match additional patterns, reducing false negatives where legitimate verification commands were not recognized.

### Pre-Execution Plan Verification (v2.65.0)

Before dispatching an `execute-task` unit, GSD runs pre-execution plan verification checks. These checks validate that the task plan's expected inputs exist on disk, that referenced files have not been deleted by a prior task, and that the task's declared dependencies are satisfied. Pre-execution failures block dispatch and surface a diagnostic explaining what is missing.

### Post-Execution Cross-Task Consistency Checks (v2.65.0)

After a task completes, GSD runs cross-task consistency checks that compare the task's outputs against expectations declared by downstream tasks in the same slice. If a completed task was supposed to produce a file that a later task depends on and that file is missing, the inconsistency is flagged.

### Blocking Behavior and Strict Mode (v2.65.0)

Enhanced verification supports a `strict` mode where both pre-execution and post-execution check failures are blocking -- auto mode pauses until the issue is resolved. In non-strict mode (default), post-execution consistency warnings are logged but do not block dispatch of the next task.

```yaml
enhanced_verification:
  strict: false
```

### Depth Verification Answer Validation (v2.66.0)

Before unlocking the write-gate after a depth verification prompt, GSD validates the LLM's answer against the expected verification criteria. Shallow or evasive answers are rejected and the write-gate remains locked, forcing the LLM to provide substantive evidence before proceeding.

### Auto-Fix Retries

When `verification_auto_fix` is true (default) and verification fails:

1. The failure context (failed command stderr, exit codes) is formatted and prepended to the next dispatch prompt.
2. The same `execute-task` unit is re-dispatched with the verification failure context.
3. Up to `verification_max_retries` (default: 2) retry attempts.
4. If retries are exhausted, auto mode pauses.

### Non-Blocking Gate for Auto-Discovered Commands (v2.29.0)

When verification commands are auto-discovered from `package.json` scripts (source 3), failures are treated as advisory rather than blocking. This prevents auto-mode from stalling on pre-existing lint warnings or flaky tests that the current task did not introduce.

### Verification Classes and Milestone Validation (v2.51.0)

Verification classes (e.g., `unit-test`, `integration-test`, `type-check`, `lint`) are injected into milestone validation prompts. Before milestone completion, GSD checks verification class compliance -- each declared class must have passing evidence from the slice verification runs. This ensures milestones cannot complete without satisfying the project's declared verification contract.

### followUps and knownLimitations Extraction (v2.51.0)

`parseSummary()` now extracts `followUps` and `knownLimitations` sections from slice and milestone summaries. These are surfaced in the milestone validation prompt, giving the validator awareness of acknowledged gaps and deferred work.

### Reactive Batch Verification + Dependency-Based Carry-Forward (v2.38.0)

ADR-004 introduced reactive batch verification. Instead of running all verification commands after every task, GSD tracks which files each task modified and only runs verification commands whose input files were affected. Verification results carry forward across tasks based on dependency declarations -- if Task 3's verification passed and Task 4 did not touch any of Task 3's files, Task 3's results are carried forward without re-running.

### Runtime Error Capture

`captureRuntimeErrors()` scans background shell processes and browser console logs for runtime errors:

| Source | Severity | Blocking |
|--------|----------|----------|
| bg-shell crashed | crash | Yes |
| bg-shell non-zero exit | crash | Yes |
| bg-shell SIGABRT/SIGSEGV/SIGBUS | crash | Yes |
| Browser "Unhandled" error | crash | Yes |
| Browser general error | error | No |
| bg-shell recent errors (alive) | error | No |
| Browser deprecation warning | warning | No |

### Dependency Audit

`runDependencyAudit()` checks if dependency files changed (package.json, lockfiles), then runs `npm audit --json` and returns structured vulnerability warnings.

---

## Provider Error Recovery

GSD classifies provider errors and auto-resumes when safe:

| Error Type | Examples | Action |
|-----------|----------|--------|
| Rate limit | 429, "too many requests" | Auto-resume after retry-after header or 60s |
| Server error | 500, 502, 503, "overloaded" | Auto-resume after 30s |
| Transient | Stream truncation, terminated/connection errors | Auto-resume with backoff |
| Permanent | "unauthorized", "invalid key" | Pause indefinitely |

### Transient Error Classification (v2.29.0-v2.51.0)

Auto-resume now covers transient server errors beyond rate limits (v2.29.0). The error classifier recognizes:
- **Stream truncation:** JSON parse errors from prematurely closed streams (v2.51.0)
- **Terminated/connection errors:** Network disconnects and connection resets (v2.45.0)

These are classified as transient and trigger auto-resume with backoff rather than permanent pause.

### Rate-Limit Model Fallback (v2.53.0)

When a rate-limit error occurs, GSD first attempts model fallback before pausing. If the current model's fallback chain has alternatives available, the next model is tried immediately. Only when no fallbacks remain does the standard rate-limit pause engage.

### Escalating Backoff

Each consecutive transient auto-resume doubles the delay. After 5 consecutive failures (`MAX_TRANSIENT_AUTO_RESUMES`), auto mode pauses indefinitely.

### Model Fallback Chain

On error, GSD tries:
1. Retry the current model (up to 2 attempts for transient network errors, with 3s/6s backoff).
2. Switch to the next model in the phase's `fallbacks` list.
3. Restore the model captured at auto-mode start (`autoModeStartModel`).
4. If all fail, classify the error and pause/auto-resume based on classification.

---

## Git Isolation

### Worktree Mode (default)

Each milestone runs in its own git worktree at `.gsd/worktrees/<MID>/` on a `milestone/<MID>` branch. All slice work commits sequentially on this branch. When the milestone completes, it is squash-merged to main.

### Branch Mode

Work happens in the project root on a `milestone/<MID>` branch. Useful for submodule-heavy repos where worktrees have issues.

### None Mode

Work happens directly on the current branch. No worktree, no milestone branch. Ideal for hot-reload workflows where file isolation breaks dev tooling.

### Milestone Merge

On milestone completion or transition, `tryMergeMilestone()` handles both worktree and branch isolation modes. For worktrees, it calls `mergeMilestoneToMain()` which squash-merges the worktree branch. For branch mode, it merges on the current repo.

---

## Parallel Execution

**File:** `src/resources/extensions/gsd/parallel-orchestrator.ts`

When the project has independent milestones, they can run simultaneously. Each milestone gets its own worker process and worktree.

Workers are separate child processes spawned via `child_process`, each running with `GSD_MILESTONE_LOCK` env var set. The coordinator monitors workers via session status files.

Configuration:

```yaml
parallel:
  enabled: false
  max_workers: 2        # 1-4
  budget_ceiling: 50.00
  merge_strategy: "per-milestone"
  auto_merge: "confirm"
```

### Worker Model Override (v2.66.0)

Parallel milestone workers can use a different model than the coordinator. The `parallel` preferences accept a model override so that workers run on a cheaper or faster model while the coordinator retains the primary model for orchestration decisions.

### Slice-Level Parallelism (v2.64.0)

Beyond milestone-level parallelism, GSD supports dependency-aware parallel dispatch within a single milestone. Each eligible slice whose `depends` declarations are fully satisfied can run in its own parallel worker subprocess.

Configuration:

```yaml
slice_parallel:
  enabled: false
  max_workers: 2
```

The environment variable `GSD_PARALLEL_WORKER` is set in each worker process to prevent recursive dispatch -- a worker will never spawn additional parallel workers. Slice-level parallelism is orthogonal to milestone-level parallelism; both can be active simultaneously when independent slices exist within independent milestones.

### Parallel Research Slices and Milestone Validation (v2.66.0)

Research slices within a milestone can now be dispatched in parallel when they have no mutual dependencies. Milestone validation also runs in parallel across completed milestones when multiple milestones finish in the same dispatch cycle.

---

## LLM Safety Harness (v2.64.0)

**File:** `src/resources/extensions/gsd/llm-safety-harness.ts`

The safety harness prevents the LLM from running destructive operations or querying `gsd.db` directly via bash during auto-mode execution.

### Destructive Operation Blocking

Bash commands issued by the LLM are screened against a deny list of destructive patterns. Commands that would modify critical GSD state files, drop database tables, or perform irreversible filesystem operations are rejected before execution.

### Direct DB Query Prevention (v2.64.0)

The LLM is blocked from querying `gsd.db` directly through bash (e.g., `sqlite3 .gsd/gsd.db`). All database access must go through the structured tool interface, which enforces schema constraints and audit logging.

### Checkpoint and Rollback

Before each code-executing unit, GSD creates a git ref checkpoint. If the unit produces an error and `auto_rollback` is enabled in preferences, GSD rolls back the working tree to the checkpoint ref, undoing any partial changes from the failed unit. This prevents half-completed code modifications from accumulating across retries.

### Evidence Collection

During execution, the harness collects evidence of what the LLM actually did -- tool calls, file modifications, and command outputs. This evidence is persisted alongside the unit record and used by the verification gate and post-mortem forensics.

---

## Context Engineering (v2.67.0)

### Tiered Context Injection (M005)

GSD injects context into dispatch prompts using a tiered relevance model. Instead of loading the full project context for every unit, M005 scores each context artifact (decisions, knowledge base entries, prior summaries) against the current unit's scope and only injects artifacts above a relevance threshold. This produces a 65%+ reduction in per-turn token cost without sacrificing the LLM's awareness of relevant project history.

### Decision Scope Cascade (R005)

R005 derives the scope of each decision record from slice metadata. When a decision is recorded during slice S03 of milestone M002, the scope is tagged accordingly. During dispatch, decisions are filtered by scope cascade: task-level decisions are injected only for that task's unit, slice-level for the slice's units, and milestone-level for all units within the milestone. This prevents irrelevant architectural decisions from consuming context budget in unrelated slices.

---

## Discussion Gate Enforcement (v2.67.0)

Milestone discussion uses a 4-layer gated question flow with mechanical enforcement. The four gates are:

| Gate | Purpose |
|------|---------|
| Scope | Define what the milestone covers and excludes |
| Architecture | Establish technical approach and key design decisions |
| Error | Identify failure modes and error handling strategy |
| Quality | Set acceptance criteria and verification requirements |

### Fail-Closed Behavior

Discussion gate enforcement is fail-closed: if the gate confirmation is missing or ambiguous, all execution tools are blocked. The LLM cannot proceed to planning or execution until the gate is explicitly confirmed. This prevents the LLM from skipping discussion questions and jumping straight to code generation.

### Fast Path for Queued Milestone Discussion (v2.66.0)

When a queued milestone already has a CONTEXT-DRAFT from a prior session, the discussion gate offers a fast path that skips already-answered questions and resumes from the first unanswered gate layer.

---

## Prompt Loader

**File:** `src/resources/extensions/gsd/prompt-loader.ts`

Templates live at `prompts/` and `templates/` relative to the extension directory. They use `{{variableName}}` syntax for substitution.

All templates are eagerly cached at module load time via `warmCache()`. This prevents a running session from being invalidated when another GSD launch overwrites template files on disk. The in-memory snapshot is immune to later overwrites.

Functions:
- `loadPrompt(name, vars)` -- load and substitute a prompt template
- `loadTemplate(name)` -- load a raw template from `templates/`
- `inlineTemplate(name, label)` -- load template with a labeled footer for inlining

---

## Dynamic Model Routing

**File:** `src/resources/extensions/gsd/model-router.ts`

When enabled, auto mode selects models based on task complexity:

| Tier | Example Models | Use Case |
|------|---------------|----------|
| Light | claude-haiku-4-5, gpt-4o-mini, gemini-2.0-flash | Simple tasks, slice completion, UAT |
| Standard | claude-sonnet-4-6, gpt-4o, gemini-2.5-pro | Normal tasks |
| Heavy | claude-opus-4-6, o1, o3 | Complex planning, architectural tasks |

**Downgrade-only semantics**: the returned model is always equal to or cheaper than the user's configured primary model. Never upgrades beyond configuration.

Escalation: if a downgraded task fails, `escalateTier()` promotes it: light -> standard -> heavy -> null (no further escalation).

---

## State Machine Hardening (v2.66.0)

A 5-wave systematic fix addressed state machine resilience across the entire GSD engine. Each wave targeted a specific failure category:

| Wave | Focus | Examples |
|------|-------|---------|
| Wave 1 | Critical data integrity | Prevent silent data loss in milestone/slice status transitions |
| Wave 2 | Event log reconciliation | Detect concurrent event log growth during reconcile, fix stale entries |
| Wave 3 | Session and recovery robustness | Harden cold resume, session restore, and crash recovery paths |
| Wave 4 | Atomic writes and randomized tmp paths | Prevent partial writes from corrupting state files; randomize temp file names to avoid collisions |
| Wave 5 | Consistency and cleanup | Final sweep for edge cases, orphaned state, and silent failures |

86+ regression tests were added alongside these fixes to prevent regressions. The test suite covers adversarial scenarios (concurrent writes, partial failures, interrupted operations) identified through structured adversarial review of each wave.

---

## Activity Log

On each unit completion and session shutdown, `saveActivityLog()` persists a record to `.gsd/activity/` for forensic analysis. This data powers `/gsd forensics` for post-mortem investigation.

---

## Key Safety Mechanisms Summary

| Mechanism | Purpose |
|-----------|---------|
| Session lock | Prevent concurrent auto-mode on same project |
| Concurrent startAuto() guard (v2.58.0) | Reject overlapping startAuto() calls |
| Dispatch reentrancy guard | Prevent concurrent dispatch calls |
| Skip depth guard | Yield to UI after MAX_SKIP_DEPTH skipped units |
| Resource version guard | Stop if extension files changed during session |
| Budget ceiling | Prevent overspending |
| Context pause threshold | Pause before context window overflows |
| Dispatch gap watchdog | Recover from silent dispatch chain breaks |
| Dispatch hang guard | Recover from newSession() hangs |
| Hallucination guard (v2.41.0) | Reject execute-task with zero tool calls |
| Merge anchor verification (v2.41.0) | Verify merge target before worktree teardown |
| Infrastructure error halt (v2.41.0) | Stop immediately on ENOSPC/ENOMEM/EAGAIN |
| Quality gates (v2.50.0) | 8-question evaluation at planning and completion |
| Verification class compliance (v2.51.0) | Block milestone completion without passing evidence |
| LLM safety harness (v2.64.0) | Block destructive operations and direct DB queries |
| Checkpoint/rollback (v2.64.0) | Git ref checkpoint before code units; rollback on error |
| Pre-execution plan verification (v2.65.0) | Validate task inputs and dependencies before dispatch |
| Post-execution consistency checks (v2.65.0) | Cross-task output validation after execution |
| Depth verification answer validation (v2.66.0) | Reject shallow answers before write-gate unlock |
| Discussion gate enforcement (v2.67.0) | Fail-closed 4-layer gated discussion before execution |
| State machine hardening (v2.66.0) | 5-wave fix with 86+ regression tests |
| Stale lockfile age guard (v2.66.0) | 30s age guard prevents premature lock cleanup |
| Sliding-window stuck detection (v2.39.0) | Catch infinite loops without false positives |
| SIGHUP/SIGINT lock cleanup (v2.41.0) | Clean lock files on crash signals |
| EPIPE guard | Clean exit when stdout pipe closes |
| ECOMPROMISED guard | Clean exit when lock mtime drifts |
| SIGTERM handler | Graceful stop on process termination |
