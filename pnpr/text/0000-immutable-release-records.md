# Immutable release records for pnpr

> Status: draft for discussion. No implementation or protocol version is approved.

## Summary

Make an immutable release record the authoritative representation of each
accepted package artifact and its install-critical metadata. Store authoritative
metadata in a transactional SQL database, with archives and large evidence
bundles in the filesystem or object storage. Generate npm package documents
from these records and separately stored mutable state.
Bind the package's declared origin, archive integrity, and verified manifest
together, while preserving pnpr's existing registry revision and tar-integrity
contracts. Tags, selected revisions, deprecations, screening decisions, and
access rules remain independently updateable.

This is a storage and verification foundation. Companion drafts discuss
resolution snapshots with artifact retention and atomic workspace publication.
Neither requires a new package namespace or replacing the public npm registry.

## Motivation

An npm package document combines version manifests with tags and other mutable
package state. The abbreviated representation reduces its size but still
contains a map of versions. A tag change and a new release therefore invalidate
the same representation, although an accepted release's contents need not change.

The npm publish format also carries version metadata separately from its
archive. Implementations must reconcile what the resolver reads with what the
installer extracts. A tarball checksum alone does not establish that separately
served dependency metadata describes that tarball.

For example, a publisher uploads `@example/widgets@1.2.0`. The submitted
manifest declares one dependency range, while the archive's `package.json`
declares another. pnpr should reject the disagreement before publication and
serve one accepted interpretation thereafter.

pnpr already has important parts of this model:

- Declared registry namespaces assign a name to one origin without fallback.
- Hosted publication checks artifact integrities and prevents overwrites.
- The registry revision RFC binds each revision to an immutable tar and keeps
  revision-specific resolution metadata.

This proposal extends those contracts to ordinary hosted releases and their
upstream observations. It does not introduce a competing revision mechanism.

## Detailed Explanation

### Identity and immutable contents

A release record contains these logical fields. Their wire spelling and
canonical encoding must be settled before ratification.

| Field | Contract |
| --- | --- |
| Schema | Identifies the record format and manifest interpretation rules. |
| Authority | Identifies the configured origin and its namespace. |
| Package | Ecosystem, validated name, version, and revision when applicable. |
| Manifest | Install-critical metadata derived from the accepted archive. |
| Artifacts | Complete algorithm, digest, size, and byte representation of each resource. |
| Evidence | References to verified source signatures, provenance, and derivations. |

The record has a digest over a deterministic, versioned encoding. A signature
over that digest is an endorsement by a configured signer. Neither knowing the
digest nor presenting a self-signed record grants trust or access.

An artifact digest identifies bytes. A release record additionally binds those
bytes to a package and authority. Identical bytes from separate private
registries do not establish shared ownership, authorization, or publisher identity.

The authority identifier must distinguish origins that reuse the same package
name, including registries behind different URL paths on one host. Changing a
proxy hostname need not change the upstream authority. Rebinding an authority
to a different source requires an explicit operator action and an audit event;
it cannot follow a redirect or a cache miss.

### Manifest acceptance

For a hosted npm publication, pnpr:

1. Verifies submitted archive integrity and validates the archive under bounded
   decoding and entry-count limits.
2. Reads the manifest that the supported installer would select. Rejects
   duplicate or ambiguous manifest entries and unsafe archive paths.
3. Applies a versioned normalization rule shared with the pnpm manifest reader.
4. Compares submitted and archive-derived install-critical fields. A mismatch
   rejects the publication with the package and field named.
5. Stores the accepted manifest and artifacts in an immutable release record.

Install-critical fields include dependency and peer declarations, optional and
bundled dependency semantics, platform and engine constraints, executable
mappings, lifecycle-script presence, and shrinkwrap semantics. The exact field
set and absent-value rules are a ratification requirement. A referenced
shrinkwrap remains part of the verified archive and must be interpreted
consistently by the resolver and installer.

Registry-generated data such as authentication identity and tarball locations
is not compared as publisher-authored manifest content. Deprecation messages
remain mutable annotations. Any dependency correction or replacement artifact
uses an explicit revision or a separately authenticated, selected policy
overlay; it never silently rewrites an accepted manifest.

For proxied releases, fetching the metadata alone creates an unverified
observation, not a verified record. The server can resolve through the existing
path, but cannot claim manifest-to-artifact verification until it has fetched
and checked the archive. A policy requiring verified records must perform that
work before accepting the result. Conflicting observations are recorded and
refused under that policy; old accepted records are not overwritten.

### Compression, revisions, and signatures

Reuse the existing integrity conventions:

- Ordinary resources keep the integrity target their protocol specifies.
- Every registry revision, including revision zero, keeps its existing
  uncompressed-tar integrity and immutable revision URL.
- A verified source `.tgz` and a derived tar are separate artifact subjects.
  Store their derivation relationship without relabeling a signature over one
  as a signature over the other.

The record digest is an additional metadata binding, not a replacement for SRI
or a new interpretation of `dist.integrity`.

A registry endorsement means that the registry accepted the manifest-to-content
binding. It is distinct from publisher provenance and does not prove a package
is benign. Existing npm signatures and provenance bundles remain available in
their original form.

### SQL metadata and backend contract

Use SQL for authoritative metadata and its relationships. Keep package archives
and large evidence bundles in the filesystem or S3-compatible object storage.
The logical storage split is:

| Data | Storage |
| --- | --- |
| Canonical release-record bytes and indexed identity fields | SQL |
| Package/version/revision bindings and artifact references | SQL |
| Tags, selected revisions, annotations, and policy bindings | SQL |
| Catalog generations, snapshots, receipts, and retention references | SQL |
| Publication outcomes and idempotency records | SQL |
| Archives and large evidence bundles | Filesystem or object storage |
| Generated npm metadata JSON | Disposable cache |

Preserve each record's exact canonical bytes alongside its digest. Indexed
columns support lookup and constraints, but database JSON serialization must
not determine the record digest. Storage row IDs and backend locations must
not enter signed package identities.

pnpr depends on domain operations such as accepting a release, committing a
publication, reading a package view, and retaining a dependency set. Each backend
owns its queries, migrations, and transaction mechanics. Do not expose a remote
SQL connection or vendor-specific types to registry handlers and resolvers.

Every supported backend must provide:

- Atomic metadata transactions and enforced uniqueness of release bindings.
- Commit-time conflict detection for concurrent publication and policy changes.
- Durable, queryable publication outcomes after a lost response or restart.
- Defined read consistency, including a path for current authorization checks.
- A consistent export/import procedure and shared backend conformance tests.

Transaction isolation and policy validation must jointly satisfy the publication
contract. A transaction that reads an old permission and writes a different
row is insufficient unless the backend detects or serializes that conflict.
Offline or asynchronous replica writes cannot acknowledge publication before
the authoritative commit. Replica and cache reads are usable only where their
freshness satisfies the operation's policy and generation requirements.

Embedded SQLite is the initial target for a single-instance deployment. Turso
is a candidate shared SQL backend, subject to conformance and workload testing.
The exact engine and replication mode must be evaluated; SQL syntax alone does
not establish equivalent isolation, durability, or freshness guarantees. Another
SQL backend can replace it through the same domain operations.

Backend replacement requires a consistent logical export or a quiesced transfer,
verification of record digests and referenced blobs, and a controlled cutover.
Preserve authority and package identities, canonical bytes, revision numbers,
retained generations, and transaction outcomes through their retry period.
Re-establish operator-approved trust and policy bindings at the destination;
importing records must not grant access. This is an operational transfer
between supported backends. Alpha format consolidation continues to follow
pnpr's existing experimental-format policy.

Keep this contract in this RFC. The
[snapshot draft](https://github.com/pnpm/rfcs/pull/45) and
[publication draft](https://github.com/pnpm/rfcs/pull/46) define their operations
against it. A backend substitution must preserve those operations and the
public registry protocol, not just load the same tables.

### Mutable state and compatibility views

Store these separately from immutable records:

- Tags and current revision selections.
- Deprecations and advisory or screening references.
- Access policy and retention references.
- Package presentation data, including the current README.

The npm-compatible full, abbreviated, and exact-version responses are views
over release records and this state. Deprecating a version does not change its
record digest. Selecting a revision changes the view using the existing
revision protocol.

Existing npm clients can consume the compatibility views without understanding
release records. Clients must negotiate support before relying on a record
binding. Tar-only and revision resources retain their existing client support
limits; this RFC does not declare them compatible with every npm client.

For example, SQL can bind `@example/widgets@1.2.0` to an immutable record that
declares `@example/core: ^2.0.0` and the archive digest. A separate tag row points
`latest` at that release. Deprecating `1.2.0` updates its annotation while the
record bytes and digest stay unchanged. The generated npm version entry joins
the accepted manifest and archive location with the current annotation.

Cache the serialized full and abbreviated responses separately. Publication,
tag, revision-selection, and annotation changes invalidate the affected package
view. Cache keys bind the representation and committed package generation,
plus any authorization context needed by the existing cache contract. Regenerate
eagerly or once on the next read, coalescing concurrent misses for the same view.
Ordinary reads can serve the cached JSON without recomposing every record.
Give live views bounded freshness and revalidate their generation on expiry or
startup. A lost invalidation event cannot leave an old view cached indefinitely.

### Authorization and lifecycle

Authorize every record, evidence, and artifact read through its package and
registry context. A global digest endpoint must not bypass package policy.
Deduplicated physical storage does not make private content public, and
unauthorized callers must not learn whether a private record exists.

An immutable record can outlive an artifact after an authorized deletion. Its
existence is not an availability guarantee. Retained snapshots provide that
separate guarantee. Current security policy can deny an otherwise valid record.

### Performance

Verify archives once per accepted artifact and reuse verified derivations.
A tag change should not invalidate immutable record or artifact caches.
Use indexed package lookups and batched metadata reads. Keep uploads, scanning,
and network requests outside SQL write transactions. Download counters and
request logs must not create a metadata write on every install.

Scope catalog coordination to the organization and hosted catalog rather than
one global row. Scale artifact traffic through object storage and authorized
caches. Database partitioning must preserve each advertised transaction boundary.
Avoid requiring every replica to hold the entire upstream metadata dataset.

Upstream verification can add downloads to lockfile-only resolution. Keep this
cost explicit and measure it separately from ordinary installs. This draft
claims no speedup before measurement.

File-level transfer is outside this draft. The existing
[pnpm agent proposal](https://github.com/pnpm/rfcs/pull/9) already discusses
that transport. A future extension can use verified release records to bind a
file manifest to an accepted artifact, while retaining client verification and
package authorization. Archive transport remains independently usable.

## Rationale and Alternatives

1. **Keep the package document authoritative.** Requires fewer storage changes,
   but immutable content and mutable policy continue to share a mutation and
   cache boundary. It also leaves the manifest-to-artifact binding implicit.
2. **Sign the whole package document.** Binds metadata but requires a new
   signature for tag and annotation changes and complicates historical reads.
   Separate immutable records preserve a useful long-lived verification unit.
3. **Keep all authoritative metadata in object storage.** Preserves an
   object-store-only deployment, but requires custom coordination for release
   uniqueness, tag changes, retention, and publication outcomes. SQL makes those
   relationships and commit boundaries explicit at the cost of operating a
   metadata database and coordinating its backup with the blob store.
4. **Use an OCI artifact manifest directly.** Provides a digest-based envelope,
   but does not settle npm manifest interpretation, revision selection, or
   package authorization. Reusing a standard envelope remains possible without
   making npm users adopt OCI repository semantics.

## Implementation

Target pnpr and new client support in pnpm v12. Do not add a feature to pnpm v11.

1. Specify the field set, deterministic encoding, authority mapping, and
   signature trust configuration. Add fixtures for the record contract.
2. Extend the existing publication staging and archive-reading code to produce
   verified records. Share normalization with the existing pnpm manifest code.
3. Split `pnpr-storage` metadata operations from artifact storage. Add SQL-backed
   immutable records and mutable state behind the domain interface, starting
   with embedded SQLite. Retain existing artifact validation and staging.
4. Generate registry responses and historical revision views from those records.
5. Add upstream verification and negotiated record retrieval. Preserve the
   existing authorization-aware cache and route classification.
6. Add a shared SQL backend only after its transaction, replica freshness,
   durability, and export/import behavior pass the same conformance suite.

pnpr is experimental. Consolidate its internal format in place, without
introducing migration fallbacks between alpha builds. Public client protocol
changes still require explicit capability and format-version handling.

Validation must cover mismatched dependencies, ambiguous archive manifests,
unsafe paths, bounded decoding, concurrent acceptance, and crash recovery.
Verify that deprecation and tag updates preserve record digests, historical
revisions keep their manifests, private digest reads cannot bypass policy,
and source signatures retain their original subjects. Benchmark cold and warm
reads and the extra downloads required by verified lockfile-only resolution.
Exercise concurrent publication and policy revocation, lost commit responses,
replica lag, cache invalidation loss, and export/import with identical record
digests. Measure metadata cache misses, ingestion throughput, write contention,
and retained-history growth before selecting a shared backend.

## Prior Art

- [Registry mounts and declared namespaces](./0000-pnpr-registry-mounts.md).
- [Integrity-addressed patch revisions](./0000-integrity-addressed-patch-revisions.md),
  especially historical manifests and source-to-tar derivations.
- [Tar integrity for every source](../../text/0000-tar-integrity-for-every-source.md).
- [npm package metadata](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md)
  and [registry signatures](https://docs.npmjs.com/about-registry-signatures/).
- [npm publication implementation](https://github.com/npm/cli/blob/latest/workspaces/libnpmpublish/lib/publish.js).
- [SQLite deployment guidance](https://www.sqlite.org/whentouse.html) and
  [Turso's Rust backend options](https://docs.turso.tech/sdk/rust/reference),
  to evaluate against the required storage contract.

## Unresolved Questions and Bikeshedding

- Which deterministic encoding, digest algorithm, and envelope should the
  record protocol adopt?
- What is the complete install-critical field set, including normalization and
  shrinkwrap treatment?
- How should operators establish portable authority identities and approve a
  source rebinding without embedding deployment hostnames in every lockfile?
- Which record signatures should clients require, and how are trust roots
  configured and rotated?
- Should hosted manifest verification become mandatory immediately, and what
  diagnostics should help publishers repair rejected packages?
- Which verified-record modes should upstream lockfile-only requests expose?
- What SQL schema, indexes, and isolation strategy meet the domain contract on
  embedded SQLite and the first shared backend?
- Which Turso engine and deployment mode, or alternative shared SQL backend,
  passes the conformance, durability, and workload requirements?
- What logical export format, cutover procedure, and coordinated metadata/blob
  backup process preserve identities and retained availability?

---
Written by an agent (Codex, GPT-6).
