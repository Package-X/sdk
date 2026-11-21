# PackageX (`packx`) Engineering & Agent Ground Rules

Read this file before modifying specifications or authoring implementation code in `PackageX`.

## 1. Workflow Discipline
- **Planning vs. Implementation Separation:** Keep advance planning and spec edits strictly separate from `.rs` code implementation unless explicitly instructed to move into a coding phase.
- **Keep `/plan/` Synchronized:** Whenever an architectural decision, binary layout, CLI flag, or exit code changes, update the corresponding specification in `/plan/` in the same change set.

## 2. Code Formatting & Style Standards
- **Tab Indentation Only:** Always use literal TAB characters (`\t`) for all indentation across Rust, TOML, JSON, `.px`, `.pxs`, `.s.px`, and nested lists—never mix spaces and tabs.
- **Onboarding Comments:** Every module, public function, struct, and non-trivial block must carry clear, concise inline comments explaining *what* it does and *why* so new contributors can onboard immediately.
- **DRY & Modular Crates:** Keep workspace crates decoupled cleanly along the boundaries defined in `plan/architecture/overview.md`.

## 3. Non-Negotiable Architectural Invariants
1. **White-Label End-User Footprint:** Never leak `PackageX` or `px-` strings into end-user install directories, uninstaller paths (`%LOCALAPPDATA%\<package_name>\uninstaller\<app_id>.exe` or `%ProgramData%`), staging/backup temp folders, shell `$PATH` sentinel comments, or PE Version Resources.
2. **Zero UAC on Inspection Flags:** Always stamp Windows PE stubs with `asInvoker`. Handle `elevate = true` via runtime self-elevation (`ShellExecuteW("runas")`) *only* after `-packagex-detail` and `--dry-run` are ruled out.
3. **Signed PE Overlay Safety:** Never assume `PxFooter` lives at raw `EOF - 64` on Windows. Always inspect the PE Optional Header Certificate Table offset first (`cert_table_offset - 64`) so Authenticode signatures never break footer discovery.
4. **Same-Volume Atomic Staging:** Always stage extracted files in a hidden sibling directory on the same filesystem/volume as `$instdir$`—never cross-volume from `$temp$`—so commit swaps stay atomic.
5. **Standard `std` Rust + Audited Crate ASM:** Use standard `std` Rust (`opt-level = "z"`, LTO, `strip`) for `px-runtime` and `px-stub`. Rely on audited crate-level SIMD ASM (`blake3`, `sha2`, `zstd-rs`) rather than hand-rolled assembly stubs.
6. **MPL-2.0 Vendoring Hygiene:** Any vendored macOS crates derived from [`indygreg/apple-platform-rs`](https://github.com/indygreg/apple-platform-rs) or its downstream forks must retain original copyright/MPL-2.0 headers and keep file-level modifications open under MPL-2.0.

## 4. Phase 1 MVP Vertical Slice (Build Order)
When implementation begins, ship a working vertical slice before expanding to all 11 targets:
1. `px-spec` (`.px` TOML parser, `<section>.<os>.<key>` step resolver, `packx check` linter).
2. `px-toolbelt` + `px-shim` (blocking lockfile, CAS `sdk_index.json`, `--offline` diagnostics).
3. `px-assembler` + `px-stub` (Windows PE overlay `.exe`, `zstd` deterministic packer, `goblin` signed-PE footer reader, `AttachConsole`, `-packagex-detail`, `--dry-run`, `packx test` Desktop sandbox).
4. `px-runtime` + `px-script` (same-volume staging, rollback journal, `wasmi` `.s.px` Hook SDK, white-label uninstaller).
5. Expand to `.msi`, `.dmg`, `.deb`, `AppImage`, WinLook `wl_ui`, and AEAD password sealing.
