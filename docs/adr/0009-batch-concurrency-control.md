---
layout: default
title: "ADR 0009: Batch Concurrency Control"
---

# ADR 0009: Batch conversion concurrency model

> **Decision:** Groups-of-5 (`Promise.all` within each chunk, sequential across chunks). **Why:** A worker pool is over-engineering for batches of tens of files; fully parallel explodes memory on accidental large inputs. Groups-of-5 keeps memory bounded and the failure mode "slow and cancellable" instead of "dead."

## Context

Batch export lets the user convert many PNGs to SVG in one run. The question was how to schedule those conversions: sequentially, fully in parallel, via a worker pool, or in bounded groups.

Each individual conversion is CPU- and memory-heavy. Sharp decodes the PNG into a raw buffer; Potrace runs path-finding across that buffer. A 2 MB input can balloon to a much larger in-memory representation during tracing, and multiple conversions running at once compound this.

Two design constraints shaped the answer:

1. **Target scale.** Real batches are dozens of files, not thousands. People vectorising logos and icons do not queue up 10,000 files. I decided not to design for a scale I do not serve.
2. **Defensive against accidents.** The user picks a folder via file dialog. It is entirely possible they pick the wrong one, their photo library, a screenshot archive, and suddenly the app is staring at 5,000 PNGs. The system should degrade gracefully in that case, not lock up or exhaust memory before the user realises their mistake.

## Options considered

- **Fully sequential.** Simple, but wastes available parallelism. 20 files × ~200 ms = 4 seconds of pure wait time. Bad UX for the common case.
- **Fully parallel (`Promise.all` on the whole list).** Fast for small batches. On a 5,000-file accident it would try to spawn 5,000 Sharp decoders simultaneously, memory exhaustion before the first conversion completes. Unacceptable failure mode.
- **Worker pool.** A fixed set of worker threads pulls from a shared queue. Classic answer for "large workload, limited resources." It shines at scale, thousands of small tasks with variable runtime, where keeping workers continuously busy matters.
- **Bounded parallel groups (groups-of-N).** Sequential outer loop over chunks, `Promise.all` within each chunk. Processes N files, waits for that group to complete, starts the next.

## Decision

**Groups-of-N with `batchSize = 5`.** Sequential chunks, parallel within each chunk.

```typescript
// ConversionService.ts
const batchSize = 5;
for (let i = 0; i < filePaths.length; i += batchSize) {
  const batch = filePaths.slice(i, i + batchSize);
  const batchResults = await Promise.all(
    batch.map((filePath, localIdx) => {
      const opts = Array.isArray(options) ? (options[i + localIdx] ?? {}) : options;
      return this.convertPngToSvg(filePath, opts as ConversionOptions);
    })
  );
  results.push(...batchResults);
}
```

The method accepts either a single `ConversionOptions` (uniform across files) or `ConversionOptions[]` (per-file settings), so the renderer can pass each file's individually stored options (see ADR 0007).

## Rationale

**Worker pool was the wrong tool for the scale I have.** Worker pools shine when the workload is large and the number of tasks dwarfs the number of workers, keeping workers constantly fed is worth the scheduling overhead. At my scale, groups-of-5 on a batch of 20 completes in four sequential groups with no idle workers. The extra machinery would pay for itself only at volumes I don't serve. Over-engineering for a scale I don't have is speculation tax, more code, more surface to maintain, more ways to get the concurrency semantics subtly wrong, and the simpler pattern doesn't meaningfully underperform at realistic sizes.

**The thousand-file edge case doesn't need engineering investment right now.** `batchSize = 5` is enough to keep the app from crashing with OOM *if* that edge case ever hits, not optimised for it, just survivable through it. That is the level of defence the scale deserves today. If real usage ever pushes past hundreds of files routinely, the decision re-opens; until then, "survive the accident" is the right target, not "optimise the accident."

**Groups-of-N degrades gracefully on accidental misuse.** If the user points the app at 5,000 PNGs by mistake, fully-parallel explodes immediately, thousands of Sharp decoders in flight before anyone notices. Groups-of-5 chugs: memory usage stays bounded to five concurrent conversions at any moment, the UI stays responsive, progress is visible, and the user can cancel. The failure mode is "slow and cancellable" instead of "dead." That is a deliberate blast-radius choice.

**Why 5 specifically.** Tuning constant, found empirically. Small enough to bound memory on large images; big enough that 4× parallelism is meaningful on typical batches. The number is named (`batchSize`), not inlined, so future tuning is a one-line change.

## Consequences

**Positive**
- Bounded memory regardless of batch size. The system does not fall over on accidental large inputs.
- Meaningful parallelism within each group, 4–5× speedup over purely sequential on the common case.
- One tuning constant (`batchSize`), one code path. Easy to reason about and easy to change.

**Negative**
- Groups-of-N has a serialisation cost *between* groups, the slowest file in a group holds up the next group's start. A worker pool would avoid this bubble. For batches of tens of files it is an imperceptible loss; at thousands it would be noticeable. If the tool ever serves that scale, this is the decision to revisit.
- Per-group tail-latency is bounded by the slowest file in the group, not by the average. One very large image among four small ones slows that group down disproportionately. Known trade-off; accepted.

## Follow-up

- Cancellation between groups already works (the outer loop checks for cancel flags between chunks). Cancellation mid-group would require more plumbing and hasn't been needed so far.
- If batch sizes ever grow to thousands routinely (not via accident, via real use), reconsider the worker pool. The decision here is scoped to *current* target scale, not permanent.

**Code:** `src/main/services/ConversionService.ts`, `batchConvert()` · `src/main/main.ts`, IPC handler
