# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Package manager is **bun** (`bun.lock` is committed; CI uses bun).

- `bun install` — install frontend dependencies
- `bun run tauri dev` — the real dev loop. It runs `bun run dev` for you via `beforeDevCommand`.
- `bun run dev` — Vite only, on fixed port 1420. Useful for pure UI work, but `invoke()` fails
  because there is no Tauri IPC host. Don't use it to test install behavior.
- `bun run build` — `tsc && vite build`. This is the **typecheck gate** (`noEmit`, `strict`,
  `noUnusedLocals`, `noUnusedParameters`).
- `bun run tauri build -b msi` — matches what CI builds. Windows-only; fails on other hosts for
  the same reason `cargo check` does (see Platform constraint).
- From `src-tauri/`: `cargo check`, `cargo clippy` for the Rust side — **but see Platform
  constraint below; these do not work on a non-Windows checkout.**

**There is no test suite and no linter configured.** No `test` or `lint` script exists in
`package.json`. `tsc` (via `bun run build`) and `cargo check` are the only correctness gates.

## Platform constraint

This app is Windows-only *by construction*, not by configuration: `winreg` and
`HKEY_CURRENT_USER` are used unconditionally in `src-tauri/src/lib.rs`, and the macOS/Linux
matrix entries in `.github/workflows/publish.yml` are commented out.

`winreg` is a plain (not `cfg(windows)`-gated) dependency in `Cargo.toml`, so on macOS/Linux
**`cargo check`, `cargo clippy`, `cargo build`, and `bun run tauri dev` all fail** — `winreg`
hits its own `compile_error!("OS not supported")` before this crate's code is ever compiled.
There is no way to type-check the Rust here on a non-Windows machine without either installing
the `x86_64-pc-windows-msvc` target and using `cargo check --target`, or moving `winreg` under
`[target.'cfg(windows)'.dependencies]`. Rust changes made on macOS/Linux are effectively
unverified until CI or a Windows machine builds them — review them accordingly.

## Architecture

**The entire app is a single IPC call.** `src/App.tsx` calls `invoke("run_path_update")`; the
handler is `#[tauri::command] async fn run_path_update` in `src-tauri/src/lib.rs`, registered in
`tauri::generate_handler!`. Adding any capability means touching all three of those places.
`src-tauri/src/main.rs` is a two-line shim into `qsv_easy_installer_lib::run()`; the `_lib` suffix
is a deliberate Windows name-collision workaround (see the comment in `Cargo.toml`).

**What `run_path_update` actually does** — three steps that read like one:

1. `GET https://api.github.com/repos/dathere/qsv/releases/latest` and reads `tag_name` for the
   release tag. Note `/releases/latest` **excludes prereleases and drafts**, so the installer
   tracks the latest *stable* qsv, which can lag the newest published tag.
2. Downloads that release's `x86_64-pc-windows-msvc.zip` to a tempfile, extracts `qsv.exe`, and
   writes it to `app_local_data_dir/bin/qsv.exe`.
3. Prepends that `bin` dir to the `HKCU\Environment\Path` registry value, if not already present.

Because it edits the registry `Path`, the change only takes effect in newly-opened terminals —
which is what the UI copy tells the user.

**Current error handling:** `lib.rs` uses `unwrap()` on every fallible call, and `App.tsx` uses
`.finally()`, so the "Successfully installed" alert fires even when the command panics. Don't
assume there is error handling to build on.

**IPC permissions** are gated by `src-tauri/capabilities/default.json`, which currently grants only
`core:default` and `opener:default`. A new plugin needs its permission added there.

## Releasing

`src-tauri/tauri.conf.json` sets `"version": "../package.json"`, so the app version comes from
`package.json`, and CI's `tagName: v__VERSION__` is substituted from it. A release is: bump the
version in `package.json`, then manually trigger the `Publish to Releases`
(`workflow_dispatch`) workflow, which builds an MSI and opens a **draft** GitHub release.

Note: `src-tauri/Cargo.toml` is drifted at `1.0.0` while `package.json` is at `1.1.1`. The Cargo
version is not what ships.

## Known quirk

`.github/workflows/publish.yml` keys its bun cache on `hashFiles('**/bun.lockb')`, but the
committed lockfile is `bun.lock` — so that cache key never matches a real file and the cache is
effectively dead.
