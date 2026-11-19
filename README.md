# Horizon Ports

Native ports of [Horizon](https://github.com/horizon-rust/horizon) — *Minecraft: Java Edition 26.3* rewritten in Rust — to **Nintendo Switch** and **Android**.

This is a shared workspace for two parallel porting tracks. The game code is upstream; this repository holds the platform layers, build glue, patches and documentation needed to run it on each target.

## Tracks

| Track | Target | Owner | Branch |
|---|---|---|---|
| Switch | Nintendo Switch homebrew (libnx, wgpu/NVK) | [DuduSync](https://github.com/DuduSync) | `switch` |
| Android | Android (Vulkan/wgpu, touch + gamepad) | TBD | `android` |

**Status:** early scaffold. No port code yet.

## Why one repository

The two ports share the portable Horizon crates, asset-preparation pipeline, renderer patches and tooling. Keeping them together avoids duplicating that work while platform-specific code stays isolated under `platforms/`.

```
crates/              shared Rust crates, patches and host tools
platforms/switch/    libnx entrypoint, NRO/NSP packaging, Switch input/audio/display
platforms/android/   Android app, JNI bridge, touch/gamepad input, storage
docs/                architecture, build and testing notes
third_party/         pinned upstream sources or patches
```

## Working agreement

- `main` stays buildable for both tracks.
- Platform work lands on `switch` or `android` first, then merges to `main` through a pull request.
- Keep platform-specific code out of shared crates.
- Never commit Minecraft assets, prepared game data, console keys, signing keys or credentials.
- Document build steps and known limitations in `docs/`.

## Upstream and prior art

- [horizon-rust/horizon](https://github.com/horizon-rust/horizon) — upstream Horizon, MIT.
- [MakrSas/horizon-switch](https://github.com/MakrSas/horizon-switch) — unofficial Switch port using libnx + wgpu/NVK.
- [Lciarallo/horizon-nx](https://github.com/Lciarallo/horizon-nx) — another Switch homebrew port.
- [tarkodev/horizon-switch](https://github.com/tarkodev/horizon-switch) — experimental native Switch port.

These are references, not vendored code. Check licenses before copying anything.

## Legal

Horizon Ports is unofficial and is not affiliated with Mojang, Microsoft or Nintendo. It contains no Minecraft code, assets or data. Running it requires your own legally obtained copy of *Minecraft: Java Edition 26.3*. Console keys and prepared resources stay on your machine.
