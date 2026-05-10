# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

Sandbox for visualizing 2-D numpy matrices interactively in JupyterLab using the HoloViz stack (Bokeh, Panel, HoloViews). Primary use case: render matrices as zoomable "big-pixel" images and view multiple matrices side-by-side with linked pan/zoom.

## Run

```sh
jupyter lab          # open linked_pair.ipynb and run all cells
```

Dependencies (Python 3.10+): `bokeh`, `numpy`, `jupyterlab`, `panel` (needed for any slider-driven notebook). Add `holoviews` only when a notebook actually needs it.

## Patterns to reuse

**Linked pan/zoom across plots** — share `x_range` / `y_range` between `figure`s. `linked_pair.ipynb::linked_image_pair` is the canonical example. The second figure must be constructed with `x_range=p1.x_range, y_range=p1.y_range` (the *same object*, not a copy).

**Square pixels that stay square under zoom** — pass `match_aspect=True` to every `figure` in a linked group. Without it, Bokeh stretches pixels to fill the plot area and the two halves of a side-by-side view drift out of alignment.

**Matrix orientation** — Bokeh's `image` glyph places `image[0,0]` at the *bottom-left*. To get matrix-style top-left origin, set `y_range=(h, 0)` (reversed) rather than flipping the array, so axis tick labels stay sensible.

**Color scale** — use a `LinearColorMapper` per plot for independent scales, or one shared mapper passed to both `image()` calls when comparing magnitudes matters more than per-plot contrast. `linked_image_pair` exposes this via `shared_color_scale=`.

**Tool merging** — wrap linked plots in `gridplot([[...]], merge_tools=True)` so there's one toolbar driving both panels, instead of two redundant toolbars.

**Live updates from a Panel slider** — `linked_pair_with_slider(A, transform, slider)` is the canonical pattern. Build the figures *once* against `ColumnDataSource(data={"image": [arr]})`, then in the slider's watcher reassign `src.data = {...}` (replace the dict; do not mutate keys in place — Bokeh doesn't see partial mutations). Never rebuild `figure(...)` on each update: it breaks the linked range, the toolbar resets, and the notebook flickers. Wrap the `gridplot` once in `pn.pane.Bokeh(...)` and return `pn.Column(slider, pane)`.

**Autoscale vs comparable colorbars** — when the transform changes magnitude (blur, gamma, threshold), `autoscale_b=True` updates `LinearColorMapper.low/high` so colors fill the dynamic range. Set it `False` together with `shared_color_scale=True` if the pedagogical point is comparing magnitudes, otherwise the eye is fooled.

**Hover linking across plots** — see `_add_hover_link` in `linked_pair.ipynb`. Three layers, each chosen for a reason:

1. *Linked crosshair*: create one `Span(dimension="height")` and one `Span(dimension="width")`, then add a `CrosshairTool(overlay=[vline, hline])` to **each** plot referencing the *same* Span instances. Bokeh updates the Span's `location` when one plot is hovered, and because both plots' tools point at the same model, the crosshair appears on both. Two `CrosshairTool` instances, one Span pair.
2. *Per-pixel highlight rect*: a near-empty `ColumnDataSource(data={"x": [], "y": []})` plus a `rect()` glyph on the *opposite* plot, driven by a `js_on_event("mousemove", CustomJS(...))` handler. Inside the JS, `Math.floor(cb_obj.x)` gives the integer column (the tooltip's `$x{0}` rounds rather than floors, so prefer the rect for precise feedback). Always pair with `mouseleave` that clears the source, otherwise a stale highlight stays after the cursor leaves.
3. *Tooltip*: `HoverTool(tooltips=[("col, row", "$x{0}, $y{0}"), ("value", "@image")], renderers=[image_renderer])`. Pin `renderers=` to the image glyph so the highlight rect doesn't trigger spurious tooltips. `@image` reads the underlying numpy value at the hovered cell.
4. *Dynamic title text + combined readout strip*: pass both image `ColumnDataSource`s, both `figure.title` models, and a Bokeh `Div` into the same `mousemove` `CustomJS` so a single handler updates everything. Read pixel values from JS as `src.data.image[0][iy * w + ix]` (Bokeh ships the image as a flat row-major typed array). The `Div` goes above the gridplot via `column(combined_div, gridplot(...))`. Cache the base title strings as `CustomJS` args (`base1`, `base2`) so `mouseleave` can restore them — figure titles are model state, not transient text, so anything you set sticks until you reset it.

When both arrays must share a grid (so the same `(col, row)` is meaningful in both), assert it explicitly — silent shape mismatch makes the highlight visually plausible but semantically wrong.

**Stale dynamic state after slider updates** — the dynamic title and readout were computed against the *old* `B`. After replacing `src_b.data`, also reset `p1.title.text`, `p2.title.text`, and the readout `Div.text` to their base values. The next mousemove repopulates them with correct numbers; without the reset, the user reads stale values until they hover again.

## Gotchas

- `output_notebook()` must be called once per kernel session before any `show(...)`; if plots render as raw HTML or not at all, that's the cause.
- Bokeh `image()` takes a *list* of 2-D arrays (`image=[arr]`), not the array directly.
- Big pixels look crisp because Bokeh draws each cell as a discrete rect — do not switch to `image_rgba` with pre-rendered PNGs unless you specifically need RGBA, since browser smoothing will then blur them.
