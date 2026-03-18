# CI/CD Pipeline

GSD uses a three-stage promotion pipeline that automatically moves every merged PR through Dev, Test, and Prod environments using npm dist-tags and Docker image tags. The pipeline is defined in `.github/workflows/pipeline.yml` and is triggered exclusively after the CI workflow passes on the `main` branch.

## Pipeline Overview

```
PR merged to main
        │
        ▼
   ┌─────────┐    CI passes (build, test, typecheck)
   │   DEV   │    → publishes gsd-pi@<version>-dev.<sha> with @dev tag
   └────┬────┘    → local smoke test runs against the build
        ▼ (automatic if green)
   ┌─────────┐    smoke tests against installed binary + LLM fixture replay
   │  TEST   │    → promotes to @next tag
   └────┬────┘    → builds and pushes Docker image as :next
        ▼ (manual approval required)
   ┌─────────┐    optional live LLM integration tests
   │  PROD   │    → promotes to @latest tag
   └─────────┘    → tags Docker image as :latest
                  → creates GitHub Release
```

Concurrency is controlled per-SHA: `pipeline-${{ github.sha }}` with `cancel-in-progress: false`, so concurrent pipeline runs for the same commit are queued rather than cancelled.

## Workflow Triggers

### Pipeline trigger

`pipeline.yml` triggers on `workflow_run` events — specifically when the `CI` workflow completes with `conclusion == 'success'` on the `main` branch. This means the pipeline never runs on a failing CI build. Every stage of the promotion chain gates on prior stages completing successfully.

### Supporting workflows

| Workflow | File | Trigger | Purpose |
|---|---|---|---|
| CI | `ci.yml` | PR + push to main | Build, test, typecheck — gate for all promotions |
| Release Pipeline | `pipeline.yml` | After CI succeeds on main | Three-stage promotion |
| Native Binaries | `build-native.yml` | `v*` tags | Cross-compile platform binaries |
| Dev Cleanup | `cleanup-dev-versions.yml` | Weekly (Monday 06:00 UTC) | Unpublish `-dev.` versions older than 30 days |

## Stage 1: Dev Publish

**Job name:** `dev-publish`

**Runs on:** `ubuntu-latest` inside the `ghcr.io/gsd-build/gsd-ci-builder:latest` container, which includes the Rust toolchain and eliminates 3–5 minutes of toolchain setup per run.

**Steps:**

1. Checkout the repository
2. Mark the workspace safe for git (required inside containers)
3. Set up Node.js 22 pointing at the npm registry
4. Run `npm ci` to install dependencies
5. Run `npm run build` to compile
6. Run `pipeline:version-stamp` to append a `-dev.<sha>` suffix to the package version; the stamped version is captured as a job output (`dev-version`)
7. Publish to npm with `--tag dev` using the `NPM_TOKEN` secret
8. Run local smoke tests (`npm run test:smoke`) against the freshly built package

The `@dev` tag always points to the most recently merged PR. Format: `2.28.0-dev.a3f2c1b`. Old `-dev.` versions are cleaned up weekly with a 30-day retention window.

## Stage 2: Test and Verify

**Job name:** `test-verify`

**Depends on:** `dev-publish` (uses `dev-version` output)

**Runs on:** `ubuntu-latest` (no custom container — installs gsd-pi from npm)

**Steps:**

1. Set up Node.js 22
2. Install `gsd-pi@dev` globally from npm
3. Run smoke tests against the installed binary by setting `GSD_SMOKE_BINARY=$(which gsd)`
4. Run `npm ci` in the repo for the fixture test runner
5. Run LLM fixture tests (`npm run test:fixtures`) — replays recorded LLM conversations without hitting real APIs, at zero cost
6. Promote to `@next`: `npm dist-tag add gsd-pi@<dev-version> next`
7. Log in to GitHub Container Registry (GHCR)
8. Build the runtime Docker image and push it as `:next` and `:<dev-version>`

The `@next` tag represents a test candidate that has passed both smoke tests and fixture replay. It is safe for early adopters and beta testers.

## Stage 3: Production Release

**Job name:** `prod-release`

**Depends on:** `dev-publish` and `test-verify`

**Environment:** `prod` — this environment requires manual reviewer approval from maintainers. The job pauses here until a maintainer clicks "Review deployments" in GitHub Actions and approves.

**Steps:**

1. Set up Node.js 22
2. Run live LLM tests (`npm run test:live`) — this step is `continue-on-error: true` and only executes when `ANTHROPIC_API_KEY` and `OPENAI_API_KEY` secrets are present and `GSD_LIVE_TESTS=1` is set. Live tests are opt-in via the `RUN_LIVE_TESTS` variable on the `prod` environment.
3. Promote to `@latest`: `npm dist-tag add gsd-pi@<dev-version> latest`
4. Log in to GHCR
5. Pull the versioned image, tag it as `:latest`, and push
6. Strip the `-dev.<sha>` suffix to extract the base version
7. Create a GitHub Release (`gh release create`) with auto-generated release notes

### Approving a production release

1. A version reaches Test stage automatically after CI passes
2. The `prod-release` job shows "Waiting for review" in GitHub Actions
3. Click **Review deployments**, select the `prod` environment, and click **Approve**
4. The pipeline promotes the version to `@latest` and creates the GitHub Release

## Stage 4: CI Builder Image Update

**Job name:** `update-builder`

**Runs in parallel** with `dev-publish` (no dependency relationship).

This job checks whether the `Dockerfile` changed in the triggering commit. If it did, it rebuilds and pushes `ghcr.io/gsd-build/gsd-ci-builder:latest`. This keeps the CI build environment current without requiring a separate manual process.

## Docker Images

| Image | Base | Purpose | Tags |
|---|---|---|---|
| `ghcr.io/gsd-build/gsd-ci-builder` | `node:22-bookworm` | CI build environment with Rust toolchain | `:latest`, `:<date>` |
| `ghcr.io/gsd-build/gsd-pi` | `node:22-slim` | User-facing runtime | `:latest`, `:next`, `:v<version>` |

## Version Strategy and Dist-Tags

| Tag | Published when | Version format | Audience |
|---|---|---|---|
| `@dev` | Every merged PR | `2.28.0-dev.a3f2c1b` | Developers verifying fixes |
| `@next` | Auto-promoted from dev (after tests pass) | Same version | Early adopters, beta testers |
| `@latest` | Manually approved prod release | Same version | Production users |

Installing specific channels:

```bash
npx gsd-pi@dev      # bleeding edge, every merged PR
npx gsd-pi@next     # test candidate
npx gsd-pi@latest   # stable production release
npx gsd-pi          # same as @latest
```

Using Docker:

```bash
docker run --rm -v $(pwd):/workspace ghcr.io/gsd-build/gsd-pi:next --version
docker run --rm -v $(pwd):/workspace ghcr.io/gsd-build/gsd-pi:latest --version
```

## Checking if a Fix Has Landed

1. Find the PR's merge commit SHA (first 7 characters)
2. Check if it is in `@dev`: `npm view gsd-pi@dev version` — if the version ends in `-dev.<your-sha>`, your PR is in dev
3. Check if it promoted to `@next`: `npm view gsd-pi@next version`
4. Check if it is in production: `npm view gsd-pi@latest version`

## Gating Tests in CI

The pipeline only starts after `ci.yml` passes. Key gates:

- **Unit tests** (`npm run test:unit`) — includes `auto-session-encapsulation.test.ts`, which enforces that all auto-mode state is encapsulated in `AutoSession`. Any PR that adds module-level mutable state to `auto.ts` fails CI and blocks the entire pipeline.
- **Integration tests** (`npm run test:integration`)
- **Extension typecheck** (`npm run typecheck:extensions`)
- **Package validation** (`npm run validate-pack`)

## LLM Fixture Tests

The fixture system records and replays LLM conversations without hitting real APIs, making test runs free and deterministic.

**Running fixture tests:**

```bash
npm run test:fixtures
```

**Recording new fixtures:**

```bash
GSD_FIXTURE_MODE=record GSD_FIXTURE_DIR=./tests/fixtures/recordings \
  node --experimental-strip-types tests/fixtures/record.ts
```

Fixtures are JSON files in `tests/fixtures/recordings/`. Each file captures a conversation's request/response pairs and replays them by turn index.

**When to re-record fixtures:**

- Provider wire format changes (e.g., a new field in an Anthropic response)
- Tool definitions change (affects request shape)
- System prompt changes that cause turn count mismatches

## Rolling Back a Release

If a broken version reaches production:

```bash
# Roll back npm
npm dist-tag add gsd-pi@<previous-good-version> latest

# Roll back Docker
docker pull ghcr.io/gsd-build/gsd-pi:<previous-good-version>
docker tag ghcr.io/gsd-build/gsd-pi:<previous-good-version> ghcr.io/gsd-build/gsd-pi:latest
docker push ghcr.io/gsd-build/gsd-pi:latest
```

For `@dev` or `@next`, the next successful merge overwrites the tag automatically — no manual rollback needed.

## GitHub Configuration Requirements

| Setting | Value |
|---|---|
| Environment: `dev` | No protection rules |
| Environment: `test` | No protection rules |
| Environment: `prod` | Required reviewers: maintainers |
| Secret: `NPM_TOKEN` | All environments |
| Secret: `ANTHROPIC_API_KEY` | Prod environment only |
| Secret: `OPENAI_API_KEY` | Prod environment only |
| Variable: `RUN_LIVE_TESTS` | `false` (set to `true` to enable live LLM tests) |
| GHCR | Enabled for the `gsd-build` org |

## GSD Integration with CI (Headless Mode)

GSD itself can be run headlessly in CI pipelines using `gsd headless`. This is useful for teams that want to run GSD's auto-mode as part of their own automation:

```bash
gsd headless auto         # run auto-mode unattended
gsd headless new-milestone # create a milestone programmatically
gsd headless query         # read-only state inspection, returns JSON
```

`gsd headless auto` auto-restarts the entire process on crash (default 3 attempts with exponential backoff). `gsd headless query` returns phase, cost, progress, and next-unit as parseable JSON without spawning an LLM session — useful for status checks in CI dashboards.

## Team Workflow: PRs, Reviews, and Branch Strategy

GSD uses a trunk-based model centered on `main`:

- All feature work is done in short-lived branches and submitted as PRs
- PRs must pass CI (`ci.yml`) before they can be merged
- Merging to `main` automatically triggers the pipeline
- The `prod` environment gate requires a maintainer to explicitly approve each production promotion

For teams using GSD to develop software, the recommended branch strategy mirrors this: enable `mode: team` in `.gsd/preferences.md`, which sets `git.push_branches: true` and `git.pre_merge_check: true`. Each milestone gets its own `milestone/<MID>` branch and is squash-merged to main independently.

Multiple developers can run auto-mode simultaneously on different milestones. Milestone dependencies are declared in `M00X-CONTEXT.md` frontmatter using `depends_on: [M001-eh88as]`. GSD enforces that dependent milestones complete before starting downstream work.
