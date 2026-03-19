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
| Global | `~/.gsd/preferences.md` | All projects |
| Global (legacy) | `~/.pi/agent/gsd-preferences.md` | All projects (fallback) |
| Project | `.gsd/preferences.md` | Current project only |

GSD also checks uppercase variants (`PREFERENCES.md`) as a fallback for files created by an earlier bootstrap bug.

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

| Mode | `git.auto_push` | `git.push_branches` | `git.pre_merge_check` | `unique_milestone_ids` |
|------|:---------------:|:-------------------:|:--------------------:|:---------------------:|
| `solo` | true | false | false | false |
| `team` | false | true | true | true |

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
| `execution` | `execute-task` |
| `execution_simple` | `execute-task` when classified as simple by complexity router |
| `completion` | `complete-slice`, `complete-milestone`, `reassess-roadmap`, `run-uat`, `validate-milestone` |
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

### `git`

Git behavior configuration. All fields optional.

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `isolation` | string | `"worktree"` | Auto-mode isolation strategy |
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
| `auto_pr` | boolean | `false` | Create PR on milestone completion (requires `gh` CLI) |
| `pr_target_branch` | string | (main branch) | Target branch for auto-created PRs |

#### Git Isolation Modes

| Mode | Description |
|------|-------------|
| `worktree` (default) | Each milestone runs in `.gsd/worktrees/<MID>/` on a `milestone/<MID>` branch. Squash-merged to main on completion. |
| `branch` | Work in project root on a `milestone/<MID>` branch. Useful for submodule-heavy repos. |
| `none` | Work directly on current branch. No worktree or milestone branch. For hot-reload workflows. |

Isolation mode is resolved by `getIsolationMode()` in `preferences.ts`.

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
| `enabled` | boolean | `false` | Master toggle |
| `tier_models.light` | string | (auto-detected) | Model for simple tasks |
| `tier_models.standard` | string | (auto-detected) | Model for standard tasks |
| `tier_models.heavy` | string | (auto-detected) | Model for complex tasks |
| `escalate_on_failure` | boolean | `true` | Escalate tier on failure |
| `budget_pressure` | boolean | `true` | Bias toward cheaper models under budget pressure |
| `cross_provider` | boolean | `true` | Consider models across providers |
| `hooks` | boolean | `true` | Apply routing to hook units |

**Downgrade-only semantics:** the router never upgrades beyond the user's configured model. It can only select equal or cheaper alternatives.

Known model tier assignments (from `model-router.ts`):

| Tier | Models |
|------|--------|
| Light | claude-haiku-4-5, gpt-4o-mini, gemini-2.0-flash |
| Standard | claude-sonnet-4-6, gpt-4o, gemini-2.5-pro, deepseek-chat |
| Heavy | claude-opus-4-6, o1, o3 |

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

Skill routing preferences. Skills can be bare names (looked up in `~/.gsd/agent/skills/`) or absolute paths.

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
| `model` | string | (active) | Optional model override |
| `max_cycles` | number | `1` | Max fires per trigger (1-10) |
| `artifact` | string | -- | Skip if this file exists (idempotency) |
| `retry_on` | string | -- | If produced, re-run trigger unit then re-run hooks |
| `agent` | string | -- | Agent definition file |
| `enabled` | boolean | `true` | Disable without removing |

Prompt substitutions: `{milestoneId}`, `{sliceId}`, `{taskId}`.

Known unit types for `after`: `research-milestone`, `plan-milestone`, `research-slice`, `plan-slice`, `execute-task`, `complete-slice`, `replan-slice`, `reassess-roadmap`, `run-uat`, `complete-milestone`.

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
| `model` | string | (active) | Optional model override |
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

### `search_provider`

Search provider preference for web search during research phases.

| Value | Behavior |
|-------|----------|
| `"auto"` | Default behavior (native for Anthropic, Brave/Tavily for others) |
| `"brave"` | Force Brave Search backend |
| `"tavily"` | Force Tavily Search backend |
| `"ollama"` | Use Ollama for search |
| `"native"` | Force native Anthropic search only |

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
budget_enforcement, context_pause_threshold, notifications, remote_questions,
git, post_unit_hooks, pre_dispatch_hooks, dynamic_routing, token_profile,
phases, auto_visualize, auto_report, parallel, verification_commands,
verification_auto_fix, verification_max_retries, search_provider,
compression_strategy, context_selection
```

---

## Interactive Management

| Command | Description |
|---------|-------------|
| `/gsd prefs` | Open global preferences wizard |
| `/gsd prefs global` | Interactive wizard for `~/.gsd/preferences.md` |
| `/gsd prefs project` | Interactive wizard for `.gsd/preferences.md` |
| `/gsd prefs status` | Show current files, merged values, skill resolution |
| `/gsd prefs import-claude` | Import Claude marketplace plugins as GSD components |
| `/gsd mode` | Switch workflow mode (solo/team) |
| `/gsd config` | Tool API key configuration wizard |

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
  isolation: worktree
  commit_docs: true

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

# Hooks
post_unit_hooks:
  - name: code-review
    after: [execute-task]
    prompt: "Review {sliceId}/{taskId} for quality and security."
    artifact: REVIEW.md
---
```
