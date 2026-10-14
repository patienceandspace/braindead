# Braindead

A fork of [Trippy Wormhole](https://github.com/patienceandspace/wormhole), taken
once its engine became deterministic and export-capable. Right now it is a
faithful copy under a new name — **the creative direction is still undecided.**
Ask before assuming where it's headed.

Repo: https://github.com/patienceandspace/braindead
Everything lives in a single self-contained `index.html`: no build step, no
dependencies, no audio assets. Keep it that way unless there's a reason not to.

Dev server: `npx serve -l 3458 .` (port 3458, so it can run beside the wormhole
on 3457).

## Load-bearing invariants

These are not style preferences. Each one cost real debugging or measurement.

**Never deploy without checking `.netlify/state.json` first.** This fork is
deliberately unlinked. Its parent's link points at `trippywormhole`, and a
careless deploy from here would silently overwrite that live site. It has
already happened once in the parent repo — a stale `siteId` sent a deploy to the
wrong site.

**`wt` must never be derived from `t`.** `wt` is the wall clock driving all
modulators; `t` is the travel clock. `t`'s increment depends on `drift()`, so if
`drift()` read `t`, a speed of zero would freeze `t` — and with it the very
oscillation that would bring the speed back. The wormhole used to grind to a
permanent dead stop this way. Speed is also clamped to a 0.2 floor for the same
reason.

**Export is frame-driven, never clock-driven.** `stepSim()` advances exactly one
frame per call and every random draw comes from a seeded PRNG, so a given
(seed, frame) always yields identical pixels — that is what makes preview and
export match. Do not add `MediaRecorder` + `captureStream()`: it records in
wall-clock time and silently drops frames when a 4K render falls behind, baking
judder into the file. This omission is deliberate.

**Warp inversion must not touch the scene canvas.** The translucent backdrop
fill is what produces the motion trails, so each frame builds on the last.
Inverting in place would feed back. Invert into a scratch canvas instead
(difference-blended white, which matches CSS `invert()` exactly).

**Drawing is resolution-independent.** Absolute sizes are authored against a
1080p reference and scaled by `S = H/1080`. Anything new that is sized in pixels
needs the same treatment or it will look wrong at 4K.

## Settled questions — don't re-research these

- **Video can never be lossless here.** Browsers encode 4:2:0 only; chroma
  saturates around 44 dB at 4K no matter the bitrate. At 1080p, 600 and
  1200 Mbps produced byte-identical output. PNG frames are the only true master.
- **No 4:4:4 path is worth taking.** H.264 High444PP and VP9 profiles 1/3 are
  unsupported in Chrome; AV1 High is the only 4:4:4 option and its encoder
  saturates early enough to measure *worse* than H.264.
- **Lossy video beats lossy frames per byte.** At equal size, H.264 ~50 dB luma
  vs WebP q0.95's ~40. Reach for WebP frames only when you need frames.

Full measurements are in `README.md`.
