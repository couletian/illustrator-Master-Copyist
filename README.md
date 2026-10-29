**English** | [简体中文](README.zh-CN.md)
<img width="4267" height="4267" alt="replica-studio-icon" src="https://github.com/user-attachments/assets/ca2dad72-9339-49a4-9db8-60a00f5d9dca" />

# illustrator Master-Copyist

> **Recreate any image in Illustrator with a single AI prompt.**
>
> Possibly the most powerful open-source, free, Agent-driven Adobe Illustrator recreation Skill available today.

**illustrator Master-Copyist** is a "raster → editable vector" reconstruction toolkit for scientific schematics. It breaks a PNG/JPG into **an editable live-text layer, an independent line-art layer, an independent color/geometry layer, and a deletable background layer** — all as native Adobe Illustrator objects. Not outlined text, not bitmaps, not white boxes covering lettering.

---

## Table of Contents

- [What It Is](#what-it-is)
- [What It Produces](#what-it-produces)
- [What It Can Do](#what-it-can-do)
- [Core Features](#core-features)
- [Environment Dependencies](#environment-dependencies)
- [Quick Start](#quick-start)
- [Workflow and Principles](#workflow-and-principles)
  - [Route A: Raster Reconstruction](#route-a-raster-reconstruction)
  - [Route B: Line-Art Reconstruction (AniLines One-Stroke Geometry)](#route-b-line-art-reconstruction-anilines-one-stroke-geometry)
  - [Fill Bounds and Background (Mandatory)](#fill-bounds-and-background-mandatory)
  - [Delivery and Acceptance](#delivery-and-acceptance)
- [Toolkit](#toolkit)
- [Self-Iteration Mechanism](#self-iteration-mechanism)
- [Measured Data and Performance](#measured-data-and-performance)
- [Known Limits and Disclosures](#known-limits-and-disclosures)
- [Directory Structure](#directory-structure)
- [Troubleshooting Quick Reference](#troubleshooting-quick-reference)
- [Acknowledgments](#acknowledgments)
- [License](#license)

---

## What It Is

`illustrator Master-Copyist` is a **Skill that Agents can load directly**, supporting Codex, Claude Code, and OpenCode. It does not depend on any cloud vectorization API — all core scripts run locally. It bundles its own Python runtime, VTracer executable, and AniLines line-art model weights, and communicates natively with Adobe Illustrator through a shared Windows/macOS CLI.

The core problem it solves:

> **Scientific figures cannot merely "look like" the original — they must remain editable.**

Traditional vectorization tools output dead paths: text becomes outlines, layers collapse into one, and the background is baked into the artwork. When a journal asks for a revision, you cannot change a single word. This Skill's approach is **"separate text from graphics"**:

1. Transcribe every label into a structured manifest (real font, size, baseline, color, anchor);
2. Erase only the text from a copy, preserving the artwork;
3. Faithfully trace the remaining artwork;
4. Inject real `<text>` back at the original baselines;
5. Generate native TextFrames + vector paths in Illustrator;
6. Finally overlay an independent `AniLines lineart` layer, plus line-art-derived matte, transparent background, and a deletable background layer.

It is explicitly **not** intended for: recovering experimental data via tracing, redrawing statistical plots as schematics, or applying this workflow indiscriminately to every uploaded image. Statistical plots should use original vector exports and real data; tracing and color quantization cannot recover numeric measurements.

---

## What It Produces

The delivered `.ai` document's layer structure (top to bottom):

| Layer | Content | Description |
| --- | --- | --- |
| `AniLines lineart` | Independent one-stroke line-art layer | AniLines neural network extraction + glyph-level text filtering + one-stroke geometry assembly; individually toggleable/recolorable/deletable |
| `Editable text` | Independent live-text layer | Each label is a native TextFrame; font, size, color, baseline all editable |
| `图层 1` (trace geometry) | Independent color/geometry layer | VTracer faithful-traced vector paths; often merged into one GroupItem after SVG import, but `pathItems` can still count all contours |
| `Background` | Deletable background layer | A single rectangle in the source page color; hiding or deleting it leaves the artboard fully transparent |

Also produced:

- `figure.ai` — Native Illustrator document
- `figure.svg` — Structured SVG (paths + real `<text>`, no bitmaps/external links/scripts)
- `figure.png` — White-background preview (200% export)
- `figure_nobg.png` — Transparent-background preview
- `qa.json` / `candidate_comparison.csv` / `playback.json` — Audit records
- `lineart_strokes.json` — One-stroke line-art geometry, scriptable redraw in Illustrator
- `text_manifest.json` — Per-label text specification

**Built-in matting capability**: `fill_matte.py` generates an alpha matte from line-art geometry; `vectorize_cut.py` preserves alpha and feeds RGBA directly to VTracer, so transparent areas produce no paths and the page background disappears by construction. `verify_fill_bounds.py` checks spill, background color delta, corner alpha, and page-color residue; `visual_check_bg.py` performs two-background visual acceptance on black and magenta.

---

## What It Can Do

1. **Convert AI-generated images into vector graphics** for secondary editing, layout, and journal submission.
2. **Convert publicly available images into background-free vector graphics** for material collection, tracing, and structural reference (mind the original image copyright).
3. **Recreate paper/review mechanism figures**: multi-panel layouts, arrows, cells, receptors, pathways, color-chip labels, badge numbers — all as editable objects.
4. **Rebuild existing SVG**: validate, prepare, inject live text, import natively, preserving the original vector structure.
5. **Draw new schematics from scratch**: given explicit content, write structured SVG directly (paths + basic shapes + real `<text>`, arrowheads as polygons), then run native drawing.
6. **Deliver line art independently**: extract one-stroke line art with AniLines, overlay it as an independent layer onto an existing `figure.ai` without damaging the original artwork.
7. **Transparent/alternate-background delivery**: line-art-derived matte + deletable Background layer, so the figure can be keyed onto any background.
8. **Batch/resumable drawing**: the scheduler writes in batches, periodically saves a job-owned `figure.ai`, exports PNG, and resumes by full path after interruption.

---

## Core Features

### 1. Complete Separation of Text and Graphics

- Labels go through the manifest / live-text path; letters are never outlined and covered with white boxes.
- Line-art text removal uses **glyph-level connected-component filtering** (`lineart_text_filter.py`): only components whose bounding box falls entirely inside a text ink box are deleted; leader lines, borders, and cell outlines crossing the box are always kept. **Flat-box erasure is never used** — it would damage graphics that cross the text box.
- The erasure stage has a **residual-glyph loop**: erase → scan leftover glyph clusters → merge boxes / re-place → re-erase → rescan, until only graphic fragments remain. Five rounds of testing cut 03_1 from 11 iterations to 3.

### 2. Line Art Is the Sole Authority for Fill Bounds

- **Background is never determined by color.** The page may be a gradient, photographic backdrop, beige, or pink; sampling page color is merely a per-image convenience, never the rule.
- `fill_matte.py` binarizes the line art → directional closing seals 1–3 px hairline gaps → floods the exterior from the canvas border → writes RGBA with alpha=0 outside the line-art-bounded figure.
- Directional closing is critical: an isotropic kernel large enough to seal frame corners also seals the 5–10 px gutters between panels, turning the whole page opaque.
- `restore_ink_alpha.py` is mandatory: dashed dividers, dashed annotation boxes, and open contours cannot form closed loops, so flood classifies them as exterior and they disappear entirely; this script writes alpha=255 back at ink locations based on the text-free line art. 033 testing recovered 52,853 px.

### 3. Native TextFrames, Not Outlined Text

- Each label is a native Illustrator text frame; font, size, color, baseline, anchor all editable.
- Letter spacing is always 0. Negative letter-spacing causes Illustrator to double-render the same text frame via both the JSX runtime and SVG import (double ghosting). Width mismatches are fixed by rescaling font size as `fs × target width / rendered width`, never by negative tracking.
- Font selection distinguishes regular/bold from Black/Light; missing fonts stop the preparation stage rather than writing the wrong font into the document.
- Metrics use PIL-measured values (including kerning), not fontTools pen accumulation (which excludes kerning and is systematically 7–8% too narrow).

### 4. Independent Line-Art Layer (AniLines)

- Bundles AniLines (MIT, gyrojeff/AniLines) basic/detail dual models, 69 MB weights each, vendored in `third_party/anilines/`.
- Default `--scale 2` detail mode: input is Lanczos-upscaled 2× before inference so thin lines are not smeared away. Measured: stroke count 778 → 1400, closed loops 97 → 286, median width 2.00 → 1.41 px.
- One-stroke geometry assembly: skeletonization → Rutovitz crossing-number node detection → node clustering → chain extraction → tangent-continuous merging; loops stay `closed=true`, guaranteeing "one circle, one stroke" by construction.
- Ink color correction: the skeleton often sits 1–2 px off the ink, so each centerline point takes the darkest pixel in a 5×5 (fallback 9×9) window, and the per-stroke ink is the median of those minima; strokes whose darkest pixel is still ≥250 are treated as inkless and skipped rather than painted in page color (avoiding white lines that vanish on white and glare on dark backgrounds).

### 5. Alpha-Preserving Tracing + Deletable Background

- `vectorize_cut.py` feeds RGBA directly to the pinned VTracer CLI with parameters identical to the faithful profile (4× LANCZOS, spline, speckle 4, precision 8, gradient-step 4, simplify 0.6, 128 colors), then rescales back with `scale(0.25)` and restores the viewBox. Transparent areas produce no paths.
- `import_to_illustrator.py --bg-color` creates a deletable `Background` layer at the bottom; `--png-nobg` exports a separate transparent-background PNG.
- `verify_fill_bounds.py` four checks + `visual_check_bg.py` two-background visual check — neither replaces the other.

### 6. Self-Iteration Mechanism

At the end of every task round, high-value reusable scripts are automatically reviewed, categorized into `toolkit/`, and `toolkit/调用指引手册.md` is updated. The more you use it, the stronger, faster, and cheaper it becomes.

### 7. Fully Local, No Vectorization API Key

Core scripts do not upload images or require a vectorization API key. Dependency installation is a separate explicit step, not a per-image download. Model and Adobe access remain the user's responsibility; showing an image to a cloud model is not fully offline.

---

## Environment Dependencies

### Runtime (Bundled)

| Item | Description |
| --- | --- |
| Python | **Skill's own `.venv`**: portable CPython 3.13.14, full-install layout, no external Python needed, usable from any working directory |
| Windows interpreter | `.venv/Scripts/python.exe` |
| macOS interpreter | `.venv/bin/python` |
| CLI entry point | `skill_dir/scripts/figure_cli.py` |
| Invocation prefix | Add `-X utf8` to all commands; quote paths containing spaces; prefix quoted paths with `&` in PowerShell |

> Release packages usually bundle `.venv` directly. If distributing from source without it, run `python -X utf8 scripts/setup_environment.py --with-vtracer` once to create `.venv` and install per `requirements.lock`. **Do not** pip-install into `.venv`, and do not rely on system Python; if a package is missing on a new machine, copy it along with its `.dist-info` from another machine's `.venv/Lib/site-packages`.

### Python Package Layers

| Layer | Packages | Used by |
| --- | --- | --- |
| Core | Pillow, numpy, defusedxml, fontTools, opencv-python-headless (cv2), pip | figure_cli / figurelib / toolkit stages 01–05 |
| Line-art (AniLines) | **torch 2.14.1+cpu**, scipy, scikit-image, networkx, sympy, mpmath, jinja2, markupsafe, fsspec, filelock, imageio, tifffile, lazy-loader, packaging, typing-extensions | `scripts/lineart_*.py` stages 1–3 (extract / text_filter / strokes) |
| Bundled non-Python | `bin/vtracer.exe` (VTracer 1.0.0-alpha.4, pinned SHA256), `third_party/anilines/` (code + dual 69 MB weights) | Vectorization backend, AniLines model |
| External (user side) | Adobe Illustrator 2026 + COM bridge (PowerShell 5.1) / macOS AppleScript | `lineart_overlay.py native`, native drawing, delivery |

### Adobe Compatibility

| Environment | Implementation | Evidence |
| --- | --- | --- |
| Windows + Illustrator 27.2.0 | Native paths, live text, separate text layer, AI save, PNG export, resume | Tested on hardware, including a user-visually-accepted paper figure recreation |
| Windows, other Illustrator versions | Enumerate registered COM versions, test required capabilities individually | Not tested version by version |
| macOS Intel / Apple silicon | Discover running app bundle, AppleScript + shared JSX; dual-architecture pinned VTracer | Adapter contract checks; no macOS hardware result yet |
| Linux | SVG preparation, text cleanup, tracing, QA | No native Illustrator connection |

Measured Illustrator path on this machine (2026-10-29):

```text
D:\Program Files (x86)\Adobe IIIustrator\Adobe Illustrator 2026\Support Files\Contents\Windows\Illustrator.exe
```

The directory name `IIIustrator` has **three i's** (Adobe installer's own misspelling); searching for `Illustrator` will not find it. Version 30.0.0, ProgID `Illustrator.Application`.

On macOS, the first connection may require granting Automation permission to the terminal/Agent under "System Settings → Privacy & Security → Automation"; the adapter only selects an already-running bundle and does not launch another version.

### Check Commands

```bash
<python> -X utf8 <cli> doctor
<python> -X utf8 <cli> illustrator-doctor
```

`illustrator-doctor` reports the actual version, active document, artboard coordinates, and required capabilities. For a new system/new Adobe version, first run a disposable-document test:

```bash
<python> -X utf8 <cli> native-test --create-document --out-dir <new-test-directory>
```

---

## Quick Start

### 0. Place the Skill (pick one directory per Agent)

| Agent | Project directory | Invocation |
| --- | --- | --- |
| Codex | `<project>/.agents/skills/illustrator-Master Copyist/` | `$illustrator-Master Copyist` or an explicit workflow request |
| Claude Code | `<project>/.claude/skills/illustrator-Master Copyist/` | `/illustrator-Master Copyist` |
| OpenCode | `<project>/.opencode/skills/illustrator-Master Copyist/` | Load the named skill; also supports project `.agents/skills` / `.claude/skills` |

Choose one discovery location per Agent to avoid duplicate entries. Scripts do not modify Agent settings, write hooks/MCP, or register global skills.

### 1. Environment Self-Check

```bash
<python> -X utf8 <cli> doctor
<python> -X utf8 <cli> illustrator-doctor
```

### 2. Choose an Input Route

- **New schematic**: write structured SVG (paths + basic shapes + real `<text>`, arrowheads as polygons).
- **Existing vector SVG**: directly `validate` + `prepare`.
- **Raster recreation**: follow Route A; read `references/workflow.md` and `references/input-and-validation.md` first.
- **Line-art reconstruction**: follow Route B; read `references/lineart-pipeline.md`.

### 3. Minimal Raster Recreation Command Chain

```bash
# 1) Transcription + measurement + manifest (toolkit Stage 1–2)
<python> -X utf8 toolkit/01_detect/detect_text_lines.py --image <orig.png> --out-dir <work>/lines --overlay
<python> -X utf8 toolkit/02_manifest/local_match.py --image <orig.png> --spec <work>/spec.json --out <work>/located.json --overlay <work>/locate_check.png
<python> -X utf8 toolkit/02_manifest/grow_boxes.py --image <orig.png> --in <work>/located.json --out <work>/grown.json --gap 12
<python> -X utf8 toolkit/02_manifest/fit_sizes.py --in <work>/grown.json --out <work>/text_manifest.internal.json --colors <work>/colors.json
<python> -X utf8 toolkit/02_manifest/conv_manifest.py --in <work>/text_manifest.internal.json --out <work>/text_manifest.json --image <orig.png>
<python> -X utf8 toolkit/02_manifest/verify_text_fit.py --manifest <work>/text_manifest.json --out <work>/fit_report.json --tol 2.5

# 2) Text erasure + residual-glyph loop (toolkit Stage 3)
<python> -X utf8 toolkit/03_erase/erase05.py --source <orig.png> --manifest <work>/text_manifest.json --out <work>/text_clean.png --passes 2
<python> -X utf8 toolkit/03_erase/scan_remnants.py --image <work>/text_clean.png --manifest <work>/text_manifest.json --apply
# If re-placement occurs → re-run erase05 → scan again, until only graphic fragments remain

# 3) Line-art extraction + glyph-level text filtering (skill lineart pipeline)
<python> -X utf8 scripts/lineart_extract.py <orig.png> <work>/la_raw.png --mode detail
<python> -X utf8 scripts/lineart_text_filter.py --lineart <work>/la_raw.png --text-manifest <work>/text_manifest.json --measure-from <orig.png> --out <work>/la_clean.png

# 4) Line-art matte → transparent base → ink restoration → alpha-preserving tracing
<python> -X utf8 toolkit/03_erase/fill_matte.py --source <work>/text_clean.png --lineart <work>/la_clean.png --out <work>/fig_cut.png --shrink 2
<python> -X utf8 toolkit/03_erase/restore_ink_alpha.py --cut <work>/fig_cut.png --lineart <work>/la_clean.png --out <work>/fig_cut_ink.png --report <work>/ink.json
<python> -X utf8 toolkit/04_fixup/vectorize_cut.py --image <work>/fig_cut_ink.png --out <work>/geom.svg --vtracer <skill>/bin/vtracer.exe
<python> -X utf8 toolkit/04_fixup/split_compound_paths.py --in <work>/geom.svg --out <work>/fixed_geom.svg

# 5) Inject live text
<python> -X utf8 <skill>/scripts/figure_cli.py merge-text --svg <work>/fixed_geom.svg --text-manifest <work>/text_manifest.json --output <work>/figure_master.svg

# 6) Native import delivery + live-text layering
<python> -X utf8 toolkit/05_delivery/import_to_illustrator.py --svg <work>/figure_master.svg --ai <work>/figure_final.ai --png <work>/figure_final.png --png-nobg <work>/figure_nobg.png --bg-color "#fefefe" --skill-dir "<skill directory>"
<python> -X utf8 toolkit/05_delivery/split_text_layer.py --ai <work>/figure_final.ai --out-ai <work>/figure_final.ai --out-png <work>/figure_final.png --skill-dir "<skill directory>"

# 7) Line-art overlay layer (mandatory finishing step)
<python> -X utf8 scripts/lineart_strokes.py <work>/la_clean.png <orig.png> <work>/strokes.json --scale 2 --smooth 0.8
<python> -X utf8 scripts/lineart_overlay.py native --strokes <work>/strokes.json --figure-ai <work>/figure_final.ai --out-ai <work>/final.ai --out-png <work>/final.png --viewbox 1536 1024

# 8) Acceptance
<python> -X utf8 toolkit/05_delivery/verify_fill_bounds.py --png <work>/final.png --png-nobg <work>/figure_nobg.png --matte <work>/fig_cut_ink.png --source <orig.png> --text-manifest <work>/text_manifest.json --tol 3
<python> -X utf8 toolkit/05_delivery/visual_check_bg.py --png-nobg <work>/figure_nobg.png --out-dir <work>/vis
```

> `--viewbox W H` is mandatory for non-1536×1024 artboards. `import_to_illustrator.py` must be given `--skill-dir` explicitly; otherwise the default points to the original author's machine path and `import figurelib` fails directly.

---

## Workflow and Principles

### Overview

```text
Original image
 ├─ Stage 1  Text line detection / template matching / local F1 localization
 ├─ Stage 2  Manifest: content + boxes + font size + baseline + color
 ├─ Stage 3  Text erasure ⇄ residual-glyph loop
 ├─ Stage 4  Line-art extraction → glyph-level text removal → one-stroke geometry
 ├─ Stage 5  Line-art matte → ink restoration → alpha-preserving tracing → split/fixup
 ├─ Stage 6  merge-text injects live text
 ├─ Stage 7  Native import → Background layer → live-text layering
 ├─ Stage 8  AniLines lineart independent layer overlay
 └─ Stage 9  Numeric acceptance + two-background visual acceptance
```

### Route A: Raster Reconstruction

Applies to: AI-generated images, paper screenshots, public schematic figures, multi-panel mechanism diagrams.

**Core principle: transcribe first, erase second, trace third, inject live text last.**

1. **Transcription and Measurement**
   - Reading images is only for reading **content**, never coordinates. The Read tool downscales large images, and eyeballed coordinates can be off by 20–100 px; all placement, font size, and erasure targets come from program-measured connected-component ink boxes.
   - Line detection (`detect_text_lines.py`) produces `lines.json` and 1:1 annotated crops.
   - When text is known, prefer template matching (`match_text_template.py`); for texture-dense mechanism diagrams, use local window + binary overlap F1 localization (`local_match.py`). 033 testing: 60 of 69 template matches were low-score and misaligned; `local_match` had only 3 low-score.
   - Localization boxes must go through `grow_boxes.py`: template outer boxes are 6–12% narrower than the source ink, and failing to grow leaves half a letter at each end.
   - Multi-character labels fit font size by **width** (line width matches the source); 1–2 character labels fit by **height** (badge numbers vary greatly in width).
   - `verify_text_fit.py` is the last gate before erasure: measures `dw/dh/dx/dbase` per run. 033 final medians: `|dw|=0.1`, `|dx|=0.0`, `|dbase|=0.0`.

2. **Text Erasure**
   - Prefer `erase05.py`: mask-level erasure that only modifies glyph-mask pixels, leaving all other pixels in the box untouched — no rectangular ghosts.
   - For pure white backgrounds and dense small labels, `erase_text_v2.py` works: automatic background-type routing (flat → flat fill / artwork → inpaint), with forced inpaint IDs and a bounded residual-glyph pass.
   - For labels on tinted strips, gradient edges, or white text on color chips, `erase05.py` is mandatory; white text color must never be sampled — always use `#ffffff`; polarity is judged by local contrast, not globally.
   - Residual-glyph loop: `scan_remnants.py` scans → merge boxes / re-place → re-erase → rescan, 2–3 rounds.

3. **Tracing**
   - `faithful` profile: VTracer 1.x, 4× Lanczos, up to 128 colors, filter-speckle=4, color-precision=8, gradient-step=4, simplify=0.6, optimize=0; resampled image capped at 32 MP.
   - Same-winding compound path splitting (`split_compound_paths.py`) resolves QA's "Ambiguous nonzero compound fill".
   - Local ghosts can be patched with `delete_ghost_paths.py`, but the residual-glyph loop is the root fix.

4. **Live-Text Injection**
   - `merge-text` turns each manifest run into real `<text>` placed at the original baseline.
   - When all labels originally sit above the artwork, a separate `Editable text` layer is created; interleaved text retains the original paint order.
   - `letter_spacing` is always 0; `font_size ≤ 45` and `≤ box height / 0.72`.

5. **Native Import**
   - `import_to_illustrator.py`: `taskkill Illustrator` → sleep ≥70 s → launch → `app.open(master.svg)` → create `Background` layer → `export PNG 200%` → `saveAs(.ai)`, all in one call.
   - Launch must use `DEVNULL + DETACHED`; otherwise the child process inherits the stdout pipe and the background wrapper process never exits after writing files.
   - The SVG import route keeps the artboard 1:1; batch-runtime drawing adds a 0.72-scale margin and has been abandoned.

### Route B: Line-Art Reconstruction (AniLines One-Stroke Geometry)

Applies to: scenarios requiring an independent line-art layer, one-stroke geometry as a drawing source, or overlay onto an existing `figure.ai`.

```text
original.png
 ├─ Stage 1  lineart_extract.py   AniLines basic/detail → lineart.png
 ├─ Stage 2  lineart_text_filter.py  glyph-level text removal → clean.png
 ├─ Stage 3  lineart_strokes.py  one-stroke assembly → strokes.json
 ├─ Stage 4a lineart_overlay.py preview   local composite preview
 └─ Stage 4b lineart_overlay.py native    independent AniLines lineart layer
```

**Stage 1 — AniLines Extraction**

```bash
<python> -X utf8 scripts/lineart_extract.py <image.png> <lineart.png> --mode detail --scale 2
```

- `basic`: fewest strokes, best for one-stroke decomposition.
- `detail`: preserves fine texture, default.
- `--scale 2`: the fully convolutional network upscales input 2× with Lanczos before inference so thin lines are not smeared by the Nyquist limit; all three stages default to `--scale 2`, with `--scale 1` as the legacy fast path. Output coordinates are always in original-image space.
- Weights: `third_party/anilines/weights/anilines_{basic,detail}.pth` (69 MB each, MIT, gyrojeff/AniLines). Reinstall: download `basic.pth` / `detail.pth` from `https://hf-mirror.com/gyrojeff/AniLines/resolve/main/` and add the `anilines_` prefix.

**Stage 2 — Glyph-Level Text Filtering**

```bash
<python> -X utf8 scripts/lineart_text_filter.py 
  --lineart <lineart.png> 
  --text <text_spec.json> 
  [--text-manifest <text_manifest.json> --measure-from <original.png>] 
  --out <cleaned.png>
```

Algorithm: binarize the line art → label 8-connected components → **delete only components whose bounding box falls entirely inside a text ink box (+3 px tolerance)**. Leader lines, label chips, and cell outlines crossing the box are always kept. No inpaint, no flat fill, no damage to crossing graphics.

Text box sources (union):

- `text_spec.json` — `lines[].bbox`, measured tight ink boxes (preferred).
- `text_manifest.json` + `--measure-from` — `runs[].erase_bbox` is often inflated 7–50 px, so it is used only as a **search window**; the tight ink box is re-measured from the original inside each window.

> Known limitation: Stage 2 keeps glyph fragments touching graphics (fully-inside rule). Acceptable — the alternative (box erasure) damages graphics, which is worse.

**Stage 3 — One-Stroke Geometry Assembly**

```bash
<python> -X utf8 scripts/lineart_strokes.py <cleaned.png> <original.png> <strokes.json> [--scale 2] [--smooth 0.8]
```

Binarize → close 1 px cracks + area denoise → skeletonize → Rutovitz crossing-number node detection → node clustering → chain extraction → tangent-continuous merging → per-stroke polyline (closed flag, width, ink color sampled from the original).

- "One circle, one stroke" is guaranteed by construction: pure loops stay `closed=true`, and fragments crossing junctions merge by tangent continuity (`parts>1`).
- `--scale 2` consumes 2× line art; distance-transform widths are halved back to original units, giving sub-pixel width estimates.
- `--smooth` applies a light Gaussian along each path to remove skeleton staircase artifacts so polylines follow the original curves.
- Ink color sampling: each centerline point takes the darkest pixel in a 5×5 (fallback 9×9) window, and per-stroke ink is the median of those minima; strokes whose darkest pixel is still ≥250 are treated as inkless and skipped with a printed count. `--keep-blank` keeps them; `--blank-lum` adjusts the threshold.

**Stage 4a — Local Preview**

```bash
<python> -X utf8 scripts/lineart_overlay.py preview 
  --strokes <strokes.json> --figure-png <figure.png> --out <preview.png> 
  [--mode full] [--viewbox W H]
```

`--mode full` is for SVG-import documents (artboard fit); the default `batch` matches the skill's 0.72 placement. `--viewbox W H` is mandatory for non-1536×1024 artboards.

**Stage 4b — Native Independent Layer**

```bash
<python> -X utf8 scripts/lineart_overlay.py native 
  --strokes <strokes.json> --figure-ai <figure.ai> 
  --out-ai <figure_lineart.ai> --out-png <figure_lineart.png> 
  [--viewbox W H]
```

Opens `figure.ai`, resolves the placement transform (prefers the root note `scientific-vector-local|hash|left|top|scale`; SVG-import documents without the note fall back to viewBox-centered fit), adds an independent `AniLines lineart` layer mapped 1:1 onto the artwork, and saves to a new file — **the original `figure.ai` is never overwritten**.

**Quality Gate**

`lineart_eval.py` measures against the original (excluding text areas): `edge_recall` (original Canny edges covered by ink within ±tol — detail preserved), `edge_precision` (ink lying on original edges — fit), and `ink_density` (over-extraction monitor). Compare before/after parameter changes at the same effective resolution.

### Fill Bounds and Background (Mandatory)

> **Line art is the sole authority. Never determine background by color.**

Every filled object must stay inside the marks bounded by the line art; the page background must never be baked into the artwork. If a reconstruction leaks color into the margin or carries an opaque page-sized background path, it is not deliverable — the user needs to key the figure onto another background.

**Fixed order: line art first, then fill bounds.**

1. Complete the `AniLines lineart` layer (extract → glyph filter → strokes).
2. Derive the matte from the same line art.
3. Then trace the fills.
4. The line-art layer is an independent deliverable and **must not be edited afterward** — no recoloring, no erasing, no "repairing" strokes to make the fills look better. The only permitted edit is the glyph-level text filter (labels must be live text).

**Pipeline Steps**

```bash
# 1) Line-art matte → transparent base
<python> -X utf8 toolkit/03_erase/fill_matte.py --source <text_clean.png> --lineart <la_clean.png> --out <fig_cut.png> --shrink 2

# 2) Ink restoration to alpha (mandatory)
<python> -X utf8 toolkit/03_erase/restore_ink_alpha.py --cut <fig_cut.png> --lineart <la_clean.png> --out <fig_cut_ink.png> --report <ink.json>

# 3) Alpha-preserving faithful tracing
<python> -X utf8 toolkit/04_fixup/vectorize_cut.py --image <fig_cut_ink.png> --out <geom.svg> --vtracer <skill>/bin/vtracer.exe

# 4) (Optional) Drop page-colored large fills
<python> -X utf8 toolkit/04_fixup/drop_bg_fills.py --svg <master.svg> --out <master_cut.svg> --source <orig.png> --min-area 8000

# 5) Delivery: Background layer + transparent-background PNG
<python> -X utf8 toolkit/05_delivery/import_to_illustrator.py --svg <master_cut.svg> --ai <figure_final.ai> --png <figure_final.png> --png-nobg <figure_nobg.png> --bg-color "#fefefe" --skill-dir "<skill directory>"
```

**Key Details**

- `fill_matte.py`'s directional closing: an isotropic kernel large enough to seal frame corners also seals the 5–10 px panel gutters, turning the whole page opaque. `--shrink 2` (default) compensates VTracer's ~2 px systematic outward growth along alpha edges; measure before changing it.
- `vectorize_cut.py` replaces `figure_cli.py vectorize`: the latter's faithful profile preprocesses with `convert('RGB')`, dropping alpha and writing the background back.
- `drop_bg_fills.py` is only for pure-color pages: it samples page color from the original image border via `--source`, **never hardcoding white**; the area threshold (default 8000 px²) must not be removed — deleting by color alone would also remove content colors like cytoplasm and alveolar lumens (measured on 01: 605 of 1585 page-colored paths were content). Skip this step if the page is not pure-color; geometric matting itself already guarantees no spill.
- Text is exempt from the fill constraint, but text frames must not carry their own background patches; chips/boxes behind labels belong to the artwork layer and are traced as geometry, never arriving as text-frame fills.

**Acceptance (both are mandatory; neither replaces the other)**

```bash
# Numeric
<python> -X utf8 toolkit/05_delivery/verify_fill_bounds.py 
  --png <final.png> --png-nobg <nobg.png> --matte <fig_cut_ink.png> 
  --source <orig.png> --text-manifest <manifest> --tol 3

# Visual: two images on black + magenta backgrounds
<python> -X utf8 toolkit/05_delivery/visual_check_bg.py --png-nobg <nobg.png> --out-dir vis
```

| Metric | Meaning | Threshold |
| --- | --- | --- |
| `spill_ratio` | Opaque pixels outside matte + tolerance (text boxes excluded) | ≤ 0.10% |
| `bg_channel_delta` | Exported border color vs. source border color | ≤ 2 |
| `corner_alpha` | Corner alpha | == 0 |
| Largest page-colored component | Whether a page slab remains | < 8000 px² (only required if cleanup was performed) |
| `white_left_ratio` | Informational | — |

**How to set `--tol`**: VTracer systematically grows outward along alpha edges, and `fill_matte --shrink` only compensates part of it. Measured with a synthetic square: a 200×200 image with a 100×100 opaque square in the middle traces back at 47.5..152.6 (should be 50..150) under faithful parameters → 2.5–3 px outward growth. On this machine's VTracer 1.0.0-alpha.4, use `--tol 3`; hardcoding 2 treats tracer bias as "fill spill" (033: tol=2 → spill 0.0122 FAIL; tol=3 → 0.00056 PASS).

**Visual interpretation**: On the black background, no color blocks may appear outside the figure or in panel gutters; on the magenta background, any non-magenta patch indicates fill spill. Interior fills such as cytoplasm, chips, and alveolar lumens that are **bounded by their own strokes** are content and may remain — judge by "does it have its own stroke", not by "is the color light".

**Known soft spot**: At unclosed line-art locations, the region connects to the page margin, flood classifies it as exterior, and the interior fill is removed along with it. This manifests as a region vanishing — check `matte.json`'s `kept_ratio`, go back to the line art to close the gap, or increase `--seal`; **do not disable the matte**. Strokes themselves being removed is solved by `restore_ink_alpha.py`; the two are different issues.

### Delivery and Acceptance

**Acceptance Checklist**

- [ ] Every label and important structure compared against the source: arrows, colors, layout, details.
- [ ] Native text count == reviewed SVG runs count, fonts correct.
- [ ] Raster reconstruction includes an independent `AniLines lineart` layer (not merged into artwork).
- [ ] Real paths + text delivered, no embedded bitmaps, external links, or scripts.
- [ ] The user's existing artwork and original files are preserved.
- [ ] Inspect **Illustrator's actual exported PNG (200%)** label by label: no double ghosting, no font-size blowout, no residual overlap. Local path previews do not fully render text and cannot replace this check.
- [ ] `native_export_complete_visual_review_required` only means automated native checks completed; **visual review is still mandatory**; only claim native `.ai` delivery after Illustrator has actually saved it.
- [ ] `verify_fill_bounds.py` four checks PASS, and both `visual_check_bg.py` composites have been visually inspected.
- [ ] `.ai` carries an independent `Background` layer (source page color); hiding it leaves no opaque pixel.
- [ ] Disclose tracing differences; visual acceptance is not proof of pixel-perfect identity.

**Deliverables**

`.ai`, `.svg`, `.png` (white background), `.png` (transparent background), `text_manifest.json`, `qa.json`, `playback.json`, `lineart_strokes.json`, and necessary audit records.

**Layer Order**

`import_to_illustrator` creates `Background` (bottom) → `split_text_layer` adds `Editable text` (top) → `lineart_overlay native` adds `AniLines lineart` (topmost). Final top-to-bottom:

```text
AniLines lineart
Editable text
图层 1 (trace geometry)
Background
```

Two normal phenomena that "look like missing objects":

- After SVG import, graphics are merged into **1 GroupItem** (`trace geometry`), and the layer panel shows only `图层 1 (1)`; this is not lost graphics — `pathItems` still counts all contours.
- `pathItems` count ≠ manifest runs count: the former are traced paths, the latter are text frames — two independent sets.

---

## Toolkit

`toolkit/` is the supplementary toolset distilled from five rounds of real-image recreation, covering the middle and both ends that the skill core does not include. **At the start of every stage, first look in `toolkit/` for a ready-made script that does the job or can be adapted; writing from scratch is a last resort ("非必要不从头开始写代码"). Work on a copy, never modify the `toolkit/` original directory; record adaptations in the job log.**

Full usage: [`toolkit/调用指引手册.md`](toolkit/调用指引手册.md). Script index:

| Stage | Script | Purpose / When to Use |
| --- | --- | --- |
| 1 Detection | `01_detect/detect_text_lines.py` | First step for every image; produces `lines.json` + 1:1 annotated crops |
| 2 Manifest | `02_manifest/match_text_template.py` | Preferred when text is known; PIL-rendered templates + multi-scale matchTemplate + position prior; low scores auto-marked `<-- low-score` |
| 2 Localization (texture images) | `02_manifest/local_match.py` | Default localizer for texture-dense images; local window + binary overlap F1; flat backgrounds score 0 |
| 2 Box Growing | `02_manifest/grow_boxes.py` | Mandatory after localization; extends horizontally by column gaps, vertically by row ink; badge numbers use `--skip-vertical-suffix _num` |
| 2 Finalization | `02_manifest/fit_sizes.py` | Multi-character fits by width, 1–2 character by height; aspect-ratio out-of-bounds falls back to height; `--colors` passes text colors |
| 2 Validation | `02_manifest/verify_text_fit.py` | Last gate before erasure; measures `dw/dh/dx/dbase` |
| 2 Library | `02_manifest/pil_metrics.py` | PIL-measured metrics (including kerning); renders at reference size then scales linearly |
| 2 Font Selection | `02_manifest/font_pick.py` | When the source font is unavailable, measure before choosing; reports width/height consistency and glyph F1 separately |
| 2 Manifest | `02_manifest/build_manifest.py` | Row-box assignment + font size/baseline/fill; output compatible with merge-text |
| 2 Library | `02_manifest/text_metrics.py` | Pen-aware metrics/placement/background-type discrimination (flat/artwork routing) |
| 2 Optional | `02_manifest/snap_runs.py` | Snaps runs in place to nearby row boxes; low-confidence results re-checked by Stage 3b |
| 2 Measurement | `02_manifest/probe.py` | The only reliable source when hand-writing boxes; `--light` for light-on-dark, `--thr 175~190` for labels over artwork, `--json` feeds directly into manifest |
| 2c Refinement | `02_manifest/refine_boxes.py` | When boxes are contaminated by panel borders/color bars/arrows; output boxes come from the rendered template itself; `score<0.55` goes to manual review |
| 2e Format Conversion | `02_manifest/conv_manifest.py` | Adds canvas/font_size/font_family/reviewed; rounds `erase_bbox` (floats cause `lineart_text_filter` to throw TypeError) |
| 2 Example | `02_manifest/runs_example_03_2.json` | 66-run example (with bold/italic, hints) |
| 2 Example | `02_manifest/spec_example_03_3.json` | 69-run template-matching spec example |
| 3a Erasure (preferred) | `03_erase/erase05.py` | Mask-level erasure; only modifies glyph-mask pixels; `polarity=-1` for light-on-dark; `disc` for in-circle constraint |
| 3a Erasure (alternative) | `03_erase/erase_text_v2.py` | Pure white backgrounds/dense small labels; automatic background-type routing; `--inpaint-ids` forced; `--residual` bounded residual pass |
| 3a Erasure (v1) | `03_erase/erase_text.py` | Three branches; white-text dominant-color fill `--white-ids` |
| 3b Residual Loop | `03_erase/scan_remnants.py` | Mandatory 2–3 rounds per image; wherever residual glyphs are, the real text is; `--apply` writes back |
| 3.6 Matting | `03_erase/fill_matte.py` | Route A core; alpha determined solely by line art; directional closing seals gaps; `--shrink` compensates VTracer growth |
| 3.6b Restoration | `03_erase/restore_ink_alpha.py` | Mandatory after fill_matte; rescues dashed lines/open contours; 033 measured `restored_px=52853` |
| 4a Splitting | `04_fixup/split_compound_paths.py` | Mandatory when QA reports "Ambiguous nonzero compound fill" |
| 4b Ghost Removal | `04_fixup/delete_ghost_paths.py` | Remedial only; zones should be generous |
| 4c Tracing Variant | `04_fixup/vectorize_cut.py` | Alpha-preserving faithful tracing; parameters identical to skill faithful |
| 4d Background Cleanup | `04_fixup/drop_bg_fills.py` | Only drops large page-colored fills; page color sampled from `--source`; area threshold must not be removed |
| 4e Clipping (Reference) | `04_fixup/clip_to_matte.py` | **Not for delivery**: produces `<defs>` which conflicts with `validate` |
| 4f Line-Art Text Removal | `04_fixup/filter_text_strokes.py` | Removes line-art strokes falling inside text boxes; pad default 8, use 12 for severe ghosting |
| 5 Delivery | `05_delivery/import_to_illustrator.py` | Native SVG import → `.ai` + PNG (Background layer); built-in taskkill→sleep≥70→launch→import |
| 5 Delivery (attach) | `05_delivery/import_attach.py` | When the user already has Illustrator open; skips restart and attaches directly; JSX identical to the former line by line |
| 5b Layering | `05_delivery/split_text_layer.py` | Mandatory before delivery; moves all textFrames to the top `Editable text` layer |
| 5 Acceptance (numeric) | `05_delivery/verify_fill_bounds.py` | Spill / bg delta / corner alpha / page-color residue |
| 5 Acceptance (visual) | `05_delivery/visual_check_bg.py` | Two images on black + magenta |
| 8 Full-Bleed Collage | `05_delivery/verify_panel_layout.py` | Panel coverage, gutter transparency, unexpected holes, label landing |

**Measured Calibration Values (directly reusable on the same machine/version)**

| Quantity | Measured Value | Usage |
| --- | --- | --- |
| VTracer outward growth along alpha edges | 2.5–3 px | `verify_fill_bounds --tol 3`; `fill_matte --shrink` only compensates 2 px |
| Template outer box narrowing vs. source ink | 6–12% (increases with string length) | Localization boxes must go through `grow_boxes`, never used as `erase_bbox` |
| Source font vs. Arial width difference | ~7% narrower (fs_w/fs_h median 0.872–0.92) | Width fitting gives correct line width but slightly small height — a disclosable inherent limitation |
| `fill_matte` misclassified open-line pixels | ~3.4% (033: 52853 / 1572864) | Without `restore_ink_alpha`, entire dashed lines are lost |
| Residual outward growth after `--shrink` | ~1 px | Pair with `--tol 3`; do not press down to 2 |

---

## Self-Iteration Mechanism

At the end of every task round, all high-value, reusable, or reference-worthy scripts are automatically reviewed:

1. Categorized into the corresponding `toolkit/` stage directory;
2. `toolkit/调用指引手册.md` index and acceptance criteria updated;
3. Tool adaptations recorded in the job log (what changed, why).

After five rounds of recreation + multiple independent recreation experiments, the toolkit contains **24 scripts + 2 example data files**, each of which has passed `py_compile` + real-image smoke testing; new scripts must also pass end-to-end chaining (033 independent batch: `local_match → grow_boxes → fit_sizes → conv_manifest → verify_text_fit`, 69 runs with median `|dw|=0.1`, `|dx|=0.0`, `|dbase|=0.0`).

Historical defects are also recorded in the manual (e.g., `scan_remnants`'s 5-tuple unpacking crash, `verify_text_fit` not passing anchor causing centered labels' dx to be inflated, `font_pick` judging wide fonts poorly using only height fitting) to prevent the next person from stepping on the same mines.

---

## Measured Data and Performance

### Five-Round Recreation Overview (2026-10-26 ~ 10-27)

| Round | Figure | Spec | Text | Paths | Convergence |
| --- | --- | --- | --- | --- | --- |
| 1 | 01_Introduction | 1536×1024 | 57 | 24,447 objects | First version caught 10 text deviations in visual inspection; converged in round 2 |
| 2 | 02_1_TR_AM | 1536×1024 | 70 | 26,077 | Negative-letter-spacing ghosting resolved in ~3 rounds |
| 3 | 02_2_Mo_AM | 1536×1024 | 67 | 38,991 | Converged in 2 rounds after row-grouping bug fix |
| 4 | 03_1_Steady_state | 1774×887 | 58 | 33,029 | First round poor; 11 rounds of residual-glyph-loop repair → basically converged |
| 5 | 03_2_Disease_dynamics | 1536×1024 | 67 | 46,329 | Converged within 4 rounds using the loop method |

Total output: **305 live-text TextFrames, ~200,000 vector paths**, artboard 1:1 with source.

### Other Independent Recreations

| Figure | Spec | Result |
| --- | --- | --- |
| 03_3_Dysfunction | Five panels | Template matching hit all 69; `local_match` had only 3 low-score; final median deviation 0.0–0.1 px |
| 04_Therapy | 1536×1024 four panels | 8 of 79 boxes contaminated by borders/color bars; fixed by `refine_boxes`; 395-field regression with 0 differences |
| 1080×1446 six-panel full-bleed collage | Six panels with different base colors | Standard criteria structurally fail; switched to layout-geometry alpha, `erase_by_ring` three-way color picking, `verify_panel_layout` |

### Efficiency and Cost

- **Large complex review figures**: with DeepSeek connected, average time ~**40 minutes**, cost ~**2–3 RMB** (user-measured).
- Automation should "refuse" rather than "guess": combined width/height fitting with a rejection threshold (score>1.0 → manual), 03_2 auto-assigned 63/66 with only 3 manual; pure manual mapping in 02_2 went through three rounds.
- The residual-glyph loop cut 03_1 from 11 iterations to 3.
- Place the verification loop at the cheapest layer: numeric scanning (residual glyphs/font size/spill) → rendered visual inspection (ghosting/blowout/residual overlap); two-level gating is most cost-effective.

### Methodology Lessons (Highest-Value Rules)

1. **Trust only program-measured ink boxes, never eyeballed coordinates.** The Read tool downscales large images, and mixing 5×/2×/1× crops shifts perceived positions by 20–100 px. Placement, font size, and erasure targets all come from connected-component measurements.
2. **Letter spacing is always 0.** Negative letter-spacing causes Illustrator to double-render the same text frame; width mismatches are fixed with `fs × target width / rendered width`.
3. **Separate box purposes.** Localization box, font-size target, and erasure region are three different things. Using a widened erasure box to fit font size inflates it by 20–45 pt. `fs ≤ box height / 0.72` and `≤ 45`.
4. **Erasure needs a residual-glyph loop.** Erase → scan residual clusters (width-matched, height ≤30 px, ≥2 characters) → merge boxes → re-erase → rescan. One pass never converges.
5. **Never damage graphics to remove text.** Delete only vector paths whose bounding box falls entirely inside a text box (dilate ≤7 px); keep glyph fragments rather than swallowing arrows, granules, or color bars.
6. **Ghosts are usually from the erasure layer, not the line-art layer.** When a label looks doubled, crop the same region from the base figure and compare; most ghosts pre-exist in the traced geometry (glyph outlines 5–15 px larger than the erasure box).
7. **Reliable delivery:** `taskkill Illustrator` → sleep ≥70 s → launch → `app.open(master.svg)` → `saveAs(.ai)` + `export PNG 200%`, all in one call. Check the exported PNG, not the path preview.
8. **Know the inherent boundaries.** Gradient quantization (128 colors), font substitution, and white-on-chip erasure cost are method boundaries — disclose them, do not treat them as bugs.

---

## Known Limits and Disclosures

| Category | Limitation | Description |
| --- | --- | --- |
| Color | Gradients quantized to 128 flat colors | Faithful profile limit; inherent method limitation |
| Font | Arial approximation when source font is unavailable | Width fitting gives correct line width but ~7% smaller height; disclosable difference |
| Erasure | White-on-chip and over-artwork text has a "half-erased fragments worse than not erasing" cost | Conservative solution is to keep the source glyph and drop the live text there |
| Ghosting | Some labels have 5–15 px light ghosts (source glyph larger than erasure box) | Each fix takes one round of "merge box → re-erase → re-trace" (~6 min/group) |
| Line art | Open lines/dashed lines are flood-classified as exterior | `restore_ink_alpha.py` is mandatory; if regions still vanish, close the line art or raise `--seal`; do not disable the matte |
| Full-bleed collage | Standard criteria structurally fail when multiple panels have different base colors | See `toolkit/调用指引手册.md` §8: layout-geometry alpha, `erase_by_ring`, `verify_panel_layout` |
| Input type | Complex microscopy photographs generally unsuitable for the all-vector pipeline | See `input-and-validation.md` |
| Data | Tracing cannot recover experimental data/numeric measurements | Statistical plots should use original vector exports and real data |
| Native | Windows 27.2.0 hardware-validated; macOS only adapter contract checks | Other versions require individual validation; no universal compatibility claimed |
| Degradation | Without Illustrator, cannot produce `.ai` / live-text layering / native line-art layer | Degraded delivery: SVG + PNG + `lineart_strokes.json` + line-art bitmap preview, with explicit reason stated |

**Degraded Delivery** (`illustrator-doctor` reports `Open Illustrator yourself` and confirms it is not installed):

- SVG (8109 objects / 48 live-text / 0 bitmaps)
- PNG (white background)
- PNG (transparent background)
- `lineart_strokes.json` (3330 one-stroke paths, scriptable redraw in Illustrator as an independent layer)
- `lineart_layer.png` (line-art bitmap preview)
- Must explicitly inform the user that `.ai` was not produced and why

---

## Directory Structure

```text
illustrator-Master Copyist/
├── SKILL.md                          # Skill main document (read by Agent)
├── README.md                         # This document
├── LICENSE                           # Upstream MIT license
├── NOTICE.md                         # Sources and license notice
├── .venv/                            # Self-contained CPython 3.13.14 + all dependencies
├── bin/
│   └── vtracer.exe                   # VTracer 1.0.0-alpha.4 (pinned SHA256)
├── third_party/
│   └── anilines/                     # AniLines code + dual 69 MB weights (MIT)
│       └── weights/
│           ├── anilines_basic.pth
│           └── anilines_detail.pth
├── scripts/
│   ├── figure_cli.py                 # Shared CLI entry point
│   ├── figurelib/                    # SVG/geometry/COM/AppleScript/scheduler
│   ├── setup_environment.py          # Install dependencies from source (optional)
│   ├── selfcheck.py                  # Package self-check (no Illustrator/network)
│   ├── lineart_extract.py            # Stage 1
│   ├── lineart_text_filter.py        # Stage 2
│   ├── lineart_strokes.py            # Stage 3
│   ├── lineart_overlay.py            # Stage 4a/4b
│   └── lineart_eval.py               # Quality gate
├── toolkit/
│   ├── 调用指引手册.md                # Authoritative toolkit usage
│   ├── 01_detect/
│   ├── 02_manifest/
│   ├── 03_erase/
│   ├── 04_fixup/
│   └── 05_delivery/
└── references/
    ├── workflow.md                   # Raster reconstruction and recovery
    ├── input-and-validation.md       # Input/acceptance specification
    ├── installation-and-compatibility.md
    ├── lineart-pipeline.md           # Line-art pipeline
    └── remake-test-report.md         # Five-round measured report
```

> Subject to the actual release package; the `toolkit/` script list grows with self-iteration.

---

## Troubleshooting Quick Reference

| Symptom | Action |
| --- | --- |
| Illustrator connection fails | Confirm the app is open in the same user session; use PowerShell 5.1 COM bridge on Windows |
| Multiple Illustrator versions running | Windows: `--prog-id Illustrator.Application.N`; macOS: `--app-path` with the exact running `.app` |
| macOS denies automation | User grants terminal/Agent permission under "Privacy & Security → Automation"; scripts do not change system permissions |
| Missing font | Identify the intended font or agree on a substitute; update SVG/manifest; start a new job |
| Pale nuclei disappear / background flattened | Compare higher-color candidates and inspect local regions; white-background edits only target identified background objects |
| Small closed cubic loop rejected | The parser supports splitting a single closed cubic curve; use the updated skill and start a new job |
| No clear placement space | Use a blank artboard/document; `--allow-overlap` only when already authorized |
| Adobe dialog blocks drawing | Inspect the dialog and confirm the previous operation is idle before starting |
| Success log but incorrect image | Check the actual PNG, font faces, object hierarchy; execution success is not visual acceptance |
| `ModuleNotFoundError: figurelib` | Explicitly pass `--skill-dir` to `import_to_illustrator.py` / `split_text_layer.py` |
| Background wrapper process never exits after writing | Launch Illustrator with `DEVNULL + DETACHED` |
| Layer panel selects tens of thousands of paths in one click | Run `split_text_layer.py` before delivery to separate live text into `Editable text` layer |
| Line-art layer entirely gray/noisy | Do not extract line art from the text-cleaned image; use original → `lineart_extract` → `lineart_text_filter` → remove <12 px small components |
| White text delivered as pinkish-purple | Always use `#ffffff` for white text; do not sample (source white text's G channel is pulled down by background bleed) |
| Central divider/iNOS box disappears | `fill_matte` classified open lines as exterior; run `restore_ink_alpha.py`; do not disable the matte |
| Panel gutters filled opaque | `fill_matte` closing kernel too large sealed white gutters; use directional closing and check `--seal` |
| `spill_ratio` just barely FAILs | Set `--tol` to the measured 3, not the default 2; first measure this machine's VTracer growth |
| Full-bleed collage corner_alpha non-zero | Panels extend to canvas edges; corners legitimately have content; use `verify_panel_layout.py` instead |

---

## Acknowledgments

- Thanks to the open-source neural-network-based line-art extraction model **[AniLines](https://github.com/gyrojeff/AniLines)** (MIT, gyrojeff); this Skill's independent line-art layer and one-stroke geometry pipeline are built on it.
- Thanks to the predecessor **[scientific-illustrator-local](https://github.com/yrui-cmd/cell-lct)** project for inspiration; this Skill restructures and extends it with line-art reconstruction, fill matting, toolkit, and self-iteration mechanisms.
- Thanks to **DeepSeek** and **MiMo** for powerful code capability support.
- Thanks to [VTracer](https://github.com/visioncortex/vtracer) (visioncortex) for the faithful vectorization backend.
- Thanks to everyone who checked label by label and repaired residual ghost by ghost across five rounds of real-image recreation and multiple independent recreation experiments.

---

## License

- This Skill adapts and restructures MIT-licensed material from [yrui-cmd/cell-lct](https://github.com/yrui-cmd/cell-lct) (commit `4af98d8057f48a944141f9122da7aba83d544d28`, 2026-09-06T16:20:16Z). Upstream copyright and MIT terms are preserved in [`LICENSE`](LICENSE).
- This package **does not contain** any Xiaomiao server, proprietary algorithms or weights, API key, payment flow, `cell_no_ai` service, AI-origin-mark removal service, Adobe application, model, or font files; it does not bypass any service authentication.
- [VTracer 1.0.0-alpha.4](https://github.com/visioncortex/vtracer/releases/tag/1.0.0-alpha.4) is an optional, separately installed dependency under its own license; the installer downloads only the pinned official release asset and verifies SHA256.
- Python dependencies retain their respective licenses.
- Model and Adobe access remain the user's responsibility; showing an image to a cloud model is not fully offline.
- Please keep `LICENSE` and `NOTICE.md` with the Skill.

---

*README compiled from SKILL.md v0.2.1, `toolkit/调用指引手册.md` (2026-10-29 edition), the five-round measured report, and the installation/input/line-art pipeline documents. Commands and thresholds are subject to the bundled documentation; for different Illustrator versions and machines, first run `doctor` / `illustrator-doctor` / `native-test` for validation.*
