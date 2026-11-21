# Manually Dispatched SDK Release Workflow

## 1. Release control

SDK publishing is maintainer-controlled. The primary release workflow is manually dispatched through GitHub Actions. A maintainer selects source ref, channel, target matrix and publication intent. Pushes and pull requests may validate changes but must never automatically publish SDK releases or move catalog pointers.

Only Stable and Nightly exist. There is no Beta.

## 2. Manual workflow inputs

| Input | Purpose |
|---|---|
| source_ref | Branch, tag or exact commit selected by maintainer |
| channel | stable or nightly |
| stable_version | Required semantic version for Stable |
| target_matrix | Selected platform/architecture/OS packages |
| package_selection | SDK packages/components to compile |
| publish | Whether immutable GitHub Release assets should be created |
| update_catalog | Whether a new signed catalog revision should be published |
| notes | Maintainer-authored release notes |

Validation:
- Stable requires a valid semantic version.
- Nightly identity is derived from checked-out commit.
- Full source commit is recorded.
- A manually typed Nightly identity cannot disagree with the actual source commit.
- Invalid channel/version/target combinations fail before compilation.
- Enabling publish does not implicitly promote a package to Stable catalog recommendations.

## 3. Output layout

Each selected target stages:

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
    ├── SBOM.spdx.json
    ├── THIRD-PARTY-NOTICES.txt
    └── provenance/
~~~

The requested primary output path is sdk/<platform>-<architecture>-<os>/lib.exe. The SDK package can contain many required auxiliary pieces. The precise lib.exe role/platform naming is an open design item; do not emit misleading executable extensions for non-Windows artifacts.

The target directory is staging output. Generated binaries should normally be uploaded as Actions artifacts and attached to immutable GitHub Releases, not committed to source.

## 4. Workflow stages

### Stage 1 — Validate identity
- Checkout selected ref and resolve full commit.
- Validate Stable semantic version or derive Nightly prefix from the checked-out commit.
- Reject prefix collision or an already-published identity with different bytes.
- Record workflow run ID and actor.
- Validate targets against supported matrix.

### Stage 2 — Resolve build matrix
- Determine target Rust triples and required runner types.
- Pin Rust toolchain, Cargo.lock and native dependencies.
- Calculate required components for each target.
- Avoid compiling unrelated platform components.
- Record host/target restrictions and untested cross-compiles.

### Stage 3 — Compile SDK parts
- Compile common components once when host/target allows.
- Compile target writers, stubs, format helpers and data.
- Produce primary entry artifact plus every auxiliary part required by manifest.
- Apply consistent strip/debug-symbol policy.
- Keep credentials, temp paths and developer state out of output.

### Stage 4 — Test and verify
- Component unit/integration tests.
- Format writer conformance tests.
- SDK load/capability handshake.
- Engine compatibility smoke tests across declared supported ranges or a documented representative matrix.
- Reproducibility tests where promised.
- Package layout, hash and size checks.
- Offline package installation test.
- Target runtime tests; compilation alone is not runtime verification.

### Stage 5 — Assemble target package
- Populate target staging directory.
- Generate package manifest and exact component pins.
- Generate full checksums, SBOM, third-party notices and provenance.
- Validate dependency closure and every manifest path.
- Create canonical archives.
- Produce build/verification report.

### Stage 6 — Sign metadata
- Sign manifests/catalog payload using the designated release key.
- Use protected Actions environment/approved signing provider.
- Keep signing secrets out of source and logs.
- Sign after artifact bytes/hashes are final.
- Any post-sign artifact change invalidates package metadata signature.

### Stage 7 — Publish or retain build-only artifacts

If publish is false:
- upload Actions workflow artifacts;
- do not create or update release tags;
- do not change catalog.

If publish is true:
- create immutable channel-specific GitHub Release identity;
- attach target archives, manifest, checksums, SBOM, notices, provenance and verification report;
- reject replacement of existing identity with different bytes.

### Stage 8 — Optional catalog update
- Add immutable package record.
- Validate compatibility and dependency closure.
- Generate catalog revision and sign it.
- Publish catalog metadata.
- Preserve history and revocations.

## 5. Release identities

Stable component example:

~~~text
px-stub-windows-x64@1.4.2
~~~

Illustrative stable package identity:

~~~text
px-sdk-windows-x64@2.4.0
~~~

Nightly examples:

~~~text
px-stub-windows-x64@nightly.a1b2c3d4
px-sdk-windows-x64@nightly.a1b2c3d4
~~~

Nightly metadata keeps full source commit, internal semantic version, repository/ref, workflow run, build time, target, package manifest digest and artifact hashes. Visible identity remains the eight-character commit prefix.

A prefix collision blocks publication. Never overwrite a published identity with different bytes.

## 6. Suggested workflow files

Proposed, not implemented:
- .github/workflows/sdk-build.yml
- .github/workflows/sdk-release.yml
- .github/workflows/sdk-catalog.yml

All release/promotion entry points are manual. PR CI may build/test but cannot publish tags, releases or catalog updates. Permissions are least-privilege and release/catalog publishing use a protected environment.

## 7. Reporting

Report source commit, release identity, targets, compiled components, tests, skipped host/runtime tests and reasons, artifact paths/sizes/hashes, SBOM/license status, compatibility, publish state and catalog revision.

Distinguish **compiled**, **tested**, **packaged**, **signed**, **released** and **catalog-promoted**. An uploaded workflow artifact is not a release.

## 8. Failure and recovery

- A failed target cannot publish a partial package.
- Overall release succeeds only when all selected required targets pass.
- Network retries are bounded; digest/signature failures are hard failures.
- Catalog failure does not invalidate already-published immutable assets; report them as published but not catalog-discoverable.
- Reruns compare existing identity bytes and never overwrite different content.
- Cleanup removes temporary staging only.

## 9. Acceptance criteria

- [ ] Workflow is manually dispatched.
- [ ] Stable requires semver; Nightly derives identity from selected commit.
- [ ] No Beta inputs, tags or entries.
- [ ] Each selected target emits directory and all required SDK pieces.
- [ ] Published packages contain manifest, full hashes, SBOM, licenses and provenance.
- [ ] Identity collision/byte replacement is rejected.
- [ ] Build-only runs never publish or change catalogs.
- [ ] Catalog promotion is explicit.
- [ ] Verification failure prevents publication.
- [ ] Reports distinguish compilation, testing, packaging, signing, release and promotion.
