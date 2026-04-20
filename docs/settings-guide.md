---
layout: default
title: Settings Guide
---

# SVG conversion settings guide

Full parameter reference for Vektory's tracing and export options.

---

## Tracing Parameters

### Black and White Threshold (Range: 0.0–1.0, Default: 0.5)

**Purpose**: Controls which pixels are considered black vs white during tracing.

- Lower values (0.0–0.3): More pixels become black — thicker lines and filled areas
- Higher values (0.7–1.0): Fewer pixels become black — thinner lines, more white space
- **Best for**:
  - `0.1–0.3`: Dark images, logos with thick strokes, high-contrast graphics
  - `0.4–0.6`: Balanced images, general-purpose conversion
  - `0.7–0.9`: Light images, fine line art, detailed illustrations

### Corner Handling Policy (Default: minority)

**Purpose**: Determines how the algorithm handles corner decisions during path tracing.

| Value | Behaviour |
|-------|-----------|
| `minority` | Turns toward fewer black pixels — most images (default) |
| `majority` | Turns toward more black pixels — thick, bold elements |
| `black` | Always turns to keep black pixels on the left |
| `white` | Always turns to keep white pixels on the left |
| `left` | Always turns left at corners |
| `right` | Always turns right at corners |

### Minimum Artifact Area (Range: 0–10, Default: 2)

**Purpose**: Filters out small speckles, noise, and unwanted artifacts.

- Lower values (0–2): Preserves fine details, keeps small elements
- Higher values (6–10): Removes more noise, creates cleaner results
- **Best for**:
  - `0–1`: Detailed artwork, preserving fine textures
  - `2–4`: General use, balanced noise removal
  - `5–10`: Clean logos, removing background noise

### Corner Smoothness (Range: 0.0–2.0, Default: 1.0)

**Purpose**: Controls the smoothness of generated paths.

- Lower values (0.0–0.5): Angular, geometric shapes, sharp corners
- Higher values (1.5–2.0): Very smooth, flowing curves, organic shapes
- **Best for**:
  - `0.0–0.5`: Technical drawings, geometric designs
  - `0.6–1.0`: Balanced, general purpose
  - `1.1–2.0`: Organic shapes, flowing designs, artistic illustrations

---

## Optimization Settings

### Path Simplification Tolerance (Range: 0.0–1.0, Default: 0.2)

**Purpose**: Controls how aggressively the SVG paths are simplified.

- Lower values (0.0–0.2): More accurate curves, larger files, more path points
- Higher values (0.6–1.0): Simpler curves, smaller files, fewer path points
- **Best for**:
  - `0.0–0.1`: High precision needed, complex artwork
  - `0.2–0.4`: Balanced, general use
  - `0.5–1.0`: Simple designs, web optimization

### Smooth Paths After Tracing (Checkbox, Default: Enabled)

Enables advanced curve optimization. Leave enabled for most cases; disable only when processing speed matters more than output quality.

### Optimize Curve Fitting (Checkbox, Default: Enabled)

Applies additional curve optimization to reduce file size. Leave enabled for web use; disable when maximum accuracy is required.

---

## Color and Background Options

### Fill Color (Color Picker, Default: #000000)

Sets the color of all traced SVG paths. Use brand colors for logos or any custom color scheme.

### Background Color (Color Picker, Default: Transparent)

- **Transparent**: No background (best for web overlays, logos, icons)
- **Solid Color**: Fills the SVG area with the chosen color (best for print or specific design requirements)

---

## Preset Configurations

| Preset | Threshold | Corner Policy | Artifact Area | Smoothness | Tolerance | Best for |
|--------|-----------|---------------|---------------|------------|-----------|----------|
| Simple | 0.5 | minority | 4 | 1.0 | 0.3 | Logos, simple graphics, clean designs |
| Detailed | 0.3 | minority | 1 | 0.8 | 0.1 | Complex artwork, fine detail preservation |
| Smooth | 0.6 | majority | 8 | 1.2 | 0.4 | Organic shapes, flowing designs |

---

## Path Editor

The built-in path editor lets you inspect and modify the raw SVG path data after tracing. Use it to fine-tune areas the algorithm got wrong, correct small artifacts, or learn SVG path syntax.
