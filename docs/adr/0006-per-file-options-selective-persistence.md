---
layout: default
title: "ADR 0006: Per-File Options with Selective Persistence"
---

# ADR 0006: Per-file conversion options with selective persistence

## Context

I wanted users to work with multiple PNG files in one session, each potentially needing different threshold or corner settings. At the same time, session state (file list, conversion results, per-file settings) should not persist across app restarts — it is tied to the current working context and would be confusing if restored stale.

## Decision

- Maintain a dual-layer options system: `conversionOptions` (global, synced to UI controls) and `fileOptionsByPath: Record<string, ConversionOptions>` (per-file overrides stored by path).
- When the user switches files, load that file's stored options or fall back to the current global defaults.
- Use Zustand's `persist` middleware with `partialize` to persist **only** `themeMode` and `showEditor`. Conversion options, file lists, and SVG cache are intentionally excluded.

## Rationale

Selective persistence avoids two failure modes: surprising users with stale file lists or cached SVGs from a previous session, and silently losing UI preferences (theme, editor panel) that are cheap to restore. The dual-layer model lets each file remember its last-used settings without any additional UI — switching files is enough.

## Consequences

- **Positive:** Each file remembers its settings within a session. Restarting the app always starts with a clean slate.
- **Negative:** Per-file settings are lost on restart. Users who want to preserve a particular options set must use Presets instead.

**Code:** `src/renderer/store/useAppStore.ts` — `fileOptionsByPath`, `updateConversionOptions`, `partialize`
