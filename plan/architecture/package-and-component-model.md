# SDK Package, Component and Manifest Model

## 1. Overview

The SDK repository builds coherent, versioned target SDK packages. A package contains multiple compiled pieces and metadata, but PackX installs a tested SDK package with its required dependency closure. The user does not install every Rust crate as a separate package.

Four layers exist:
1. Component: independently built/tested SDK unit.
2. SDK package: curated tested composition for a target or target family.
3. Catalog: signed index of immutable package identities and compatibility.
4. Toolchain: selected by PackX for a specific Engine version, target and project lock state.

## 2. Components versus SDK packages

Components make ownership, tests, provenance and internal versioning clear. They are not necessarily user-facing installation units.

Potential component families:
- common resource inventory and PDC;
- PE stub/overlay;
- MSI database reader/writer and transforms;
- MSIX writer/verifier;
- macOS PKG/DMG;
- Linux DEB/RPM/AppImage/Flatpak;
- px-electron adapter;
- signing/verification adapters;
- target-specific UI and runtime resources.

Do not split every Rust crate into a downloadable component. Keep components together when they always ship, test and update together and splitting them would add loader complexity without real value.

## 3. SDK package identity

An SDK package declares:
- stable package ID and package version;
- target OS/architecture and ABI;
- supported installer formats/operations;
- SDK API compatibility;
- supported Engine range;
- required and optional components;
- dependency constraints;
- artifact hashes and sizes;
- Stable/Nightly identity and provenance;
- license inventory and SBOM.

The SDK package version represents a tested composition. Internal component versions may evolve independently, but the package manifest pins exact versions and full hashes.

## 4. Example identifiers

Illustrative package IDs:
- px-sdk-windows-x64;
- px-sdk-windows-arm64;
- px-sdk-macos-x64;
- px-sdk-macos-arm64;
- px-sdk-linux-x64;
- px-sdk-linux-arm64.

Illustrative component IDs:
- px-stub-windows-x64;
- px-writer-msi-windows;
- px-writer-msix-windows;
- px-electron-windows-x64;
- px-writer-pkg-macos;
- px-writer-deb-linux-x64.

Naming policy is finalized after target slugs and package boundaries are frozen.

## 5. Conceptual package manifest

~~~json
{
  "schema": "packagex-sdk-package",
  "schema_version": 1,
  "package": {
    "id": "px-sdk-windows-x64",
    "version": "2.4.0",
    "channel": "stable",
    "target": {
      "platform": "desktop",
      "os": "windows",
      "architecture": "x86_64"
    }
  },
  "source": {
    "repository": "Package-X/sdk",
    "commit": "full-git-commit",
    "workflow_run": "github-actions-run-id"
  },
  "compatibility": {
    "sdk_api": "1",
    "engine": ">=2.3.0 <2.7.0"
  },
  "components": [
    {
      "id": "px-stub-windows-x64",
      "version": "1.4.2",
      "sha256": "full-sha256",
      "size": 123456
    }
  ],
  "artifacts": [
    {
      "path": "lib.exe",
      "role": "primary-entry",
      "sha256": "full-sha256",
      "size": 123456
    }
  ],
  "licenses": ["licenses/THIRD-PARTY-NOTICES.txt"]
}
~~~

The final schema must define canonical encoding, signature envelope, digest algorithm, URL policy and backward-compatibility rules.

## 6. Artifact inventory

Every artifact records:
- normalized relative path;
- role and file type;
- target OS/architecture where relevant;
- byte size and full SHA-256;
- component ownership;
- executable/library/data classification;
- optional signature/provenance reference.

Never serialize absolute build-machine paths, signing secrets, local usernames or temporary directories.

## 7. Dependencies and minimal installation

Dependencies are explicit and bounded.

- Required dependencies are installed with the SDK package.
- Optional components install only when the Engine/project requests a capability that needs them.
- Dependency ranges resolve to exact immutable package/component versions at installation time.
- Cycles are rejected during catalog validation.
- One target package must not silently pull unrelated OS targets.
- PackX verifies the complete dependency closure before registration.

This supports the user requirement that the Engine downloads only what is actually needed.

## 8. Immutability and correction

Published stable and nightly artifacts are immutable:
- never replace bytes behind an existing stable version;
- never move a nightly commit identity to different content;
- a rebuild with different bytes receives a new identity or fails;
- withdrawals are catalog metadata, not silent history deletion;
- corrections require a new version or new commit identity.

## 9. License and provenance

Include:
- SPDX license identifiers for PackageX-owned components;
- upstream license/third-party notices;
- MPL-2.0 notices for affected vendored macOS sources and modifications;
- SBOM (SPDX or CycloneDX);
- source repository/full commit;
- compiler/dependency identity;
- checksum manifest;
- release workflow/run identity.

Never re-label upstream licenses. Preserve file-level copyleft obligations.

## 10. Pre-publication package validation

Before catalog eligibility:
- validate manifest schema;
- verify every artifact size/hash;
- verify required component closure;
- confirm target/architecture consistency;
- validate Engine/API compatibility;
- ensure no secrets or host paths are embedded;
- validate license/provenance completeness;
- run SDK load/capability tests;
- run representative create/inspect/modify/verify tests;
- generate signed package metadata.

## 11. Acceptance criteria

- [ ] Package installs without resolving component versions dynamically at load time.
- [ ] Exact component versions and digests are pinned.
- [ ] Optional components are not installed unless needed.
- [ ] Every target package declares OS/architecture/ABI.
- [ ] Stable and Nightly are the only channels.
- [ ] Manifest records source commit, hashes, license and compatibility.
- [ ] Published identities are immutable.
