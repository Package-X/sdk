# Engine ↔ SDK Build Contract

## 1. Contract objective

The Engine and SDK cooperate without duplicating responsibilities or relying on undocumented calls. The Engine owns project meaning; the SDK owns target-specific mechanics. The contract is the stable boundary between them.

It must work whether the SDK is linked as a Rust library, invoked as a worker executable, or hosted through another transport. The schema and semantics are authoritative; the transport is replaceable.

## 2. Responsibility split

### Engine

- Parse and validate the .px project.
- Resolve variables, target selection, feature policy, install behavior and dependency declarations.
- Collect project resources and create a canonical resource inventory.
- Calculate requested output formats and the build graph.
- Check SDK capabilities and compatibility.
- Provide the BuildRequest and controlled resource access.
- Own global policies: reproducibility, signing requirements, offline constraints and orchestration.
- Receive SDK outputs and accept/reject them against project-level requirements.

The Engine must not reimplement an SDK writer to avoid calling it.

### SDK

- Announce package identity, API version and capabilities.
- Validate target-specific requirements and format constraints.
- Read only resources referenced by the request.
- Prepare target-specific resource representations.
- Apply deduplication/compression where appropriate.
- Create or modify artifacts through format-aware writers.
- Run format-aware verification.
- Return artifacts, hashes, sizes, warnings/errors, metadata and statistics.
- Report unsupported behavior rather than silently dropping features.

The SDK must not act as a second project parser or override Engine policy.

## 3. Lifecycle

1. Discover SDK package and component versions.
2. Query SDK identity, API compatibility and capabilities.
3. Resolve Engine build plan and target selection.
4. Prepare BuildRequest and resource inventory.
5. Validate request and resources.
6. Prepare format-specific resources and writer state.
7. Execute writer operations.
8. Finalize tables, indexes, manifests, metadata and other format structures.
9. Verify resulting artifacts.
10. Return BuildResult.
11. Engine validates the result and completes orchestration.

The capability query must be possible without starting a full build.

## 4. Conceptual request schema

~~~json
{
  "contract": "packagex-sdk-build",
  "contract_version": 1,
  "request_id": "stable-request-id",
  "engine": { "version": "2.3.0", "build_id": "engine-build-id" },
  "sdk": { "package_id": "px-sdk-windows-x64", "package_version": "2.4.0" },
  "target": { "os": "windows", "architecture": "x86_64", "format": "msi" },
  "project": { "app_id": "com.example.product", "version": "1.0.0", "manifest_sha256": "full-sha256" },
  "policy": { "reproducible": true, "offline": false, "verify_after_write": true },
  "resources": [
    { "id": "resource-001", "logical_path": "bin/product.exe", "size": 123456, "sha256": "full-sha256", "kind": "application-file" }
  ],
  "operation": { "kind": "create", "format_options": {} },
  "output": { "artifact_name": "product.msi", "staging_id": "request-output-id" }
}
~~~

This is a conceptual schema, not a frozen Rust ABI or serialized specification.

### Request field requirements

- **contract / contract_version:** identify the public request/result schema.
- **request_id:** correlate logs, cancellation, diagnostics and output reports; not an authorization token.
- **engine:** caller identity for compatibility and diagnostics.
- **sdk:** expected package identity; the loaded SDK must prove it matches.
- **target:** OS, architecture, output format and ABI-relevant target facts.
- **project:** stable application identity, project version and manifest digest; no secrets.
- **policy:** explicit reproducibility, offline and verification requirements.
- **resources:** stable resource IDs, normalized logical paths, sizes, hashes and roles.
- **operation:** create, inspect, modify, transform, verify or another declared capability.
- **output:** Engine-owned staging handle or scoped output destination, not unrestricted SDK filesystem access.

Absolute host paths must not enter reproducible metadata. The Engine validates logical paths before submitting them.

## 5. Resource-provider contract

The Engine should not copy every project file into a huge serialized request. Use a resource provider that supports:

- Open by stable resource ID.
- Read/stream content with bounded buffers.
- Retrieve declared size and digest.
- Verify final digest at end-of-stream.
- Enforce read-only access unless a separate SDK output channel is used.
- Cancel and close cleanly.

The provider must not resolve arbitrary paths supplied by an untrusted manifest. IDs resolve only through the Engine-controlled provider or a narrowly scoped staging root.

Resource types may include ordinary files/directories, generated manifests, compiled .pxs UI, .s.px WebAssembly, Electron application payloads, protected Electron runtime-reference control blocks and runtime archive resources, MSI streams/cabinets and target-specific resource objects. The reference's final encoded form and slot mapping are created by the SDK writer; the Engine receives only declared artifact/capability metadata.

## 6. Conceptual result schema

~~~json
{
  "contract": "packagex-sdk-build",
  "contract_version": 1,
  "request_id": "stable-request-id",
  "status": "success",
  "sdk": { "package_id": "px-sdk-windows-x64", "package_version": "2.4.0", "component_versions": { "px-writer-msi": "1.4.2" } },
  "artifacts": [
    { "artifact_id": "installer-main", "logical_name": "product.msi", "format": "msi", "size": 987654, "sha256": "full-sha256", "role": "primary" }
  ],
  "diagnostics": [],
  "verification": { "performed": true, "status": "passed", "checks": ["database-open", "table-consistency", "artifact-hash"] },
  "statistics": { "input_bytes": 123456, "output_bytes": 987654, "deduplicated_bytes": 0, "compressed_bytes": 0, "elapsed_ms": 0 }
}
~~~

### Result statuses

- **success:** every required output was produced and required verification passed.
- **success_with_warnings:** output is usable but non-fatal conditions need to be shown.
- **unsupported:** requested capability/combination is not implemented.
- **invalid_request:** malformed request, target, operation or resource declaration.
- **failed:** write, finalize or verification operation failed.
- **cancelled:** operation cancelled and uncommitted output was cleaned or marked recoverable.

A successful return from a writer function is not itself a successful build. Required post-write checks must finish.

## 7. Diagnostics

Each diagnostic includes a stable code, severity, stage, human-readable summary, optional structured details, related target/resource/operation/artifact ID and recovery guidance where practical. Do not expose signing keys, passwords, raw credentials or sensitive host paths.

The Engine consumes structured diagnostics. Human terminal output is not the machine contract.

## 8. Capability handshake

The SDK must report:

- package ID, package version, full source commit and API compatibility;
- supported OS/architecture combinations and output formats;
- supported operations per format (create, inspect, modify, transform, verify, sign);
- native installation-scope capabilities for formats that expose install semantics (per-user, machine-wide, user registration, machine provisioning, and required privilege);
- whether the target format can execute the embedded restricted installer helper;
- whether Electron runtime separation/restoration is supported for that format and under which signing/layout constraints;
- required/optional components and runtime dependencies;
- minimum Engine version and tested compatibility range;
- host restrictions and target-specific caveats;
- experimental capabilities and known limitations.

A format listing is not proof that all operations work. A package may create an MSI while not yet supporting arbitrary edits to third-party MSI files.

## 9. Error, cancellation and idempotency

- Public errors are typed and stable; internal panic text is not the API.
- Worker crashes are contained where the selected transport permits.
- Cancellation stops future writes, closes handles and removes temporary data.
- A failed modification never replaces the source artifact.
- Each request uses an isolated staging identity. Partial output cannot be mistaken for a completed artifact.
- The result states whether no output, staged output, committed output or recovery-required output exists.

## 10. Determinism and performance

For reproducible requests: sort deterministic collections, normalize archive paths/metadata, pin SDK and component versions, pin compression/chunker parameters, freeze timestamps where target semantics permit, and avoid process-global mutable state.

Performance requirements: stream large files, use bounded buffers and worker concurrency, reuse verified immutable resources, avoid avoidable copies, and isolate concurrent requests. Do not add user-facing speed tiers.

## 11. Security boundary

Validate paths, resource lengths, metadata sizes, target values and download references. Do not execute project scripts as a side effect of inspection. External tools must be pinned and invoked through controlled adapters. Signing credentials arrive through an authorized key handle/provider, never in serialized BuildRequest JSON.

## 12. Acceptance criteria

- [ ] Capability handshake is available before a build.
- [ ] One request can reference many resources without serializing all bytes inline.
- [ ] BuildResult returns artifacts, full hashes, diagnostics, statistics and verification status.
- [ ] Unsupported operations fail before mutation where possible.
- [ ] Cancellation and failed writes preserve original artifacts.
- [ ] Concurrent requests do not share mutable request state.
- [ ] Reproducible requests produce identical output on declared deterministic targets.
- [ ] The transport can change without changing contract semantics.
