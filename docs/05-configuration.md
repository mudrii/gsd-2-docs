# Configuration Reference

GSD preferences are stored as YAML frontmatter inside markdown files. Settings cascade from global to project scope, with project values overriding global ones.

---

## File Locations

| Scope | Path | Applies to |
|-------|------|-----------|
| Global | `~/.gsd/PREFERENCES.md` | All projects |
| Project | `.gsd/PREFERENCES.md` | Current project only |
| Tool API keys | `~/.gsd/agent/auth.json` | All projects |
| Custom model definitions | `~/.gsd/agent/models.json` | All projects |
| Routing history | `.gsd/routing-history.json` | Current project |
| Metrics ledger | `.gsd/metrics.json` | Current project |

### `.gsd/` Directory Structure

```
.gsd/
  PREFERENCES.md          — project preferences (YAML frontmatter, renamed from preferences.md in v2.52)
  metrics.json            — token/cost ledger for all units
  routing-history.json    — adaptive model routing history
  completed-units.json    — skipped/completed unit keys
  KNOWLEDGE.md            — persistent project knowledge
  CAPTURES.md             — pending thought captures
  OVERRIDES.md            — hard-steer overrides
  reports/                — exported HTML milestone reports
    index.html            — browseable report index
  runtime/
    headless-context.md   — context file for headless new-milestone
  hooks/
    post-worktree-create  — optional post-worktree hook script
```

### Manage with `/gsd prefs`

The interactive wizard is the recommended way to edit preferences:

```
/gsd prefs             — global preferences wizard
/gsd prefs project     — project preferences wizard
/gsd prefs status      — show merged effective preferences
```

> **Migration note (v2.52):** `preferences.md` was renamed to `PREFERENCES.md` for consistency with other uppercase GSD files. GSD reads both filenames but prefers the uppercase variant. Worktrees also prefer `PREFERENCES.md` (v2.56).

---

## Preferences File Format

Preferences are stored in YAML frontmatter within a markdown file. The body of the file (below the `---` closing fence) is ignored.

```yaml
---
version: 1
models:
  research: claude-sonnet-4-6
  planning: claude-opus-4-6
  execution: claude-sonnet-4-6
  completion: claude-sonnet-4-6
skill_discovery: suggest
auto_supervisor:
  soft_timeout_minutes: 20
  idle_timeout_minutes: 10
  hard_timeout_minutes: 30
budget_ceiling: 50.00
token_profile: balanced
---
```

---

## Merge Behavior

When both global and project preference files exist, GSD merges them at load time:

| Field type | Merge behavior |
|-----------|---------------|
| Scalar fields (`skill_discovery`, `budget_ceiling`, etc.) | Project value wins if set |
| Array fields (`always_use_skills`, `custom_instructions`, etc.) | Concatenated — global first, then project |
| Object fields (`models`, `git`, `auto_supervisor`) | Shallow-merged — project overrides per-key |

---

## All Settings

### `version`

**Type:** `number`
**Default:** `1`

Schema version marker. Always set to `1` for current format.

---

### `mode`

**Type:** `"solo" | "team"`
**Default:** none (not set)

Workflow mode. Setting this via `/gsd mode` applies a coordinated set of defaults:

| Setting | `solo` | `team` |
|---------|--------|--------|
| `git.auto_push` | `true` | `false` |
| `git.push_branches` | `false` | `true` |
| `git.pre_merge_check` | `false` | `true` |
| `git.merge_strategy` | `"squash"` | `"squash"` |
| `git.isolation` | `"none"` | `"none"` |
| `git.commit_docs` | `true` | `true` |
| `unique_milestone_ids` | `false` | `true` |

---

### `models`

**Type:** object
**Default:** uses active session model for all phases

Per-phase model selection. Omit a key to use whatever model is currently active in the session.

```yaml
models:
  research: claude-sonnet-4-6
  planning: claude-opus-4-6
  execution: claude-sonnet-4-6
  execution_simple: claude-haiku-4-5-20250414
  completion: claude-sonnet-4-6
  subagent: claude-sonnet-4-6
```

| Phase key | Used for |
|-----------|---------|
| `research` | Milestone and slice research units |
| `planning` | Milestone and slice planning units |
| `execution` | Standard task execution units |
| `execution_simple` | Tasks classified as "simple" by the complexity router |
| `completion` | Slice and milestone completion units |
| `subagent` | Delegated subagent tasks (scout, researcher, worker) |

**Provider targeting:** use `provider/model` format (e.g., `bedrock/claude-sonnet-4-6`) or the `provider` field in object format.

#### Per-phase fallbacks

Each phase can specify a primary model and ordered fallbacks:

```yaml
models:
  planning:
    model: claude-opus-4-6
    fallbacks:
      - openrouter/z-ai/glm-5
      - openrouter/moonshotai/kimi-k2.5
    provider: bedrock    # optional: target a specific provider
```

When the primary model fails (provider unavailable, rate limited, credits exhausted), GSD automatically tries each fallback in order.

#### Custom Model Definitions (`models.json`)

Define custom models in `~/.gsd/agent/models.json` to add models not in the default registry — self-hosted endpoints, fine-tuned models, or new releases. GSD resolves this file with fallback logic:

1. `~/.gsd/agent/models.json` — primary
2. `~/.pi/agent/models.json` — fallback
3. If neither exists, creates `~/.gsd/agent/models.json`

#### Community Provider Extensions

For providers not built into GSD, community extensions add full provider support:

| Extension | Provider | Models | Install |
|-----------|----------|--------|---------|
| `pi-dashscope` | Alibaba DashScope | Qwen3, GLM-5, MiniMax M2.5, Kimi K2.5 | `gsd install npm:pi-dashscope` |

---

### `token_profile`

**Type:** `"budget" | "balanced" | "quality"`
**Default:** `"balanced"`

Coordinates model selection, phase skipping, and context compression level. A single setting that configures multiple dimensions at once.

| Profile | Typical Savings | Phase skipping | Context level |
|---------|----------------|----------------|---------------|
| `budget` | 40-60% | Skips research, slice research, reassessment | Minimal |
| `balanced` | 10-20% | Skips slice research only | Standard |
| `quality` | 0% (baseline) | All phases run | Full |

**`budget` profile details:**

| Dimension | Setting |
|-----------|---------|
| Planning model | Sonnet |
| Execution model | Sonnet |
| Simple task model | Haiku |
| Completion model | Haiku |
| Subagent model | Haiku |
| Milestone research | Skipped |
| Slice research | Skipped |
| Roadmap reassessment | Skipped |
| Context inline level | Minimal |

Best for: prototyping, small projects, well-understood codebases, cost-conscious iteration.

**`balanced` profile details:**

| Dimension | Setting |
|-----------|---------|
| Planning model | User's default |
| Execution model | User's default |
| Simple task model | User's default |
| Completion model | User's default |
| Subagent model | Sonnet |
| Milestone research | Runs |
| Slice research | Skipped |
| Roadmap reassessment | Runs |
| Context inline level | Standard |

Best for: most projects, day-to-day development.

**`quality` profile details:** Every phase runs. Every context artifact is inlined. No shortcuts. Best for complex architectures, greenfield projects requiring deep research, critical production work.

---

### `phases`

**Type:** object
**Default:** derived from `token_profile`

Fine-grained control over which phases run in auto mode. Explicit `phases` settings override profile defaults.

```yaml
phases:
  skip_research: false              # skip milestone-level research
  skip_reassess: false              # skip roadmap reassessment after each slice
  skip_slice_research: true         # skip per-slice research
  require_slice_discussion: false   # pause auto-mode before each slice for discussion
```

| Field | Type | Description |
|-------|------|-------------|
| `skip_research` | boolean | Skip milestone-level research phase |
| `skip_reassess` | boolean | Skip roadmap reassessment after each slice |
| `skip_slice_research` | boolean | Skip per-slice research phase |
| `require_slice_discussion` | boolean | Pause auto mode before each slice, requiring user discussion before continuing |

---

### `compression_strategy`

**Type:** `"truncate" | "compress"`
**Default:** `"compress"` for `budget` and `balanced` profiles; `"truncate"` for `quality`

Controls how context is handled when it exceeds the budget:

| Strategy | Behavior |
|----------|----------|
| `truncate` | Drop entire sections at section boundaries (pre-v2.29 behavior) |
| `compress` | Apply heuristic text compression first (removes redundant whitespace, abbreviates verbose phrases, deduplicates content, removes low-information boilerplate), then truncate if still over budget |

```yaml
compression_strategy: compress
```

---

### `context_selection`

**Type:** `"full" | "smart"`
**Default:** `"smart"` for `budget` profile; `"full"` for `balanced` and `quality`

Controls how files are inlined into dispatch prompts:

| Mode | Behavior |
|------|----------|
| `full` | Inline entire files |
| `smart` | Use TF-IDF semantic chunking for large files (>3KB), including only relevant portions |

```yaml
context_selection: smart
```

---

### `show_token_cost` (v2.44)

**Type:** `boolean`
**Default:** `false`

Show per-prompt token cost in the TUI footer. When enabled, each prompt displays the input/output token count and estimated USD cost.

```yaml
show_token_cost: true
```

---

### `searchExcludeDirs` (v2.34+)

**Type:** `string[]`
**Default:** `[]`

Directories to exclude from file search and picker results. Patterns are matched against relative paths.

```yaml
searchExcludeDirs:
  - node_modules
  - .git
  - dist
```

---

### `respectGitignoreInPicker`

**Type:** `boolean`
**Default:** `true`

When `true`, the file picker respects `.gitignore` rules and excludes ignored files from results.

```yaml
respectGitignoreInPicker: true
```

---

### `skill_discovery`

**Type:** `"auto" | "suggest" | "off"`
**Default:** `"suggest"`

Controls how GSD finds and applies skills during auto mode.

| Value | Behavior |
|-------|----------|
| `auto` | Skills found and applied automatically during research |
| `suggest` | Skills are identified during research but require confirmation before being applied |
| `off` | Skill discovery is disabled entirely |

---

### `skill_staleness_days`

**Type:** `number`
**Default:** `60`

Number of days after which an unused skill is considered stale and deprioritized in automatic matching. Stale skills remain invokable explicitly. Set to `0` to disable staleness detection.

```yaml
skill_staleness_days: 60
```

---

### `always_use_skills`

**Type:** `string[]`
**Default:** `[]`

Skills that are always loaded, regardless of task context. Accepts bare names or absolute paths.

```yaml
always_use_skills:
  - debug-like-expert
  - /path/to/custom-skill
```

---

### `prefer_skills`

**Type:** `string[]`
**Default:** `[]`

Skills that are preferred when applicable — loaded when the task matches but not mandatory.

```yaml
prefer_skills:
  - frontend-design
```

---

### `avoid_skills`

**Type:** `string[]`
**Default:** `[]`

Skills that are never automatically applied. Still usable via explicit invocation.

```yaml
avoid_skills:
  - aggressive-refactor
```

---

### `skill_rules`

**Type:** array of rule objects
**Default:** `[]`

Situational skill routing with human-readable trigger conditions. Each rule specifies when it applies and what action to take.

```yaml
skill_rules:
  - when: task involves authentication
    use: [clerk]
  - when: frontend styling work
    prefer: [frontend-design]
  - when: working with legacy code
    avoid: [aggressive-refactor]
```

| Field | Type | Description |
|-------|------|-------------|
| `when` | string | Human-readable condition matched against task context |
| `use` | string[] | Skills to always apply when condition matches |
| `prefer` | string[] | Skills to prefer when condition matches |
| `avoid` | string[] | Skills to avoid when condition matches |

---

### `custom_instructions`

**Type:** `string[]`
**Default:** `[]`

Durable instructions appended to every session prompt. For project-specific knowledge (patterns, gotchas, lessons), prefer `.gsd/KNOWLEDGE.md` — use `/gsd knowledge` to add entries there.

```yaml
custom_instructions:
  - "Always use TypeScript strict mode"
  - "Prefer functional patterns over classes"
```

---

### `auto_supervisor`

**Type:** object
**Default:** `{ soft_timeout_minutes: 20, idle_timeout_minutes: 10, hard_timeout_minutes: 30 }`

Timeout thresholds for auto mode supervision.

```yaml
auto_supervisor:
  model: claude-sonnet-4-6    # optional: model for supervisor
  soft_timeout_minutes: 20    # warn LLM to wrap up
  idle_timeout_minutes: 10    # detect stalls
  hard_timeout_minutes: 30    # pause auto mode
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `model` | string | active model | Optional model override for the supervisor |
| `soft_timeout_minutes` | number | `20` | Sends a wrap-up warning to the LLM |
| `idle_timeout_minutes` | number | `10` | Detects and handles stalled execution |
| `hard_timeout_minutes` | number | `30` | Forces a pause of auto mode |

---

### `uat_dispatch`

**Type:** `boolean`
**Default:** `false`

When `true`, automatically runs a UAT (User Acceptance Test) unit after each slice completes.

```yaml
uat_dispatch: true
```

---

### `unique_milestone_ids`

**Type:** `boolean`
**Default:** `false`

Appends a random suffix to milestone IDs to avoid collisions in team workflows where multiple developers may create milestones concurrently.

```yaml
unique_milestone_ids: true
# Produces: M001-eh88as instead of M001
```

---

### `budget_ceiling`

**Type:** `number` (USD)
**Default:** none (no ceiling)

Maximum USD to spend during auto mode. No dollar sign — just the number. When reached, enforces the `budget_enforcement` policy.

```yaml
budget_ceiling: 50.00
```

---

### `budget_enforcement`

**Type:** `"warn" | "pause" | "halt"`
**Default:** `"pause"` when `budget_ceiling` is set

How the budget ceiling is enforced when spending reaches or exceeds the limit.

| Value | Behavior |
|-------|----------|
| `warn` | Log a warning and continue executing |
| `pause` | Pause auto mode, wait for user action |
| `halt` | Stop auto mode entirely |

```yaml
budget_enforcement: pause
```

---

### `context_pause_threshold`

**Type:** `number` (0-100, percentage)
**Default:** `0` (disabled)

Context window usage percentage at which auto mode pauses for checkpointing. Set to `0` to disable.

```yaml
context_pause_threshold: 80    # pause at 80% context usage
```

---

### `verification_commands`

**Type:** `string[]`
**Default:** `[]`

Shell commands that run automatically after every task execution. Failures trigger auto-fix retries before advancing to the next unit.

```yaml
verification_commands:
  - npm run lint
  - npm run test
```

---

### `verification_auto_fix`

**Type:** `boolean`
**Default:** `true`

When `true`, automatically retries the task when verification commands fail.

---

### `verification_max_retries`

**Type:** `number`
**Default:** `2`

Maximum number of auto-fix retry attempts when verification commands fail.

---

### `auto_report`

**Type:** `boolean`
**Default:** `true`

Automatically generate an HTML report snapshot after each milestone completes. Reports are written to `.gsd/reports/` as self-contained HTML files with embedded CSS/JS.

```yaml
auto_report: true
```

---

### `auto_visualize`

**Type:** `boolean`
**Default:** `false`

Automatically show the workflow visualizer after milestone completion.

```yaml
auto_visualize: true
```

---

### `search_provider`

**Type:** `"brave" | "tavily" | "ollama" | "native" | "auto"`
**Default:** `"auto"`

Search provider preference for web search during research phases.

| Value | Behavior |
|-------|----------|
| `auto` | Default behavior — uses native Anthropic search when available |
| `native` | Force native Anthropic search only |
| `brave` | Force Brave Search (requires `BRAVE_API_KEY`) |
| `tavily` | Force Tavily Search (requires `TAVILY_API_KEY`) |
| `ollama` | Use Ollama-backed search |

Note: `brave` and `tavily` disable native Anthropic search.

---

### `dynamic_routing`

**Type:** object
**Default:** disabled

Complexity-based automatic model selection. Routes simple work to cheaper models and reserves expensive models for complex tasks. Can reduce token consumption by 20-50% on capped plans.

```yaml
dynamic_routing:
  enabled: true
  tier_models:
    light: claude-haiku-4-5
    standard: claude-sonnet-4-6
    heavy: claude-opus-4-6
  escalate_on_failure: true
  budget_pressure: true
  cross_provider: true
  hooks: true
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Master toggle |
| `tier_models` | object | built-in mapping | Explicit model per tier (optional) |
| `escalate_on_failure` | boolean | `true` | Bump tier on task failure (Light → Standard → Heavy) |
| `budget_pressure` | boolean | `true` | Auto-downgrade when approaching budget ceiling |
| `cross_provider` | boolean | `true` | Consider models from other configured providers |
| `hooks` | boolean | `true` | Apply routing to post-unit hooks |

**Complexity tiers:**

| Tier | Typical Work | Default Model Level |
|------|-------------|-------------------|
| Light | Slice completion, UAT, hooks | Haiku-class |
| Standard | Research, planning, execution, milestone completion | Sonnet-class |
| Heavy | Replanning, roadmap reassessment, complex execution | Opus-class |

**Unit type defaults:**

| Unit Type | Default Tier |
|-----------|-------------|
| `complete-slice`, `run-uat` | Light |
| `research-*`, `plan-*`, `execute-task`, `complete-milestone` | Standard |
| `replan-slice`, `reassess-roadmap` | Heavy |
| `hook/*` | Light |

**Task analysis signals for `execute-task`:**

| Signal | Simple (Light) | Complex (Heavy) |
|--------|---------------|----------------|
| Step count | ≤ 3 | ≥ 8 |
| File count | ≤ 3 | ≥ 8 |
| Description length | < 500 chars | > 2000 chars |
| Code blocks | — | ≥ 5 |
| Complexity keywords | None | Present |

Complexity keywords: `research`, `investigate`, `refactor`, `migrate`, `integrate`, `complex`, `architect`, `redesign`, `security`, `performance`, `concurrent`, `parallel`, `distributed`, `backward compat`

**Budget pressure downgrade schedule:**

| Budget Used | Effect |
|------------|--------|
| < 50% | No adjustment |
| 50-75% | Standard → Light |
| 75-90% | More aggressive downgrading |
| > 90% | Nearly everything → Light; only Heavy stays at Standard |

**Key rule:** downgrade-only semantics. The user's configured model is always the ceiling — dynamic routing never upgrades beyond what you've configured.

---

### `git`

**Type:** object
**Default:** see field defaults below

Git behavior configuration. All fields are optional.

```yaml
git:
  auto_push: false
  push_branches: false
  remote: origin
  snapshots: false
  pre_merge_check: false
  commit_type: feat
  main_branch: main
  merge_strategy: squash
  isolation: none
  commit_docs: true
  manage_gitignore: true
  worktree_post_create: .gsd/hooks/post-worktree-create
  auto_pr: false
  pr_target_branch: develop
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `auto_push` | boolean | `false` | Push commits to remote after committing |
| `push_branches` | boolean | `false` | Push milestone branch to remote |
| `remote` | string | `"origin"` | Git remote name |
| `snapshots` | boolean | `false` | WIP snapshot commits during long tasks |
| `pre_merge_check` | bool/string | `false` | Run checks before worktree merge (`true`, `false`, or `"auto"`) |
| `commit_type` | string | inferred | Override conventional commit prefix. Accepted values: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`, `ci`, `build`, `style` |
| `main_branch` | string | `"main"` | Primary branch name |
| `merge_strategy` | string | `"squash"` | How worktree branches merge into main: `"squash"` (combine all commits) or `"merge"` (preserve individual commits) |
| `isolation` | string | `"none"` | Auto-mode git isolation: `"worktree"` (separate directory), `"branch"` (work in project root — useful for submodule-heavy repos), or `"none"` (commits on current branch, no worktree or milestone branch). Default changed from `"worktree"` to `"none"` in v2.46.0 |
| `commit_docs` | boolean | `true` | Commit `.gsd/` planning artifacts to git. Set `false` to keep them local-only |
| `manage_gitignore` | boolean | `true` | When `false`, GSD will not modify `.gitignore` — no baseline patterns, no self-healing |
| `worktree_post_create` | string | none | Script to run after worktree creation. Receives `SOURCE_DIR` and `WORKTREE_DIR` env vars |
| `auto_pr` | boolean | `false` | Automatically create a pull request when a milestone completes. Requires `auto_push: true` and `gh` CLI installed and authenticated |
| `pr_target_branch` | string | main branch | Target branch for auto-created PRs. Defaults to `main_branch` if not set |

#### `git.worktree_post_create`

Script to run after a worktree is created (both auto-mode and manual `/worktree`). Useful for copying `.env` files, symlinking asset directories, or running setup commands that worktrees don't inherit from the main tree.

The script receives two environment variables:
- `SOURCE_DIR` — the original project root
- `WORKTREE_DIR` — the newly created worktree path

The path can be absolute or relative to the project root. The script runs with a 30-second timeout. Failure is non-fatal — GSD logs a warning and continues.

Example hook script (`.gsd/hooks/post-worktree-create`):

```bash
#!/bin/bash
cp "$SOURCE_DIR/.env" "$WORKTREE_DIR/.env"
cp "$SOURCE_DIR/.env.local" "$WORKTREE_DIR/.env.local" 2>/dev/null || true
ln -sf "$SOURCE_DIR/assets" "$WORKTREE_DIR/assets"
```

#### `git.auto_pr`

Automatically create a pull request when a milestone completes. Requires:
- `auto_push: true` — the milestone branch must be pushed before a PR can be created
- `gh` CLI installed and authenticated (`gh auth login`)

How it works:
1. Milestone completes — GSD squash-merges the worktree to the main branch
2. Pushes the main branch to remote (if `auto_push: true`)
3. Pushes the milestone branch to remote
4. Creates a PR from the milestone branch to `pr_target_branch` via `gh pr create`

PR creation failure is non-fatal — GSD logs and continues.

---

### `notifications`

**Type:** object
**Default:** all enabled

Controls what notifications GSD sends during auto mode.

```yaml
notifications:
  enabled: true
  on_complete: true      # notify on unit completion
  on_error: true         # notify on errors
  on_budget: true        # notify on budget thresholds
  on_milestone: true     # notify when milestone finishes
  on_attention: true     # notify when manual attention needed
```

---

### `remote_questions`

**Type:** object
**Default:** none (disabled)

Routes interactive questions to Slack, Discord, or Telegram during headless auto mode. When GSD encounters a decision point, it posts the question to the configured channel and polls for a response.

```yaml
remote_questions:
  channel: slack              # or discord or telegram
  channel_id: "C1234567890"
  timeout_minutes: 5          # 1-30, default 5
  poll_interval_seconds: 5    # 2-30, default 5
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `channel` | `"slack" \| "discord" \| "telegram"` | Yes | Which platform to use |
| `channel_id` | string or number | Yes | Slack: `C0123456789` (9-12 uppercase chars). Discord: `1234567890123456789` (17-20 digit snowflake) |
| `timeout_minutes` | number | No | How long to wait for a response before timing out (1-30, default: 5) |
| `poll_interval_seconds` | number | No | How often to check for a response (2-30, default: 5) |

**Setting up integrations:**

```
/gsd remote slack      — interactive Slack setup wizard
/gsd remote discord    — interactive Discord setup wizard
/gsd remote telegram   — interactive Telegram setup wizard
```

**Required bot permissions:**
- **Slack:** `chat:write`, `reactions:read`, `reactions:write`, `channels:read`, `groups:read`, `channels:history`, `groups:history`
- **Discord:** Send Messages, Read Message History, Add Reactions, View Channel
- **Telegram:** Bot token from @BotFather, bot added to target chat

**Response formats:**
- Single question: react with a number emoji (1-5) or reply with `1`
- Multiple questions: reply with semicolons (`1;2;custom text`) or one answer per line
- Free text: reply with any text (captured as a user note)

When no response is received within `timeout_minutes`, GSD continues with a timeout result and the LLM makes a conservative default choice.

---

### `post_unit_hooks`

**Type:** array of hook objects
**Default:** `[]`

Custom hooks that fire after specific unit types complete. Used for automated code review, linting, documentation generation, etc.

```yaml
post_unit_hooks:
  - name: code-review
    after: [execute-task]
    prompt: "Review the code changes for quality and security issues."
    model: claude-opus-4-6          # optional: model override
    max_cycles: 1                   # max fires per trigger (1-10, default: 1)
    artifact: REVIEW.md             # optional: skip if this file exists
    retry_on: NEEDS-REWORK.md       # optional: re-run trigger unit if this file appears
    agent: review-agent             # optional: agent definition to use
    enabled: true                   # optional: disable without removing
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Hook identifier (used by `/gsd run-hook`) |
| `after` | string[] | Unit types that trigger this hook |
| `prompt` | string | Prompt sent to the LLM when hook fires. Supports substitutions: `{milestoneId}`, `{sliceId}`, `{taskId}` |
| `model` | string | Optional model override for hook execution |
| `max_cycles` | number | Maximum times the hook fires per trigger event (1-10, default: 1) |
| `artifact` | string | Relative path — skip hook if this file already exists |
| `retry_on` | string | Relative path — if this file appears after hook runs, re-dispatch the triggering unit |
| `agent` | string | Agent definition to use for hook execution |
| `enabled` | boolean | Toggle hook without removing it from config |

**Valid `after` unit types:**

`research-milestone`, `plan-milestone`, `research-slice`, `plan-slice`, `execute-task`, `complete-slice`, `replan-slice`, `reassess-roadmap`, `run-uat`

**Example — post-execution review:**

```yaml
post_unit_hooks:
  - name: code-review
    after: [execute-task]
    prompt: "Review {sliceId}/{taskId} for quality and security issues."
    artifact: REVIEW.md
```

---

### `pre_dispatch_hooks`

**Type:** array of hook objects
**Default:** `[]`

Hooks that intercept units before dispatch. Three actions are available:

#### `modify` — prepend or append text to the unit prompt

```yaml
pre_dispatch_hooks:
  - name: add-standards
    before: [execute-task]
    action: modify
    prepend: "Follow our coding standards document."
    append: "Run linting after changes."
    enabled: true
```

#### `skip` — skip the unit entirely

```yaml
pre_dispatch_hooks:
  - name: skip-research
    before: [research-slice]
    action: skip
    skip_if: RESEARCH.md    # optional: only skip if this file exists
    enabled: true
```

#### `replace` — replace the unit prompt entirely

```yaml
pre_dispatch_hooks:
  - name: custom-execute
    before: [execute-task]
    action: replace
    prompt: "Execute the task using TDD methodology."
    unit_type: execute-task-tdd    # optional: override unit type label
    model: claude-opus-4-6         # optional: model override
    enabled: true
```

All pre-dispatch hooks support `enabled: true/false` to toggle without removing.

---

### `parallel`

**Type:** object
**Default:** disabled

Run multiple milestones simultaneously in separate worktrees with coordinated merge control.

```yaml
parallel:
  enabled: false                     # master toggle
  max_workers: 2                     # concurrent workers (1-4)
  budget_ceiling: 50.00              # aggregate cost limit in USD
  merge_strategy: "per-milestone"    # "per-slice" or "per-milestone"
  auto_merge: "confirm"              # "auto", "confirm", or "manual"
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Master toggle for parallel orchestration |
| `max_workers` | number | `2` | Maximum concurrent workers (1-4) |
| `budget_ceiling` | number | none | Aggregate cost limit in USD across all workers |
| `merge_strategy` | string | `"per-milestone"` | When to merge: `"per-slice"` or `"per-milestone"` |
| `auto_merge` | string | `"confirm"` | Merge behavior: `"auto"` (no prompt), `"confirm"` (prompt before merging), `"manual"` (never auto-merge) |

---

## Environment Variables

GSD reads these environment variables at startup:

| Variable | Description |
|----------|-------------|
| `GSD_VERSION` | GSD version string (set by the binary) |
| `GSD_HOME` | Override the global `~/.gsd` directory location (v2.39) |
| `GSD_PROJECT_ID` | Override the project hash used for project identification (v2.39) |
| `GSD_WORKFLOW_PATH` | Override path to the workflow protocol file (default: `~/.pi/GSD-WORKFLOW.md`) |
| `TAVILY_API_KEY` | Tavily web search API key |
| `BRAVE_API_KEY` | Brave web search API key |
| `CONTEXT7_API_KEY` | Context7 documentation lookup API key |
| `JINA_API_KEY` | Jina page extraction API key |
| `GROQ_API_KEY` | Groq voice transcription API key |
| `DISCORD_BOT_TOKEN` | Discord bot token for remote questions |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token for remote questions |

Environment variable values take precedence over keys stored in `~/.gsd/agent/auth.json`. On each session start, `loadToolApiKeys()` reads `auth.json` and sets environment variables for any keys not already set in the environment.

---

## Complete Example

```yaml
---
version: 1

# Model selection
models:
  research: openrouter/deepseek/deepseek-r1
  planning:
    model: claude-opus-4-6
    fallbacks:
      - openrouter/z-ai/glm-5
  execution: claude-sonnet-4-6
  execution_simple: claude-haiku-4-5-20250414
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
  hard_timeout_minutes: 25

# Git
git:
  auto_push: true
  merge_strategy: squash
  isolation: worktree
  commit_docs: true

# Skills
skill_discovery: suggest
skill_staleness_days: 60
always_use_skills:
  - debug-like-expert
skill_rules:
  - when: task involves authentication
    use: [clerk]

# Notifications
notifications:
  on_complete: false
  on_milestone: true
  on_attention: true

# Reports
auto_visualize: true
auto_report: true

# Verification
verification_commands:
  - npm run lint
  - npm run test
verification_auto_fix: true
verification_max_retries: 2

# Hooks
post_unit_hooks:
  - name: code-review
    after: [execute-task]
    prompt: "Review {sliceId}/{taskId} for quality and security."
    artifact: REVIEW.md
---
```

---

## Configuration Recipes

### Cost-Optimized Setup

```yaml
---
version: 1
token_profile: budget
budget_ceiling: 25.00
budget_enforcement: pause
models:
  execution_simple: claude-haiku-4-5-20250414
dynamic_routing:
  enabled: true
  budget_pressure: true
---
```

### Team Workflow with PR Automation

```yaml
---
version: 1
mode: team
git:
  auto_push: true
  push_branches: true
  auto_pr: true
  pr_target_branch: develop
  pre_merge_check: true
unique_milestone_ids: true
---
```

### Headless CI Pipeline

```yaml
---
version: 1
token_profile: balanced
budget_ceiling: 50.00
budget_enforcement: halt
remote_questions:
  channel: slack
  channel_id: "C0123456789"
  timeout_minutes: 15
  poll_interval_seconds: 10
notifications:
  on_milestone: true
  on_error: true
  on_attention: true
verification_commands:
  - npm run test
  - npm run lint
---
```

### Quality-First with Full Context

```yaml
---
version: 1
token_profile: quality
models:
  planning: claude-opus-4-6
  execution: claude-opus-4-6
phases:
  skip_research: false
  skip_reassess: false
  skip_slice_research: false
---
```

### Per-Phase Model Overrides with Budget Profile

```yaml
---
version: 1
token_profile: budget
phases:
  skip_research: false    # keep milestone research despite budget profile
models:
  planning: claude-opus-4-6    # use Opus for planning despite budget profile
---
```
