# Upstream Integration Map: Reusing Existing Installer and Packaging Projects

> Research baseline: 21 November 2026. Upstream versions and project health are volatile. The versions below are observations from current upstream documentation, not pins for a future Cargo.lock. Before implementation, re-check tags, source, licenses, advisories, maintenance and API compatibility.

## 1. Objective

PackageX should not reimplement mature installer formats merely because the Engine is written in Rust. The SDK's job is to provide a consistent PackageX interface around reliable format-specific implementations.

The preferred strategy is:

1. Reuse an established library when it exposes the operation PackageX needs.
2. Build a small Rust adapter around that library.
3. Use an external, pinned upstream tool when the format is too broad or depends on a native platform toolchain.
4. Implement only the missing glue, PackageX policy, deterministic resource pipeline and validation.
5. Keep an upstream tool replaceable behind the SDK adapter.
6. Run actual native OS validation tools in CI for the platforms whose installer engines ultimately consume the artifact.

This is the same general strategy already intended for macOS: use the existing Apple-platform Rust ecosystem for XAR, CPIO, BOM, Mach-O signing, package assembly and DMG operations; add PackageX adapters and fill only demonstrated gaps.

**Important:** an upstream library being able to create a package does not automatically mean it can inspect, edit, sign, verify or install every package it creates. Each operation is a separate declared SDK capability.

## 2. Evidence labels

Use these labels in dependency reviews and Cargo manifests:

- **Verified capability:** current upstream documentation explicitly exposes the operation.
- **Good candidate:** the project appears to cover the required format, but PackageX still needs a proof-of-concept and corpus tests.
- **External adapter:** PackageX generates input/configuration and invokes a pinned upstream tool in a controlled runner.
- **Gap:** a known missing capability that needs a bounded PackageX implementation or another upstream dependency.
- **Do not assume:** a feature that must not be advertised until tested.

Do not turn a Good candidate into a hard runtime dependency before the proof-of-concept passes.

## 3. Recommended component map

| PackageX SDK component | Upstream foundation | Initial responsibility | Important boundary |
|---|---|---|---|
| px-writer-msi | msi, cfb, cab; Windows Installer API on Windows | Create, inspect and modify supported MSI databases | Portable crate edits use staged-copy safety; native API handles Windows transaction/transform workflows |
| px-writer-msix | Microsoft MSIX Packaging SDK | Create and inspect MSIX/APPX packages across host platforms | Build the SDK's MSIX_PACK feature; signing and Windows installation validation are separate |
| px-writer-pe | Existing PackageX PE overlay plus goblin/object readers as appropriate | Assemble PackageX EXE installers and inspect PE structure | Preserve PxFooter/Authenticode certificate-table invariants |
| px-apple-bundle | apple-bundles, plist support and existing Rust filesystem tooling | Construct .app bundle trees and metadata | Bundle construction is not a signature or Gatekeeper validation |
| px-writer-pkg | apple-flat-package, apple-xar, cpio-archive, apple-bom | Build component/product .pkg archives | BOM writer completeness must be proven; do not silently emit incomplete BOMs |
| px-writer-dmg | apple-dmg | Construct .dmg images | Validate the output on macOS using Apple's disk-image tools |
| px-apple-sign | apple-codesign | Mach-O/bundle signing, notarization requests and ticket stapling | Apple's own execution assessment remains authoritative |
| px-writer-deb | debian-packaging; cargo-deb as reference or optional Cargo-app adapter | Build .deb payload/control archives | debian-packaging is MPL-2.0; cargo-deb library interface is documented as experimental |
| px-writer-rpm | rpm crate | Build RPM metadata and payload | Not a complete replacement for rpmbuild or the system RPM database |
| px-writer-appimage | appimagetool / its Rust implementation, uruntime, mkdwarfs; backhand for SquashFS-specific work | Build AppDir into AppImage | Current appimagetool path uses a runtime and DWARFS; backhand is not a drop-in replacement for that pipeline |
| px-writer-flatpak | Official flatpak-builder, flatpak, OSTree tools | Generate a Flatpak manifest and invoke the established build/export pipeline | Run in an isolated Linux build environment; don't reimplement OSTree |
| px-electron | Electron distribution layout, @electron/get, @electron/asar, @electron/rebuild, Electron Fuses | Detect Electron, preserve original app layout, build app-only payload where supported, encode protected runtime reference and package a restricted installer helper | No arbitrary binary surgery, loose public .ref.json sidecar, global end-user runtime store or additional Electron launcher |
| px-pdc | fastcdc, zstd, SHA-256 implementation already chosen by the Engine | Deduplicate/chunk/compress SDK payload inputs | Pin chunker version and codec settings in the payload format |

The names above are planned PackageX components. They are not claims that these crates or adapters already exist in this repository.

---

## 4. Windows implementation route

### 4.1 MSI: use the existing Rust database writer first

**Primary candidate:** [mdsteele/rust-msi](https://github.com/mdsteele/rust-msi) / [msi on docs.rs](https://docs.rs/msi/latest/msi/).

The current msi crate documentation exposes:
- open and open_rw for existing packages;
- Package::open over a generic Read + Seek object, writable when the object also implements Write;
- table creation and table/row inspection;
- select_rows, insert_rows, update_rows and delete_rows;
- embedded stream enumeration, read, write and removal;
- mutable summary information;
- flush() to write buffered changes.

That is enough to build a real PackageX MSI editor instead of inventing our own database format.

**Related foundations:**
- [cfb](https://docs.rs/cfb/latest/cfb/) reads and writes the Compound File Binary / structured-storage container used by MSI. Its documentation explicitly describes stream/storage mutation and resizing.
- [cab](https://docs.rs/cab/latest/cab/) provides Cabinet archive writing/reading and is MIT-licensed.
- [msi](https://docs.rs/msi/latest/msi/) is MIT-licensed.

#### How px-writer-msi should use them

1. Open the source MSI read-only first.
2. Record its SHA-256, signed state, product/package identifiers, table names, primary keys, stream names and selected metadata.
3. Build a typed MsiChangePlan from the Engine's requested operations.
4. Check every referenced table, column, row key and stream before mutation.
5. Copy the original MSI to a request-scoped staging file on the same filesystem as the final output.
6. Open the staged MSI read/write using msi::open_rw or Package::open over a read/write file.
7. Apply only supported row/stream operations.
8. Flush and close the package.
9. Reopen the staged MSI using a separate read-only handle.
10. Validate tables, key relationships, streams, package identity and expected changes.
11. Compare the source and output logical inventories.
12. Re-sign the final output if a signature is required.
13. Atomically publish the staged output and return the report.

Conceptual adapter flow:

~~~rust
// Pseudocode: names and error types are illustrative, not frozen APIs.
fn modify_msi(
    source: &Path,
    destination: &Path,
    plan: MsiChangePlan,
) -> Result<MsiEditReport> {
    let source_identity = inspect_and_hash(source)?;
    validate_plan_against_source(&source_identity, &plan)?;

    let staged = copy_to_sibling_temp(source, destination)?;
    {
        let mut package = msi::open_rw(&staged)?;
        apply_typed_table_changes(&mut package, &plan)?;
        apply_supported_stream_changes(&mut package, &plan)?;
        package.flush()?;
    }

    verify_msi_against_plan(&staged, &source_identity, &plan)?;
    publish_atomically(&staged, destination)?;
    build_edit_report(source, destination, &plan)
}
~~~

This is the default staged transaction. The msi crate's flush() is not a replacement for Windows Installer's MsiDatabaseCommit transaction API. A write error, process crash or disk-full event must leave the original source untouched because the PackageX edit takes place in a separate staging copy.

**Direct in-place mode is not the first implementation target.** It may be added later for controlled scenarios. Even with a compound-file library that supports stream resizing, PackageX must not promise physical-byte preservation of all untouched sectors. The guarantee is limited logical change plus safe replacement.

#### When to use native Windows Installer APIs

For features whose semantics specifically depend on Windows Installer, add a Windows-only backend using the native MSI database API through a narrowly scoped Rust FFI wrapper (windows/windows-sys bindings or a small audited C ABI layer).

Use it for:
- transaction-mode database editing and explicit MsiDatabaseCommit;
- Windows-native validation of table/schema constraints;
- transform generation/application/summary information where the native functions provide the required compatibility;
- behavior that cannot be proven equivalent in the portable msi implementation.

The portable msi backend remains the normal cross-host build path. The native API is a capability enhancement, not a reason to make every PackageX build require Windows.

#### Cabinet integration

For a new MSI:
1. Convert Engine resources to deterministic package-file records.
2. Build the required directory, component, file, feature, media and cabinet relationships.
3. Produce the embedded/external cabinet with cab.
4. Add the cabinet stream and corresponding MSI records through msi.
5. Validate that every File/Media reference resolves and every cabinet member has the expected size/hash.
6. Reopen and inspect the output before signing.

The initial writer should support a clearly enumerated MSI authoring subset. Do not claim arbitrary third-party MSI round-tripping until unknown table/stream preservation is demonstrated by a test corpus.

#### MST transforms

An MST is not an arbitrary patch. It is a Windows Installer transform with rules and validation information.

- Use native Windows Installer transform generation first for the initial supported MST operation.
- Preserve the exact base MSI digest in the transform report.
- Test the transform against that exact base MSI.
- Report unsupported changes to embedded cabinets and summary information; do not claim an MST can carry them.
- Keep direct staged MSI editing available for changes that are not representable as a transform.

The native APIs and transaction details are documented by Microsoft:
- [Windows Installer database functions](https://learn.microsoft.com/windows/win32/msi/database-functions)
- [MsiOpenDatabaseW](https://learn.microsoft.com/windows/win32/api/msiquery/nf-msiquery-msiopendatabasew)
- [MsiViewModify](https://learn.microsoft.com/en-us/windows/win32/api/msiquery/nf-msiquery-msiviewmodify)
- [MsiDatabaseCommit](https://learn.microsoft.com/en-us/windows/win32/api/msiquery/nf-msiquery-msidatabasecommit)

#### MSI test strategy

Use both Rust fixture tests and a Windows-native test runner:
- table CRUD and stream CRUD round-trip tests;
- corrupt table/invalid key/unknown stream negative tests;
- signed input modification and re-sign behavior;
- large cabinets and external media cases;
- crash/failure injection before flush, after flush, before publish and after publish;
- msiexec install/repair/uninstall smoke tests in disposable Windows VMs;
- compare the output with native Windows Installer inspection/validation.

### 4.2 MSIX: use Microsoft's own cross-platform SDK

**Primary candidate:** [Microsoft MSIX Packaging SDK](https://github.com/microsoft/msix-packaging), licensed MIT.

This is especially attractive for PackageX because Microsoft publishes a cross-platform native library:
- DLL on Windows;
- dylib on macOS;
- shared object on Linux and Android.

Its documented API includes package/bundle packing and unpacking support through PackPackage, UnpackPackage and UnpackBundle, plus the makemsix command wrapper.

**Build detail that matters:** the upstream project documents that packing is not enabled by default. The build must explicitly enable MSIX_PACK=on / --pack and USE_VALIDATION_PARSE=on. Without the validation-parse option the pack operation fails. The project builds through CMake and has platform-specific build scripts.

#### Integration design

Preferred:
- Build the MSIX SDK as a separately versioned native SDK subcomponent for each host OS/architecture.
- Put a very small C ABI shim in front of the C++ APIs.
- Wrap that C ABI in a safe Rust px-writer-msix adapter.
- Keep the native library and wrapper isolated from the core PackageX Engine.
- Expose a structured operation request/result, not a generic raw pointer interface to the Engine.

Fallback:
- Build the upstream makemsix executable as an SDK worker.
- Invoke it using a controlled argument list, request-scoped temporary directories and structured output parsing.
- Pin its exact upstream commit/build identity and test its command behavior before relying on it.

The C ABI route avoids per-build process startup and allows the Engine to reuse the same SDK contract. The executable route is a valid first proof-of-concept if FFI integration becomes the critical path.

#### What PackageX still owns

- Translating the Engine's normalized resources and metadata into MSIX package inputs.
- Mapping PackageX capability declarations onto MSIX manifest capabilities.
- Producing deterministic inputs and a package report.
- Deciding whether requested installer behavior is compatible with MSIX's application model.
- Signing the finished package.
- Validating with the actual Windows AppX deployment/packaging tooling.

The MSIX Packaging SDK does not eliminate the need for final signing and native Windows installation tests. Treat packaging, signature creation, signature verification and actual deployment as separate capabilities.

#### MSIX validation

- Reopen the generated package with the MSIX SDK and check package structure.
- Run Microsoft's MakeAppx/MSIX validation tooling on Windows CI when available.
- Sign with Microsoft's SignTool in the Windows signing job.
- Verify the signature and attempt installation/deployment in a clean Windows VM.
- Test block-map correctness after every content mutation.

This route is preferable to implementing OPC ZIP layout and AppxBlockMap generation entirely from scratch.

### 4.3 Authenticode and signing tool selection

- On Windows, use the official Windows SDK SignTool for final publisher signing and verification.
- [osslsigncode](https://github.com/mtrojnar/osslsigncode) can sign/verify PE, CAB, MSI and APPX-family files and is useful as a cross-platform research/optional adapter.
- Its repository is GPL-3.0. Do not silently link it into permissively licensed SDK code or bundle it as an unnoticed mandatory executable. If used, isolate it as an optional external tool and perform a distribution/license review first.
- Sign only after all format mutations and package finalization have completed.

### 4.4 WiX is reference material, not a hidden dependency

WiX is a useful reference for Windows Installer authoring, custom actions and bootstrapper behavior. However, WiX v6 introduced an Open Source Maintenance Fee and v7 enforces EULA acceptance in its distribution flow. PackageX must not quietly require WiX releases for normal commercial builds without an explicit licensing/business decision.

Reference: [WiX Open Source Maintenance Fee](https://docs.firegiant.com/wix/osmf/).

---

## 5. macOS implementation route

The Apple ecosystem is the reference model for how PackageX should reuse mature format libraries and write only the missing adapters.

The main upstream family is [indygreg/apple-platform-rs](https://github.com/indygreg/apple-platform-rs). It contains focused crates rather than one monolithic packaging library:
- apple-codesign;
- apple-bundles;
- apple-flat-package;
- apple-xar;
- apple-bom;
- apple-dmg;
- cpio-archive;
- App Store Connect/notarization support.

### 5.1 .app bundles

Use apple-bundles and upstream plist/asset tooling as the starting point for:
- creating the standard .app directory tree;
- generating Info.plist and bundle metadata;
- placing the main executable and helper apps;
- copying frameworks, resources, icons and localized strings;
- applying bundle identifiers and minimum OS metadata.

PackageX adds its own bundle specification and resource validator. Do not confuse successful directory assembly with a valid signed application bundle.

### 5.2 .pkg component and product packages

A flat macOS package is an Apple-flavored XAR archive. It is not simply a ZIP file with a .pkg extension.

The upstream apple-flat-package documentation describes component package contents such as:
- Bom: bill of materials describing installed files and attributes;
- PackageInfo: component metadata;
- Payload: CPIO archive containing installed files;
- Scripts: CPIO archive containing permitted install scripts.

Product packages can contain multiple component packages and Distribution metadata/resources.

Use:
- apple-flat-package for component/product package structures;
- apple-xar for the outer XAR container;
- cpio-archive for payload/script archives;
- apple-bundles and plist code for application bundle metadata;
- apple-bom for BOM reading and the portions of BOM construction it actually supports.

#### Known BOM gap

The current apple-bom crate documentation says writing support is still a work in progress. This is a real implementation gap, not something to hide in a dependency table.

Preferred sequence:
1. Prototype a component package with the upstream package/XAR/CPIO libraries.
2. Test whether the current BOM builder produces a complete valid file for the supported package subset.
3. If not, isolate BOM generation behind a px-apple-bom module.
4. Reuse upstream BOM data structures and format primitives where possible.
5. Implement only missing writer records/index construction, preserving the upstream license notices for any copied/modified source.
6. Compare generated BOMs and package contents using Apple's pkgutil/lsbom-style inspection tools on a macOS runner.
7. Do not advertise .pkg creation as supported until an actual installer run and receipt/file validation pass on macOS.

A fallback is to invoke Apple's pkgbuild/productbuild on a macOS runner for the operation that the portable writer cannot yet reproduce. This can be an optional build-host capability rather than a dependency for every PackageX user. It does mean the affected build cannot be completed on a Windows/Linux host until the Rust BOM gap is closed.

### 5.3 .dmg

Use the apple-dmg crate as the primary Rust candidate. The SDK adapter prepares a deterministic staging folder and DMG specification (volume name, file layout, background/icon resources and window presentation settings where supported), then asks the crate to emit the disk image.

Don't invent DMG block layout yourself until a measured upstream gap is established.

Validation on a macOS runner:
- hdiutil verify on the generated image;
- attach/mount the image read-only;
- inspect mounted contents and file metadata;
- detach cleanly;
- sign/notarize/staple when the selected distribution policy requires it.

If a presentation feature is not implemented by the Rust crate, report that capability as unsupported or use a pinned Apple-tool fallback on the macOS builder. Do not silently produce a technically valid but broken-looking image.

### 5.4 Apple code signing, notarization and stapling

**Primary candidate:** [apple-codesign](https://docs.rs/apple-codesign/latest/apple_codesign/), from the Apple Rust project.

The crate documents support for:
- parsing Mach-O and fat/universal Mach-O signature structures;
- constructing embedded signatures;
- signing individual Mach-O binaries and bundles;
- signing nested bundles;
- creating CodeResources;
- submitting notarization requests and querying status;
- stapling notarization tickets to supported artifacts including bundles, DMGs and PKGs.

The current crate documentation also identifies important limits:
- it does not provide turnkey access to the macOS Keychain private key; importing a .p12 signing identity is supported;
- some advanced bundles/PKGs may not sign/notarize/staple successfully;
- it does not attempt to implement every Gatekeeper/Apple OS executable verification rule.

Therefore the PackageX signing pipeline is:

1. Complete all bundle/package mutations.
2. Apply selected Electron fuses or other binary mutations before signing.
3. Sign nested code in the required order and then sign the outer bundle.
4. Submit for notarization using an authorized Apple credential/API configuration.
5. Poll the notarization result.
6. Staple the accepted ticket.
7. Run native Apple verification/assessment on a macOS CI runner.
8. Record notarization request identity, ticket state and final artifact hashes.

apple-codesign is an implementation foundation, not proof that Gatekeeper will accept every binary it creates. Native codesign, spctl and installer/disk-image validation on macOS remain mandatory acceptance tests.

### 5.5 Licensing and Apple runners

apple-codesign is MPL-2.0. Treat its source and transitive Apple crates with per-crate/per-file license checks. Keep modified MPL-covered files open under the MPL requirements and retain original notices.

The SDK may build much of the pure-Rust Apple package/signing toolchain on Linux or Windows. However:
- native macOS validation must run on macOS;
- any component requiring Apple's toolchain/SDK must be built on an eligible macOS runner;
- signing/notarization credentials are not bundled in the SDK;
- the macOS runner is a validation/signing service, not a reason to ship Apple's SDK headers or an Apple SDK inside the PackageX SDK.

---

## 6. Linux implementation route

### 6.1 Debian .deb

**Candidates:**
- [debian-packaging](https://docs.rs/debian-packaging/latest/debian_packaging/) from indygreg/linux-packaging-rs;
- [cargo-deb](https://github.com/kornelski/cargo-deb) as a reference and optional path for Cargo-built Rust applications.

debian-packaging aims to provide pure-Rust Debian packaging primitives, deterministic archive/repository behavior and library-level interfaces. Its current package is MPL-2.0, so apply the same file-level license discipline as the Apple crates.

cargo-deb is MIT-licensed and has a broad working CLI, but its own docs describe its library interface as experimental and its core use case is turning Cargo projects into Debian packages. It is not automatically the best library for packaging an arbitrary application resource tree.

#### px-writer-deb flow

1. Build the normalized filesystem staging tree from Engine resource references.
2. Generate Debian control metadata (Package, Version, Architecture, Maintainer, Description, dependencies, conflicts/replaces and other supported fields).
3. Generate maintainer scripts only from explicit, reviewed PackageX installer actions.
4. Add service unit files, desktop entries, icons and documentation as package files with correct owner/mode.
5. Produce deterministic control.tar.* and data.tar.* archives.
6. Wrap those members in the required Debian ar archive.
7. Set package architecture and compression policy.
8. Run dpkg-deb --info, dpkg-deb --contents and a disposable container install/uninstall test.

Do not assume an arbitrary .s.px script maps safely to a Debian postinst script. The Engine capability matrix decides whether a project action can be lowered to the native package-manager lifecycle.

### 6.2 RPM

**Primary candidate:** [rpm-rs/rpm-rs](https://github.com/rpm-rs/rpm-rs) / [rpm on docs.rs](https://docs.rs/rpm/latest/rpm/).

The current Rust library documents a programmatic PackageBuilder, file-entry metadata, scriptlets, compression controls and signature-header-related structures. It is intended to create/read RPM package files without requiring the host's RPM libraries.

Its own README is clear about non-goals:
- it does not install RPM files or manipulate the system RPM database;
- it does not aim to reimplement all of rpmbuild;
- it omits some legacy signature algorithms/format features.

#### px-writer-rpm flow

1. Convert the Engine resource inventory into RPM file entries.
2. Generate package metadata and NEVRA identity.
3. Declare dependencies, conflicts, provides, file ownership and modes.
4. Add scriptlets only when they have a well-defined native RPM lifecycle mapping.
5. Select a supported payload compressor and exact settings.
6. Build the RPM through PackageBuilder.
7. Apply/verify publisher signature through an approved signer.
8. Validate with rpm -qp --info, rpm -qp --list, rpm --checksig and distro-container install/uninstall tests.

Do not advertise full rpmbuild compatibility. PackageX is assembling completed application payloads, not compiling arbitrary RPM spec files.

### 6.3 AppImage

**Primary candidate:** [appimagetool](https://docs.rs/appimagetool/latest/appimagetool/) and its upstream Rust implementation.

The current toolchain path described by the project:
- validates an AppDir;
- resolves an AppImage runtime (uruntime);
- invokes/builds a DWARFS filesystem image through mkdwarfs;
- emits the .AppImage and optionally zsync control metadata.

This is a better direct AppImage path than assuming a SquashFS library can write every current AppImage.

[backhand](https://docs.rs/backhand/latest/backhand/) remains useful for reading, creating and modifying SquashFS filesystems. It is a format-specific building block and a potential compatibility backend for SquashFS-based AppImage variants, but it is not a drop-in substitute for the current DWARFS-based appimagetool flow.

#### px-writer-appimage flow

1. Generate a valid AppDir (AppRun, .desktop file, icon, application files and required runtime dependencies).
2. Validate AppDir layout and ELF architecture.
3. Resolve a pinned compatible AppImage runtime.
4. Resolve a pinned mkdwarfs builder or the approved appimagetool binary/component.
5. Create the DWARFS image and AppImage wrapper.
6. Produce zsync metadata only when enabled.
7. Verify the resulting AppImage and run it in a clean Linux VM/container matching the supported baseline.

Runtime and builder binaries have their own licenses, version pins, checksums and provenance. They are not anonymous files downloaded at build time.

### 6.4 Flatpak

**Use the official Flatpak toolchain rather than writing OSTree.**

The official [flatpak-builder](https://docs.flatpak.org/en/latest/flatpak-builder.html) pipeline:
- consumes a JSON/YAML manifest;
- downloads and verifies source modules;
- builds them in a Flatpak SDK/build environment;
- applies sandbox permissions;
- exports the application to an OSTree repository;
- can sign repository commits with GPG.

flatpak-builder also caches build results and produces a resolved manifest with source revisions, making it a much better base than reimplementing repository objects and sandbox metadata.

#### px-writer-flatpak flow

1. Map Engine project metadata and file resources into a Flatpak manifest.
2. Select a declared Flatpak runtime and matching SDK.
3. Translate the Engine's required permissions into Flatpak finish permissions; never broaden them silently.
4. Generate build modules from pinned local/remote sources and hashes.
5. Execute flatpak-builder inside an isolated Linux build environment.
6. Export to a repository.
7. Sign repository commits with the publisher's GPG identity when publishing a repository.
8. Optionally create a single-file bundle for direct distribution.
9. Validate using the Flatpak CLI and run in a sandboxed test environment.

This is a host-specific worker component. Only Toolchains requesting Flatpak output need it. The Windows or macOS SDK package should not include Linux flatpak-builder.

References:
- [Flatpak build introduction](https://docs.flatpak.org/en/latest/building-introduction.html)
- [Flatpak Builder](https://docs.flatpak.org/en/latest/flatpak-builder.html)
- [Flatpak Builder command reference](https://docs.flatpak.org/en/latest/flatpak-builder-command-reference.html)

---

## 7. Electron: preserve the native app layout and restore exact runtime when supported

Electron is a first-class SDK build capability. The SDK detects the app/runtime boundary, creates the application payload and produces a protected runtime reference for the installer. It does not create an end-user runtime manager, shared runtime store, or additional Electron launcher.

### 7.1 Do not binary-surgery electron.exe

Electron's normal distribution already separates:
- Electron runtime executable and helper files;
- application content in resources/app or resources/app.asar;
- optional app.asar.unpacked files, native modules and companion resources.

Chromium is part of the runtime executable and companion distribution. It is not a stable removable byte range inside an arbitrary application executable. Do not extract Chromium from an opaque EXE or patch signed runtime binaries to manufacture an app-only package.

### 7.2 Detect and prepare the application payload

Use the official Electron packaging layout as the canonical input contract:
- [Electron Application Packaging](https://www.electronjs.org/docs/latest/tutorial/application-distribution)
- [Electron Forge packaging tutorial](https://www.electronjs.org/docs/latest/tutorial/tutorial-packaging)

For ASAR archives:
- [@electron/asar](https://github.com/electron/asar) is the upstream pack/list/extract implementation.
- Preserve the ASAR digest, unpacked inventory and native-module list.
- Unpacked files remain on disk under the matching app.asar.unpacked layout.
- Never hide native .node modules or files that must be opened directly from the filesystem inside ASAR.

The SDK detects standard packaged Electron layouts, source projects with lockfiles, and known packager output. It records confidence and exact version/build identity. Ambiguous or customized binaries are preserved as bundled input; the SDK must not substitute stock Electron unless identity is proven.

If an interactive build detects Electron, the Engine asks whether to use recommended Electron optimization settings. The SDK reports detection and capability; the Engine owns the prompt. Yes enables the default profile, no preserves the full distribution. A non-interactive build must set the choice explicitly and must not hang waiting for input.

### 7.3 Exact runtime reference and download resolution

Electron's upstream [@electron/get](https://github.com/electron/get) is the reference for exact release artifact resolution. It provides useful behavior for:
- exact Electron version;
- OS and architecture;
- artifact naming;
- custom mirror support;
- verified download caching and progress;
- proxy-aware downloads.

The SDK may implement the required native Rust downloader without requiring Node.js. @electron/get remains an upstream behavior/reference integration. @electron/asar, @electron/packager, @electron/rebuild and @electron/fuses remain build-side SDK dependencies or reference tools; the embedded installer helper does not require Node.js.

The runtime reference identifies:
- exact Electron version and build identity;
- target OS and architecture;
- exact version-pinned release artifact URL (never a floating latest URL);
- archive format, expected size and full SHA-256 digest;
- extracted runtime inventory/tree digest;
- original app-relative runtime paths and named payload destination slots;
- ABI/native module requirements, fuse/customization profile and license inventory;
- schema version, application payload identity and installer layout identity.

The SDK may use JSON as its internal build-time source representation. The final installer must not expose a loose readable .ref.json file. It serializes the reference into an opaque authenticated control block alongside the application payload. The format writer maps logical slot IDs to the final payload locations; the helper never accepts arbitrary offsets.

### 7.4 Protected control block and restricted embedded helper

The final installer binds the control block to the authenticated installer layout and publisher authorization. Default opaque protection hides internal paths, URLs and operation metadata from casual inspection, while AEAD authentication detects tampering. This is not password-grade secrecy from the machine owner: a local executable that must decrypt data cannot keep its decryption behavior secret from a determined local reverse engineer.

Integrity and authorization come from the signed installer/publisher identity, authenticated control data, build-time-pinned artifact digest, applicable OS-native signature verification, and strict capability/path restrictions—not encryption alone.

The installer embeds a target-specific restricted helper (the Windows implementation may be named script.exe internally). Its capabilities are limited to reading the authenticated plan, downloading only the exact authorized artifact, checking size/hash/signatures, safely extracting within limits, verifying the extracted inventory and writing only to named logical slots inside the active install transaction. It rejects arbitrary URLs, arbitrary paths/offsets, downloaded scripts, unapproved executable launches and writes outside its transaction staging root. It never rewrites the currently running installer EXE in place.

The helper is not PackageX.exe, a package manager, a global runtime service or a general-purpose script runner. Developer .s.px hooks retain their independent WASM capability model. The developer's optional custom updater executable is a separate trust/privilege boundary.

### 7.5 Runtime restoration is native-format dependent

The SDK does not claim that app-only runtime restoration works in every installer format:

| Target | Planning capability | Restriction |
|---|---|---|
| Windows custom EXE overlay | Candidate for app-only restore | Helper restores into same-volume install staging, verifies, then commits transactionally. |
| Windows MSI | Candidate, requires MSI transaction proof | Helper/custom-action or bootstrap path must respect Windows Installer repair, rollback and install-context semantics. |
| Windows MSIX | No direct install-time insertion | Package files are OS-managed/read-only and signed. Keep Electron bundled unless a separate signed dependency/package design is proven. |
| macOS PKG | Conditional | Do not add files to a signed .app after its resource seal is created. An external-runtime application design needs explicit signing/notarization validation. |
| macOS DMG drag-and-drop | No embedded helper invocation | Ordinary drag-copy does not execute an installer helper. Keep Electron bundled. |
| Linux DEB/RPM | Candidate with native maintainer hooks | Must respect package-manager transaction/privilege semantics and per-distro testing. |
| Linux AppImage | No helper in ordinary direct execution | Keep Electron bundled unless a separate installer/bootstrap workflow is explicitly selected. |
| Linux Flatpak | No mutation of deployed package | Use bundled Electron or a separately designed Flatpak runtime/extension; do not edit immutable deployments. |

If the selected format cannot safely restore the runtime, the target reports unsupported for app-only optimization. A bundled fallback is used only when the developer's policy explicitly permits it.

### 7.6 First install and updates

For an eligible target, the helper authenticates the installer/reference, downloads the exact pinned artifact to staging, verifies expected size and full SHA-256, applies available OS-native signature checks, safely extracts and validates its inventory, then restores runtime files to their original relative paths. The original application executable remains the launch entry. No PackageX Electron launcher is created.

ASAR integrity metadata and fuse settings may alter an Electron executable or signed bundle on a per-application basis. When that happens, the stock upstream archive is not the final runtime identity: use an exact developer-supplied customized runtime artifact with its own pinned digest/signature, or keep the runtime bundled. Never download a stock archive and silently assume it reproduces an app-specific signed binary.

The current installed runtime belongs to the installed application generation; it is not a cache. During updates:
- keep the current application/runtime generation running;
- if the runtime identity is unchanged, do not download Electron again;
- otherwise stage only the exact runtime needed by the newest applicable application release;
- retain at most one pending runtime/generation, discarding a superseded pending release;
- wait for the application to exit, then commit app payload and runtime together;
- health-check the new generation and retain the old generation for rollback until healthy.

There is no end-user shared runtime store, version library, global cache, reference counting or PackageX runtime service. A separate SDK-side verified developer cache may reuse exact release archives across builds.

References:
- [Electron installation and downloads](https://www.electronjs.org/docs/latest/tutorial/installation)
- [Electron Fuses](https://www.electronjs.org/docs/latest/tutorial/fuses)
- [Electron ASAR integrity](https://www.electronjs.org/docs/latest/tutorial/asar-integrity)
- [SDK Electron architecture](../architecture/electron.md)


## 8. Payload optimization: use established chunking and codec implementations

### 8.1 Chunking

**Candidate:** [fastcdc](https://docs.rs/fastcdc/latest/fastcdc/).

The Rust implementation exposes multiple FastCDC variants and documents deterministic chunk boundaries for the same input. It also warns that different algorithm implementations can produce different cut points.

PackageX must therefore pin:
- crate version;
- selected algorithm implementation/module;
- min/average/max chunk settings;
- exact byte-level test vectors.

Do not change chunker implementation silently: doing so changes chunk identities and delta-reuse behavior. A new algorithm identity is a payload-format change or a new supported chunker version.

### 8.2 Compression

**Candidate:** [zstd](https://docs.rs/zstd/latest/zstd/) Rust bindings.

It provides streaming encoder/decoder APIs and dictionary support. The zstdmt feature enables multithreaded compression inside the C library; the build feature is optional.

The first implementation should:
- use streaming reader/writer APIs;
- hash the original bytes before compression;
- compress each unique chunk once;
- store raw chunks when compressed output is not smaller by the configured minimum saving;
- pin codec version and parameters in payload metadata;
- limit concurrency and buffer size;
- verify reconstructed output against original hashes.

Do not hard-code zstd-19 as a universal performance answer. Benchmark several levels on real PackageX installer data and select one default based on total build time, peak RAM and final artifact size. The user-facing CLI does not expose Fast/Medium/High build tiers.

### 8.3 App-specific exclusion rules

Do not assume all files benefit from another compression pass:
- JPEG/PNG/AVIF/WebP images;
- MP4/WebM/audio codecs;
- ZIP/JAR/ASAR or other package archives;
- encrypted/compressed assets;
- Electron runtime archives.

PDC may still deduplicate identical raw files/chunks without recompressing them. For ASAR, inspect its archive/header semantics and preserve the source format.

---

## 9. External tools are SDK components too

Some upstreams are best integrated as pinned workers rather than linked Rust libraries. They still need the same package manifest, version, hash, license, capability and test discipline.

| Worker/component | Used for | Only installed when |
|---|---|---|
| Microsoft MSIX native SDK | MSIX pack/unpack | Toolchain requires MSIX |
| Windows SignTool | Authenticode/MSIX signing and verification | Windows signing or native validation is requested |
| pkgbuild/productbuild fallback | macOS component/product package when portable BOM path is not yet sufficient | macOS host capability is selected and portable writer cannot cover operation |
| hdiutil/pkgutil/codesign/spctl | macOS native verification and fallback | macOS validation/signing stage runs |
| flatpak-builder/flatpak | Flatpak build/export/verification | Flatpak output requested |
| appimagetool/mkdwarfs/AppImage runtime | AppImage generation | AppImage output requested |
| Node + @electron/asar | ASAR creation/extraction | Electron input is a raw application directory or an ASAR transformation is requested |

These helpers are not ambient PATH dependencies. The SDK manifest locates their exact package paths and records versions/hashes. Process invocation uses a fixed executable path, explicit arguments, bounded environment, captured stdout/stderr and exit-code mapping. Never invoke a shell with project-controlled command text.

## 10. Build-host strategy

The SDK package directory must identify the OS/architecture on which its compiled SDK entry/components execute. Installer output target is a separate capability dimension.

Example:
- a Windows x64 SDK package may contain the MSI adapter and cross-host portable PKG/DEB writers;
- a macOS ARM64 SDK package may contain Apple verification helpers and native build components;
- a Linux x64 SDK package may contain Flatpak/AppImage workers.

A package's manifest lists all output targets/formats it can produce from that build host. It is incorrect to infer output support from the folder name alone.

The current requested path pattern sdk/<platform>-<architecture>-<os>/lib.exe needs one final naming decision: lib.exe cannot be assumed to be a valid macOS/Linux executable name. Preserve the desired per-host directory structure, but use a platform-appropriate entry binary/library extension unless lib.exe is explicitly defined as an archive/data filename rather than an executable.

## 11. Upstream license and maintenance review

Do not infer license from project reputation or repository owner. Every package/component needs:
- its own SPDX identifier;
- transitive dependency license scan;
- third-party notices;
- source URL and pinned tag/commit;
- license copies where required;
- note of any runtime/build helper distributed in the SDK.

Known findings from current upstream metadata:
- msi / rust-msi: MIT.
- cab: MIT.
- Microsoft MSIX Packaging SDK: MIT.
- apple-codesign: MPL-2.0.
- debian-packaging: MPL-2.0.
- cargo-deb: MIT; library API marked experimental in its docs.
- rpm: MIT OR Apache-2.0.
- osslsigncode: GPL-3.0; optional/external only after distribution review.
- WiX v6/v7: Open Source Maintenance Fee terms apply to qualifying commercial use.

For apple-flat-package, apple-bom, apple-dmg, apple-xar, cpio-archive, fastcdc, zstd, appimagetool and every transitive crate, verify the exact crate version's license metadata before pinning. Do not assume all crates in a GitHub monorepo share one license or that a new release has the same dependency set.

## 12. Recommended proof-of-concept sequence

1. **MSI table editor:** open a known MSI, change one Property row and one supported table row, add/remove a stream in a staged copy, reopen and compare. Then test on Windows with native tools.
2. **MSI builder:** create a simple application MSI with one cabinet, install/uninstall it in a Windows VM, and verify ARP/receipt behavior.
3. **MSIX adapter:** build MSIX SDK with pack feature enabled on Windows, Linux and macOS host runners; create package, sign on Windows, validate and install on Windows.
4. **macOS PKG:** build a component package with apple-flat-package, prove BOM output completeness, test with native pkg tools and actual installer.
5. **macOS DMG/signing:** create a DMG, sign bundle and image, notarize/staple, validate with native macOS tools.
6. **DEB/RPM:** create packages from one Engine resource inventory and compare metadata/contents using native distro tools and clean containers.
7. **AppImage:** build from AppDir and launch on the minimum supported Linux baseline.
8. **Flatpak:** generate a manifest, build/export in an isolated Linux runner, install into a test user runtime and verify permissions.
9. **Electron:** exercise detection, app-only payload construction, protected reference encoding, restricted-helper restoration on supported formats and immutable app/runtime update generations; verify bundled fallback on formats that cannot safely reconstruct signed or immutable packages.
10. **PDC:** benchmark whole-file dedup, FastCDC and Zstandard independently; then test combinations on real application payloads including Electron.

Each step produces:
- a minimal fixture repository/input;
- exact upstream versions and feature flags;
- build commands and runner image;
- golden expected metadata;
- a native verification log;
- an accepted capability statement;
- a documented unsupported-feature list.

## 13. Source references

Research snapshot checked 21 November 2026:

- [msi crate](https://docs.rs/msi/latest/msi/) and [rust-msi source](https://github.com/mdsteele/rust-msi)
- [cfb](https://docs.rs/cfb/latest/cfb/) and [cab](https://docs.rs/cab/latest/cab/)
- [Microsoft MSIX Packaging SDK](https://github.com/microsoft/msix-packaging) and [MIT license](https://github.com/microsoft/msix-packaging/blob/master/LICENSE)
- [Apple Rust project](https://github.com/indygreg/apple-platform-rs)
- [apple-codesign](https://docs.rs/apple-codesign/latest/apple_codesign/)
- [apple-flat-package](https://docs.rs/apple-flat-package/latest/apple_flat_package/)
- [apple-bom](https://docs.rs/apple-bom/latest/apple_bom/)
- [apple-xar](https://docs.rs/apple-xar/latest/apple_xar/)
- [rpm](https://docs.rs/rpm/latest/rpm/)
- [debian-packaging](https://docs.rs/debian-packaging/latest/debian_packaging/)
- [cargo-deb](https://github.com/kornelski/cargo-deb)
- [backhand](https://docs.rs/backhand/latest/backhand/)
- [appimagetool](https://docs.rs/appimagetool/latest/appimagetool/)
- [Flatpak Builder documentation](https://docs.flatpak.org/en/latest/flatpak-builder.html)
- [Electron Application Packaging](https://www.electronjs.org/docs/latest/tutorial/application-distribution)
- [Electron installation and downloads](https://www.electronjs.org/docs/latest/tutorial/installation)
- [@electron/get](https://github.com/electron/get)
- [@electron/asar](https://github.com/electron/asar)
- [fastcdc](https://docs.rs/fastcdc/latest/fastcdc/)
- [zstd](https://docs.rs/zstd/latest/zstd/)
- [osslsigncode](https://github.com/mtrojnar/osslsigncode)
- [WiX maintenance-fee terms](https://docs.firegiant.com/wix/osmf/)
