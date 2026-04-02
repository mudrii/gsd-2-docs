# Parallel Milestone Orchestration

Run multiple milestones simultaneously in isolated git worktrees. Each milestone gets its own worker process, its own branch, and its own context window — while a coordinator tracks progress, enforces budgets, and keeps everything in sync.

Added in **v2.24.0**. Disabled by default — opt-in only, zero impact to existing users.

---

## Quick Start

1. Enable parallel mode in your preferences:

```yaml
---
parallel:
  enabled: true
  max_workers: 2
---
```

2. Start parallel execution:

```
/gsd parallel start
```

GSD scans your milestones, checks dependencies and file overlap, shows an eligibility report, and spawns workers for eligible milestones.

3. Monitor progress:

```
/gsd parallel status
```

4. Stop when done:

```
/gsd parallel stop
```

---

## What Parallel Orchestration Is

In standard auto mode, milestones run sequentially: M001 must complete before M002 begins. Parallel orchestration breaks this constraint for milestones that don't depend on each other. Each eligible milestone gets:

- A separate `gsd` worker process
- Its own git worktree at `.gsd/worktrees/<MID>/` on branch `milestone/<MID>`
- Its own context windows (completely isolated agent sessions)
- Its own `.gsd/` state files (lock, metrics, completed-units)

A coordinator in your primary GSD session manages all workers, tracks aggregate cost, and handles merge-back when milestones complete.

The feature is designed with a strict safety model: it is disabled by default, all operations require explicit opt-in, workers cannot spawn nested parallel sessions (`GSD_PARALLEL_WORKER` env guard), and the coordinator always stays in the project root.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Coordinator (your GSD session, project root)                   │
│                                                                 │
│  Responsibilities:                                              │
│  - Eligibility analysis (deps + file overlap)                   │
│  - Worker spawning and lifecycle management                     │
│  - Budget tracking across all workers                           │
│  - Signal dispatch (pause/resume/stop)                          │
│  - Session status monitoring (heartbeat reading)                │
│  - Merge reconciliation                                         │
│  - Orchestrator state persistence (.gsd/orchestrator.json)      │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Worker 1   │  │   Worker 2   │  │   Worker 3   │  ...       │
│  │    M002      │  │    M003      │  │    M005      │           │
│  │  gsd process │  │  gsd process │  │  gsd process │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                 │                 │                   │
│         ▼                 ▼                 ▼                   │
│  .gsd/worktrees/   .gsd/worktrees/   .gsd/worktrees/            │
│  M002/             M003/             M005/                      │
│  (milestone/M002)  (milestone/M003)  (milestone/M005)           │
│                                                                 │
│  IPC via files:                                                 │
│  .gsd/parallel/M002.status.json  (worker → coordinator)         │
│  .gsd/parallel/M002.signal.json  (coordinator → worker)         │
└─────────────────────────────────────────────────────────────────┘
```

### Worker Spawning

Each worker is spawned as a Node.js subprocess running the GSD CLI in JSON output mode:

```
node <GSD_BIN_PATH> --mode json --print "/gsd auto"
```

The worker process runs in the worktree directory (`cwd: worker.worktreePath`) with two critical environment variables set:

| Variable | Value | Purpose |
|----------|-------|---------|
| `GSD_MILESTONE_LOCK` | `<MID>` | Restricts `deriveState()` to only see the assigned milestone |
| `GSD_PARALLEL_WORKER` | `"1"` | Prevents nested parallel session spawning |

The `GSD_MILESTONE_LOCK` variable is checked in `deriveState()` before processing the milestone list. When set, all other milestones are filtered out — the worker behaves as if it is the only GSD session in the repository.

---

## Eligibility Analysis

Before starting parallel execution, GSD runs `analyzeParallelEligibility()` which evaluates every milestone against three rules:

### Rule 1: Not Complete or Parked

Milestones with a SUMMARY.md (complete) or PARKED.md marker are excluded. Parked milestones are registered as `parked` in the registry and skipped.

### Rule 2: Dependencies Satisfied

Every entry in a milestone's `dependsOn` list (parsed from CONTEXT.md frontmatter) must have status `complete` in the registry. If any dependency is not complete, the milestone is ineligible with a `"Blocked by incomplete dependencies: <IDs>"` reason.

### Rule 3: File Overlap (Warning Only)

For all eligible milestones, the orchestrator collects the `filesLikelyTouched` arrays from every slice plan. If two eligible milestones have overlapping files, both receive a warning annotation — but neither is excluded. The worktree isolation means they won't interfere at the filesystem level; conflicts are detected and resolved at merge time.

### Eligibility Report

```
# Parallel Eligibility Report

## Eligible for Parallel Execution (2)

- **M002** — Auth System
  All dependencies satisfied.
- **M003** — Dashboard UI
  All dependencies satisfied. WARNING: has file overlap with another eligible milestone.

## Ineligible (2)

- **M001** — Core Types
  Already complete.
- **M004** — API Integration
  Blocked by incomplete dependencies: M002.

## File Overlap Warnings (1)

- **M002** <-> **M003** — 2 shared file(s):
  - `src/types.ts`
  - `src/middleware.ts`
```

---

## Worker Isolation

Each worker is completely isolated from every other worker and from the coordinator:

| Resource | Isolation Method |
|----------|-----------------|
| **Filesystem** | Git worktree — each worker has its own checkout at `.gsd/worktrees/<MID>/` |
| **Git branch** | `milestone/<MID>` — one branch per milestone |
| **Milestone visibility** | `GSD_MILESTONE_LOCK` — `deriveState()` only sees the assigned milestone |
| **Context window** | Separate process — each worker has its own agent sessions |
| **Metrics** | Each worktree has its own `.gsd/metrics.json` |
| **Crash recovery** | Each worktree has its own `.gsd/auto.lock` |
| **Nested parallel** | `GSD_PARALLEL_WORKER=1` — workers cannot spawn child parallel sessions |

### Worktree Creation

When a worker starts, the coordinator creates a git worktree:

```
createWorktree(basePath, milestoneId, {
  branch: "milestone/<MID>",
  startPoint: <integrationBranch>   // or reuseExistingBranch: true if branch exists
})
```

The worktree is created at `.gsd/worktrees/<MID>/`. Planning artifacts (DECISIONS.md, REQUIREMENTS.md, PROJECT.md, KNOWLEDGE.md, and all milestone directories) are copied from the project root into the new worktree so the worker starts with full context.

If the milestone branch already exists (from a prior session), the worktree re-attaches to it without resetting. Plan checkbox state is forward-merged from the project root to ensure the worktree sees any `[x]` checkboxes written before the crash.

### Post-Create Hook

A user-configured hook script runs after every worktree creation:

```yaml
git:
  worktree_post_create: ./scripts/setup-worktree.sh
```

The script receives `SOURCE_DIR` and `WORKTREE_DIR` as environment variables. Use this to copy `.env` files, create symlinks, or run `npm install` in the new worktree. Hook failures are non-fatal (logged, but execution continues).

---

## NDJSON Monitoring Protocol

Workers run with `--mode json`, emitting one JSON event per line on stdout. The coordinator reads this stream in real time to track cost and progress.

The primary event of interest is `message_end`:

```json
{
  "type": "message_end",
  "message": {
    "role": "assistant",
    "usage": {
      "cost": {
        "total": 0.0043
      }
    }
  }
}
```

When a `message_end` event arrives:

1. `worker.cost += event.message.usage.cost.total`
2. Aggregate cost is recalculated across all workers.
3. If `msg.role === "assistant"`, `worker.completedUnits` is incremented.
4. The session status file is updated on disk (`.gsd/parallel/<MID>.status.json`).

`extension_ui_request` events (GSD notifications) also trigger session status updates, keeping the dashboard heartbeat current.

Non-JSON lines (stderr leakage, debug output) are silently discarded.

---

## File-Based IPC

Workers and the coordinator communicate through two file types per milestone:

### Status Files: `.gsd/parallel/<MID>.status.json`

Written by workers (and by the coordinator on their behalf at spawn time). Read by the coordinator during `refreshWorkerStatuses()`. Contains:

```json
{
  "milestoneId": "M002",
  "pid": 12345,
  "state": "running",
  "currentUnit": null,
  "completedUnits": 7,
  "cost": 0.087,
  "lastHeartbeat": 1700000000000,
  "startedAt": 1699990000000,
  "worktreePath": "/path/to/project/.gsd/worktrees/M002"
}
```

All status file writes use atomic write (write-to-temp + rename) to prevent partial reads.

### Signal Files: `.gsd/parallel/<MID>.signal.json`

Written by the coordinator. Workers check for signals between units (in `handleAgentEnd`).

```json
{
  "signal": "pause",
  "sentAt": 1700000000000
}
```

Valid signals: `"pause"`, `"resume"`, `"stop"`.

### Signal Lifecycle

```
Coordinator                      Worker
    │                                │
    ├── sendSignal("pause") ──────→  │
    │                                ├── consumeSignal() (between units)
    │                                ├── pauseAuto()
    │                                │   (finishes current unit, then waits)
    │                                │
    ├── sendSignal("resume") ──────→ │
    │                                ├── consumeSignal()
    │                                └── resumes dispatch loop
    │
    ├── sendSignal("stop") ────────→ │
    │   + SIGTERM ──────────────────→│
    │                                ├── consumeSignal() or SIGTERM handler
    │                                └── stopAuto() → process exits
```

On stop, the coordinator sends both a file signal and SIGTERM (for immediate response). If the worker doesn't exit within 750ms, SIGKILL is sent.

---

## Budget Enforcement

### Per-Worker Budget

Each worker independently respects the project-level `budget_ceiling` preference, which can pause the worker if its own cost exceeds the ceiling.

### Aggregate Budget

The coordinator enforces a parallel-specific aggregate budget:

```yaml
parallel:
  budget_ceiling: 50.00    # USD across all workers combined
```

The coordinator checks `isBudgetExceeded()` before spawning each new worker. When the ceiling is reached:

1. No new units are dispatched across any worker.
2. Workers receive stop signals.
3. The dashboard shows "Budget ceiling reached."

### 80% Alert

When aggregate cost reaches 80% of the budget ceiling, the dashboard shows a budget alert. This gives you time to react before the ceiling is hit.

---

## PID Tracking and Crash Recovery

### Orchestrator State Persistence

The coordinator persists its full state to `.gsd/orchestrator.json` after every meaningful change:

```json
{
  "active": true,
  "workers": [
    {
      "milestoneId": "M002",
      "title": "Auth System",
      "pid": 12345,
      "worktreePath": "...",
      "startedAt": 1699990000000,
      "state": "running",
      "completedUnits": 7,
      "cost": 0.087
    }
  ],
  "totalCost": 0.174,
  "startedAt": 1699990000000,
  "configSnapshot": { "max_workers": 2, "budget_ceiling": 50.00 }
}
```

All writes use atomic write (write-to-temp + rename).

### Orphan Detection on Start

When `/gsd parallel start` is called, the orchestrator:

1. Reads all existing session status files.
2. Checks each PID with `process.kill(pid, 0)` (signal 0 checks liveness without killing).
3. Dead PIDs are cleaned up — their status files are removed.
4. Living PIDs are reported as orphans (workers from a prior session still running).

### Stale Session Detection

Sessions are considered stale when either:
- The worker PID is no longer alive (`process.kill(pid, 0)` throws).
- The last heartbeat is older than 30 seconds.

`cleanupStaleSessions()` is called during `refreshWorkerStatuses()` on every dashboard refresh cycle. Stale workers are marked as `"error"` state in the orchestrator.

### Worker Crash Recovery

When a worker process crashes, the coordinator detects the dead PID via heartbeat expiry and marks the worker as crashed. The worker's disk state (`.gsd/auto.lock`, `.gsd/completed-units.json`, `.gsd/milestones/<MID>/`) is preserved in the worktree. On restart:

1. Run `/gsd doctor --fix` to clean up stale session files.
2. Run `/gsd parallel status` to see current state.
3. Run `/gsd parallel start` to spawn new workers for remaining milestones.

The new worker will find the existing milestone branch, re-attach the worktree to it, reconcile plan checkboxes, and resume from the last completed unit.

---

## Merge Strategy

When milestones complete, their worktree changes merge back to main.

### Configuration

```yaml
parallel:
  merge_strategy: "per-milestone"   # "per-slice" or "per-milestone"
  auto_merge: "confirm"              # "auto", "confirm", or "manual"
```

| Option | Behavior |
|--------|----------|
| `per-milestone` (default) | Waits for the full milestone to complete before merging |
| `per-slice` | Merges after each slice completes |
| `auto` | Merges automatically without prompting |
| `confirm` | Prompts before merging (default) |
| `manual` | Requires explicit `/gsd parallel merge` command |

### Merge Order

Milestones merge sequentially in ID order (M001 before M002, etc.) to minimize conflicts. The squash merge sequence for each milestone:

```
1. Auto-commit any dirty state in the worktree
2. Reconcile worktree gsd.db into main gsd.db
3. chdir to project root
4. git checkout <main-branch>
5. git merge --squash milestone/<MID>
6. Auto-resolve .gsd/ state file conflicts (accept milestone branch version)
7. git commit -m "feat(M002): Auth System\n\nCompleted slices:\n- S01: ...\n- S02: ..."
8. Auto-push (if git.auto_push: true)
9. Create PR (if git.auto_pr: true)
10. Remove worktree directory
11. Delete milestone branch
```

### Conflict Handling

Conflicts are separated by category:

| Conflict type | Resolution |
|--------------|------------|
| `.gsd/` state files | **Auto-resolved** — accept milestone branch version (it has the latest execution state) |
| Code files | **Stop and report** — halts the merge, shows which files conflict |

When code conflicts are detected, the merge halts with a `MergeConflictError`. Resolve manually in the worktree at `.gsd/worktrees/<MID>/` and retry:

```
/gsd parallel merge M002
```

### Example Merge Output

```
/gsd parallel merge

# Merge Results

- **M002** — merged successfully (pushed)
- **M003** — CONFLICT (2 file(s)):
  - `src/types.ts`
  - `src/middleware.ts`
  Resolve conflicts manually and run `/gsd parallel merge M003` to retry.
```

---

## Configuration Reference

Add to `~/.gsd/preferences.md` or `.gsd/preferences.md`:

```yaml
---
parallel:
  enabled: false            # Master toggle (default: false)
  max_workers: 2            # Concurrent workers (1-4, default: 2)
  budget_ceiling: 50.00     # Aggregate cost limit in USD (optional)
  merge_strategy: "per-milestone"   # "per-slice" or "per-milestone"
  auto_merge: "confirm"             # "auto", "confirm", or "manual"
---
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `enabled` | boolean | `false` | Master toggle. Must be `true` for `/gsd parallel` commands to work. |
| `max_workers` | number (1–4) | `2` | Maximum concurrent worker processes. Higher values use more memory and API budget. |
| `budget_ceiling` | number | none | Aggregate cost ceiling in USD across all workers. When reached, no new units are dispatched. |
| `merge_strategy` | `"per-slice"` or `"per-milestone"` | `"per-milestone"` | When worktree changes merge back to main. Per-milestone waits for the full milestone. |
| `auto_merge` | `"auto"`, `"confirm"`, `"manual"` | `"confirm"` | How merge-back is handled. `confirm` prompts before merging; `manual` requires explicit command. |

---

## Commands

| Command | Description |
|---------|-------------|
| `/gsd parallel start` | Analyze eligibility, confirm, and start workers |
| `/gsd parallel status` | Show all workers with state, units completed, and cost |
| `/gsd parallel stop` | Stop all workers (sends signal + SIGTERM) |
| `/gsd parallel stop M002` | Stop a specific milestone's worker |
| `/gsd parallel pause` | Pause all workers (finish current unit, then wait) |
| `/gsd parallel pause M002` | Pause a specific worker |
| `/gsd parallel resume` | Resume all paused workers |
| `/gsd parallel resume M002` | Resume a specific worker |
| `/gsd parallel merge` | Merge all completed milestones back to main |
| `/gsd parallel merge M002` | Merge a specific milestone back to main |

---

## File Layout

```
.gsd/
├── orchestrator.json            # Coordinator state (PIDs, costs, config)
├── parallel/                    # Coordinator ↔ worker IPC
│   ├── M002.status.json         # Worker heartbeat + progress
│   ├── M002.signal.json         # Coordinator → worker signals
│   ├── M003.status.json
│   └── M003.signal.json
└── worktrees/                   # Git worktrees (one per milestone)
    ├── M002/                    # M002's isolated checkout
    │   ├── .git                 # worktree gitdir pointer
    │   ├── .gsd/                # M002's own state files
    │   │   ├── auto.lock        # M002's crash lock
    │   │   ├── metrics.json     # M002's cost metrics
    │   │   ├── completed-units.json
    │   │   └── milestones/
    │   │       └── M002/        # M002's planning artifacts
    │   └── src/                 # M002's working copy of source
    └── M003/
        └── ...
```

Both `.gsd/parallel/` and `.gsd/worktrees/` are gitignored — they are runtime-only coordination files that never get committed.

---

## Safety Model

| Safety Layer | Protection |
|-------------|------------|
| **Feature flag** | `parallel.enabled: false` by default — existing users are completely unaffected |
| **Eligibility analysis** | Dependency and file overlap checks before starting any worker |
| **Worker isolation** | Separate processes, worktrees, branches, and context windows per milestone |
| **`GSD_MILESTONE_LOCK`** | Each worker's `deriveState()` only sees its assigned milestone |
| **`GSD_PARALLEL_WORKER`** | Workers cannot spawn nested parallel sessions |
| **Budget ceiling** | Aggregate cost enforcement across all workers |
| **Signal-based shutdown** | Graceful stop via file signals + SIGTERM + SIGKILL fallback |
| **Doctor integration** | Detects and cleans up orphaned sessions with dead PIDs |
| **Atomic writes** | All IPC files use write-to-temp + rename to prevent partial reads |
| **Conflict-aware merge** | Stops on code conflicts, auto-resolves `.gsd/` state conflicts |

---

## TUI Monitor Dashboard (v2.54)

Starting in v2.54.0, parallel orchestration includes a real-time TUI monitor dashboard with self-healing capabilities. The dashboard provides:

- Live worker status with phase, progress, and cost for each milestone
- Automatic detection and recovery of failed workers (self-healing)
- Zombie worker cleanup for processes stuck in error state
- Visual indicators for worker health and resource consumption

The dashboard runs as a persistent overlay that auto-refreshes, complementing the snapshot-style `/gsd parallel status` command.

---

## `/gsd parallel watch` (v2.56)

The `/gsd parallel watch` command provides a native TUI overlay specifically for worker monitoring. It shows:

- All active workers with real-time status updates
- Per-worker cost tracking and unit completion counts
- Health indicators and heartbeat status
- Session lock state for each worker

This is lighter-weight than the full monitor dashboard and designed for quick status checks during parallel execution.

---

## Self-Healing Workers (v2.54)

Starting in v2.54.0, parallel workers include self-healing capabilities. When a worker encounters a recoverable error (transient provider failure, temporary lock contention, stale state), it can automatically recover without coordinator intervention. The self-healing system:

- Detects and cleans up zombie workers stuck in error state
- Automatically restarts failed workers when the underlying issue is transient
- Reports unrecoverable failures to the coordinator for manual intervention

---

## Session Lock Contention Fixes (v2.56)

v2.56.0 resolved session lock contention issues that could cause parallel workers to stall. The fixes address three related bugs:

- Lock file contention between workers sharing the same project root
- Race conditions when multiple workers attempt simultaneous state transitions
- Stale lock detection that was too aggressive, causing healthy workers to be killed

These fixes significantly improve reliability of parallel execution with 3+ concurrent workers.

---

## Doctor Integration

`/gsd doctor` detects parallel-specific issues:

**Stale parallel sessions**: Worker processes that died without cleanup. Doctor finds `.gsd/parallel/*.status.json` files with dead PIDs or heartbeats older than 30 seconds and removes them.

Run `/gsd doctor --fix` to clean up automatically.

---

## Troubleshooting

### "Parallel mode is not enabled"

Set `parallel.enabled: true` in your preferences file. The feature is off by default and requires explicit opt-in.

### "No milestones are eligible for parallel execution"

All milestones are either complete, parked, or blocked by dependencies. Check `/gsd queue` to see milestone status and dependency chains. A milestone is blocked if any of its `dependsOn` entries have not yet reached `complete` status.

### Worker crashed — recovery steps

1. Run `/gsd doctor --fix` to remove stale session files.
2. Run `/gsd parallel status` to see which milestones are still in progress.
3. Run `/gsd parallel start` — the orchestrator will find the existing milestone branch, re-attach the worktree, reconcile checkpoint state, and resume.

The worktree's `.gsd/auto.lock` and `completed-units.json` are preserved through crashes. The resumed worker picks up from the last completed unit.

### Merge conflicts after parallel completion

1. Run `/gsd parallel merge` to identify which milestones have conflicts.
2. Navigate to the worktree: `cd .gsd/worktrees/<MID>/`.
3. Resolve conflicts in the source files.
4. Stage and commit the resolution.
5. Retry: `/gsd parallel merge <MID>`.

Note that `.gsd/` state file conflicts are auto-resolved and never require manual intervention.

### Workers appear stuck / not progressing

Check if the budget ceiling was reached: `/gsd parallel status` shows per-worker cost and aggregate total. If the ceiling is hit, either increase `parallel.budget_ceiling` or remove it entirely:

```yaml
parallel:
  budget_ceiling:    # remove or comment out to disable
```

Also check if a worker is waiting for slice discussion approval (`require_slice_discussion: true` in preferences will pause workers before each slice).

### Worker PID shows 0 in status

The worker process failed to spawn. Common causes:
- `GSD_BIN_PATH` is not set or points to a non-existent file.
- The worktree directory could not be created (git not available, disk full).

Check the coordinator's stderr output and verify the GSD binary path is correct.
