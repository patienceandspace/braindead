# Braindead

A fork of [Trippy Wormhole](https://github.com/patienceandspace/wormhole), taken
at the point where the engine became deterministic and export-capable. Single
self-contained HTML file — no build step, no dependencies, no audio assets.

Not yet deployed. Everything below is inherited from the parent and describes
the fork's starting point, not its destination.

> **Where this is going:** TBD. The fork exists so the wormhole can stay as it is
> while this one diverges.

## Controls

| Input | Effect |
| --- | --- |
| Hold click / touch | Warp Drive — ~10x speed, inverted colors, runaway delay feedback, gear-shift sweep |
| Left / Right arrows | Throttle toward or away from the vanishing point (0.15x–5x) |
| `D` | Reopen the ride designer mid-flight |

The **Seed** field makes a ride reproducible — the same seed replays the same
stars, ring styles and glyph transmissions exactly.

## Design Your Ride

The opening panel exposes seven parameter lanes: Speed, Focal Drift, Camera Shake,
Color Flow, Ring Frequency, Chaos, and Ring Style. Each lane runs in one of three
modes — `auto` (the engine's own organic behavior), `keys` (drag your own keyframe
curve), or `lfo` (drag for rate and depth). Loop length is selectable from 2 to 30
minutes.

*Polite random* seeds a unique but civilized ride. *Fuck my shit up* does not.

## Ring styles

Every so often one ring in the tunnel is different. Eight kinds of different:
ghost, complement flare, jagged static, chromatic echo, hard polygon, whispered
glyph transmission, upstream swimmer, and failing neon. One style is dominant at a
time, with occasional strays and slower shifts between dominants. The interval
between anomalies breathes between metronomic and chaotic.

## Exporting a 4K60 video

Use **Chrome or Edge** — the frame export relies on the File System Access API.

Everything is deterministic: `stepSim()` advances the simulation exactly one frame
at a time and every random draw comes from a seeded PRNG, so a given
(seed, frame) always produces the same image. That means the preview and the
exported sequence are identical, and export is **frame-driven, not clock-driven** —
each 4K frame takes as long as it needs to encode without dropping or duplicating.

Drawing is resolution-independent: absolute sizes are authored against a 1080p
reference and scaled by `S = H/1080`, so a 4K frame is the composition enlarged,
not a 1080p composition with hairline strokes stranded in a bigger canvas.

1. Set seed, size, fps, duration and format in the **Export** panel.
2. Hit **estimate** for measured frame count, disk use and time on *your* machine.
3. **export frames** → pick a folder → `braindead_00000.png` … lands in it.
4. **export wav** renders the soundtrack separately (an image sequence has no
   audio track) through an `OfflineAudioContext`, seeded identically so it lines
   up with the frames.
5. Mux with ffmpeg:

```bash
ffmpeg -framerate 60 -i braindead_%05d.webp -i braindead-audio.wav \
  -c:v libx264 -pix_fmt yuv420p -crf 16 -c:a aac -shortest braindead.mp4
```

### Frames or a single video file?

Two export paths, both **frame-driven** — the encoder is handed frames with
explicit timestamps, so neither drops nor duplicates however slow encoding gets.

> Deliberately *not* offered: `MediaRecorder` + `captureStream()`. It records in
> wall-clock time, so when a 4K render can't keep up it silently drops frames and
> bakes judder into the file. Frame-exactness is the whole point here.

**export video** (WebCodecs) is roughly 7× faster and much smaller, but it can
never be lossless — see the chroma ceiling below. Measured on this piece, PSNR
against the source pixels at 4K, luma and chroma reported separately:

| Method | Size / 30 s | Luma | Chroma |
| --- | --- | --- | --- |
| **PNG frames** | ~20 GB | **∞** | **∞** |
| H.264 1600 Mbps | 5.5 GB | 54.6 dB | 44.0 dB |
| H.264 800 Mbps | 3.3 GB | 50.5 dB | 43.2 dB |
| WebP q0.95 frames | 3.3 GB | \~40 dB | \~40 dB |
| H.264 400 Mbps | 1.6 GB | 46.4 dB | 41.8 dB |
| H.264 150 Mbps | 0.6 GB | 40.8 dB | 39.7 dB |

Two findings worth keeping:

**Chroma is the ceiling, and bitrate cannot buy past it.** Browsers only encode
4:2:0 — three quarters of the colour resolution is thrown away *before* the
encoder runs. Push the bitrate and luma climbs freely (40 → 55 dB) while chroma
crawls (39.7 → 44 dB) and then stops. At 1080p, 600 and 1200 Mbps produced
byte-identical output: the encoder saturates and silently discards the extra
budget. This matters here more than for most footage — the piece is saturated
neon on black, and thin bright colour edges are exactly what 4:2:0 smears.
Only AV1 High offers 4:4:4 in Chrome (H.264 High444PP and VP9 profiles 1/3 are
unsupported), and Chrome's AV1 encoder saturates early enough that it measured
*worse* than H.264 anyway — hence H.264 as the default.

**Lossy video beats lossy frames per byte.** At the same 3.3 GB, H.264 at
800 Mbps scores ~50 dB luma against WebP q0.95's ~40 dB. WebP is itself 4:2:0,
so it buys no colour advantage. If you're not going lossless, the video path
strictly dominates — reach for WebP frames only when you need *frames*.

So: **PNG frames for a true master** (the default), H.264 at the auto-suggested
bitrate when a 20 GB folder isn't practical. Note also that PSNR *understates*
the cost on this content: smooth gradients band, and banding reads worse to the
eye than its error magnitude suggests.

### Frame format

This piece is glow-heavy, which PNG compresses badly — a 4K frame runs ~11 MB.
That is the price of a lossless master, and PNG is the default for that reason.
WebP (~6× smaller) and JPEG (fastest) are there for drafts and dailies. Budget
**gigabytes** either way: 30 s of 4K60 is 1800 frames.

Encoding, not rendering, is the bottleneck (a 4K frame paints in well under a
millisecond and encodes in seconds), so frames are handed to a pool of Web
Workers that encode in parallel while the main thread paints ahead and writes
finished blobs. In-flight work is bounded to the pool size — unbounded dispatch
would queue thousands of 33 MB bitmaps and exhaust memory.

## Notes for future work

- Modulators run on `wt`, a wall clock deliberately independent of `t` (the travel
  clock). They must stay that way: `t`'s increment depends on `drift()`, so if
  `drift()` read `t`, a zero speed would freeze `t` and with it the very
  oscillation that would bring the speed back — a permanent dead stop.
- Designer lanes drive visuals; audio follows only the warp lane, and otherwise
  evolves on its own.
- Throttle (arrow keys) is a live-only control and is pinned to 1 during export.
  Warp is exportable — automate the **Warp Drive** lane instead of holding the
  mouse.
- `PREROLL` frames are rendered before frame 0 so exports open with motion trails
  already built up rather than fading in from black.

## Local preview

```bash
npx serve -l 3458 .
```
