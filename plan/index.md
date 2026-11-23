# PackageX SDK — Master Planning Index

## 1. Purpose and ownership

This repository specifies and will eventually contain the build-side SDK packages consumed by the PackageX Engine. SDK packages perform target-specific work to turn a validated Engine build plan into installer artifacts.

| System | Owns | Does not own |
|---|---|---|
| PackageX Engine ([packagex](https://github.com/Package-X/packagex)) | Project parsing, validation, normalized build IR, target selection, resource inventory, build orchestration and artifact acceptance | Global SDK installation or every target writer implementation |
| PackageX SDK (this repository) | Target SDK components, writers, format adapters, SDK-side resource transforms, Electron analysis/package preparation, verification and build reports | Project policy, global SDK installation or Engine version routing |
| PackX ([packx](https://github.com/Package-X/packx)) | Thin shim, Toolbelt installation, SDK acquisition, compatibility lookup, CAS, locks, offline import/export and Toolchain selection | Replacing project semantics or implementing all target formats |
| Embedded installer helper (inside each installer) | Executes only installer-authorized operations such as exact Electron runtime acquisition and restoration | Global runtime service, package manager, or end-user PackageX executable |

## 2. Architectural baseline

### 2.1 Engine orchestrates; SDK components perform

The Engine sends a versioned BuildRequest with resolved project/build information, target, resource references, format options, output policy and integrity expectations. The SDK returns a BuildResult with artifacts, hashes, sizes, diagnostics, capabilities, metadata and verification results. The Engine must never need to parse terminal output to determine success.

### 2.2 SDKs are coherent packages with multiple parts

A target SDK is installed as a coherent, versioned package. It may contain executable workers, libraries, writers, stubs, format resources, templates, validators, metadata and license notices. PackX installs the package and its dependency closure, not arbitrary Rust crates one by one.

### 2.3 Creation and editing are first-class

A writer can create a new artifact or open a supported existing artifact for controlled logical changes. It validates preconditions, applies format-specific mutations, finalizes, verifies and commits. An MSI writer and a PKG writer share a lifecycle contract, not an artificial byte-level editing algorithm.

### 2.4 No speed tiers

No Fast / Medium / High build-speed selector is planned. The default path should be efficient and safe. Reproducibility, verification, signing, offline operation and debug diagnostics are meaningful behavior options, not arbitrary performance tiers.

### 2.5 Stable and Nightly only

- Stable: semantic-versioned production packages.
- Nightly: source-commit identified packages.
- Beta: not a release channel.

~~~text
px-stub-windows-x64@1.4.2
px-stub-windows-x64@nightly.a1b2c3d4
~~~

The full commit SHA, internal semantic version metadata, workflow run, target and hashes remain in signed metadata. The eight-character prefix is the visible Nightly download identity and must resolve unambiguously.

## 3. Document map

| Document | Focus |
|---|---|
| [Build contract](architecture/build-contract.md) | Build lifecycle, Engine inputs, SDK outputs, request/result models, transport boundary and performance behavior |
| [Writers and transactions](architecture/writers-and-transactions.md) | Common writer lifecycle, typed changes, staged transactions, MSI database editing, MST transforms and signatures |
| [Target adapters](architecture/target-adapters.md) | Target dimensions, platform writers, output directory, primary SDK artifact and auxiliary pieces |
| [Electron](architecture/electron.md) | Electron detection, app-only payload optimization, opaque runtime reference, restricted installer helper, immutable runtime restoration and updater staging |
| [Installation scope](architecture/installation-scope.md) | Per-user and machine-wide policy, native format capability matrix and cross-OS scope constraints |
| [Payload optimization](architecture/payload-optimization.md) | Whole-file deduplication, content-defined chunks, Zstandard policy and build reports |
| [Package/component model](architecture/package-and-component-model.md) | Components, target packages, manifests, dependencies, immutability and provenance |
| [Compatibility/catalog](architecture/compatibility-and-catalog.md) | API/Engine compatibility, Stable/Nightly identities, signed catalogs and revocation |
| [Manual release workflow](release/manual-release-workflow.md) | Manual Actions dispatch, target matrix, compile/test/package/publish and catalog promotion |
| [Upstream integration map](implementation/upstream-integration-map.md) | Concrete upstream crates/tools, API capabilities, platform build recipes, licensing, known gaps and proof-of-concept sequence |

## 4. Implementation approach and evidence

The SDK uses established open-source format implementations behind PackageX adapters. The new [Upstream Integration Map](implementation/upstream-integration-map.md) records the specific candidates and how to integrate them. It labels verified capabilities, candidates, external workers and known gaps; it is not a dependency lockfile and does not claim the adapters have been implemented. Upstream versions, feature flags, licenses and maintenance status must be rechecked before pinning.

The intended pattern is to reuse libraries for file/container formats, use native platform tools for authoritative final validation/signing, and write PackageX glue only where the upstream interface does not cover the required Engine contract.

## 5. End-to-end build pipeline

1. Engine validates the .px project and creates a normalized build request.
2. PackX ensures that a compatible SDK package is installed and selects a Toolchain.
3. Engine queries SDK identity/capabilities and provides the request plus controlled resource access.
4. SDK validates inputs and produces a target-specific operation plan.
5. SDK prepares resources and writer state.
6. Writers create or modify artifacts.
7. SDK finalizes and verifies outputs.
8. SDK returns a structured BuildResult and artifact inventory.
9. Engine checks the result against project policy and completes build orchestration.

The process boundary (in-process library, worker executable, or both behind a transport-neutral contract) remains an implementation decision. The request/result semantics must not depend on that transport.

## 6. Release output layout

The release workflow stages target output beneath the requested path pattern:

~~~text
sdk/
└── <platform>-<architecture>-<os>/
    ├── lib.exe
    ├── writers/
    ├── formats/
    ├── stubs/
    ├── resources/
    ├── metadata/
    ├── manifest.json
    ├── checksums.json
    ├── licenses/
    └── provenance/
~~~

The requested primary artifact location is **sdk/<platform>-<architecture>-<os>/lib.exe**. Its exact role and platform-specific naming must be settled before implementation; a Windows .exe is not a portable executable for macOS/Linux. The package contains all additional compiled/data artifacts required by its declared capabilities.

Generated binaries should normally be uploaded as Actions artifacts and immutable GitHub Release assets, not committed as ordinary source files.

## 7. Decisions captured

- [x] Engine and SDK communicate through a versioned two-way contract.
- [x] Engine orchestrates; SDK components perform target-specific work.
- [x] Writers support creation and format-aware modification.
- [x] MSI editing uses database-aware operations and transaction/verification requirements.
- [x] Electron app payload and runtime are distinct when safe.
- [x] Electron runtime is exact-version/target/integrity identified.
- [x] SDK distributions are coherent target packages containing multiple pieces.
- [x] Stable and commit-identified Nightly are the only channels.
- [x] SDK releases are manually initiated through GitHub Actions.
- [x] No user-facing build speed tiers.
- [x] Unneeded target components must not be downloaded merely because they exist.

## 8. Open decisions before implementation

1. Exact role of lib.exe and corresponding names/types for macOS/Linux.
2. In-process, worker-process or dual transport for the SDK contract.
3. Which formats support create, inspect, modify, transform and verify in the first release.
4. Which portable MSI database backend satisfies editing/conformance requirements, and where Windows Installer APIs are used.
5. How Electron fuses, branding, signing and native modules affect runtime-sharing identity.
6. Exact SDK package/component versioning and compatibility range syntax.
7. Canonical target slug vocabulary.
