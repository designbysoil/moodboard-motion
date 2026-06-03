# CLAUDE.md — Drift (moodboard motion tool)

## What this is
A single-file, all-white minimal web tool that turns a set of uploaded images/mp4s into a
looping 16:9 "moodboard motion" clip in the style of a reference source video. The motion is **one big
canvas** of fixed-position tiles with a **single camera that pans across it**, pausing on **three
framings ("holds")** — the camera path was traced precisely from the source. Depth parallax + a
center-scale bulge add dimensionality. Exports an mp4 (or webm fallback) and previews live on canvas.

Everything lives in **one file**: `index.html` (HTML + CSS + JS inline). There is no
build step. Open it in a browser, or serve the folder (`python3 -m http.server`) and open it.
Libraries load from cdnjs at runtime:
- Three.js r128 (WebGL renderer)
- GSAP 3.12.5 (animation timeline)

If those `<script src>` tags fail, the app shows "Could not load WebGL/animation libraries."

## Why the design is the way it is (read before changing motion)
This was reverse-engineered frame-by-frame from a source clip (`Superintelligence_for_work_Sana.mp4`).
Key findings that the code encodes — do not casually undo these:
- **One big canvas, not 3 separate tile-sets.** All tiles sit at fixed canvas positions; a camera pans
  over them and pauses on 3 framings (holds). This is why panned-away tiles stay visible at the frame
  edges and transitions never empty out. (An earlier build modeled 3 disjoint stations that slid off/in
  — replaced; don't go back.)
- **The camera path was traced precisely** (OpenCV ORB + accumulated similarity over all 360 frames).
  Path: home → pan **down-right** to B → pan **up** to C → back to home. See `HOLDS`. The measured
  rotation (≤3.4°) and a brief mid-pan zoom-punch were intentionally **omitted** for cleanliness.
- Transitions ease **slow → fast → slow** with a spring **landing** (overshoot-and-settle, NO recoil/
  wind-up): `springEase`, zero velocity at both ends so holds/seam stay clean.
- **Bulge = measured.** Tracking individual tiles showed the center "bulge" is a **uniform per-tile scale
  magnification** keyed to distance-from-frame-center — aspect ratio stays locked, edges stay straight.
  It is NOT a lens warp or a geometric dome. Implemented as CPU scale (no shader). See "Bulge".
- Each tile also has an **independent scale pulse** (own phase/integer-cycle/frequency).
- Crop palette is **4:5-dominant** (≈63%), plus 1:1, 3:2, 16:9 accents. Tiles are center-cropped (cover).
- The loop must be **seamless** (camera at t=L === camera at t=0) for clean export. Verified: seam delta 0.

## File structure (search anchors inside index.html)
- `<style>` — all-white minimal UI; mono type; `.frame` holds the canvas.
- `const params` — live tunables (see Controls).
- `const SCENES` — the three hold compositions (the heart of the look). See "Data model".
- `HOLDS` — the 3 traced camera framings `{lookAt:[x,y], zoom}`; `PHASE` — loop-phase keyframes (uneven
  hold/transition timing). `DEPTHZ` — depth→renderOrder; `DEPTH_PAR` — depth→parallax gain; `BULGE_R`;
  `DISP_REF`/`DISP_START` — dispersion onset.
- `srcEase` (legacy) / `springEase` (easeInOutBack — the camera transition ease).
- `initGL()` — Three.js setup: `WebGLRenderer({preserveDrawingBuffer:true})`, **OrthographicCamera in
  pixel space** (y-up: top=OUT_H, bottom=0), unit `PlaneGeometry`. Starts `animate()`.
- `addFiles / media / addThumb / removeMedia` — upload handling + thumbnail tray.
- `applyCover()` — sets texture `repeat`/`offset` to center-crop to a tile's ratio.
- `rebuildScene()` — assigns media→slots and builds tile meshes at FIXED canvas positions (uses `HOLDS`).
- `buildTimeline()` — the GSAP timeline: tweens the single `cam={cx,cy,zoom}` through `HOLDS` per `PHASE`.
- `animate()` — per-frame render loop: projects each canvas tile through `cam` (+ parallax + bulge + pulse).
- controls wiring — sliders (Speed/Bulge/Parallax/Drift/Disperse/Scale/Pulse) + Shuffle.
- `pickMime / export onclick` — MediaRecorder export.

## Data model
`SCENES` is `[station0, station1, station2]`; each is an array of slot specs measured from a source frame:
```
{ hx, hy, long, r, d }
  hx,hy : normalized screen CENTER at that station (0..1)
  long  : long-side length in px at that station
  r     : target crop ratio (w/h). 0.80 = 4:5 portrait (dominant), 1.0, 1.33/1.5, 1.78
  d     : depth tier 'NEAR'|'MID'|'FAR' -> DEPTHZ (renderOrder) + DEPTH_PAR (parallax gain)
```
- station0 = "home", station1 = rocket/dome/starfield, station2 = glass-sculpture/eye/eclipse. The
  per-station framing comes from the traced `HOLDS[k].zoom` (no more `STATION_ZOOM` fudge).
- `hx,hy` are the tile's measured screen center **at its hold**; `rebuildScene` converts that to a fixed
  **canvas** position: `cw=measuredW/zoom_k`, `canvasCenter = HOLDS[k].lookAt + (measuredScreenCtr−frameCtr)/zoom_k`.

`HOLDS` (traced) = `[{lookAt:[961,541],zoom:1.0}, {lookAt:[1608,1284],zoom:0.881}, {lookAt:[1750,-395],zoom:0.921}]`
on a ~3008×2879 canvas (image coords, y-down). `PHASE` = loop-fraction keyframes with the source's uneven
timing (hold A 0–0.092 · A→B–0.308 · hold B–0.411 · B→C–0.65 · hold C–0.761 · C→home–1.0).

A built **tile** object:
```
{ mesh, station, type('img'|'vid'), el, tileAR, cropped, z,
  canvas:{cx,cy,w,h},   // FIXED position + size on the big canvas (zoom-1 reference)
  homeLookAt,           // = HOLDS[station].lookAt — parallax anchor (parallax is 0 here)
  distAtHold,           // screen dist from frame-centre at its hold (legacy; bulge no longer anchored)
  par,                  // DEPTH_PAR[d] parallax gain (NEAR +, FAR −)
  amp,cyc,ph,           // independent pulse: amplitude, integer cycles/loop, phase
  dax,day,dcx,dcy,dpx,dpy }  // micro-drift ellipse: amplitudes, integer cycles, phases (seamless)
```
There is **no per-tile enter/exit pose and no opacity animation** — tiles are always opaque at a fixed
canvas spot; the camera framing alone reveals/hides them. Render math (in `animate`): project
`canvas` through `cam` with parallax `pdx=(homeLookAt−cam.c)*par*parallax` (zero at the tile's hold), then
`size = canvas.w * cam.zoom * scale * pulse * bulge`.

## Motion model (buildTimeline) — camera pan over one canvas
The only animated thing is `cam={cx,cy,zoom}`. `buildTimeline` tweens it through `HOLDS` at `PHASE*L`
times: each transition is one `srcEase` tween from one hold's `{lookAt,zoom}` to the next; **hold spans
have no tween** (camera sits still). A 0.001s pin tween at `L-0.001` fixes total duration = `L`.

The first keyframe sets `cam` to hold A; the last transition (C→home) returns it to hold A, so
**cam at t=L === cam at t=0 → seamless** (verified: seam delta == 0). Source loop didn't perfectly close,
so the C→A leg is retargeted to exact home. If you re-trace or add a hold, keep the last keyframe == the
first hold or the seam breaks.

The render loop (`animate`) projects each tile's fixed `canvas` position through `cam`:
`sx=(canvas.cx − cam.cx + pdx)*cam.zoom + OUT_W/2` (image coords → y-up). **Depth parallax** `pdx` is
zero at a tile's own hold (so each composition is crisp when framed) and grows as the camera leaves —
NEAR tiles pan more than FAR. All tiles are always rendered, so adjacent-hold tiles bleed into the frame
edges (the "big canvas" behavior). There is **no camera pan during holds** — but each tile has its own
**micro-drift** (below) so holds never feel frozen.

### Bulge (center-scale magnification, CPU — no shader)
Tiles **swell uniformly** (aspect preserved, edges straight — NOT a lens-warp or dome) as they near the
frame center: `bulge = 1 + B·max(0, 1 − (dCur/BULGE_R)²)`, `B=params.bulge`, `dCur` = live screen-distance
from center, `BULGE_R≈720`. **Direct, not anchored** — earlier code divided by the bulge-at-the-tile's-hold
to keep holds at exact measured size, but that *canceled the visible swell* (most tiles sit near-center at
their hold). The user wanted the fisheye visible, so the magnification is now applied straight: the central
hero is genuinely larger at a hold, and tiles clearly grow as they cross the middle mid-pan.

### Dispersion (CPU, position)
As tiles pass `DISP_START` (~0.72 of the half-diagonal) toward the edge they get pushed **radially
outward**: `sx += (sx−cx)·disperse·max(0, rad−DISP_START)`. Zero inside the frame, so held compositions
stay tight; exiting tiles **spread apart** instead of moving in lockstep. `disperse=params.disperse`.

### Micro-drift (CPU, position)
Every tile drifts on its own small ellipse, always on (incl. holds), so nothing is ever static:
`drx = drift·dax·sin(loopProg·2π·dcx + dpx)`, `dry = drift·day·cos(loopProg·2π·dcy + dpy)`. `dcx,dcy` are
**integers** → seamless across the loop. `drift=params.drift`.

### Easing
Camera transitions use **`springEase`** = smootherstep + a terminal overshoot bump
(`t³(6t²−15t+10) + _OS·t²·sin²(πt)`). Slow start (NO recoil/anticipation dip), accelerate, **overshoot the
target ~9% near t≈0.78, then settle** — a spring *landing*, not a wind-up. `_OS` sets the overshoot. Zero
velocity at both ends → holds and the loop seam stay clean. (Earlier used easeInOutBack, but its
anticipation read as recoil — removed.) `srcEase` is still defined but no longer used for the camera.

### Pulse / Scale (render loop, live, no rebuild)
- Pulse: `1 + pulse*amp*sin(loopProg*2π*cyc + ph)`. `cyc` is an **integer** → seamless. Keep it integer.
- Global Scale multiplies **size only** (not position).

## Controls → params
| UI       | param          | notes |
|----------|----------------|-------|
| Speed    | `loopSec`      | total loop seconds; rebuilds timeline |
| Bulge    | `bulge`        | center fisheye magnification strength (live) |
| Parallax | `parallax`     | NEAR-vs-FAR pan separation (live) |
| Drift    | `drift`        | per-tile micro-drift amplitude — keeps holds alive (live) |
| Disperse | `disperse`     | how much exiting tiles spread radially outward (live) |
| Scale    | `scale`        | global size multiplier (live) |
| Pulse    | `pulse`        | independent per-tile scale swing (live) |
| Shuffle  | `seed`         | reseeds media→slot assignment + pulse/drift params → `rebuildScene()` |

Only Speed rebuilds the timeline; everything else is render-loop only (live, no rebuild).

## Media handling
- Images become `THREE.Texture(img)`; mp4s become `THREE.VideoTexture(video)` (muted/loop/autoplay,
  live playing tile). Cover-crop applied via `applyCover()`; videos re-crop once `videoWidth` is known
  (handled in `animate()` via the `cropped` flag).
- **No duplicate cards**: each uploaded media maps to exactly one slot. Slots are filled in a priority
  order (largest/central slots of each station first, interleaved) so sparse uploads still center well.
  Capacity = total slots across the 3 holds (currently 8+7+8 = 23). Extra uploads beyond capacity
  are not placed; fewer uploads leave some slots empty (no repeats).

## Export
`renderer.domElement.captureStream(30)` → `MediaRecorder`. Tries mp4 (avc1) first, falls back to webm.
`preserveDrawingBuffer:true` is required for capture. On export it does `tl.pause(0); tl.play();` and
records exactly `loopSec` — so the recording is one clean, seamless loop. Output: `drift-loop.mp4`/`.webm`.

## Invariants / gotchas
- Keep the loop seamless (see Motion model). Test by eye at the loop point, and remember pulse `cyc`
  must stay integer.
- Ortho camera is in **pixel space** (1920×1080), y-up; layouts convert with `y = OUT_H - hy*OUT_H`.
- `depthTest:false, depthWrite:false, transparent:true` on tile materials; layering is by `renderOrder = z`.
- Browser-storage APIs are not used and should not be (artifact constraint). No `<form>` posts.
- Holds are flat (camera still). Don't add motion during a hold span.

## Re-mapping a hold from a new source frame (the workflow used)
The `SCENES` numbers came from detecting tile bounding boxes in clean **held** frames; the `HOLDS`
camera path came from tracking global motion across all frames:
1. Extract frames: `ffmpeg -i source.mp4 -vf scale=1920:1080 frames/%04d.png`.
2. **Compositions:** for a chosen held frame, threshold non-white pixels, `scipy.ndimage` connected
   components, filter by area + fill ratio, take bounding boxes. Per box: `hx=cx/W, hy=cy/H,
   long=max(w,h), r=round(w/h,2)` (bucket to 0.80/1.0/1.33/1.5/1.78); `d` by size. → `SCENES[k]`.
3. **Camera path:** OpenCV ORB + `estimateAffinePartial2D` between consecutive frames, accumulate →
   absolute `cam.lookAt`/`zoom`; the low-velocity spans are the holds. → `HOLDS[k]` + `PHASE`.
   (Scripts used: `/tmp/track_camera.py`, `/tmp/track_green.py`, `/tmp/track_lime.py`.)

## Ideas / TODO (not yet built)
- Re-add the omitted fidelity extras as options: camera rotation (≤3.4°) and the mid-pan zoom-punch.
- Optional true `PerspectiveCamera` for real 3D parallax (vertex-shader sizing) — current ortho keeps
  composition matching exact and simpler; revisit if more parallax depth is wanted.
- Per-tile focal-point nudge for cover-crop (currently always center crop).
- Higher slot capacity / 4th station; per-station tile-count balancing for very sparse uploads.
- Server-side render path for frame-exact 1080p mp4 regardless of device (current capture is realtime).
