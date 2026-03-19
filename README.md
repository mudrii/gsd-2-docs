# GSD-2 Documentation

Comprehensive documentation for **GSD** (Get Shit Done) — the autonomous coding agent built on the [Pi SDK](https://github.com/badlogic/pi-mono).

- **Current release:** v2.33.1 (2026-03-19)
- **License:** MIT
- **Package:** [`gsd-pi`](https://www.npmjs.com/package/gsd-pi) on npm
- **Source:** [github.com/gsd-build/gsd-2](https://github.com/gsd-build/gsd-2)
- **Discord:** [discord.gg/gsd](https://discord.gg/gsd)

> This documentation covers all released versions (v0.0.1 through v2.33.1). All content is fact-checked against the source code, official documentation, and online sources.

---

## Core Documentation

| # | Document | Description |
|---|----------|-------------|
| 01 | [Architecture Overview](01-architecture-overview.md) | Monorepo structure, bootstrap flow, native Rust engine, extension system, session management, headless mode |
| 02 | [Auto Mode & Dispatch](02-auto-mode-and-dispatch.md) | State machine (`deriveState` → `resolveDispatch`), 16 dispatch rules, phase lifecycle, crash recovery, timeout supervision, verification enforcement |
| 03 | [Configuration & Preferences](03-configuration-and-preferences.md) | All preferences with types/defaults, per-phase model selection, token profiles, git isolation modes, workflow modes, full working example |
| 04 | [Extensions & Tools](04-extensions-and-tools.md) | All 17 bundled extensions, 5 agents, 16 skills, dashboard/visualizer, Browser Tools, Search providers, MCP client, Remote Questions, TTSR |
| 05 | [Version History](05-version-history.md) | Complete changelog for all 62 released versions (v0.1.6 → v2.33.1), grouped into 15 eras |
| 06 | [Feature Evolution](06-feature-evolution.md) | 25 major features tracked across versions, introduction timeline for 55+ features |
| 07 | [User Guide](07-user-guide.md) | Installation, onboarding, step/auto/headless modes, two-terminal workflow, git strategy, teams, parallel orchestration, cost management, troubleshooting |
| 08 | [Fact-Check Report](08-fact-check-report.md) | Online verification of npm package, GitHub repo, Pi SDK, VS Code extension, Discord, and documentation accuracy |
| 09 | [Developer Guide](09-developer-guide.md) | Extension development, Pi SDK overview, context pipeline, UI/TUI system, ADR-001 & ADR-003, VS Code extension, Studio app |
| 10 | [API & Internals](10-api-and-internals.md) | ExtensionContext/API, custom tools, commands, state management, system prompt modification, compaction, model providers, error handling, key rules |

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
| [Auto Mode](docs/06-auto-mode.md) | State machine, phases, guided flow, crash recovery, verification |
| [Parallel Orchestration](docs/07-parallel-orchestration.md) | Multi-worker setup, eligibility, monitoring, budget enforcement |

### Packages

| Document | Description |
|----------|-------------|
| [pi-tui](docs/08-packages/pi-tui.md) | Terminal UI library — rendering, components, keyboard protocol |
| [pi-ai](docs/08-packages/pi-ai.md) | AI provider abstraction — streaming, models, 25+ providers |
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
| [CI/CD Pipeline](docs/13-ci-cd.md) | Three-stage promotion (Dev → Test → Prod) |
| [Troubleshooting](docs/14-troubleshooting.md) | Installation issues, runtime errors, crash recovery |
| [Changelog Summary](docs/15-changelog.md) | Condensed changelog with migration notes |

---

## Key Releases

| Version | Date | Highlights |
|---------|------|------------|
| v2.33.1 | 2026-03-19 | Lock cleanup, worktree sync safety, dispatch loop hardening |
| v2.32.0 | 2026-03-19 | Health widget, environment health checks, progress score |
| v2.30.0 | 2026-03-18 | ADR-003 pipeline simplification, built-in skill authoring, workflow templates |
| v2.29.0 | 2026-03-17 | Largest release — 70+ fixes, VS Code extension, massive refactoring |
| v2.24.0 | 2026-03-16 | Parallel milestone orchestration, validate-milestone phase |
| v2.17.0 | 2026-03-14 | Token optimization profiles, complexity-based model routing |
| v2.14.0 | 2026-03-13 | Native Rust engine (NAPI), 10-100x speedups |
| v2.10.0 | 2026-03-12 | Worktree isolation, crash recovery, cost tracking |

---

## Statistics

- **Total documentation:** ~24,000 lines across 30 files
- **Source files reviewed:** 487 TypeScript/JavaScript files
- **Extensions documented:** 17 bundled extensions
- **Agents documented:** 5 bundled agents (scout, researcher, worker, javascript-pro, typescript-pro)
- **Skills documented:** 16 bundled skills
- **Versions covered:** 62 released versions (v0.0.1 → v2.33.1)
- **Online fact-checks:** npm, GitHub, VS Code Marketplace, Discord verified

---

Generated on 2026-03-19 from GSD-2 source code at commit `main` (v2.33.1).
