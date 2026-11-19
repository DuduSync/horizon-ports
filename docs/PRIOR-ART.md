# Prior art

Horizon (Minecraft: Java Edition 26.3 in Rust) has already been ported to the Nintendo Switch by more than one project. Do not start from zero.

| Project | What it is | State |
|---|---|---|
| [horizon-rust/horizon](https://github.com/horizon-rust/horizon) | Upstream Horizon, MIT. Desktop Linux/macOS/Windows. | Active upstream |
| [MakrSas/horizon-switch](https://github.com/MakrSas/horizon-switch) | Fork with a playable Switch homebrew port under `switch/`. | Single-player works, up to 60 fps at RD 6 handheld, NSP, Joy-Con, touch, HD Rumble. No sound or multiplayer. |
| [Lciarallo/horizon-nx](https://github.com/Lciarallo/horizon-nx) | Another Switch homebrew port (libnx + wgpu). | Early |
| [tarkodev/horizon-switch](https://github.com/tarkodev/horizon-switch) | Experimental native Switch port (Rust, libnx, wgpu/NVK). | Experimental |

## What the working Switch port needed

- Custom Rust target `aarch64-nintendo-switch-newlib` (`target_os = "horizon"`, newlib), linked by devkitA64 against libnx.
- `std` built from source with `-Z build-std` and a patched `rust-src`.
- Patched crates: `libc`, `getrandom`, `ring`, `winit` (NX backend), `wgpu`, `wgpu-core`, `arboard`.
- `third_party/wgpu-hal` with Horizon EGL and NVK VI surfaces.
- `nxshim.c`: newlib gaps (`getrandom`, `sysconf`, `dlopen`), pthread keys, RomFS mount, crash reports, thread placement.
- `crates/horizon-nx`: controllers, HD Rumble, memory, RomFS, allocator.
- NVK built once with Docker/Podman (~20 GB disk) via `switch/scripts/build-nvk.sh`.
- NSP packaging with hacbrewpack plus the console's `prod.keys`.

## ARM64 is not the same as Switch

Horizon builds and runs on desktop ARM64: Apple Silicon and Windows on ARM (`aarch64-pc-windows-msvc`). That proves the CPU architecture, game logic, math and much of the renderer are clean on AArch64. It does not prove a Switch or Android port.

What desktop ARM64 does **not** provide:

- Windows ARM64 uses Win32/MSVC, DirectX 12 or Vulkan and PC drivers. Switch uses Horizon OS, newlib, libnx, NVK/mesa-switch and NSP packaging.
- The Switch port needed a custom Rust target, a patched `rust-src`/`-Z build-std`, patched `libc`/`winit`/`wgpu` crates and `nxshim.c`. None of that exists in the Windows build.
- Audio, input, storage and packaging are new platform layers on every target.

Android benefits more from the ARM64 work: it is ARM64, Linux, bionic and Vulkan. The remaining work is the Android platform layer (NDK, `android-activity`, JNI, lifecycle, touch, storage). For Switch, the real shortcut is the existing `MakrSas/horizon-switch` port, not the Windows ARM64 build.

## Why this is easier than Roblox NX

Horizon is source-available Rust with portable crates. A port adapts the platform layer: target, libc gaps, window/input, GPU backend, storage, packaging.

Roblox NX loads a proprietary Android ARM64 `libroblox.so` with no source. The work is binary compatibility: bionic vs newlib, TLS, pthread/TSD, allocator internals (mimalloc/ThreadCache), fake JNI, EGL, AAssetManager and the app-bridge lifecycle. That is a runtime, not a port.

## Implication for this repo

Start the Switch track from `MakrSas/horizon-switch` (MIT) or import its `switch/` layer, then keep upstream Horizon as the source of game code. Check the licences of NVK, mesa-switch and every patch before copying.
