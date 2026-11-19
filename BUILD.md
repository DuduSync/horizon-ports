# Building the Switch NRO

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

## Local NSP packaging

An NSP needs the console's `prod.keys` (never commit it). Use the ELF artifact with `hacbrewpack`:

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
- The NSP is packaged locally from the ELF and the console's `prod.keys`; it is not committed.

## Status

- Audio and full builds pass in CI.
- Account UI is integrated but needs a real Azure client ID and a console test.
- Mods are not enabled: `wasmtime` does not compile for `target_os = "horizon"` yet.
