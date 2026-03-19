# GSD Auto Mode and Dispatch System

## Overview

Auto mode is GSD's autonomous execution engine. It reads `.gsd/` state from disk, determines the next unit of work, creates a fresh agent session with a focused prompt, lets the LLM execute, then reads disk state again and dispatches the next unit. The loop runs until all milestones are complete, a budget ceiling is hit, or the user intervenes.

Entry point: `/gsd auto` (autonomous) or `/gsd next` (step mode -- one unit at a time with pause between each).

All mutable auto-mode state lives in the `AutoSession` class (`auto/session.ts`). The single module-level instance `s` is the only mutable binding. This invariant is enforced by tests.

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
| needs-discussion | `needs-discussion` | stop |
| pre-planning (no context) | `pre-planning`, no CONTEXT.md | stop |
| pre-planning (no research) | `pre-planning`, no RESEARCH.md | `research-milestone` |
| pre-planning (has research) | `pre-planning` | `plan-milestone` |
| planning (no research, not S01) | `planning`, no slice RESEARCH.md | `research-slice` |
| planning | `planning` | `plan-slice` |
| replanning-slice | `replanning-slice` | `replan-slice` |
| executing (missing task plan) | `executing`, no task PLAN file | `plan-slice` (recovery) |
| executing | `executing` | `execute-task` |
| validating-milestone | `validating-milestone` | `validate-milestone` |
| completing-milestone | `completing-milestone` | `complete-milestone` |

If no rule matches, auto mode stops with "Unhandled phase."

### Phase Skip Behavior by Token Profile

| Phase | `budget` | `balanced` | `quality` |
|-------|----------|------------|-----------|
| Milestone Research | Skipped | Runs | Runs |
| Slice Research | Skipped | Skipped | Runs |
| Reassess Roadmap | Opt-in only (`reassess_after_slice`) | Opt-in only | Opt-in only |
| Milestone Validation | Runs | Runs | Runs |

Research skipping is controlled by `prefs.phases.skip_research` and `prefs.phases.skip_slice_research`.

---

## Phase Lifecycle

The typical flow for a single milestone:

```
pre-planning
  |
  +-- research-milestone (if not skipped)
  +-- plan-milestone --> creates ROADMAP.md
  |
planning (per slice)
  |
  +-- research-slice (if not skipped, not S01 with milestone research)
  +-- plan-slice --> creates S##-PLAN.md + task plans
  |
executing (per task)
  |
  +-- execute-task --> marks task done in plan, commits code
  |   +-- [verification gate runs after each execute-task]
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
  |
completing-milestone
  |
  +-- complete-milestone --> writes milestone SUMMARY.md
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

Session creation has a timeout (`NEW_SESSION_TIMEOUT_MS`) to prevent permanent hangs.

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

If the same unit is dispatched twice (the LLM did not produce the expected artifact), GSD applies escalating recovery:

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

### Auto-Fix Retries

When `verification_auto_fix` is true (default) and verification fails:

1. The failure context (failed command stderr, exit codes) is formatted and prepended to the next dispatch prompt.
2. The same `execute-task` unit is re-dispatched with the verification failure context.
3. Up to `verification_max_retries` (default: 2) retry attempts.
4. If retries are exhausted, auto mode pauses.

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
| Permanent | "unauthorized", "invalid key" | Pause indefinitely |

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

## Activity Log

On each unit completion and session shutdown, `saveActivityLog()` persists a record to `.gsd/activity/` for forensic analysis. This data powers `/gsd forensics` for post-mortem investigation.

---

## Key Safety Mechanisms Summary

| Mechanism | Purpose |
|-----------|---------|
| Session lock | Prevent concurrent auto-mode on same project |
| Dispatch reentrancy guard | Prevent concurrent dispatch calls |
| Skip depth guard | Yield to UI after MAX_SKIP_DEPTH skipped units |
| Resource version guard | Stop if extension files changed during session |
| Budget ceiling | Prevent overspending |
| Context pause threshold | Pause before context window overflows |
| Dispatch gap watchdog | Recover from silent dispatch chain breaks |
| Dispatch hang guard | Recover from newSession() hangs |
| EPIPE guard | Clean exit when stdout pipe closes |
| ECOMPROMISED guard | Clean exit when lock mtime drifts |
| SIGTERM handler | Graceful stop on process termination |
