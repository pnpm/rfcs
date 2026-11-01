# Atomic workspace publication for pnpr

> Status: draft for discussion. No implementation or protocol version is approved.

## Summary

Publish a related set of packages by preparing immutable artifacts and then
committing their release records, bindings, selections, and outcome in one SQL
metadata transaction. The transaction advances a hosted catalog generation.
A committed generation exposes the whole batch. An aborted transaction exposes
none of it.
Snapshot-aware readers pin that generation for their operation, so concurrent
publication cannot give them a mixture of the old and new release.

This strengthens pnpr's existing journaled batch publication. It builds on
the [SQL metadata contract](https://github.com/pnpm/rfcs/pull/48) and composes with
resolution snapshots. An initial implementation can restrict batches to one
organization and one configured hosted catalog; it does not promise transactions
across independent servers.

## Motivation

Consider a workspace publishing `@example/core@2.0.0` and
`@example/widgets@2.0.0`, where widgets requires the new core. Per-package
publication exposes a window in which one package is available and the other
is not. Updating their tags separately creates the same issue.

pnpr already verifies a batch before applying it and journals publication for
recovery. Its current contract nevertheless permits readers to see part of a
batch while it applies. A concurrent writer that wins a version slot can also
leave the rest of the batch published with a conflict reported to the caller.

Those are documented limits, not an absence of batch support. This proposal
removes the partial-publication outcome for supported batches and gives
snapshot-aware readers a consistent release view.

### Evidence

On the npm registry, publication is one request per package and a lost
response has no recovery path. 78 npm/npm and npm/cli issues report "cannot
publish over previously published version" or `EPUBLISHCONFLICT`, many for a
version nobody can see.

| Report | Year | 👍/comments | What happened |
| --- | --- | --- | --- |
| [npm/cli#10078](https://github.com/npm/cli/issues/10078) | 2026 | 0/0 | A lerna release retried a publish that had succeeded. "22 of 23 packages published and one missing, so the release ended up half-done." |
| [npm/cli#9889](https://github.com/npm/cli/issues/9889) | 2026 | 3/4 | `pnpm publish -r` of platform bindings. "The first publish that triggered this reported success." One package is wedged and the rest are live. |
| [npm/cli#9313](https://github.com/npm/cli/issues/9313) | 2026 | 0/1 | Workspace publish rate-limited partway. "many packages publish successfully, but a few later packages fail repeatedly with 429". |
| [npm/cli#7907](https://github.com/npm/cli/issues/7907) | 2024 | 0/6 | `npm publish -ws` stops at the first version that already exists, so later packages are never published. |
| [npm/cli#5058](https://github.com/npm/cli/issues/5058) | 2022 | 0/16 | "ran `npm version major` enough times until I got past those ghost versions". |
| [npm/npm#4922](https://github.com/npm/npm/issues/4922) | 2014 | 0/39 | The document `PUT` returned 201 and the attachment `PUT` returned 409. Every later version also conflicted. |
| [npm/rfcs#778](https://github.com/npm/rfcs/issues/778) | 2024 | 0/5 | One OTP prompt per package in a workspace publish. |
| [npm/npm#13](https://github.com/npm/npm/issues/13) | 2010 | 0/4 | Transactional publishing requested in npm's 13th issue. |

### Current pnpr gaps this must close

- **A byte-identical retry is a conflict.** `check_publishable_versions` in
  `pnpr/crates/pnpr/src/server/publishing/manifest.rs` returns
  `VersionAlreadyPublished` whenever a tarball arrives for an existing version,
  even when the bytes match. A client whose response was lost cannot learn
  that it succeeded.
- **One request body carries every archive.** The batch body is base64 JSON
  under the router-wide 100 MiB limit (`MAX_PUBLISH_BODY_BYTES` in
  `pnpr/crates/pnpr/src/server.rs`), so batch size is bounded by one body.
- **Keyless CI cannot use it.** An OIDC workload credential may only
  `PUT /npm/~<registry>/<package>` (`check_workload_request` in
  `pnpr/crates/pnpr/src/server/oidc.rs`). It cannot call the batch endpoint or
  move a dist-tag, so the CI release path, which most needs atomicity, cannot
  use it.

## Detailed Explanation

### Transaction boundary

A hosted catalog is the authority for a declared set of packages under one
organization. It has a current reference to an immutable catalog generation.
Each generation identifies package release records and mutable selections,
including tags and selected revisions.

The SQL metadata database is the authority for catalog generations, release
bindings, publication outcomes, and idempotency keys. Archives and large
evidence bundles stay in the filesystem or object storage. Backend-specific
queries implement the shared domain contract; publication does not expose a
Turso-specific API or depend on one vendor's replication behavior.

A transaction may contain new releases and the tag or revision-selection
changes that should accompany them. All participants must belong to the same
supported catalog. Reject an unsupported cross-catalog request before commit;
do not silently split it into several transactions.

Cross-ecosystem publication can share this boundary when all the ecosystem
views read the same catalog. Independent npm, Cargo, Python, or OCI indexes
cannot each be updated as separate authorities while claiming atomicity.
Protocol views must be derived from the committed catalog reference.

Every hosted writer that affects catalog state participates in this model:
single-package publication, batch publication, unpublish, tag updates,
staged-publication promotion, and registry revision selection. A legacy path
that mutates a separate authoritative package document would break the guarantee.

### Prepare and commit

1. Authenticate the publisher and authorize every intended mutation.
2. Validate package records, artifact integrities, and requested selections.
   Upload and durably stage artifacts without exposing them as releases.
   Protect staged objects from garbage collection while commit is pending.
3. Begin a short authoritative SQL transaction and acquire or recheck the
   idempotency claim. A previously committed identical request follows the
   authorized outcome-query path instead of publishing again.
4. Read the relevant catalog state and validate binding uniqueness, mutation
   preconditions, and current permissions through the policy conflict gate below.
5. Insert the immutable release records, versioned bindings, new selections,
   catalog generation, and durable transaction outcome.
6. Commit the SQL transaction. This is the publication's commit point. A
   constraint or policy conflict rolls back the entire transaction. A lost
   commit acknowledgement requires outcome recovery before claiming success
   or failure.
7. Return the committed generation and recorded outcome. Invalidate generated
   package views through durable generation changes.

Every artifact referenced by the new generation must be readable and verified
before SQL commit. A successful response cannot precede that durability
boundary. Object storage is outside the SQL transaction. A failed metadata
commit can leave unreferenced uploaded objects for coordinated cleanup; it must
never leave a visible release referencing an unfinished upload. Archive uploads,
scanning, and other network work stay outside SQL write transactions.

Archives are uploaded one at a time, addressed by their integrity, before the
commit request. The commit body references archives by integrity and carries
no archive bytes, so batch size is not bounded by one request body. An upload
of bytes that are already stored is a no-op, which makes a resumed release
upload only what is missing. pnpr's shared-artifact blob endpoints
(`PUT` and `HEAD /-/pnpr/v0/artifacts/blob?integrity=...`) already have this
shape, and its OCI blob upload handles large chunked bodies.

Use an authoritative policy epoch for the catalog, updated by every permission
mutation affecting it. Publication and policy mutation must lock or perform
conflicting writes on that epoch inside their metadata transactions, with
authorization evaluated against the protected state. Equivalent serializable
validation is acceptable when the backend proves the same conflict behavior.
A policy change that commits first forces publication to re-evaluate or abort;
one serialized after publication does not undo its completed commit.

SQL transactions alone do not establish this guarantee. Snapshot isolation
that lets publication read an old permission while writing unrelated rows can
permit write skew. The exact lock or conditional-update mechanism must be
specified and tested for each backend before it advertises the capability.

### Concurrent writers and retries

If a concurrent writer invalidates a binding or generation precondition, the
SQL transaction aborts without exposing part of the batch. The server can retry
only after revalidating the complete batch and current authorization. Database
uniqueness constraints enforce version/revision assignments; a conflicting
version fails the entire transaction.

Existing releases never change their bytes. An identical request retry may
return its earlier result, but a new transaction cannot quietly adopt another
publisher's competing release to make its batch appear successful.

Clients supply an idempotency key scoped to the organization and publisher
identity. Bind that key to a digest of the complete request. Reusing it with a
different request is a conflict. The request identity is scoped to a stable
principal, not a bearer-token string that changes during token rotation.

Existing npm clients send no idempotency key, and npm, pnpm, and lerna retry
a timed-out publish. For a single-package publication without a key, the
request's content is the key: a publication of `name@version` whose archive
integrity and accepted manifest equal the committed release returns the
recorded success, not a conflict. A different archive for the same version
is still a conflict.

A timed-out commit has an unknown outcome to the client. The client queries or
retries the same transaction to learn its durable state. Idempotency results
must survive crashes and the documented retry period. A retry must not create
another catalog generation for an already committed transaction.

### Publication visibility and read consistency

Atomic publication means that every catalog generation contains either all of
the batch or none of it. It does not mean that several independent requests to
live endpoints necessarily read the same generation.

- A snapshot-aware resolve captures one generation and uses it for every hosted
  metadata lookup, including tags and selected revisions.
- An exact artifact reference continues to identify the accepted bytes even
  after a newer generation is committed, subject to retention and current policy.
- Ordinary npm clients can use generated compatibility views, but separate
  requests to a live base can straddle the commit. This RFC does not promise
  multi-request snapshot isolation to those clients.

Generate compatibility metadata on demand from the authoritative generation,
or publish generation-specific cached views. Cache keys must include the
generation. An asynchronously refreshed per-package document cannot become a
second source of truth. A live CDN may serve an older complete view within its
declared freshness window; snapshot endpoints must remain generation-specific.

The server must advertise both publication support and snapshot-read support
explicitly. Do not change the meaning of an existing batch capability bit to
claim the stronger guarantees without a protocol-version change.

### Publications

A committed transaction that publishes at least one package version is also
recorded as a **publication**, which clients can read. A single-package
publish is a publication with one member, so every version belongs to
exactly one publication. A publication has:

- an identifier that pnpr assigns, and optionally a name the publisher
  chooses, such as `v12.0.0`, unique within the organization;
- its members: the ecosystem, name, version, and revision of every package
  version it published, whatever ecosystem each belongs to;
- the identity that published it, with the source repository and commit of
  a workload identity, the commit time, and the catalog generation it
  created.

All of this is fixed at commit. Tag moves and revision selections that commit
alone are transactions but not publications; they change selections, not the
set of published versions.

A publication belongs to no single ecosystem, so it is served from the
server root rather than an ecosystem's prefix:

```text
GET <pnpr>/-/publications/<org>/<id-or-name>
GET <pnpr>/-/publications/<org>?member=<ecosystem>:<name>@<version>
```

The second form finds the publication of one package version. A caller sees
only the members whose packages it may read, and a publication with none
answers `404`, so a publication never discloses a package to a caller who
cannot read it. A version's entry in an ecosystem view may name its
publication where the view has room for it, as pnpr's release record does
with a `publication` field.

Publications give a release a place for what belongs to the whole set rather
than one package, such as its
[release notes](https://github.com/pnpm/rfcs/pull/56).

### Crash recovery and garbage collection

Before SQL commit, staged artifacts are not published and may eventually be
collected after their protection expires. After commit, publication is complete
even if the response is lost. Recovery reads the durable SQL transaction outcome
and catalog state. The database's commit result, rather than the survival of a
replica-local journal, determines whether publication became authoritative.

Uncommitted candidates are protected while a transaction can still commit.
Garbage collection must coordinate with active transactions and retained
generations. Old generations remain reachable while a snapshot or retention
reference needs them. Deleting a tag never makes a retained artifact eligible
for accidental collection.

Recovery and idempotency information required to decide a committed outcome
must commit in the same SQL transaction as the release bindings and selections.
A replica-local journal may assist artifact staging. Backend export/import
must preserve outcomes for the retry period and generation identities, so a
database replacement cannot duplicate a publication or lose a known result.

### Authorization and policy

A transaction handle, catalog digest, or object digest grants no read access.
Authorize status queries and all generation, record, and artifact reads.
Staged objects must not be readable through ordinary registry routes before
commit, even if their digest is guessed.

Pinning a catalog generation freezes package selection, not current access or
security policy. A new block can refuse a package in an older generation.
Authorized diagnostics should distinguish a policy refusal from an absent
release. Read authorization must not be satisfied by publication permissions
or another organization's reference to identical bytes.

OIDC workload principals can submit transactions. Each member package and
each tag or revision selection is authorized against the workload binding,
the same as a single publication. A binding grants a package list and a set of
actions, such as publish and tag, rather than one `PUT` route.

Staged publication can hold a whole transaction. Approval commits the held
transaction as one generation, so a reviewed release is never promoted one
package at a time.

### Storage and performance

Store generation membership as versioned SQL bindings or a persistent structure
that shares unchanged metadata. Do not copy an entire catalog for each commit.
Index package and generation lookups and batch metadata reads. The schema must
preserve historical generations without keeping a database transaction open
for a client's entire install.

The catalog and policy epoch can be contention points. Initially scope them to
an organization and hosted catalog, reuse existing artifact storage, and
benchmark realistic concurrent publishers before expanding the boundary. Avoid
a global catalog counter, per-download writes, and long write transactions.
Partitioning is acceptable only when it preserves the advertised boundary;
splitting a batch across shards without coordination is not atomic publication.

## Rationale and Alternatives

1. **Keep journaled per-package updates.** Provides useful recovery and remains
   the baseline, but cannot hide intermediate state from concurrent readers or
   prevent a late conflict from producing a partial batch.
2. **Use a distributed lock around package updates.** Serializes writers but
   does not prevent unlocked readers from seeing intermediate state. Making
   every registry read participate would add coordination to the hot path.
3. **Use an object-store catalog pointer as the commit authority.** A
   compare-and-swap can publish one immutable generation, but policy checks,
   idempotency results, retention, and recovery still need coordinated state.
   SQL is the chosen metadata authority and commits these relationships
   together. Operating and backing up that database is an explicit deployment
   cost; object storage continues to hold the large artifacts.

## Implementation

Target pnpr, with new publishing and snapshot-read support in pnpm v12 only.

1. Define SQL catalog generations, binding constraints, transaction identities,
   outcomes, and a policy epoch in `pnpr-storage`, under the shared storage
   contract. Specify the isolation and durability mechanism for each backend.
2. Route all hosted mutations through the same SQL commit path. Reuse existing
   validation, artifact staging, and immutable revisions.
3. Make npm and other ecosystem views read the catalog. Remove any competing
   authoritative update path before advertising support for that ecosystem.
4. Add idempotent transaction status and capability-negotiated batch semantics.
5. Record publications with their members and serve them from the
   publication routes, filtered by the caller's read access.
6. Connect server-side resolution and retained snapshots to pinned generations.

Required validation includes concurrent publications of overlapping versions,
non-overlapping writers, conflicting tag updates, and policy revocation racing
commit. Inject crashes before and after each durability boundary, including a
lost successful response. Exercise a restarted replica with an empty local
cache and journal. Readers pinned to old and new generations must each see a
complete batch. Garbage collection must never delete an active transaction's
objects or a retained generation. Test existing clients separately from
snapshot-aware clients and report their actual consistency guarantees.

Run the same publication conformance suite against each SQL backend, including
replica lag and policy write skew. Verify that replacing a backend through
export/import preserves record digests, generations, revision assignments,
current permissions, and idempotent outcomes. Failures between artifact upload
and SQL commit must leave no visible release and permit safe staging cleanup.

Benchmark publication latency, contention retries, generation storage growth,
and cold/warm metadata reads at realistic catalog sizes. This draft makes no
claim that atomicity is free or faster than the existing publication path.

## Prior Art

- [Existing batch publication and its limits](https://github.com/pnpm/pnpm/blob/7535582cd7c42df60f61ea6edaaf3cc49d861316/pnpr/docs/ecosystems.md#one-publish-for-a-workspace-that-spans-ecosystems).
- [Existing concurrent-writer and journal behavior](https://github.com/pnpm/pnpm/blob/7535582cd7c42df60f61ea6edaaf3cc49d861316/pnpr/docs/storage.md).
- [Native monorepo versioning](../../text/0006-monorepo-versioning.md), which
  concerns version planning and does not itself establish registry read isolation.
- [Registry revisions](./0000-integrity-addressed-patch-revisions.md), whose
  immutable artifacts and selections fit the catalog model.

## Unresolved Questions and Bikeshedding

- What versioned SQL catalog schema and policy-epoch locking or validation
  strategy preserve the contract on each supported backend?
- Is one organization and hosted catalog the right initial transaction scope?
- How should clients query transaction outcomes and pin read generations? A
  publication's name names a committed transaction; should it also serve as
  the idempotency key?
- How long should idempotency records and unreferenced generations survive?
- Should content-keyed idempotency for keyless clients also cover batches,
  where the batch digest is the key?
- Should the first implementation cover npm only, or every ecosystem whose
  hosted mutation paths can share the catalog authority?
- What contention and storage measurements should gate making this the default
  hosted publication path?

---
Written by an agent (Codex, GPT-6).
