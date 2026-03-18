# GSD Extension

The GSD extension (`/gsd`) is the primary workflow orchestration layer built on top of the pi framework. It implements a hierarchical milestone-slice-task planning model, a multi-phase state machine, and a comprehensive auto-mode loop that drives LLM sessions from initial planning through verification and completion. The extension is registered under the `/gsd` command namespace and hooks into pi's agent lifecycle at well-defined points.

## Overview

GSD stands for Get Stuff Done. The extension converts a vague project goal into a structured decomposition: a **milestone** contains multiple **slices**, each slice contains multiple **tasks**. All state is stored as Markdown files under a `.gsd/` directory in the project root. The LLM agent reads from and writes to these files; GSD drives the session sequence and enforces workflow invariants.

Key behaviors:
- `/gsd` — contextual wizard that detects current phase and offers the right options
- `/gsd auto` — starts an auto-mode loop that opens fresh sessions until a milestone is complete
- `/gsd stop` — gracefully halts auto-mode after the current unit finishes
- `/gsd status` — displays a progress dashboard from derived state

## Registration and Lifecycle (index.ts)

The extension registers itself via the pi `ExtensionAPI`. The entry point (`index.ts`) wires together three categories of hooks:

**Commands registered:**
- `registerGSDCommand` — the main `/gsd` dispatch tree
- `registerExitCommand` — clean stop of auto-mode
- `registerWorktreeCommand` — worktree create/merge/remove

**Hooks registered:**

```typescript
// before_agent_start — inject GSD system context for GSD projects
// agent_end — auto-mode advancement (advance to the next unit after session ends)
// session_before_compact — save continue.md OR block during auto-mode
```

The `before_agent_start` hook reads state from disk and injects a GSD system prompt block into every session started inside a GSD project. The `agent_end` hook is the heartbeat of auto-mode: it calls `handleAgentEnd()` which checks whether the completed unit's expected artifact exists, marks the unit done, and dispatches the next unit in the queue. The `session_before_compact` hook either saves a `CONTINUE.md` checkpoint or blocks compaction while auto-mode is running (to avoid corrupting mid-unit context).

## Directory Structure

The extension source spans approximately 130 TypeScript files organized by responsibility:

### Core Infrastructure
| File | Description |
|------|-------------|
| `index.ts` | Extension registration, hook wiring, command dispatch |
| `types.ts` | All shared type definitions — pure interfaces, no runtime deps |
| `state.ts` | State derivation from disk — the source of truth |
| `cache.ts` | Unified cache invalidation across state, path, and parse caches |
| `paths.ts` | Path resolution helpers for `.gsd/` directory layout |
| `files.ts` | Markdown file parse/write helpers (roadmap, plan, summary, continue) |
| `errors.ts` | Typed GSD error codes and `GSDError` class |
| `constants.ts` | Shared runtime constants |
| `atomic-write.ts` | Atomic file write helper (write-then-rename) |
| `safe-fs.ts` | Filesystem operation wrappers with error suppression |

### State and Database
| File | Description |
|------|-------------|
| `gsd-db.ts` | SQLite persistence layer — decisions, requirements, artifacts, memories |
| `context-store.ts` | In-memory context accumulation during sessions |
| `history.ts` | Session history log access |
| `workspace-index.ts` | Workspace-level index of active GSD projects |

### Planning and Workflow
| File | Description |
|------|-------------|
| `guided-flow.ts` | Phase sequencing and milestone ID discovery |
| `guided-flow-queue.ts` | Work-unit queue management |
| `queue-order.ts` | Dependency-aware ordering of queued units |
| `queue-reorder-ui.ts` | Interactive reorder UI for the work queue |
| `milestone-actions.ts` | Milestone-level operations (create, complete, park) |
| `milestone-ids.ts` | Milestone ID generation and validation |
| `roadmap-slices.ts` | Roadmap parsing and slice dependency resolution |
| `commands.ts` | Main GSD command dispatch and sub-command routing |
| `commands-config.ts` | Configuration-related sub-commands |
| `commands-handlers.ts` | Command handler implementations |
| `commands-inspect.ts` | Inspect/debug sub-commands |
| `commands-maintenance.ts` | Maintenance and repair sub-commands |
| `commands-prefs-wizard.ts` | Preferences setup wizard |
| `init-wizard.ts` | First-run project initialization wizard |
| `quick.ts` | Quick-action shortcuts for common operations |
| `undo.ts` | Undo/rollback of last unit |

### Auto-mode
| File | Description |
|------|-------------|
| `auto.ts` | Auto-mode public API (start, stop, pause, agent-end handler) |
| `auto-start.ts` | Auto-mode session startup, lock acquisition, model selection |
| `auto-dispatch.ts` | Unit dispatch logic — selects the next unit to run |
| `auto-direct-dispatch.ts` | Direct (non-queued) dispatch path |
| `auto-prompts.ts` | Prompt assembly for each unit type |
| `auto-unit-closeout.ts` | Post-unit artifact validation and state advancement |
| `auto-idempotency.ts` | Idempotency key tracking to prevent duplicate units |
| `auto-recovery.ts` | Recovery from partial or failed units |
| `auto-post-unit.ts` | Post-unit hook execution |
| `auto-model-selection.ts` | Per-unit model resolution in auto-mode |
| `auto-verification.ts` | Verification gate integration in auto-mode |
| `auto-dashboard.ts` | Auto-mode progress dashboard |
| `auto-budget.ts` | Budget enforcement during auto-mode |
| `auto-timers.ts` | Session timeout and idle detection timers |
| `auto-timeout-recovery.ts` | Recovery from timed-out units |
| `auto-stuck-detection.ts` | Detection of stuck/looping auto sessions |
| `auto-supervisor.ts` | Optional supervisor model for auto-mode oversight |
| `auto-observability.ts` | Metrics and logging during auto-mode |
| `auto-tool-tracking.ts` | Tool call tracking within units |
| `auto-worktree.ts` | Worktree selection for auto-mode |
| `auto-worktree-sync.ts` | Cross-worktree artifact synchronization |
| `auto/` | Subdirectory with additional auto-mode modules |

### Health and Verification
| File | Description |
|------|-------------|
| `doctor.ts` | Doctor command — audit GSD state integrity, apply fixes |
| `doctor-checks.ts` | Git health and runtime health check implementations |
| `doctor-types.ts` | `DoctorIssue`, `DoctorReport`, `DoctorSeverity` types |
| `doctor-format.ts` | Doctor report formatting for display and prompt injection |
| `doctor-proactive.ts` | Proactive background health checks |
| `verification-gate.ts` | Command discovery and execution for task verification |
| `verification-evidence.ts` | JSON evidence artifact writing and markdown table formatting |
| `dispatch-guard.ts` | Pre-dispatch validation guards |
| `observability-validator.ts` | Validates observability surface declarations in summaries |

### Preferences and Model Routing
| File | Description |
|------|-------------|
| `preferences.ts` | Preferences loading, merging, and rendering (entry point) |
| `preferences-types.ts` | `GSDPreferences` interface and all related types |
| `preferences-validation.ts` | Preference value validation and normalization |
| `preferences-models.ts` | Model resolution functions — `resolveModelForUnit`, fallback chain |
| `preferences-skills.ts` | Skill reference resolution for `always_use_skills`, `prefer_skills` |
| `preferences-hooks.ts` | Hook resolution helpers |
| `model-router.ts` | Dynamic model router — complexity-tier-based model downgrade |
| `routing-history.ts` | Per-unit routing decision log |
| `complexity-classifier.ts` | Task complexity classification (`light`/`standard`/`heavy`) |
| `model-cost-table.ts` | Static model cost table for cross-provider comparisons |
| `provider-error-pause.ts` | Provider error detection and session pause |

### Worktrees and Git
| File | Description |
|------|-------------|
| `worktree.ts` | Worktree utilities — detection, branch naming, git HEAD resolution |
| `worktree-manager.ts` | Worktree create/remove/diff/merge operations |
| `worktree-command.ts` | `/worktree` sub-command handler |
| `git-service.ts` | Git operations — commit, staging, branch management, `GitServiceImpl` |
| `git-constants.ts` | Git environment constants (no-prompt env) |
| `git-self-heal.ts` | Abort and reset for corrupt merge/rebase states |
| `gitignore.ts` | `.gitignore` baseline pattern management |
| `native-git-bridge.ts` | Rust native module bridge for git operations |
| `collision-diagnostics.ts` | Branch name collision detection |

### Visualization and Export
| File | Description |
|------|-------------|
| `visualizer-overlay.ts` | TUI overlay container and tab navigation |
| `visualizer-views.ts` | Per-tab view renderers (progress, deps, metrics, timeline, etc.) |
| `visualizer-data.ts` | Data loader and aggregator for the visualizer |
| `dashboard-overlay.ts` | Simplified dashboard overlay for `/gsd status` |
| `export-html.ts` | Self-contained HTML report generator |
| `export.ts` | Markdown and JSON export handlers |
| `reports.ts` | Report aggregation utilities |

### Crash Recovery and Infrastructure
| File | Description |
|------|-------------|
| `crash-recovery.ts` | Lock-file crash detection — `auto.lock` lifecycle |
| `context-budget.ts` | Context window budget allocation and section-boundary truncation |
| `token-counter.ts` | Token estimation for different providers |
| `prompt-compressor.ts` | Heuristic prompt compression before truncation |
| `prompt-cache-optimizer.ts` | Cache-aware prompt ordering |
| `prompt-ordering.ts` | Deterministic prompt section ordering |
| `prompt-loader.ts` | Loads prompt files from the prompts/ directory |
| `notifications.ts` | Notification dispatch (on_complete, on_error, on_budget, etc.) |
| `activity-log.ts` | Per-session activity log writer and pruner |
| `session-status-io.ts` | Parallel session status file I/O |
| `session-forensics.ts` | Post-crash session forensics from JSONL files |
| `forensics.ts` | Broader project forensics |
| `detection.ts` | GSD project detection in current directory |
| `debug-logger.ts` | Debug timing and count instrumentation |
| `metrics.ts` | Usage metrics — ledger, aggregation by phase/slice/model/tier |
| `diff-context.ts` | Git diff context injection into prompts |
| `structured-data-formatter.ts` | Structured output formatting |
| `jsonl-utils.ts` | JSONL read/write helpers |
| `unit-runtime.ts` | Per-unit runtime metadata tracking |

### Skills and Marketplace
| File | Description |
|------|-------------|
| `skill-discovery.ts` | Skill file discovery and resolution |
| `skill-health.ts` | Skill staleness and health checks |
| `skill-telemetry.ts` | Skill usage telemetry |
| `marketplace-discovery.ts` | Skill marketplace discovery |
| `namespaced-registry.ts` | Namespaced skill registry |
| `namespaced-resolver.ts` | Namespaced skill reference resolution |
| `skills/` | Built-in skill files |

### Capture and Triage
| File | Description |
|------|-------------|
| `captures.ts` | Capture entry CRUD — quick notes for deferred work |
| `triage-ui.ts` | Interactive triage UI for pending captures |
| `triage-resolution.ts` | Capture resolution workflows |

### Parallel Orchestration
| File | Description |
|------|-------------|
| `parallel-orchestrator.ts` | Multi-milestone parallel execution coordinator |
| `parallel-eligibility.ts` | Eligibility checks for parallel execution |
| `parallel-merge.ts` | Parallel branch merge handler |

### Memory
| File | Description |
|------|-------------|
| `memory-extractor.ts` | Extracts memories from activity logs |
| `memory-store.ts` | Memory CRUD backed by `gsd-db.ts` |

### Post-unit Hooks
| File | Description |
|------|-------------|
| `post-unit-hooks.ts` | Post-unit hook dispatch and cycle-count tracking |

### Import and Migration
| File | Description |
|------|-------------|
| `claude-import.ts` | Import Claude conversation history into GSD |
| `md-importer.ts` | Import markdown files as GSD artifacts |
| `plugin-importer.ts` | Plugin/extension import utilities |
| `migrate/` | Schema and data migration scripts |

### Miscellaneous
| File | Description |
|------|-------------|
| `key-manager.ts` | API key management for tool integrations |
| `summary-distiller.ts` | Distill long summaries to compressed form |
| `semantic-chunker.ts` | Semantic chunking for context selection |
| `validate-directory.ts` | Directory structure validation |
| `file-watcher.ts` | File system watcher for live state updates |
| `docs/` | Bundled workflow documentation |
| `prompts/` | Prompt template files |
| `templates/` | Planning document templates |
| `tests/` | Extension test suite |

---

## Core Types and State

### Key Types (types.ts)

The `Phase` type is the central state machine value — it represents where GSD is in the workflow at any point:

```typescript
export type Phase =
  | 'pre-planning'
  | 'needs-discussion'
  | 'discussing'
  | 'researching'
  | 'planning'
  | 'executing'
  | 'verifying'
  | 'summarizing'
  | 'advancing'
  | 'validating-milestone'
  | 'completing-milestone'
  | 'replanning-slice'
  | 'complete'
  | 'paused'
  | 'blocked';
```

`GSDState` is the aggregate derived from disk — the snapshot of "where we are right now":

```typescript
export interface GSDState {
  activeMilestone: ActiveRef | null;
  activeSlice: ActiveRef | null;
  activeTask: ActiveRef | null;
  phase: Phase;
  recentDecisions: string[];
  blockers: string[];
  nextAction: string;
  activeWorkspace?: string;
  registry: MilestoneRegistryEntry[];
  requirements?: RequirementCounts;
  progress?: {
    milestones: { done: number; total: number };
    slices?: { done: number; total: number };
    tasks?: { done: number; total: number };
  };
}
```

`MilestoneRegistryEntry` tracks status across all milestones in the project:

```typescript
export interface MilestoneRegistryEntry {
  id: string;
  title: string;
  status: 'complete' | 'active' | 'pending' | 'parked';
  dependsOn?: string[];
}
```

Planning artifacts are typed with precise interfaces. `RoadmapSliceEntry` represents a slice in a milestone's roadmap:

```typescript
export interface RoadmapSliceEntry {
  id: string;       // e.g. "S01"
  title: string;
  risk: RiskLevel;  // 'low' | 'medium' | 'high'
  depends: string[]; // other slice IDs that must complete first
  done: boolean;
  demo: string;     // the "After this:" sentence
}
```

Hook configuration types support extensible pre- and post-unit behavior. `PostUnitHookConfig` fires after a unit type completes; `PreDispatchHookConfig` can intercept, modify, skip, or replace a unit before it dispatches.

### State Derivation (state.ts)

`deriveState()` is the authoritative state source — it reads all `.gsd/` files from disk and produces a `GSDState`. The function is memoized with a 100ms TTL within a single dispatch cycle:

```typescript
const CACHE_TTL_MS = 100;
let _stateCache: StateCache | null = null;

export async function deriveState(basePath: string): Promise<GSDState> {
  if (
    _stateCache &&
    _stateCache.basePath === basePath &&
    Date.now() - _stateCache.timestamp < CACHE_TTL_MS
  ) {
    return _stateCache.result;
  }
  // ... _deriveStateImpl()
}
```

The derivation algorithm performs a two-phase milestone scan:
1. **Phase 1** — build a roadmap cache and determine which milestones are complete. A milestone is complete when all its roadmap slices are marked done (`[x]`) and a `SUMMARY.md` exists.
2. **Phase 2** — build the registry, resolve dependency ordering, and identify the active milestone.

When the native Rust batch parser is available (`nativeBatchParseGsdFiles`), all `.md` files under `.gsd/` are read in a single call and cached in memory, eliminating O(N) individual `fs.readFile` calls.

The derivation checks for several intermediate states before reaching `executing`:
- `replanning-slice` — if any completed task summary has `blocker_discovered: true` and no `REPLAN.md` exists
- `summarizing` — if all tasks are done but the slice summary is missing
- `validating-milestone` — if all slices are done but no terminal validation file exists
- `completing-milestone` — if validation is terminal but no milestone summary exists

`invalidateStateCache()` must be called after any `.gsd/` file write. The `invalidateAllCaches()` function in `cache.ts` coordinates this across all three caches atomically:

```typescript
export function invalidateAllCaches(): void {
  invalidateStateCache(); // state.ts
  clearPathCache();       // paths.ts
  clearParseCache();      // files.ts
  clearArtifacts();       // gsd-db.ts
}
```

### Database Layer (gsd-db.ts)

The database provides a SQLite persistence layer with a provider fallback chain: `node:sqlite` (Node.js built-in, preferred) → `better-sqlite3` (npm) → null (graceful degradation when unavailable).

Schema version 3 defines five tables:

| Table | Purpose |
|-------|---------|
| `decisions` | Architectural decisions (D001, D002, ...) with supersession chain |
| `requirements` | Project requirements with status tracking and owner assignment |
| `artifacts` | Read-through cache of `.gsd/` markdown files (path-keyed) |
| `memories` | LLM-extracted insights from activity logs with confidence scores |
| `memory_processed_units` | Tracks which units have been processed for memory extraction |

The database is opened in WAL mode for file-backed instances. All tables expose `_active` views that filter out superseded rows:

```sql
CREATE VIEW active_decisions AS SELECT * FROM decisions WHERE superseded_by IS NULL;
CREATE VIEW active_requirements AS SELECT * FROM requirements WHERE superseded_by IS NULL;
CREATE VIEW active_memories AS SELECT * FROM memories WHERE superseded_by IS NULL;
```

The `artifacts` table is intentionally a cache. `clearArtifacts()` deletes all rows, forcing the next `deriveState()` call to re-read from the native Rust batch parser. Worktree operations use `copyWorktreeDb()` to snapshot the database and `reconcileWorktreeDb()` to merge changes back via SQLite's `ATTACH DATABASE` mechanism.

---

## Health and Verification

### Doctor System (doctor.ts, doctor-checks.ts, doctor-types.ts)

The doctor system audits the `.gsd/` directory for invariant violations and optionally repairs them. It is invoked via `/gsd doctor` and also runs proactively in `doctor-proactive.ts`.

Issue severity levels: `error` (blocks progress), `warning` (degraded state), `info` (cosmetic).

Fix levels control what the doctor will auto-repair:
- `task` — only mechanical fixes (create missing directories, update checksums)
- `all` — full repair including completion state transitions (create slice summaries, mark roadmap checkboxes)

Completion state transition codes (`all_tasks_done_missing_slice_summary`, `all_tasks_done_roadmap_not_checked`) are deliberately excluded from auto-fix when `fixLevel` is `"task"` — these belong to the dispatch lifecycle, not to mechanical bookkeeping.

**Git health checks** (`checkGitHealth`):
- Orphaned auto-worktrees for completed milestones (fixable: remove worktree)
- Stale milestone branches for completed milestones (fixable: delete branch)
- Corrupt merge/rebase state in `.git/` (fixable: `abortAndReset`)
- Runtime files tracked by git (fixable: `git rm --cached`)
- Legacy slice branches (`gsd/*/*`) from the old branching architecture (informational)

**Runtime health checks** (`checkRuntimeHealth`):
- Stale `auto.lock` whose PID is not running (fixable: delete lock)
- Stale parallel session status files (fixable: remove status file)
- Orphaned `completed-units.json` keys referencing missing artifacts (fixable: remove keys)
- Stale `hook-state.json` cycle counts from prior sessions (fixable: clear)
- Activity log bloat over 500 files or 100 MB (fixable: prune to 7-day retention)
- `STATE.md` missing or stale relative to derived state (fixable: rebuild)
- `.gitignore` missing critical runtime patterns (fixable: add patterns)

Slice-level checks audit each slice in each milestone's roadmap:
- Task marked done but missing summary (`task_done_missing_summary`)
- Task has summary but not marked done in plan (`task_summary_without_done_checkbox`)
- All tasks done but no slice summary (`all_tasks_done_missing_slice_summary`)
- All tasks done but no UAT file (`all_tasks_done_missing_slice_uat`)
- All tasks done but roadmap not checked (`all_tasks_done_roadmap_not_checked`)
- Slice checked in roadmap but no summary (`slice_checked_missing_summary`)
- `blocker_discovered: true` in a task summary but no `REPLAN.md` (`blocker_discovered_no_replan`)

Title validation rejects milestone and slice titles containing `—` (em dash), `–` (en dash), or `/` (forward slash) — these conflict with GSD's state document delimiter conventions.

### Verification Gate (verification-gate.ts)

The verification gate runs executable checks to confirm task work is correct before marking the task done. Command discovery uses a first-non-empty-wins strategy:

1. **Preference commands** — from `verification_commands` in `preferences.md`
2. **Task plan verify field** — from the `- Verify:` subline of the task plan entry, split on `&&`
3. **package.json scripts** — probes for `typecheck`, `lint`, `test` in that order
4. **None** — gate passes vacuously (no commands to run)

Commands from the task plan are sanitized before execution. `isLikelyCommand()` uses heuristics to reject English prose that was mistakenly placed in a `Verify:` field:

```typescript
const KNOWN_COMMAND_PREFIXES = new Set([
  "npm", "npx", "yarn", "pnpm", "bun", "bunx",
  "node", "ts-node", "tsx", "tsc",
  "eslint", "prettier", "vitest", "jest", "pytest",
  // ... etc
]);
```

`runVerificationGate()` executes all discovered commands sequentially via `spawnSync`, capturing stdout and stderr (truncated to 10 KB each). Output is captured in a `VerificationResult`:

```typescript
export interface VerificationResult {
  passed: boolean;
  checks: VerificationCheck[];
  discoverySource: "preference" | "task-plan" | "package-json" | "none";
  timestamp: number;
  runtimeErrors?: RuntimeError[];
  auditWarnings?: AuditWarning[];
}
```

`captureRuntimeErrors()` additionally scans bg-shell processes and browser console logs. Severity classification: `crashed` status or non-zero exit on a dead process → blocking crash; alive process with recent errors → non-blocking error; browser console unhandled rejection → blocking crash.

`runDependencyAudit()` triggers `npm audit` when `git diff --name-only HEAD` shows changes to top-level dependency files (`package.json`, `package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `bun.lockb`).

### Verification Evidence (verification-evidence.ts)

`writeVerificationJSON()` persists a `T##-VERIFY.json` artifact alongside task files. The JSON uses schema version 1 and intentionally omits stdout/stderr to keep file sizes bounded. The evidence record includes:

```typescript
export interface EvidenceJSON {
  schemaVersion: 1;
  taskId: string;
  unitId: string;
  timestamp: number;
  passed: boolean;
  discoverySource: string;
  checks: EvidenceCheckJSON[];  // command, exitCode, durationMs, verdict
  retryAttempt?: number;
  maxRetries?: number;
  runtimeErrors?: RuntimeErrorJSON[];
  auditWarnings?: AuditWarningJSON[];
}
```

`formatEvidenceTable()` renders a five-column markdown table suitable for inclusion in task summaries and doctor output.

---

## Preferences System

### GSDPreferences (preferences-types.ts)

Preferences are stored in YAML frontmatter inside `preferences.md` files. Two locations are supported: `~/.gsd/preferences.md` (global) and `<project>/.gsd/preferences.md` (project). Project preferences are merged on top of global with array fields deduplicated.

All recognized top-level preference keys:

| Key | Type | Description |
|-----|------|-------------|
| `version` | `number` | Schema version for forward-compatibility |
| `mode` | `"solo" \| "team"` | Workflow mode — applies defaults for each mode |
| `always_use_skills` | `string[]` | Skills loaded unconditionally when relevant |
| `prefer_skills` | `string[]` | Skills preferred but not mandatory |
| `avoid_skills` | `string[]` | Skills avoided unless clearly needed |
| `skill_rules` | `GSDSkillRule[]` | Conditional skill rules with `when` conditions |
| `custom_instructions` | `string[]` | Additional LLM instructions injected into system prompt |
| `models` | `GSDModelConfig \| GSDModelConfigV2` | Per-phase model overrides |
| `skill_discovery` | `"auto" \| "suggest" \| "off"` | Skill auto-discovery mode |
| `skill_staleness_days` | `number` | Days until unused skills are deprioritized (0 = disabled) |
| `auto_supervisor` | `AutoSupervisorConfig` | Supervisor model config (model, timeouts) |
| `uat_dispatch` | `boolean` | Enable UAT dispatch after slice completion |
| `unique_milestone_ids` | `boolean` | Append random suffix to milestone IDs (team mode default) |
| `budget_ceiling` | `number` | Maximum USD spend before enforcement action |
| `budget_enforcement` | `"warn" \| "pause" \| "halt"` | Action taken when budget ceiling is reached |
| `context_pause_threshold` | `number` | Context window percentage to trigger continue-here |
| `notifications` | `NotificationPreferences` | Notification channel configuration |
| `remote_questions` | `RemoteQuestionsConfig` | Slack/Discord/Telegram channel for remote Q&A |
| `git` | `GitPreferences` | Git integration settings |
| `post_unit_hooks` | `PostUnitHookConfig[]` | Hooks fired after unit completion |
| `pre_dispatch_hooks` | `PreDispatchHookConfig[]` | Hooks fired before unit dispatch |
| `dynamic_routing` | `DynamicRoutingConfig` | Complexity-based model downgrade config |
| `token_profile` | `"budget" \| "balanced" \| "quality"` | Token usage profile |
| `phases` | `PhaseSkipPreferences` | Phase skip flags (e.g. `skip_research`) |
| `parallel` | `ParallelConfig` | Parallel multi-milestone orchestration config |
| `verification_commands` | `string[]` | Explicit verification commands (highest priority) |
| `verification_auto_fix` | `boolean` | Auto-fix verification failures when possible |
| `verification_max_retries` | `number` | Max verification retry attempts |
| `search_provider` | `"brave" \| "tavily" \| "ollama" \| "native" \| "auto"` | Search backend selection |
| `compression_strategy` | `"truncate" \| "compress"` | Context overflow strategy |
| `context_selection` | `"full" \| "smart"` | File inlining mode |

### Mode Defaults

`WorkflowMode` applies preset configurations as the lowest-priority layer:

```typescript
export const MODE_DEFAULTS: Record<WorkflowMode, Partial<GSDPreferences>> = {
  solo: {
    git: { auto_push: true, push_branches: false, pre_merge_check: false,
           merge_strategy: "squash", isolation: "worktree", commit_docs: true },
    unique_milestone_ids: false,
  },
  team: {
    git: { auto_push: false, push_branches: true, pre_merge_check: true,
           merge_strategy: "squash", isolation: "worktree", commit_docs: true },
    unique_milestone_ids: true,
  },
};
```

### Model Preferences

`GSDModelConfigV2` supports per-phase primary models with fallback chains:

```typescript
export interface GSDPhaseModelConfig {
  model: string;      // primary model ID
  provider?: string;  // disambiguates when the same ID exists across providers
  fallbacks?: string[]; // ordered fallbacks for rate limit / credits exhausted
}
```

Phases recognized: `research`, `planning`, `execution`, `execution_simple`, `completion`, `subagent`.

---

## Model Routing

### Dynamic Model Router (model-router.ts)

The model router implements downgrade-only semantics: the configured model for a phase is the capability ceiling. Based on task complexity classification, a cheaper model from a lower tier may be substituted. The router never upgrades beyond the user's configuration.

Three complexity tiers: `light`, `standard`, `heavy`. The capability tier of any model is looked up against a static registry:

```typescript
const MODEL_CAPABILITY_TIER: Record<string, ComplexityTier> = {
  "claude-haiku-4-5": "light",
  "claude-3-5-haiku-latest": "light",
  "claude-sonnet-4-6": "standard",
  "claude-opus-4-6": "heavy",
  "gpt-4o-mini": "light",
  "gpt-4o": "standard",
  "gemini-2.0-flash": "light",
  "gemini-2.5-pro": "standard",
  // ...
};
```

`resolveModelForComplexity()` returns a `RoutingDecision` that includes the selected model, a fallback chain, the complexity tier, whether a downgrade occurred, and a human-readable reason.

When a downgrade is made, the fallback chain is structured to escalate back if the cheaper model fails:

```
fallbacks = [user_configured_fallbacks..., configured_primary]
```

This means failures at the downgraded tier automatically escalate back to the configured primary. `escalateTier()` supports explicit tier promotion after verified failure.

### Provider Error Handling (provider-error-pause.ts)

Provider errors (rate limits, credits exhausted, temporary outages) are handled separately from routing. When a provider error is detected, the extension can pause auto-mode, record the error, and optionally retry with the next fallback model.

---

## Worktree System

### Worktree Utilities (worktree.ts)

GSD uses git worktrees to isolate milestone work. Each milestone gets its own worktree under `.gsd/worktrees/<name>/` with a corresponding `worktree/<name>` branch.

`detectWorktreeName()` identifies whether the current path is inside a GSD worktree by looking for the `/.gsd/worktrees/` path segment:

```typescript
export function detectWorktreeName(basePath: string): string | null {
  const marker = "/.gsd/worktrees/";
  const idx = normalizedPath.indexOf(marker);
  if (idx === -1) return null;
  return normalizedPath.slice(idx + marker.length).split("/")[0] || null;
}
```

`resolveProjectRoot()` strips the worktree suffix so commands that call `process.cwd()` always operate against the real project root, not a worktree subdirectory.

Slice branch names are namespaced by worktree to prevent git conflicts when multiple worktrees work on the same milestone/slice IDs:

```
Main tree:   gsd/M001/S01
In worktree: gsd/<worktreeName>/M001/S01
```

The integration branch (the branch slice work merges back into) is recorded in a `<MID>-META.json` file when auto-mode starts. This ensures that if a user was on a feature branch when auto-mode began, slice branches merge back to that feature branch and not to `main`.

### Worktree Management (worktree-manager.ts)

`createWorktree()` creates a new git worktree under `.gsd/worktrees/<name>/`:

1. Validates the name (alphanumeric, hyphens, underscores only)
2. Prunes stale worktree entries from previous removals
3. Determines the start point (explicit or auto-detected main branch)
4. If the branch already exists but is not in use, force-resets it to the start point (unless `reuseExistingBranch` is set — used when resuming auto-mode)
5. Calls `git worktree add` via the native bridge

`diffWorktreeGSD()` diffs the `.gsd/` directory between a worktree branch and the main branch, returning a `WorktreeDiffSummary` (added/modified/removed file lists) with runtime artifacts excluded:

```typescript
const SKIP_PATHS = [".gsd/worktrees/", ".gsd/runtime/", ".gsd/activity/"];
const SKIP_EXACT = [".gsd/STATE.md", ".gsd/auto.lock", ".gsd/metrics.json"];
```

`mergeWorktreeToMain()` performs a squash merge of the worktree branch into the main branch, requiring the caller to be on the main branch first.

`reconcileWorktreeDb()` in `gsd-db.ts` merges the worktree's SQLite database back into the main database using `ATTACH DATABASE`. It detects and reports conflicts where both databases modified the same row, then applies `INSERT OR REPLACE` for all three tables (decisions, requirements, artifacts).

---

## Visualization and Export

### TUI Visualizer (visualizer-views.ts, visualizer-data.ts)

The TUI visualizer is a multi-tab overlay that renders live project data in the terminal. The `loadVisualizerData()` function aggregates from multiple sources:

```typescript
export interface VisualizerData {
  milestones: VisualizerMilestone[];   // roadmap hierarchy
  phase: Phase;                         // current workflow phase
  totals: ProjectTotals | null;         // aggregate cost/token/unit totals
  byPhase: PhaseAggregate[];            // cost/tokens broken down by phase
  bySlice: SliceAggregate[];            // cost/tokens per slice
  byModel: ModelAggregate[];            // cost per model
  byTier: TierAggregate[];              // cost per complexity tier
  units: UnitMetrics[];                 // per-unit execution history
  criticalPath: CriticalPathInfo;       // topological critical path analysis
  agentActivity: AgentActivityInfo | null; // live auto-mode status
  changelog: ChangelogInfo;             // completed slice summaries
  sliceVerifications: SliceVerification[]; // verification results from summaries
  knowledge: KnowledgeInfo;             // KNOWLEDGE.md rules/patterns/lessons
  captures: CapturesInfo;               // pending captures
  health: HealthInfo;                   // budget, pressure, routing stats
  discussion: VisualizerDiscussionState[]; // milestone discussion state
  stats: VisualizerStats;               // missing/updated slice counts
}
```

The visualizer uses mtime-based file caching to avoid re-parsing unchanged roadmap and plan files between tab switches.

**Tabs rendered by `visualizer-views.ts`:**

- **Progress** — risk heatmap, feature snapshot (missing/updated slices), milestone registry with slice and task trees, verification badges per slice
- **Dependencies** — milestone dependency graph, slice dependency graph for the active milestone, critical path analysis, data flow (provides/requires)
- **Metrics** — cost/token/unit totals, bar charts by phase and model, tier breakdown, cost projections and burn rate
- **Timeline** — chronological unit execution history; Gantt chart for terminals ≥ 90 columns, list view for narrow terminals
- **Agent** — live auto-mode status, progress bar, completion rate, budget pressure indicators
- **Changelog** — completed slice summaries with one-liners, file modifications, key decisions, patterns
- **Knowledge** — KNOWLEDGE.md rules, patterns, lessons from structured markdown tables
- **Captures** — pending/triaged/resolved captures grouped by status
- **Health** — budget ceiling vs. spend, context pressure metrics, tier routing breakdown
- **Export** — export options (markdown, JSON, plain text snapshot)

Critical path analysis uses Kahn's topological sort algorithm, weighted by remaining incomplete slices for milestones and incomplete slices for the within-milestone slice path.

### HTML Export (export-html.ts)

`generateHtmlReport()` produces a single self-contained HTML file with all CSS and JavaScript inlined (no external dependencies). The report is printable to PDF from any browser and includes:

- Branding header with project name, path, GSD version, and generation timestamp
- Project summary and overall progress
- Progress tree (milestones → slices → tasks, with critical path markers)
- Execution timeline (chronological unit history)
- Slice dependency graph as SVG DAGs per milestone
- Cost and token metrics with bar charts (phase/slice/model/tier breakdowns)
- Health and configuration overview
- Changelog (completed slice summaries with file modifications)
- Knowledge base (rules, patterns, lessons)
- Captures log
- Artifacts and milestone planning/discussion state

The design follows a restrained palette without emoji, making it suitable for formal project status reports.

---

## Infrastructure

### Cache System (cache.ts)

Three module-scoped caches are maintained across the extension:

1. **State cache** (`state.ts`) — memoizes `deriveState()` for 100ms TTL per basePath
2. **Path cache** (`paths.ts`) — memoizes `readdirSync` results for directory listings
3. **Parse cache** (`files.ts`) — memoizes parsed markdown file results keyed by path

The `invalidateAllCaches()` function in `cache.ts` clears all three plus the SQLite artifacts cache atomically. This function must be called after any `.gsd/` file write. Not calling it after file mutations was the root cause of bugs #431 and #793, where stale reads caused incorrect phase detection and infinite skip loops.

### Crash Recovery (crash-recovery.ts)

Auto-mode writes a lock file at `.gsd/auto.lock` at session start and updates it before each unit dispatch. The lock records:

```typescript
export interface LockData {
  pid: number;
  startedAt: string;
  unitType: string;        // current unit type
  unitId: string;          // current unit ID
  unitStartedAt: string;
  completedUnits: number;  // progress counter
  sessionFile?: string;    // path to the pi session JSONL
}
```

On the next startup, if `auto.lock` exists, `isLockProcessAlive()` checks whether the PID is still running via `process.kill(pid, 0)`. If the PID is dead (ESRCH), the lock is stale — the session crashed. The doctor system detects stale locks and offers to clear them. The `sessionFile` field allows session forensics to read the surviving JSONL and reconstruct what happened up to the crash point.

Lock writes use `atomicWriteSync` (write to temp file, then rename) to prevent corrupt lock files from partial writes.

### Context Budget (context-budget.ts)

The context budget engine allocates portions of the model's context window to different sections of the assembled prompt. Ratios are module-level constants:

```typescript
const SUMMARY_RATIO = 0.15;         // prior-task summaries
const INLINE_CONTEXT_RATIO = 0.40;  // inline context (plans, decisions, code)
const VERIFICATION_RATIO = 0.10;    // verification sections
const CONTINUE_THRESHOLD_PERCENT = 70; // trigger continue-here checkpoint
```

`computeBudgets()` returns character budgets (not token counts) for each category, along with a `taskCountRange` scaled to context window size:

| Context window | Max tasks per slice |
|----------------|---------------------|
| 500K+ tokens | 8 |
| 200K+ tokens | 6 |
| 128K+ tokens | 5 |
| < 128K tokens | 3 |

`truncateAtSectionBoundary()` enforces section-boundary truncation (D003 decision): content is split at `### ` headings and `---` dividers, then sections are kept greedily until the budget is exhausted. Mid-section cuts are not permitted. Dropped sections are replaced with a `[...truncated N sections]` marker so the LLM knows content was omitted.

`resolveExecutorContextWindow()` determines the effective context window for budget calculations via a three-step fallback:
1. Look up the configured execution model in the model registry
2. Fall back to the current session's reported context window
3. Fall back to the 200K default (D002 decision)

When the `compression_strategy` preference is `"compress"`, `reduceToFit()` first applies heuristic compression via `compressToTarget()` before falling back to section-boundary truncation.

### Git Service (git-service.ts)

`GitServiceImpl` is the stateful git operations class. It implements smart staging that excludes GSD runtime files:

```typescript
export const RUNTIME_EXCLUSION_PATHS: readonly string[] = [
  ".gsd/activity/",
  ".gsd/runtime/",
  ".gsd/worktrees/",
  ".gsd/auto.lock",
  ".gsd/metrics.json",
  ".gsd/completed-units.json",
  ".gsd/STATE.md",
  ".gsd/gsd.db",
  ".gsd/DISCUSSION-MANIFEST.json",
];
```

Smart staging runs `git add -A` then `git reset HEAD <exclusion>` for each runtime path. This approach (rather than pathspec excludes) avoids a git failure mode where `:(exclude)` patterns fail when `.gsd/` is in `.gitignore`.

`getMainBranch()` resolves the integration branch using a four-step priority chain:
1. Explicit `main_branch` preference
2. Recorded integration branch from `<MID>-META.json`
3. Worktree base branch (`worktree/<name>`)
4. Auto-detected repo default (`origin/HEAD` → `main` → `master` → current branch)

`GitPreferences` exposes `isolation` to control git isolation strategy:
- `"worktree"` (default) — creates a milestone worktree for isolated work
- `"branch"` — works directly in the project root (for submodule-heavy repos)
- `"none"` — no isolation, commits land on the user's current branch

Conventional commit messages are generated by `buildTaskCommitMessage()` using a one-liner from the task summary and keyword-based commit type inference (`fix`, `refactor`, `docs`, `test`, `perf`, `chore`, default `feat`).
