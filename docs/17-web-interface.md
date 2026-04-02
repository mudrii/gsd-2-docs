# Web Interface

## Overview

GSD includes a browser-based web interface built on Next.js 16 with a PTY-backed terminal. The UI renders a full workspace dashboard in the browser, giving you project management, real-time progress monitoring, and multi-project support without leaving a browser tab.

Introduced in **v2.41.0** ([#1717](https://github.com/gsd/gsd/pull/1717)).

Key technology:
- **Next.js** app router with standalone output mode
- **xterm.js** (v6) for terminal rendering
- **node-pty** for server-side pseudo-terminal sessions
- **Radix UI** + **Tailwind CSS** for the component layer
- **Zustand-style** external store (`gsd-workspace-store.tsx`) for client state
- **next-themes** for dark/light mode switching

## Starting Web Mode

Launch the web interface with either form:

```bash
gsd --web
gsd web
```

Both start a local web server, wait for the `/api/boot` endpoint to become ready (up to 180 seconds), then open the dashboard in your default browser.

You can also target a specific project directory:

```bash
gsd --web /path/to/project
gsd web start /path/to/project
gsd web /path/to/project
```

### CLI Flags

| Flag | Default | Added | Description |
|------|---------|-------|-------------|
| `--web` | -- | v2.41.0 | Enable web mode |
| `--host <addr>` | `127.0.0.1` | v2.42.0 | Bind address for the web server |
| `--port <port>` | random available | v2.42.0 | Port for the web server |
| `--allowed-origins <origins>` | none | v2.42.0 | Comma-separated additional CORS origins |

Example with all flags:

```bash
gsd --web --host 0.0.0.0 --port 8080 --allowed-origins "https://tunnel.example.com"
```

The `--allowed-origins` flag is intended for secure tunnel setups (Tailscale Serve, Cloudflare Tunnel, ngrok, etc.) where the browser origin differs from the server bind address.

### Stopping Web Mode

```bash
gsd web stop              # stop the instance for the current directory (legacy PID file)
gsd web stop /path/to/dir # stop the instance for a specific project
gsd web stop all          # stop all running instances
```

GSD maintains a multi-instance registry (`web-instances.json`) that tracks every running web server by project path, PID, port, and start time. The `stop` subcommand consults this registry and falls back to a legacy PID file when no registry entry exists.

### Stale Process Cleanup

Before launching, GSD checks whether a previous web server is still registered for the same project directory. If one is found (e.g. from a terminal that was closed without clean shutdown), GSD kills it automatically to prevent `EADDRINUSE` errors. This was added in v2.43.0 ([#1934](https://github.com/gsd/gsd/pull/1934), [#2034](https://github.com/gsd/gsd/pull/2034)).

### Host Resolution

GSD resolves the web host in two ways depending on how it was installed:

1. **Packaged standalone** -- looks for `dist/web/standalone/server.js` and spawns it directly with the Node.js binary.
2. **Source dev** -- looks for `web/package.json` and runs `npm run dev` with `--hostname` and `--port` arguments.

## Authentication

The web server generates a random 32-byte hex bearer token at launch. The token is passed to the browser via the URL fragment:

```
http://127.0.0.1:<port>/#token=<hex>
```

URL fragments are never sent in HTTP requests or logged by servers/proxies, keeping the token local to the machine.

### Token Lifecycle

1. On first load, the client extracts the token from the fragment.
2. The token is persisted to `localStorage` (keyed as `gsd-auth-token`) so it survives page refreshes and is accessible from all tabs on the same origin.
3. The fragment is cleared from the address bar via `history.replaceState`.
4. All subsequent API calls attach the token via the `Authorization: Bearer` header.
5. For EventSource (SSE) connections, which cannot send custom headers, the token is appended as a `?_token=` query parameter.

Using `localStorage` (switched from `sessionStorage` in v2.53.0, [#2785](https://github.com/gsd/gsd/pull/2785)) means the token is shared across all tabs on the same origin. Because each GSD instance binds to a unique random port, the origin already scopes the token to that instance.

Cross-tab synchronization is handled via the `storage` event -- when one tab writes a new token, other tabs pick it up immediately.

### Auth Token Gate

When no token is available (missing `#token=` fragment and no `localStorage` entry), `authFetch()` returns a synthetic 401 Response instead of making an unauthenticated request. This lets the UI show an unauthenticated boot state and recovery screen rather than silently cascading 401 errors. Added in v2.52.0 ([#2740](https://github.com/gsd/gsd/pull/2740)).

### Server-Side Validation

The proxy middleware (`proxy.ts`) validates every `/api/*` request:

1. **Origin check** -- if an `Origin` header is present, it must match the expected `http://<host>:<port>` or be listed in `GSD_WEB_ALLOWED_ORIGINS`. Mismatches return 403.
2. **Bearer token check** -- the `Authorization: Bearer` header (or `_token` query parameter for SSE) must match `GSD_WEB_AUTH_TOKEN`. Mismatches return 401.

If no `GSD_WEB_AUTH_TOKEN` is configured (e.g. dev mode without the launch harness), auth is bypassed entirely.

## Dashboard

The main page (`page.tsx`) dynamically imports the `GSDAppShell` component with SSR disabled. The app shell provides:

- **Sidebar** with milestone explorer and collapsible navigation
- **Multiple views**: dashboard, power (terminal), chat, roadmap, files, activity, visualizer
- **Project tabs** via `ProjectStoreManagerProvider` -- switch between projects without restarting the server
- **Command surface** for slash command dispatch from the browser
- **Status bar** with connection state, project info, and scope badges
- **Onboarding gate** for first-run API key setup and provider configuration
- **Update banner** for version notifications

### Workspace Store

Client-side state is managed by `gsd-workspace-store.tsx`, a React context + `useSyncExternalStore` pattern (similar to Zustand). Key state includes:

| State | Description |
|-------|-------------|
| `WorkspaceStatus` | `idle`, `loading`, `ready`, `error`, `unauthenticated` |
| `WorkspaceConnectionState` | `idle`, `connecting`, `connected`, `reconnecting`, `disconnected`, `error` |
| `BridgePhase` | `idle`, `starting`, `ready`, `failed` |

The store uses `authFetch` and `appendAuthParam` from `auth.ts` for all API communication, and handles command surface state for diagnostics, doctor reports, forensics, settings, session browsing, git summaries, and more.

### Slash Commands

The workspace store integrates `browser-slash-command-dispatch` to let users run GSD slash commands directly from the browser UI. The command surface provides an overlay with results, diagnostics, and action feedback.

### Context-Aware Launch

When launching from within a project directory that lives under a configured dev root, GSD automatically resolves to the one-level-deep project directory. This is read from the web preferences file and ensures the browser opens directly into the correct project.

## Dark Mode and Theming

The app uses `next-themes` with `ThemeProvider` defaulting to `"dark"`. The theme attribute is applied as a CSS class on the `<html>` element.

### Terminal Theming

Terminal colors are defined in `xterm-theme.ts` with separate palettes for dark and light mode:

**Dark theme** -- `#0a0a0a` background, `#e4e4e7` foreground, zinc-based ANSI colors.

**Light theme** -- `#f5f5f5` background, `#18181b` foreground, with adjusted ANSI colors for readability on light surfaces (e.g. ANSI white maps to `#52525b` instead of a light gray).

Terminal options include:
- Cursor: blinking bar
- Font: SF Mono / Cascadia Code / Fira Code / Menlo fallback chain
- Font size: 13px default
- Line height: 1.35
- Scrollback: 10,000 lines

Light theme terminal contrast was progressively improved across v2.52.0 through v2.56.0 ([#2674](https://github.com/gsd/gsd/pull/2674), [#2819](https://github.com/gsd/gsd/pull/2819)).

Dark mode token floor and flattened opacity tier system were added in v2.52.0 ([#2734](https://github.com/gsd/gsd/pull/2734)).

## Mobile Support

The web UI was made mobile responsive in v2.45.0 ([#2354](https://github.com/gsd/gsd/pull/2354)). The layout includes:

- Viewport meta tag: `width=device-width, initialScale=1, maximumScale=1, userScalable=false`
- Mobile navigation toggle (hamburger menu)
- Mobile milestone explorer toggle

## Architecture

### Next.js App Router

The web UI uses Next.js 16 with the app router. The Next.js config (`next.config.mjs`) sets:

- `output: 'standalone'` for self-contained deployment
- `outputFileTracingRoot` pointed at the repo root
- `images.unoptimized: true`
- `serverExternalPackages: ['@gsd/native', 'node-pty']` to keep native addons out of the webpack bundle
- Custom webpack `extensionAlias` to resolve `.js` imports to `.ts` source (Turbopack does not support this, so builds use the `--webpack` flag)
- Explicit externalization of `node:module` and `@gsd/native` for the server bundle

### PTY Manager

`pty-manager.ts` manages server-side pseudo-terminal instances. Each terminal session has:

- A unique session ID
- An `IPty` instance from `node-pty`
- A set of listener callbacks for streaming output
- A circular buffer capped at 1 MB (`MAX_SESSION_BUFFER_BYTES`) that drops oldest chunks when full
- An `alive` flag for lifecycle tracking

Sessions are stored on `globalThis` to persist across Turbopack/HMR module re-evaluations in dev mode. PTY output is buffered and streamed to clients via SSE; input arrives via POST.

### Proxy

`proxy.ts` acts as Next.js middleware on all `/api/*` routes, enforcing bearer token authentication and origin validation (see [Authentication](#authentication) above).

### Workspace Store

`gsd-workspace-store.tsx` is the central client-side state container. It manages boot data, connection state, terminal lines, command surface overlays, and project context. All API calls go through `authFetch()` which handles token injection and synthetic 401 responses.

### Shutdown Gate

`shutdown-gate.ts` prevents premature server exit when a user refreshes the page:

1. `pagehide` fires in the browser, sending `POST /api/shutdown`
2. `scheduleShutdown()` starts a 3-second timer
3. If the page refreshes, `GET /api/boot` calls `cancelShutdown()` and clears the timer
4. If the tab is truly closed, the timer fires and calls `process.exit(0)`

When `GSD_WEB_DAEMON_MODE=1` (e.g. behind a reverse proxy for remote access), `scheduleShutdown()` is a no-op -- no client tab can exit the server. This was added in v2.58.0 ([#2842](https://github.com/gsd/gsd/pull/2842)).

### Environment Variables

The web server process receives these environment variables from the launcher:

| Variable | Description |
|----------|-------------|
| `GSD_WEB_HOST` | Bind address |
| `GSD_WEB_PORT` | Port number |
| `GSD_WEB_AUTH_TOKEN` | Bearer token for API authentication |
| `GSD_WEB_PROJECT_CWD` | Default project working directory |
| `GSD_WEB_PROJECT_SESSIONS_DIR` | Session storage directory |
| `GSD_WEB_PACKAGE_ROOT` | Root of the GSD package |
| `GSD_WEB_HOST_KIND` | `packaged-standalone` or `source-dev` |
| `GSD_WEB_ALLOWED_ORIGINS` | Comma-separated additional CORS origins |
| `GSD_WEB_DAEMON_MODE` | Set to `1` to disable shutdown-on-tab-close |
| `NEXT_PUBLIC_GSD_DEV` | Set to `1` in source-dev mode |

## Known Issues and Troubleshooting

### Windows: Web Build Skipped

The web build is skipped on Windows because Next.js webpack encounters `EPERM` errors on system directories. The CLI remains fully functional; only the browser UI is unavailable. This has been the case since v2.41.0.

### EADDRINUSE on Launch

If a previous GSD web server was not cleanly shut down (e.g. terminal closed, crash), the port may still be in use. Since v2.43.0, GSD automatically kills stale server processes for the same project directory before launching. If you still encounter port conflicts across different projects, use `gsd web stop all` to clear all instances.

### Compiled .js Module Resolution

Subprocess calls under `node_modules` may fail to resolve compiled `.js` modules. This was fixed in v2.44.0 ([#2320](https://github.com/gsd/gsd/pull/2320)) by updating the resolution logic for all subprocess calls.

### Node v24 Compatibility

Node v24 introduced breaking changes to type stripping that caused `ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING` on web boot. Fixed in v2.42.0 ([#1864](https://github.com/gsd/gsd/pull/1864)).

### Auth Token Loss on Refresh

Before v2.42.0, refreshing the page required re-authentication. The token was first persisted to `sessionStorage` (v2.42.0, [#1877](https://github.com/gsd/gsd/pull/1877)), then moved to `localStorage` (v2.53.0, [#2785](https://github.com/gsd/gsd/pull/2785)) to enable multi-tab support.

### Dashboard Metrics Showing Zero

When dashboard metrics are zero, the web UI falls back to project-level totals. Fixed in v2.58.0 ([#2847](https://github.com/gsd/gsd/pull/2847)).
