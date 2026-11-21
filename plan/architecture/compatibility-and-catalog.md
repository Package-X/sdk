# SDK Catalog, Versioning and Engine Compatibility

## 1. Purpose

The signed SDK catalog is the machine-readable source PackX uses to discover package identities, targets, dependencies, versions, hashes and tested compatibility. It is distinct from SDK binaries and from the project's .px manifest.

The catalog answers:
- which SDK packages exist for a target;
- which Stable versions and Nightly snapshots are available;
- which Engine versions support each package;
- which SDK API and components are required;
- which dependencies, URLs, sizes, hashes, signatures and provenance apply.

## 2. Version layers

| Layer | Example | Meaning |
|---|---|---|
| Engine | 2.3.0 | Project/build semantics |
| SDK package | 2.4.0 | Tested target SDK composition |
| SDK API | 1 | Engine ↔ SDK contract compatibility |
| Component | 1.4.2 | Internal writer/stub/adapter version |
| Nightly identity | nightly.a1b2c3d4 | Visible commit-based package identity |
| Full source commit | full Git SHA | Immutable source provenance |

Developers normally install a target SDK package. The manifest pins its internal component versions.

## 3. Only Stable and Nightly

### Stable
- Semantic-versioned.
- Manually published.
- Production selection by default.
- Immutable after publication.

Example:

~~~text
px-stub-windows-x64@1.4.2
~~~

### Nightly
- Built from an explicitly selected Git commit.
- Visible download/listing identity is the first eight hexadecimal characters of that commit.
- Full commit SHA and internal semantic-version metadata remain in signed metadata.
- Immutable after publication.

Example:

~~~text
px-stub-windows-x64@nightly.a1b2c3d4
~~~

There is no Beta channel or Beta tag format. The eight-character prefix must resolve unambiguously. A collision blocks publication or requires a defined alternate identity rule; it must never select an arbitrary commit.

## 4. Catalog model

Conceptual shape:

~~~json
{
  "schema": "packagex-sdk-catalog",
  "schema_version": 1,
  "catalog_revision": 42,
  "generated_at": "normalized-timestamp",
  "channels": {
    "stable": {},
    "nightly": {}
  },
  "packages": [],
  "components": [],
  "revocations": []
}
~~~

The catalog is small enough for PackX to query before choosing a target package. Detailed manifests/provenance are fetched only for selected entries. The signed payload includes catalog revision and freshness data to resist stale-catalog rollback.

## 5. Compatibility declarations

Each SDK package declares:
- SDK API version/range;
- tested Engine compatibility range;
- OS/architecture/ABI;
- minimum host capabilities;
- required component constraints;
- supported format operations and limitations.

The catalog may publish a friendly compatibility matrix, for example:

~~~text
PackageX Engine 2.3 supports SDK packages 2.3 through 2.6.
~~~

That is a compatibility fact, not an instruction to always install SDK 2.6. Numeric ordering alone is insufficient: API compatibility, target support and tested integration results must be considered.

## 6. Compatibility registry, not telemetry

The website and machine endpoint publish compatibility facts. This is a **Compatibility Registry**, not usage telemetry.

- PackX downloads published compatibility and package metadata.
- The registry does not require project files, installed-file inventories or build history.
- Resolution does not send secrets or project contents.
- Any future aggregate usage metrics must be separate, opt-in and unnecessary for package resolution.

## 7. Resolution precedence

1. Exact package digest/version in a project/toolchain lock.
2. Explicit user/package SDK pin.
3. Engine-declared compatibility plus target and requested features.
4. Stable catalog version satisfying the constraints.
5. Nightly only when explicitly requested by user or environment.

Nightly is never auto-selected just because it is newer. A specific nightly prefix resolves to one full commit or fails without falling back to another nightly or Stable.

## 8. Latest pointers

The catalog may expose a recommended Stable version and latest Nightly identity per target. These are mutable catalog pointers; pointed-to package artifacts remain immutable.

A lock records the resolved immutable version/commit and artifact hashes, not only latest.

## 9. Promotion, withdrawal and revocation

Publishing an SDK artifact and promoting it in a catalog are distinct:
- manual build creates immutable artifacts;
- catalog update makes them discoverable;
- Stable promotion is explicit;
- withdrawn/broken packages remain represented in catalog history;
- compromised package metadata can be revoked with reason/time.

PackX must report revocation and offer a controlled recovery path rather than silently deleting pinned working packages.

## 10. Signature and freshness

The catalog is signed by a trusted PackageX release key. PackX verifies:
- canonical payload signature and key identity;
- revision/freshness policy;
- package manifest digest;
- artifact size and digest;
- selected channel and immutable identity.

HTTPS is transport protection, not the authenticity root. The signed catalog and manifest hashes anchor package integrity.

## 11. Acceptance criteria

- [ ] Compatibility can be queried without downloading target binaries.
- [ ] Registry lookup sends no project contents.
- [ ] Stable and Nightly only; Beta absent from schema and CLI.
- [ ] Nightly visible identity is eight commit characters, full SHA in metadata.
- [ ] Package identities are immutable.
- [ ] Compatibility includes API, target and feature support, not version comparison alone.
- [ ] Invalid/stale catalog is rejected.
- [ ] Explicit pins are never silently replaced by latest.
