---
layout: default
title: "ADR 0002: Context Isolation and Preload API"
---

# ADR 0002: Context isolation and preload-only API for renderer

## Context

Electron renderer processes can be exposed to Node or IPC in unsafe ways. I need a clear, secure boundary: the renderer must only trigger file operations and conversion via the main process, with no direct Node or `require` usage in the UI process.

## Decision

- **Context isolation** is enabled; **nodeIntegration** is disabled in the renderer.
- A **preload script** (`preload.ts`) runs in an isolated context and uses `contextBridge` to expose a single, typed API (`window.electronApi`) to the renderer.
- All file system access, dialogs, and conversion run in the main process. The renderer calls methods like `openFiles()`, `convertPngToSvg(path, options)`, `saveSvg(content, filename)` via the preload API; the preload uses `ipcRenderer.invoke()` to call `ipcMain.handle()` in the main process.
- Main can push events to the renderer (e.g. menu actions: export, file selection) via `ipcRenderer.on()` listeners also exposed through the preload. These are one-directional pushes from main — the renderer never initiates communication this way.

## Consequences

- **Positive:** Clear security boundary; renderer cannot touch the file system or Node directly. API is easy to document and type (TypeScript interfaces in preload and shared types).
- **Negative:** Every new capability requires a preload method and an IPC handler in main. I keep the surface small and consistent (file ops, conversion, validation).
- **Follow-up:** All renderer-initiated communication goes through `invoke()`/`handle()`. New push events from main (menu actions, system events) use `ipcRenderer.on()` exposed via the preload; no ad-hoc raw channel access from renderer code.
