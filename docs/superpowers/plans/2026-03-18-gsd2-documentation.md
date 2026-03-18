# GSD-2 Comprehensive Documentation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Generate complete, fact-checked documentation for gsd-2 v2.28.0 (released versions only) in ~/src/tui/gsd-2-docs, organized for developers and users.

**Architecture:** Documentation is organized into 8 major sections covering overview, architecture, packages, commands, auto-mode orchestration, extension development, building coding agents, and release history. Each section agent works independently and writes to its own directory. A final integration agent assembles the root README and index.

**Tech Stack:** TypeScript monorepo (npm workspaces), Node.js ≥20.6.0, pi framework (TUI agent runtime), Electron (studio), 5 packages + 1 root GSD extension

---

## File Structure

```
gsd-2-docs/
├── README.md                          # Root index — assembled last
├── docs/
│   ├── 01-overview.md                 # What is GSD-2, versions, features
│   ├── 02-installation.md             # Installation, quick-start
│   ├── 03-architecture.md             # Package structure, data flow
│   ├── 04-commands.md                 # All slash commands, CLI commands
│   ├── 05-configuration.md            # All config options with types
│   ├── 06-auto-mode.md                # Auto orchestration deep-dive
│   ├── 07-parallel-orchestration.md   # Parallel milestone workers
│   ├── 08-packages/
│   │   ├── pi-tui.md                  # Terminal UI library
│   │   ├── pi-ai.md                   # AI provider abstraction
│   │   ├── pi-agent-core.md           # Agent loop
│   │   ├── pi-coding-agent.md         # Coding agent core
│   │   └── studio.md                  # Electron desktop app
│   ├── 09-gsd-extension.md            # GSD extension source walkthrough
│   ├── 10-building-coding-agents.md   # Summary of the 26-part guide
│   ├── 11-extending-pi.md             # Extension development guide
│   ├── 12-context-and-hooks.md        # Context pipeline, hooks reference
│   ├── 13-ci-cd.md                    # CI/CD pipeline documentation
│   ├── 14-troubleshooting.md          # Troubleshooting guide
│   └── 15-changelog.md               # Released versions v2.24–v2.28
```

---

## Task 1: Overview & Installation Documentation
**Agent:** overview-agent
**Source files to read:**
- `~/src/tui/gsd-2/README.md`
- `~/src/tui/gsd-2/CHANGELOG.md` (v2.24–v2.28 only)
- `~/src/tui/gsd-2/docs/getting-started.md`
- `~/src/tui/gsd-2/docs/what-is-pi/01-what-pi-is.md` through `19-building-branded-apps-on-top-of-pi.md`
- `~/src/tui/gsd-2/docs/migration.md`
- `~/src/tui/gsd-2/package.json` (version, description, engines)

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/01-overview.md`
- `~/src/tui/gsd-2-docs/docs/02-installation.md`

---

## Task 2: Architecture Documentation
**Agent:** architecture-agent
**Source files to read:**
- `~/src/tui/gsd-2/docs/architecture.md`
- `~/src/tui/gsd-2/docs/ADR-001-branchless-worktree-architecture.md`
- `~/src/tui/gsd-2/docs/PRD-branchless-worktree-architecture.md`
- `~/src/tui/gsd-2/docs/git-strategy.md`
- `~/src/tui/gsd-2/package.json` (workspaces)
- `~/src/tui/gsd-2/src/loader.ts`
- `~/src/tui/gsd-2/tsconfig.json`

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/03-architecture.md`

---

## Task 3: Commands & Configuration Reference
**Agent:** commands-agent
**Source files to read:**
- `~/src/tui/gsd-2/docs/commands.md`
- `~/src/tui/gsd-2/docs/configuration.md`
- `~/src/tui/gsd-2/docs/remote-questions.md`
- `~/src/tui/gsd-2/docs/skills.md`
- `~/src/tui/gsd-2/docs/cost-management.md`
- `~/src/tui/gsd-2/docs/dynamic-model-routing.md`
- `~/src/tui/gsd-2/docs/token-optimization.md`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/commands.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/commands-config.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/commands-handlers.ts`

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/04-commands.md`
- `~/src/tui/gsd-2-docs/docs/05-configuration.md`

---

## Task 4: Auto-Mode & Parallel Orchestration Documentation
**Agent:** automode-agent
**Source files to read:**
- `~/src/tui/gsd-2/docs/auto-mode.md`
- `~/src/tui/gsd-2/docs/parallel-orchestration.md`
- `~/src/tui/gsd-2/docs/git-strategy.md`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/auto.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/auto-start.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/auto-dispatch.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/auto-prompts.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/auto-recovery.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/parallel-orchestrator.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/guided-flow.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/state.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/milestone-actions.ts`

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/06-auto-mode.md`
- `~/src/tui/gsd-2-docs/docs/07-parallel-orchestration.md`

---

## Task 5: Packages Documentation
**Agent:** packages-agent
**Source files to read:**
- `~/src/tui/gsd-2/packages/pi-tui/src/tui.ts`
- `~/src/tui/gsd-2/packages/pi-tui/src/index.ts`
- `~/src/tui/gsd-2/packages/pi-tui/package.json`
- `~/src/tui/gsd-2/packages/pi-ai/src/index.ts`
- `~/src/tui/gsd-2/packages/pi-ai/src/models.ts`
- `~/src/tui/gsd-2/packages/pi-ai/src/types.ts`
- `~/src/tui/gsd-2/packages/pi-ai/package.json`
- `~/src/tui/gsd-2/packages/pi-agent-core/src/agent.ts`
- `~/src/tui/gsd-2/packages/pi-agent-core/src/agent-loop.ts`
- `~/src/tui/gsd-2/packages/pi-agent-core/src/types.ts`
- `~/src/tui/gsd-2/packages/pi-agent-core/package.json`
- `~/src/tui/gsd-2/packages/pi-coding-agent/src/main.ts`
- `~/src/tui/gsd-2/packages/pi-coding-agent/src/config.ts`
- `~/src/tui/gsd-2/packages/pi-coding-agent/src/index.ts`
- `~/src/tui/gsd-2/packages/pi-coding-agent/package.json`
- `~/src/tui/gsd-2/studio/package.json`
- `~/src/tui/gsd-2/studio/src/main/index.ts`
- `~/src/tui/gsd-2/studio/src/renderer/src/App.tsx`
- `~/src/tui/gsd-2/docs/pi-ui-tui/README.md`

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/08-packages/pi-tui.md`
- `~/src/tui/gsd-2-docs/docs/08-packages/pi-ai.md`
- `~/src/tui/gsd-2-docs/docs/08-packages/pi-agent-core.md`
- `~/src/tui/gsd-2-docs/docs/08-packages/pi-coding-agent.md`
- `~/src/tui/gsd-2-docs/docs/08-packages/studio.md`

---

## Task 6: GSD Extension Source Code Documentation
**Agent:** gsd-extension-agent
**Source files to read:**
- `~/src/tui/gsd-2/src/resources/extensions/gsd/index.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/types.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/state.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/doctor.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/doctor-checks.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/verification-gate.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/preferences.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/model-router.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/worktree.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/worktree-manager.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/visualizer-views.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/visualizer-data.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/export-html.ts`
- `~/src/tui/gsd-2/src/resources/extensions/gsd/crash-recovery.ts`

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/09-gsd-extension.md`

---

## Task 7: Building Coding Agents & Extension Development Guide
**Agent:** guides-agent
**Source files to read:**
- All `~/src/tui/gsd-2/docs/building-coding-agents/*.md`
- `~/src/tui/gsd-2/docs/extending-pi/README.md`
- All `~/src/tui/gsd-2/docs/extending-pi/*.md`
- `~/src/tui/gsd-2/docs/context-and-hooks/README.md`
- All `~/src/tui/gsd-2/docs/context-and-hooks/*.md`

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/10-building-coding-agents.md`
- `~/src/tui/gsd-2-docs/docs/11-extending-pi.md`
- `~/src/tui/gsd-2-docs/docs/12-context-and-hooks.md`

---

## Task 8: CI/CD, Troubleshooting & Changelog
**Agent:** ops-agent
**Source files to read:**
- `~/src/tui/gsd-2/docs/ci-cd-pipeline.md`
- `~/src/tui/gsd-2/docs/troubleshooting.md`
- `~/src/tui/gsd-2/docs/visualizer.md`
- `~/src/tui/gsd-2/docs/captures-triage.md`
- `~/src/tui/gsd-2/docs/node-lts-macos.md`
- `~/src/tui/gsd-2/docs/working-in-teams.md`
- `~/src/tui/gsd-2/.github/workflows/pipeline.yml`
- `~/src/tui/gsd-2/CHANGELOG.md` (released versions v2.24.0–v2.28.0 only)

**Deliverables:**
- `~/src/tui/gsd-2-docs/docs/13-ci-cd.md`
- `~/src/tui/gsd-2-docs/docs/14-troubleshooting.md`
- `~/src/tui/gsd-2-docs/docs/15-changelog.md`
