# studio

## Package Metadata

| Field | Value |
|---|---|
| Name | `@gsd/studio` |
| Version | `0.0.0` |
| Description | (none) |
| License | (not specified in package.json) |
| Visibility | `private: true` — not published to npm |
| Module type | ESM (`"type": "module"`) |
| Main entry | `dist/main/index.js` (Electron main process) |

## Purpose in the Ecosystem

`@gsd/studio` is the **desktop GUI shell** for the GSD toolkit. It is an Electron application that wraps the same agent capabilities available in the terminal CLI into a windowed environment. The package is currently at an early scaffold stage (`version: 0.0.0`) and provides:

- A dark-themed Electron host window with platform-native title bar
- A React 19 renderer process with a design-system-level component foundation
- A defined but stub-implemented IPC bridge (`StudioBridge`) between the renderer and the main process
- A token-based design system (`colors`, `fonts`, `fontSizes`) that drives the visual language
- Tailwind CSS v4 for utility-class styling in the renderer

Because this package is `private`, it is never published. It is built and run directly from the monorepo.

## Scripts

```bash
npm run dev      # electron-vite dev server (hot reload for renderer)
npm run build    # electron-vite production build
npm run preview  # preview the production build
npm run test     # node --test test/*.test.mjs
```

The build toolchain is **electron-vite**, which runs Vite for the renderer process and TypeScript compilation for the main and preload processes.

## Dependencies

### Runtime dependencies

| Package | Purpose |
|---|---|
| `react` ^19.2.0 | UI rendering framework |
| `react-dom` ^19.2.0 | React DOM adapter |
| `@phosphor-icons/react` ^2.1.10 | Icon library (duotone icon set) |
| `react-resizable-panels` ^4.7.3 | Drag-to-resize panel layout |
| `zustand` ^5.0.8 | Lightweight state management |

### Dev dependencies

| Package | Purpose |
|---|---|
| `electron` ^41.0.3 | Desktop application runtime |
| `electron-vite` ^5.0.0 | Build toolchain for Electron + Vite |
| `tailwindcss` ^4.2.1 | CSS utility framework |
| `@tailwindcss/vite` ^4.2.1 | Tailwind v4 Vite plugin |
| `@vitejs/plugin-react` ^5.1.0 | React JSX transform for Vite |
| `typescript` ^5.9.3 | TypeScript compiler |

## Architecture

The application follows the standard Electron three-process model:

```
┌─────────────────────────────────────────┐
│         Main process                    │
│  studio/src/main/index.ts               │
│  Electron BrowserWindow, app lifecycle  │
└─────────────┬───────────────────────────┘
              │ contextBridge
┌─────────────▼───────────────────────────┐
│         Preload script                  │
│  studio/src/preload/index.ts            │
│  Exposes window.studio (StudioBridge)   │
└─────────────┬───────────────────────────┘
              │ window.studio.*
┌─────────────▼───────────────────────────┐
│         Renderer process                │
│  studio/src/renderer/src/App.tsx        │
│  React 19, Tailwind, Zustand            │
└─────────────────────────────────────────┘
```

### Main process (`src/main/index.ts`)

Creates a single `BrowserWindow` with:

```typescript
{
  width: 1400,
  height: 900,
  minWidth: 1100,
  minHeight: 720,
  backgroundColor: '#0a0a0a',
  titleBarStyle: 'hiddenInset',          // macOS: traffic-light buttons inset
  trafficLightPosition: { x: 16, y: 16 } // macOS only
  webPreferences: {
    preload: 'path/to/preload.mjs',
    contextIsolation: true,
    nodeIntegration: false                // renderer has no direct Node.js access
  }
}
```

In development the renderer loads from `ELECTRON_RENDERER_URL` (the Vite dev server). In production it loads the built `renderer/index.html`.

### Preload script (`src/preload/index.ts`)

Exposes the `StudioBridge` interface to the renderer via `contextBridge.exposeInMainWorld('studio', ...)`:

```typescript
export type StudioStatus = {
  connected: boolean;
};

export type StudioBridge = {
  onEvent: (callback: (event: unknown) => void) => () => void;
  sendCommand: (command: string, args?: Record<string, unknown>) => void;
  spawn: () => void;
  getStatus: () => Promise<StudioStatus>;
};
```

All four methods are currently stub implementations:
- `onEvent` — registers a listener but never fires events; returns a no-op unsubscriber
- `sendCommand` — silently discards commands
- `spawn` — no-op
- `getStatus` — always resolves `{ connected: false }`

This is the IPC surface that the renderer will use once the main process integrates with `@gsd/pi-coding-agent`. The bridge is accessible in the renderer as `window.studio`.

### Renderer entry (`src/renderer/src/App.tsx`)

The current renderer is a single `App` component that serves as a visual proof-of-concept for the design system. It renders a two-column layout (1.2fr / 0.8fr on large screens) with:

- A hero card showing the app name, description, and three status pills displaying the shell, theme accent, and monospace font
- A sidebar with typography proof, a code surface block, and three mini-metric cards for accent colour, UI font size, and monospace font

Token values from `lib/theme/tokens.ts` are rendered directly in the UI to verify the design system is wired correctly:

```tsx
import { colors, fonts, fontSizes } from './lib/theme/tokens';

// Displays '#d4a04e' (amber accent)
<dd className="font-medium text-accent">{colors.accent}</dd>

// Displays "'JetBrains Mono', ui-monospace, monospace"
<dd className="font-mono text-[13px]">{fonts.mono}</dd>
```

## Design System — `lib/theme/tokens.ts`

The token file exports three constant objects:

### `colors`

```typescript
export const colors = {
  bgPrimary:    '#0a0a0a',   // deep black page background
  bgSecondary:  '#111111',   // slightly lighter card/panel background
  bgTertiary:   '#1a1a1a',   // input or nested surface
  bgHover:      '#222222',   // hover state
  border:       '#262626',   // default border
  borderActive: '#333333',   // focused/active border
  textPrimary:  '#e5e5e5',   // primary readable text
  textSecondary:'#a3a3a3',   // supporting text
  textTertiary: '#737373',   // muted/label text
  accent:       '#d4a04e',   // warm amber system accent
  accentHover:  '#e0b366',   // accent on hover
  accentMuted:  'rgba(212, 160, 78, 0.15)', // translucent accent for backgrounds
} as const;
```

### `fonts`

```typescript
export const fonts = {
  sans: "'Inter', system-ui, sans-serif",
  mono: "'JetBrains Mono', ui-monospace, monospace",
} as const;
```

### `fontSizes`

```typescript
export const fontSizes = {
  hero:    '4.75rem',   // large display headings
  display: '3.5rem',
  title:   '2rem',
  bodyLg:  '1.125rem',
  body:    '0.9375rem', // default body text (15px)
  caption: '0.75rem',   // labels, chips
} as const;
```

These tokens are the source of truth for the visual language. They flow into Tailwind CSS custom properties (via `tailwind.config`) and into inline styles / className strings in React components.

## How to Use / Run

```bash
# From the monorepo root or from studio/
pnpm run dev    # starts electron-vite dev server + Electron
pnpm run build  # builds to dist/
pnpm run preview # runs the built app
```

During development, the Vite dev server runs at an automatically assigned port (typically `http://localhost:5173`). Electron loads this URL via `ELECTRON_RENDERER_URL`.

## Accessing the Bridge from Renderer Code

```typescript
// The bridge is typed via the global augmentation (add to renderer/src/env.d.ts)
declare global {
  interface Window {
    studio: import('../../preload/index').StudioBridge;
  }
}

// Then use:
const status = await window.studio.getStatus();
console.log('Connected:', status.connected);

window.studio.sendCommand('prompt', { text: 'Refactor this file' });

const unsubscribe = window.studio.onEvent((event) => {
  console.log('Agent event:', event);
});
```

## Current State and Roadmap

At v0.57.1 (monorepo version) / `0.0.0` (package version), the studio is a **T01 scaffold**: the Electron shell, design system, and IPC bridge types are in place, but no actual agent functionality has been wired. The `StudioBridge` stub methods, the `connected: false` status response, and the placeholder `App.tsx` content confirm this is a foundation-only state.

Planned future work referenced in the renderer:
- Three-column layout primitives (described as landing in T02)
- Real IPC wiring from `window.studio.spawn()` to a `@gsd/pi-coding-agent` process
- Event-driven streaming of agent output into the renderer
- Full session management UI

## Relationship to Other Packages

Studio is a consumer package only — it does not export any code used by sibling packages. Its eventual full form will call into `@gsd/pi-coding-agent` (via the preload IPC bridge to a spawned process) in the same way a user would call the `pi` CLI. The `react-resizable-panels` dependency anticipates a multi-panel layout with a file tree, conversation view, and editor pane.
