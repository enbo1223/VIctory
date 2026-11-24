# VIctory

Runs Grand Theft Auto VI (Xbox Series X|S) natively on Windows, from files
the user supplies.

The title's x86-64 code runs unchanged inside our process; VIctory supplies
the platform underneath it and converts the title's data, ahead of time where
it can and on first use where it must (shaders). See
[docs/architecture.md](docs/architecture.md).

## Status (2026-11-23)

**Not playable yet.** The title boots, runs its menus at its own 30 fps and
loads the world, where rendering fails.

| Stage | State |
|-------|-------|
| Boot, first-boot screens (accessibility, language, settings, legal) | works, 30 fps |
| Intro video with sound | works |
| Start screen, profile and saves | works |
| Loading into the world (about 50 s) | works |
| The world (first mission's intro) | reached and runs with indirect draws; its title card renders, the scene after it is still dark; 0.5-1.8 s per frame |

Tested on one machine: Windows 11, AMD RX 6700 XT.

> **Warning.** Entering the world on a real GPU can currently hang it, and
> Windows then resets the display driver (every monitor goes black for a
> moment). To look further, use a diagnostics build with
> `VICTORY_D3D12_WARP=1`, which renders on the software adapter.

What each part does today and what it still needs:

| Subsystem | Today | Needs |
|-----------|-------|-------|
| Loader | Runs the title's executable and its DLLs in our process; crash reports name the faulting code and are kept in `crashes.log` | – |
| Runtime services | One signed-in offline user, licences, packages, saves that survive a kill at any step, text entry dialog | Speech is silent; the dialog and relaunch are untested live |
| Network | Offline by design: the title cannot reach the network | – |
| Files and streaming | Archives and streamed data read, zlib and BCPack decoded in parallel | Loading is limited by writes into title memory |
| Input | XInput pads, keyboard, keyboard stand-in pad | Trigger rumble |
| Audio | Menu, video and loading sound; XMA voices and Opus streams decode | Reverb is silent; Opus results unverified against the title |
| GPU: menus | Draws, compute and presentation on host D3D12; only changed title memory is copied | Frame pacing drifts on one screen in release builds |
| GPU: world | Mesh pipelines, shader-written buffers, descriptor lifetimes, indirect execution (arguments clamped), pipelines made by worker threads and kept on disk | Most of the scene is still missing (untranslated shaders drop draws); frames take seconds; real queries, depth bounds, HDR and scaling are missing |
| Shaders | Console GPU code recompiled to DXIL at first use, cached on disk. Menus: all. World: mesh 8,191 of 8,506, vertex/pixel 10,617 of 12,346, compute 1,643 of 2,539 | 64-bit image atomics, merged geometry stages, the ray-tracing builder's shaders; some compute programs need a GPU with 64-lane waves |
| Textures | Console tiled layouts and detiling for the modes the title uses | Render-target metadata planes |
| Series S mode | Runs through the menus to the loading screen | The world, as above |

## Getting started

Requirements: Windows 10/11 x64, a CPU with AVX2, a Direct3D 12 GPU with
shader model 6.6 (and mesh shaders for the world), Rust (pinned in
`rust-toolchain.toml`).

The title also needs several GiB of available Windows commit at startup.
Check **Committed** under **Task Manager → Performance → Memory**. If
`crashes.log` reports Windows error 1455, close other applications or increase
the Windows paging file size, then retry.

```powershell
cargo xtask install-hooks                # asset-policy check on every commit
copy victory.example.toml victory.toml   # then set your paths (outside the repo)
cargo test --workspace                   # needs no game data
cargo run -p victory -- config           # check your paths
```

Run the title:

```powershell
cargo build --release -p victory
target\release\victory.exe run <game dir>\<the title's exe> --execute
```

The window caption shows the framerate. Saves, settings and `crashes.log`
are kept under the data folder (default `%LOCALAPPDATA%\victory\data`), derived
data under the cache folder.

Input: XInput pads. With none connected the keyboard is a pad while the
title's window has focus:

| Keys | Pad |
|------|-----|
| W A S D / I J K L | left / right stick |
| arrows | D-pad |
| Enter, Backspace, X, Y | A, B, X, Y |
| Q / E, Z / C | shoulders, triggers |
| F / H | stick clicks |
| Esc, Tab | Menu, View |

## Commands

```
victory inspect <exe|dll> [--imports] [--exports]
victory census [paths...] [--missing] [--strict]
victory lib-exports <import.lib> [--list]
victory shader-info [paths...] [--verbose] [--disasm]
victory translate [paths...] [--validate] [--pipelines] [--list] [--jobs N]
victory run <exe> [--execute] [--unimplemented trap|zero] [--strict]
            [--console series-x|series-s] [-- <title arguments>]
victory config
```

Logging: `VICTORY_LOG=victory_loader=trace,info` (a release build logs
info and above).

## Debugging

A release build has no debugging aids. Build them in with

```powershell
cargo build --profile diag -p victory --features diagnostics   # target\diag\victory.exe
```

| Variable | Effect |
|----------|--------|
| `VICTORY_LOG=<target>=debug,...` | debug and trace output |
| `VICTORY_STACKS=20,60` | every thread's stack at those seconds |
| `VICTORY_WATCH=<rva>,...` | logs a title u32 when it changes |
| `VICTORY_KEYS=30:Enter,35:X` | presses keys at those seconds, focus or not |
| `VICTORY_KEYS_AS_PAD=1` | with the above, scripted keys also press buttons on the pad the title reads, even if a real pad holds slot 0 (a keyboard fallback alone is only created when no pad is connected) |
| `VICTORY_RUMBLE_TEST=1` | with a DualSense on HID, plays a 1.5 s startup pattern (both motors, then trigger resistance, then all four together); no title or menu needed |
| `VICTORY_CRASH_SAVE=<rva>:<len>,...` | saves title memory to the cache folder on a crash |
| `VICTORY_PRESENT_1440P=1` | diagnostics: present the world's 2560x1440 target instead of the title's black 4K upscale plane |
| `VICTORY_CAPTURE=<dir>` | saves presented frames (powers of two, or `VICTORY_CAPTURE_FRAMES=<list>`) |
| `VICTORY_DUMP_TARGETS=1` | with the two above, saves every render target of those frames |
| `VICTORY_D3D12_DEBUG=1` | host debug layer |
| `VICTORY_D3D12_GBV=1` | with the above, GPU-based validation (bad descriptor use reported, not faulted; slow) |
| `VICTORY_D3D12_DRED=1` | on a GPU failure, the allocation hit and the commands in flight |
| `VICTORY_SKIP_PIPELINES=<hash>,...` | omit draws and dispatches, including indirect ones, whose pipeline contains a listed 16-digit shader hash |
| `VICTORY_GPU_TIMING=1` | with the above, each list of a submission run and timed on its own; slow lists logged with their slowest commands |
| `VICTORY_D3D12_WARP=1` | host device on the software adapter (slow; cannot reset the driver) |
| `VICTORY_SHADER_LOOP_LIMIT=<n>` | translated shader loops stop after `n` iterations per lane instead of 2^20 (own shader cache entries); if a lower `n` makes world frames much faster, loops hitting the limit are what makes them slow |
| `VICTORY_AUDIO_CAPTURE=<file.wav>` | records what the title plays (our process only) |
| `VICTORY_XMA_DUMP=<dir>` | saves the first 32 XMA voices' context and input |

Every build appends crashes and host GPU failures to `crashes.log` in the
title's data folder.

### World GPU failures on a real adapter

The host debug layer needs a D3D12 SDK component that is not part of the
runtime redistributable; without it `VICTORY_D3D12_DEBUG=1` only reports
`0x887A002D`. Device-removed data works without it:

```powershell
$env:VICTORY_D3D12_DRED = '1'
$env:VICTORY_LOG = 'info'
.\target\diag\victory.exe run <game.exe> --execute 2> gpu.log
```

A capture of the Xbox 1.0.2.1 world with those settings reported
`0x887A0006` after a submission of 28 lists. DRED located the first stopped
list at command 153 of 197 (op 5, an indirect draw) with mesh
`db4835a364449a45` and pixel `c10e7d95d8266d2a`. Those shaders read a
43-million-element SRV and a 182-million-element (174 MB) SRV; the copy
statistics show 4–5 GB copied into the latter range *per frame* and frames
over a second long, long enough for Windows to time out the GPU. This is a
limitation of the unfinished world translation, not of the game files.

To avoid that draw for diagnosis, keep only the hash of one of its shaders:

```powershell
$env:VICTORY_SKIP_PIPELINES = 'db4835a364449a45,c10e7d95d8266d2a'
```

Or use `VICTORY_D3D12_WARP=1`, which cannot reset the driver but is much
slower. `VICTORY_SHADER_LOOP_LIMIT` can also be tried for the general slowness
of world frames.

With the community fixes merged here (mirrored heap slots the title writes,
bounded one-view root SRV tables, aligned CBV tables that no longer exhaust
the capped dynamic range, and guarded `OMSetRenderTargets` ranges), a world
run on an RTX 4060 with `VICTORY_D3D12_DRED=1` stayed up for 160 seconds and
rendered frames of the world.

Console compute programs that declare `WaveSize(64)` cannot become host
pipelines on 32-lane hosts (NVIDIA reports a wave range of 32..32): the host
refused them with `0x80070057` and dropped their dispatches. Translation now
queries `D3D12_FEATURE_D3D12_OPTIONS1` and lowers those programs to wave32 /
emulated wave64 automatically; no switch is needed. On an RTX 4060 Laptop this
removed all 120 pipeline refusals. A full run with `VICTORY_D3D12_DRED=1` then
survived the whole 240-second test window with **0 device removals** and
captured real world frames. World rendering is still slow (0.4–0.9 fps) and
still drops draws whose host pipelines are not ready or whose shaders do not
translate; those are translation limits, not device loss. A profile selector
before the world can be passed in unattended runs with
`VICTORY_KEYS_AS_PAD=1`.

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).
