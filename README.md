# Drift — moodboard motion

A single-file, all-white web tool that turns a set of uploaded images/mp4s into a looping 16:9
"moodboard motion" clip. It lays every tile on one large canvas and pans a camera across it, pausing on
**three held compositions**, then exports a seamless mp4 (or webm fallback).

The motion was reverse-engineered frame-by-frame from a reference clip: the camera path, the three hold
framings, the center "bulge", and the easing were all measured from the source rather than guessed.

## Use it

No build step. Either:

- Open `index.html` directly in a browser, or
- Serve the folder and open it:
  ```bash
  python3 -m http.server
  # then open http://localhost:8000/index.html
  ```

Drop in images / mp4s (or click to browse), then **Export mp4**. Three.js and GSAP load from a CDN at
runtime, so an internet connection is required.

## Controls

| Control  | Effect |
|----------|--------|
| Speed    | total loop length |
| Bulge    | center fisheye-style magnification (tiles swell as they cross the middle) |
| Parallax | how much NEAR tiles pan more than FAR ones |
| Drift    | per-tile micro-drift so holds never feel frozen |
| Disperse | how much tiles spread apart as they exit the frame |
| Scale    | global size multiplier |
| Pulse    | independent per-tile scale breathing |
| Shuffle  | reshuffle which media lands in which slot |

## How it works

See [`CLAUDE.md`](./CLAUDE.md) for the full design notes — the camera-pan-over-one-canvas model, the
traced `HOLDS`/`PHASE` keyframes, the depth parallax, the measured center bulge, the springy
(recoil-free) easing, and the seamless-loop invariants.

## Stack

- [Three.js](https://threejs.org/) r128 (WebGL renderer, orthographic camera in pixel space)
- [GSAP](https://gsap.com/) 3.12.5 (the camera timeline)
- `MediaRecorder` + `canvas.captureStream()` for export
