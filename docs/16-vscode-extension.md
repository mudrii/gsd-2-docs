# VS Code Extension

## Overview & Requirements

The GSD-2 VS Code extension (published by **FluxLabs**) brings the full GSD-2 coding agent into VS Code. It provides a sidebar dashboard for monitoring and controlling the agent, a `@gsd` chat participant for conversational interaction, real-time activity tracking, source control integration with per-file diffs and rollback, code lens actions, diagnostic integration, and 40+ commands accessible from the Command Palette.

The extension spawns `gsd --mode rpc` as a child process and communicates over JSON-RPC via stdin/stdout. Agent events stream in real time. A change tracker captures file state before modifications, enabling SCM diffs and checkpoint rollback. UI requests from the agent (questions, confirmations, selections) are handled via native VS Code dialogs.

### Requirements

| Requirement | Minimum Version |
|---|---|
| VS Code | 1.95.0 |
| Node.js | 22.0.0 |
| GSD CLI (`gsd-pi`) | Installed globally |
| Git | Installed and on PATH |

---

## Installation

1. Install the GSD CLI globally:

```
npm install -g gsd-pi
```

2. Install the extension from the VS Code Marketplace:
   - Open VS Code
   - Go to **Extensions** (`Cmd+Shift+X` / `Ctrl+Shift+X`)
   - Search for **"GSD"** (publisher: FluxLabs)
   - Click **Install**

The extension activates automatically when VS Code finishes startup (`onStartupFinished`). It is a **workspace** extension, meaning it runs in the context of your open workspace folder.

---

## Quick Start

1. Open a project folder in VS Code.
2. Click the **GSD icon** (robot) in the Activity Bar on the left sidebar.
3. Click **Start Agent** in the sidebar or run `Ctrl+Shift+P` > **GSD: Start Agent**.
4. Start working:
   - Click **Auto** in the sidebar to launch autonomous mode.
   - Or type `@gsd` in the VS Code Chat panel (`Cmd+Shift+I`) to talk to the agent directly.

A persistent status bar item at the bottom of VS Code shows connection status, the current model, cost, and a spinner when the agent is streaming.

---

## Sidebar Dashboard

The sidebar is a webview panel registered as the primary view in the GSD Activity Bar container. It provides a compact, card-based layout with a two-line header and collapsible sections.

### Header

The header displays at a glance:

- **Connection status** -- whether the agent is running
- **Model** -- the active provider and model ID
- **Session** -- current session name or ID
- **Messages** -- message count for the session
- **Thinking** -- current thinking level (off / low / medium / high)
- **Context** -- color-coded usage bar showing context window consumption
- **Cost** -- cumulative session cost

### Sections

All sections are collapsible and remember their state between sessions:

| Section | Contents |
|---|---|
| **Workflow** | Auto, Next, Quick, Capture, Status buttons |
| **Stats** | Token usage breakdown, cost, message and turn counts |
| **Actions** | New Session, Export HTML, Session Stats, Show History, Fork Session, Fix Errors, and more |
| **Settings** | Toggle auto-compaction, toggle auto-retry, steering mode, follow-up mode, approval mode |

The Settings section starts collapsed by default.

### Sidebar Messages

The sidebar routes workflow button actions through the VS Code Chat panel so responses are visible in the normal chat flow. For example, clicking **Auto** sends `@gsd /gsd auto` to the Chat panel.

---

## Workflow Controls

One-click buttons in the sidebar for GSD's core operating modes. All route through the Chat panel so you see the full response inline.

| Button | What It Does |
|---|---|
| **Auto** | Start autonomous mode -- the agent researches, plans, and executes |
| **Next** | Execute one unit of work, then pause |
| **Quick** | Quick task without planning (opens an input box for a description) |
| **Capture** | Capture a thought for later triage (opens an input box) |
| **Status** | Request a status report from the agent |

---

## Chat Integration (`@gsd`)

The extension registers a **chat participant** named `@gsd` (full name: "GSD Agent") that appears in VS Code's built-in Chat panel. The participant is marked as "sticky," meaning it stays selected once chosen.

### Usage

Open VS Code Chat with `Cmd+Shift+I` (or `Ctrl+Shift+I`) and type:

```
@gsd refactor the auth module to use JWT
@gsd /gsd auto
@gsd fix the errors in this file
```

### Auto-Start

If the agent is not running when you send a message, the chat participant automatically starts it. If startup fails, it displays an error with instructions to install `gsd-pi`.

### Context Injection

The chat participant automatically enriches your messages with relevant context:

- **File context** -- use `#file` references in your message to include specific files
- **Selection context** -- if you have code selected in the active editor, it is automatically included
- **Diagnostic context** -- when your message contains keywords like "fix", "error", "problem", "warning", "bug", "lint", or "diagnos", the active file's diagnostics from the Problems panel are appended automatically

### Streaming Response

While the agent works, the chat response stream shows:

- Progress indicators for each tool execution
- File anchors (clickable links to files the agent reads or modifies)
- Token usage footer at the end of each response (input tokens, output tokens, cost)

---

## Source Control Integration

Agent-modified files appear in a dedicated **"GSD Agent"** section in the Source Control panel. This is implemented as a custom `SourceControl` provider (ID: `gsd`) with QuickDiff support.

### How It Works

The **change tracker** captures the original content of every file before the agent modifies it. The **SCM provider** registers a `gsd-original` URI scheme to serve original file content, enabling VS Code's native diff editor.

### Actions

| Action | Where | Description |
|---|---|---|
| **Click a file** | SCM resource list | Opens a before/after diff in VS Code's native diff editor |
| **Accept** (checkmark icon) | Inline on each file | Accept the agent's changes for that file (clears tracking) |
| **Discard** (discard icon) | Inline on each file | Revert the file to its original content |
| **Accept All** (check-all icon) | SCM title bar | Accept all agent changes at once |
| **Discard All** (discard icon) | SCM title bar | Revert all agent modifications to their original state |

The SCM input box placeholder reads "Describe changes to accept..." and pressing Enter triggers Accept All.

### Gutter Indicators

VS Code's built-in gutter diff indicators (green/red bars) show exactly what lines changed, powered by the QuickDiff provider.

---

## Activity Feed

The **Activity** panel is a `TreeView` in the GSD sidebar container that shows a real-time log of every tool the agent executes.

### Display

Each activity item shows:

- **Status icon** -- running (sync~spin), success (check), or error (error)
- **Tool name and summary** -- e.g., "Read sidebar.ts", "Edit extension.ts", "Bash: npm test", "Grep: pattern"
- **Duration** -- how long the tool execution took
- **Click-to-open** -- for file operations (Read, Write, Edit), clicking the item opens the file

### Tool Icons

| Tool | Icon |
|---|---|
| Read | file |
| Write | new-file |
| Edit | edit |
| Bash | terminal |
| Grep | search |
| Glob | file-directory |
| Agent | organization |

### Controls

- **Clear Activity Feed** button in the view title bar (clear-all icon)
- Maximum items configurable via `gsd.activityFeedMaxItems` (default: 100, range: 10--500)

---

## Session Management

The **Sessions** panel is a `TreeView` that lists all past GSD sessions for the current workspace.

### Features

- Sessions are loaded from `.jsonl` files in the session directory (the same directory as the current session file)
- Listed in reverse chronological order (newest first)
- The current session is highlighted with a green indicator
- Click any session to switch to it
- Sessions persist to disk automatically

### Commands

| Command | Description |
|---|---|
| **GSD: New Session** | Start a fresh conversation |
| **GSD: Switch Session** | Load a different session via QuickPick |
| **GSD: Set Session Name** | Rename the current session |
| **GSD: Fork Session** | Fork from a previous message via QuickPick |
| **GSD: Refresh Sessions** | Refresh the session list (also available as a button in the view title bar) |

---

## Conversation History

The **Conversation History** panel is a webview that displays the full message log for the current session, fetched via the `get_messages` RPC call.

### Features

- Shows all user messages, assistant responses, and system messages
- Renders tool calls with their inputs and outputs
- Displays thinking blocks (collapsible)
- **Search/filter** -- find text within the conversation
- **Fork-from-here** -- fork the session from any message in the history

Open it with `Cmd+Shift+P` > **GSD: Show Conversation History**. The panel retains its state when hidden.

---

## Code Lens

The extension provides four inline code lens actions above every function and class declaration. Code lens is supported for **TypeScript**, **TypeScript React**, **JavaScript**, **JavaScript React**, **Python**, **Go**, and **Rust**.

### Symbol Detection

Symbols are detected via regex patterns that match:

- **TS/JS**: `function`, `async function`, `class`, `abstract class`, method declarations with visibility modifiers
- **Python**: `def`, `async def`, `class`
- **Go**: `func`, including receiver methods
- **Rust**: `fn`, `pub fn`, `async fn`

### Actions

| Code Lens | What It Does |
|---|---|
| **Ask GSD** | Explain the function or class |
| **Refactor** | Improve clarity, performance, or structure |
| **Find Bugs** | Review for bugs and edge cases |
| **Tests** | Generate test coverage |

Code lens can be disabled via the `gsd.codeLens` setting.

---

## File Decorations

Files that the agent has written or edited during the current session receive a **"G" badge** in the VS Code Explorer. This is implemented as a `FileDecorationProvider` that listens for `tool_execution_start` events with `Write` or `Edit` tool names.

The badge appears with:
- The letter **"G"** as the badge text
- A tooltip indicating modification by the GSD agent

Clear decorations with `Cmd+Shift+P` > **GSD: Clear File Decorations**. Decorations are also cleared when the agent disconnects.

---

## Line-Level Decorations

When the agent modifies a file, line-level decorations provide visual feedback directly in the editor.

### Decoration Types

| Decoration | Appearance |
|---|---|
| **Added lines** | Green background (`rgba(78, 201, 176, 0.07)`), green overview ruler marker |
| **Modified lines** | Yellow background (`rgba(204, 167, 0, 0.07)`), yellow overview ruler marker |
| **Gutter indicator** | 3px solid left border (`rgba(78, 201, 176, 0.4)`) on all agent-touched lines |

### Behavior

- Decorations update whenever the change tracker fires, when the active editor changes, or when the document text changes
- Hovering any decorated line shows "Modified by GSD Agent"
- Line-level diffs are computed by comparing the original (pre-agent) content with the current file content

---

## Diagnostics

The diagnostic bridge integrates VS Code's Problems panel with the GSD agent in two directions:

### Sending Diagnostics to the Agent

| Command | Scope | Description |
|---|---|---|
| **GSD: Fix Problems in File** | Active file | Reads all diagnostics for the active file and sends them as a "fix these problems" prompt |
| **GSD: Fix All Problems** | Workspace | Collects errors and warnings across the entire workspace and sends them to the agent |

### Automatic Diagnostic Context

In the chat participant, mentioning keywords like "fix", "error", "problem", "warning", "bug", "lint", or "diagnos" automatically includes the active file's diagnostics in the message.

### Agent Diagnostics

The extension also creates a `DiagnosticCollection` named "gsd" where the agent can surface its own findings. These can be cleared with **GSD: Clear Agent Diagnostics**.

---

## Checkpoints & Rollback

Automatic checkpoints are created at the start of each agent turn by the change tracker.

### Checkpoint Contents

Each checkpoint stores:
- A numeric ID
- A label (timestamp + first action, e.g., "10:32 -- Edit sidebar.ts")
- A timestamp
- A map of file paths to their original content at checkpoint creation time

### Actions

| Action | Description |
|---|---|
| **GSD: Restore Checkpoint** | Revert all file changes back to a specific checkpoint |
| **GSD: Discard All Changes** | Revert all agent modifications to their original state (SCM panel) |
| **Discard per-file** | Revert individual files from the SCM resource list |

Checkpoints are displayed newest-first. Clicking a checkpoint in the tree view triggers a restore.

---

## Plan Viewer

The plan viewer is a `TreeView` that shows a live execution plan of agent tool executions as they happen.

### Step Display

Each step shows:
- **Status icon** -- pending (circle-outline), running (sync~spin), done (check), error (error)
- **Description** -- what the tool is doing (e.g., "Edit sidebar.ts", "Bash: npm test")
- **Duration** -- execution time in milliseconds for completed steps, "running..." for in-progress steps
- **Tooltip** -- full tool name, description, status, and timestamp

The plan viewer clears when the agent disconnects. Use **GSD: Clear Plan View** to clear it manually.

---

## Git Integration

The extension provides git operations scoped to agent-modified files.

### Commands

| Command | Description |
|---|---|
| **GSD: Commit Agent Changes** | Stages all agent-modified files and commits them with a user-provided message. Default message: `feat: agent changes (N files)`. After committing, the change tracker clears (accepts all changes). |
| **GSD: Create Branch for Agent Work** | Prompts for a branch name (validated: no spaces), creates the branch, and switches to it. |
| **GSD: Show Agent Diff** | Runs `git diff` in the workspace and displays the output. |

---

## Approval Modes

The permission manager controls how much autonomy the agent has over file modifications and command execution.

| Mode | Behavior |
|---|---|
| **auto-approve** | Agent runs freely without prompts (default) |
| **ask** | Prompts before file writes and command execution |
| **plan-only** | Read-only mode -- agent can analyze but not modify files |

### Changing the Mode

- **Settings section** in the sidebar
- **GSD: Select Approval Mode** (`Cmd+Shift+P`) -- QuickPick with descriptions
- **GSD: Cycle Approval Mode** -- rotates through auto-approve, ask, plan-only

The mode is stored in the `gsd.approvalMode` workspace setting and reloads automatically when the configuration changes.

---

## Agent UI Requests

When the agent needs user input (questions, confirmations, multi-select), VS Code dialogs appear automatically. This prevents the agent from hanging on `ask_user_questions` and keeps the interaction within VS Code's native UI patterns.

---

## Bash Terminal

A dedicated pseudoterminal named **"GSD Agent"** routes the agent's Bash tool output in real time. When the agent runs a Bash command:

1. The terminal opens (preserving editor focus).
2. The command is displayed in gray (`$ command`).
3. Streaming output from `tool_execution_update` events is written as it arrives.
4. The terminal closes when the agent disconnects.

---

## Slash Command Completion

The extension provides a `CompletionItemProvider` that surfaces GSD slash commands when you type `/` at the start of a line. Completion is available in **Markdown**, **Plaintext**, **TypeScript**, **TypeScript React**, **JavaScript**, and **JavaScript React** files.

Commands are fetched from the running agent via the `get_commands` RPC call and cached. The cache refreshes when the connection is established. Each completion item shows the command name, description, and source (extension, prompt, or skill).

---

## Status Bar

A persistent status bar item appears at the bottom-left of VS Code:

- **Disconnected**: `$(hubot) GSD`
- **Connected**: `$(hubot) GSD | model-id | $0.0042`
- **Streaming**: adds a spinning sync icon

Clicking the status bar item opens the GSD sidebar. The status bar refreshes every 10 seconds and on connection state changes.

---

## Command Palette

All commands are accessible via `Cmd+Shift+P` (or `Ctrl+Shift+P`). The complete list:

### Agent Lifecycle

| Command | Description |
|---|---|
| `GSD: Start Agent` | Connect to the GSD agent |
| `GSD: Stop Agent` | Disconnect the agent |
| `GSD: Abort Current Operation` | Interrupt the current operation |
| `GSD: Steer Agent` | Send a steering message mid-operation |

### Sessions

| Command | Description |
|---|---|
| `GSD: New Session` | Start a fresh conversation |
| `GSD: Switch Session` | Load a different session |
| `GSD: Set Session Name` | Rename the current session |
| `GSD: Fork Session` | Fork from a previous message |
| `GSD: Refresh Sessions` | Refresh the session list |
| `GSD: Show Conversation History` | Open the conversation viewer |

### Messaging

| Command | Description |
|---|---|
| `GSD: Send Message` | Send a message to the agent |
| `GSD: Copy Last Response` | Copy the last agent response to the clipboard |
| `GSD: Run Bash Command` | Execute a shell command via the agent |
| `GSD: List Available Commands` | Browse available slash commands |

### Model & Thinking

| Command | Description |
|---|---|
| `GSD: Switch Model` | Pick a model from QuickPick |
| `GSD: Cycle Model` | Rotate to the next model |
| `GSD: Set Thinking Level` | Choose off / low / medium / high |
| `GSD: Cycle Thinking Level` | Rotate through thinking levels |

### Context

| Command | Description |
|---|---|
| `GSD: Compact Context` | Trigger context compaction |
| `GSD: Export Conversation as HTML` | Save the session as an HTML file |
| `GSD: Show Session Stats` | Display token usage and cost |

### Source Control

| Command | Description |
|---|---|
| `GSD: Accept All Agent Changes` | Accept all SCM changes |
| `GSD: Discard All Agent Changes` | Revert all agent modifications |
| `Accept Changes` | Accept changes for a single file (inline SCM action) |
| `Discard Changes` | Discard changes for a single file (inline SCM action) |
| `GSD: Restore Checkpoint` | Restore to a specific checkpoint |

### Diagnostics

| Command | Description |
|---|---|
| `GSD: Fix Problems in File` | Send the active file's diagnostics to the agent |
| `GSD: Fix All Problems` | Send all workspace errors to the agent |
| `GSD: Clear Agent Diagnostics` | Clear the agent's diagnostic collection |

### Git

| Command | Description |
|---|---|
| `GSD: Commit Agent Changes` | Git commit agent-modified files |
| `GSD: Create Branch for Agent Work` | Create a new branch for agent work |
| `GSD: Show Agent Diff` | View git diff of agent changes |

### Code Lens

| Command | Description |
|---|---|
| `GSD: Ask About Symbol` | Explain the symbol at cursor |
| `GSD: Refactor Symbol` | Refactor the symbol at cursor |
| `GSD: Find Bugs in Symbol` | Review the symbol for bugs |
| `GSD: Generate Tests for Symbol` | Generate tests for the symbol |

### Permissions & Modes

| Command | Description |
|---|---|
| `GSD: Select Approval Mode` | Choose auto-approve / ask / plan-only |
| `GSD: Cycle Approval Mode` | Rotate through approval modes |
| `GSD: Toggle Steering Mode` | Toggle steering mode (all / one-at-a-time) |
| `GSD: Toggle Follow-Up Mode` | Toggle follow-up mode (all / one-at-a-time) |
| `GSD: Toggle Auto-Retry` | Toggle automatic retry on errors |
| `GSD: Abort Retry` | Cancel a retry in progress |

### Display

| Command | Description |
|---|---|
| `GSD: Clear File Decorations` | Remove "G" badges from Explorer |
| `GSD: Clear Activity Feed` | Clear the Activity panel |
| `GSD: Clear Plan View` | Clear the Plan Viewer |

---

## Configuration

All settings are under the `gsd` namespace. Configure them in VS Code Settings (`Cmd+,`) or in `.vscode/settings.json`.

| Setting | Type | Default | Description |
|---|---|---|---|
| `gsd.binaryPath` | `string` | `"gsd"` | Path to the GSD binary. Change this if `gsd` is not on your PATH. |
| `gsd.autoStart` | `boolean` | `false` | Automatically start the GSD agent when the extension activates. |
| `gsd.autoCompaction` | `boolean` | `true` | Enable automatic context compaction. Applied when the agent starts. |
| `gsd.codeLens` | `boolean` | `true` | Show code lens actions (Ask GSD, Refactor, Find Bugs, Tests) above functions and classes. |
| `gsd.showProgressNotifications` | `boolean` | `false` | Show a progress notification with a cancel button while the agent is working. Disabled by default because the Chat panel provides inline progress. |
| `gsd.activityFeedMaxItems` | `number` | `100` | Maximum number of items shown in the Activity feed. Range: 10--500. |
| `gsd.showContextWarning` | `boolean` | `true` | Show a warning notification when context window usage exceeds the threshold. |
| `gsd.contextWarningThreshold` | `number` | `80` | Context window usage percentage (50--95) that triggers a warning. |
| `gsd.approvalMode` | `string` | `"auto-approve"` | Agent permission mode. Values: `"auto-approve"` (agent runs freely), `"ask"` (prompt before file changes), `"plan-only"` (read-only, no writes). |

---

## Keyboard Shortcuts

All shortcuts use a two-chord pattern starting with `Cmd+Shift+G` (or `Ctrl+Shift+G` on Windows/Linux):

| Shortcut (Mac) | Shortcut (Win/Linux) | Command |
|---|---|---|
| `Cmd+Shift+G` `Cmd+Shift+N` | `Ctrl+Shift+G` `Ctrl+Shift+N` | New Session |
| `Cmd+Shift+G` `Cmd+Shift+P` | `Ctrl+Shift+G` `Ctrl+Shift+P` | Send Message |
| `Cmd+Shift+G` `Cmd+Shift+A` | `Ctrl+Shift+G` `Ctrl+Shift+A` | Abort |
| `Cmd+Shift+G` `Cmd+Shift+I` | `Ctrl+Shift+G` `Ctrl+Shift+I` | Steer Agent |
| `Cmd+Shift+G` `Cmd+Shift+M` | `Ctrl+Shift+G` `Ctrl+Shift+M` | Cycle Model |
| `Cmd+Shift+G` `Cmd+Shift+T` | `Ctrl+Shift+G` `Ctrl+Shift+T` | Cycle Thinking Level |

---

## Architecture Summary

The extension is composed of 19 TypeScript source files, each responsible for a single concern:

| File | Component |
|---|---|
| `extension.ts` | Entry point -- activates all components, registers all commands |
| `gsd-client.ts` | RPC client -- spawns `gsd --mode rpc`, JSON line framing, all RPC methods |
| `sidebar.ts` | Sidebar webview -- compact dashboard with collapsible sections |
| `chat-participant.ts` | `@gsd` chat participant -- context injection, streaming responses |
| `activity-feed.ts` | Activity feed TreeView -- real-time tool execution log |
| `session-tree.ts` | Sessions TreeView -- browse and switch sessions |
| `conversation-history.ts` | Conversation history webview -- full message viewer with search and fork |
| `change-tracker.ts` | Change tracker -- captures file snapshots for diff and rollback |
| `scm-provider.ts` | SCM provider -- "GSD Agent" in Source Control with QuickDiff |
| `checkpoints.ts` | Checkpoint TreeView -- snapshot/restore per agent turn |
| `code-lens.ts` | Code lens -- Ask, Refactor, Find Bugs, Tests above symbols |
| `file-decorations.ts` | File decorations -- "G" badge on modified files in Explorer |
| `line-decorations.ts` | Line decorations -- green/yellow highlights on changed lines |
| `diagnostics.ts` | Diagnostic bridge -- reads Problems panel, surfaces agent findings |
| `git-integration.ts` | Git integration -- commit, branch, diff for agent changes |
| `permissions.ts` | Permission manager -- approval mode cycling and enforcement |
| `slash-completion.ts` | Slash completion -- `/` auto-complete for GSD commands |
| `bash-terminal.ts` | Bash terminal -- pseudoterminal for agent shell output |
| `plan-viewer.ts` | Plan viewer TreeView -- live execution plan of tool steps |
