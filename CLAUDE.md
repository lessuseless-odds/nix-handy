# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

Read @AGENTS.md — it is the primary guide (dev commands, architecture, code style, CLI flags, i18n, GitHub/PR workflow) and applies here too.

## About this repository

`nix-handy` is a fork of [cjpais/Handy](https://github.com/cjpais/Handy) whose sole addition is first-class Nix packaging: a flake (`flake.nix`), a NixOS module (`nix/module.nix`), and a Home Manager module (`nix/hm-module.nix`). Application code (`src/`, `src-tauri/`) otherwise tracks upstream — AGENTS.md's architecture notes apply unchanged.

## Nix packaging

- `flake.nix` defines the `handy` package (`pkgs.rustPlatform.buildRustPackage`), a NixOS module, a Home Manager module, and a dev shell.
  ```bash
  nix build .#handy       # build the package
  nix develop             # dev shell with Rust + Bun toolchains (auto-runs `bun install`)
  ```
- Bun dependencies are vendored via [bun2nix](https://github.com/nix-community/bun2nix): `.nix/bun.nix` is generated from `bun.lock`, not hand-written.
  - Kept in sync automatically by the `postinstall` script (`scripts/check-nix-deps.ts`), which runs after every `bun install`/`add`/`remove`/`update`. It hashes `bun.lock`, compares against `.nix/bun-lock-hash`, and regenerates `.nix/bun.nix` via `bunx bun2nix` only when the hash changed (no-op otherwise).
  - Can be run manually: `bun scripts/check-nix-deps.ts`.
  - **If `bun.lock` changes, commit `.nix/bun.nix` and `.nix/bun-lock-hash` alongside it** — the flake build will drift or fail otherwise.
- The Nix build sandbox has no audio devices, model files, or GPU, so `doCheck = false` on the package derivation — Rust tests only run outside Nix (see Testing below).
- `postPatch` in `flake.nix` carries several sandbox-only source patches (disabling the updater-artifact bundle, stripping the `postinstall` hook, pointing `libappindicator-sys` at the Nix store, disabling `cbindgen` in `ferrous-opencc`'s build script). If the corresponding upstream Cargo dependency is bumped, check whether these patches still apply/are still needed.
- `nix/module.nix` (NixOS) adds a udev rule opening `/dev/uinput` to the `input` group — required for `rdev`'s global-shortcut `grab()`. `nix/hm-module.nix` (Home Manager) wires up a `systemd --user` service for autostart.

## Testing

AGENTS.md doesn't cover test commands; use these:

- **Rust unit tests** are inline (`#[cfg(test)]`) across most of `src-tauri/src` (managers, `audio_toolkit`, `catalog`, `settings.rs`, etc.):
  ```bash
  cd src-tauri && cargo test
  ```
- **Frontend E2E tests** use Playwright (`tests/`, config in `playwright.config.ts`), served via `bunx vite dev` on port 1420:
  ```bash
  bun run test:playwright       # headless
  bun run test:playwright:ui    # interactive UI runner
  ```
- **i18n locale validation**: `bun run check:translations`.
