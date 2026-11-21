# Electron Application Adapter and Managed Runtime

## 1. Goal and ownership

Electron support is a first-class SDK responsibility. The goal is to package Electron apps with a small application payload where possible, avoid bundling duplicate Chromium runtimes, and provide safe runtime acquisition.

The Engine supplies project intent/resources. The SDK inspects and prepares Electron application files and writes a runtime requirement. PackageX Runtime handles runtime installation/acquisition on the end-user machine. PackX manages developer SDK packages, not application Electron runtimes.

**The optimization is application/runtime separation. It is not removing Chromium bytes from an arbitrary executable.**

## 2. How ordinary Electron distributions are structured

A conventional Electron distribution has:
- an Electron executable and companion runtime files;
- a resources directory;
- the developer application under resources/app or resources/app.asar;
- possible app.asar.unpacked content, native modules, locales and other resources.

On macOS, the app payload is normally under Electron.app/Contents/Resources. Exact layout differs by platform and packaging tool.

The Chromium engine is compiled into the Electron executable and runtime distribution. There is no general safe section of app.exe that PackageX can cut out and later paste back. PackageX must not patch PE sections or remove arbitrary binary regions to manufacture a stock runtime.

Reference: [Electron Application Distribution](https://www.electronjs.org/docs/latest/tutorial/application-distribution).

## 3. SDK component: px-electron

px-electron is build-time tooling. It provides:
- Electron layout detection and confidence reporting;
- version, OS and architecture identification;
- application resource inventory;
- ASAR and unpacked-directory analysis;
- native Node module and ABI inventory;
- runtime descriptor generation;
- exact runtime package resolution for bundled builds;
- runtime digest/signature verification;
- launcher/layout planning;
- capability diagnostics for unsupported/customized app bundles.

Inspection must not execute the application or arbitrary package lifecycle scripts. A developer-approved build step is a separate explicit operation.

## 4. Supported input classes

### A. Standard packaged Electron folder

Analyze a folder containing the renamed Electron executable, runtime companions, resources/app.asar or resources/app, optional app.asar.unpacked, locales and application resources. This is the preferred managed-runtime source.

### B. Source project or package output

When project metadata and packaging configuration are available, identify the exact Electron dependency/version, target architecture, app resources and native module requirements. Respect lockfiles and package-manager resolution. Do not run scripts during passive inspection.

### C. Rebranded/custom Electron package

A package may rename electron.exe, change icons/version resources, modify crash reporter names, flip Electron fuses or carry a custom Electron build. Detectable runtime identity and customization must be reported. If exact version/build identity cannot be proven, do not silently substitute stock Electron.

### D. Opaque single-file or self-extracting EXE

Treat as opaque unless a dedicated trusted adapter can safely inspect it. Do not claim Chromium can be safely extracted from an arbitrary executable. Fall back to packaging the whole application unchanged or report managed-runtime mode unsupported.

## 5. Application payload

The application payload contains developer-owned code/assets:

~~~text
application/
├── app.asar
├── app.asar.unpacked/
├── native-modules/
├── resources/
└── application-manifest.json
~~~

It may include JavaScript bundles, HTML/CSS/themes, images/fonts/locales, package metadata, ASAR data, native .node modules, sibling DLL/shared libraries and additional resources.

Do not omit a file just because it is outside app.asar. Some resources and native modules deliberately live beside the archive. Preserve logical relative paths, executable bits/file modes and all required companion files.

Inventory each path, size, digest and role. Store a canonical source hash so post-build reconstruction can be verified.

## 6. Runtime payload

An Electron runtime package includes the exact Electron distribution:
- Electron executable or macOS application bundle;
- Chromium/V8 and embedded Node.js;
- helper applications and companion DLLs/frameworks/shared libraries;
- locale/resource packs expected by the release;
- runtime license and third-party notices.

Identify it by more than major version. The key includes exact Electron version, OS, architecture, upstream artifact identity/archive digest, build customization identity, fuse profile when changed, and ABI requirements.

## 7. Delivery modes

| Mode | Behavior | Trade-off |
|---|---|---|
| Bundled | Put the complete Electron runtime in the installer | Large installer, works offline |
| Managed | Install app payload and reuse an exact runtime in the PackageX Runtime Store; download during setup if absent | Small app installer, runtime may need network, shared storage possible |
| Deferred | Install app payload and download runtime on first launch | Fast initial setup, first launch requires network unless cached |

Managed is the default optimization candidate for online installs. Bundled remains the safe choice for offline, customized or self-contained distributions. Deferred mode is explicit opt-in and requires clear first-launch progress, cancellation, retry and offline errors.

Deferred acquisition postpones bandwidth cost; it does not remove Chromium or necessarily reduce total installed size.

## 8. Runtime download source and integrity

Use official Electron prebuilt release artifacts by default. The Electron downloader @electron/get resolves artifacts by exact version/platform/architecture and supports mirrors, caching and progress.

References:
- [@electron/get](https://github.com/electron/get)
- [Electron installation](https://www.electronjs.org/docs/latest/tutorial/installation)
- [Electron distribution layout](https://www.electronjs.org/docs/latest/tutorial/application-distribution)

Download pipeline:
1. Resolve exact artifact from a trusted, signed runtime descriptor/catalog.
2. Use HTTPS and an allowlisted official source or explicitly trusted mirror.
3. Stream to temporary storage with bounded buffers and progress.
4. Verify archive digest against signed PackageX metadata.
5. Verify upstream provenance/checksums where available.
6. Verify platform signatures where applicable.
7. Extract with path traversal, unsafe symlink, file-count and expanded-size protections.
8. Verify extracted inventory.
9. Atomically register an immutable runtime identity.

A mirror is a transport, not a trust root. A hash served only beside an untrusted archive is not sufficient.

## 9. Why Electron is large and where savings come from

Electron includes Chromium and Node.js, so its runtime remains substantial. PackageX cannot make that dependency disappear while preserving Electron behavior.

Savings come from:
- app-only installer payload when runtime acquisition is separate;
- reuse of an exact already-installed runtime;
- archive/runtime cache reuse;
- content-addressed storage;
- downloading only when the declared app requires it;
- offline bundles when needed;
- future delta updates only after runtime-specific deltas are proven reliable.

Report application payload bytes, runtime download bytes and installed runtime footprint separately. Never call a small bootstrapper a small total installation.

## 10. Shared runtime eligibility

Two applications can share a runtime only when all runtime-affecting identity fields match:
- exact runtime archive digest;
- Electron version, OS and architecture;
- ABI/build identity;
- fuse state;
- binary patches/rebranding affecting runtime bytes;
- complete companion runtime inventory.

Application resources, app.asar, metadata and native modules remain app-specific.

The conventional Electron layout expects the app under the runtime's resources directory. A shared store therefore needs a tested application view/launcher strategy. Candidates include an explicit app-path launch mode where supported, or an immutable per-app view with links/hardlinks to identical runtime files plus application-specific resources.

Do not assume links behave identically on Windows, macOS and Linux. Each platform needs launch, update, code-signing, uninstall and repair tests. Until proven, use an app-local runtime for that platform.

## 11. Fuses, branding and signatures

Electron fuses are package-time binary settings and must be applied before code signing.

- px-electron may inspect/report fuse state.
- Fuse changes are explicit, not silent.
- Fuse-modified runtimes have a different content identity and may not share stock runtimes.
- Application-specific icons, file descriptions, product names and crash reporter naming may require app-local binary metadata/custom runtime.
- A verified runtime store entry is immutable.
- Windows Authenticode and macOS code-signing/notarization behavior must be tested for each delivery mode.

References: [Electron Fuses](https://www.electronjs.org/docs/latest/tutorial/fuses) and [Electron Code Signing](https://www.electronjs.org/docs/latest/tutorial/code-signing).

## 12. Native Node modules

Inventory native .node modules, CPU architecture, declared Electron/Node ABI and external library dependencies. Reject incompatibility or report a required rebuild. Two apps sharing Chromium do not automatically share native modules or application-specific dependencies. Do not infer ABI compatibility from the Electron major version alone.

## 13. Runtime store lifecycle

The managed runtime store belongs to PackageX Runtime, not PackX Toolbelt.

~~~text
PackageX-RuntimeStore/
└── electron/
    └── windows-x64/
        └── <runtime-content-digest>/
            ├── runtime/
            ├── runtime-manifest.json
            ├── checksums.json
            └── licenses/
~~~

A runtime is verified before registration, immutable/read-only afterward where practical, reference-counted by installed applications, repairable by fetching the exact descriptor, and removable only when no installed application/process uses it.

Uninstall removes app payload and decrements the runtime reference. It cannot remove a runtime still used by another application.

## 14. Runtime descriptor

~~~json
{
  "runtime": "electron",
  "version": "exact-semver",
  "os": "windows",
  "architecture": "x86_64",
  "artifact_sha256": "full-sha256",
  "runtime_identity": "content-addressed-identity",
  "download_policy": "managed",
  "application_entry": "application/app.asar",
  "native_module_abi": "declared-abi",
  "fuse_profile": "declared-profile",
  "license_manifest": "licenses/electron.json"
}
~~~

The descriptor is signed as part of the PackageX package trust chain. It is data, never executable code.

## 15. Failure and security behavior

- Never execute a downloaded runtime before digest and applicable signature checks pass.
- Reject path traversal, absolute/device paths, unsafe links and archive bombs.
- Enforce file-count and expanded-size limits.
- Reject version/target mismatch against signed descriptor.
- Install new immutable identity; never overwrite runtime in place.
- On deferred download failure, preserve app payload and present retryable runtime-unavailable state.
- On integrity failure, quarantine/delete temporary data and fail closed.
- Never log signed URL query parameters, credentials or signing material.

## 16. Acceptance criteria

- [ ] Standard Electron folders separate app payload/runtime without modifying runtime executable.
- [ ] Opaque one-file EXEs are not claimed to be safely separable by default.
- [ ] Exact version, target, digest and ABI are recorded.
- [ ] Bundled, Managed and Deferred are distinct delivery policies.
- [ ] Download verifies before extraction/registration.
- [ ] Runtime sharing follows exact identity and platform launch tests.
- [ ] Different fuses/custom runtime bytes do not share stock runtime entries.
- [ ] Signatures/notarization are preserved or reapplied correctly.
- [ ] Runtime reference counting protects other installed apps.
- [ ] App payload and runtime sizes are reported separately.
- [ ] Offline exports can include the exact required Electron runtime.
