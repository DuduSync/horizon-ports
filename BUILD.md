# Building the Switch NRO

## Switch distribution

The console gets exactly **one** file: `/switch/Horizon.nro`. Each build overwrites it, and it is launched from the hbmenu. Versioned NROs and NSPs are no longer packaged or uploaded; the NSP instructions below are kept for legacy builds only. The build watcher is `watch_nro.py` (`RUN_ID` + `NRO_PREFIX`), which uploads the NRO to the stable path and removes old `Horizon-*.nro`/`Horizon-*.nsp` files.

The port is built from the upstream `MakrSas/horizon-switch` v0.1.1 plus three patches in `patches/`:

1. `launcher-mvp.patch` — integrated dashboard, worlds, account placeholder, settings, offline name, pt-BR.
2. `auth-msa.patch` — Microsoft device-code login, Xbox/XSTS/Minecraft, `AuthSession`, encrypted token store.
3. `account-ui.patch` — Account screen wired to the auth API (device code, PIN/OSK, sign out).

The audio backend comes from `tarkodev/horizon-switch`'s patched `cpal` (libnx `audren`), copied into `switch/vendor/cpal`.

## GitHub Actions

`.github/workflows/switch-full-nro.yml` clones the base, applies the patches, builds NVK, builds the game with `--features audio`, and uploads:

- `Horizon-full-nro` — `Horizon-full.nro` for hbmenu.
- `Horizon-full-elf` — `horizon.elf` for local NSP packaging.

The CI uses the prebuilt `devkitpro/devkita64` image (with all portlibs) and skips `dkp-pacman -Syu`, because `pkg.devkitpro.org` returns 403 to GitHub runners. The game toolchain is nightly `2026-10-25`, the one the v0.1.1 `std.patch` expects.

## Local NSP packaging (legacy)

The current Switch flow is NRO-only. This section is kept for older builds that still use an installed NSP. An NSP needs the console's `prod.keys` (never commit it). Use the ELF artifact with `hacbrewpack`:

```sh
# inside a Linux environment with switch-tools and hacbrewpack
elf2nso horizon.elf exefs/main
npdmtool npdm.json exefs/main.npdm
nacptool --create "Horizon" "Horizon contributors" "0.1.1-full" control/control.nacp --titleid=0100485952450000
hacbrewpack -k prod.keys --titleid 0100485952450000 --noromfs \
  --exefsdir exefs --controldir control --nspdir out
```

The NSP uses `sdmc:/switch/horizon/resources.romfs` (625 MB, with sounds and 143 language files).

## Current artifacts

Build run `44518820144` (audio + launcher + auth + account UI):

- `Horizon-full.nro` — 53,109,580 bytes, SHA-256 `65315b7c620f262255ff14dd9d69f65f0f96b7ed8788af2881c645fc142b0b81`.
- `horizon.elf` — SHA-256 `bc9010f055f209fb621f8a8fe2c35eb8093b5d91f014f76666405a36cf9cfaf0`.
- `Horizon-full.nsp` — 28,710,104 bytes, SHA-256 `c7665ed789c13d8ab93dc62985c7467d31bad0df6f3e932d7223078147a18fc7`, packaged locally from the ELF and the console's `prod.keys` and uploaded to `/switch/Horizon-full.nsp`. The key was deleted after packaging and is not in this repository.

## v2 artifacts

Build run `44657286518`:

- `Horizon-v2.nro` — 54,469,452 bytes, SHA-256 `22a610f74473cd60962c72d8646f3436cdd032d27029d20a2a5576bb3887891f`.
- `Horizon-v2.nsp` — 29,534,424 bytes, SHA-256 `2752dc3be3b07b96836a6938079245791e560328fd009fbdf9a252a92001364e`.

v2 includes the separate launcher (dark, Geist, sidebar), the original Minecraft title untouched, QR device-code login, the official mod browser, USB keyboard/mouse (Bluetooth needs a USB dongle; the Switch 1 does not pair generic Bluetooth HID), and 1920x1080 docked / 1280x720 handheld. The NRO/NSP were uploaded to `/switch/`.

## v3 artifacts

Build run `44794429415`:

- `Horizon-v3.nro` — 54,473,548 bytes, SHA-256 `0977ae6dedbc1ef517786775ae0c75866b2b044ec9bcc260e896fb2016cce1aa`.
- `horizon.elf` — 64,640,312 bytes, SHA-256 `e8cdf1f373503fc0798d82dc01f6bb63db7ff5f320d589a0ba08a3f5c32991bb`.
- `Horizon-v3.nsp` — 29,535,960 bytes, SHA-256 `e62700b8c0b776ab6b31de4211ffa3d3fb44dd1068bba0ea3504b9bd40188408`, packaged locally with the console's `prod.keys` and uploaded to `/switch/Horizon-v3.nsp`. The NRO is also in `/switch/Horizon-v3.nro`.

v3 fixes the launcher's coordinate space (1040x700 dp instead of the 426x240 GUI pixels), rasterizes the Geist faces at the screen density, and re-rasterizes run-time glyphs after the atlas texture is rebuilt.

## v4 artifacts

Build run `44801452331`:

- `Horizon-v4.nro` — 54,481,740 bytes, SHA-256 `5eb88a894f71e267bad4605758b7b059db3ae59e270cf8181276ca852f69c32d`.
- `horizon.elf` — 64,647,584 bytes, SHA-256 `6f5e7dbbc127fedbc1a96b70544a4a5c43af199ba48e8d6dbbaf5d14f6343493`.
- `Horizon-v4.nsp` — 29,546,712 bytes, SHA-256 `4b3e995d120f110f4d22fead80c9a668a58b9c90f6ca030581118127f99fc146`, uploaded to `/switch/Horizon-v4.nsp`.

v4 adds the live `appletGetDefaultDisplayResolution` query for 1080p docked, the 24x36 mouse cursor, the `cursorSpeed` setting, raw-delta pointer movement and the HID report-rate log.

## v6 artifacts

Build run `44813489808` (cached: 30 minutes total):

- `Horizon-v6.nro` — 54,489,932 bytes, SHA-256 `0007c751445c254bd07a55aa45a795d6f2b8308ad927d3bfbd86aa9a091bc4ef`.
- `horizon.elf` — 64,684,496 bytes, SHA-256 `fa53bb019da9d5eb4f6e6d605d9a878708be49400e82ed7dd5ce828a63f88def`.
- `Horizon-v6.nsp` — 29,552,344 bytes, SHA-256 `35cf704a4960f2df7798d7aefdb321cb6a094b547d64d36f3b375189adfdec80`, uploaded to `/switch/Horizon-v6.nsp`.

v6 adds the branded loading screen, the launcher interface-size setting (100–175%, 150% default), controller/analog navigation with focus outlines on launcher buttons and mod rows, mod-install logging, and the Switch caps of 16 chunks render distance / 128 chunks LOD that prevent the NVK section-arena OOM seen at 64 chunks. The workflow caches NVK, Docker layers and the Cargo target; the next builds should stay near half an hour.

## Status

- Audio and full builds pass in CI.
- Account UI is integrated but needs a real Azure client ID and a console test.
- Mods are not enabled: `wasmtime` does not compile for `target_os = "horizon"` yet.
