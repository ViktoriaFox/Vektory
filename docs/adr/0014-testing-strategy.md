---
layout: default
title: "ADR 0014: Risk-based testing strategy"
---

# ADR 0014: Risk-based testing strategy

> **Decision:** Automate the deterministic and security-critical surfaces (conversion pipeline, SVG sanitisation, file path safety); validate visual UX manually. **Why:** Line coverage and bug coverage are weakly correlated — over-testing the visual layer produces ceremony tests that catch shape changes, not behaviour.

## Context

The test suite expresses what *kind* of failures the architect is willing to ship, and which kinds they want to be told about before they reach a user. For Vektory, those choices were made deliberately, shaped by the part of my background most relevant to this question: I have spent years in QA before pivoting to architecture.

A monochrome PNG-to-SVG converter has three surfaces where bugs are expensive and detection delays them:

1. **The conversion pipeline.** Sharp preprocessing + Potrace tracing + `pathBounds` geometry. Pure deterministic transforms; output should be byte-stable for a given input. A regression here ships *visibly broken SVGs*, the entire product output is wrong.
2. **The security boundary.** SVG sanitisation against XSS payloads (ADR 0002), file path validation against traversal/null-byte/DoS attacks. A regression here is a security failure with privileged-process consequences.
3. **The visual UX.** Gradients, spacing, slider feel, drag-and-drop, dark/light theme integrity, batch progress. None of this is unit-testable in any meaningful way without ceremony tests that catch shape changes rather than behaviour. Visual UX is validated *by looking at it*. And let's be honest, sometimes something mathematically completely in the middle, but for a human eye - it looks misaligned, just because of the shape of the logo.

These three surfaces want three different testing strategies. A uniform approach (e.g. "everything has unit tests") would over-test the third and under-test the first.

## Options considered

- **No automated tests, manual only.** Defensible at v1 prototyping speed; indefensible past the point where the conversion pipeline has accumulated three independent bug fixes whose root causes are documented in ADR 0010 and ADR 0012. Without automated regression coverage, "I fixed this once" does not stay fixed. Rejected.
- **Coverage-driven testing (chase a number).** Write tests until line coverage hits some threshold. Standard textbook advice. Produces a lot of low-value tests around getters, framework calls, and trivial utilities, while still missing the bugs that actually ship, which is the QA-known-truth that line coverage and bug coverage are weakly correlated. Rejected.
- **Full pyramid (unit + integration + E2E + visual regression).** Aspirational. Requires CI infrastructure, snapshot baselines, cross-platform test matrices, headless Electron runners. Justified for a team-maintained product; over-investment for a solo v1 portfolio project where one person validates the full app manually before each release. Rejected at this scope.
- **Risk-based: automate the deterministic and security-critical, manually validate the visual.** Unit/integration tests on conversion math, image pipeline, sanitisation, file path safety. Manual testing for UI, drag-and-drop, theme, real-image variety, cross-OS install. **Chosen.**

## Decision

**Vitest** as the test framework. **Three test suites** covering the surfaces where automation pays the most:

1. **`ConversionService.test.ts`**: pure-function unit tests for `pathBounds` plus integration tests using real Sharp and real Potrace against generated test images.
2. **`svgSanitizer.test.ts`**: XSS attack-payload tests in a `jsdom` environment.
3. **`FileService.test.ts`**: security-pattern tests against path traversal, null-byte injection, and DoS-by-length, plus directory-sandboxing logic.

Plus `format.test.ts` for trivial utility coverage. Total: 56 test cases.

Coverage instrumentation is configured to report only on these three modules (`ConversionService.ts`, `svgSanitizer.ts`, `format.ts`).

Manual testing is the strategy for everything else: UI flows, drag-and-drop, theme switching, batch UX feel, real-image variety beyond the synthetic test fixtures, cross-platform installation behaviour.

## Rationale

### Why risk-based, not coverage-based

The QA-trained instinct is that bugs do not distribute uniformly across a codebase. They cluster at boundaries, at parameter handling, at I/O, at concurrency, at security surfaces, and they avoid getters, framework wiring, and pure rendering of static state. A test budget spent on the second category produces test files that turn green on every commit and catch nothing. The same budget spent on the first category catches real regressions.

For Vektory, the bug-prone surfaces are easy to name because they are documented in other ADRs. The Sharp preprocessing pipeline order (ADR 0012) was *empirically reverse-engineered* from three independent failure modes; a regression that re-introduces one of those bugs would silently produce wrong output. The cubic-Bezier extrema math (ADR 0010) replaced a convex-hull approximation that over-estimated viewBoxes; a regression to convex hull would silently re-introduce the wrong-icon-size bug. The renderer SVG sanitiser is the second defensive layer of ADR 0002's threat model; a regression here is a security failure.

Each of those is exactly the kind of bug that re-occurs without a regression test. So that is where the tests are.

### Tests as regression contracts on architectural decisions

The test suite is structured around the ADRs:

- `pathBounds`'s arch-bezier test has the inline comment *"the key regression test"*, it locks in ADR 0010's tight-bounding-box behaviour against a future refactor that would naively re-implement convex hull.
- `ConversionService`'s `revertColors: true` test is annotated *"regression for the negate bug"*, it locks in ADR 0012's pipeline order against accidental reordering.
- `svgSanitizer`'s tests use real attack payloads (`<script>alert(1)</script>`, `onload="fetch(evil)"`), they verify the second defensive layer of ADR 0002's defense-in-depth model.
- `FileService`'s tests use real attack patterns (path traversal `../../etc/passwd`, null byte `\0`, length DoS), they verify the file-side hardening that complements ADR 0002.

Essentially: **the ADRs say what the architecture decided, the tests prove it stays decided**.

### Why integration tests use real Sharp and real Potrace

The integration tests in `ConversionService.test.ts` generate real PNG fixtures with Sharp itself (a 50×50 white square with a 20×20 black centre, and the inverse) and feed them through the actual pipeline. No mocking of Sharp; no mocking of Potrace.

The QA-known reason: when the bugs being defended against are bugs *in how those libraries compose* (alpha behaviour, threshold semantics, byte-perfect input shape), mocked tests verify the mock, not the real behaviour.

### Why Vitest

The project is already Vite-based for the renderer (ADR 0004). Vitest reuses the same config, the same module resolver, the same TypeScript setup, the same alias paths etc.
For a single-developer codebase already on Vite, Vitest is the path of least configuration that gives the most actual testing capability.

### Why no E2E, no visual regression, no CI matrix, for now

Each of these has a real future-cost trade-off:

- **E2E (Playwright + Electron).** Would test full flows: drop a folder, see preview, adjust slider, export batch. Real value. Cost: spawning headless Electron, fixture management, CI runner that supports it. For a solo v1 release validated by manual end-to-end testing, the marginal value over manual testing is small. Held until the codebase has multiple maintainers or release cadence makes manual testing of every flow infeasible.
- **Visual regression on SVG output.** Compare exported SVG (or rendered SVG-to-PNG) against a baseline. Would catch any geometry regression that current tests miss. Cost: baseline management, deterministic rendering pipeline, infrastructure (Argos / Percy / homegrown). Held; current `pathBounds` regression tests cover the math, and integration tests cover the pipeline shape, which catches most realistic regressions.
- **Cross-platform CI matrix.** Run tests on Win + macOS in CI. Sharp's prebuilt-per-platform behaviour (ADR 0003) makes this useful. Cost: GitHub Actions matrix configuration and runner time. Likely the next addition; manual install validation has covered it for v1.

## Consequences

**Positive**
- The deterministic-output guarantees that ADR 0010 and ADR 0012 depend on are protected by named regression tests.
- The XSS and path-traversal threat models from ADR 0002 are protected by tests using real attack payloads.
- Vitest reuses Vite tooling, testing has zero infrastructure overhead from the build pipeline's perspective.
- The test suite is small enough to run fast (~10 seconds locally) yet meaningful enough to catch regressions in the highest-cost places.
- Manual testing is named as the strategy for visual UX rather than left as an unstated default, this is honest scoping.

**Negative**
- **No automated coverage of UI flows.** Acceptable for solo v1, would be a real gap for a multi-developer release cadence.
- **No visual regression on rendered SVG.**
- **No cross-platform CI.** Sharp's per-platform prebuilds (see ADR 0003 follow-up about `electron-rebuild`) are validated by manual install testing.
- **Integration tests depend on Sharp's runtime correctness.** If Sharp itself ships a bug, our tests against real Sharp would happily pass against the broken behaviour.

## Follow-up

- **Visual regression on SVG output** is the most valuable next addition. Would lock in the *visual* behaviour of `cubicExtrema` + `pathBounds` together, not just the math.
- **Cross-platform CI** (Windows + macOS via GitHub Actions matrix) is the second priority, closes the "Sharp prebuild on a fresh build environment" gap that ADR 0003 flags as a known operational sharp edge.
- **E2E coverage** (Playwright + Electron) is the third priority, justified when manual testing of full flows on every release becomes the slow link in the release process.
- The coverage allowlist in `vitest.config.ts` should grow with the architecture: when a new module is added that is high-risk (deterministic output, security-adjacent, or pipeline-critical), it joins the coverage list. Renderer state, framework wiring, and trivial utilities stay off the list deliberately.

**Code:** `vitest.config.ts` · `src/main/services/ConversionService.test.ts` · `src/main/services/FileService.test.ts` · `src/renderer/utils/svgSanitizer.test.ts` · `src/renderer/utils/format.test.ts`
