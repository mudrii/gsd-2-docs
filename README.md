# GSD-2 Documentation

Comprehensive documentation for **GSD** (Get Shit Done) — the autonomous coding agent built on the Pi framework.

- **Current release:** v2.28.0 (2026-03-17)
- **License:** MIT
- **Package:** `gsd-pi` on npm
- **Source:** [github.com/gsd-build/gsd-2](https://github.com/gsd-build/gsd-2)

> This documentation covers released versions only (v2.24.0–v2.28.0). All content is fact-checked against the source code and official documentation.

---

## Contents

### Getting Started

| Document | Description |
|----------|-------------|
| [01 — Overview](docs/01-overview.md) | What GSD-2 is, key capabilities, package ecosystem, modes of operation |
| [02 — Installation](docs/02-installation.md) | Prerequisites, install, first-run wizard, quick-start workflow |

### Architecture

| Document | Description |
|----------|-------------|
| [03 — Architecture](docs/03-architecture.md) | Monorepo structure, package dependency graph, agent loop, session model, branchless worktree (ADR-001), data flow |

### Reference

| Document | Description |
|----------|-------------|
| [04 — Commands](docs/04-commands.md) | All CLI commands, slash commands, keyboard shortcuts |
| [05 — Configuration](docs/05-configuration.md) | All preferences, environment variables, model routing, git settings, hooks |

### Core Features

| Document | Description |
|----------|-------------|
| [06 — Auto Mode](docs/06-auto-mode.md) | State machine, phases, guided flow, crash recovery, verification gate, file formats |
| [07 — Parallel Orchestration](docs/07-parallel-orchestration.md) | Multi-worker setup (v2.24+), eligibility, NDJSON monitoring, budget enforcement, merge strategy |

### Packages

| Document | Description |
|----------|-------------|
| [pi-tui](docs/08-packages/pi-tui.md) | Terminal UI library — rendering, components, keyboard protocol |
| [pi-ai](docs/08-packages/pi-ai.md) | AI provider abstraction — streaming, models, 25+ providers |
| [pi-agent-core](docs/08-packages/pi-agent-core.md) | Agent loop — turn/tool/stop-reason cycle, hooks, events |
| [pi-coding-agent](docs/08-packages/pi-coding-agent.md) | Coding agent — session manager, tools, extension system, compaction |
| [studio](docs/08-packages/studio.md) | Electron desktop app — IPC bridge, three-process architecture, design tokens |

### Developer Guides

| Document | Description |
|----------|-------------|
| [09 — GSD Extension](docs/09-gsd-extension.md) | Full source walkthrough — 130+ files, state, doctor, preferences, worktrees, visualizer |
| [10 — Building Coding Agents](docs/10-building-coding-agents.md) | 26-chapter synthesis — decomposition, state machines, context engineering, parallelization |
| [11 — Extending Pi](docs/11-extending-pi.md) | Extension development — lifecycle, events, context/API, custom tools, commands, packaging |
| [12 — Context & Hooks](docs/12-context-and-hooks.md) | Context pipeline, hook reference, injection patterns, system prompt anatomy |

### Operations

| Document | Description |
|----------|-------------|
| [13 — CI/CD Pipeline](docs/13-ci-cd.md) | Three-stage promotion (Dev → Test → Prod), workflows, version strategy |
| [14 — Troubleshooting](docs/14-troubleshooting.md) | Installation issues, runtime errors, crash recovery, captures, visualizer |
| [15 — Changelog](docs/15-changelog.md) | Released versions v2.24.0–v2.28.0 with migration notes |

---

## Version Coverage

| Version | Date | Key Features |
|---------|------|-------------|
| v2.28.0 | 2026-03-17 | `gsd headless query`, `/gsd update`, HTML retrospective reports, reliability hardening |
| v2.27.0 | 2026-03-17 | HTML report generator, crash recovery for parallel orchestrator, AutoSession class, 7 extracted modules |
| v2.26.0 | 2026-03-17 | Model selector by provider, `require_slice_discussion`, worker NDJSON monitoring, hashline edit mode |
| v2.25.0 | 2026-03-16 | Native web search TUI rendering, meaningful commit messages, incremental memory, 14 new E2E tests |
| v2.24.0 | 2026-03-16 | **Parallel milestone orchestration**, headless `new-milestone`, interactive update prompt, `validate-milestone` phase |

---

## Quick Links

- [Install GSD](docs/02-installation.md#installation)
- [Start auto mode](docs/06-auto-mode.md#starting-auto-mode)
- [All slash commands](docs/04-commands.md#slash-commands)
- [Configuration reference](docs/05-configuration.md#preferences-reference)
- [Write an extension](docs/11-extending-pi.md#getting-started)
- [Troubleshoot issues](docs/14-troubleshooting.md)
