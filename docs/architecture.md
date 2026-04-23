---
layout: default
title: Architecture
---

# Architecture

Vektory follows Electron's three-process model with a strict security boundary: the renderer is sandboxed — it has no access to Node, the file system, or raw IPC. All capabilities are exposed through a single typed bridge.

---

## Containers

![Vektory architecture — Designer interacts with the Renderer process; the Preload script is the only bridge between Renderer and Main; Main owns Sharp, Potrace, and all file system access](assets/architecture.png)

- `nodeIntegration: false`, `contextIsolation: true` — renderer cannot touch Node or IPC directly
- Preload exposes two patterns: `invoke()` for renderer-initiated calls; `on()` callbacks for main-pushed events (menu actions)
- Main process owns all I/O, conversion, and path validation

See [ADR 0002](adr/0002-context-isolation-and-preload-api) · [ADR 0001](adr/0001-electron-and-typescript)

---

## Conversion flow

<pre class="mermaid">
sequenceDiagram
    actor User
    participant UI as Renderer (React)
    participant Store as Zustand Store
    participant Bridge as Preload
    participant Main as Main Process
    participant Sharp
    participant Potrace

    User->>UI: Drop PNG / select file
    UI->>Store: setSelectedFiles()
    Store-->>UI: currentFile updated
    Note over UI: 500 ms debounce
    UI->>Bridge: electronAPI.convertPngToSvg(path, options)
    Bridge->>Main: ipcRenderer.invoke('convert-png-to-svg')
    Main->>Sharp: flatten → greyscale → threshold
    Sharp-->>Main: Buffer
    Main->>Potrace: trace(buffer, options)
    Potrace-->>Main: SVG path data
    Main->>Main: pathBounds() — cubic Bezier extrema
    Main-->>Bridge: svgContent
    Bridge-->>UI: svgContent
    UI->>UI: sanitizeSvg() — DOMPurify
    UI->>Store: setCurrentSvg(sanitized)
    Note over Store: cache[path] = { svg, options }
    UI->>UI: svgContainer.innerHTML = svg
</pre>

- Option changes trigger a 500 ms debounced re-conversion — timeout ID is stored in Zustand so any component can cancel it
- SVG is sanitized with DOMPurify before storing in state, then written directly to `innerHTML` (bypasses React re-render for large SVGs)
- Result is cached per file keyed on the full options object — switching back to a previously converted file is instant

See [ADR 0007](adr/0007-options-based-svg-cache) · [ADR 0010](adr/0010-tight-viewbox-cubic-bezier)

---

## Batch export

<pre class="mermaid">
sequenceDiagram
    actor User
    participant UI as Renderer
    participant Bridge as Preload
    participant Main as Main Process
    participant FS as File System

    User->>UI: Export All SVGs
    UI->>Bridge: electronAPI.batchConvertAndSave(paths, optionsArray)
    Bridge->>Main: ipcRenderer.invoke('batch-convert-and-save')
    loop groups of 5
        Main->>Main: Promise.all(group.map(convertPngToSvg))
        Main->>FS: write SVG files
    end
    Main-->>Bridge: BatchConversionResult[]
    Bridge-->>UI: results
    UI->>UI: show summary
</pre>

- Files are processed in sequential groups of 5 using `Promise.all` per group
- Each file can carry its own `ConversionOptions` via the options array
- Concurrency of 5 prevents memory exhaustion from simultaneous Sharp + Potrace processes

See [ADR 0009](adr/0009-batch-concurrency-control)

---

## State model

**`useAppStore`** — conversion options, file list, SVG cache, theme, editor visibility.
- Only `themeMode` and `showEditor` survive a restart (Zustand `persist` + `partialize`)
- Each file stores its own last-used options in `fileOptionsByPath`; switching files restores those settings

**`usePreviewStore`** — independent `ViewState` (zoom, pan) for the SVG view and the Original view.
- Toggling between views preserves each view's zoom and pan position independently

See [ADR 0006](adr/0006-per-file-options-selective-persistence) · [ADR 0008](adr/0008-dual-independent-view-state)

---

## Build pipeline

| Process | Tool | Output |
|---------|------|--------|
| Main process | `tsc` | `dist/main/` |
| Renderer | Vite | `dist/renderer/` |
| Dev orchestration | `concurrently` + `wait-on` | Electron starts after Vite server is ready |

Main process changes require a full Electron restart. Renderer-only changes benefit from Vite HMR.

See [ADR 0003](adr/0003-react-and-vite-for-renderer)
