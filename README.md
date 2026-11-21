# PackageX SDK

**The target-specific build SDK used by the PackageX Engine to create, inspect, edit, verify, and emit installer artifacts.**

The SDK is the build workforce of PackageX. The Engine owns project interpretation and build orchestration; SDK components perform format-aware operations and return artifacts, structured reports, diagnostics, and metadata to the Engine.

> **Project status:** Planning repository. The repository currently contains the project license and this planning/documentation baseline; no SDK implementation is claimed by this README.

## Responsibility boundary

- **PackageX Engine** parses and validates projects, resolves target intent, prepares resources and build plans, selects SDK capabilities, and decides whether returned artifacts satisfy the build.
- **PackageX SDK** implements target-specific writers, format adapters, resource handling, compression/deduplication, Electron analysis and packaging, artifact verification, and build reporting.
- **PackX** installs and maintains Engine versions and SDK packages, resolves compatibility, and constructs a compatible Toolchain for a build.
- **PackageX Runtime** executes installer behavior on the end-user system. Build-time SDK tooling is not a prerequisite for an installed application.

## Core design rules

1. SDKs are distributed as coherent target packages containing multiple compiled pieces, not as one universal binary or a pile of independently installed Rust crates.
2. The Engine and SDK communicate through a versioned, two-way build contract. The Engine supplies structured inputs; the SDK returns structured outputs and diagnostics.
3. Writers support artifact creation and format-aware modification. Mutable formats use explicit transaction, validation, commit, and rollback semantics.
4. MSI editing is database-aware. PackageX must not patch arbitrary byte offsets or promise physical byte-range preservation.
5. Electron application payloads and Electron runtime payloads are separate deliverables when safe. Runtime acquisition uses an exact version, OS, architecture, and integrity identity.
6. Payload optimization uses deduplication and compression only where it produces a real benefit. Native installer formats retain native structure and semantics.
7. Stable SDK packages use semantic versions. Nightly packages are visibly identified by an eight-character source commit, with full commit and internal semantic-version metadata retained. There is no Beta channel.
8. SDK release publication is manually initiated and controlled through GitHub Actions.

## Planning map

- [Master SDK plan](plan/index.md)
- [Build pipeline and Engine/SDK contract](plan/architecture/build-contract.md)
- [Writer lifecycle and transactional editing](plan/architecture/writers-and-transactions.md)
- [Target adapters and compiled SDK layout](plan/architecture/target-adapters.md)
- [Electron application and runtime handling](plan/architecture/electron.md)
- [Payload deduplication and compression](plan/architecture/payload-optimization.md)
- [SDK packages, components and manifests](plan/architecture/package-and-component-model.md)
- [Compatibility and catalog model](plan/architecture/compatibility-and-catalog.md)
- [Manual release workflow](plan/release/manual-release-workflow.md)
- [Upstream integration map and implementation recipes](plan/implementation/upstream-integration-map.md)

## Related repositories

- [PackageX Engine and central architecture](https://github.com/Package-X/packagex)
- [PackX Toolbelt and Toolchain manager](https://github.com/Package-X/packx)

Examples in the plan are proposed contracts or illustrations unless explicitly marked as existing behavior. Open decisions are called out rather than silently treated as implemented.
