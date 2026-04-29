---
layout: default
title: Changelog
---

# Changelog

Format: [Keep a Changelog](https://keepachangelog.com/en/1.0.0/) · Versioning: [Semantic Versioning](https://semver.org/)

---

## [1.0.0] — 2026

### Architecture

- **Three-process Electron model**: main process (Node.js), preload bridge (isolated context), renderer (React + Vite). `contextIsolation: true`, `nodeIntegration: false` enforced throughout. See [ADR 0001](adr/0001-electron-and-typescript), [ADR 0002](adr/0002-context-isolation-and-preload-api).
- **Typed IPC surface**: renderer-initiated calls use `ipcRenderer.invoke()` → `ipcMain.handle()`, exposed through a typed `window.electronAPI` interface via `contextBridge`. Main-pushed events (menu actions) use `ipcRenderer.on()` listeners, also exposed through the preload. See [ADR 0002](adr/0002-context-isolation-and-preload-api).
- **Dual build pipeline**: `tsc` compiles the main process; Vite bundles the renderer. `concurrently` + `wait-on` orchestrate the development workflow. See [ADR 0004](adr/0004-react-and-vite-for-renderer).
- **Selective state persistence**: `useAppStore` persists only `themeMode` and `showEditor` across sessions via Zustand middleware. Conversion options, file list, and SVG cache are session-scoped and reset on restart.
- **Independent preview view state**: `usePreviewStore` maintains separate zoom/pan state for the SVG and Original views, so switching views does not reset your position in either.
- **Radix UI component layer**: all interactive primitives (sliders, selects, tabs, tooltips, checkboxes) use a single CSS-variable token system (`--space-*`, `--bg-*`, `--accent-*`). No hardcoded values in component files.
- **Tight SVG bounding box**: `pathBounds()` in `ConversionService` solves cubic bezier extrema analytically to compute a pixel-accurate `viewBox`, rather than using the control-point convex hull which over-estimates.

### Added

- PNG to SVG conversion via the Potrace algorithm with a Sharp preprocessing pipeline (blur, greyscale, threshold).
- Real-time preview with side-by-side comparison (original PNG / traced SVG).
- Tracing parameter controls: threshold, turn policy, artifact area (turdSize), corner smoothness (alphaMax), path optimisation tolerance.
- Three built-in presets: Simple, Detailed, Smooth.
- Batch conversion with controlled concurrency (5 files at a time) and bulk SVG export.
- Manual SVG path editor — inspect and edit generated path data in-app.
- Drag-and-drop file input and folder scanning for PNG files.
- Per-file conversion options — each file remembers its own settings within a session.
- Three-state theme toggle: Light / Dark / Auto (follows OS preference). Persisted across sessions.

### Known limitations

- Monochrome (single fill colour) tracing only. Multi-colour output deferred to a future version. See [ADR 0013](adr/0013-ship-monochrome-tracing-first).
- The macOS binary is not notarised — Gatekeeper will show a warning on first launch. See the [Install Guide](install) for the workaround.
- The Windows installer is not signed with an EV certificate — SmartScreen will show a warning. See the [Install Guide](install) for the workaround.

### Roadmap

- Dark background detection and colour inversion
- Multi-colour / palette-based vectorisation

- Apple Developer ID notarisation
- Windows EV code signing
- Smart threshold suggestion for low-contrast images
