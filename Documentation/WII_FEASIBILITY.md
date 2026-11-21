# Wii port: feasibility

Can the universal build (`mvc2g`, [port/generic](../port/generic/README.md))
become a native Wii Homebrew Channel application? This document answers the
ten questions of the brief from the repository and from what could be built
and measured without a Wii or a disc. Plan:
[WII_PORT_ROADMAP.md](WII_PORT_ROADMAP.md). Tools:
[WII_DEPENDENCIES.md](WII_DEPENDENCIES.md).

**Date:** 2026-11-21. **Branch:** `wii-port-feasibility`.

**What this session had:** a Linux x86-64 cloud machine (Intel Xeon, nominally
2.3 GHz), Emscripten 6.0.13, wabt 1.0.34, GCC 13 for `powerpc-linux-gnu`, and
qemu-user 8.2.

**What it didn't have:** a Wii, devkitPPC/libogc, or a disc image. So nothing
here ran on a Wii, the game never booted past "disc not recognised", and every
Wii speed figure is an **estimate** from instruction counts. Estimates are
labelled as such.

## Summary

| # | Question | Answer |
|---|---|---|
| 1 | Does the universal build compile for the Wii's PowerPC? | **Yes, measured.** The full 48.6 MB wasm2c translation compiles for 32-bit big-endian PowerPC 750 with GCC 13 and no source changes, and runs its initialiser under qemu |
| 2 | Dependencies incompatible with the Wii? | **None in the module.** It imports 13 trivial functions. The host side needs `mmap`-free memory, which wasm2c already picks on 32-bit, and libfat instead of stdio paths |
| 3 | Can the renderer work on Wii hardware? | **Yes, but only on the GPU.** The renderer's three programs map onto GX fixed-function state; `swgl` is too slow (row 4). This is the biggest piece of new work |
| 4 | Can the software renderer reach acceptable speed? | **No (estimate).** About 340 million PowerPC instructions per fight-like frame, i.e. roughly 0.5–0.7 s per picture on Broadway. That's 1–2 pictures per second, about 30× short of 60 |
| 5 | Toolchain? | devkitPPC + libogc + libfat (devkitPro). The existing Emscripten → wasm2c pipeline stays |
| 6 | Controller input? | GameCube pads through libogc `PAD_*`, mapped to the PS2 pad bits the module already takes (`mvc2g_pad`). Straightforward |
| 7 | Audio? | 48 kHz stereo s16 from `mvc2g_audio(800)` into AI DMA (or ASND). Straightforward; can be checked byte for byte without listening. Mixer cost on Broadway is unmeasured |
| 8 | Showing the 640x480 picture? | With GX rendering it lands in the EFB and is copied to the XFB, which is the normal Wii path. For the interim `swgl` path, a CPU RGBA→YUV conversion or a GX texture quad |
| 9 | Loading the user's disc image from SD/USB? | libfat `sd:/` / `usb:/` + `fopen`. `page_disc_read` is already a 20-line function over `FILE*`. The image is ~210 MB, well under FAT32's limit |
| 10 | Big architecture changes? | **No for the game. Yes for the picture:** a GX backend under the existing `gl*` layer. Possibly memory reduction (row "Memory" below) |

**The two open risks**, which the roadmap's first gates (W0, W2) are there to
settle:

1. **Memory.** The module's memory must be one contiguous block. On the Wii
   that means MEM2: 64 MB, of which homebrew can use about 52 MB. The module
   currently reserves 192 MB, and its real high-water mark was not measured
   (no disc). The floor is about 36.4 MB before any port-side buffer
   (§ Memory).
2. **CPU time for the game step itself.** On x86-64 the game step plus mixer
   is about 1.5 ms. Scaled to Broadway, that is somewhere around 15–30 ms
   against a 16.7 ms budget (estimate, § CPU). It could fit, or it could be
   up to twice too slow. Only a measurement on the console decides it.

## 1. Compiling for the Wii's CPU (measured)

The Wii's CPU (Broadway) is a 729 MHz PowerPC 750CL: 32-bit, big-endian,
hardware float, 32 KB + 32 KB L1, 256 KB L2.

The project already designed for this case:

- the game's 32-bit pointers become offsets into one memory block, so pointer
  size doesn't matter;
- `-DWABT_BIG_ENDIAN=1`, `fix_w2c.py` and the host's `m_get`/`m_put` handle
  big-endian, verified on s390x (64-bit) under qemu (README, 2026-11-13);
- `-ffp-contract=off` is required on CPUs with fused multiply-add, which
  includes PowerPC.

What was done here (2026-11-21):

| Step | Result |
|---|---|
| `ninja -C build/port-em mvc2_host` (Emscripten 6.0.13, 18,571 steps) | built |
| `port/generic/build.sh` (link, wasm2c, x86-64 build) | built in 2 min 40 s; `mvc2g.wasm` 3.37 MB, `mvc2g_w2c.c` 48.6 MB. Hash comparison skipped (no disc) |
| `powerpc-linux-gnu-gcc -O2 -mcpu=750 -ffp-contract=off -DWABT_BIG_ENDIAN=1` on the translation, `wasm-rt-impl.c` and `host_w2c.c`, static link | **compiled and linked, no source changes, no errors** (6 min for the 48.6 MB file) |
| Code size of the translation, PowerPC | **17.6 MB at `-O2`, 10.8 MB at `-Os`** (`.text`) |
| `qemu-ppc-static -cpu 750cl mvc2g_ppc fake.iso 10` | instantiates (allocates the 192 MB memory), runs `_initialize`, `mvc2g_boot` rejects the zero-filled image and the program exits 2, **the same as the x86-64 build on the same file** |
| `swgl_bench` (the rasteriser alone, no disc needed) on PowerPC under qemu | **checksum `023144b8`, identical to x86-64.** `swgl` is endian-clean on this case |

What wasn't done: the 1200-frame state-hash comparison on 32-bit big-endian.
It needs the disc. It is roadmap step W0.2, and nothing in this report
replaces it. The project has tested 64-bit big-endian, never 32-bit.

Why not compile the port natively for PowerPC (like the i386 desktop build)?
The game's C reads little-endian disc data and structures directly. A native
big-endian build would need byte swaps throughout the game. wasm2c's mirrored
memory avoids all of that, and the project already depends on it.

**Code placement.** At `-Os` the code (≈11 MB) plus libogc and buffers fits
in MEM1 (24 MB) with room left. At `-O2` (≈18 MB) it is tight. Whether `-O2`
is faster than `-Os` on Broadway's 32 KB I-cache is a W2.5 measurement. HBC's
acceptance of a DOL this large has to be checked in W1.

## 2. Incompatible dependencies

The module is self-contained. Its 13 imports (README "What a device
supplies"): `page_disc_read`, WASI `fd_write` (log), `clock_time_get`,
`environ_*`, `proc_exit`, three unused file calls, and an Emscripten asm stub.
`host_w2c.c` implements all of them in plain C. On the Wii:

| Host need | Wii |
|---|---|
| `fopen`/`fseeko`/`fread` on the disc image | libfat devoptab (`sd:/`, `usb:/`), same calls |
| `calloc` of the module memory | works; on 32-bit wasm2c picks `WASM_RT_USE_MMAP=0`. The block must come from MEM2 (§ Memory) |
| `clock_gettime(CLOCK_MONOTONIC)` (test timing only) | libogc `gettime()` / `ticks_to_microsecs` |
| `setjmp`/`longjmp` (wasm2c traps) | newlib on devkitPPC has them |
| Signals | not needed: 32-bit builds use explicit bounds checks, not guard pages |
| Threads | none; the module is single-threaded |

**Cost of the 32-bit runtime:** without a 64-bit address space wasm2c can't
use guard pages, so **every load and store in the game carries a bounds
check** (`WASM_RT_MEMCHECK_BOUNDS_CHECK` is forced). The 64-bit builds
measured so far don't pay this. The translation has about 34,000 `i32_load`
and 57,000 store sites. A no-check build option is possible, but it is a
deliberate trade-off (W2.5), not a default.

The module is also linked with `EMULATE_FUNCTION_POINTER_CASTS`
(`port/CMakeLists.txt`): indirect calls through the game's callback tables go
through adapters. This costs the same on every platform, but it is part of
what Broadway has to run.

## 3. The renderer on Wii hardware

The renderer is a GL ES 3 front end (`r_gl.c`, `r_mesh.c`, `r_char.c`,
`r_font.c`) that emulates the PS2 GS. Underneath it is either a real GL driver
or `swgl.c`. It has only **three programs** (SWGL.md): batch (2D sprites,
HUD, menus), mesh (3D stages) and present. All of them fit Hollywood's
fixed-function GX:

| Program feature | GX equivalent | Fit |
|---|---|---|
| Vertex transform (MVP, viewport) | XF matrices, or pre-transformed positions | good |
| Up to 3 directional lights + ambient, clamp | GX hardware lights (8 available), channel clamp | good; exact rounding (`floor`) differs |
| Texture modulate (GS ×2, 0x80 = 1.0) | TEV stage, scale ×2 | good |
| RGBA8 textures | GX RGBA8 (tiled; CPU conversion at upload) | good |
| Index textures through a palette atlas, hand-done bilinear | CI8 + TLUT with hardware bilinear, **but GX TLUTs are 16-bit (RGB5A3/RGB565/IA8)**, or expand to RGBA8 on the CPU | colour depth loss with CI8; CPU cost with RGBA8 |
| Alpha test `a != 0`, blending, depth GEQUAL, depth write mask | alpha compare, blend modes, Z modes | good |
| Linear fog | GX fog (linear) | good |
| Colour interpolated affinely (GS), only UV perspective-correct | GX interpolates perspective-correctly | small differences on 3D stages |
| ADC: skip triangles with a vertex outside the clip box | CPU-side reject before submitting | good, CPU cost per mesh vertex |
| `swgl`'s texel-tie rule | not controllable | pixel differences on scaled sprites (GL has them too) |

A fight frame is about 5,000 triangles and 600,000 fragments (SWGL.md). That
is small for Hollywood's fill rate and vertex rate. The GPU isn't the
constraint. The CPU work around it is: converting textures that change every
frame (character sprite pages), and copying vertices out of the module's
mirrored big-endian memory into the GX FIFO.

**How it plugs in.** The browser build already compiles the renderer with its
`gl*` calls as imports of the wasm module (WebGL). The Wii does the same with
a GX implementation in the host, `port/wii/gx_gl.c`. The renderer front end is
unchanged, and there is still one drawing path, which SWGL.md asks for.

**Accuracy decision for the owner.** The cheap GX palette path (CI8 + 16-bit
TLUT) cannot reproduce 8-bit palettes exactly. The exact path (expand to
RGBA8, or a TEV indirect-texture palette lookup) costs CPU or complexity. Per
AGENTS.md ("never add an option that makes the game worse"), this should be
measured against `swgl` screenshots and decided explicitly.

## 4. Software renderer speed (estimate from measured instruction counts)

Measured here (2026-11-21), `swgl_bench` (the rasteriser's main case:
perspective triangles, palette texture, bilinear, depth, then a blended
layer; about 600,000 pixels per frame):

| Build | Instructions per frame | How counted |
|---|---|---|
| x86-64, GCC 13 `-O2` | 274.5 million | callgrind, 2 frames minus 1 frame |
| PowerPC 750, GCC 13 `-O2 -mcpu=750 -ffp-contract=off` | 341.6 million | qemu `-one-insn-per-tb -d exec`, 2 frames minus 1 frame |
| wall clock, x86-64 (this machine) | 15.9–16.9 ms | three runs of 40 frames |

That is the same work as a real fight frame's drawing: SWGL.md measured 269
million x86-64 instructions per fight frame in the translated build.

Broadway runs at 729 MHz. For this kind of integer code, an in-order-ish
2-issue 750 core manages roughly 0.7–1.0 instructions per cycle. This is an
assumption, not a measurement. That gives:

> 341.6 M instructions ÷ (0.51–0.73 G instructions/s) ≈ **0.47–0.67 s per
> picture**, i.e. 1.5–2 pictures per second.

Even if the IPC assumption is off by a factor of 2, software rendering stays
an order of magnitude short of 60 pictures a second. The frame-skip loop
keeps the game at full speed while drawing few pictures. That is fine for
menus and as a first milestone, but not for playing fights. **Conclusion:
fights need GX (§3).**

## 5. CPU time for the game step (estimate; the main open question)

From the repository's measurements (SWGL.md, generic README, 2026-11-13,
translated x86-64 build):

- 2000 steps without drawing: 18.97 billion instructions, so **about 9.5 M
  x86-64 instructions per step** on average, including boot and menus.
- "Game step and sound": 1.5 ms per fight frame.

Scaling to Broadway:

| Factor | Value | Basis |
|---|---|---|
| x86-64 → PowerPC instruction count | × ~1.25 | measured on `swgl_bench` (341.6 / 274.5); assumed to hold for game code |
| Bounds checks (absent in the 64-bit builds) | × ~1.1–1.4 | assumption; to be measured (W0.3) |
| Broadway throughput | 0.5–0.73 G instructions/s | assumption (§4) |
| **Estimate per step** | **≈ 18–33 ms** | 9.5 M × 1.25 × (1.1–1.4) ÷ (0.5–0.73 G/s) |

Budget: 16.7 ms per step, all included.

What argues that it can fit anyway:

- The PS2's EE (MIPS R5900, **295 MHz**) ran the same game logic at 60 Hz.
  Broadway has 2.5 times the clock and a comparable or better core.
- Part of the 9.5 M is work the PS2 did in hardware: the mixer (48 voices,
  sinc resampling, float, done by the SPU2 on the PS2) and the renderer front
  end. The mixer is a candidate for cheap optimisation once measured.
- The average includes boot-time decompression, which doesn't recur in fights.

What argues against:

- wasm2c code is slower than native code, through bounds checks, the mirrored
  memory addressing on big-endian, and function-pointer-cast adapters.
- The module's memory has to live in MEM2 (GDDR3), which has higher latency
  than MEM1, and the L2 is only 256 KB.

**Verdict on this point: unknown, plausibly within a factor of 2 either way.**
W0.3 (callgrind split of game vs mixer) and W2 (time per step on the console)
settle it. If it's slow, W2.5 lists the levers in order.

## 6. Memory (partly measured; the other main open question)

Wii memory: **MEM1** 24 MB (fast 1T-SRAM, where the DOL is loaded) and
**MEM2** 64 MB GDDR3, of which IOS keeps the top, leaving about 52 MB for
homebrew (to be confirmed in W1). The two are not adjacent in the address
space.

wasm2c needs the module's memory as **one contiguous block**, so it has to
fit in MEM2.

Measured from the built module (2026-11-21, `wasm-objdump`):

| | |
|---|---|
| Memory declared | 3072 pages = **192 MB**, fixed (`build.sh` sets `INITIAL_MEMORY=201326592`, no growth) |
| Static data + 1 MB stack (initial stack pointer) | **8.4 MB** (8,783,360 bytes) |
| EE heap the game allocates at boot (`boot.c`, 22 MB arena + 6 MB handle pool) | **28 MB** |
| **Floor before any port-side buffer** | **≈ 36.4 MB** |

On top of that, inside the module: `swgl`'s textures (stored decoded), its
colour, depth and canvas buffers (≈ 3.7 MB at 640x480), decoded sound banks
(`hdbank.c`, `dcse.c`), the AFS tables and loaded files (`fs.c`), meshes
expanded to float vertices, and the music stream buffers. RENDERER.md notes
that the browser build's wasm memory "never grows past 64 MB" in a full arcade
run, but there the GPU holds the textures. Here they are inside the module.

So the real high-water mark is somewhere between ~37 MB and well above
52 MB, and it wasn't measured: there was no disc. **To measure it**, this
branch adds `mvc2g_mem_top()` (the heap break) to the module and `--mem` to
`mvc2g_native` (W0.1):

```sh
build/port-generic/mvc2g_native private/disc/mvc2.iso 20000 --fuzz 3 --mem
```

If it's over about 48 MB, W1.5 trims host-side buffers first. Moving
textures out of the module, which W5's GX path does anyway, is the largest
saving. The game's own 28 MB is not touched.

## 7. Toolchain

devkitPPC (GCC for `powerpc-eabi`), libogc, libfat, and `elf2dol`, all from
devkitPro. The wasm2c output compiled with Ubuntu's `powerpc-linux-gnu-gcc`
13, so devkitPPC (a newer GCC) is very likely to compile it too. That's an
expectation, not yet a result, because devkitPPC wasn't available in this
session. Versions and flags: WII_DEPENDENCIES.md.

## 8. Input

`mvc2g_pad(port, bits)` takes PS2 pad bits before each step. libogc's
`PAD_ScanPads()` / `PAD_ButtonsHeld(chan)` / `PAD_StickX/Y` / `PAD_TriggerL/R`
give everything needed. The mapping follows `host_sdl.c` (face buttons by
position, analogue triggers past a threshold, stick with a dead zone). Read
right before `mvc2g_frame()`, so there is no added frame of lag from the host.
Two pads = two players. Classic Controller (`WPAD_*`) is optional and later.

## 9. Audio

`mvc2g_audio(800)` returns 800 frames of 48 kHz stereo, signed 16-bit,
little-endian, per step. Wii path: copy out of module memory, swap to
big-endian, into 32-byte-aligned buffers (3,200 bytes per step is a multiple
of 32), `DCFlushRange`, then AI DMA at 48 kHz with a completion callback, or
ASND. Three one-step buffers is about 50 ms of latency. Lower is possible once
underruns are measured.

You can't listen to it yet, so the plan checks it without listening:

- DMA completions per second (should be 60);
- underrun counter;
- a WAV dump to SD compared byte for byte with the desktop's `--wav` output
  for the same frames. Same module, same samples: they must be identical.

The cost of mixing on Broadway is unmeasured and is part of §5's budget.

**Frame timing.** NTSC/PAL60 retrace is 59.94 Hz and the game is a 60.00 Hz
machine. Options: lock to VSync and accept 0.1% slow, or run on the CPU timer
and let one step in ~1000 skip display. Audio drift against video is handled
by dropping or repeating a sample per buffer. Decided and recorded in W7.

## 10. Display

- **Interim (`swgl`):** the canvas is RGBA8, top row first, 640x480. Convert
  it on the CPU to the XFB's YUV 4:2:2 (about 3–5 ms of Broadway time,
  estimated), or tile it into a GX RGBA8 texture and draw one quad. GX then
  converts to YUV during the EFB → XFB copy.
- **Final (GX):** the renderer draws into the EFB (640x480 fits in its
  640x528), copied to the XFB once per picture. No readback, no CPU
  conversion.
- 480p when the console offers it, otherwise 480i. 4:3 native.

## 11. Loading the user's disc image

- The user copies their own image to `sd:/mvc2/mvc2.iso` or
  `usb:/mvc2/mvc2.iso`. `.bin` (raw 2352-byte sectors) also works, since
  `fs.c` already detects it.
- libfat (`fatInitDefault()`) mounts both. `page_disc_read` stays as it is.
- The game reads synchronously and spins within a frame (`func_0012DA40` on
  `ADXF_GetStat`, RENDERER.md §8), so reads must be synchronous, as they are.
  A full arcade run is under 400 reads, 91 MB in total (RENDERER.md). At SD
  speeds of a few MB/s that is a pause of up to a second or two on loading
  screens. The game state doesn't change because of it; the frame-skip loop
  catches up. Measured in W1.
- Music streams from the image continuously (ADX), but only at tens of KB/s.
- Nothing from the disc is ever bundled. A build with the image inside
  (`EMBED=1`) would not fit in the Wii's memory anyway.

## 12. Does the architecture need substantial changes?

- **Game, module, translation, mixer, renderer front end: no.** The
  universal build's design (one module, five host functions, integer-exact
  state, big-endian support) is what makes a Wii port realistic. The Wii host
  is a sibling of `host_sdl.c`.
- **Picture: yes, one new backend.** GX under the `gl*` layer. This is
  probably 70% of the project's effort.
- **Memory: possibly.** If the high-water mark is above MEM2, host-side
  buffers have to shrink. W5 already moves textures out of the module.
- **Speed: unknown.** If the game step is too slow on Broadway, the module
  itself needs work (bounds checks, mixer, hot spots), behind the existing
  state-hash comparison.

## Changes made in this analysis

- `port/generic/mvc2g.c`: new export `mvc2g_mem_top()` (the heap break).
  Adding an export doesn't change any game state.
- `port/generic/host_w2c.c`: `--mem` prints the heap break whenever it grows.
- `port/generic/README.md`: documents `--mem`.
- This document, the roadmap and the dependency list.

## Appendix: commands used

```sh
# Emscripten module and the universal build (no disc: the comparison is skipped)
source ../emsdk/emsdk_env.sh
emcmake cmake -S port -B build/port-em -G Ninja -DCMAKE_BUILD_TYPE=Release
ninja -C build/port-em mvc2_host
port/generic/build.sh

# 32-bit big-endian PowerPC build of the translation (Ubuntu: gcc-powerpc-linux-gnu, qemu-user-static)
cd build/port-generic && mkdir -p ppc/inc && cp /usr/src/wasm2c/*.h ppc/inc/ && cd ppc
F="-O2 -mcpu=750 -ffp-contract=off -DWABT_BIG_ENDIAN=1 -w -Iinc -I.."
powerpc-linux-gnu-gcc $F -c ../mvc2g_w2c.c -o mvc2g_w2c.o
powerpc-linux-gnu-gcc $F -c /usr/src/wasm2c/wasm-rt-impl.c -o rt.o
powerpc-linux-gnu-gcc $F -c ../../../port/generic/host_w2c.c -o host.o
powerpc-linux-gnu-gcc -static mvc2g_w2c.o rt.o host.o -lm -o mvc2g_ppc
# with a disc (W0.2): must equal build.sh's wasm.txt
qemu-ppc-static -cpu 750cl ./mvc2g_ppc DISC 1200 --hash > ppc.txt && cmp ../wasm.txt ppc.txt

# rasteriser benchmark: checksum and instruction counts
# (GLES3/KHR headers copied from emsdk's system/include into glinc/)
cc -O2 -DPORT_SWGL -Iglinc -Iport/platform port/render/swgl.c port/render/swgl_bench.c -lm -o swgl_bench_x86
powerpc-linux-gnu-gcc -O2 -mcpu=750 -ffp-contract=off -static -DPORT_SWGL -Iglinc -Iport/platform \
    port/render/swgl.c port/render/swgl_bench.c -lm -o swgl_bench_ppc
valgrind --tool=callgrind ./swgl_bench_x86 N
qemu-ppc-static -cpu 750cl -one-insn-per-tb -d exec,nochain -D /dev/stdout ./swgl_bench_ppc N | grep -c '^Trace'
```
