# Contributing

Two platform tracks share one repository.

## Branches

- `main` — integration; must stay buildable.
- `switch` — Nintendo Switch work.
- `android` — Android work.

Open pull requests into `main`. Keep platform-specific changes out of shared crates.

## Rules

- No Minecraft assets, prepared game data, console keys, signing keys or credentials.
- Document build steps and limitations in `docs/`.
- Prefer upstream Horizon changes over forks of engine code; keep platform glue in `platforms/`.
- Check licences before copying code from other ports.

## Owners

| Area | Owner |
|---|---|
| Switch | DuduSync |
| Android | TBD |
