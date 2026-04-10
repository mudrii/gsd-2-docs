# GSD Configuration and Preferences

## Overview

GSD preferences control model selection, timeout behavior, budget limits, verification commands, git isolation, and other auto-mode settings. They are stored in YAML frontmatter within markdown files and managed via `/gsd prefs`.

**Files:**
- `src/resources/extensions/gsd/preferences.ts` -- loading, merging, rendering
- `src/resources/extensions/gsd/preferences-types.ts` -- type definitions, constants
- `src/resources/extensions/gsd/preferences-validation.ts` -- validation
- `src/resources/extensions/gsd/preferences-models.ts` -- model resolution, fallbacks
- `src/resources/extensions/gsd/preferences-skills.ts` -- skill reference resolution

---

## Preference File Locations

| Scope | Path | Applies To |
|-------|------|-----------|
| Global | `~/.gsd/PREFERENCES.md` | All projects |
| Global (legacy) | `~/.gsd/preferences.md` | All projects (fallback) |
| Global (legacy) | `~/.pi/agent/gsd-preferences.md` | All projects (fallback) |
| Project | `.gsd/PREFERENCES.md` | Current project only |

The canonical filename was renamed from `preferences.md` to `PREFERENCES.md` in v2.52.0 for consistency with other uppercase GSD artifacts (`DECISIONS.md`, `KNOWLEDGE.md`, etc.). GSD still loads lowercase variants as a fallback, so existing files continue to work. New files created by the preferences wizard use the uppercase name.

### File Format

Preferences use YAML frontmatter in a markdown file:

```yaml
---
version: 1
token_profile: balanced
budget_ceiling: 25.00
models:
  execution: claude-sonnet-4-6
---
```

Only the YAML frontmatter block (between `---` delimiters) is parsed. Content after the closing `---` is ignored.

---

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GSD_HOME` | `~/.gsd` | Override the global GSD directory. All paths derive from this unless individually overridden. Affects preferences, skills, sessions, and per-project state. (v2.39) |
| `GSD_PROJECT_ID` | (auto-hash) | Override the automatic project identity hash. Per-project state goes to `$GSD_HOME/projects/<GSD_PROJECT_ID>/` instead of the computed hash. Useful for CI/CD or sharing state across clones of the same repo. (v2.39) |
| `GSD_STATE_DIR` | `$GSD_HOME` | Per-project state root. Controls where `projects/<repo-hash>/` directories are created. Takes precedence over `GSD_HOME` for project state. |
| `GSD_CODING_AGENT_DIR` | `$GSD_HOME/agent` | Agent directory containing managed resources, extensions, and auth. Takes precedence over `GSD_HOME` for agent paths. |
| `GSD_PARALLEL_WORKER` | (unset) | Set to `"1"` in parallel worker subprocesses (both milestone-level and slice-level). Used internally to prevent nesting and to signal worker context. (v2.64.0) |
| `GSD_PERSIST_WRITE_GATE_STATE` | (unset) | Set to `"1"` to enable write gate state persistence. Used by the workflow MCP server to persist write gate snapshots across sessions. (v2.66.0) |
| `GSD_RTK_DISABLED` | (unset) | Set to `"1"` to disable RTK integration regardless of `experimental.rtk` preference. Environment variable takes precedence over preferences for manual override. |
| `OLLAMA_HOST` | `http://localhost:11434` | Override the Ollama server URL. Respected by the native Ollama provider extension for non-default endpoints. (v2.59.0) |
| `CONTEXT7_API_KEY` | (unset) | Context7 API key for higher rate limits on library documentation lookups. Can also be set via `/gsd config`. |

---

## Loading and Merging

`loadEffectiveGSDPreferences()` is the primary entry point. It:

1. Loads global preferences (`loadGlobalGSDPreferences()`)
2. Loads project preferences (`loadProjectGSDPreferences()`)
3. Merges them (project overrides global)
4. Applies workflow mode defaults as the lowest-priority layer

### Merge Behavior

| Field Type | Strategy |
|-----------|----------|
| Scalar (`skill_discovery`, `budget_ceiling`, `token_profile`) | Project wins if defined |
| Array (`always_use_skills`, `verification_commands`, `custom_instructions`) | Concatenated, deduplicated (global first, then project) |
| Object (`models`, `git`, `auto_supervisor`, `notifications`) | Shallow-merged, project overrides per key |
| Hook arrays (`post_unit_hooks`, `pre_dispatch_hooks`) | Merged by name -- project hooks with the same name replace global hooks |

---

## Workflow Modes

Set via `mode` preference or `/gsd mode` command.

| Mode | `git.auto_push` | `git.push_branches` | `git.pre_merge_check` | `unique_milestone_ids` | `git.isolation` |
|------|:---------------:|:-------------------:|:--------------------:|:---------------------:|:---------------:|
| `solo` | true | false | `"auto"` | false | `"none"` |
| `team` | false | true | true | true | `"none"` |

Mode defaults are the lowest-priority layer -- any explicit user value overrides them.

---

## All Preferences

### `version`

Schema version. Currently `1`.

### `mode`

Workflow mode: `"solo"` or `"team"`. Applies coordinated defaults for git, milestone IDs, and documentation.

### `models`

Per-phase model selection. Each key accepts a model string or an object with fallbacks.

**Phases:**

| Phase Key | Maps to Unit Types |
|-----------|-------------------|
| `research` | `research-milestone`, `research-slice` |
| `planning` | `plan-milestone`, `plan-slice`, `replan-slice` |
| `discuss` | `discuss-milestone`, `discuss-slice` |
| `execution` | `execute-task` |
| `execution_simple` | `execute-task` when classified as simple by complexity router |
| `completion` | `complete-slice`, `complete-milestone`, `reassess-roadmap`, `run-uat`, `validate-milestone` |
| `validation` | `validate-milestone` |
| `subagent` | Delegated subagent tasks (scout, researcher, worker) |

**Simple string format:**

```yaml
models:
  research: claude-sonnet-4-6
  planning: claude-opus-4-6
  execution: claude-sonnet-4-6
```

**Object format with fallbacks:**

```yaml
models:
  planning:
    model: claude-opus-4-6
    provider: bedrock
    fallbacks:
      - openrouter/z-ai/glm-5
      - openrouter/moonshotai/kimi-k2.5
```

When a model fails to switch (provider unavailable, rate limited, credits exhausted), GSD automatically tries the next model in the `fallbacks` list. The fallback chain also handles transient network errors (connection reset, timeout, DNS) with up to 2 retries per model before switching.

**Provider targeting:** Use `provider/model` format (e.g., `bedrock/claude-sonnet-4-6`) or the `provider` field in object format.

Omitting a key causes that phase to use whatever model is currently active.

**Dynamic routing without `models` section (v2.55.0):** When `dynamic_routing.enabled` is true, the router works even if no `models` section is defined. It uses the currently active model as the ceiling and applies tier-based or capability-aware downgrading from there.

### `token_profile`

Coordinates model selection, phase skipping, and context compression.

| Value | Default | Behavior |
|-------|:-------:|----------|
| `budget` | | Skips milestone + slice research phases, uses cheaper models, minimal context inlining |
| `balanced` | Yes | All phases run (except reassess which is opt-in), standard model selection, standard context inlining |
| `quality` | | All phases run, prefers higher-quality models, full context inlining |

The token profile sets defaults for several other preferences. Explicit values in preferences always override profile defaults.

### `phases`

Fine-grained control over which phases run in auto mode. Usually set automatically by `token_profile`, but can be overridden explicitly.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `skip_research` | boolean | false | Skip milestone-level research |
| `skip_reassess` | boolean | false | Skip roadmap reassessment |
| `skip_slice_research` | boolean | true | Skip per-slice research |
| `skip_milestone_validation` | boolean | false | Skip milestone validation (writes pass-through VALIDATION) |
| `reassess_after_slice` | boolean | false | Fire reassess-roadmap after each slice completion (opt-in) |
| `require_slice_discussion` | boolean | false | Pause auto-mode before each slice for human discussion |

### `skill_discovery`

Controls how GSD finds and applies skills during auto mode.

| Value | Behavior |
|-------|----------|
| `auto` | Skills found and applied automatically |
| `suggest` | Skills identified during research but not auto-installed (default) |
| `off` | Skill discovery disabled |

### `skill_staleness_days`

Skills unused for N days get deprioritized. Default: `60`. Set `0` to disable.

### `auto_supervisor`

Timeout thresholds for auto mode supervision.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | string | (active model) | Model for supervisor interventions |
| `soft_timeout_minutes` | number | `20` | Warn LLM to wrap up |
| `idle_timeout_minutes` | number | `10` | Detect stalls (no tool calls) |
| `hard_timeout_minutes` | number | `30` | Pause auto mode |

### `budget_ceiling`

Maximum USD to spend during auto mode. No `$` sign -- just the number.

```yaml
budget_ceiling: 50.00
```

Alert thresholds fire at 75%, 80%, 90%, and 100%. Desktop notifications are sent at 80%+.

### `budget_enforcement`

How the budget ceiling is enforced when 100% is reached.

| Value | Behavior |
|-------|----------|
| `warn` | Log a warning but continue |
| `pause` | Pause auto mode; `/gsd auto` to override and continue (default when ceiling is set) |
| `halt` | Stop auto mode entirely |

### `context_pause_threshold`

Context window usage percentage (0-100) at which auto mode pauses. Default: `0` (disabled).

```yaml
context_pause_threshold: 80
```

### `show_token_cost` (v2.44.0)

Opt-in: show per-prompt and cumulative session token cost in the footer.

```yaml
show_token_cost: true    # default: false
```

When enabled, each prompt response displays the token cost for that turn and a running total for the session.

### `verification_commands`

Shell commands that run automatically after every task execution.

```yaml
verification_commands:
  - npm run lint
  - npm run test
```

Commands from preferences are blocking -- failures prevent advancement.

Discovery priority when no preference commands are set:
1. Task plan `verify` field (blocking)
2. `package.json` scripts: `typecheck`, `lint`, `test` (advisory, non-blocking)

### `verification_auto_fix`

When true (default), verification failures trigger auto-fix retries. The agent sees the failure output and attempts to fix issues before advancing.

### `verification_max_retries`

Maximum auto-fix retry attempts. Default: `2`.

### `search_provider` (v2.29.0)

Search provider preference for web search during research phases.

| Value | Behavior |
|-------|----------|
| `"auto"` | Default behavior (native for Anthropic, Brave/Tavily for others) |
| `"brave"` | Force Brave Search backend |
| `"tavily"` | Force Tavily Search backend |
| `"ollama"` | Use Ollama for search |
| `"native"` | Force native Anthropic search only |

### `git`

Git behavior configuration. All fields optional.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `isolation` | string | `"none"` | Auto-mode isolation strategy (changed from `"worktree"` in v2.46.0) |
| `auto_push` | boolean | `false` | Push commits to remote after committing |
| `push_branches` | boolean | `false` | Push milestone branch to remote |
| `remote` | string | `"origin"` | Git remote name |
| `snapshots` | boolean | `false` | WIP snapshot commits during long tasks |
| `pre_merge_check` | bool/string | `false` | Run checks before merge (`true`/`false`/`"auto"`) |
| `commit_type` | string | (inferred) | Override conventional commit prefix |
| `main_branch` | string | `"main"` | Primary branch name |
| `merge_strategy` | string | `"squash"` | How worktree branches merge: `"squash"` or `"merge"` |
| `commit_docs` | boolean | `true` | Commit `.gsd/` artifacts to git |
| `manage_gitignore` | boolean | `true` | When false, GSD will not modify `.gitignore` |
| `worktree_post_create` | string | (none) | Script to run after worktree creation |
| `auto_pr` | boolean | `false` | Create PR on milestone completion (v2.39.0; requires `gh` CLI) |
| `pr_target_branch` | string | (main branch) | Target branch for auto-created PRs |

#### Git Isolation Modes

| Mode | Description |
|------|-------------|
| `none` (default since v2.46.0) | Work directly on current branch. No worktree or milestone branch. For hot-reload workflows. |
| `worktree` | Each milestone runs in `.gsd/worktrees/<MID>/` on a `milestone/<MID>` branch. Squash-merged to main on completion. |
| `branch` | Work in project root on a `milestone/<MID>` branch. Useful for submodule-heavy repos. |

The default isolation mode was changed from `worktree` to `none` in v2.46.0. If worktree creation fails at runtime (e.g., filesystem limitations, git submodule issues), GSD automatically downgrades to `none` isolation rather than aborting (v2.50.0).

Isolation mode is resolved by `getIsolationMode()` in `preferences.ts`.

#### `git.auto_pr` (v2.39.0)

Automatically create a pull request when a milestone completes. Designed for teams using Gitflow or branch-based workflows where work should go through PR review before merging to a target branch.

```yaml
git:
  auto_push: true
  auto_pr: true
  pr_target_branch: develop
```

**Requirements:**
- `auto_push: true` -- the milestone branch must be pushed before a PR can be created
- [`gh` CLI](https://cli.github.com/) installed and authenticated (`gh auth login`)

**How it works:**
1. Milestone completes -> GSD squash-merges the worktree to the main branch
2. Pushes the main branch to remote (if `auto_push: true`)
3. Pushes the milestone branch to remote
4. Creates a PR from the milestone branch to `pr_target_branch` via `gh pr create`

If `pr_target_branch` is not set, the PR targets the `main_branch` (or auto-detected main branch). PR creation failure is non-fatal -- GSD logs and continues.

#### Worktree Post-Create Hook

Script to run after worktree creation. Receives `SOURCE_DIR` and `WORKTREE_DIR` environment variables. Runs with a 30-second timeout. Failure is non-fatal.

```yaml
git:
  worktree_post_create: .gsd/hooks/post-worktree-create
```

### `dynamic_routing`

Complexity-based model routing. When enabled, auto mode selects cheaper models for simple tasks and reserves expensive models for complex work.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `true` | Master toggle (enabled by default since v2.59.0) |
| `capability_routing` | boolean | `true` | Enable capability-aware model scoring (v2.62.0). When enabled and multiple models are eligible, ranks by task-capability match instead of cheapest-in-tier. |
| `tier_models.light` | string | (auto-detected) | Model for simple tasks |
| `tier_models.standard` | string | (auto-detected) | Model for standard tasks |
| `tier_models.heavy` | string | (auto-detected) | Model for complex tasks |
| `escalate_on_failure` | boolean | `true` | Escalate tier on failure |
| `budget_pressure` | boolean | `true` | Bias toward cheaper models under budget pressure |
| `cross_provider` | boolean | `true` | Consider models across providers |
| `hooks` | boolean | `true` | Apply routing to hook units |

**Downgrade-only semantics:** the router never upgrades beyond the user's configured model. It can only select equal or cheaper alternatives.

**Dynamic routing without `models` (v2.55.0):** You can enable `dynamic_routing.enabled: true` without defining a `models` section. The router uses the currently active model as the ceiling and downgrades from there based on complexity classification.

Known model tier assignments (from `model-router.ts`):

| Tier | Models |
|------|--------|
| Light | claude-haiku-4-5, gpt-4o-mini, gemini-2.0-flash |
| Standard | claude-sonnet-4-6, gpt-4o, gemini-2.5-pro, deepseek-chat |
| Heavy | claude-opus-4-6, o1, o3 |

#### Capability-Aware Model Routing (ADR-004)

Starting with v2.52.0, the model routing architecture evolved from a pure cost/tier approach to a **capability-aware** two-dimensional system that combines complexity classification ("how hard") with capability scoring ("what kind"). This is defined in ADR-004. As of v2.62.0, the full pipeline is wired: task metadata extraction, capability scoring, `before_model_select` hook, and verbose scoring output.

**Key concepts:**

- Each model gains an optional **capability profile** with seven dimensions: `coding`, `debugging`, `research`, `reasoning`, `speed`, `longContext`, `instruction` (scores 0-100).
- Each dispatched unit gets a **task requirement vector** computed from `(unitType, TaskMetadata)`, not just unit type alone.
- The router filters to tier-eligible models (downgrade-only), then ranks by capability score when multiple models are eligible.
- Models without capability profiles receive uniform scores (50 in each dimension), falling back to cheapest-in-tier behavior.
- All existing invariants are preserved: downgrade-only semantics, budget pressure, retry escalation, fallback chains.

**`before_model_select` hook (v2.62.0):** Extensions can register a `before_model_select` hook to override model selection entirely. The hook receives the classification result, task metadata, and eligible models. Return `{ modelId: "..." }` to override, or `undefined` to let built-in capability scoring proceed.

**Flat-rate provider routing guard (v2.64.0):** Dynamic model routing is automatically disabled for flat-rate providers (e.g., GitHub Copilot) where all models cost the same per request. Downgrading to a cheaper model would degrade quality without providing cost benefit.

**Budget pressure downgrading:** When budget usage exceeds thresholds, the router biases toward cheaper models in the eligible tier. This works alongside the `budget_ceiling` and `budget_enforcement` preferences.

**Escalation on retry failure:** When a task fails and retries, the router escalates to the next higher tier (if `escalate_on_failure` is enabled) before attempting the retry.

**Provider registry (`models.custom.ts`):** Models from providers not tracked by the auto-generated models.dev catalog are defined in `packages/pi-ai/src/models.custom.ts`. This includes Alibaba Coding Plan models (Qwen, GLM, Kimi K2.5, MiniMax M2.5) and Z.AI models. GLM-5.1 was added to the Z.AI provider in v2.57.0.

**Unit type mapping for discuss dispatches (v2.57.0):** The model router now uses honest `unitTypes` for `discuss-milestone` and `discuss-slice` dispatches instead of aliasing them to planning unit types. This ensures discuss phases resolve to the correct model phase when per-phase models are configured.

**Non-API-key provider support (v2.44.0):** GSD supports provider extensions that do not use traditional API keys, such as the Claude Code CLI provider. These providers implement their own authentication flow (e.g., OAuth, CLI token) and are surfaced in the model picker alongside standard providers.

Users can override capability profiles in `models.json`:

```json
{
  "providers": {
    "anthropic": {
      "modelOverrides": {
        "claude-sonnet-4-6": {
          "capabilities": {
            "debugging": 90,
            "research": 85
          }
        }
      }
    }
  }
}
```

### `service_tier` (v2.42.0)

OpenAI service tier preference for supported models. Toggle with `/gsd fast`.

| Value | Behavior |
|-------|----------|
| `"priority"` | Priority tier -- 2x cost, faster responses |
| `"flex"` | Flex tier -- 0.5x cost, slower responses |
| (unset) | Default tier |

```yaml
service_tier: priority
```

### `forensics_dedup` (v2.43.0)

Opt-in: search existing issues and PRs before filing from `/gsd forensics`. Uses additional AI tokens.

```yaml
forensics_dedup: true    # default: false
```

### `uat_dispatch`

Enable automatic UAT (User Acceptance Test) runs after slice completion. Default: `false`.

### `unique_milestone_ids`

Generate milestone IDs with a random 6-character suffix to avoid collisions in team workflows.

```yaml
unique_milestone_ids: true
# Produces: M001-eh88as instead of M001
```

### `auto_report`

Auto-generate self-contained HTML reports after milestone completion. Default: `true`.

Reports are written to `.gsd/reports/` with embedded CSS/JS (no external dependencies).

### `auto_visualize`

Show the workflow visualizer automatically after milestone completion. Default: `false`.

### `notifications`

Control desktop notifications during auto mode.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `true` | Master toggle |
| `on_complete` | boolean | `true` | Notify on unit completion |
| `on_error` | boolean | `true` | Notify on errors |
| `on_budget` | boolean | `true` | Notify on budget thresholds |
| `on_milestone` | boolean | `true` | Notify when milestone finishes |
| `on_attention` | boolean | `true` | Notify when manual attention needed |

### `remote_questions`

Route interactive questions to Slack/Discord/Telegram for headless auto mode.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `channel` | string | -- | `"slack"`, `"discord"`, or `"telegram"` |
| `channel_id` | string/number | -- | Channel identifier |
| `timeout_minutes` | number | `15` | Question timeout (clamped 1-30) |
| `poll_interval_seconds` | number | `10` | Poll interval (clamped 2-30) |

### `always_use_skills` / `prefer_skills` / `avoid_skills`

Skill routing preferences. Skills can be bare names (looked up in `~/.agents/skills/` and `.agents/skills/`) or absolute paths.

```yaml
always_use_skills:
  - debug-like-expert
prefer_skills:
  - frontend-design
avoid_skills: []
```

### `skill_rules`

Situational skill routing with human-readable triggers:

```yaml
skill_rules:
  - when: task involves authentication
    use: [clerk]
  - when: frontend styling work
    prefer: [frontend-design]
```

Each rule supports `use`, `prefer`, and `avoid` arrays.

### `custom_instructions`

Durable instructions appended to every agent session's system prompt:

```yaml
custom_instructions:
  - "Always use TypeScript strict mode"
  - "Prefer functional patterns over classes"
```

### `post_unit_hooks`

Custom hooks that fire after specific unit types complete.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | -- | Unique hook identifier |
| `after` | string[] | -- | Unit types that trigger this hook |
| `prompt` | string | -- | Prompt sent to the LLM session |
| `model` | string | (active) | Optional model override (uses model-router resolution since v2.41.0) |
| `max_cycles` | number | `1` | Max fires per trigger (1-10) |
| `artifact` | string | -- | Skip if this file exists (idempotency) |
| `retry_on` | string | -- | If produced, re-run trigger unit then re-run hooks |
| `agent` | string | -- | Agent definition file |
| `enabled` | boolean | `true` | Disable without removing |

Prompt substitutions: `{milestoneId}`, `{sliceId}`, `{taskId}`.

Known unit types for `after`: `research-milestone`, `plan-milestone`, `research-slice`, `plan-slice`, `execute-task`, `complete-slice`, `replan-slice`, `reassess-roadmap`, `run-uat`, `complete-milestone`.

**Hook model resolution (v2.41.0):** The `model` field on hooks now uses the same model-router resolution as regular dispatches, instead of being limited to the Claude-only registry. This means hook models can reference any configured provider (e.g., `openrouter/deepseek/deepseek-r1`).

**Workflow-logger integration (v2.46.0):** Hook execution events are recorded in the workflow logger alongside engine, tool, manifest, and reconcile paths. This provides a unified audit trail for all state transitions, including hook-triggered ones.

### `pre_dispatch_hooks`

Hooks that intercept units before dispatch.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `name` | string | -- | Unique hook identifier |
| `before` | string[] | -- | Unit types to intercept |
| `action` | string | -- | `"modify"`, `"skip"`, or `"replace"` |
| `prepend` | string | -- | Text prepended to prompt (modify action) |
| `append` | string | -- | Text appended to prompt (modify action) |
| `prompt` | string | -- | Replacement prompt (replace action) |
| `unit_type` | string | -- | Override unit type label (replace action) |
| `skip_if` | string | -- | Only skip if this file exists (skip action) |
| `model` | string | (active) | Optional model override (uses model-router resolution since v2.41.0) |
| `enabled` | boolean | `true` | Disable without removing |

### `parallel`

Run multiple milestones simultaneously.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Master toggle |
| `max_workers` | number | `2` | Concurrent workers (clamped 1-4) |
| `budget_ceiling` | number | -- | Aggregate cost limit in USD |
| `merge_strategy` | string | `"per-milestone"` | `"per-slice"` or `"per-milestone"` |
| `auto_merge` | string | `"confirm"` | `"auto"`, `"confirm"`, or `"manual"` |
| `worker_model` | string | (inherited) | Model override for parallel milestone workers (v2.66.0). When set, workers use this model (e.g., `"claude-haiku-4-5"`) instead of inheriting the coordinator's model. Useful for cost savings on execution-heavy milestones. |

### `slice_parallel` (v2.64.0)

Slice-level parallelism within a milestone. When enabled, independent slices within the same milestone can execute concurrently in separate worktrees.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Enable slice-level parallelism |
| `max_workers` | number | `2` | Maximum concurrent slice workers |

```yaml
slice_parallel:
  enabled: true
  max_workers: 3
```

Unlike milestone-level `parallel`, which runs entire milestones concurrently, `slice_parallel` operates within a single milestone and respects slice dependency ordering. Slices with unmet dependencies wait until their prerequisites complete. Each worker runs in its own worktree with `GSD_PARALLEL_WORKER=1` set in the environment.

### `safety_harness` (v2.64.0)

LLM safety harness configuration for auto-mode damage control. Monitors, validates, and constrains LLM behavior during auto-mode execution. Enabled by default with a warn-and-continue policy.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `true` | Master toggle |
| `evidence_collection` | boolean | `true` | Collect evidence of LLM actions |
| `file_change_validation` | boolean | `true` | Validate file changes against plan |
| `destructive_command_warnings` | boolean | `true` | Warn on destructive shell commands |
| `content_validation` | boolean | `true` | Validate generated content |
| `checkpoints` | boolean | `true` | Enable git checkpoints for rollback |
| `auto_rollback` | boolean | `false` | Automatically rollback to last checkpoint on error |
| `timeout_scale_cap` | number | -- | Cap timeout scaling factor |

```yaml
safety_harness:
  enabled: true
  checkpoints: true
  auto_rollback: true
```

### `enhanced_verification` (v2.65.0)

Pre-execution and post-execution verification checks that validate task plans before execution and audit results afterward.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enhanced_verification` | boolean | `true` | Master toggle (opt-out). Set `false` to disable all enhanced verification. |
| `enhanced_verification_pre` | boolean | `true` | Enable pre-execution checks (package existence, file references). Only applies when `enhanced_verification` is true. |
| `enhanced_verification_post` | boolean | `true` | Enable post-execution checks (runtime error detection, audit warnings). Only applies when `enhanced_verification` is true. |
| `enhanced_verification_strict` | boolean | `false` | Strict mode: treat any pre-execution check failure as blocking (default: warnings only for non-critical failures). |

```yaml
enhanced_verification: true
enhanced_verification_pre: true
enhanced_verification_post: true
enhanced_verification_strict: false
```

### `github` (v2.39.0)

GitHub sync configuration. When enabled, GSD auto-syncs milestones, slices, and tasks to GitHub Issues, PRs, and Milestones.

```yaml
github:
  enabled: true
  repo: "owner/repo"
  labels: [gsd, auto-generated]
  project: "Project ID"
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Enable GitHub sync |
| `repo` | string | (auto-detected) | GitHub repository in `owner/repo` format |
| `labels` | string[] | `[]` | Labels to apply to created issues and PRs |
| `project` | string | (none) | GitHub Project ID for project board integration |

Requires `gh` CLI installed and authenticated. Sync mapping is persisted in `.gsd/.github-sync.json`.

### `reactive_execution.subagent_model` (v2.66.0)

Model override for subagent workers spawned during reactive execution. When set, subagents use this model instead of inheriting the parent session's model.

```yaml
reactive_execution:
  subagent_model: claude-haiku-4-5
```

### `experimental`

Opt-in experimental features. All features in this block are disabled by default and may change or be removed without a deprecation cycle.

#### `experimental.rtk` (v2.51.0)

Enable managed RTK (Real-Time Kompression) shell-command compression. When enabled, GSD wraps shell commands through the RTK binary to reduce token usage during command execution. The RTK binary is auto-managed in `~/.gsd/bin/`.

```yaml
experimental:
  rtk: true    # default: false
```

The web UI also exposes an RTK toggle when this preference is enabled, showing RTK session savings in the auto-mode dashboard.

### `stale_commit_threshold_minutes`

Minutes without a commit before flagging uncommitted changes as stale. When the threshold is exceeded and the working tree is dirty, doctor will auto-commit a safety snapshot. Default: `30`. Set `0` to disable.

### `compression_strategy`

Compression strategy for context that exceeds token budget.

| Value | Behavior |
|-------|----------|
| `"truncate"` (default) | Drop sections that exceed budget |
| `"compress"` | Apply heuristic compression before truncating |

### `context_selection`

Context selection mode for file inlining.

| Value | Behavior |
|-------|----------|
| `"full"` | Inline entire files |
| `"smart"` | Use semantic chunking |

Default derived from token profile.

---

## TUI Settings (Non-Preference)

These settings are managed through the interactive settings selector (`/settings`) and stored in `~/.gsd/settings.json`, separate from YAML preferences. They control editor behavior rather than auto-mode orchestration.

### `searchExcludeDirs` (v2.29.0)

Directories to exclude from the `@` file autocomplete search. Useful for filtering large directories like `node_modules`, `.git`, `dist`, or vendor directories from the file picker.

```json
{
  "searchExcludeDirs": ["node_modules", ".git", "dist", "vendor"]
}
```

### `respectGitignoreInPicker` (v2.29.0)

When true (default), the `@` file picker respects `.gitignore` rules and hides gitignored files. Set to false to show all files regardless of gitignore status.

```json
{
  "respectGitignoreInPicker": true
}
```

### `retentionDays`

Controls how long activity log files are retained. On Windows, `retentionDays=0` is handled specially: it deletes all log files except the current highest-sequence one, rather than treating 0 as "delete everything" (v2.45.0 fix).

---

## Agent Instructions File

Separate from preferences, GSD loads "always follow" instructions from:

1. `~/.gsd/agent-instructions.md` (global)
2. `.gsd/agent-instructions.md` (project)

Both are loaded and concatenated (global first). These are injected into every agent session's system prompt under "## Agent Instructions". Unlike `custom_instructions` in preferences (which are skill-routing directives), agent instructions are free-form markdown that the agent must follow.

---

## Custom Model Definitions

Define custom models in `~/.gsd/agent/models.json`. Resolution order:
1. `~/.gsd/agent/models.json` (primary)
2. `~/.pi/agent/models.json` (fallback)
3. If neither exists, creates `~/.gsd/agent/models.json`

For providers not tracked by the auto-generated registry, model definitions are maintained in `packages/pi-ai/src/models.custom.ts`. This file is the provider registry for custom endpoints (Alibaba Coding Plan, Z.AI) that use proprietary API endpoints not listed on models.dev. The v2.52.0 release replaced model-ID pattern matching with capability metadata for provider resolution, using this registry.

---

## Tool API Keys

Stored globally in `~/.gsd/agent/auth.json`. Managed via `/gsd config`.

| Tool | Environment Variable | Purpose |
|------|---------------------|---------|
| Tavily Search | `TAVILY_API_KEY` | Web search for non-Anthropic models |
| Brave Search | `BRAVE_API_KEY` | Web search for non-Anthropic models |
| Context7 Docs | `CONTEXT7_API_KEY` | Library documentation lookup |

Keys are loaded at session start via `loadToolApiKeys()`. Environment variables take precedence over saved keys.

---

## Preference Validation

`validatePreferences()` checks for:
- Unknown top-level keys (potential typos)
- Type mismatches on known fields
- Deprecated field names
- Invalid enum values

Validation warnings are surfaced in the UI at session start and included when rendering preferences for the system prompt.

### Known Preference Keys

The canonical set (from `KNOWN_PREFERENCE_KEYS`):

```
version, mode, always_use_skills, prefer_skills, avoid_skills, skill_rules,
custom_instructions, models, skill_discovery, skill_staleness_days,
auto_supervisor, uat_dispatch, unique_milestone_ids, budget_ceiling,
budget_enforcement, context_pause_threshold, notifications, cmux,
remote_questions, git, post_unit_hooks, pre_dispatch_hooks, dynamic_routing,
token_profile, phases, auto_visualize, auto_report, parallel,
verification_commands, verification_auto_fix, verification_max_retries,
search_provider, context_selection, widget_mode, reactive_execution,
gate_evaluation, github, service_tier, forensics_dedup, show_token_cost,
stale_commit_threshold_minutes, context_management, experimental, codebase,
slice_parallel, safety_harness, enhanced_verification,
enhanced_verification_pre, enhanced_verification_post,
enhanced_verification_strict, discuss_preparation, discuss_web_research,
discuss_depth
```

---

## Interactive Management

| Command | Description |
|---------|-------------|
| `/gsd prefs` | Open global preferences wizard |
| `/gsd prefs global` | Interactive wizard for `~/.gsd/PREFERENCES.md` |
| `/gsd prefs project` | Interactive wizard for `.gsd/PREFERENCES.md` |
| `/gsd prefs status` | Show current files, merged values, skill resolution |
| `/gsd prefs import-claude` | Import Claude marketplace plugins as GSD components |
| `/gsd mode` | Switch workflow mode (solo/team) |
| `/gsd config` | Tool API key configuration wizard |
| `/gsd fast` | Toggle OpenAI service tier (priority/flex) |
| `/gsd show-config` | Show effective configuration -- models, routing, toggles, and active overrides (v2.66.0) |
| `/btw` | Ephemeral side question -- ask a quick question from conversation context without interrupting the current workflow (v2.60.0) |

---

## Full Example

```yaml
---
version: 1

# Workflow mode
mode: solo

# Model selection with fallbacks
models:
  research: openrouter/deepseek/deepseek-r1
  planning:
    model: claude-opus-4-6
    fallbacks:
      - openrouter/z-ai/glm-5
  execution: claude-sonnet-4-6
  execution_simple: claude-haiku-4-5
  completion: claude-sonnet-4-6

# Token optimization
token_profile: balanced

# Dynamic model routing
dynamic_routing:
  enabled: true
  capability_routing: true
  escalate_on_failure: true
  budget_pressure: true

# Budget
budget_ceiling: 25.00
budget_enforcement: pause
context_pause_threshold: 80

# Supervision
auto_supervisor:
  soft_timeout_minutes: 15
  idle_timeout_minutes: 10
  hard_timeout_minutes: 25

# Git
git:
  auto_push: true
  merge_strategy: squash
  isolation: none
  commit_docs: true
  auto_pr: true
  pr_target_branch: develop

# Verification
verification_commands:
  - npm run lint
  - npm run test
verification_auto_fix: true
verification_max_retries: 2

# Skills
skill_discovery: suggest
always_use_skills:
  - debug-like-expert
skill_rules:
  - when: task involves authentication
    use: [clerk]

# Notifications
notifications:
  on_complete: false
  on_milestone: true

# Visualizer
auto_visualize: true

# Service tier
service_tier: priority

# Diagnostics
forensics_dedup: true
show_token_cost: true

# Parallel (milestone-level)
parallel:
  enabled: false
  worker_model: claude-haiku-4-5

# Slice-level parallelism
slice_parallel:
  enabled: false
  max_workers: 2

# Safety harness
safety_harness:
  enabled: true
  checkpoints: true
  auto_rollback: false

# Enhanced verification
enhanced_verification: true
enhanced_verification_strict: false

# Experimental
experimental:
  rtk: true

# Hooks
post_unit_hooks:
  - name: code-review
    after: [execute-task]
    prompt: "Review {sliceId}/{taskId} for quality and security."
    artifact: REVIEW.md
---
```
