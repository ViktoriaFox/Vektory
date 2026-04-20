---
layout: default
title: "ADR 0001: Electron and TypeScript"
---

# ADR 0001: Use Electron and TypeScript for the desktop app

## Context

I need a cross-platform desktop application that can read/write files, run CPU-bound image processing (Potrace, Sharp), and provide a responsive UI. The product is a PNG-to-SVG converter with real-time preview and batch export.

## Decision

- **Electron** for the desktop shell: one codebase for Windows and macOS; access to Node and native APIs from the main process; web technologies for the UI.
- **TypeScript** for the whole codebase: type safety, shared types between main and renderer, better refactoring and tooling.

## Consequences

- **Positive:** Single codebase, fast iteration, rich ecosystem (React, Vite, Potrace, Sharp). TypeScript catches many bugs at compile time and keeps main/renderer contracts clear via `shared/types`.
- **Negative:** Larger bundle and memory footprint than a native app; dependency on Chromium/Node release cycle. I mitigate by keeping the main process focused (file + conversion only) and the renderer on a modern stack (Vite, React).
- **Follow-up:** Preload script and IPC API must be explicitly designed (see [ADR 0002](0002-context-isolation-and-preload-api)). Renderer stack chosen separately ([ADR 0003](0003-react-and-vite-for-renderer)).
