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

## Why this is easier than Roblox NX

Horizon is source-available Rust with portable crates. A port adapts the platform layer: target, libc gaps, window/input, GPU backend, storage, packaging.

Roblox NX loads a proprietary Android ARM64 `libroblox.so` with no source. The work is binary compatibility: bionic vs newlib, TLS, pthread/TSD, allocator internals (mimalloc/ThreadCache), fake JNI, EGL, AAssetManager and the app-bridge lifecycle. That is a runtime, not a port.

## Implication for this repo

Start the Switch track from `MakrSas/horizon-switch` (MIT) or import its `switch/` layer, then keep upstream Horizon as the source of game code. Check the licences of NVK, mesa-switch and every patch before copying.
