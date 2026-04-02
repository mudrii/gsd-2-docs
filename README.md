# GSD-2 Documentation

Comprehensive documentation for **GSD** (Get Shit Done) — the autonomous coding agent built on the [Pi SDK](https://github.com/badlogic/pi-mono).

- **Current release:** v2.58.0 (2026-03-28)
- **License:** MIT
- **Package:** [`gsd-pi`](https://www.npmjs.com/package/gsd-pi) on npm
- **Source:** [github.com/gsd-build/gsd-2](https://github.com/gsd-build/gsd-2)
- **Discord:** [discord.gg/gsd](https://discord.gg/gsd)

> This documentation covers all released versions (v0.0.1 through v2.58.0). All content is fact-checked against the source code, official documentation, and online sources.

---

## Core Documentation

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Overview](01-architecture-overview.md) | Monorepo structure, bootstrap flow, native Rust engine, extension system, session management, headless mode, web UI, daemon |
| 02 | [Auto Mode & Dispatch](02-auto-mode-and-dispatch.md) | State machine, dispatch rules, phase lifecycle, quality gates, crash recovery, verification enforcement, git trailers |
| 03 | [Configuration & Preferences](03-configuration-and-preferences.md) | All preferences with types/defaults, capability-aware model routing (ADR-004), isolation modes, environment variables |
| 04 | [Extensions & Tools](04-extensions-and-tools.md) | 21+ bundled extensions, 5 agents, 56+ skills (30 packs), Ollama, Claude Code CLI, GitHub Sync, MCP client |
| 05 | [Version History](05-version-history.md) | Complete changelog for ~100 released versions (v0.0.1 → v2.58.0), grouped into 20 eras |
| 06 | [Feature Evolution](06-feature-evolution.md) | 45+ major features tracked across versions, introduction timeline for 84+ features |
| 07 | [User Guide](07-user-guide.md) | Installation, step/auto/headless/web modes, VS Code extension, commands, parallel orchestration, cost management |
| 08 | [Fact-Check Report](08-fact-check-report.md) | Online verification of npm, GitHub, VS Code Marketplace, Discord, web mode, daemon, SQLite engine |
| 09 | [Developer Guide](09-developer-guide.md) | Extension development, ADR-001/003/004, VS Code extension (3 phases), daemon/Discord, testing |
| 10 | [API & Internals](10-api-and-internals.md) | SQLite state engine, RPC v2, model providers, error pipeline, compaction, DB-backed tools |

## Supplementary Reference

Detailed reference documentation organized by topic:

### Getting Started & Reference

| Document | Description |
|----------|-------------|
| [Overview](docs/01-overview.md) | What GSD-2 is, key capabilities, modes of operation |
| [Installation](docs/02-installation.md) | Prerequisites, install, first-run wizard, quick-start |
| [Architecture](docs/03-architecture.md) | Package dependency graph, agent loop, session model, data flow |
| [Commands](docs/04-commands.md) | All CLI commands, slash commands, keyboard shortcuts |
| [Configuration](docs/05-configuration.md) | All preferences, environment variables, model routing, hooks |

### Core Features

| Document | Description |
|----------|-------------|
| [Auto Mode](docs/06-auto-mode.md) | State machine, phases, guided flow, crash recovery, verification, quality gates |
| [Parallel Orchestration](docs/07-parallel-orchestration.md) | Multi-worker setup, TUI monitor, self-healing, budget enforcement |
| [Headless Mode](docs/19-headless-mode.md) | Non-interactive execution, RPC v2, --bare/--events/--answers/--resume flags |
| [SQLite State Engine](docs/20-sqlite-state-engine.md) | Tool-driven writes, single-writer engine v2/v3, schema migrations, reconciliation |

### Integrations

| Document | Description |
|----------|-------------|
| [VS Code Extension](docs/16-vscode-extension.md) | Sidebar, @gsd chat, SCM integration, checkpoints, diagnostics, plan viewer |
| [Web Interface](docs/17-web-interface.md) | Browser-based UI, authentication, dashboard, mobile support |
| [Daemon & Discord](docs/18-daemon-and-discord.md) | Background daemon, Discord bot, launchd, event bridge, orchestrator |

### Packages

| Document | Description |
|----------|-------------|
| [pi-tui](docs/08-packages/pi-tui.md) | Terminal UI library — rendering, components, keyboard protocol |
| [pi-ai](docs/08-packages/pi-ai.md) | AI provider abstraction — streaming, models, 30+ providers |
| [pi-agent-core](docs/08-packages/pi-agent-core.md) | Agent loop — turn/tool/stop-reason cycle, hooks, events |
| [pi-coding-agent](docs/08-packages/pi-coding-agent.md) | Coding agent — session manager, tools, extension system |
| [studio](docs/08-packages/studio.md) | Electron desktop app — IPC bridge, three-process architecture |

### Developer Guides

| Document | Description |
|----------|-------------|
| [GSD Extension](docs/09-gsd-extension.md) | Full source walkthrough — 130+ files, state, doctor, preferences |
| [Building Coding Agents](docs/10-building-coding-agents.md) | 26-chapter synthesis — decomposition, context engineering |
| [Extending Pi](docs/11-extending-pi.md) | Extension development — lifecycle, events, custom tools, packaging |
| [Context & Hooks](docs/12-context-and-hooks.md) | Context pipeline, hook reference, injection patterns |

### Operations

| Document | Description |
|----------|-------------|
| [CI/CD Pipeline](docs/13-ci-cd.md) | Three-stage promotion (Dev → Test → Prod), PR risk checker |
| [Troubleshooting](docs/14-troubleshooting.md) | Installation, runtime errors, crash recovery, SQLite, web mode |
| [Changelog Summary](docs/15-changelog.md) | Condensed changelog with migration notes |

---

## Key Releases

| Version | Date | Highlights |
|---------|------|------------|
| v2.58.0 | 2026-03-28 | Ollama extension, Discord shard listeners, concurrent auto guard |
| v2.54.0 | 2026-03-27 | Headless M002, real-time parallel TUI monitor with self-healing |
| v2.52.0 | 2026-03-27 | VS Code extension phase 1, RPC v2, capability metadata routing |
| v2.46.0 | 2026-03-25 | Single-writer engine v2/v3, workflow logger, state machine guards |
| v2.44.0 | 2026-03-24 | SQLite state engine (tool-driven writes), Docker sandbox |
| v2.42.0 | 2026-03-22 | Declarative workflow engine (YAML), rule registry, event journal |
| v2.41.0 | 2026-03-21 | Web interface, doctor worktree checks, 80+ bug fixes |
| v2.39.0 | 2026-03-20 | GitHub sync, AI triage, /gsd doctor 13 enhancements, auto-PR |
| v2.30.0 | 2026-03-18 | ADR-003 pipeline simplification, skill authoring, workflow templates |
| v2.29.0 | 2026-03-17 | Largest release — 70+ fixes, CI/CD pipeline, massive refactoring |
| v2.14.0 | 2026-03-13 | Native Rust engine (NAPI), 10-100x speedups |
| v2.10.0 | 2026-03-12 | Worktree isolation, crash recovery, cost tracking |

---

## Statistics

- **Total documentation:** ~21,000+ lines across 35 files
- **Source files reviewed:** 1,304 TypeScript/JavaScript files
- **Extensions documented:** 21+ bundled extensions
- **Agents documented:** 5 bundled agents (scout, researcher, worker, javascript-pro, typescript-pro)
- **Skills documented:** 56+ skills across 30 packs
- **Versions covered:** ~100 released versions (v0.0.1 → v2.58.0)
- **Online fact-checks:** npm, GitHub, VS Code Marketplace, Discord verified (2026-04-02)

---

Generated on 2026-04-02 from GSD-2 source code at commit `main` (v2.58.0).
