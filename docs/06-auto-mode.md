# Auto Mode

Auto mode is GSD's autonomous execution engine. Run `/gsd auto`, walk away, and return to built software with a clean git history.

## What Auto Mode Is

Auto mode is a **state machine driven by files on disk**. It reads `.gsd/STATE.md` (and the full `.gsd/` directory tree), determines the next unit of work, creates a fresh agent session, injects a focused prompt with all relevant context pre-inlined, and lets the LLM execute. When the LLM finishes, auto mode reads disk state again and dispatches the next unit.

Every unit of work — planning a slice, executing a task, completing a milestone — gets its own fresh context window. There is no accumulated context across units: no degraded quality from context bloat, no hallucinated state. The dispatch prompt is constructed with everything the LLM needs to start immediately.

Auto mode does not require continuous human supervision. It handles crash recovery, provider errors, verification failures, and stuck units automatically. For projects where you want truly unattended execution, combine it with headless mode (see below).

---

## The GSD Workflow: Roadmap → Milestones → Slices → Tasks

GSD organizes work into a four-level hierarchy:

```
ROADMAP.md
└── Milestone (M001, M002, ...)
    └── Slice (S01, S02, ...)
        └── Task (T01, T02, ...)
```

- **Milestone**: a shippable feature or body of work. Has a ROADMAP file listing its slices with success criteria.
- **Slice**: a coherent unit of implementation within a milestone. Has a PLAN file listing tasks.
- **Task**: a single executable unit of work within a slice. Has its own PLAN file with a specific goal.

Auto mode works through this hierarchy sequentially by default. Milestones run one at a time; slices within a milestone run in order; tasks within a slice run in order. Parallel orchestration (covered in the companion document) lets multiple milestones run concurrently.

---

## State Machine: Phases

Auto mode derives its current position entirely from files on disk via `deriveState()`. The function reads the `.gsd/` tree — roadmaps, slice plans, task summaries — and returns the current phase without maintaining in-memory state across sessions. This makes the state machine crash-safe: any restart reads the same files and arrives at the same position.

### Phase Diagram

```
                       ┌──────────────────────┐
                       │     pre-planning     │
                       │  (no context/roadmap) │
                       └────────┬─────────────┘
                                │ guided flow creates CONTEXT.md
                                ▼
               ┌────────────────────────────────┐
               │  research-milestone (optional)  │
               │  Produces: RESEARCH.md          │
               └───────────────┬────────────────┘
                               │
                               ▼
               ┌────────────────────────────────┐
               │       plan-milestone            │
               │  Produces: ROADMAP.md           │
               └───────────────┬────────────────┘
                               │ for each slice:
                               ▼
          ┌──────────────────────────────────────────┐
          │               planning                   │
          │  research-slice (optional) → plan-slice  │
          │  Produces: RESEARCH.md, PLAN.md           │
          └──────────────────┬───────────────────────┘
                             │ for each task in slice:
                             ▼
          ┌──────────────────────────────────────────┐
          │               executing                  │
          │  execute-task × N                        │
          │  Produces: T##-SUMMARY.md per task       │
          └──────────────────┬───────────────────────┘
                             │ all tasks done
                             ▼
          ┌──────────────────────────────────────────┐
          │             summarizing                  │
          │  complete-slice                          │
          │  Produces: SUMMARY.md, UAT.md            │
          │  Marks slice [x] in ROADMAP.md           │
          └──────────────────┬───────────────────────┘
                             │
                    ┌────────┴────────┐
                    │                 │
                    ▼                 ▼
           run-uat (optional)  reassess-roadmap (optional)
           UAT-RESULT.md       ASSESSMENT.md
                    │                 │
                    └────────┬────────┘
                             │ next slice, or if all slices done:
                             ▼
          ┌──────────────────────────────────────────┐
          │         validating-milestone             │
          │  validate-milestone                      │
          │  Produces: VALIDATION.md                 │
          │  Verdict: pass / needs-attention /       │
          │           needs-remediation              │
          └──────────────────┬───────────────────────┘
                             │ terminal verdict
                             ▼
          ┌──────────────────────────────────────────┐
          │         completing-milestone             │
          │  complete-milestone                      │
          │  Produces: SUMMARY.md                    │
          │  Merges worktree → main                  │
          └──────────────────────────────────────────┘
```

### Phase Reference

| Phase | Unit Type | Expected Artifact |
|-------|-----------|-------------------|
| `pre-planning` | research-milestone, plan-milestone | RESEARCH.md, ROADMAP.md |
| `planning` | research-slice, plan-slice | RESEARCH.md, PLAN.md + task PLAN files |
| `replanning-slice` | replan-slice | REPLAN.md |
| `executing` | execute-task | T##-SUMMARY.md + checkbox [x] in slice PLAN |
| `summarizing` | complete-slice | SUMMARY.md + UAT.md + slice [x] in ROADMAP |
| `validating-milestone` | validate-milestone | VALIDATION.md |
| `completing-milestone` | complete-milestone | SUMMARY.md (milestone-level) |

The dispatch table in `auto-dispatch.ts` evaluates rules in order. The first matching rule wins and returns the unit type, unit ID, and prompt to dispatch. If no rule matches, auto mode stops with an "unhandled phase" error and directs you to `/gsd doctor`.

### Phase Skipping

Token profiles can skip certain phases to reduce cost:

| Phase | `budget` | `balanced` | `quality` |
|-------|----------|------------|-----------|
| Milestone Research | Skipped | Runs | Runs |
| Slice Research | Skipped | Skipped | Runs |
| Reassess Roadmap | Skipped | Runs | Runs |

When `prefs.phases.skip_milestone_validation` is set, the validate-milestone phase writes a minimal pass-through VALIDATION file and skips the agent session entirely.

---

## Guided Flow: How Slices Are Planned and Dispatched

When auto mode starts with no active milestone (or the active milestone has no roadmap), it calls `showSmartEntry()` — the guided flow wizard. This is an interactive step that walks you through project setup:

1. Detects whether a project is already initialized.
2. Shows the current queue and asks what to build.
3. Creates CONTEXT.md for the selected milestone with your input.
4. Writes STATE.md as the final step (serves as the "discussion finalized" gate for auto-start).

After the guided flow completes, auto mode checks for a CONTEXT.md or ROADMAP.md on the milestone directory, then checks that STATE.md exists, before proceeding to dispatch the first unit (typically research-milestone or plan-milestone).

For multi-milestone discussions, a `DISCUSSION-MANIFEST.json` tracks completion of readiness gates. Auto mode does not start until `gates_completed` equals the total count.

### Slice Discussion Gate

When `require_slice_discussion: true` is set in preferences, auto mode pauses before each new slice begins and presents the slice plan for your review. After confirmation, execution continues. This is enforced by the `pauseAfterDispatch` flag on the dispatch action.

---

## Context Pre-Loading

Every unit prompt is carefully constructed with pre-inlined artifacts:

| Inlined Artifact | Purpose |
|------------------|---------|
| Task plan | What to build |
| Slice plan | Where this task fits |
| Prior task summaries | What is already done |
| Dependency summaries | Cross-slice context |
| Roadmap excerpt | Overall direction |
| Decisions register | Architectural context |
| KNOWLEDGE.md | Cross-session learned rules |

The amount of context inlined is controlled by your token profile (`budget`, `balanced`, `quality`). Budget mode inlines minimal context; quality mode inlines everything. A semantic chunker and prompt compressor manage context that would exceed the LLM's effective window.

---

## Session Lifecycle

### Start

```
/gsd auto
```

The bootstrap sequence (`bootstrapAutoSession`) runs on fresh start:

1. Ensures git is initialized (`git init` if needed).
2. Ensures `.gitignore` has baseline patterns.
3. Bootstraps `.gsd/` directory if missing.
4. Initializes `GitServiceImpl` for the session.
5. Checks for a crash lock from the previous session.
6. Invalidates all caches and derives current state.
7. Runs stale worktree state recovery if needed.
8. Calls guided flow if no active milestone exists.
9. Sets up session state (counters, timers, completed keys).
10. Registers SIGTERM handler.
11. Creates or enters the auto-worktree (in worktree isolation mode).
12. Initializes metrics and routing history.
13. Collects secrets from manifest if needed.
14. Self-heals stale runtime records and stale `.git/index.lock`.
15. Writes the initial lock file.
16. Dispatches the first unit.

### Pause

Press **Escape**. The conversation is preserved. You can interact with the agent, inspect state, or resume.

### Resume

```
/gsd auto
```

Auto mode reads disk state and picks up where it left off. The resume path reads the crash lock (if present), synthesizes a recovery briefing from tool calls that made it to disk, and resumes with full context.

### Stop

```
/gsd stop
```

Stops auto mode gracefully. Can be run from a different terminal — the command writes a stop signal that the running session detects between units.

### Steer

```
/gsd steer
```

Hard-steer plan documents during execution without stopping the pipeline. Changes are picked up at the next phase boundary.

### Headless (v2.26)

```
gsd headless auto
```

Crashes trigger automatic restart with exponential backoff: 5s → 10s → 30s cap, default 3 attempts. Configure with `--max-restarts N`. SIGINT/SIGTERM bypasses restart. Combined with crash recovery, this enables true overnight "run until done" execution.

---

## Crash Recovery Mechanisms

### Lock File

A lock file at `.gsd/auto.lock` tracks the current unit (type, ID, session file, PID, timestamp). If the session dies, the next `/gsd auto` reads the surviving lock. If the PID is no longer alive, it proceeds with recovery.

### Session Forensics

`synthesizeCrashRecovery()` reads the activity log for the crashed session, reconstructs all tool calls that made it to disk, and produces a recovery briefing that is injected into the first prompt of the resumed session. This lets the LLM pick up with full context of what was happening before the crash.

### Artifact Verification

Every unit has an expected artifact. `verifyExpectedArtifact()` checks:

- The primary artifact file exists on disk.
- For `plan-slice`: the plan contains actual task entries (not just a scaffold), and individual task plan files exist for every listed task.
- For `execute-task`: the task summary exists AND the task checkbox is marked `[x]` in the slice plan.
- For `complete-slice`: both SUMMARY.md and UAT.md exist, AND the slice is marked `[x]` in the roadmap.

If verification fails, the unit is considered incomplete and will be re-dispatched.

### Stuck Detection

If the same unit dispatches twice without producing its expected artifact, GSD retries once with a deep diagnostic prompt. If it fails again, auto mode writes a blocker placeholder artifact (so the pipeline can advance) and stops with the exact file it expected, giving you a specific intervention target.

Loop remediation steps are also generated for manual recovery:

```
# For a stuck execute-task:
1. Write T##-SUMMARY.md (even partial)
2. Mark T## [x] in the slice PLAN
3. Run gsd doctor
4. Resume auto-mode
```

### Merge State Reconciliation

If a crash left leftover merge state (`.git/MERGE_HEAD` or `.git/SQUASH_MSG`), auto mode detects this on startup. If conflicts are resolved, it finalizes the commit. If `.gsd/` state file conflicts remain, it auto-resolves by accepting the milestone branch version. If code conflicts remain, it aborts and resets, then re-derives state.

### Provider Error Recovery

| Error type | Examples | Action |
|-----------|----------|--------|
| **Rate limit** | 429, "too many requests" | Auto-resume after retry-after header or 60s |
| **Server error** | 500, 502, 503, "overloaded", "api_error" | Auto-resume after 30s |
| **Permanent** | "unauthorized", "invalid key", "billing" | Pause indefinitely (requires manual resume) |

No manual intervention needed for transient errors.

---

## Verification Gate

The verification gate runs after every `execute-task` unit. It enforces code quality mechanically, not by LLM compliance.

### Configuration

```yaml
verification_commands:
  - npm run lint
  - npm run test
verification_auto_fix: true    # auto-retry on failure (default: true)
verification_max_retries: 2    # max retry attempts (default: 2)
```

Task plans can also specify a per-task verify command in the `verify` field of the task entry in the slice plan.

### Verification Flow

```
execute-task completes
        │
        ▼
runVerificationGate() — runs all configured commands
        │
        ├── passed → continue to next unit
        │
        └── failed
              │
              ├── retries remaining (attempt + 1 <= max_retries)
              │     └── clear completion key
              │         inject failure context into next prompt
              │         re-dispatch execute-task
              │
              └── retries exhausted
                    └── pauseAuto() — stop for human review
```

Verification evidence is written to `.gsd/milestones/<MID>/<SID>/tasks/<TID>-VERIFICATION.json` for audit purposes. The file includes check results, exit codes, stderr snippets, and retry count.

### Runtime Error Capture

In addition to configured commands, the verification gate captures runtime errors from process monitoring. Blocking runtime errors cause the gate to fail even if shell commands pass.

### Dependency Audit

The verification gate also runs a dependency audit (checking for known vulnerabilities), which produces warnings written to stderr. Audit failures are non-blocking by default.

---

## Auto-Mode Preferences

Configure in `~/.gsd/preferences.md` or `.gsd/preferences.md`:

```yaml
---
# Session behavior
require_slice_discussion: false   # pause before each slice for human review (v2.26)
auto_report: true                 # generate HTML report on milestone completion (v2.26)

# Verification
verification_commands:
  - npm run lint
  - npm run test
verification_auto_fix: true
verification_max_retries: 2

# Timeout supervision
auto_supervisor:
  soft_timeout_minutes: 20     # warn LLM to wrap up
  idle_timeout_minutes: 10     # detect stalls, intervene
  hard_timeout_minutes: 30     # pause auto mode

# Git
git:
  isolation: worktree           # "worktree" | "branch" | "none"
  auto_push: false
  commit_docs: true
  main_branch: main
  worktree_post_create: ./scripts/setup-worktree.sh   # optional hook

# Phase skipping
phases:
  skip_research: false
  skip_slice_research: false
  skip_reassess: false
  skip_milestone_validation: false
---
```

### `require_slice_discussion`

When `true`, auto mode pauses before each slice and presents the slice context. After you confirm, execution continues. Useful for high-stakes projects where you want to review the plan before the agent builds.

### `commit_docs`

When `false`, GSD adds `.gsd/` to `.gitignore` and keeps all planning artifacts local-only. Useful for teams where only some members use GSD, or when company policy requires a clean repository.

### `auto_report`

When `true` (default), GSD auto-generates a self-contained HTML report in `.gsd/reports/` after each milestone completes. Reports include: project summary, progress tree, slice dependency graph (SVG DAG), cost/token metrics with bar charts, execution timeline, changelog, and knowledge base. No external dependencies — all CSS and JS are inlined.

Generate manually anytime:

```
/gsd export --html
/gsd export --html --all    # all milestones (v2.28)
```

---

## Timeout Supervision

Three timeout tiers prevent runaway sessions:

| Timeout | Default | Behavior |
|---------|---------|----------|
| Soft | 20 min | Warns the LLM to wrap up |
| Idle | 10 min | Detects stalls, intervenes |
| Hard | 30 min | Pauses auto mode |

Recovery steering nudges the LLM to finish durable output (commit, write summaries) before timing out.

---

## Context Pressure Monitor (v2.26)

When context usage reaches 70%, GSD sends a wrap-up signal to the agent, nudging it to finish durable output before the context window fills. This prevents sessions from hitting the hard context limit mid-task with no artifacts written.

---

## Incremental Memory: KNOWLEDGE.md (v2.26)

GSD maintains a `KNOWLEDGE.md` file at `.gsd/KNOWLEDGE.md` — an append-only register of project-specific rules, patterns, and lessons learned. The agent reads it at the start of every unit and appends to it when discovering:

- Recurring issues that caused retries
- Non-obvious patterns in the codebase
- Rules that future sessions should follow

This gives auto mode cross-session memory that survives context window boundaries.

---

## Meaningful Commit Messages (v2.26)

Commits are generated from task summaries — not generic "complete task" messages. Each commit message reflects what was actually built:

```
feat(S01/T01): core type definitions
feat(S01/T02): markdown parser for plan files
fix(M001/S03): bug fixes and doc corrections
docs(M001/S04): workflow documentation
```

---

## Doctor: Health Checks and Self-Healing

`/gsd doctor` runs preflight health checks:

- **Crash lock**: detects and reports stale crash locks from dead PIDs.
- **Stale runtime records**: clears unit records where the artifact already exists (completed but closeout didn't finish).
- **Stale `.git/index.lock`**: removes lock files older than 60 seconds from crashed git processes.
- **Detached HEAD**: automatically reattaches to the correct branch.
- **Orphaned worktrees**: detects abandoned worktrees and offers cleanup.
- **Parallel sessions**: detects stale parallel worker sessions with dead PIDs or expired heartbeats.

Run `/gsd doctor --fix` to apply all self-healing actions automatically.

Self-healing also runs automatically at auto-mode startup:

```
selfHealRuntimeRecords()   — clears stale runtime records, persists completion keys
reconcileMergeState()      — finalizes or aborts leftover merge state
git-lock cleanup           — removes stale .git/index.lock files > 60s old
```

---

## File Formats

### STATE.md

A quick-glance status file written after every unit completes. Used by the dashboard and on initial startup. The source of truth is always derived from the full `.gsd/` tree — STATE.md is a cache.

Typical content:

```markdown
# GSD State

**Active Milestone**: M001 — Core Authentication
**Phase**: executing
**Active Slice**: S02 — JWT Middleware
**Active Task**: T03 — Token validation handler
**Progress**: 2/5 slices complete
```

### ROADMAP.md

Written by the `plan-milestone` unit. Contains the milestone title, description, success criteria, and an ordered list of slices with checkboxes:

```markdown
# M001: Core Authentication

## Success Criteria
- Users can register and log in
- JWT tokens are issued and validated
- Sessions expire after 24 hours

## Slices
- [ ] **S01**: User registration and password hashing
- [ ] **S02**: JWT issuance and validation middleware
- [ ] **S03**: Session management and refresh tokens
```

Slices are marked `[x]` as they complete. `isMilestoneComplete()` returns true when all slices are checked.

### KNOWLEDGE.md

An append-only file at `.gsd/KNOWLEDGE.md`. Written by the agent during execution. Format is free-form markdown, typically organized by topic:

```markdown
# Project Knowledge

## Testing
- Always run `npm test -- --passWithNoTests` to avoid failures on empty test suites
- Integration tests require a running database — use the test docker-compose

## Architecture
- Authentication middleware must be applied before rate limiting
- The `User` type is defined in `src/types/user.ts` and is the canonical source
```

---

## Dashboard

`Ctrl+Alt+G` or `/gsd status` shows real-time progress:

- Current milestone, slice, and task
- Auto mode elapsed time and phase
- Per-unit cost and token breakdown
- Cost projections
- Completed and in-progress units
- Pending capture count
- Parallel worker status (when running parallel milestones — includes 80% budget alert)

---

## Adaptive Replanning

After each slice completes, the roadmap is reassessed. The `reassess-roadmap` unit produces an ASSESSMENT.md file. If the work revealed new information that changes the plan, slices are reordered, added, or removed before continuing. This can be skipped with the `balanced` or `budget` token profiles.

---

## Dynamic Model Routing

When enabled, auto mode automatically selects cheaper models for simple units (slice completion, UAT) and reserves expensive models for complex work (replanning, architectural tasks). See Dynamic Model Routing documentation for configuration.

---

## Post-Mortem Investigation

When auto mode fails or produces unexpected results, `/gsd forensics` provides structured post-mortem analysis. It inspects activity logs, crash locks, and session state to identify root causes — whether the failure was a model error, missing context, a stuck loop, or a broken tool call.

---

## Failure Recovery Hardening (v2.28)

v2.28 added multiple reliability safeguards:

- **Atomic file writes**: write-to-temp + rename prevents corruption on crash.
- **OAuth fetch timeouts**: 30-second timeout prevents indefinite hangs.
- **RPC subprocess exit detection**: detects and reports subprocess failures.
- **Blob garbage collection**: prevents unbounded disk growth.

Combined with crash recovery and headless auto-restart, auto mode is designed for true "fire and forget" overnight execution.

---

## Tips for Effective Auto-Mode Use

**Keep milestones focused.** Milestones that span too many concerns produce large roadmaps that are harder to validate. Aim for milestones that can complete in one session (a few hours).

**Write good CONTEXT.md files.** The CONTEXT.md is the input to planning. Include: what you want built, what already exists, constraints, and non-obvious context. More detail in CONTEXT produces better ROADMAP files.

**Use `require_slice_discussion` on high-stakes work.** Reviewing the slice plan before execution catches planning errors before they become code. Costs one pause per slice.

**Let verification commands fail fast.** If `npm run test` takes 10 minutes, configure faster verification checks (`npm run test:unit`) and reserve slower suites for manual review.

**Check KNOWLEDGE.md after the first milestone.** The agent appends lessons learned. Review and edit them to ensure only accurate rules persist.

**Use `/gsd steer` for course corrections.** If you realize mid-execution that the plan needs adjustment, `/gsd steer` applies changes at the next phase boundary without stopping the pipeline.

**Use headless mode for overnight runs.** `gsd headless auto` with `--max-restarts 5` will keep going through transient errors, rate limits, and even session crashes.
