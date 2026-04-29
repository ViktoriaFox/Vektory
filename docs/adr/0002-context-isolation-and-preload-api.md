---
layout: default
title: "ADR 0002: Context Isolation and Preload-only API"
---

# ADR 0002: Context isolation and preload-only API for the renderer

## Context

I come from a security background. When I build or review any application, I evaluate it through a security lens by default. With Electron that reflex matters especially: the default configuration is **permissive** in ways that most web tutorials don't flag clearly enough.

In a default Electron setup with `nodeIntegration: true`, any JavaScript running in the renderer has **full access to the Node API**, `require('fs')`, child processes, the file system. A single DOM XSS-able path (an image, a user-supplied string, a third-party script) turns into full workstation compromise. For a desktop tool that reads user files and writes SVGs to disk, it's the core workflow.

Two specific threats applied directly to Vektory:

1. **Unrestricted file system access from renderer code.** If the renderer can touch `fs` directly, any injection vector in the UI lets an attacker read or write arbitrary files.
2. **XSS via SVG content.** SVG is XML. SVG supports scripting attributes (`onload`, `<script>` tags, `javascript:` URIs, foreign objects). Vektory's central workflow renders user-supplied SVG (the conversion output) into the DOM for live preview. Rendering un-sanitized SVG is a XSS vector. Combined with threat #1, an attacker could chain the two: craft a malicious PNG → vectorization emits SVG with an `onload` payload → preview injects it → payload runs with Node privileges → full file system access.

The architecture needed to close both vectors by design.

## Options considered

**Default Electron (`nodeIntegration: true`, no context isolation)**
Trivially exploitable. Rejected on day 1.

**`nodeIntegration: false` only, without context isolation**
Partial protection. The renderer loses direct Node access, but without context isolation, the preload script shares a JavaScript context with the page, meaning anything the preload pulls in becomes reachable from the renderer. Still exploitable. Rejected.

**Full restrictive stack**
`nodeIntegration: false` + `contextIsolation: true` + `sandbox` + `contextBridge` + CSP. Electron's official recommendation, and the only option that actually closes both threat vectors.

## Decision

**A four-layer defense-in-depth model**:

1. **Chromium sandbox**: renderer runs in the standard Chromium process sandbox. OS-level isolation prevents general code execution outside the sandbox boundary even if a Chromium exploit lands.
2. **Context isolation** (`contextIsolation: true`): the renderer's JavaScript context and the preload's JavaScript context are **separate V8 contexts**. The preload can import Node and Electron APIs; none of those are reachable from the renderer's context. Data crosses the boundary via **structural cloning** (serialized copies), not shared references. Object identity is lost at the boundary, which is the whole point, no exploit can leak a function or a live object reference across.
3. **`contextBridge.exposeInMainWorld`**: the renderer sees *only* what I explicitly expose. The API is a named allowlist: `openFiles`, `convertPngToSvg`, `saveSvg`, `batchConvertAndSave`, `validateFiles`, and a small set of event listeners for main-pushed events (menu actions). Adding a new capability requires deliberate code in three places (main handler, preload binding, renderer type). There's no "just call `fs` from renderer" shortcut by design.
4. **Content Security Policy** (via `webPreferences`): final layer. Even within the renderer, CSP restricts what scripts can load and execute.

The architecture:

<pre class="mermaid">
%%{init: {
  'theme': 'base',
  'themeVariables': {
    'background': '#1e182c',
    'primaryColor': '#2a2240',
    'primaryTextColor': '#ECEAF2',
    'primaryBorderColor': '#8A75C8',
    'lineColor': '#8F8AA3',
    'textColor': '#ECEAF2',
    'clusterBkg': '#1e182c',
    'clusterBorder': '#8A75C8',
    'edgeLabelBackground': '#241e34'
  }
}}%%
flowchart TB
    subgraph renderer ["Renderer, Chromium sandbox"]
        ReactApp["React App<br>UI · live preview"]
    end

    subgraph preload ["Preload, isolated context"]
        CB["contextBridge<br>explicit allowlist"]
    end

    subgraph main ["Main, full Node.js"]
        Handler["ipcMain.handle(...)"]
        Handler --> Deps["Sharp · Potrace · fs · dialog"]
    end

    ReactApp -- "window.electronAPI.x(...)" --> CB
    CB == "ipcRenderer.invoke (structural clone)" ==> Handler

    style renderer fill:#241e34,stroke:#7F9E90,stroke-width:1.5px
    style preload fill:#3a3050,stroke:#D8B96A,stroke-width:1.5px
    style main fill:#2a2240,stroke:#8A75C8,stroke-width:1.5px
</pre>

## Rationale

**Why this specific stack.**
Each layer defends against a different failure mode:

- **Chromium sandbox**: defends against a renderer-code-execution exploit escaping to the OS.
- **Context isolation**: defends against the renderer *gaining privileged JS objects* even if the renderer itself is compromised. Without it, a single XSS is enough; with it, an XSS gets you a sandboxed page with no privileged access.
- **contextBridge allowlist**: defends against the preload accidentally exposing more than intended. The API surface is explicit and reviewable.
- **CSP**: defends against script injection within the renderer itself (e.g. unexpected inline scripts, external script loading).


**Structural cloning.**
Every piece of data crossing the preload boundary is a serialized copy. I can't pass a class instance, a function, a live DOM node, or a Node stream. Architecturally it's the whole reason contextIsolation works: **object identity does not cross the boundary, so no exploit can escape by holding a reference**. I lean into this by defining all IPC payloads as plain serializable shapes in a shared `types.ts` (see [ADR 0001](0001-electron-and-typescript) for the contract-typing reason).

**The IPC channels.**
The set of `ipcMain.handle(...)` handlers in main: `convert-png-to-svg`, `batch-convert`, `save-svg`, `open-files`, `open-folder`, `validate-files`, `select-save-directory`, `get-file-info`, `find-png-files`, `check-paths-are-directories`, `batch-convert-and-save`. Each maps to a named method on `window.electronAPI`. Adding a capability requires deliberate work in three places, main handler, preload binding, renderer TypeScript type.

## Consequences

**Positive**
- Both primary threats (file system access from renderer, XSS-to-privilege chain via SVG) are closed by architecture.
- The API surface is legible and typed end-to-end.
- Adding new capabilities is deliberate, no silent expansion of what the renderer can do.

**Negative**
- **Every new capability costs three edits.** Main handler, preload binding, renderer type.
- **Serializable-only contracts.** No passing class instances, functions, streams, or live references across the boundary. All IPC payloads must be plain JSON-serializable objects. Shapes need to be designed for serialization up front.

## On DOMPurify, emergent through workflow, validated through review

Vektory sanitizes SVG content with **DOMPurify** before injecting it into the DOM for live preview. This is the second defensive layer specifically against XSS via SVG:

- **Context isolation** ensures that even if malicious JS runs in the renderer, it has no privileged Node/IPC access.
- **DOMPurify** ensures that malicious SVG **doesn't run in the renderer at all**.

Together: an attacker-supplied SVG can't execute, and even if sanitizer bypass existed, it couldn't escape the sandbox.

How DOMPurify ended up in the codebase, this connects to a thread that runs through [ADR 0001](0001-electron-and-typescript) and [ADR 0011](0011-radix-over-mui-and-headless-ui). I did not explicitly plan "add DOMPurify." I grounded the LLM collaborating on the implementation in a **security-primed context**: contextIsolation as default, threat model articulated, hostile SVG input expected, XSS vectors named. Within that context, DOMPurify surfaced naturally as the model's suggestion for sanitizing SVG before DOM injection. I reviewed the fit against the threat model, verified the integration path, and accepted it.

Which leads us to **good LLM grounding produces emergent architectural quality**. The discipline I invest in framing the problem pays back in architectural suggestions I wouldn't have explicitly planned. The LLM becomes a collaborator surfacing patterns within the context I've set up.

This connects:
- [ADR 0001](0001-electron-and-typescript): *"Pick tools I can review, keep the human in the loop."*
- [ADR 0011](0011-radix-over-mui-and-headless-ui): *"Unopinionated tools require explicit discipline in the prompt."*
- **This ADR**: *"Well-grounded security context surfaces emergent hardening, emergent through workflow, validated through review."*


## Follow-up

- The preload API and IPC channel list should be periodically audited against current capabilities.
- If I ever need to pass non-serializable data (e.g. streaming a large file), the architecture is ready via `MessageChannel`, still via contextBridge.
- CSP policy should be tightened further as the app matures (stricter `script-src`, `img-src`, etc.).