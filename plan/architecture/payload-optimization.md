# Payload Deduplication and Compression (PDC)

## 1. Goal

PDC reduces installer size and repeated storage without changing the logical files the application installs. The Engine provides the canonical uncompressed resource inventory and policy; SDK components identify duplicate content, chunk eligible resources, compress unique data and reconstruct it during installation.

PDC is a payload service, not a replacement for native installer formats. It must never rewrite application semantics or compress data twice when doing so increases size.

## 2. Processing pipeline

1. Receive Engine's normalized resource inventory.
2. Verify source file sizes and full cryptographic digests.
3. Detect identical whole files.
4. For eligible large files, calculate content-defined chunks.
5. Assign each unique uncompressed chunk a full digest identity.
6. Compress each unique chunk when it produces a measured benefit.
7. Write unique chunk data and per-file reconstruction references.
8. Emit format-specific payload representation.
9. Reconstruct/verify logical file hashes.
10. Return savings and overhead statistics to Engine.

## 3. Whole-file deduplication

Implement whole-file deduplication first. If multiple logical paths contain identical bytes, store the content once and reference it from each path.

Path, permissions, timestamps, install role and other per-file metadata remain separate logical entries. Do not deduplicate files that the target format or application semantics require to become independent mutable copies unless install-time materialization creates those copies.

## 4. Content-defined chunking

Chunk-level deduplication follows the whole-file layer. Fixed-size blocks are simple, but a small insertion near the start can shift every later boundary and destroy reuse. Content-defined chunking derives boundaries from content so unchanged regions are more likely to retain chunk identities after local edits.

Candidate starting parameters, subject to benchmark:
- minimum: 64 KiB;
- target average: 256 KiB;
- maximum: 1 MiB.

These are tuning candidates, not a frozen on-disk format. The algorithm and parameters are versioned in build metadata.

## 5. Identity and integrity

Each uncompressed chunk receives a full cryptographic digest, initially SHA-256. A shortened prefix may be used in a filename only if the full digest is checked and collisions are handled.

- Verify digest and size before reusing a cached chunk.
- Reject corrupt/conflicting content identities.
- Verify complete reconstructed file digest.
- Never rely on a short ID as the integrity check.
- Keep source identity distinct from compressed representation identity.

## 6. Compression policy

Zstandard is the planned general-purpose codec for PackageX-native payloads. Existing PackageX plans mention zstd-19; default level and parameters must be selected by benchmarks and target constraints.

- Compress unique chunks, not duplicate references.
- Store raw bytes when compression is not beneficial.
- Skip ineffective recompression of JPEG/PNG, MP4, ZIP, encrypted data and already-compressed package formats.
- Record codec, version, parameters, dictionary identity, compressed size and uncompressed size.
- Any dictionary must be explicitly packaged, hashed and versioned.
- Dictionary training remains experimental until it demonstrates useful cross-project results.

There are no user-facing Fast/Medium/High speed tiers. SDK chooses efficient bounded defaults.

## 7. Native format boundaries

- PackageX custom PE-overlay payloads can use a deterministic chunk index and payload stream.
- MSI/MSIX/PKG/DEB/RPM writers remain responsible for valid native package structures and native compression/member rules.
- PDC may deduplicate input resources before a native writer only where format semantics allow it.
- Electron app.asar, app.asar.unpacked and runtime archives remain separate resource classes. Do not recompress or restructure them in a way Electron cannot load.
- Runtime binaries are normally managed as exact verified runtime packages, not embedded into developer SDK tools.

## 8. Canonical inventory and reproducibility

The Engine's canonical uncompressed resource inventory remains the source of truth. PDC is derived output.

Reproducible mode pins:
- normalized/sorted logical paths;
- file metadata policy;
- chunker algorithm/version and parameters;
- codec version/settings;
- deterministic chunk ordering;
- dictionary digest;
- input source digests.

Deltas should be calculated against canonical uncompressed files/chunk identities rather than unstable compressed container offsets. Keep the source inventory recoverable; never make a prior compressed archive the only representation.

## 9. Build report

Return:
- total original logical payload bytes;
- input file count;
- duplicate whole-file bytes removed;
- duplicate chunk bytes removed;
- unique uncompressed bytes;
- compressed data bytes;
- skipped pre-compressed bytes;
- index/metadata overhead;
- final payload size;
- compression ratio and elapsed time.

Illustrative report:

~~~text
Payload input:             120.0 MiB
Whole-file duplicates:       8.0 MiB
Chunk duplicates:            6.0 MiB
Unique input data:          106.0 MiB
Compressed data:             61.0 MiB
Skipped pre-compressed:      12.0 MiB
Index and metadata:           1.2 MiB
Final payload:               62.2 MiB
~~~

Values are examples only. Report measured results, never present estimates as achieved savings.

## 10. Resource and CPU efficiency

Use bounded streaming buffers, bounded worker concurrency, immutable verified chunk reuse and minimal copies. Avoid holding all source files and compressed output in memory. A deterministic single-worker diagnostic mode may exist without changing semantic output. Low-memory hosts must not be forced to allocate payload-sized buffers.

## 11. Acceptance criteria

- [ ] Identical files are stored once where target semantics permit.
- [ ] Local edits preserve useful chunk reuse.
- [ ] Reconstructed source file hashes match.
- [ ] Compression is skipped when it expands content.
- [ ] Chunk order/metadata are deterministic.
- [ ] Native writers retain valid package semantics.
- [ ] Reports separate deduplication from compression savings.
- [ ] Multi-gigabyte payloads use bounded memory.
