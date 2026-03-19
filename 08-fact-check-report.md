# GSD-2 Documentation Fact-Check Report

Online verification of claims made in the GSD-2 documentation, conducted 2026-03-19.

## npm Package Status

**Package name:** `gsd-pi`
**Verified:** Yes, the package exists on npm.

| Dist-Tag | Version | Notes |
|----------|---------|-------|
| `@latest` | 2.33.1 | Published 2026-03-19 |
| `@next` | 2.33.1-dev.29a8268 | Published 2026-03-19 |
| `@dev` | 2.33.1-dev.9bafd68 | Published 2026-03-19 |

**Package metadata (confirmed via `npm view`):**
- Description: "GSD -- Get Shit Done coding agent"
- Homepage: https://github.com/gsd-build/gsd-2#readme
- Repository: git+https://github.com/gsd-build/gsd-2.git
- Engine requirement: `node >= 22.0.0`

**Documentation claims vs actual state:**
- The docs state `npm install -g gsd-pi` -- confirmed correct.
- The docs state Node.js >= 22.0.0 required -- confirmed, matches `engines` field in package.json. However, the troubleshooting doc (`troubleshooting.md` line 49) states ">= 20.6.0" which contradicts the actual engine requirement and the getting-started doc. This is a **discrepancy**.
- The CI/CD docs describe `@dev`, `@next`, `@latest` dist-tags -- confirmed, all three exist.
- The CI/CD docs describe dev version format as `2.27.0-dev.a3f2c1b` -- confirmed, actual format matches (e.g., `2.33.1-dev.29a8268`).
- The project is actively maintained with multiple publishes per day as of 2026-03-19.

**Download statistics:** Could not retrieve specific download counts from web search. The npm stats tools (npm-stat.com, npmtrends.com) exist but search results did not return GSD-specific data.

## GitHub Repository Verification

**Repository:** `gsd-build/gsd-2`
**Verified:** Yes, the repository exists at https://github.com/gsd-build/gsd-2

**Confirmed details:**
- Organization: `gsd-build` (https://github.com/gsd-build)
- Description: "A powerful meta-prompting, context engineering and spec-driven development system that enables agents to work for long periods of time autonomously without losing track of the big picture"
- The repo has an active issues tracker, releases page, and GitHub Actions workflows
- The docs directory is published at https://github.com/gsd-build/gsd-2/tree/main/docs
- The v1 predecessor exists at https://github.com/gsd-build/get-shit-done

**Documentation claims vs actual state:**
- Issues URL `github.com/gsd-build/GSD-2/issues` in troubleshooting.md -- confirmed valid (GitHub is case-insensitive for repo names)
- GHCR images reference `ghcr.io/gsd-build/gsd-pi` -- not directly verified but consistent with GitHub org ownership

## VS Code Extension Verification

**Extension:** `FluxLabs.gsd-2`
**Verified:** Yes, the extension exists on the VS Code Marketplace.

- URL confirmed: https://marketplace.visualstudio.com/items?itemName=FluxLabs.gsd-2
- Publisher: FluxLabs (matches docs claim)
- Features confirmed: @gsd chat participant, sidebar dashboard, 15 commands

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
**Verified:** Yes, the Discord server exists.

- Server name: "GSD: Get Shit Done"
- Approximate member count: ~3,200 members (as reported by search results)
- The server is for sharing projects, getting help, and discussions about GSD

## Community References and Coverage

**Found references:**

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

## Discrepancies Found

### 1. Node.js Version Requirement Inconsistency

- `getting-started.md` states: "Requires Node.js >= 22.0.0 (24 LTS recommended)"
- `troubleshooting.md` (line 49) states: "Node.js version too old -- requires >= 20.6.0"
- Actual `engines` field in npm package: `node >= 22.0.0`
- **Verdict:** The troubleshooting doc is outdated. The correct minimum is 22.0.0.

### 2. Model Names May Be Speculative

The docs reference models like `claude-opus-4-6`, `claude-sonnet-4-6`, `claude-haiku-4-5`, `gpt-4.5-preview`, and `gemini-2.5-pro`. These model identifiers are used internally by GSD for its routing tables. Some (like `gpt-4.5-preview`) may not correspond to currently available models from the provider. The cost table in `dynamic-model-routing.md` should be treated as approximate and subject to change.

### 3. CI/CD Pipeline Version References

The CI/CD docs use `2.27.0-dev.a3f2c1b` as an example version. The actual latest version is 2.33.1, confirming the docs were written when the project was at an earlier version. The format and pipeline mechanics remain accurate.

### 4. Docker Image References

The docs reference `ghcr.io/gsd-build/gsd-pi` Docker images. These were not directly verified via search but are consistent with the GitHub org and CI/CD pipeline description.

## Claims That Could Not Be Verified

1. **Enterprise adoption claim:** The GitHub README reportedly states GSD is "used by engineers at Amazon, Google, Shopify and Webflow." This claim appeared in search result summaries but could not be independently verified.

2. **Token savings percentages:** Claims of "40-60% reduction" with the budget profile and "20-50% reduction" with dynamic routing are presented as typical ranges. These are plausible given the described mechanisms but are self-reported metrics.

3. **"~70 process spawns per dispatch cycle" eliminated by libgit2:** Specific performance claim from git-strategy.md. Plausible but not independently verified.

4. **Native binaries cross-compilation:** The CI/CD docs reference `build-native.yml` for cross-compiling platform binaries on `v*` tags. Not directly verified.

## Summary

| Item | Status |
|------|--------|
| npm package (`gsd-pi`) | Verified, actively published |
| GitHub repo (`gsd-build/gsd-2`) | Verified, actively maintained |
| VS Code extension (`FluxLabs.gsd-2`) | Verified on Marketplace |
| Pi SDK (`badlogic/pi-mono`) | Verified |
| Discord (`discord.gg/gsd`) | Verified, ~3,200 members |
| Community coverage | Multiple blog posts and comparisons found |
| Node.js version discrepancy | Found (troubleshooting.md says >= 20.6.0, actual is >= 22.0.0) |
| Documentation accuracy overall | High -- minor version reference staleness, one version requirement inconsistency |

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
