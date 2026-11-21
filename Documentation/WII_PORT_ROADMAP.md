# Wii port: roadmap

How to get from the universal build (`mvc2g`) to a Homebrew Channel
application, in milestones that can each be built and checked on their own.
Why this order: [WII_FEASIBILITY.md](WII_FEASIBILITY.md). Tools:
[WII_DEPENDENCIES.md](WII_DEPENDENCIES.md).

Written 2026-11-21. Nothing below has run on a Wii yet.

## Principles

- **Adapt, don't rewrite.** The game, its data layout, the renderer front
  end (`r_gl.c`, `r_mesh.c`, `r_char.c`) and the mixer stay as they are. Wii
  code lives in `port/wii/`. `src/` and `naomi/src/` are never touched
  (AGENTS.md).
- **The state hash is the referee.** Every milestone that changes how the
  module is built or run must still give the same per-frame state hash as
  the wasm (`build.sh`), first under qemu (32-bit big-endian PowerPC Linux),
  then on the Wii itself (hashes printed to a file on the SD card).
- **Measure on the console.** Speed figures come from the Wii's own timer
  (`gettime()` / `ticks_to_microsecs`), written to `sd:/mvc2/log.txt` with
  the date and what was running. Dolphin and qemu speeds are not evidence.
- **Arcade behaviour is the target** (AGENTS.md). A Wii-only change that
  makes the game play differently is not acceptable. Visual compromises
  forced by the hardware, such as colour depth, are recorded as deviations
  and go to the owner for a decision.
- **Nothing copyrighted** in commits or shared builds. Test builds read the
  user's disc image from SD/USB. An `EMBED=1` build stays in `private/`.

## Milestones

Each one lists what it delivers, how to check it, and what it depends on.
They are in order. The ones marked **gate** decide whether to carry on.

### W0: measurements on the desktop (no Wii needed) — **gate**

Answers the questions the feasibility study could not, because there was no
disc in the analysis session.

1. `build/port-generic/mvc2g_native DISC 20000 --fuzz 3 --mem`, plus one
   run each of the play-through scripts that load the most (versus with
   large characters, the ending): the module's memory high-water mark
   (`--mem` was added for this). **Pass: below about 48 MB**, so that it fits
   in MEM2 with room for libogc, IOS and the host's buffers.
2. The same 1200-frame state-hash comparison as `build.sh`, but with the
   PowerPC-Linux build of the translation under `qemu-ppc-static`
   (commands in WII_FEASIBILITY.md, appendix). **Pass: identical to
   `wasm.txt`.** This is the first test of a 32-bit big-endian target; the
   project has so far tested only 64-bit ones.
3. Instruction counts under callgrind (x86-64) for the game step alone
   (`--nodraw`), in fights. Gives the Wii a rough budget before there is any
   hardware figure.

Unblocks: W1. If (1) fails, W1.5 (memory reduction) comes before anything
else.

### W1: "hello, disc" — the smallest thing to run on a Wii

A `boot.dol` that:

- initialises video (text console), the GameCube pad and libfat;
- opens `sd:/mvc2/mvc2.iso` (or `usb:/mvc2/mvc2.iso`) and prints its size
  and the first sector's ISO9660 volume ID;
- allocates the module's memory in MEM2 and reports what it got;
- times a 1 MB sequential read and 100 random 2 KB reads;
- returns to HBC when START is pressed.

No game code. Proves the toolchain, HBC packaging (`apps/mvc2/boot.dol`,
`meta.xml`, an original `icon.png`), the storage path and the size of MEM2.
**Check:** runs on the console and the numbers are written to
`sd:/mvc2/log.txt`.

### W1.5: memory fit (only if W0.1 failed)

Make the module fit in about 48 MB without changing the game:

- size `INITIAL_MEMORY` to the measured high-water mark plus margin instead
  of 192 MB;
- look at which host-side buffers dominate (decoded textures in `swgl`,
  decoded sound banks in `hdbank.c`/`dcse.c`, disc read buffers in `fs.c`)
  and keep them smaller or compressed where it doesn't change output;
- the game's own 28 MB EE heap is the floor and is not touched.

**Check:** the `build.sh` hash comparison is unchanged and `--mem` is under
the limit across the W0 runs.

### W2: the game runs headless on the Wii — **gate**

`host_wii.c` with the `host_w2c.c` imports, the disc read on libfat, and the
module built with devkitPPC. No picture and no sound yet: it runs N frames
with `mvc2g_draw(0)` and writes the state hash of every frame and the time
per step to the SD card.

- **Check 1 (correctness):** the hashes equal `wasm.txt` from the desktop
  for the same disc and frame count.
- **Check 2 (speed, the key number of the project):** milliseconds per game
  step through boot, menus and a `--fuzz` fight, without drawing. **Pass:
  under about 10 ms**, which leaves room for audio and presenting a picture.
  If it is 10–16 ms, carry on, but the picture has to come from the GPU (W5)
  and audio has to be cheap. If it is over 16.7 ms, the game can't run at
  full speed on the Wii without speeding up the module itself (W2.5).

### W2.5: speeding up the module (only if W2 was slow)

In order of expected payoff, each measured on the console:

1. `-O2` vs `-Os` vs `-O3` on Broadway's 32 KB I-cache.
2. Bounds checks: wasm2c keeps one on every memory access on 32-bit
   targets. Allowing a build option that turns them off, which relies on the
   game never accessing memory out of bounds (as the 64-bit guard-page
   builds already do), is a measured decision, not a default.
3. `EMULATE_FUNCTION_POINTER_CASTS`: indirect calls go through adapters
   (port/CMakeLists.txt). Measure how many there are per step.
4. A profile of the hot spots on the console (the timebase register around
   the main subsystems), and only then targeted work.

### W3: picture, simple path

Show `mvc2g_canvas()` (drawn by `swgl` on the CPU) on the TV: convert the
RGBA8 canvas into the external framebuffer (YUV 4:2:2) or into a GX
texture drawn on one quad. Frame skip as in `host_sdl.c`: always 60 game
steps a second, draw only when there's time.

**Check:** title screen, menus and character select appear correctly on a
TV. Record pictures per second in a fight.

**Expectation:** `swgl` takes about 270 million instructions per fight
frame on x86-64. The Wii manages a few pictures per second at best. That's
enough to prove the pipeline and play menus, not to play fights. W3 is a
stepping stone, not the product.

### W4: GameCube controller

`PAD_ScanPads()` once per step. Map to PS2 pad bits as `host_sdl.c` does:
A = CROSS, B = SQUARE, X = CIRCLE, Y = TRIANGLE, L/R analogue past a
threshold = L2/R2, Z = R1, START = START, D-pad and main stick (dead zone)
= directions. Port 1 = player 1, port 2 = player 2. Button config stays the
game's own (it remaps PS2 buttons). Hot-plugging is handled by checking
`PAD_ScanPads`' connected mask.

**Check:** go through menus and select characters with the pad. Input lag:
the pad is read right before `mvc2g_frame()`, so it adds no frame of its
own. Measure end to end later with a camera if needed.

This can land before W3 (on the text console) if that helps testing.

### W5: picture from the GPU (GX) — the milestone that makes fights playable

Replace `swgl` underneath the existing renderer with Hollywood's GX:

- Build the module **without** `PORT_SWGL`, so the `gl*` calls `r_gl.c` and
  `r_mesh.c` make become imports of the module, the same way the browser
  build turns them into WebGL calls.
- `port/wii/gx_gl.c` implements the ~60 entry points in `platform/gl.h` for
  the three programs the renderer has: batch (2D sprites), mesh (3D
  stages, lit and fogged) and present. GLSL isn't interpreted. Each program
  is written once as GX state (vertex format, TEV stages, alpha compare,
  blend, z mode), the same way `swgl.c` does it in C.
- Textures: RGBA8 → GX RGBA8 (tiled); index textures → CI8 with a TLUT, or
  expanded to RGBA8 at upload. GX TLUTs are 16-bit (RGB5A3/RGB565/IA8), so
  CI8 loses colour depth against the arcade. **This is an accuracy decision
  for the owner** (feasibility §3). Character sprite pages that change every
  frame need a fast conversion path.
- The ADC triangle reject, the GS's affine colour interpolation and
  `swgl`'s texel tie rule have to be reproduced or recorded as differences.
- Vertex data is read out of the module's memory (mirrored on big-endian)
  and written to the GX FIFO.

**Check:** the screenshot comparison SWGL.md uses (share of pixels within 2
levels), against `swgl` output for the same frames, read back from the EFB
on the console. Record fight pictures per second. **Target: 60 pictures a
second in fights.**

Split it into sub-milestones: W5a the batch program (2D: menus, sprites,
HUD), W5b the mesh program (stages), W5c palette and fog fidelity.

### W6: audio

Copy `mvc2g_audio(800)` (48 kHz stereo, s16) into 32-byte-aligned DMA
buffers, swap to big-endian, flush the cache, and queue them to the audio
interface at 48 kHz (`AUDIO_InitDMA`/`AUDIO_StartDMA` with a completion
callback, or ASND with one voice). Three buffers of one step each (3,200
bytes, a multiple of 32) give about 50 ms of latency. When behind, keep the
mixer in step as `host_sdl.c` does (render and drop).

**Check (without hearing it, which isn't possible yet):** the DMA callback
count matches 60 per second over a minute. Buffer underruns are counted and
logged. The same samples are written to `sd:/mvc2/test.wav` for 10 s and
compared on a PC with the desktop's `--wav` output of the same frames,
which must be byte-identical. Listening tests come later.

Measure the mixer's cost per step on the console (`synth.c` has 48 voices
and a 16-tap sinc resampler). If it is too slow, the cheap fix is to run the
resampler only for voices whose rate isn't 48 kHz, after measuring.

### W7: timing and presentation

- Lock the loop to the TV's retrace (`VIDEO_WaitVSync`) at 59.94 Hz
  (NTSC/PAL60). The game runs at 60.0 steps a second on the arcade. Either
  accept a 0.1% slower game or drop one step every ~16.7 s. Record the
  choice.
- 480p when the console is set to it (component cable), otherwise 480i.
- 4:3. The game is 640x480.
- Audio rate drift against video: adjust by dropping or repeating one sample
  per buffer, not by resampling the game's output.

### W8: settings and saves

- Memory card persistence (`mc.c`) to `sd:/mvc2/memcard/`, same as the
  desktop's host directory.
- The settings store (`settings_get`) read from `sd:/mvc2/settings.ini`,
  so the arcade-rule switches can be set (the universal build has none
  today).

### W9: Wii Remote + Classic Controller (optional)

`WPAD_*` with the Classic Controller extension mapped like the GameCube
pad. A bare Wii Remote is a poor fit for a fighting game and isn't a goal.

### W10: Wii Menu channel (exploratory, last)

A forwarder channel that launches `apps/mvc2/boot.dol` from SD/USB. It
contains no game code or data. Installing channels needs a WAD manager and
a modified console. Document the risks (bricking, legality of the tools)
and leave it to the user. The project never distributes a WAD that contains
the game.

## What is reused and what is new

| Component | Reused as is | Wii-specific |
|---|---|---|
| Game code and data layout | ✅ `port/host`, `port/stubs`, glue | — |
| wasm module and translation | ✅ `build.sh`, `fix_w2c.py` | devkitPPC compile flags |
| Host imports (WASI, disc, m_get/m_put) | ✅ `host_w2c.c` (included) | libfat file I/O |
| Main loop, frame skip | pattern from `host_sdl.c` | libogc timers, VSync |
| Renderer front end | ✅ `r_gl.c`, `r_mesh.c`, `r_char.c`, `r_font.c`, `decode.c` | — |
| Rasteriser | `swgl.c` (W3 only) | `gx_gl.c` (W5) |
| Mixer | ✅ `port/audio/*` | AI/ASND output |
| Input | mapping from `host_sdl.c` | PAD_* (and WPAD_*) |
| Display | — | VIDEO_*/GX, XFB |
| Saves, settings | `mc.c` | SD paths |
| Tests | ✅ state-hash comparison | hashes written to SD, compared on the PC |

## Effort, roughly

These are guesses, not measurements, for someone who knows libogc:
W0 a day; W1 a day or two; W2 a few days (most of it build plumbing); W3 and
W4 a day each; W5 two to four weeks (the real work); W6 a few days; W7–W8 a
week together.
