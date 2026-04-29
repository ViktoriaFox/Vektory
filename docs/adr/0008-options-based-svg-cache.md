---
layout: default
title: "ADR 0008: Options-Based SVG Cache Invalidation"
---

# ADR 0008: Options-based SVG cache invalidation

## Context

Converting a PNG with Sharp + Potrace is CPU-bound and takes 100–500 ms per file depending on size and complexity. When a user switches between files in a batch, re-running the full conversion even when nothing changed was wasteful and made the UI feel slow.

## Options considered

- **No cache.** Re-run conversion on every file switch. Correct, simple, and slow, 100–500 ms latency every time the user clicks a previously-tuned file. Rejected on UX grounds.
- **LRU-bounded cache with explicit eviction.** Standard solution for unbounded memory growth. Useful at scale; in this app, bounded by session length and number of files in the working batch (tens, not thousands). Over-engineered for the actual cache size.
- **Hash-based invalidation key.** Hash `ConversionOptions` into a fixed-length key, store cache keyed by `(filePath, optionsHash)`. Robust against object-key-order issues but adds a hashing step on every file switch and complicates debugging when comparing keys by eye.
- **Equality on the options object via `JSON.stringify`.** Compare serialised forms of the cached options vs current options. Simple, deterministic for the shapes I actually have, debuggable.

## Decision

Cache each file's SVG alongside the exact `ConversionOptions` used to generate it. On file switch, compare the cached options against the current options using `JSON.stringify` equality. Only trigger re-conversion when the cache is empty (`null`) or the options have changed.

```typescript
// useAppStore.ts
svgCache: Record<string, { options: ConversionOptions; svg: string }>;

// on file switch:
const cached = svgCache[file.path];
const optsMatch = cached && JSON.stringify(cached.options) === JSON.stringify(fileOptions);
const currentSvg = optsMatch ? cached.svg : null;

// useApp.ts, only convert if currentSvg is null:
if (currentFile && currentSvg === null) triggerConversion();
```

## Rationale

`JSON.stringify` comparison is simple and deterministic for flat option objects constructed from the same shape. More sophisticated strategies (structural diffing, hashing) would add complexity for no practical benefit at this scale. The cache is intentionally not persisted, it is rebuilt as needed within a session.

### Cache lives in the renderer Zustand store, not in main

This is a deliberate architectural placement. The cache is tied to *UI* concerns, which file is currently selected, which options the UI is showing, when to skip re-conversion before issuing an IPC call. Moving the cache into the main process would mean every file switch still pays one round-trip across the IPC boundary just to ask *"do you have this cached?"*, a question the renderer can answer locally for free. Keeping the cache in the renderer means the UX-critical fast path (instant switch between two previously-tuned files) never crosses process boundaries.

Main is intentionally stateless about UI: it converts on request, returns SVG, doesn't remember. That separation falls out of the security boundary in [ADR 0002](0002-context-isolation-and-preload-api), main is the privileged process and gets less mutable state by design, not more. The cache is renderer-scoped session state, in the same store family as `fileOptionsByPath` and `currentFile` ([ADR 0007](0007-per-file-options-selective-persistence)).

## Consequences

**Positive**
- Switching between previously converted files is instant with no Potrace call and no IPC round-trip.
- Cache logic is co-located with the UI state that drives it, in one Zustand store, easy to reason about.

**Negative**
- Object key order must be consistent for `JSON.stringify` equality to hold. Options objects are always constructed from the same shape, making this safe in practice, but it is an invariant that depends on construction discipline.
- **Silent stale on `ConversionOptions` shape changes.** If a I would add a field to `ConversionOptions` (a new tuning parameter, an experimental flag), every existing cache entry is structurally different from new ones. The serialised form changes, equality fails, every file re-converts on next switch. This is *correct* behaviour, the conversion would actually produce different output once the field is wired into the pipeline, but: there is no warning that the entire session-level cache just invalidated. For a session-scoped cache rebuilt within seconds of use, this is acceptable; if the cache ever gains persistence (cross-session, on-disk), shape changes would need a versioned key to avoid silently re-converting hundreds of files on first launch after an update.
- **External-edit invalidation gap.** The cache key is `(filePath, options)`. The file *content* is not part of the key. If the user opens a PNG in Vektory, then edits the same file in an external tool (Photoshop, GIMP, in-place save from another image editor) while it is still in Vektory's working set, the path is unchanged and the options are unchanged, so on the next file switch the cache returns the SVG generated from the *previous* PNG content. The stale result looks correct (no error, no warning) until the user notices that the preview doesn't match what is on disk.
Today's workaround is to re-add the file to the working set, which forces a fresh conversion. The proper fix is to include the file's `mtime` (modification timestamp) in the cache key, one `fs.stat` per file switch, effectively zero cost, and it is the standard signal desktop tools use for exactly this case. A content hash would also work but is heavier; `mtime` is the right level of investment.

## Follow-up

- **Add `mtime` to the cache key.** Closes the external-edit gap with negligible runtime cost. Tracked as a known limitation until shipped.
- If the cache ever needs to span sessions (persisted to disk), introduce a versioned key incorporating a hash of the `ConversionOptions` shape, so that a release that changes the shape doesn't silently re-convert every file on first launch.
- A content-hash variant is the heavier alternative to `mtime`. Worth revisiting only if `mtime` proves unreliable on a particular platform or filesystem (e.g. some networked filesystems with weak timestamp guarantees).

**Code:** `src/renderer/store/useAppStore.ts` · `src/renderer/hooks/useApp.ts`
