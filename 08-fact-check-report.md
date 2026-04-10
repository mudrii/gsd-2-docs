# GSD-2 Documentation Fact-Check Report

Online verification of claims made in the GSD-2 documentation, conducted 2026-04-10. Previous report: 2026-04-02.

## npm Package Status

**Package name:** `gsd-pi`
**Verified:** Yes, the package exists on npm.

| Dist-Tag | Version | Notes |
|----------|---------|-------|
| `@latest` | 2.67.0 | Confirmed via `npm view` on 2026-04-10 |
| `@next` | 2.67.0-dev.2142d3e | Confirmed via `npm view` on 2026-04-10 |
| `@dev` | 2.67.0-dev.5399650 | Confirmed via `npm view` on 2026-04-10 |

**Total stable versions published:** 123

**Package metadata (confirmed via `npm view` and local `package.json`):**
- Description: "GSD -- Get Shit Done coding agent"
- Homepage: https://github.com/gsd-build/gsd-2#readme
- Repository: git+https://github.com/gsd-build/gsd-2.git
- Engine requirement: `node >= 22.0.0`
- License: MIT
- Binary names: `gsd`, `gsd-cli`
- Publisher: `glittercowboy`

**Documentation claims vs actual state:**
- The docs state `npm install -g gsd-pi` -- confirmed correct.
- The docs state Node.js >= 22.0.0 required -- confirmed, matches `engines` field in package.json.
- The troubleshooting doc (`troubleshooting.md`) now correctly states ">= 22.0.0". The previous >= 20.6.0 discrepancy has been **fixed** since the 2026-03-19 report.
- **9 runtimes** are now supported: Claude Code, OpenCode, Gemini CLI, Codex, GitHub Copilot, Antigravity, Cursor, Windsurf, Augment.
- The getting-started doc states "Requires Node.js >= 22.0.0 (24 LTS recommended)" -- confirmed correct.
- The CI/CD docs describe `@dev`, `@next`, `@latest` dist-tags -- confirmed, all three exist.
- The CI/CD docs describe dev version format as `2.27.0-dev.a3f2c1b` -- confirmed, actual format matches (e.g., `2.58.0-dev.778d6ac`).
- The project is actively maintained with frequent publishes as of 2026-04-10 (jumped from 2.58.0 to 2.67.0 in under two weeks).
- **Note:** DeepWiki's auto-generated documentation for gsd-2 incorrectly states "Node.js 20.6.0 or later" -- this is stale third-party documentation, not a GSD docs issue.

**Download statistics:** Could not retrieve specific download counts from web search. The npm stats tools (npm-stat.com, npmtrends.com) exist but search results did not return GSD-specific data.

## GitHub Repository Verification

**Repository:** `gsd-build/gsd-2`
**Verified:** Yes, the repository exists at https://github.com/gsd-build/gsd-2

**Confirmed details:**
- Organization: `gsd-build` (https://github.com/gsd-build)
- Description: "A powerful meta-prompting, context engineering and spec-driven development system that enables agents to work for long periods of time autonomously without losing track of the big picture"
- Stars: ~5,200 (as of 2026-04-10, up from ~4,037 on 2026-04-02)
- Forks: ~536 (up from ~417 on 2026-04-02)
- Open issues: 230
- Contributors: 76
- Commits: 3,022
- License: MIT
- Language: TypeScript
- Last updated: April 10, 2026
- The repo has an active issues tracker, releases page, and GitHub Actions workflows
- The docs directory is published at https://github.com/gsd-build/gsd-2/tree/main/docs
- The v1 predecessor exists at https://github.com/gsd-build/get-shit-done (~46,500 stars)

**Documentation claims vs actual state:**
- Issues URL `github.com/gsd-build/GSD-2/issues` in troubleshooting.md -- confirmed valid (GitHub is case-insensitive for repo names)
- GHCR images reference `ghcr.io/gsd-build/gsd-pi` -- not directly verified but consistent with GitHub org ownership and the `docker:build-runtime` npm script in package.json
- The `packages/daemon` workspace referenced in docs exists in the source tree at `packages/daemon`

## VS Code Extension Verification

**Extension:** `FluxLabs.gsd-2`
**Verified:** Yes, the extension exists on the VS Code Marketplace.

- URL confirmed: https://marketplace.visualstudio.com/items?itemName=FluxLabs.gsd-2
- Publisher: FluxLabs (matches docs claim)
- Version: v0.3.0
- Installs: 1,578 (up from previous report)
- Features confirmed: @gsd chat participant, sidebar redesign with SCM provider, checkpoints, diagnostics, 33 commands
- The extension requires the CLI (`gsd-pi`) to be installed first and connects via RPC

## Pi SDK Verification

**Repository:** `badlogic/pi-mono`
**Verified:** Yes, the underlying SDK exists at https://github.com/badlogic/pi-mono

**Confirmed details:**
- Author: Mario Zechner (badlogic)
- Description: "AI agent toolkit: coding agent CLI, unified LLM API, TUI & web UI libraries, Slack bot, vLLM pods"
- npm package: `@mariozechner/pi-coding-agent`
- The SDK provides four modes: interactive, print/JSON, RPC, and SDK embedding
- Default tools: read, write, edit, bash
- Extensibility via skills, prompt templates, extensions, and packages
- Sessions auto-save to `~/.pi/agent/sessions/`

**Documentation claims vs actual state:**
- The docs state GSD is "built on the Pi SDK" -- confirmed. The pi-mono repo contains a `packages/coding-agent` directory, and GSD-2 is described as a standalone CLI built on this SDK.
- The docs reference Pi credential import ("If you have an existing Pi installation, provider credentials are imported automatically") -- consistent with the shared SDK architecture.
- The blog post "What I learned building an opinionated and minimal coding agent" by Mario Zechner (mariozechner.at) provides additional context on Pi's design philosophy.

## Discord Community Verification

**Invite link:** `discord.gg/gsd`
**Verified:** Yes, the Discord server exists and is confirmed active.

- Server name: "GSD: Get Shit Done"
- The server is for sharing projects, getting help, and discussions about GSD
- GSD Wizard bot provides RAG-based Q&A for documentation and troubleshooting
- A secondary link (discord.com/invite/5JJgD5svVS) also appears in search results, suggesting the vanity URL `discord.gg/gsd` redirects to the same server

## Community References and Coverage

**Found references (carried forward from previous report, with additions):**

1. **Medium article by Agent Native (Feb 2026):** "GET SH*T DONE: Meta-prompting and Spec-driven Development for Claude Code and Codex" -- covers GSD's approach to spec-driven development.

2. **Medium article by Vishal Mysore (Mar 2026):** "Spec-Driven Development Is Eating Software Engineering: A Map of 30+ Agentic Coding Frameworks (2026)" -- includes GSD in a landscape analysis of agentic coding frameworks.

3. **Medium article by Rick Hightower (Feb 2026):** "Agentic Coding: GSD vs Spec Kit vs OpenSpec vs Taskmaster AI: Where SDD Tools Diverge" -- direct comparison of GSD with competitor frameworks.

4. **Codecentric blog post:** "GSD for Claude Code: A Deep Dive into the Workflow System" -- technical deep dive into GSD's architecture.

5. **NeonNook Substack:** "The Rise of 'Get-Shit-Done' AI Product Frameworks" -- covers the broader trend that GSD is part of.

6. **Pasquale Pillitteri blog:** "Goodbye Vibe Coding: Spec-Driven Development Framework" -- compares GSD with BMAD and Ralph Loop frameworks.

7. **DeepWiki:** Documentation mirrors exist for both gsd-build/get-shit-done and gsd-build/gsd-2.

8. **GitClassic:** Mirror/index of gsd-build/gsd-2.

9. **GSD-OpenCode fork:** https://github.com/rokicool/gsd-opencode -- a port of GSD for OpenCode.

10. **Twitter/X:** @gsd_foundation account exists and posts about releases.

11. **The New Stack (new):** "Beating context rot in Claude Code with GSD" -- examines GSD's approach to defeating context rot, noting it should be considered a temporary workaround reflecting current LLM limitations.

12. **DEV Community (new):** "The Complete Beginner's Guide to GSD (Get Shit Done) Framework for Claude Code" by Ali Kazmi -- beginner-oriented tutorial.

13. **gsd.build website (new):** Official website at https://gsd.build/ exists.

## Best of JS Verification

**Verified:** Yes, GSD-2 is tracked on Best of JS.

- Stars: 5.15k
- Growth: +141 stars/day
- Contributors: 76

These numbers are consistent with the GitHub repository stats above.

## Discord Community Update (2026-04-10)

**Invite link:** `discord.gg/gsd`
**Verified:** Yes, the Discord server remains active.

- The vanity URL `discord.gg/gsd` is confirmed working
- GSD Wizard bot is present in the server, providing RAG-based Q&A for GSD documentation and troubleshooting
- Member count variance noted in previous reports remains; the server is actively moderated with regular discussions

## GSD-1 Predecessor

The v1 predecessor repository at https://github.com/gsd-build/get-shit-done has accumulated ~46,500 stars. GSD-2 is the complete rewrite that replaced it.

## Release Verification (v2.59.0 - v2.67.0)

Releases v2.59.0 through v2.67.0 were verified on the GitHub releases page (https://github.com/gsd-build/GSD-2/releases). All versions exist with corresponding release notes and tagged commits. The release cadence has remained rapid, with multiple releases per week.

## New Claims Verified (2026-04-10)

### Web Interface (`gsd --web`)

**Verified:** Yes, confirmed in source docs (`docs/web-interface.md`) and package.json scripts.

- Introduced in v2.41.0
- Built with Next.js, communicates via bridge service
- CLI flags added in v2.42.0: `--host`, `--port`, `--allowed-origins`
- Features: project dashboard, real-time SSE progress, multi-project support, onboarding flow, model selection
- Package.json contains `gsd:web` and `gsd:web:stop` scripts confirming implementation
- Documentation claim of `--web [path]` flag in `commands.md` is consistent with source

### Headless Mode (`gsd headless`)

**Verified:** Yes, confirmed in source docs (`docs/commands.md`) and referenced across multiple documentation files.

- Runs `/gsd` commands without a TUI -- designed for CI, cron jobs, and scripted automation
- Spawns a child process in RPC mode, auto-responds to interactive prompts
- Subcommands: `gsd headless auto`, `gsd headless next`, `gsd headless query`, `gsd headless new-milestone`
- `gsd headless query` returns instant JSON snapshot (~50ms, no LLM)
- Flags: `--timeout`, `--max-restarts`, `--json`, `--model`, `--context`, `--context-text`, `--auto`, `--answers`
- Exit codes: 0 = complete, 1 = error/timeout, 2 = blocked
- Auto-restart on crash with exponential backoff (default 3 attempts)
- Remote questions route to Slack/Discord/Telegram in headless mode

### SQLite State Engine

**Verified:** Yes, confirmed in source code and docs.

- The `sql.js` (WASM SQLite) dependency is present in `package.json` at version `^1.14.1`
- Multiple source files in `src/resources/extensions/gsd/` reference SQLite operations
- `/gsd inspect` command exposes SQLite DB diagnostics
- Docs describe "Era 19: SQLite State Engine (v2.44.0 - v2.46.1)" with tool-driven write-side state transitions replacing markdown
- Single-writer engine with state machine guards, actor identity, and reversibility (v2.46.0)
- Schema migrations v8-v11 documented
- Memory storage switched from `better-sqlite3` to `sql.js` (WASM) for Node 25+ compatibility
- Test files confirm extensive SQLite usage: state machine walkthrough tests, vacuum recovery tests, worktree DB tests, state corruption regression tests

### Daemon / Discord Integration

**Verified:** Yes, the `packages/daemon` workspace exists in the source tree.

- Developer guide documents this as a v2.57.0+ feature
- Long-running background service managing headless GSD sessions bridged to Discord channels
- macOS service registration via `launchd.ts` (generates `com.gsd.daemon.plist`)
- Could not independently verify the daemon is published as a standalone package or that the Discord bridge is functional -- only confirmed the source code exists

### Capability-Based Model Selection

**Verified:** Yes, confirmed via GitHub issue references and documentation.

- Replaced model-ID pattern matching with capability metadata (#2740)
- Makes custom provider integration more reliable
- 20+ LLM providers supported including Anthropic, OpenAI, Google, OpenRouter, GitHub Copilot, Amazon Bedrock, Azure

## Previously Found Discrepancies -- Status Update

### 1. Node.js Version Requirement Inconsistency -- RESOLVED

- **Previous finding:** `troubleshooting.md` stated ">= 20.6.0", contradicting the actual engine requirement of >= 22.0.0.
- **Current status:** `troubleshooting.md` now correctly states ">= 22.0.0". This discrepancy has been fixed.

### 2. Model Names May Be Speculative -- STILL APPLIES

The docs reference models like `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`, `gpt-4.5-preview`, and `gemini-2.5-pro`. These model identifiers are used internally by GSD for its routing tables. Some may not correspond to currently available models from the provider. The cost table in `dynamic-model-routing.md` should be treated as approximate and subject to change.

### 3. CI/CD Pipeline Version References -- STALE BUT HARMLESS

The CI/CD docs use `2.27.0-dev.a3f2c1b` as an example version. The actual latest version is now 2.67.0. The format and pipeline mechanics remain accurate; only the example version number is stale.

### 4. Docker Image References -- STILL UNVERIFIED

The docs reference `ghcr.io/gsd-build/gsd-pi` Docker images. These were not directly verified via search but are consistent with the GitHub org ownership and the `docker:build-runtime` script in package.json.

## New Discrepancies Found (2026-04-10)

### 5. Third-Party Documentation Staleness

DeepWiki's auto-generated mirror of gsd-2 docs states the Node.js requirement is "20.6.0 or later" which is outdated. This is not a GSD documentation issue but may confuse users who find the DeepWiki mirror via search.

### 6. Discord Member Count Variance

The previous report stated ~3,200 members; current search results show ~2,276. This may reflect different data sources, point-in-time variance, or counting methodology differences (online vs total). The server itself is confirmed active.

### 7. VS Code Extension Command Count

The previous report stated 15 commands; the current search result says 33 commands. This likely reflects genuine feature growth between v2.33 and v2.58 rather than a documentation error.

## Claims That Could Not Be Verified

1. **Enterprise adoption claim:** The GitHub README reportedly states GSD is "used by engineers at Amazon, Google, Shopify and Webflow." This claim appeared in search result summaries but could not be independently verified.

2. **Token savings percentages:** Claims of "40-60% reduction" with the budget profile and "20-50% reduction" with dynamic routing are presented as typical ranges. These are plausible given the described mechanisms but are self-reported metrics.

3. **"~70 process spawns per dispatch cycle" eliminated by libgit2:** Specific performance claim from git-strategy.md. Plausible but not independently verified.

4. **Native binaries cross-compilation:** The CI/CD docs reference `build-native.yml` for cross-compiling platform binaries on `v*` tags. Not directly verified, but native platform packages (`@gsd-build/engine-darwin-arm64`, etc.) are listed as optional dependencies in package.json, confirming the cross-compilation pipeline produces output.

5. **Daemon Discord bridge functionality:** The `packages/daemon` workspace exists and is documented, but the actual Discord integration could not be verified as working. Only source code presence was confirmed.

## Summary

| Item | Status | Change from 2026-04-02 |
|------|--------|------------------------|
| npm package (`gsd-pi`) | Verified, actively published (v2.67.0), 123 stable versions | Version jumped from 2.58.0 to 2.67.0 |
| GitHub repo (`gsd-build/gsd-2`) | Verified, ~5,200 stars, ~536 forks, 76 contributors, 3,022 commits | Stars up from ~4,037, forks up from ~417 |
| VS Code extension (`FluxLabs.gsd-2`) | Verified on Marketplace, v0.3.0, 1,578 installs | Install count and version now tracked |
| Pi SDK (`badlogic/pi-mono`) | Verified | No change |
| Discord (`discord.gg/gsd`) | Verified, active, GSD Wizard bot for RAG Q&A | Wizard bot confirmed |
| Best of JS | 5.15k stars, +141 stars/day growth | New check |
| GSD-1 predecessor (`get-shit-done`) | ~46,500 stars | Confirmed |
| Community coverage | Multiple blog posts and comparisons found | No change |
| Web interface (`gsd --web`) | Verified in source and docs (v2.41.0+) | No change |
| Headless mode (`gsd headless`) | Verified in source and docs | No change |
| SQLite state engine | Verified in source, deps, and docs (v2.44.0+) | No change |
| Daemon / Discord bridge | Source code exists (`packages/daemon`) | No change |
| Multi-runtime support | 9 runtimes confirmed | New check |
| Releases v2.59.0-v2.67.0 | All verified on GitHub releases page | New check |
| Node.js version discrepancy | **Resolved** -- troubleshooting.md now correct | No change |
| DeepWiki staleness | Still shows "Node.js 20.6.0" -- third-party issue | Ongoing |
| Documentation accuracy overall | High -- minor version reference staleness, model name caveats remain | Improved |

Sources:
- [gsd-build/gsd-2 on GitHub](https://github.com/gsd-build/gsd-2)
- [gsd-build org on GitHub](https://github.com/gsd-build)
- [GSD-2 VS Code Extension](https://marketplace.visualstudio.com/items?itemName=FluxLabs.gsd-2)
- [badlogic/pi-mono on GitHub](https://github.com/badlogic/pi-mono)
- [Pi coding agent on npm](https://www.npmjs.com/package/@mariozechner/pi-coding-agent)
- [GSD Discord](https://discord.com/invite/gsd)
- [GSD-2 Releases](https://github.com/gsd-build/GSD-2/releases)
- [gsd-build/get-shit-done (v1)](https://github.com/gsd-build/get-shit-done)
- [What I learned building a coding agent - Mario Zechner](https://mariozechner.at/posts/2025-11-30-pi-coding-agent/)
- [Spec-Driven Development landscape (Medium)](https://medium.com/@visrow/spec-driven-development-is-eating-software-engineering-a-map-of-30-agentic-coding-frameworks-6ac0b5e2b484)
- [GSD vs Spec Kit vs OpenSpec (Medium)](https://medium.com/@richardhightower/agentic-coding-gsd-vs-spec-kit-vs-openspec-vs-taskmaster-ai-where-sdd-tools-diverge-0414dcb97e46)
- [GSD for Claude Code deep dive (Codecentric)](https://www.codecentric.de/en/knowledge-hub/blog/the-anatomy-of-claude-code-workflows-turning-slash-commands-into-an-ai-development-system)
- [@gsd_foundation on X](https://x.com/gsd_foundation/status/2032177283062993280)
- [Beating context rot in Claude Code with GSD (The New Stack)](https://thenewstack.io/beating-the-rot-and-getting-stuff-done/)
- [Beginner's Guide to GSD (DEV Community)](https://dev.to/alikazmidev/the-complete-beginners-guide-to-gsd-get-shit-done-framework-for-claude-code-24h0)
- [GSD official website](https://gsd.build/)
- [DeepWiki gsd-2 docs](https://deepwiki.com/gsd-build/gsd-2/6.1-package-structure-and-build-system)
- [GSD-OpenCode fork](https://github.com/rokicool/gsd-opencode)
