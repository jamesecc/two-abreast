# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file interactive 3D simulation (`index.html` — the only source file) built to win an argument: a group of cyclists genuinely riding a maximum of two abreast *looks* three or four wide when viewed from a car behind, because perspective compresses depth. Everything (CSS, markup, JS) lives in that one file with **no `<!doctype>`/`<html>`/`<head>`/`<body>` wrapper tags** — deliberately, so the same file both opens locally in a browser and publishes as a claude.ai Artifact (the Artifact publisher wraps it in its own skeleton; nested wrappers would break). Do not add wrapper tags. No dependencies, no build step; the Artifact CSP forbids external resources, so everything must stay inline.

The published Artifact URL is https://claude.ai/code/artifact/ef81be70-efad-45d2-877d-841f73f36c7e — republish the same file path from the original conversation, or pass this URL as `url` to the Artifact tool from a new one, to keep the link stable after edits.

## Commands

Syntax-check the inline JS (extract script body, then node):
```bash
python3 -c "
import re
src = open('index.html').read()
open('/tmp/sim.js','w').write(re.search(r'<script>(.*)</script>', src, re.S).group(1))"
node --check /tmp/sim.js
```

Visual verification via headless Chrome (there are no tests; screenshots are the test suite):
```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new --disable-gpu \
  --window-size=1280,800 --virtual-time-budget=1500 --screenshot=/tmp/out.png \
  "file://$PWD/index.html?view=1&chaos=7&n=12&car=18"
```
- URL params drive any state for screenshots: `view` (0=top-down … 1=driver), `chaos` 0–10, `n` 5–15, `pair` 0–100, `curb` 0.1–1.5, `car` 5–50, `flip=1` (right-hand traffic), `ot` 0–13 (jump into the overtake timeline at t seconds — note headless virtual time advances ~1–2 s past the requested t before the shot). Params also skip the auto-sweep.
- **Headless Chrome enforces a minimum window width of 500px.** To test narrower (phone) viewports, write a wrapper HTML containing `<iframe src="file://…/index.html?…" style="width:390px;height:844px;border:0">` and screenshot that — media queries respond to iframe width.
- `--virtual-time-budget` does not reliably advance the animation/sweep; use the `view` URL param instead of waiting for the auto-sweep.
- Console errors surface as `INFO:CONSOLE` lines with `--enable-logging=stderr`.

## Architecture

One `<script>` (~700 lines), ordered: utils → world constants → state → formation generator → camera → renderer → scene pieces → apparent-abreast metric → minimap → HUD stats → main rAF loop → controls. World units are meters; left-hand (UK) traffic with curb at x=0, 7 m carriageway, riders travel toward +z.

- **Formation generator (`generate`/`updatePositions`)**: `generate()` stores per-rider *unit* jitters, phases, and row/column assignment from a seeded RNG (mulberry32); `updatePositions()` scales them live by the chaos slider each frame, so sliders morph the same formation instead of rerolling. **Core invariant: no row ever exceeds 2 riders, and row spacing/jitter bounds guarantee `trueCount()` (riders within 1 m longitudinally) never exceeds 2.** A per-frame pass enforces ≥0.85 m lateral separation inside each pair. Breaking this invariant destroys the whole argument the page makes.
- **Renderer**: hand-rolled perspective projection onto a 2D canvas (no Three.js — CSP). `camera()` returns a look-at basis; all drawing goes through near-plane-clipping helpers (`projPoly`/`fillPoly`/`seg`/`circle3d`/`sphere3d`) with painter's-algorithm ordering (sky/sun/clouds/hills → ground & road detail → hedges+trees+fence (one depth-sorted list) → car → riders far-to-near → bonnet overlay → vignette). Heads use `sphere3d` (billboard) — planar circles render edge-on as lines from behind. Scene texture (asphalt speckle, tar seams, grass tufts, cat's eyes, fence posts) is keyed by stable integer identity `m` along z, same as the hedges, so detail scrolls with the world instead of shimmering. Lighting is sun-from-the-right: rider jerseys get a darker -x half, ground shadows are `circle3d` ellipses offset toward -x.
- **Overtake mode (`carPose`/`OT`)**: the car's position (local lane x + driver-seat z) has a single source of truth, `carPose()`, read by the camera, the exterior car, the minimap, and the stats via `cam.pose`. The `OT` timeline (seconds: approach → pull out across the center line → pass in the far lane at x = ROAD_W−1.1 for clearance → pull in at zMax+14 → hold → dip-to-white → reset) drives it; `state.overtake = {t0, d0}` is set by the ≫ button or the `ot` URL param, and the whiteout covers the ahead→following reset. The "looks like" stat blanks to – whenever the eye isn't behind the group.
- **Camera sweep**: orbits the group's center from ~89° overhead down to the driver's eye (`alpha`/`R` lerp), rather than lerping positions — this keeps the whole group framed mid-sweep. The gaze target blends to the road-ahead point only at the end (`tt = s⁴`). The driver-eye nudge (d-pad/arrow keys) offsets the orbit's endpoint.
- **"Looks like N abreast" metric (`apparentCount`)**: union length of rider shoulder intervals in world-x divided by one rider width (0.5 m) — deliberately *not* screen-space clustering, which undercounts chains of partially overlapping riders.
- **Scrolling scenery**: riders stay fixed in z; the world scrolls past. Hedges/trees are keyed by a stable integer identity `m` (`z = m*spacing − scroll`) so per-plant size/offset travels with the plant — indexing by screen slot makes plants visibly teleport.
- **Responsive HUD**: fixed-position glass panels over the canvas with breakpoints at ≤1120px (stat pill becomes full-width strip, side panels drop to top:74px), ≤760px (panel starts collapsed with narrowed width to clear the minimap, minimap 0.5×), ≤520px *height* (landscape phones: d-pad moves bottom-left, hint hidden), ≤430px (third stat hidden). The JS collapse-on-load check must stay in sync with the CSS query (`max-width: 760px, max-height: 520px`). Layout was verified by screenshot across ~20 viewports; re-verify affected sizes after HUD changes.
