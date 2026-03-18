# Troubleshooting

This guide covers installation failures, common runtime errors, API key problems, Node.js version issues, performance degradation, crash recovery, and how to use GSD's built-in diagnostic tools effectively.

## The Doctor Command

The first tool to reach for when something seems wrong is `/gsd doctor`.

```
/gsd doctor
```

Doctor validates the entire `.gsd/` directory and repairs what it can automatically:

- File structure and naming conventions
- Roadmap-to-slice-to-task referential integrity
- Completion state consistency across plan files
- Git worktree health (worktree and branch modes only — skipped in `none` mode)
- Stale lock files and orphaned runtime records
- Missing `STATE.md` (rebuilt from plan and roadmap files on disk)
- Roadmap checkboxes (fixed with `fixLevel: 'all'`)

Run doctor after any crash, after a manual edit to `.gsd/` files, or when auto-mode behaves unexpectedly. It is safe to run at any time.

## Installation Issues

### `npm install -g gsd-pi` fails

**Missing workspace packages:** Fixed in v2.10.4+. Update to a current version.

**`postinstall` hangs on Linux:** Playwright's `--with-deps` flag triggers a sudo prompt, which hangs in non-interactive shells. Fixed in v2.3.6+. If you are on an older version, set `PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1` before installing, or update.

**Node.js version too old:** GSD requires Node.js 20.6.0 or later and works best on an LTS (even-numbered) release. Check your version:

```bash
node --version
```

If this shows a version below 20.6.0, or an odd-numbered development release (e.g., 23.x or 25.x), upgrade to the current LTS before installing GSD. See the Node.js version section below for detailed instructions.

**Read-only file permissions from Nix store:** Fixed in v2.24.0. If you are installing from a Nix-managed Node.js, the cpSync from the Nix store may leave files read-only. Update to v2.24.0 or later.

### Verifying a clean install

```bash
node --version    # v22.x.x or v24.x.x
gsd --version
```

If `gsd` is not found after a global install, check that npm's global bin directory is in your `PATH`:

```bash
npm config get prefix    # shows the prefix, e.g. /usr/local
# Add /usr/local/bin to PATH if missing
```

## Node.js Version Issues (macOS)

### Homebrew installs the current (non-LTS) release

If you installed Node.js via `brew install node`, Homebrew tracks the latest current release, which can be an odd-numbered development version (23.x, 25.x). These are not LTS releases and may have breaking changes.

**Install Node 24 LTS:**

```bash
brew unlink node
brew install node@24
brew link --overwrite node@24
```

**Prevent accidental upgrades:**

```bash
brew pin node@24
```

**Verify:**

```bash
node --version    # v24.x.x
npm install -g gsd-pi
gsd --version
```

### Managing multiple Node versions

If you need to switch between Node versions across projects, use a version manager rather than Homebrew's link mechanism:

- **nvm:** `nvm install 24 && nvm use 24`
- **fnm** (faster, Rust-based): `fnm install 24 && fnm use 24`
- **mise** (polyglot): `mise use node@24`

These tools support per-project version pinning via `.node-version` or `.nvmrc` files.

## API Key Configuration Problems

### Auto mode pauses with an auth error

Auth and billing errors (`"unauthorized"`, `"invalid key"`) do not auto-resume. GSD stops and waits for manual intervention.

**Check your API key is set:**

```bash
echo $ANTHROPIC_API_KEY    # or whichever provider you use
```

**Common causes:**

- Key was rotated but the environment variable was not updated
- Key belongs to a different organization or project
- Billing limit reached on the provider account
- Key lacks the required permissions for the model being used

**After fixing the key:** Resume with `/gsd auto`. You do not need to restart from scratch.

### Fallback model configuration

To avoid hard stops on provider errors, configure fallbacks in `.gsd/preferences.md`:

```yaml
models:
  execution:
    model: claude-sonnet-4-6
    fallbacks:
      - openrouter/minimax/minimax-m2.5
```

GSD will try the primary model first, then fall through to fallbacks in order before treating the error as unrecoverable.

## Common Runtime Errors

### Auto mode loops on the same unit

**Symptoms:** The same unit (e.g., `research-slice` or `plan-slice`) dispatches repeatedly until hitting the dispatch limit.

**Causes:**
- Stale in-memory file listing after a crash — the cache does not reflect artifacts that exist on disk
- The LLM did not produce the expected artifact file

**Fix:** Run `/gsd doctor` to repair state and rebuild the cache, then resume with `/gsd auto`. If the loop continues, verify that the expected artifact file exists on disk in the correct location.

### Auto mode stops with "Loop detected"

**Cause:** A unit failed to produce its expected artifact twice consecutively.

**Fix:** Inspect the task plan for ambiguity. Refine the plan manually if needed, then run `/gsd auto` to resume.

### Budget ceiling reached

**Symptoms:** Auto mode pauses with "Budget ceiling reached."

**Fix:** Increase `budget_ceiling` in `.gsd/preferences.md`, or switch to the `budget` token profile to reduce per-unit cost. Then resume with `/gsd auto`.

### Provider errors during auto mode

Starting in v2.26, GSD handles transient provider errors automatically:

| Error type | Auto-resume | Delay |
|---|---|---|
| Rate limit (429, "too many requests") | Yes | retry-after header, or 60 seconds |
| Server error (500, 502, 503, "overloaded") | Yes | 30 seconds |
| Auth/billing ("unauthorized", "invalid key") | No | Manual resume required |

For rate limits, the maximum delay is 300 seconds (raised from 60s in v2.24.0).

### Wrong files appear in worktree

**Symptoms:** Planning artifacts or code appear in the wrong directory.

**Cause:** The LLM wrote to the main repo instead of the worktree.

**Fix:** This was corrected in v2.14+. The dispatch prompt now includes explicit working directory instructions. If you are on an older version, update to v2.14 or later.

### Stale lock file

**Symptoms:** Auto mode will not start; reports "another session is running."

**Fix:** GSD checks whether the PID recorded in the lock file is actually running. If no other session is alive, delete the lock manually:

```bash
rm .gsd/auto.lock
```

### Git merge conflicts in `.gsd/` files

**Symptoms:** Worktree merge fails with conflicts in `.gsd/` files.

**Fix:** GSD auto-resolves conflicts on `.gsd/` runtime files. For content conflicts in code files, GSD gives the LLM an opportunity to resolve them via a fix-merge session. If that fails, resolve manually and re-run `/gsd auto`.

## LSP Issues

### "LSP isn't available in this workspace"

GSD auto-detects language servers based on project files (`package.json` → TypeScript, `Cargo.toml` → Rust, `go.mod` → Go). If no servers are detected, LSP features are skipped silently.

**Check LSP status:**

```
lsp status
```

This shows which servers are active and diagnoses why none were found, including which project markers were detected but which server commands are missing from `PATH`.

**Install missing language servers:**

| Project type | Install command |
|---|---|
| TypeScript/JavaScript | `npm install -g typescript-language-server typescript` |
| Python | `pip install pyright` or `pip install python-lsp-server` |
| Rust | `rustup component add rust-analyzer` |
| Go | `go install golang.org/x/tools/gopls@latest` |

After installing, run `lsp reload` to restart detection without restarting GSD.

## Performance Issues

### High CPU usage during long auto-mode sessions

Reduced CPU usage on long sessions was fixed in v2.27.0. Update to the latest version if you observe sustained high CPU consumption.

### Slow startup time

Provider SDKs are lazy-loaded as of v2.24.0, which reduces startup time significantly. If startup is still slow, check for large `.gsd/activity/` log files — these accumulate over time and can be archived or deleted.

### Context pressure and wrap-up

GSD monitors context window usage and sends a wrap-up signal at 70% capacity (wired in v2.27.0). If you see auto-mode completing units earlier than expected, this is the context-pressure monitor acting preventively rather than a bug.

## Crash Recovery

### Reset auto mode state

If auto mode will not start cleanly after a crash:

```bash
rm .gsd/auto.lock
rm .gsd/completed-units.json
```

Then run `/gsd auto` to restart from current disk state. GSD will re-derive state from plan files.

### Reset routing history

If adaptive model routing is producing poor model selections:

```bash
rm .gsd/routing-history.json
```

### Full state rebuild

```
/gsd doctor
```

Doctor rebuilds `STATE.md` from plan and roadmap files and repairs all detected inconsistencies in a single pass.

### Headless auto-restart

In headless mode, GSD auto-restarts on crash with exponential backoff (default: 3 attempts). This is enabled by default for `gsd headless auto` and requires no configuration.

## Captures and Triage

Captures let you steer auto-mode without pausing it. Use them to record ideas, bug observations, or scope changes while a session runs:

```
/gsd capture "add rate limiting to the API endpoints"
/gsd capture "the auth flow should support OAuth, not just JWT"
```

Captures are appended to `.gsd/CAPTURES.md` with a timestamp and unique ID. GSD processes them automatically at natural seams between tasks (between `handleAgentEnd` calls).

**Classification types:**

| Type | Meaning | Resolution |
|---|---|---|
| `quick-task` | Small, self-contained fix | Executed immediately as an inline task |
| `inject` | New task needed in current slice | Injected into the active slice plan |
| `defer` | Important but not urgent | Deferred to roadmap reassessment |
| `replan` | Changes the current approach | Triggers a slice replan with capture context |
| `note` | Informational only | Acknowledged, no plan changes |

To trigger triage manually rather than waiting for the next natural seam:

```
/gsd triage
```

The dashboard progress widget shows a pending capture count badge when captures are waiting.

**Worktree awareness:** Captures always resolve to the original project root's `.gsd/CAPTURES.md`, not the worktree's local copy. This ensures a capture made from a steering terminal is visible to the auto-mode session running inside a worktree.

## Visualizer for Debugging

The workflow visualizer (`/gsd visualize`) provides a real-time view of project state useful for diagnosing auto-mode behavior:

```
/gsd visualize
```

The visualizer has four tabs:

**Progress tab:** A tree view of milestones, slices, and tasks with completion status, discussion indicators, and task counts at each level. Use this to identify which units have completed and which are blocked.

**Dependencies tab:** An ASCII dependency graph of slice relationships derived from `depends:` fields in the roadmap. Use this to spot blocked slices that are waiting on incomplete prerequisites.

**Metrics tab:** Bar charts of cost and token usage broken down by phase, by slice, and by model. Use this to identify which phases or models are consuming disproportionate budget.

**Timeline tab:** Chronological execution history showing unit type, ID, start/end timestamps, duration, model used, and token counts. Use this to trace the sequence of dispatches that led to an unexpected state.

The visualizer refreshes from disk every 2 seconds, so it stays current alongside a running auto-mode session.

**Keyboard controls:**

| Key | Action |
|---|---|
| `Tab` / `Shift+Tab` | Next / previous tab |
| `1`–`4` | Jump to tab |
| `↑` / `↓` | Scroll within tab |
| `Escape` or `q` | Close |

**HTML export:** For shareable post-mortems, use `/gsd export --html`. This generates a self-contained HTML file in `.gsd/reports/` with the same data as the TUI — progress tree, dependency graph (SVG DAG), cost charts, execution timeline, changelog, and knowledge base. All CSS and JS are inlined, so no external dependencies are needed. The file can be printed to PDF from any browser.

## Forensics

For structured post-mortem analysis of auto-mode failures:

```
/gsd forensics
```

This produces a structured analysis of what went wrong, drawing from session logs in `.gsd/activity/` (JSONL format).

## Getting Help

- **GitHub Issues:** [github.com/gsd-build/GSD-2/issues](https://github.com/gsd-build/GSD-2/issues) — file a bug or search for known issues
- **Dashboard:** Press `Ctrl+Alt+G` or run `/gsd status` for real-time diagnostics
- **Forensics:** `/gsd forensics` for structured post-mortem analysis after an auto-mode failure
- **Session logs:** `.gsd/activity/` contains JSONL session dumps for crash forensics
- **Doctor:** `/gsd doctor` for automated repair of `.gsd/` integrity issues
