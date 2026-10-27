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

A release record contains these logical fields. The concrete format below is a
strawman for review; adopting it is a ratification decision.

| Field | Contract |
| --- | --- |
| Schema | Identifies the record format and manifest interpretation rules. |
| Authority | Identifies the configured origin and its namespace. |
| Package | Ecosystem, validated name, version, and revision when applicable. |
| Interpretation | Versioned rules for parsing and deriving install metadata. |
| Inputs | Exact byte digests of the archive manifest and shrinkwrap, if present. |
| Manifest | Install-critical metadata derived from the accepted archive. |
| Artifact | Complete algorithm, digest, size, and byte representation of the accepted resource. |

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

### Record envelope and canonical encoding strawman

Use [RFC 8785 JCS](https://www.rfc-editor.org/rfc/rfc8785) for canonical record
bytes and SHA-512 SRI for record integrity. This formatted example uses
placeholder digests. It describes an ordinary `.tgz` release, so it carries no
registry revision marker:

```json
{
  "record": {
    "schema": "pnpr-release-record/1",
    "authority": "example-hosted-npm",
    "package": {
      "ecosystem": "npm",
      "name": "@example/widgets",
      "version": "1.2.0"
    },
    "interpretation": "npm-install-manifest/1",
    "inputs": {
      "packageJson": {
        "path": "package/package.json",
        "integrity": "sha512-<exact-package-json-bytes>"
      },
      "shrinkwrap": null
    },
    "manifest": {
      "dependencies": { "@example/core": "^2.0.0" },
      "hasInstallScript": false
    },
    "artifact": {
      "kind": "tgz",
      "integrity": "sha512-<archive-bytes>",
      "size": 4231
    }
  },
  "recordIntegrity": "sha512-<canonical-record-bytes>",
  "endorsements": [],
  "evidence": []
}
```

Compute `recordIntegrity` as `sha512-` followed by the standard base64 encoding
of `SHA512(JCS_UTF8(record))`. Hash only the inner `record`; integrity,
endorsements, and evidence are outside that input. A detached endorsement must
authenticate the record integrity under a release-record-specific signing
context. The signature format, algorithms, and trust-root configuration remain ratification
items. An empty endorsement list supplies no signer authentication.

The JCS input rejects duplicate keys, invalid Unicode, and unsupported numeric
values. Schema integer fields stay within the exact JSON safe-integer range;
larger values require a future schema or an explicitly specified string field.
Use JCS's property ordering and number serialization, preserve string contents
without Unicode normalization, and hash the resulting UTF-8 bytes. A generic
sorted-key JSON serializer is not sufficient.

The `inputs` digests cover exact file payload bytes before parsing, including
whitespace and line endings. Paths identify the validated archive entries used
by the interpretation profile. Present shrinkwrap data has the same path and
integrity descriptor; `null` means absent. A revision record adds the existing
revision number and describes its tar artifact under the revision protocol.
Source `.tgz` evidence and tar derivations retain distinct subjects.

The inner record is a deterministic function of the declared authority, package
identity, interpretation profile, and exact accepted artifact bytes. It contains
no observation times, signature bundles, provenance, verification outcomes,
transport locations, or other facts that can change independently of that
artifact. Alternate encodings and source-to-tar mappings are detached derivation
evidence, not additional mutable artifact descriptors inside the record.

Outer `endorsements` authenticate the release record. Outer `evidence` carries
or references separately authenticated source signatures, provenance, derivation
links, and analysis. Preserve each source statement's original artifact subject;
an authenticated derivation can link it to the record without relabeling it as
a direct signature over the record or derived tar. An outer list is not itself
authenticated by `recordIntegrity`; verify each required statement and linkage.

Delayed evidence, upstream re-signing, and signer rotation leave canonical record
bytes and integrity unchanged. The envelope can change as those detached sets
change, so its HTTP validator must include its evidence/endorsement generation.
The record digest alone is not an ETag for the whole envelope. Cache payloads
and evidence separately and enforce current trust and freshness policy on reads.

### Versioned manifest interpretation

`npm-install-manifest/1` names a protocol contract, not whichever normalization
behavior the current pnpm release happens to implement. Publish shared fixtures
for accepted input bytes, derived fields, canonical record bytes, and digests.
The server and new pnpm v12 client can share Rust implementation code, while
the fixtures define interoperability independently of implementation language.
This does not add record support to the maintenance-only pnpm v11 client.

The profile must specify absent versus empty fields, dependency precedence,
`bin` and `directories.bin`, bundled-dependency aliases, peer metadata,
platform restrictions, script detection, and shrinkwrap interpretation before
ratification. Preserve the raw input digests even when normalization makes two
different input representations equivalent.

A correction that changes an accepted derived manifest creates a new record
under a new interpretation profile. Keep the previous bytes, digest, and
profile-specific binding; never rehash or overwrite historical records in
place. Clients must explicitly support the new profile. An unsupported or
withdrawn profile produces a policy or capability error rather than an
unannounced reinterpretation. Artifact revision numbers do not change when
only the interpretation changes and the archive bytes stay identical.

### Manifest acceptance

For a hosted npm publication, pnpr:

1. Verifies submitted archive integrity and validates the archive under bounded
   decoding and entry-count limits.
2. Reads the manifest that the supported installer would select. Rejects
   duplicate or ambiguous manifest entries and unsafe archive paths.
3. Applies the named interpretation profile and records its exact input digests.
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

### Upstream observations and disagreement policy

For proxied releases, fetching metadata creates an unverified package-index
observation: the captured upstream document, candidate set, tags, and validators.
Preserve this separately from archive-derived records. A version observation
contains one version's profile-interpreted install claims and its claimed
artifact kind and integrity, bound to the authority, package identity, and
interpretation profile. Its integrity hashes the JCS encoding of those claims.
It excludes unrelated versions, tags, deprecations, transport URLs, signatures,
and attestations. Retain the source-document reference separately for audit.

Compare version observations with archive-derived records under the same
interpretation profile. Key comparison evidence by version-observation integrity
and record integrity. Publishing an unrelated version, changing a tag, or adding
source signatures does not invalidate an unchanged version's comparison. Changing
its install claims or claimed artifact integrity does. Package-index snapshots
still capture the full candidate and tag state needed for resolution replay.
A signature on an archive-derived record alone does not attest that an upstream
observation matched it. Exported comparison evidence needs its own authentication.

These are proposed policy meanings, not final configuration names:

| Mode | Resolution contract |
| --- | --- |
| Compatibility | Can resolve from observed upstream metadata; reports it as unverified. |
| Verify and match | Requires a verified archive and matching install-critical metadata; disagreements fail closed. |
| Archive authoritative | Explicitly accepts the archive-derived manifest despite a recorded upstream disagreement. |

Verification-required requests use verify-and-match unless the operator has
explicitly selected archive-authoritative acceptance for that package, complete
artifact integrity, and interpretation profile. There is no implicit exemption
for old publications and no downgrade to compatibility after verification fails.
The record, policy choice, observations, and disagreement remain inspectable to
authorized callers and are bound into resolution receipts.
Archive-authoritative acceptance never waives archive integrity, package
identity, archive safety, or source trust requirements.

Archive-authoritative acceptance changes the metadata used to resolve that
artifact. Re-run affected resolution using the accepted archive manifest, and
generate both compatibility and native views from that same selection. Never
resolve with the upstream dependency list and then replace only its checksum or
manifest after the graph is complete. Old observations, records, and snapshots
remain unchanged. This is an explicit policy selection, not a silent mutation
of publisher metadata or an assertion that the upstream agreed.

Verify candidate artifacts before accepting their graph contributions. Range
selection can use observed metadata to nominate candidates, then backtrack if
verified metadata changes their applicability. Fetching every version in a
packument is not required, but rejected candidates can require extra downloads.
Bound verification work, coalesce concurrent fetches, and cache verified
archive/input/profile derivations. Comparison caches additionally bind the
version-observation integrity, so a prior match cannot validate changed upstream
claims. A refreshed full upstream document may still require parsing; retaining
per-version comparisons avoids repeating archive verification because an
unrelated part of that document changed.

Cold lockfile-only resolution under verification can download archives it would
otherwise not need. Expose those downloads and the chosen mode in diagnostics
and benchmarks. Historical discrepancies are a compatibility case to measure;
this draft does not claim their prevalence without an upstream corpus study.

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

Embedded SQLite is the initial target for a single-instance deployment.
PostgreSQL and Turso are first-class candidates for shared deployments, subject
to the same conformance and workload tests. Evaluate PostgreSQL transactions
with explicit policy-row locking or serializable validation and full-transaction
retry. Evaluate Turso's specific engine, remote transaction/session boundaries,
and replica mode separately. SQL syntax alone does not establish equivalent
isolation, durability, or freshness guarantees. Another SQL backend can replace
them through the same domain operations.

All statements in a domain transaction must share one authoritative database
transaction through a pinned connection or documented session handle. Separate
HTTP statements cannot be assumed to share that boundary. Serialization failures
retry the whole metadata transaction, preserving the idempotency contract.

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

### Relational keys strawman

The following logical tables make the key constraints concrete without choosing
vendor types or a complete production schema. All listed key columns are
non-null. `package_key` abbreviates `(authority, ecosystem, name)` and is scoped
to its catalog. `version_observation_key` expands to the complete primary key
of `version_observations` below.
`revision_key` is `ordinary` outside the revision protocol or
`rN` for an explicit registry revision, including `r0`; it is an internal key,
not a new version or integrity syntax.

| Relation | Primary key | Constraint or reference |
| --- | --- | --- |
| `release_records` | `record_integrity` | Insert-only canonical bytes and interpretation profile; recompute integrity on acceptance. |
| `artifact_slots` | `(catalog, package_key, version, revision_key)` | Immutable artifact-integrity assignment. |
| `record_bindings` | `(catalog, package_key, version, revision_key, profile)` | References an artifact slot and one release record with matching identity, artifact, and profile. |
| `package_observations` | `(catalog, package_key, index_integrity)` | Immutable source document, candidate set, and tag observation. |
| `version_observations` | `(catalog, package_key, version, revision_key, profile, observation_integrity)` | Immutable per-version install and artifact claims, excluding mutable evidence and package-index state. |
| `manifest_comparisons` | `(record_integrity, version_observation_key)` | Immutable comparison result and field differences; both references remain authorized. |
| `catalog_generations` | `(catalog, generation)` | References its parent and committed changes; policy and current-generation rows belong to this catalog. |
| `publication_outcomes` | `(catalog, principal, idempotency_key)` | Binds request integrity and committed generation in the publication transaction. |
| `snapshot_observations` | `(snapshot, package_key)` | References the captured package-index observation. |
| `snapshot_version_observations` | `(snapshot, version_observation_key)` | References the per-version claims used by that resolution. |
| `snapshot_records` | `(snapshot, record_integrity)` | References the profile-specific release records used. |
| `retention_members` | `(retention_set, resource_integrity)` | References the required durable resource; expiry belongs to its retention set. |

Authenticated evidence and source-document links reference record, artifact,
or version-observation identities separately. A later source document can link
to the same version observation without changing that observation's integrity
or its comparison results. Their physical deduplication remains subject to the
same namespace and authorization rules.

Tags, selected revisions, and annotations are catalog state with versioned
changes so older generations can reconstruct their selections. They do not
update `release_records` or an existing profile-specific `record_bindings` row.
Store generation changes with structural sharing or version intervals; the
logical relations do not require copying every catalog entry on each commit.
Snapshots and receipts have their own immutable manifests and digests.

Release-binding foreign keys and transaction-time validation must preserve the
declared catalog and package boundary. A snapshot can span configured origins,
but each referenced package still requires authorization. An imported canonical
row cannot establish a trusted binding by itself. The domain operation checks
its record identity, artifact slot, authority, and profile before inserting the
binding. Uniqueness must cover the complete expanded key, including namespace
and interpretation; nullable revision columns must not weaken duplicate-publication
protection.

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
The upstream acceptance policy and version-observation binding gate cache hits;
compatibility results never satisfy verification-required requests.

Cache per-version rendered fragments by record integrity, interpretation,
representation, renderer version, and relevant annotation generation. A tag-only
change can reuse those fragments and update the package-level selection data.
Formats carrying detached evidence also key on its evidence/endorsement generation.
Appending a release renders its new fragment; deprecating one version renders
the affected fragment. Do not refetch, join, parse, and render every unchanged
version after each mutation. Stream or assemble the cached fragments under one
committed package view, and cache the resulting full response. Native record
reads do not require that full response.

Returning a full packument still requires emitting its bytes. This proposal
reduces repeated database and rendering work; it does not make the existing
all-version npm response constant-sized or remove its serialization/copying cost.

### Record discovery and retrieval strawman

Extend the existing `GET /-/pnpr` capability document with these proposed fields:

```json
{
  "records": [1],
  "recordCandidates": [1],
  "manifestProfiles": ["npm-install-manifest/1"]
}
```

A client opts into an extended metadata representation, proposed as
`application/vnd.pnpr.install-v1+json`. For an accepted version, `pnprRecord`
contains the complete envelope above, including the inner record, its integrity,
endorsements, and detached evidence. The client can reconstruct and verify the
canonical record locally. A pointer-only entry is not sufficient to resolve
under a record-required policy. Validate that the ordinary version fields and
the record describe the same accepted manifest and artifact.

Inline envelopes carry the endorsement and evidence set required by the active
policy, rather than every historical statement. If that required set exceeds
response limits, return an explicit error; do not truncate required verification
data and present the result as accepted.

Standard npm metadata remains available with its existing representation. An
extended package document can include already-accepted records, but requesting
it does not require verifying every historical archive. Cache variants and
`Vary: Accept` distinguish it from ordinary full and abbreviated metadata.

For cold client-side resolution, make a bounded candidate lookup directly,
without first fetching a package document and then fetching each record:

```http
POST /npm/~local/@example/widgets/-/record-candidates
```

An illustrative request body is:

```json
{
  "spec": "^1.0.0",
  "manifestProfiles": ["npm-install-manifest/1"],
  "maxCandidates": 4
}
```

The request also carries relevant existing resolution-policy and target settings
under the versioned lookup schema. The response's `versions` map supplies bounded
candidate-selection metadata and each returned version's complete `pnprRecord`
inline. Reuse existing version-nomination and policy primitives, verify only
the nominated candidates, and enforce server limits on candidate count, bytes,
and verification work. A policy block remains an explicit refusal. It must not
be disguised as an empty result or silently skipped to select another artifact.

Backtracking can request another page. Its cursor binds the captured package
index, filters, profile, and access context so a tag change cannot silently alter
the candidate set between pages. Recheck current authorization and policy.
Candidate POST responses default to private, non-shared HTTP caching; internal
cache keys include the complete semantic request and existing access descriptor.
Server-side graph resolution should likewise carry required envelopes inline
in its versioned response instead of making the client retrieve them separately.

Standalone retrieval remains useful for auditing or checking a known record:

```http
GET /npm/~local/@example/widgets/-/records/sha512/<base64url-digest>
```

Both paths are relative to the effective registry base: append
`<package-path>/-/record-candidates` or
`<package-path>/-/records/sha512/<digest>`. Current
[npm registry bases](https://github.com/pnpm/pnpm/blob/7535582cd7c42df60f61ea6edaaf3cc49d861316/pnpr/docs/endpoints.md)
are `/npm/` and `/npm/~<name>/`; older registry RFCs write `/~<name>/` as a
registry-base example.
This proposal adds no root-level alias. The package is part of the path, not a
query-string access selector. Scoped names use the existing package decoding
and validation rules; encoded and literal scoped spellings must receive the
same routing and authorization checks. Cache keys preserve registry and package
context for every accepted spelling.

The digest path uses complete digest bytes encoded as unpadded base64url; the
record retains standard SRI. Resolve the registry and validate the package path,
authorize it, then verify the record is bound there. A digest is never a global
read capability. Private responses and conditional requests enforce current
access; unauthorized requests must not reveal existence through status,
validators, or caches. No redirect can rebind a record to another authority.

Whether inline or standalone, the client checks schema, interpretation support,
record integrity, required endorsements and evidence, package identity, and
artifact binding before using the manifest. Required support cannot silently
fall back to ordinary metadata. Cached records and verified receipts avoid
repeat lookups. This removes the mandatory metadata-then-record exchange for
each candidate; it does not remove graph traversal, backtracking, or cold
archive-verification costs. Frozen installs retain their existing fast path.

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
   verified records. Reuse the pnpm manifest implementation under the versioned
   interpretation contract and preserve raw input digests.
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

Add cross-implementation canonicalization fixtures, including Unicode and number
edge cases, raw-manifest byte changes, shrinkwrap presence, and interpretation
upgrades over unchanged archives. Validate each upstream policy with matching
and conflicting metadata; archive-authoritative resolution must use the changed
dependency graph and old receipts must keep their profile. Verify that a large
package's tag-only update reuses unchanged version fragments. Measure cold and
warm full/abbreviated generation, SQL rows read, and rendering work for packages
with thousands of versions. No corpus compatibility or throughput claim is made
until those measurements exist.

Validate inline candidate responses without a preliminary package-document or
standalone-record request, including bounded paging and backtracking. Check that
tag changes, unrelated version publishes, delayed attestations, and re-signing
preserve existing record integrities and per-version comparison hits. Changed
install claims must invalidate the affected comparison; evidence updates must
change envelope validators without rekeying the record. Test default/named bases,
scoped-name spellings, and private access on both candidate and record routes.

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
- [JCS](https://www.rfc-editor.org/rfc/rfc8785), for canonical record encoding.
- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
  and [explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html),
  for the shared-backend conformance evaluation.

## Unresolved Questions and Bikeshedding

- Should the draft adopt the JCS/SHA-512 envelope strawman, and which detached
  endorsement format and signing context should authenticate it?
- What is the complete install-critical field set, including normalization and
  shrinkwrap treatment?
- How should operators establish portable authority identities and approve a
  source rebinding without embedding deployment hostnames in every lockfile?
- Which record signatures should clients require, and how are trust roots
  configured and rotated?
- Should hosted manifest verification become mandatory immediately, and what
  diagnostics should help publishers repair rejected packages?
- What configuration names and approval flow expose the upstream policy modes,
  and what does a corpus study show about mismatches and verification downloads?
- What SQL schema, indexes, and isolation strategy meet the domain contract on
  embedded SQLite and the first shared backend?
- Which PostgreSQL configuration and Turso engine/deployment mode pass the shared
  conformance, durability, and workload requirements, and which should ship first?
- Are the proposed inline envelopes, bounded candidate request/cursor schema,
  and package-scoped retrieval paths the right initial protocol and limits?
- What detached evidence envelope and linkage rules authenticate original
  subjects and keep evidence validators current without changing record identity?
- What logical export format, cutover procedure, and coordinated metadata/blob
  backup process preserve identities and retained availability?

---
Written by an agent (Codex, GPT-6).
