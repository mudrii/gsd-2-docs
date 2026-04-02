# Headless Mode

Headless mode runs GSD commands without the TUI. It spawns a child process in RPC mode, auto-responds to extension UI requests, and streams progress to stderr. Designed for CI pipelines, cron jobs, remote orchestration, and scripted workflows where no human is sitting at a terminal.

```
gsd headless [flags] [command] [args...]
```

Default command is `auto` — so `gsd headless` is equivalent to `gsd headless auto`.

---

## Evolution

Headless mode has been built up across several releases:

| Version | Capability |
|---------|-----------|
| 2.50.0 | Auto-mode timeout disabled, lock-guard auto-select |
| 2.52.0 | `--bare` mode, RPC v2 init handshake, `execution_complete` events |
| 2.54.0 | Headless integration hardening |
| 2.55.0 | Colorized verbose output (thinking blocks, phase indicators, cost, durations), text mode observability, UAT pause skip |
| 2.57.0 | `--resume` flag with prefix matching, cold auto bootstrap from DB, `execution_complete`-based completion, `completed` status in exit code mapper |
| 2.58.0 | `execution_complete` skipped for multi-turn commands (`auto`/`next`) |

---

## Exit Codes

Every headless run terminates with a well-defined exit code:

| Code | Constant | Meaning |
|------|----------|---------|
| 0 | `EXIT_SUCCESS` | Command finished successfully (`success`, `complete`, or `completed` status) |
| 1 | `EXIT_ERROR` | Error or timeout |
| 10 | `EXIT_BLOCKED` | Command reported a blocker |
| 11 | `EXIT_CANCELLED` | SIGINT or SIGTERM received |

The mapping is handled by `mapStatusToExitCode()` in `headless-events.ts`. Both `"complete"` and `"completed"` map to 0 — this accommodates the RPC v2 protocol which emits `"completed"` in `execution_complete` events.

---

## CLI Flags

### Output Control

**`--json`** — Alias for `--output-format stream-json`. Streams JSONL events to stdout.

**`--output-format <fmt>`** — Choose the output format:

- `text` (default) — Human-readable progress on stderr. Tool calls, phase transitions, and cost summaries are printed with ANSI colors when stderr is a TTY.
- `stream-json` — JSONL events streamed to stdout in real time. Same as `--json`.
- `json` — Batch mode. Events are collected silently; a single `HeadlessJsonResult` object is written to stdout at exit.

**`--bare`** — Minimal context mode. Suppresses loading of CLAUDE.md, AGENTS.md, user skills, and project preferences. Propagated to the child process via an extra CLI arg. Useful for CI environments where you want deterministic behavior without user-specific configuration.

**`--verbose`** — Show tool calls, thinking blocks, and phase transitions in text output. Without this flag, text mode shows a truncated one-liner summary of accumulated LLM text before tool calls.

**`--events <types>`** — Filter JSONL output to specific event types. Takes a comma-separated list. Implies `--json`. Example:

```
gsd headless --events agent_end,extension_ui_request auto
```

### Execution Control

**`--timeout N`** — Overall timeout in milliseconds. Default: 300,000 (5 minutes). Set to 0 to disable. For `new-milestone`, the default is elevated to 600,000 (10 minutes). For `auto` mode, the timeout is automatically disabled (the auto supervisor has its own internal per-unit timeout).

**`--model ID`** — Override the model for this run.

**`--max-restarts N`** — Maximum auto-restarts on crash. Default: 3, set to 0 to disable. Restarts use exponential backoff capped at 30 seconds. SIGINT/SIGTERM never triggers a restart.

### Session Management

**`--resume <id>`** — Resume a prior session by ID. Supports prefix matching: if `id` is a unique prefix of a session ID, that session is selected. Ambiguous prefixes produce an error listing all matches. After resolution, the headless orchestrator calls `client.switchSession()` to restore the session.

### Answer Injection

**`--answers <path>`** — Load a JSON file of pre-supplied answers and secrets. The file is validated on load and its contents are used to auto-respond to `ask_user_questions` tool calls during execution. See [Answer Injection](#answer-injection) for the file format.

### Supervised Mode

**`--supervised`** — Forward interactive UI requests to an external orchestrator via stdout/stdin. Implies `--json`. Cannot be combined with `--context -` (both require stdin). See [Supervised Mode](#supervised-mode).

**`--response-timeout N`** — Timeout in milliseconds for the orchestrator to respond to a forwarded request. Default: 30,000. If the timeout expires, the request is auto-responded using the default handler.

### New Milestone Flags

These flags apply when the command is `new-milestone`:

**`--context <path>`** — Path to a specification or PRD file. Use `-` for stdin.

**`--context-text <text>`** — Inline specification text (alternative to `--context`).

**`--auto`** — Chain into auto-mode after milestone creation succeeds. The overall timeout is disabled for the auto-mode phase.

---

## Commands

### `auto` (default)

Run all queued units continuously. This is a multi-turn command — completion is detected via terminal notifications (`"Auto-mode stopped..."`) rather than `execution_complete` events. The overall timeout is disabled by default.

```
gsd headless
gsd headless auto
gsd headless --bare auto
```

### `next`

Run one unit. Also a multi-turn command — `execution_complete` events are skipped, and completion is detected via `"Step-mode stopped..."` notifications.

```
gsd headless next
```

### `status`

Show the progress dashboard. This is a quick command — resolves on the first `agent_end` event.

```
gsd headless status
```

### `new-milestone`

Create a milestone from a specification document. Requires `--context` or `--context-text`. Bootstraps `.gsd/` if it does not exist. Writes context to `.gsd/runtime/headless-context.md` for the RPC child to read.

```
gsd headless new-milestone --context spec.md
gsd headless new-milestone --context-text "Build a REST API for user management"
cat spec.md | gsd headless new-milestone --context -
gsd headless new-milestone --context spec.md --auto
```

### `query`

Instant read-only state snapshot. No LLM session is spawned — runs in roughly 50ms. Returns JSON to stdout with three top-level keys:

```json
{
  "state": { ... },
  "next": { "action": "dispatch", "unitType": "task", "unitId": "T1.1.1" },
  "cost": { "workers": [...], "total": 0.42 }
}
```

- **`state`** — Output of `deriveState()`: phase, milestones, progress, blockers.
- **`next`** — Dry-run dispatch preview. `action` is `"dispatch"`, `"stop"`, or `"skip"`. When `"dispatch"`, includes `unitType` and `unitId`. When `"stop"`, includes `reason`.
- **`cost`** — Aggregated parallel worker costs with per-worker breakdown (milestoneId, pid, state, cost, lastHeartbeat).

The query module loads GSD extension modules via `jiti` (a TypeScript runtime loader) because headless-query is imported directly from cli.ts, bypassing the extension loader's jiti setup.

```
gsd headless query
```

### Quick Commands

These commands resolve on the first `agent_end` event without waiting for terminal notifications:

`status`, `queue`, `history`, `hooks`, `export`, `stop`, `pause`, `capture`, `skip`, `undo`, `knowledge`, `config`, `prefs`, `cleanup`, `migrate`, `doctor`, `remote`, `help`, `steer`, `triage`, `visualize`.

---

## RPC Protocol v2

Headless mode communicates with the child process over an RPC protocol using JSON lines on stdin/stdout.

### Protocol Negotiation

After starting the RPC client, headless mode attempts a v2 init handshake:

```typescript
await client.init({ clientId: 'gsd-headless' })
```

If init succeeds, v2 is enabled and the client receives structured `execution_complete` events. If init fails (older server), headless falls back to v1 string-matching for completion detection.

### Init Command and Result

The init command is sent as:

```json
{ "type": "init", "protocolVersion": 2, "clientId": "gsd-headless" }
```

The server responds with:

```typescript
interface RpcInitResult {
  protocolVersion: 2
  sessionId: string
  capabilities: {
    events: string[]    // list of supported event types
    commands: string[]  // list of supported command types
  }
}
```

### Key v2 Event Types

**`execution_complete`** — Emitted when a prompt/steer/follow_up finishes:

```typescript
interface RpcExecutionCompleteEvent {
  type: "execution_complete"
  runId: string
  status: "completed" | "error" | "cancelled"
  reason?: string
  stats: SessionStats
}
```

For single-turn commands, `execution_complete` triggers immediate completion with the mapped exit code. For multi-turn commands (`auto`, `next`), `execution_complete` is intentionally skipped — their completion is detected via terminal notifications from the extension.

**`cost_update`** — Emitted per-turn with running cost data:

```typescript
interface RpcCostUpdateEvent {
  type: "cost_update"
  runId: string
  turnCost: number
  cumulativeCost: number
  tokens: {
    input: number
    output: number
    cacheRead: number
    cacheWrite: number
  }
}
```

### RPC Commands

Commands are sent as JSON lines on stdin. Key types used by headless mode:

| Type | Purpose |
|------|---------|
| `prompt` | Send a command (e.g., `/gsd auto`) |
| `steer` | Inject steering guidance mid-conversation |
| `follow_up` | Send a follow-up message |
| `init` | v2 protocol handshake |
| `switch_session` | Resume a prior session |

### Extension UI Protocol

Extensions can request user input via `extension_ui_request` events. Headless mode auto-responds to these:

| Method | Auto-Response |
|--------|--------------|
| `select` | First option (or "Force start" for lock-guard prompts) |
| `confirm` | `{ confirmed: true }` |
| `input` | `{ value: '' }` (empty string) |
| `editor` | `{ value: prefill }` (returns the prefill content) |
| `notify` | `{ value: '' }` (fire-and-forget) |
| `setStatus` | `{ value: '' }` (fire-and-forget) |
| `setWidget` | `{ value: '' }` (fire-and-forget) |
| `setTitle` | `{ value: '' }` (fire-and-forget) |
| `set_editor_text` | `{ value: '' }` (fire-and-forget) |

Fire-and-forget methods (`notify`, `setStatus`, `setWidget`, `setTitle`, `set_editor_text`) are never forwarded to answer injection or supervised mode.

---

## Answer Injection

The `--answers` flag loads a JSON file that pre-supplies responses to interactive questions and injects secrets as environment variables.

### File Format

```json
{
  "questions": {
    "question_id_1": "Selected option text",
    "question_id_2": ["Option A", "Option B"]
  },
  "secrets": {
    "API_KEY": "sk-...",
    "DATABASE_URL": "postgres://..."
  },
  "defaults": {
    "strategy": "first_option"
  }
}
```

**`questions`** — Maps question IDs to answers. A string value selects a single option; a string array selects multiple options (for `allowMultiple` questions). The answer must exactly match one of the presented options. Question IDs come from `ask_user_questions` tool calls — the injector observes `tool_execution_start` events to capture question metadata (header, question text, options) before the corresponding UI request arrives.

**`secrets`** — Key-value pairs injected as environment variables into the RPC child process. Tracked for usage reporting.

**`defaults.strategy`** — Fallback when a question has no matching answer:

- `"first_option"` (default) — Falls through to the auto-responder, which picks the first option.
- `"cancel"` — Cancels the request.

### Metadata Observation

The injector watches for `tool_execution_start` events where `toolName === 'ask_user_questions'`. From these, it extracts question metadata (id, header, question text, options, allowMultiple flag) and indexes them by title string. When a `select` UI request arrives, the injector looks up the title to find the question ID and maps it to the pre-supplied answer.

If a UI request arrives before its metadata (out-of-order delivery), the event is deferred for up to 500ms. If metadata still has not arrived, the default strategy applies.

### Usage Reporting

At exit, headless prints answer injection statistics:

```
[headless] Answers: 3 answered, 1 defaulted, 2 secrets
[answers] Warning: question ID 'unused_q' was never matched
[answers] Warning: secret 'UNUSED_KEY' was provided but never requested
```

### Use Cases

- **CI pipelines**: Pre-supply project configuration choices so headless runs are fully deterministic.
- **Automated testing**: Provide known answers to extension questions to test specific code paths.
- **Secrets injection**: Pass API keys and credentials without storing them in the project.

---

## Session Resumption

The `--resume <id>` flag resumes a prior headless session.

### Resolution Logic

1. Exact ID match is checked first.
2. If no exact match, prefix matching is attempted.
3. A unique prefix selects that session.
4. Zero matches produces: `No session matching '<prefix>' found`
5. Multiple matches produces: `Ambiguous session prefix '<prefix>' matches N sessions:` followed by the matching IDs.

After resolution, the orchestrator calls `client.switchSession(session.path)`. If the switch is cancelled by an extension, headless exits with code 1.

```
gsd headless --resume abc123 auto
gsd headless --resume abc auto        # prefix match
```

---

## Event Streaming

With `--json` or `--output-format stream-json`, all RPC events from the child process are forwarded to stdout as JSONL (one JSON object per line).

### Filtering

The `--events` flag restricts which event types are forwarded:

```
gsd headless --events agent_end,cost_update,extension_ui_request auto
```

Only events whose `type` field matches one of the comma-separated values are emitted. All events are still tracked internally for completion detection and statistics.

### Batch JSON Mode

With `--output-format json`, no events are streamed during execution. Instead, a single structured result object is written to stdout at exit:

```typescript
interface HeadlessJsonResult {
  status: 'success' | 'error' | 'blocked' | 'cancelled' | 'timeout'
  exitCode: number
  sessionId?: string
  duration: number           // milliseconds
  cost: {
    total: number            // USD
    input_tokens: number
    output_tokens: number
    cache_read_tokens: number
    cache_write_tokens: number
  }
  toolCalls: number
  events: number
  milestone?: string
  phase?: string
  nextAction?: string
  artifacts?: string[]
  commits?: string[]
}
```

Cost aggregation uses a cumulative-max pattern: each `cost_update` event's cumulative values are compared against the running maximum. This handles out-of-order or duplicate events gracefully.

---

## Text Mode Observability

In the default `text` output format, progress is written to stderr with ANSI colors (respects `NO_COLOR` env var and non-TTY detection).

### Without `--verbose`

LLM text deltas are accumulated into a buffer. When a tool call starts or the message ends, the buffer is flushed as a single truncated line:

```
[thinking] The user wants to implement a REST API for user management with CRUD endpoints...
```

### With `--verbose`

Full streaming output is shown with labeled blocks:

**Tool calls** — Start and end events with argument summaries and durations:

```
  [tool]    Read src/api/users.ts
  [tool]    Read done 42ms
  [tool]    Bash npm test
  [tool]    Bash error 3.2s
```

Tool argument summarization is context-aware: file tools show paths, bash shows the command (truncated at 80 chars), grep shows the pattern, GSD tools show milestone/slice/task IDs.

**Thinking blocks** — Streamed inline between `[thinking]` markers.

**Text blocks** — Streamed inline between `[text]` markers.

**Phase transitions** — Shown when `setStatus` events carry a `statusKey`:

```
[phase]   Milestone M001 -- Implementing core API
[phase]   Slice S01 -- User authentication
[phase]   Task T01 -- Create user model
[phase]   Phase: discuss -- Planning slice structure
```

**Notifications** — Important notifications (committed, verification gate, milestone, blocked) are displayed in bold:

```
[gsd]     Committed: feat(api): add user endpoints
[gsd]     Verification gate: PASS
[gsd]     Blocked: Missing database credentials
```

**Session lifecycle** — Agent start/end with cost summary:

```
[agent]   Session started
[agent]   Session ended ($0.0342, 15847 tokens)
```

---

## Supervised Mode

`--supervised` turns headless into a proxy between an external orchestrator and the GSD agent. Interactive UI requests that would normally be auto-responded are instead forwarded to stdout, and the orchestrator's responses are read from stdin.

### Protocol

**Outbound (stdout)** — Extension UI requests are emitted as JSONL. If the orchestrator does not respond within `--response-timeout` (default 30s), the request is auto-responded and a timeout event is emitted:

```json
{ "type": "supervised_timeout", "id": "req_123", "method": "select" }
```

**Inbound (stdin)** — The orchestrator sends JSONL commands:

| Message Type | Fields | Effect |
|-------------|--------|--------|
| `extension_ui_response` | `id`, `value` / `values` / `confirmed` / `cancelled` | Responds to a pending UI request |
| `prompt` | `message` | Sends a new prompt to the agent |
| `steer` | `message` | Sends a steering message |
| `follow_up` | `message` | Sends a follow-up message |

### Stdin Close Fallback

If the orchestrator's stdin closes (pipe break, process exit), supervised mode falls back to auto-response for all subsequent requests and logs a warning:

```
[headless] Warning: orchestrator stdin closed, falling back to auto-response
```

---

## Auto-Mode Integration

### Timeout Handling

When `command` is `auto`, the default 5-minute timeout is automatically disabled (set to 0). Auto-mode sessions can run for minutes to hours; the auto supervisor has its own internal per-unit timeout. Users can still override with `--timeout` if needed.

### Lock-Guard Auto-Select

If auto-mode is already running and a `select` UI request arrives with a title containing "Auto-mode is running", the auto-responder picks the "Force start" option instead of the default first option. This allows headless runs to break through stale lock files.

### Milestone Chaining

When `--auto` is combined with `new-milestone`, headless watches for "Milestone MX ready." notifications. If the milestone is created successfully (not blocked, exit code 0), a second `/gsd auto` prompt is sent automatically. The overall timeout is disabled for the auto-mode phase. Completion state is fully reset between the two phases.

---

## Completion Detection

Headless mode uses multiple strategies to detect when a command has finished:

1. **`execution_complete` event (v2)** — The primary mechanism for single-turn commands. Carries a status field mapped to exit codes. Skipped for multi-turn commands (`auto`, `next`).

2. **Terminal notifications** — For multi-turn commands. The extension emits `notify` events starting with `"Auto-mode stopped"` or `"Step-mode stopped"`. These are the actual stop signals from `stopAuto()`.

3. **Quick command detection** — Commands in the `QUICK_COMMANDS` set resolve on the first `agent_end` event.

4. **Idle timeout** — Fallback. After the first tool call, if no events arrive for 15 seconds (120 seconds for `new-milestone`), the session is considered complete.

5. **Child process exit** — If the RPC child exits unexpectedly, headless resolves with `EXIT_ERROR`.

### Blocked Detection

Blocked state is detected separately from completion. A notification containing `"blocked:"` (case-insensitive) sets the blocked flag. The exit code is then `EXIT_BLOCKED` (10) instead of `EXIT_SUCCESS`.

---

## Crash Recovery

Headless mode auto-restarts on crash up to `--max-restarts` times (default 3). The restart loop:

1. Runs `runHeadlessOnce()`.
2. On success (exit 0) or blocked (exit 10) — exits immediately.
3. On SIGINT/SIGTERM — exits immediately (no restart).
4. On error — increments restart count, waits with exponential backoff (`min(5000 * attempt, 30000)` ms), and retries.
5. If max restarts reached — exits with the error code.

---

## Headless Environment

The child process is spawned with `GSD_HEADLESS=1` in its environment. Extensions can check this variable to alter behavior — for example, the GSD extension skips the UAT human pause when running headless.

---

## Summary Line

At exit, headless always prints a summary to stderr:

```
[headless] Status: complete
[headless] Duration: 42.3s
[headless] Events: 187 total, 23 tool calls
[headless] Event filter: agent_end, cost_update
[headless] Restarts: 1
[headless] Answers: 3 answered, 1 defaulted, 2 secrets
```

On failure, the last 5 events are printed for diagnostics:

```
[headless] Last events:
  tool_execution_start: Read
  tool_execution_end: Read
  extension_ui_request: notify: Blocked: missing credentials
```

---

## Examples

### CI Pipeline

```bash
# Run all queued work, structured JSON result
gsd headless --output-format json auto > result.json
exit_code=$?
if [ $exit_code -eq 10 ]; then
  echo "Blocked — needs human intervention"
elif [ $exit_code -ne 0 ]; then
  echo "Failed"
fi
```

### Scripted Milestone Creation

```bash
# Create milestone from PRD, then execute it
gsd headless new-milestone --context spec.md --auto --verbose
```

### State Polling (no LLM cost)

```bash
# Check project state without spawning an LLM session
gsd headless query | jq '.next'
```

### Filtered Event Monitoring

```bash
# Watch only cost updates and agent lifecycle
gsd headless --events cost_update,agent_start,agent_end auto
```

### Pre-Supplied Answers

```bash
# answers.json:
# {
#   "questions": { "db_provider": "PostgreSQL" },
#   "secrets": { "DATABASE_URL": "postgres://..." },
#   "defaults": { "strategy": "first_option" }
# }
gsd headless --answers answers.json auto
```

### Resume a Session

```bash
# Resume by full ID or unique prefix
gsd headless --resume abc123def auto
gsd headless --resume abc auto
```

### Minimal Context for Ecosystem Use

```bash
# Skip all user config, skills, CLAUDE.md — pure project execution
gsd headless --bare auto
```
