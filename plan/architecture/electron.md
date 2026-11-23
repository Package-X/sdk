# Electron Application Support and Payload Optimization

## 1. Goal and ownership

Electron support is a first-class SDK capability. Its purpose is to remove the duplicate Electron runtime from the ordinary application payload when the selected installer format can safely restore that exact runtime during installation.

The final application keeps its original executable and launch layout. PackageX does **not** add a px-electron-launcher, replace the app executable with a shim, install a global Electron manager, or require an end-user PackageX.exe.

| Owner | Responsibility |
|---|---|
| PackageX Engine | Owns project policy, build-prompt orchestration, installation scope, installer lifecycle, helper authorization and update transaction |
| SDK px-electron | Detects Electron, inspects the app, separates payload/runtime, creates the logical runtime reference, emits its protected payload block and reports target capabilities |
| Embedded installer helper | Performs only the operations authorized by the installer's authenticated operation plan |
| Developer updater | Optional developer-supplied updater executable and policy; may customize update discovery and UX while respecting the selected trust/integrity contract |
| PackX | Installs developer-side SDK packages only; it does not acquire or manage Electron runtimes for end users |

Passive inspection must not execute the application or arbitrary package lifecycle scripts.

## 2. Electron detection and build-time prompt

The SDK reports a structured detection result when it recognizes an Electron application. Detection sources include:

- A packaged Electron folder containing the executable, runtime companions and resources/app or resources/app.asar.
- Project manifests and lockfiles that resolve the exact Electron dependency.
- Electron Packager / Forge / electron-builder output layouts where actual files are present.
- A developer-supplied exact runtime identity for a custom Electron distribution.

The result includes confidence, exact version/build identity when provable, OS/architecture, app entry point, runtime-relative paths, ASAR/unpacked resources, native .node modules, fuse state where inspectable, and signature state.

For an interactive build, the Engine/CLI presents the SDK detection result and asks:

~~~text
Hey! We detected that your application uses Electron.

Would you like to use the default Electron optimization settings?
This is recommended.

[y] Yes
[n] No
~~~

- **Yes:** apply the SDK's default safe optimization profile for the selected target format.
- **No:** preserve the ordinary application distribution, including its Electron runtime.
- **Uncertain detection:** do not remove anything; explain what could not be identified.
- **Non-interactive build:** never wait for input. Use an explicit project/build setting or documented non-interactive default. The result must be reproducible and visible in the build report.

Detection and payload preparation belong to px-electron; the Engine owns prompting and project policy. The SDK must not silently override a developer's explicit setting.

## 3. Preserve the original application layout

The SDK separates application-owned payload from exact Electron runtime files, but does not invent a new launch layout.

Logical application payload:

~~~text
application/
├── <original-app-executable>
├── resources/
│   ├── app.asar
│   ├── app.asar.unpacked/
│   └── <other-original-resources>
├── <application-native-modules-and-companions>
└── <other-original-files>
~~~

The installer restores runtime files to the exact relative locations expected by the original application distribution. The original executable remains the entry point and is not replaced by a PackageX launcher.

The runtime reference describes original runtime paths and corresponding installer payload slots. It must not assume every Electron distribution has one universal folder layout.

The SDK preserves:
- Original logical paths, executable permissions/file modes and required companion files.
- The distinction between application-owned resources and Electron-owned runtime files.
- app.asar, app.asar.unpacked, native modules and sibling resources.
- Application-specific metadata, branding and signatures.
- The original Electron executable and its expected runtime-relative layout.

Use @electron/asar when archive inspection or construction is required. Never hide native .node modules or resources Electron must access as ordinary files inside ASAR. Electron ASAR integrity metadata and fuse configuration are application-specific and must be finalized before signing.

References:
- [Electron application distribution](https://www.electronjs.org/docs/latest/tutorial/application-distribution)
- [Electron ASAR integrity](https://www.electronjs.org/docs/latest/tutorial/asar-integrity)
- [Electron Fuses](https://www.electronjs.org/docs/latest/tutorial/fuses)

## 4. The Electron reference is a protected payload object

The build-time logical source descriptor may be named electron-runtime.ref.json, but the final installer must **not** ship that descriptor as a plainly readable JSON file.

The SDK serializes it into an opaque authenticated control block stored beside the application payload in the installer's payload region. It is part of the protected payload, not a public top-level JSON manifest.

Conceptual final installer layout:

~~~text
Installer container
├── native installer metadata
├── application payload blocks
├── protected control block
│   ├── PKX_ELECTRON_REF_V1 envelope
│   ├── authenticated operation-plan identity
│   └── logical layout/slot table
└── native signature / package integrity data
~~~

The reference contains the data needed to restore one exact runtime:

- Exact Electron version and build identity.
- Target OS and architecture.
- Exact version-pinned upstream artifact URL; never a floating latest URL.
- Archive format, expected byte length and full SHA-256 digest.
- Expected runtime file inventory and extracted-tree digest.
- Original application-relative runtime paths and named payload slot IDs.
- Application payload/inventory identity.
- ABI/native-module requirements.
- Fuse profile and runtime customization identity.
- Signature-verification policy and license/notice inventory.
- Reference schema version and installer layout identity.

The descriptor may be represented as JSON during SDK preparation, but the packaged form is encoded, encrypted/authenticated and bound to the final payload layout. The writer resolves named slots to format-specific offsets/streams while building. The helper does not accept arbitrary offsets from a downloaded file or external caller.

### Logical slot IDs, not guessed byte offsets

A stable slot such as electron-runtime-main or electron-runtime-resources identifies a destination in the SDK's canonical payload layout. The target writer maps each slot to the actual PE overlay, MSI stream/cabinet, PKG component or other target-specific storage location.

The reference stores slot identities and a layout digest. A format-specific footer/index resolves physical locations. Never hard-code a raw EXE offset in the portable descriptor: compression, signing, PE certificate tables and format metadata can move physical offsets.

## 5. Default integrity and confidentiality model

PackageX protects the control block by default:

1. **Opaque encoding/encryption:** hide casual inspection of runtime URLs, layout metadata and internal installer instructions.
2. **Authenticated envelope:** reject modified, reordered, truncated or substituted control data before the helper acts on it.
3. **Publisher authorization:** verify a build-time signature/authorization identity rooted in the signed installer or developer update trust configuration.
4. **Native artifact verification:** verify the downloaded Electron artifact's complete SHA-256 against the build-time-pinned digest, then apply OS-native publisher/signature checks wherever that platform supports them.
5. **Scoped writes:** permit writes only to the destination slots authorized by the authenticated plan.

These controls solve different problems. Encryption/obfuscation is not a substitute for a signature, and it does not by itself prevent remote-code execution. A determined owner who can inspect and debug an executable may recover data the executable must decrypt locally. The default opaque envelope protects against casual disclosure and tampering; it is not a claim of secrecy against the machine owner. Existing password/recovery sealing remains the stronger confidentiality option when required.

A URL or SHA-256 fetched at install time from the same untrusted endpoint as the runtime is not an authority. The URL, expected digest and layout authorization are pinned into the build-time protected block. OS signatures add publisher verification; the full digest still identifies the exact artifact, including on Linux where there is no single universal OS code-signing verifier.

## 6. Embedded installer helper

The installer embeds a small target-specific helper. It is a capability-limited part of the installer, not a package engine, background daemon or independently installed PackageX product.

The helper may:
- Read and authenticate the installer's protected control block.
- Download only the exact URL and artifact identity authorized by that block.
- Stream to a bounded staging area.
- Verify expected size, full SHA-256 and applicable native signatures.
- Safely extract the expected archive.
- Verify the extracted file inventory.
- Write only to declared runtime slots inside the installer-owned staging tree.
- Report progress, cancellation and typed errors to the parent installer.

The default helper must not:
- Accept arbitrary URLs, file paths, raw offsets or memory addresses from an external caller.
- Execute downloaded scripts, DLLs or executables as installation instructions.
- Modify unrelated installed applications or arbitrary executables.
- Become a general-purpose shell or package manager.
- Run when detached from a valid installer operation plan.

Developer-authored .s.px hooks continue to use the existing constrained WASM/capability model. A developer-supplied custom updater executable is a separate, explicitly trusted program with its own update key and policy; it must not turn the restricted helper's write interface into an unrestricted injection API.

For each format, the helper must use the native install mechanism actually permitted. Do not assume an identical helper hook exists in EXE, MSI, MSIX, PKG, DMG, DEB, RPM, AppImage and Flatpak.

## 7. First-install runtime acquisition

For targets that declare the app-only runtime-restoration capability:

1. Authenticate the outer installer and protected control block.
2. Validate that the reference target matches the host and selected install scope.
3. Resolve the exact URL from the protected reference.
4. Download the artifact to a same-volume staging area using bounded streaming.
5. Validate expected length and recompute the complete SHA-256.
6. Verify the platform publisher signature where applicable. Never treat a successful HTTPS connection as artifact authenticity.
7. Safely extract with path traversal, absolute/device path, unsafe-link, collision, file-count and expanded-size checks.
8. Verify the extracted runtime inventory against the protected reference.
9. Reconstruct the original application layout in the transaction staging tree.
10. Re-verify application files and signatures, then let the Engine commit the complete install transaction.

No app executable is launched before the required runtime has been restored and verified. Failure deletes/quarantines staged files and leaves an existing application untouched.

There is no global end-user Electron cache, shared runtime store, runtime reference counter or PackageX Runtime service. The currently installed Electron files are part of that application's installed generation. The SDK may maintain a separate developer-side verified download cache to avoid downloading the same build repeatedly during developer builds.

## 8. Updates: keep the current runtime, stage one next runtime

The default PackageX Electron update integration follows immutable application generations:

- The current installed application keeps the exact Electron runtime it was built and signed for.
- The updater reads the next release's application manifest. It downloads a runtime only when that release requires a different exact Electron identity.
- It may stage **one** pending runtime: the exact runtime required by the latest applicable update.
- The currently running application continues using its current runtime throughout background download, digest/signature verification and extraction.
- The updater waits for the application to exit before committing the new application payload and runtime together.
- The new generation is launched and health-checked before the previous generation is removed.
- If download, verification, commit or health-check fails, the updater preserves/restores the previous generation.
- If a newer release supersedes an already staged release, the obsolete pending generation is discarded. Do not accumulate historical Electron versions.

The current installed runtime is an application dependency, not a cache. The one pending runtime is temporary update staging, not a reusable system-wide runtime collection.

See the Engine [updater and launcher plan](https://github.com/Package-X/packagex/blob/main/plan/architecture/updater-and-launcher.md) and [runtime/rollback plan](https://github.com/Package-X/packagex/blob/main/plan/architecture/runtime-and-rollback.md).

## 9. Target-format compatibility and signature constraints

App-only runtime restoration is a capability of the selected installer format, not a universal property of every package container.

| Target | App-only Electron runtime restoration | Constraint / current planning rule |
|---|---|---|
| Windows custom EXE overlay | Candidate | Embedded helper can restore runtime into a same-volume staged app tree before commit. Preserve PE/Authenticode footer and signing order. |
| Windows MSI | Candidate | Use a carefully authored custom-action/helper path or pre-install bootstrap stage; test rollback, repair and install contexts with Windows Installer. Never bypass MSI transaction semantics. |
| Windows MSIX | Not supported by direct install-time insertion | Package files are OS-managed, read-only and package-signed. A helper cannot mutate the installed package. Keep Electron bundled unless a separately signed native dependency/package design is proven. |
| macOS PKG | Conditional / not approved by default | PKG scripts can run, but adding runtime files into an already signed app bundle breaks the outer bundle's resource seal. Keep runtime bundled unless the application is deliberately built for a verified external-runtime layout that preserves code-signing/notarization. |
| macOS DMG drag-and-drop | Not supported by direct helper execution | Copying a bundle from a mounted disk image does not run a PackageX installation helper. Keep Electron bundled for ordinary drag-and-drop DMG distribution. |
| Linux DEB/RPM | Candidate | Native package install scripts can invoke a constrained helper and restore into an app-local generation. The native package manager controls system installation and privileges; test rollback/upgrade behavior per distribution. |
| Linux AppImage | Not supported by ordinary AppImage execution | AppImage is a directly run application image, not an install transaction that automatically invokes an embedded helper. Keep Electron bundled unless a separately declared installer/bootstrap workflow is selected. |
| Linux Flatpak | Not supported by direct payload mutation | Flatpak deployments are managed immutable OSTree deployments. Use bundled Electron or a separately designed Flatpak runtime/extension dependency, not a helper that edits deployed package files. |

Windows MSIX and macOS signing constraints are intentional limitations, not invitations to patch signed packages after installation. The SDK must report unsupported or select the unoptimized bundled fallback only when the developer's policy explicitly permits that fallback.

For macOS, Apple's code-signing seal covers bundle resources and nested code. A runtime archive downloaded after signing cannot simply be inserted into the signed .app and still be assumed valid. For MSIX, Windows stages package files read-only and protects their integrity; app registration and machine provisioning are OS deployment operations.

References:
- [Windows MSIX package behavior](https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-behind-the-scenes)
- [Apple code-signing procedures](https://developer.apple.com/library/archive/documentation/Security/Conceptual/CodeSigningGuide/Procedures/Procedures.html)
- [Debian Policy: maintainer scripts](https://www.debian.org/doc/debian-policy/ch-maintainerscripts)

## 10. ASAR, fuses, native modules and signing

- Inspect/report Electron fuse state. Fuse changes are explicit and occur before final signing.
- ASAR integrity metadata must match the final app.asar and must be bound to the app's release identity.
- ASAR integrity metadata is app/release-specific. If the selected Electron version embeds the ASAR hash into an executable resource or signed bundle metadata, that modified binary is no longer the stock upstream artifact and must not be fetched/installed as though it were unchanged.
- Native .node modules remain app-specific and must match the exact Electron ABI/architecture.
- If ASAR integrity, fuses, branding or another customization changes runtime executable bytes, the result is an application-specific runtime artifact. Its exact final digest, signature and acquisition source must be supplied/approved at build time; the installer must not silently download the stock upstream archive and assume it matches.
- App-only optimization for a customized runtime is allowed only when the SDK can prove the complete final runtime generation can be reconstructed and verified (for example, using a developer-hosted exact customized archive with its own pinned digest/signature). Otherwise preserve that runtime with the bundled application and report the optimization as unsupported for that target.
- Never patch the original executable or mutate signed application content on the end-user machine merely to fit the PackageX payload format.

## 11. Open-source SDK integrations

All listed integrations are build-side SDK dependencies or references. The installed helper is a native, bounded implementation and does not require Node.js.

| Project | SDK usage |
|---|---|
| @electron/asar | Inspect/build/extract ASAR application payloads and integrity metadata |
| @electron/get | Reference/integration for exact Electron artifact resolution, download caching, mirrors and progress during SDK-side preparation |
| @electron/packager | Reference for Electron distribution layouts, application resources and supported packaging behavior |
| @electron/rebuild | Rebuild application-specific native modules for the exact Electron ABI |
| @electron/fuses | Inspect/apply explicitly selected Electron fuse profiles before signing |

The SDK's developer-side download cache is distinct from installed application files. It stores verified exact artifacts for repeated builds and is not shipped as a runtime service.

## 12. Acceptance criteria

- [ ] Standard Electron inputs are detected without executing the application or package scripts.
- [ ] Interactive build asks whether to apply recommended Electron optimization settings; non-interactive behavior is explicit and deterministic.
- [ ] Declining optimization preserves the original complete Electron distribution.
- [ ] Optimized output preserves the original app executable and application-relative layout; no PackageX launcher is inserted.
- [ ] The final installer contains an opaque authenticated Electron reference/control block, not a loose readable JSON sidecar.
- [ ] The reference pins exact version, target, URL, size, full SHA-256, inventory identity and named payload slots.
- [ ] The embedded helper is capability-limited and cannot write to arbitrary offsets or paths.
- [ ] Downloaded artifact and extracted inventory are verified before commit/launch.
- [ ] No global end-user runtime cache/store/service is created.
- [ ] Update staging retains only the current installed generation and one next pending runtime.
- [ ] The old app/runtime remains untouched until the new generation commits and passes health checks.
- [ ] MSIX, macOS signing, DMG, AppImage and Flatpak limitations are reported rather than hidden.
- [ ] App-only optimization is never advertised for a format until its native install/signing tests pass.
