# CLAUDE.md — Drift (moodboard motion tool)

## What this is
A single-file, all-white minimal web tool that turns a set of uploaded images/mp4s into a
looping 16:9 "moodboard motion" clip. Three motion modes (see "Animation modes"), each traced
frame-by-frame from a reference clip: **Holds** (default — one big canvas, a single camera pans
between three held framings), **Pan** (one continuous left→right parallax sweep, wrapping layers),
**Zoom** (perspective dolly-in with randomly spawning tiles), and **Stack** (a photo pile — full-bleed
cards deal in one at a time, instant entrances). Depth parallax + a center-scale
bulge add dimensionality; optional top-right typewriter text overlay; white/black background toggle.
Exports an mp4 (or webm fallback) and previews live on canvas. The sections below describe the
Holds source-tracing in detail; the other modes' measurements live in "Animation modes".

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
- Transitions use a **decel-biased ease-out** (`panEase`): gentle build, speed peaks early (~40%), then a
  long deceleration into the hold. **No overshoot, no spring-back** — zero velocity at both ends so holds/
  seam stay clean. The carry-through *after* arrival comes from per-tile **momentum**, not a camera spring.
  (An earlier `springEase` added a terminal overshoot bump — it read as the camera "coming back into
  position" and was removed. See "Easing".)
- **Bulge = ripple-scale (current), NOT distortion.** Cards must keep their exact shape — **no pixel
  warp**. The bulge is a **per-tile uniform SCALE** keyed to the tile's live distance from frame-centre:
  a card swells as it rides over the centre and shrinks as it leaves, like passing over a wave/ripple.
  Aspect stays locked, edges stay straight. (History: a post-process *lens shader* was tried — it warped
  the pixels into a fisheye dome — and **explicitly rejected** by the user: "Images/cards should stay in
  their shape. No distortion. Only the scale goes up and down as it passes through the center." Do NOT
  reintroduce the lens/shader.) See "Bulge".
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
- `srcEase` (legacy, unused) / `panEase` (decel-biased ease-out — the camera transition ease).
- `initGL()` — Three.js setup: `WebGLRenderer({preserveDrawingBuffer:true})`, **OrthographicCamera in
  pixel space** (y-up: top=OUT_H, bottom=0), unit `PlaneGeometry`. Starts `animate()`.
- `addFiles / media / addThumb / removeMedia` — upload handling + thumbnail tray.
- `applyCover()` — sets texture `repeat`/`offset` to center-crop to a tile's ratio.
- `rebuildScene()` — assigns media→slots and builds tile meshes at FIXED canvas positions (uses `HOLDS`).
- `buildTimeline()` — the GSAP timeline: tweens the single `cam={cx,cy,zoom}` through `HOLDS` per `PHASE`.
- `animate()` — per-frame render loop: projects each canvas tile through the (momentum-)lagged `cam`
  (+ parallax + drift + ripple-bulge scale + pulse), single `renderer.render(scene,camera)` to the canvas.
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
  par,                  // DEPTH_PAR[d] parallax gain (NEAR +, FAR −)
  amp,cyc,ph,           // independent pulse: amplitude, integer cycles/loop, phase
  dax,day,dcx,dcy,dpx,dpy,   // micro-drift ellipse: amplitudes, integer cycles, phases (seamless)
  clx,cly, inertia }    // momentum: lagged camera (trails real cam) + size-based inertia (bigger=lazier)
```
There is **no per-tile enter/exit pose and no opacity animation** — tiles are always opaque at a fixed
canvas spot; the camera framing alone reveals/hides them. Render math (in `animate`): project
`canvas` through `cam` with parallax `pdx=(homeLookAt−cam.c)*par*parallax` (zero at the tile's hold), then
`size = canvas.w * cam.zoom * scale * pulse * bulge` (bulge = per-tile dome-scale, see Bulge). Each tile
also has a per-tile **lagged camera** `(clx,cly)` for directional **momentum** — see Motion model.

## Motion model (buildTimeline) — camera pan over one canvas
The only animated thing is `cam={cx,cy,zoom}`. `buildTimeline` tweens it through `HOLDS` at `PHASE*L`
times: each transition is one `panEase` tween from one hold's `{lookAt,zoom}` to the next; **hold spans
have no tween** (camera sits still — tiles stay alive via momentum coast + ambient drift, not camera
motion). A 0.001s pin tween at `L-0.001` fixes total duration = `L`.

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

### Bulge (per-tile dome-scale, CPU — NO distortion)
A **pure per-tile uniform scale** in the render loop — the card keeps its exact shape, only its size
changes with how close its screen centre is to the frame centre (riding over an invisible 3D dome):
```
dist  = hypot(sx−HW, sy−HH)                                      // live screen-distance from frame-centre
bulge = max(0.12, 1 + b·(exp(−(dist/BULGE_SIG)²) − BULGE_C0)/(1−BULGE_C0))
sc    = sizeMul · scale · pulse · bulge                          // applied to size only (not position)
```
The profile was **measured from the source** with camera zoom cancelled by pairwise differencing
(subtract two tiles' log-size changes over the same frames — zoom drops out exactly; 1527
constraints, 20 distance bins). The measured curve is a **GAUSSIAN DOME**, not linear and not a
hard perspective sphere: a **flat crown** (tiles hold their peak size crossing d≈0–100, no cusp),
steepest shrink mid-screen (~550–650px), and a **gentle tail** (≈0.56× at d≈875, still easing).
Constants: `BULGE_SIG=579` (dome width σ), `BULGE_C0=exp(−(481/579)²)≈0.502` → bulge is exactly 1
at the **neutral ring d=481px**. `b=params.bulge` = peak swell at dead-centre (measured **0.54**
→ default 55%); far floor eases toward 1−b (clamped at 0.12). **No pixel warp, no shader, no
render target** — just `t.mesh.scale`. **Do not** replace this with a lens/post-process
distortion: the user explicitly wants cards to stay rectangular (a fisheye-lens version was built
and rejected). Applies to size only, so it never moves a tile or breaks the seam. (History: a
linear falloff was used before re-measurement; it overshot the peak and over-shrank edge tiles.)

### Dispersion (CPU, position)
As tiles pass `DISP_START` (~0.72 of the half-diagonal) toward the edge they get pushed **radially
outward**: `sx += (sx−cx)·disperse·max(0, rad−DISP_START)`. Zero inside the frame, so held compositions
stay tight; exiting tiles **spread apart** instead of moving in lockstep. `disperse=params.disperse`.

### Micro-drift (CPU, position) — slow ambient float
Every tile drifts on its own small ellipse, always on (incl. holds), so nothing is ever static:
`drx = drift·dax·sin(loopProg·2π·dcx + dpx)`, `dry = drift·day·cos(loopProg·2π·dcy + dpy)`. `dcx,dcy` are
**integers** → seamless across the loop. `drift=params.drift`. **Kept LOW frequency (1–2 cycles/loop) on
purpose:** momentum (below) now provides the post-pan glide, so drift must be slow enough that its velocity
never *fights/reverses* the coast at a hold entry. (An earlier 2–4 cycle version made holds feel crude —
the fast ellipse reversed direction just as the pan settled.) The post-pan glide itself is **Momentum** —
see its subsection below.

### Easing
Camera transitions use **`panEase`** — a **decel-biased ease-out** with **no overshoot**. It's the integral
of a velocity bell `v(p)=p²·(1−p)³` (peak speed at p≈0.4), precomputed into a cumulative table `_pcum`: the
pan builds gently, peaks early, then has a long smooth deceleration into the hold. Zero velocity at both
ends → no jerk leaving a hold, clean arrival, clean loop seam. The "land and settle" feel comes from the
per-tile **momentum** coast, NOT a camera spring. (History: `springEase` added a terminal overshoot bump
that read as the camera "coming back into position" at each hold — removed. Even earlier `easeInOutBack`'s
anticipation read as recoil — also removed. `srcEase` legacy table is still defined but unused.)

### Momentum (directional inertia, render loop, live)
Each tile projects through a **lagged camera** `(clx,cly)` that eases toward the real `cam` every frame:
`clx += (cam.cx − clx)·fol`, `fol = max(0.03, (1 − momentum·0.85)^(1 + inertia·1.6))` — small `fol` = a
**long, monotonic coast** (exponential approach, never overshoots → never reverses). So during a pan the
tile **trails** the camera, and when the pan lands it **coasts into place and settles** — true directional
momentum, not the symmetric drift ellipse. `inertia = clamp(longSide/1100, 0.25, 1.3)` → **bigger tiles
carry more momentum** (lazier follow, longer settle). Default `momentum=0.6` → big tiles coast ~0.4s into
the hold (the carry-through), small tiles settle quicker; all fully settle within each ~1.5s hold. Parallax
still keys off the *true* `cam`, so each composition resolves crisp once the lag settles. `momentum=0` →
`fol=1` → exact follow → identical to the old no-momentum behavior. **Seam:** the lag carries state across
the loop, so export (`exportBtn`) first runs `warmMomentum()` — two synchronous passes simulating the lag
against the periodic camera path — to seat `clx/cly` on the steady-state orbit before recording, so the
recorded loop closes seamlessly. Preview self-converges (it loops continuously).

### Pulse / Scale (render loop, live, no rebuild)
- Pulse: `1 + pulse*amp*sin(loopProg*2π*cyc + ph)`. `cyc` is an **integer** → seamless. Keep it integer.
- Global Scale multiplies **size only** (not position).

## Controls → params
| UI       | param          | notes |
|----------|----------------|-------|
| Speed    | `loopSec`      | total loop seconds; rebuilds timeline |
| Bulge    | `bulge`        | per-tile ripple-scale: peak swell as a card rides over frame-centre (live, no distortion) |
| Parallax | `parallax`     | NEAR-vs-FAR pan separation (live) |
| Drift    | `drift`        | per-tile micro-drift amplitude — keeps holds alive (live) |
| Momentum | `momentum`     | per-tile lagged-camera inertia — tiles coast/settle after a pan; scales w/ tile size (live) |
| Disperse | `disperse`     | how much exiting tiles spread radially outward (live) |
| Scale    | `scale`        | global size multiplier (live) |
| Pulse    | `pulse`        | independent per-tile scale swing (live) |
| Shuffle  | `seed`         | reseeds media→slot assignment + pulse/drift params → `rebuildScene()` |

Only Speed rebuilds the timeline; everything else is render-loop only (live, no rebuild).
Plus: **Motion** (`params.mode`, segmented Holds/Pan/Zoom — switching applies `MODE_DEFAULTS` for
bulge/drift/pulse/bg then rebuilds), **Background** (`params.bg`, White/Black, live in all modes),
**Text / Text X / Text Y** (`params.text/textX/textY` — top-right overlay, see below).

## Animation modes (Motion segmented control)
`params.mode` selects one of three motion grammars; Holds is the original and the default.
Each mode has its own layout builder + timeline + projection branch; the per-tile pipeline
(drift → dispersion → bulge → pulse → scale) is shared.

### Pan (left→right parallax sweep) — traced from a MASP signage mockup clip
- ONE continuous eased sweep per loop, **no holds**: `cam.cx` 0→`PAN_TRAVEL` (=2.6 screen-widths,
  measured) via `sweepEase` (integral of a Gaussian velocity bell σ≈0.195 — zero velocity at both
  ends; measured progress 6%/48%/94% at p=.25/.5/.75). `cam.zoom=1`, no vertical motion.
- **3 constant-gain parallax layers** (measured R²>0.99 linear): `gain=1+PAN_GAIN[tier]*parallax`
  → 1.09/0.91/0.65 at the default parallax 0.60. Tiles wrap modulo their layer period
  `P=gain*PAN_TRAVEL` (`wrapPan`) — displacement over one loop = exactly P → **seamless**.
  `PAN_GAIN_MIN=0.45` floors the gain so P always exceeds screen+tile width (wrap jumps stay
  offscreen at any Parallax setting). Layout: `buildPanTiles` — procedural seeded slots, 8 per
  tier, 4:5-dominant `PAN_RATIOS`. Momentum applies to the pan scalar (`clx`).

### Zoom (parallax dolly-in, random spawns) — traced from the Sana AI Summit promo clip
- Camera flies forward at **constant speed** (timeline is just a linear progress tween; measured:
  no easing). Per tile: `u=(loopProg+phase)%1`, growth `g=1/(1−K·u)` — the measured TRUE
  PERSPECTIVE law (fits rmse<1.6px; exponential fits worse). Size AND radial offset from
  frame-centre both scale by g; per-tile `K` (depth) spread = the parallax (Parallax slider
  spreads K around `ZOOM_KBAR=1.35`, clamped ≥1.02).
- Tiles **pop in instantly** (no fade/scale-in — measured) small (long 75–140px) on a seeded
  spawn ring (`d0` 170–430px from centre, floored vs tile size so a tile can never engulf the
  camera), fly outward, exit, stay offscreen (g clamped at 40) until their phase wraps →
  **exactly periodic, seamless**. K≥1.02 keeps the perspective singularity inside the loop =
  guaranteed exit. **No momentum in this branch** (a lag would smooth the u-wrap and drag tiles
  backward at respawn). `renderOrder = K*100` (nearer on top). Zoom defaults: black bg,
  bulge/drift/pulse 0.

### Stack (photo-pile card deal) — traced from the NYPL Sana AI Summit promo clip
- Large axis-aligned **full-bleed** cards (long side ≈0.38–0.75·H rendered, centre-biased spots)
  land one at a time on a pile; each new card renders on top (`renderOrder = dealIndex+10`); old
  cards never move or fade. **Entrances are INSTANT** (measured: single-frame events — no slide/
  scale/fade tween): the render loop toggles `mesh.visible` when `loopProg ≥ dealAt`. First card
  present at the seam; the rest spread over the first ~93% of the loop (`buildStackTiles`); the
  pile **hard-resets at the seam by design** (the reference does not loop seamlessly — same
  accepted pattern as the typewriter text).
- The reference's drop shadows and white-matte "speaker cards" were built and then **dropped by
  user choice** — cards are pure edge-to-edge media, no shadow, no matte. Don't reintroduce them
  unprompted.
- Stack defaults: bulge/drift/pulse/disperse 0, white bg — cards are static once dealt; only
  their video content plays. No cam, no momentum in this branch. `MODE_DEFAULTS` now carries a
  per-mode `disperse` too.

## Background + text overlay
- `params.bg` ('#fff'/'#000') → `renderer.setClearColor`; mode switch applies the mode's default
  (`MODE_DEFAULTS[mode].bg` — zoom is black like its reference), user can override live.
- Top-right **text overlay** (`initText/drawText/updateText`): drawn into a `THREE.CanvasTexture`
  plane (`renderOrder:1000`) so it is captured by export (an HTML overlay would NOT be). Left-aligned
  block; `textX` = inset from the RIGHT edge to the block's left edge, `textY` = inset from top.
  **Typewriter per loop**: ~14 chars/s starting 1s in, blinking block cursor ~1Hz; restarts each
  loop (intentional — a type-on cannot also persist across a seamless loop; the reference reads the
  same way). Text color auto-flips with bg (white on black / ink on white). Empty textarea hides it.
  Texture redraws only when char-count/blink/content/bg change (`_textKey`).

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
- Holds are flat **for the camera** (no `cam` tween during a hold span). Tile-level life during a hold is
  expected and intentional — the momentum coast settling + the slow ambient drift. Don't animate `cam`
  during a hold; do keep the per-tile motion.

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
