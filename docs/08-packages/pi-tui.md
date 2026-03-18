# pi-tui

## Package Metadata

| Field | Value |
|---|---|
| Name | `@gsd/pi-tui` |
| Version | `0.57.1` |
| Description | Terminal User Interface library (vendored from pi-mono) |
| License | (not specified in package.json) |
| Module type | ESM (`"type": "module"`) |
| Entry point | `./dist/index.js` |
| Types | `./dist/index.d.ts` |

## Purpose in the Ecosystem

`@gsd/pi-tui` is the foundational rendering layer for every terminal-based view in the GSD monorepo. It provides a **differential rendering** TUI framework that keeps flicker to a minimum by comparing the previously rendered lines with the newly rendered ones and writing only what changed. All interactive output seen in `pi-coding-agent`'s interactive mode is built on top of this package — from the input editor and markdown viewer down to loader spinners and select lists.

The library is vendored from `pi-mono` and is intentionally kept as a standalone package with no `@gsd/*` sibling dependencies, so it can be used directly by any package that needs terminal output without pulling in the full agent stack.

## Dependencies

| Package | Purpose |
|---|---|
| `chalk` ^5.6.2 | ANSI colour helpers |
| `get-east-asian-width` ^1.3.0 | Correct column-width for CJK/wide characters |
| `marked` ^15.0.12 | Markdown-to-terminal-text parser |
| `mime-types` ^3.0.1 | MIME detection for image rendering |
| `koffi` ^2.9.0 (optional) | Windows FFI for `ENABLE_VIRTUAL_TERMINAL_INPUT` |

## Key Exports

### Core TUI engine

```typescript
import { TUI, Container, Component, Focusable, isFocusable, CURSOR_MARKER, visibleWidth } from '@gsd/pi-tui';
```

| Export | Kind | Description |
|---|---|---|
| `TUI` | class | Main TUI controller — extends `Container`, owns the render loop |
| `Container` | class | A `Component` that holds an ordered list of child components |
| `Component` | interface | The contract every renderable object must implement |
| `Focusable` | interface | Mixin for components that display a hardware cursor |
| `isFocusable` | function | Type guard for `Component & Focusable` |
| `CURSOR_MARKER` | constant | APC escape used to signal cursor position to `TUI` |
| `visibleWidth` | function | Column-width of a string, accounting for ANSI escapes and wide chars |

### Terminal abstraction

```typescript
import { ProcessTerminal, Terminal } from '@gsd/pi-tui';
```

| Export | Kind | Description |
|---|---|---|
| `Terminal` | interface | Minimal surface `TUI` needs from a terminal (write, dimensions, cursor) |
| `ProcessTerminal` | class | Production implementation wrapping `process.stdin`/`process.stdout` |

### Built-in components

```typescript
import {
  Box, CancellableLoader, Editor, Image, Input, Loader,
  Markdown, SelectList, SettingsList, Spacer, Text, TruncatedText
} from '@gsd/pi-tui';
```

| Export | Description |
|---|---|
| `Box` | Fixed-width bordered container |
| `CancellableLoader` | Spinner with a cancel-key hint |
| `Editor` | Multi-line text editor with theming and optional Kitty key support |
| `Image` | Inline image via Kitty or iTerm2 protocol |
| `Input` | Single-line text input |
| `Loader` | Simple spinner |
| `Markdown` | Renders markdown to styled terminal lines |
| `SelectList` | Arrow-key navigable list of items |
| `SettingsList` | Two-column key/value list with inline editing |
| `Spacer` | Emits blank lines |
| `Text` | Wraps a plain string (with optional ANSI) into component form |
| `TruncatedText` | Like `Text` but clipped to viewport width |

### Keyboard input

```typescript
import { Key, parseKey, matchesKey, decodeKittyPrintable, isKeyRelease, isKeyRepeat,
         isKittyProtocolActive, setKittyProtocolActive, KeyEventType, KeyId } from '@gsd/pi-tui';
```

`parseKey(data)` parses a raw stdin byte sequence into a `Key` object. `matchesKey(data, "ctrl+c")` is the canonical way to test whether a raw byte string matches a named key combo.

### Keybinding manager

```typescript
import { EditorKeybindingsManager, getEditorKeybindings, setEditorKeybindings,
         DEFAULT_EDITOR_KEYBINDINGS, EditorAction, EditorKeybindingsConfig } from '@gsd/pi-tui';
```

Provides named editor actions (`insert`, `delete`, `move-line-up`, …) mapped to configurable key sequences.

### Autocomplete

```typescript
import { CombinedAutocompleteProvider, AutocompleteProvider, AutocompleteItem, SlashCommand } from '@gsd/pi-tui';
```

`CombinedAutocompleteProvider` merges multiple `AutocompleteProvider` instances and is consumed by the `Editor` component.

### Terminal image support

```typescript
import {
  renderImage, encodeKitty, encodeITerm2, detectCapabilities, getCapabilities,
  allocateImageId, deleteKittyImage, deleteAllKittyImages,
  getCellDimensions, setCellDimensions, calculateImageRows,
  ImageProtocol, TerminalCapabilities, ImageDimensions, CellDimensions, ImageRenderOptions
} from '@gsd/pi-tui';
```

Auto-detects whether the terminal supports the Kitty or iTerm2 graphics protocol and renders images accordingly.

### Stdin buffering

```typescript
import { StdinBuffer, StdinBufferEventMap, StdinBufferOptions } from '@gsd/pi-tui';
```

`StdinBuffer` coalesces raw stdin chunks into discrete escape sequences, handling 50 ms assembly windows for sequences that arrive split across multiple `data` events.

### Utilities

```typescript
import { truncateToWidth, visibleWidth, wrapTextWithAnsi } from '@gsd/pi-tui';
import { fuzzyFilter, fuzzyMatch, FuzzyMatch } from '@gsd/pi-tui';
```

## Public API — Main Classes

### `Component` interface

```typescript
interface Component {
  render(width: number): string[];
  handleInput?(data: string): void;
  wantsKeyRelease?: boolean;
  invalidate(): void;
}
```

`render(width)` must return an array of strings — one per output line — each no wider than `width` visible columns. ANSI escape codes may appear in the strings but do not contribute to the width count. `handleInput` is called with raw stdin bytes when the component has focus. `invalidate()` clears any internal rendering cache.

### `TUI` class

`TUI` extends `Container` and is the top-level controller.

```typescript
const terminal = new ProcessTerminal();
const tui = new TUI(terminal);

// Add children
tui.addChild(myComponent);

// Start the event loop
tui.start();

// Set focus so keyboard input reaches a component
tui.setFocus(myComponent);

// Request re-render (batched via process.nextTick)
tui.requestRender();

// Force full re-render (clears cache)
tui.requestRender(true);

// Show a floating overlay
const handle = tui.showOverlay(overlayComponent, {
  anchor: 'center',
  width: '80%',
  maxHeight: '60%',
  margin: 2
});

// Later...
handle.hide();

// Clean up
tui.stop();
```

Key methods:

| Method | Description |
|---|---|
| `start()` | Activates raw mode, queries Kitty protocol, triggers first render |
| `stop()` | Moves cursor to end, restores cooked mode |
| `requestRender(force?)` | Schedules differential render on next tick; `force=true` clears cache |
| `setFocus(component)` | Routes keyboard input to `component`; updates `Focusable.focused` |
| `showOverlay(component, options?)` | Renders `component` floating above base content, returns `OverlayHandle` |
| `hideOverlay()` | Removes topmost overlay |
| `hasOverlay()` | `true` if any visible overlay is active |
| `addInputListener(fn)` | Inserts a pre-focus input interceptor; returns a dispose function |
| `invalidate()` | Recursively clears render caches on all children and overlays |

### `OverlayOptions`

```typescript
interface OverlayOptions {
  width?: number | `${number}%`;
  minWidth?: number;
  maxHeight?: number | `${number}%`;
  anchor?: 'center' | 'top-left' | 'top-right' | 'bottom-left' |
           'bottom-right' | 'top-center' | 'bottom-center' |
           'left-center' | 'right-center';
  offsetX?: number;
  offsetY?: number;
  row?: number | `${number}%`;
  col?: number | `${number}%`;
  margin?: number | { top?:number; right?:number; bottom?:number; left?:number };
  visible?: (termWidth: number, termHeight: number) => boolean;
  nonCapturing?: boolean;
}
```

`nonCapturing: true` renders the overlay without stealing keyboard focus.

### `ProcessTerminal` class

```typescript
const terminal = new ProcessTerminal();
terminal.start(onInput, onResize);
terminal.write('\x1b[2J');   // raw escape
terminal.hideCursor();
terminal.showCursor();
terminal.setTitle('My App');
await terminal.drainInput(1000, 50); // flush Kitty key-release events on exit
terminal.stop();
```

On macOS/Linux `ProcessTerminal` queries the Kitty keyboard protocol (`CSI ? u`) and enables it with flags 1+2+4 (disambiguate, event types, alternate keys) if the terminal responds. On Windows it additionally enables `ENABLE_VIRTUAL_TERMINAL_INPUT` via `koffi` when available.

## Key Internal Architecture

### Differential render loop

Every `requestRender()` call schedules `doRender()` via `process.nextTick`. On each render:

1. All child components are asked to `render(width)`, returning `string[]` lines.
2. Active overlays are composited on top using column-level splicing (`compositeLineAt`).
3. The new line array is compared against `previousLines`. Only lines between `firstChanged` and `lastChanged` are written, wrapped in `CSI ?2026h / CSI ?2026l` (synchronized output mode) to prevent flicker.
4. If terminal dimensions changed, a full `ESC[2J ESC[H` clear-and-redraw is issued instead.

### CURSOR_MARKER

Focusable components embed `CURSOR_MARKER` (`\x1b_pi:c\x07`) at the cursor character position in their `render()` output. `TUI.extractCursorPosition()` scans the rendered lines for this marker, strips it, and repositions the hardware cursor with `CSI row B / CSI col G` for correct IME candidate window placement.

### Kitty keyboard protocol

`ProcessTerminal` sends `CSI ? u` immediately after enabling raw mode and waits up to 150 ms for a `CSI ? <flags> u` response. If received, it pushes flags `7` (`CSI > 7 u`). If no response, it falls back to xterm `modifyOtherKeys` mode 2 (`CSI > 4;2 m`). This is handled through `StdinBuffer` so split-arrival responses are reassembled correctly.

## How to Use / Import

```typescript
import { TUI, ProcessTerminal, Text, Loader, Markdown } from '@gsd/pi-tui';

class StatusBar implements Component {
  render(width: number): string[] {
    return [`Ready — ${new Date().toISOString()}`.slice(0, width)];
  }
  invalidate() {}
}

const terminal = new ProcessTerminal();
const tui = new TUI(terminal);
tui.addChild(new StatusBar());
tui.start();

// Update display on demand
setInterval(() => tui.requestRender(), 1000);

process.on('SIGINT', () => {
  tui.stop();
  process.exit(0);
});
```

## Environment Variables

| Variable | Effect |
|---|---|
| `PI_HARDWARE_CURSOR=1` | Force show hardware cursor at all times |
| `PI_CLEAR_ON_SHRINK=1` | Clear empty rows when content shrinks (reduces artifacts) |
| `PI_TUI_DEBUG=1` | Write render debug logs to `/tmp/tui/` |
| `PI_TUI_WRITE_LOG=<path>` | Append every `terminal.write()` call to a file |
| `PI_DEBUG_REDRAW=1` | Log full-redraw reasons to `~/.pi/agent/pi-debug.log` |
