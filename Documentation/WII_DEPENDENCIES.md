# Wii port: dependencies

What a Wii build of the universal build (`mvc2g`) needs, what it does *not*
need, and what must never be committed. Companion to
[WII_FEASIBILITY.md](WII_FEASIBILITY.md) and
[WII_PORT_ROADMAP.md](WII_PORT_ROADMAP.md).

Written 2026-11-20, with no Wii hardware and no disc image in hand. Every row
below is a requirement, not something that has been run on a Wii.

## 1. Build pipeline at a glance

```
 port/ (game C + port glue)            existing, unchanged
        │  emcc (Emscripten 6.0.13)    existing: ninja -C build/port-em mvc2_host
        ▼
 mvc2g.wasm  (the game, swgl renderer, mixer)
        │  wasm2c (wabt 1.0.34) + fix_w2c.py (big-endian bulk-memory fix)
        ▼
 mvc2g_w2c.c / .h  (plain C, ~ the whole game)
        │  powerpc-eabi-gcc (devkitPPC), -mcpu=750 -meabi -mhard-float, -DWABT_BIG_ENDIAN=1
        ▼
 + host_wii.c (new: disc file, video, audio, pads, 60 Hz loop)  + libogc + libfat
        ▼
 boot.elf  ──elf2dol──▶  boot.dol   →  sd:/apps/mvc2/boot.dol (Homebrew Channel)
```

The first two stages are the project's existing universal build and run on
the Linux build machine. Only the last stage is Wii-specific.

## 2. Build-machine tools (Linux x86-64)

| Tool | Version | Why | Source |
|---|---|---|---|
| Emscripten (emsdk) | 6.0.13 (the version AGENTS.md pins) | Builds `mvc2_host` and links `mvc2g.wasm` | emsdk |
| wabt (`wasm2c`) | 1.0.34 (Ubuntu 24.04 package) | Translates the module to C. `fix_w2c.py` is written against this version's output | `apt-get install wabt` |
| Python 3 | any 3.x | hostify, link glue, `fix_w2c.py` | system |
| CMake, Ninja | 3.20+ | the port's build | system |
| **devkitPPC** | current release (r46 or later; record the exact version when first used) | `powerpc-eabi-gcc` for the Broadway CPU, `elf2dol`, linker scripts for the Wii | devkitPro pacman (`dkp-pacman -S wii-dev`) |
| **libogc** | the release that ships with that devkitPPC | Video (VIDEO_*/GX), audio (AUDIO_*/ASND), pads (PAD_*/WPAD_*), timers, cache control | devkitPro (`wii-dev` group) |
| **libfat** (`libfat-ogc`) | same | FAT on the front SD slot (`sd:/`) and USB mass storage (`usb:/`) | devkitPro (`wii-dev` group) |
| `gcc-powerpc-linux-gnu`, `qemu-user-static` | Ubuntu 24.04 | **Optional, for testing before hardware:** builds the same translated C as a 32-bit big-endian PowerPC Linux program and runs it under qemu, so the state-hash comparison runs on the Wii's byte order and pointer size | `apt-get` |
| Dolphin emulator | current dev build | **Optional:** a quick boot check of the `.dol`. Dolphin's CPU timing is not cycle-accurate, so **no speed figure from Dolphin counts** | dolphin-emu.org |

Alternatives that were considered and are not recommended for the first build:

- **libogc2 / the Extrems fork**: newer, but the homebrew ecosystem and most
  examples use devkitPro's libogc. Can be revisited if a specific fix is needed.
- **Nintendo's Revolution SDK**: proprietary, cannot be used or committed
  under this project's rules.
- **Compiling the port natively for PowerPC (no wasm2c)**: the game code
  assumes a little-endian machine everywhere it reads the disc's data and its
  own structures. The wasm2c route is how the project already handles
  big-endian CPUs (verified on s390x), so it is the route here.

## 3. Wii-side runtime dependencies

| Need | Provided by | Notes |
|---|---|---|
| Launcher | Homebrew Channel (any recent version) | Loads `apps/mvc2/boot.dol`, passes argv from `meta.xml` |
| IOS | IOS58 (HBC's default) | Needed for USB 2.0 through libogc. Reload to IOS58 at start if HBC launched with something else (`<ahb_access/>` in `meta.xml` keeps hardware access) |
| Storage | FAT32 SD/SDHC in the front slot, or FAT32 USB drive | The disc image is ~ 210 MB, so no 4 GB file-size problem. exFAT/NTFS are not supported by stock libfat |
| Controllers | GameCube controller (ports 1–4) | Through `PAD_*` (SI). No Bluetooth needed. Wii Remote + Classic Controller through `WPAD_*` is a later, optional milestone |
| Display | 480i/480p, NTSC or PAL60 | `VIDEO_GetPreferredMode()`; PAL50 is not a target (the game is a 60 Hz machine) |

## 4. Repository code the Wii build reuses unchanged

| Part | Path |
|---|---|
| The whole game, its data layout, the platform replacements | `port/host/`, `port/stubs/`, generated glue |
| Entry points | `port/generic/mvc2g.c` |
| Software renderer (first milestones) | `port/render/swgl.c` + `r_gl.c`, `r_mesh.c`, `r_char.c`, `r_font.c`, `decode.c` |
| Mixer | `port/audio/*` (built with `-DPORT_NO_AUDIO_DEVICE`) |
| Build of the module and its translation | `port/generic/build.sh`, `fix_w2c.py` |
| Host imports (WASI stubs, disc read, `m_get`/`m_put`) | `port/generic/host_w2c.c` (included by the Wii host the same way `host_sdl.c` includes it) |
| Correctness check | the state-hash comparison in `build.sh` (wasm under node vs translated C) |

## 5. New code the Wii build needs (all under `port/wii/`, nothing else edited)

| File (proposed) | Contents |
|---|---|
| `port/wii/host_wii.c` | `main()`: libogc init, file open on `sd:/` or `usb:/`, 60 Hz loop with frame skip, pads, audio queue, video present. Includes `../generic/host_w2c.c` with `MVC2G_NO_MAIN`, as `host_sdl.c` does |
| `port/wii/video_wii.c` | RGBA8 canvas → the screen. First version: CPU conversion to the external framebuffer (YUV 4:2:2). Second version: upload as a GX texture and draw one textured quad |
| `port/wii/audio_wii.c` | 48 kHz stereo s16 through the audio interface DMA (or ASND), double-buffered |
| `port/wii/pad_wii.c` | GameCube pad → PS2 pad bits (`platform/pad.h`) |
| `port/wii/build_wii.sh` | Takes `build/port-generic/mvc2g_w2c.c` (from `build.sh`) and builds `boot.dol` with devkitPPC |
| `port/wii/meta.xml`, `icon.png` | HBC metadata (an original icon, not Capcom's art) |
| `port/wii/README.md` | How to build, install and test; measurements with dates |

## 6. Compile flags that matter

| Flag | Why |
|---|---|
| `-DWABT_BIG_ENDIAN=1` on every file of the translation and the host | wasm2c's big-endian memory layout (README "Another CPU") |
| `-ffp-contract=off` | The README requires it on any target that could fuse multiply-add; PowerPC does (`fmadd`). Without it the float results, and so the game state, can differ from the wasm |
| `-mcpu=750 -meabi -mhard-float` | Broadway; devkitPPC's `machdep` flags (`-mrvl` for the Wii) |
| `-O2` (try `-O3`/`-Os` with measurements) | `-Os` may win on a 32 KB I-cache; measure, don't assume |
| `WASM_RT_MEMCHECK_BOUNDS_CHECK` (forced on 32-bit) | wasm2c cannot use guard pages without a 64-bit address space, so every module load and store carries a bounds check. Removing it needs a patched `wasm-rt.h` and is a measured, later decision (WII_FEASIBILITY.md §4) |
| `WASM_RT_USE_MMAP=0` (automatic on 32-bit) | Memory is allocated with `calloc`; there is no `mmap` on the Wii |

## 7. Never committed

Per AGENTS.md, and repeated here because a Wii build makes it tempting:

- The disc image, anything extracted from it, and any `.dol` built with
  `EMBED=1` (the disc inside the program). Such a build stays in `private/`.
- devkitPro/libogc binaries (they are installed on the build machine, not
  vendored).
- Nintendo SDK files, keys, signed IOS/WADs, or a channel `.wad` that contains
  the game.
- Capcom or Marvel artwork in `icon.png` or channel banners.
