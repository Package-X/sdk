# Target Adapters and Compiled SDK Package Layout

## 1. Purpose

SDK releases are platform/architecture-specific packages containing every compiled piece required for their declared operations. A target package is a coherent install unit; its internal parts may be executables, libraries, stubs, writers, validators, format data, templates and metadata.

PackX installs the package and required dependency closure. The Engine selects the target and invokes the SDK capability required by the build.

## 2. Target identity

A target is more specific than an OS name. Its identity includes:
- platform family/profile;
- operating system;
- CPU architecture;
- ABI/runtime constraints where relevant;
- supported installer formats;
- minimum host/toolchain constraints for cross-build operations.

Illustrative target identities:
- desktop / x86_64 / windows;
- desktop / aarch64 / windows;
- desktop / x86_64 / linux;
- desktop / aarch64 / linux;
- desktop / x86_64 / macos;
- desktop / aarch64 / macos.

The canonical path follows the requested pattern:

~~~text
sdk/<platform>-<architecture>-<os>/
~~~

The exact slug vocabulary is still open. Descriptive slugs must not be confused with compiler target triples.

## 3. Required output layout

The manual release workflow stages target output as:

~~~text
sdk/
└── <platform>-<architecture>-<os>/
    ├── lib.exe
    ├── writers/
    ├── formats/
    ├── stubs/
    ├── resources/
    ├── templates/
    ├── metadata/
    ├── manifest.json
    ├── checksums.json
    ├── licenses/
    ├── provenance/
    └── verification/
~~~

The specifically requested primary path is sdk/<platform>-<architecture>-<os>/lib.exe. The SDK package is not limited to this file; it may need many libraries, writer binaries, stubs, runtime data, templates, manifests and license notices.

The exact meaning and binary type of lib.exe remains open. A Windows PE executable is not a portable artifact for macOS or Linux. Until the naming/entrypoint decision is finalized, releases must not place a non-Windows binary behind a misleading .exe extension. A platform-specific main artifact may use an appropriate extension while preserving the requested layout convention.

Generated binaries belong in workflow artifacts and immutable GitHub Release assets by default, not ordinary source commits.

## 4. Artifact roles

### Primary SDK entry

The entry artifact is how PackX/Engine discovers the SDK package. It may eventually be a worker executable, dispatcher or library facade. The current plan does not force in-process loading or worker-process execution.

### Writers and format adapters

Potential components include:
- MSI database writer/reader and transform engine;
- MSIX/OPC writer and block-map verifier;
- PE stub and overlay writer;
- macOS PKG/DMG writer;
- Linux DEB/RPM/AppImage/Flatpak writer;
- Electron application analyzer and package adapter;
- signing and verification adapters;
- common payload deduplication/compression engine.

The exact list is capability-driven. Do not include platform components in unrelated target packages without a concrete cross-target dependency.

### Stubs and launchers

Precompiled stubs are target artifacts. ABI, architecture, subsystem, signing and footer invariants belong in the manifest and are verified at package assembly.

### Data resources

Schemas, templates, validation tables, UI assets, compression dictionaries and format metadata are versioned package resources. A build must not load undeclared or unverified resources.

### Metadata and provenance

Each package includes:
- package manifest and component index;
- full cryptographic checksums;
- source repository and full source commit;
- compiler and build-toolchain identity;
- dependency/license inventory;
- SBOM and provenance;
- compatibility declaration;
- test/verification report.

## 5. Component boundaries

A component is a coherent testable/versionable unit inside an SDK package. It is not automatically a separately installed user-facing package.

Good boundaries: a target stub family, MSI writer, Electron adapter, UI backend, signing helper with a stable dependency boundary, common PDC engine.

Poor boundaries: every Rust crate, a single utility function, or units whose split adds more loader complexity than value.

The SDK package manifest pins exact component versions and hashes. A released package is a tested composition.

## 6. Build host versus output target

The build host is where PackX, Engine and SDK are executing. The output target is what the Engine is creating.

Some operations are portable cross-host operations. Others require host-specific APIs, toolchains or signing services. The SDK handshake must declare host restrictions so the presence of a target package is not mistaken for a guarantee that every operation works on every host.

## 7. Loading and installation safety

- PackX provides the verified SDK package root to the Engine/SDK loader.
- The Engine uses package metadata to locate components rather than guessing paths.
- Installed SDK packages are immutable after verification.
- Mutable caches and per-build scratch data live outside the installed SDK tree.
- An SDK component must never modify its own installed package directory.
- Concurrent builds use request-specific output/staging locations.

## 8. Resource ownership

- Engine owns source project resources and the canonical inventory.
- SDK owns temporary transformed resources created during the request.
- SDK returns output artifacts with explicit ownership/lifetime metadata.
- PackX owns installed SDK files and package cache metadata.
- PackageX Runtime owns end-user installed application/runtime files.

Avoid redundant copies. Use streaming providers and verified immutable content references where safe.

## 9. Publication acceptance

A target SDK package is eligible only when:
- every binary matches declared OS, architecture and ABI;
- all dynamic dependencies are included or declared;
- the entry artifact loads every declared component;
- runtime capability handshake matches the package manifest;
- declared format conformance tests pass;
- artifact hashes and sizes match;
- there are no undeclared files or machine-specific paths;
- license and provenance records are complete;
- offline installation from the release archive works.

## 10. Open decisions

1. Canonical target slug names.
2. Exact role and platform-specific name/type for lib.exe.
3. Static linking, dynamic plugin loading or worker-process model.
4. Which operations need native host APIs versus portable implementations.
5. One archive per target or a signed manifest referencing independently hashed sub-artifacts.


## 11. Installer scope and embedded-helper capability

Each target adapter reports native installation-scope and execution capabilities independently. A target package must not claim that the same install helper, privilege model or Electron runtime reconstruction works identically across all formats.

The capability handshake includes:
- supported install scope values: user, machine, and choice where meaningful;
- native package staging versus app registration/provisioning behavior;
- privilege requirements for installation and update operations;
- whether an installer-embedded restricted helper can execute;
- whether Electron application/runtime separation and reconstruction are supported;
- whether post-install mutation would invalidate native signatures or package immutability;
- scheduler, uninstaller and registration scope support.

The SDK's [Installation Scope plan](installation-scope.md) owns the cross-format scope matrix. Its [Electron plan](electron.md) owns the format-specific runtime-restoration matrix. The target adapter must return unsupported for combinations it cannot preserve safely; the Engine decides whether an explicitly permitted bundled fallback is acceptable.

The embedded installer helper is a target artifact packaged into each installer that needs its capabilities. It is not a global PackageX runtime manager and is not installed as an independent PackageX-branded executable. Its allowed writes are constrained to authenticated logical payload slots in the active install transaction.

