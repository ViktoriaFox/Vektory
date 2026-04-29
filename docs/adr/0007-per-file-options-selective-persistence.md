---
layout: default
title: "ADR 0007: Per-File Options with Selective Persistence"
---

# ADR 0007: Per-file conversion options with selective persistence

## Context

The app started single-file: one image, one set of conversion options (threshold, corner policy, artifact area, smoothness, path optimisation), one preview. The store held a single `conversionOptions` object bound to the UI controls, and that was enough.

While implementing the batch flow I hit the bug during testing: selecting a file in the list, adjusting its threshold, switching to another file, and finding the second file's threshold had moved too. The single shared `conversionOptions` was being applied across every file in the batch. That is obviously wrong: different images need different settings (a sharp-edged logo wants a different threshold from a soft photograph), and a batch with one shared control makes per-file tuning impossible.

Fixing this surfaced a second, separate question: **what persists across restarts?** Conversion options, the file list, the SVG cache, these are context that belongs to a specific working session. Restoring them would likely be confusing or wrong (stale files, cached SVGs from conversions the user doesn't remember). But *some* states are user-level: the theme mode, the code-editor panel visibility. Forcing the user to re-set those every launch is friction with no purpose.

Two distinct requirements, one store.

## Options considered

**For the options model:**
- *Single global options.* What was there already. Broken by batch, per-file tuning impossible.
- *Per-file only, no global defaults.* Each file owns its own options. Clean, but awkward on new files: every newly-added image starts from hard-coded defaults rather than "whatever the user was just working with."
- *Dual-layer: global defaults + per-file overrides.* Global `conversionOptions` follows the UI controls; `fileOptionsByPath: Record<string, ConversionOptions>` stores per-file overrides as the user tunes them. On file switch, load stored options or fall back to the global defaults.

**For persistence scope:**
- *Persist nothing.* Loses theme mode on restart, unnecessary friction for a preference the user sets once.
- *Persist everything.* Restores file lists, converted SVGs, and options from the previous session. Stale data that confuses more than it helps.
- *Persist selected fields.* Pick a small allowlist of user-level preferences; let session state be session-scoped.

## Decision

- **Dual-layer options model.** Global `conversionOptions` bound to the UI. Per-file overrides in `fileOptionsByPath`, keyed by absolute path. On file switch, the renderer loads `fileOptionsByPath[file.path]` if present, otherwise current global `conversionOptions`.
- **Zustand `persist` middleware with `partialize` allowlist: `themeMode` and `showEditor` only.** Everything else, `conversionOptions`, `fileOptionsByPath`, file list, SVG cache, is session-scoped and rebuilt fresh on each launch.

## Rationale

**Why dual-layer instead of per-file only.** The global layer is *inheritance root*. When the user drags in a new file, they expect it to start from "where I was just working," not from hard-coded defaults. Dual-layer gives that behaviour: the global state mirrors the UI controls, a new file inherits the current global, and once the user adjusts anything for that specific file, the override is recorded in `fileOptionsByPath` and the file owns its own settings from then on. No extra UI needed, switching files is the interaction.

**Why these two fields persist, and not others.** The allowlist is short:
- `themeMode`, a user-level preference, not session state. The user sets it once and expects it to stick. Re-selecting dark mode every launch would be pure friction.
- `showEditor`, workspace layout. Whether the SVG code panel is open is the same kind of preference as theme: once chosen, it should persist.

Everything else is session-scoped by design:
- `conversionOptions` and `fileOptionsByPath`, bound to the files in the current session. Restoring them would mean restoring settings for files that may no longer exist, or misapplying them to a fresh batch.
- File list, restoring it would show the user a list of files from a previous working context, possibly moved, renamed, or irrelevant now.
- SVG cache, regenerable from source + options. Persisting it would add disk overhead for no functional benefit.

A user who wants to keep a particular options set across sessions uses Presets instead, that is a separate, explicit mechanism, and the right tool for durable configuration.

## Consequences

**Positive**
- Per-file tuning works correctly. Switching between files in a batch preserves each file's individual settings with no extra UI.
- Fresh launches start clean: no stale file lists, no mismatched cached conversions, no options pointing at files that have moved.
- The preference fields that matter for user identity (theme, editor visibility) persist reliably.

**Negative**
- Per-file settings are lost on restart. A user who carefully tuned five files and closed the app will re-tune them on reopen. Documented behaviour; Presets are the durable alternative.
- The allowlist is a maintenance point: any new preference that should persist must be added to `partialize` explicitly. Easy to miss when adding a new UI toggle, mitigated by keeping the allowlist short and adjacent to the state definition so it's visible when the shape changes.

## Follow-up

- If the allowlist grows beyond a handful of fields, consider a naming convention (e.g. prefix user-level preferences) rather than a flat allowlist, to make the persist boundary self-documenting.
- Presets are the escape hatch for users who need durable conversion configurations. If that feature sees heavy use, it is evidence that more session-level state deserves to persist, re-evaluate at that point.

**Code:** `src/renderer/store/useAppStore.ts`, `fileOptionsByPath`, `updateConversionOptions`, `partialize`
