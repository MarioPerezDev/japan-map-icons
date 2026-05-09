# Map Icons — Handoff

## Overview

`Map Icons.html` is a self‑contained, single‑file web app for assembling a set of round, color‑coded map icons (places of interest in Japan: kissaten, libraries, shrines, ramen shops, etc.). Each icon is composed of:

- a **circle background** in a category‑specific bg color,
- a **drawing / glyph** uploaded by the user (PNG, JPG, or SVG) and recolored to the category's primary ink color,
- placed in a fixed icon stamp (640×640 design canvas, exported at 512px PNG or as inline SVG).

The app is a tool, not a design reference. It already runs as‑is — open the HTML in any modern browser. The intent of this handoff is to turn the prototype into a maintainable repo that you (or a developer using Claude Code) can iterate on.

## What's in the bundle

| Path | What it is |
|---|---|
| `Map Icons.html` | The full app — HTML, CSS, and JS in one file (~1340 lines). |
| `reference-kissaten.png` | A reference still used inline by the prototype's reference panel. |
| `README.md` | This file. |

## How the app works (architecture)

Everything lives in one HTML file. Below is the seam map a developer should know before refactoring.

### Data model

Two top‑level data structures held in memory and persisted to `localStorage`:

```ts
// localStorage key: "mapicons.cats.v1"
type Category = {
  id: string;            // stable id (defaults to slug)
  slug: string;          // filename slug, e.g. "kissaten"
  label: string;         // display name
  primary: string;       // ink / glyph color (hex)
  bg: string;            // circle bg color (hex)
  mood?: string;         // freeform note shown in reference table
  variants: { slug: string; label: string }[];
  // Runtime-only fields added by reindexCategories():
  no?: string;           // numeric label like "01"
  file?: string;         // export filename like "01-kissaten.png"
};

// localStorage key: "mapicons.state.v1"
type State = {
  [categoryId: string]: {
    [variantSlot: string]: {  // variant slug, or "_default"
      type: 'raster' | 'svg';
      // for raster:
      dataUrl?: string;       // original upload as data URL
      bitmap?: string;        // recolored bitmap as data URL (cached)
      // for svg:
      svg?: string;           // raw SVG string
    }
  }
};
```

The two stores are independent: the category list is the *schema*, the state is the *content*. Re‑indexing (`reindexCategories`) recomputes `no` and `file` from order — never store those.

### Rendering pipeline

1. `renderStack()` rebuilds the entire stack of cards from `CATEGORIES`.
2. For each category × slot pair, `tileMarkup()` emits a `.tile` and `setupTile()` wires up dragover/drop, click‑to‑pick, and the variant remove `×` button.
3. `renderTile()` reads the slot's payload from `state` and either:
   - draws the recolored bitmap at the icon's `primary` color into a canvas, or
   - re‑colors the SVG glyph by string‑replacing `currentColor` / `fill` and inlining it.
4. `exportPng()` and `exportSvg()` recompose the final icon (circle + glyph) at export time. Both share `stampSvg()` for the geometry so the on‑screen preview and the exported asset are pixel‑identical.

### Color recoloring

For raster uploads, `recolorToInk()` walks the image's alpha channel and stamps every non‑transparent pixel with the category's `primary` color, keeping the original alpha. For SVG uploads, `recolorSvgString()` rewrites `fill`/`stroke`/`currentColor`. Black‑on‑white line‑art uploads work best.

### UI surfaces

- **Toolbar** (top): progress count, *Download all (zip)*, *Edit config (JSON)*, *Reset all*.
- **Stack** (`#stack`): one `.cat-section` per category, holding a `.cat-row` of tile cards.
- **Add category** modal (`#cat-modal`): label, slug, mood, primary/bg colors, optional variants.
- **Add variant** modal (`#variant-modal`): label + slug for a new variant of an existing category.
- **Edit config (JSON)** modal (`#config-modal`): full‑text editor for the categories array. `Apply` validates with `validateAndNormalizeConfig()`, swaps `CATEGORIES`, and prunes orphaned state.
- **Reference panel** (`#reference`): summary table with a swatch + filename per slot.

### External dependencies

Loaded from CDN inside the HTML:

- `jszip@3.10.1` — for the *Download all (zip)* button.

No build step. No frameworks. Vanilla DOM.

## Suggested next steps for a real repo

The single file is fine for prototyping but should be split before more features land. A reasonable target layout:

```
src/
  index.html
  styles.css
  app.js                # bootstrap + toolbar wiring
  data/
    categories.js       # CATEGORIES default seed + load/save
    state.js            # per‑slot payload state
  render/
    stack.js            # renderStack / renderTile / tileMarkup
    stamp.js            # stampSvg, geometry constants
    recolor.js          # recolorToInk, recolorSvgString
  io/
    export-png.js
    export-svg.js
    download-zip.js
  ui/
    modal-category.js
    modal-variant.js
    modal-config.js
    toast.js
public/
  reference-kissaten.png
```

Other reasonable improvements:

- **Per‑category drag‑reorder.** Indices are computed from array order, so a drag‑sort already works in principle — just needs UI.
- **Undo stack.** State mutations all go through a small set of functions (`setPayload`, `saveCategories`, `applyConfig`); wrap them in a command pattern.
- **File‑system sync.** Replace `localStorage` with the File System Access API (or a server) so the JSON config + uploaded glyphs round‑trip to a real folder on disk.
- **Tests.** `recolorToInk`, `validateAndNormalizeConfig`, and `stampSvg` are pure and unit‑testable. Pull them into modules first.
- **Multiple icon stamps.** Right now everything uses one round stamp. The `stampSvg` function is the only place to change to support pin‑shaped, square, or hex variants.
- **Theme presets.** The category color picker currently takes any hex; a curated palette would make the set feel more cohesive.

## License / attribution

The app was prototyped from scratch. The single reference image (`reference-kissaten.png`) is included as a visual aid and should be replaced with project‑owned art before publishing.
