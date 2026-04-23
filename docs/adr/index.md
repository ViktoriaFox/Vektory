---
layout: default
title: Architecture Decision Records
nav_title: ADRs
---

# Architecture Decision Records

These records document the key design decisions made during development of Vektory. Each ADR captures the context, the decision, and its consequences — including trade-offs consciously accepted.

| # | Title | Decision |
|---|-------|----------|
| [0001](0001-electron-and-typescript) | Electron and TypeScript | Single codebase (Win/macOS) using Electron; TypeScript throughout for shared types between main and renderer |
| [0002](0002-context-isolation-and-preload-api) | Context isolation and preload-only API | Renderer has zero direct Node or IPC access; all calls go through a typed `contextBridge` surface (`window.electronAPI`) |
| [0003](0003-react-and-vite-for-renderer) | React and Vite for the renderer | React + Zustand for UI and state; Vite for HMR in dev, separate `tsc` build for main process |
| [0004](0004-ship-monochrome-tracing-first) | Ship monochrome tracing first | Defer multi-color / palette vectorization; ship focused single-color conversion with clear Known Limitations |
| [0005](0005-dark-background-inversion) | Dark background inversion | Preprocessing pipeline (flatten → negate → greyscale → threshold) to handle white-on-dark source images |
| [0006](0006-per-file-options-selective-persistence) | Per-file options with selective persistence | Dual-layer options (global + per-file); only `themeMode` and `showEditor` persist across restarts |
| [0007](0007-options-based-svg-cache) | Options-based SVG cache invalidation | Cache SVG + options per file; `JSON.stringify` comparison avoids re-tracing on file switch |
| [0008](0008-dual-independent-view-state) | Dual independent view state | Separate `ViewState` for SVG and Original views; zoom/pan preserved when toggling between them |
| [0009](0009-batch-concurrency-control) | Batch concurrency control | Process files in groups of 5 via `Promise.all`; prevents memory exhaustion on large batches |
| [0010](0010-tight-viewbox-cubic-bezier) | Tight SVG viewBox via cubic Bezier extrema | Solve derivative = 0 for tight bounds; zero-origin viewBox + translate for editor compatibility |
