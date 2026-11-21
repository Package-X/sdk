# Format Writers, Transactional Editing and MSI Database Operations

## 1. Purpose and guarantees

PackageX must be able to create new installers and safely modify supported existing installers. A custom writer cycle is required because the Engine needs more than a one-shot file exporter: it must inspect an artifact, plan changes, apply supported logical edits, validate the result and commit it safely.

The guarantee is **format-aware logical editing**, not arbitrary byte patching. A common lifecycle coordinates writers, but each format adapter owns its own data model and mutation rules.

The original input is immutable by default. A failed edit must never leave the source half-written or silently replace it with an invalid artifact.

## 2. Writer capability contract

Every format writer reports capabilities independently:

| Capability | Meaning |
|---|---|
| create | Generate a new artifact from a validated Engine build plan |
| inspect | Read structure, metadata and resource inventory |
| modify | Apply supported logical changes to an existing artifact |
| transform | Produce a format-native change/transform artifact |
| verify | Reopen and validate structure and integrity |
| sign | Sign through an authorized signing provider, where applicable |
| diff | Compare artifacts and describe logical changes |

A writer may support create but not modify. The Engine must query the capability handshake before selecting an operation. Capability declarations also identify host OS constraints, format revisions, unsupported tables/streams, signature state, and scratch-copy requirements.

## 3. Common write cycle

~~~text
CLOSED
  -> OPENED / INSPECTED
  -> CHANGE_PLAN_CREATED
  -> VALIDATED
  -> STAGED
  -> APPLYING
  -> FINALIZING
  -> VERIFYING
  -> COMMIT_READY
  -> COMMITTED

Any pre-commit error -> ABORTED
Commit uncertainty -> RECOVERY_REQUIRED
~~~

1. **Open:** establish format identity, source size, hash and signature state.
2. **Inspect:** read only the metadata and logical structures required by the requested operation.
3. **Plan:** translate Engine intent into typed format-level changes.
4. **Validate:** check schema constraints, object references, preconditions and writer capability.
5. **Stage:** create isolated working state or open a native transaction.
6. **Apply:** perform the planned logical mutations.
7. **Finalize:** rebuild required tables, indexes, offsets, checksums, cabinet/archive structures or metadata.
8. **Verify:** reopen the staged result with an independent reader where possible.
9. **Commit:** publish the verified staged output or commit the native transaction.
10. **Report:** return changed objects, before/after hashes, verification results and diagnostics.

The writer must expose cancellation points between safe stages. Once a native commit begins, cancellation semantics must be documented; the result may need to report recovery-required rather than pretend nothing happened.

## 4. Typed change plans

The Engine requests logical edits, not raw byte offsets.

Examples:
- Set an MSI Property value.
- Insert or update a supported MSI table row.
- Add a file/component relationship to an installer.
- Replace a stream only when the writer declares stream mutation support.
- Update an MSIX manifest resource and regenerate its block map.
- Change package metadata.
- Replace an entry in a PackageX-native payload index.
- Generate a native transform from a base and modified reference artifact.

Each change plan records:
- source artifact SHA-256 and format identity;
- format revision and writer version;
- target objects and expected existing values;
- intended changes and referenced resource hashes;
- required capabilities;
- preconditions and postconditions;
- signature and package-identity consequences.

Ambiguous object identity, invalid primary keys, broken relationships, changed source identity or unsupported mutations must fail closed.

## 5. Transaction and recovery design

### 5.1 Staged-output default

When a format has no trustworthy persistent transaction API:

1. Open source read-only and capture source identity.
2. Create a private staging copy or native temporary representation.
3. Apply changes only to staged state.
4. Finalize and verify.
5. Flush staged output and its containing directory where supported.
6. Re-check source identity to detect concurrent modification.
7. Atomically replace the destination where the filesystem permits it.
8. Retain or remove the previous artifact according to explicit backup policy.

The original remains unchanged on failure. Temporary data is cleaned or recorded for recovery.

### 5.2 Native format transactions

A format may provide a native transaction model. The writer can use it, but must still manage source/destination safety, handle lifetime, commit failure and post-commit verification.

A logical database transaction does **not** guarantee that only changed physical bytes are rewritten. Structured storage can reorganize sectors, streams, compression blocks or indexes during persistence. PackageX promises safe logical mutation and preservation of unrelated logical structures to the extent supported by the backend, not byte-for-byte preservation of untouched sectors.

### 5.3 Concurrency and idempotency

- Lock the output artifact or staging identity during mutation.
- Detect source changes between inspect and commit.
- Never share mutable database handles across independent build requests.
- Do not mutate one staging output concurrently unless the format guarantees isolated transactions.
- Every build request has an isolated staging identity.
- Repeating an operation must not accidentally apply a non-idempotent edit twice.

## 6. MSI writer architecture

MSI is a structured Windows Installer database, not an opaque binary blob. The SDK writer understands relational tables, records, embedded streams, cabinet data and summary information.

The writer exposes separate capabilities for database inspection, schema inspection, supported row/property edits, stream/cabinet operations, transform generation, verification and re-sign requirements.

### 6.1 Windows Installer API backend

On Windows, the native Windows Installer API provides a transaction-aware database route:

- Open read/write transactionally with MSIDBOPEN_TRANSACT.
- Create a view for the intended table operation.
- Fetch and modify records through documented MsiViewModify modes.
- Validate changes and capture MsiViewGetError details where available.
- Commit with MsiDatabaseCommit and capture extended error information.
- Close handles deterministically.
- On a pre-commit failure, discard staged transactional state.

Important constraints:
- MsiOpenDatabase cannot operate on a database currently in use by a running installation.
- MsiDatabaseCommit persists changes to the writable database.
- Binary data uses documented stream APIs, not ordinary record columns.
- Handle ownership and cleanup are explicit.

References:
- [MsiOpenDatabaseW](https://learn.microsoft.com/windows/win32/api/msiquery/nf-msiquery-msiopendatabasew)
- [MsiViewModify](https://learn.microsoft.com/windows/win32/api/msiquery/nf-msiquery-msiviewmodify)
- [MsiDatabaseCommit](https://learn.microsoft.com/windows/win32/api/msiquery/nf-msiquery-msidatabasecommit)
- [Windows Installer database functions](https://learn.microsoft.com/windows/win32/msi/database-functions)

### 6.2 Cross-platform MSI build and edit support

PackageX must be able to create Windows packages from supported Windows, macOS and Linux build hosts. It cannot assume Windows Installer APIs exist on every host.

Separate:
- portable MSI authoring/inspection for cross-host builds;
- Windows-native edit/validation support when native API behavior is required or provides stronger conformance.

A library that can create an MSI from scratch is not automatically safe for modifying arbitrary third-party MSIs. The portable backend must pass the same conformance suite before declaring modify support.

The initial capability matrix must test:
- create a database;
- inspect known and unknown tables safely;
- update, insert and delete supported rows;
- import/export streams;
- replace embedded cabinets;
- edit summary information;
- generate and apply transforms;
- preserve package identity and relational constraints.

Unsupported operations return unsupported. They must never silently rebuild the MSI while discarding unknown tables, streams or data.

### 6.3 Direct edit versus output copy

Default behavior preserves the source and writes to a new output path. In-place mode is opt-in and requires a recoverable backup, effective lock, staged/transactional mutation, successful verification and explicit signature consequences.

The Rust `cfb` implementation supports stream resizing and in-place logical mutation, but a compound file does not physically shrink in place when streams are removed or truncated. Its `CompoundFile::copy_to` API creates a compacted copy in a new destination. This is another reason PackageX defaults to a staged output: the writer can apply logical changes, compact when appropriate, reopen and verify the result, and only then publish it. Do not advertise byte-range preservation.

Reference: [cfb crate](https://docs.rs/cfb/latest/cfb/), including `CompoundFile::copy_to`.

## 7. MST transform output

An MST transform records supported changes to persistent Windows Installer database tables and is applied by Windows Installer during installation. It is useful when the publisher wants to keep the base MSI intact.

A transform cannot modify data outside the persistent database tables, including the Summary Information stream, substorages or files stored in embedded cabinets. Transform summary information must carry appropriate validation/error conditions, and the MST must be tested against the exact base MSI it targets.

References:
- [About Transforms](https://learn.microsoft.com/windows/win32/msi/about-transforms)
- [Generating a Transform](https://learn.microsoft.com/windows/win32/msi/generating-a-transform)
- [CreateTransformSummaryInfo](https://learn.microsoft.com/windows/win32/msi/database-createtransformsummaryinfo)

## 8. Signatures and package identity

Any content mutation may invalidate signatures or notarization.

- Inspect signed state before mutation and report consequences.
- Never claim the original signature remains valid unless verification proves it.
- Re-sign modified MSI output after all database and cabinet edits when signing is required.
- Signing is a final stage after writer finalization.
- Signing credentials are accessed through the authorized signing provider, never placed in BuildRequest JSON.
- ProductCode, UpgradeCode, PackageCode and similar identity fields change only under explicit Engine policy.

## 9. Other format writers

- **MSIX:** package manifest, block map, content types, archive layout and signing.
- **PE-overlay EXE:** Authenticode-safe overlay and certificate-table/footer invariants.
- **DMG:** image content, volume metadata, signing/notarization.
- **PKG:** package components, scripts, receipts and BOM metadata.
- **DEB/RPM:** archive members and package control/spec metadata.
- **AppImage/Flatpak:** runtime metadata, content structure and target-specific validation.

No generic patch-by-offset API is exposed to the Engine.

## 10. Acceptance tests

- [ ] Failed edits never replace the original artifact.
- [ ] Change plans contain source identity and preconditions.
- [ ] Supported MSI row edits can be committed and reopened.
- [ ] MSI row operations enforce schema and primary-key rules.
- [ ] MST applies only to its matching base package.
- [ ] Cabinet and summary-stream changes are never misrepresented as MST operations.
- [ ] MSI mutation creates a re-sign requirement and visible diagnostic.
- [ ] Concurrent source modification is detected before commit.
- [ ] Every format declares its create/inspect/modify/transform/verify capabilities independently.
- [ ] Verification checks format structure, not just output existence and hash.
