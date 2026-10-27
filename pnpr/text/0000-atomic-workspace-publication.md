# Atomic workspace publication for pnpr

> Status: draft for discussion. No implementation or protocol version is approved.

## Summary

Publish a related set of packages by preparing immutable objects and then
atomically advancing one hosted catalog reference. A committed catalog
generation exposes the whole batch, and a failed batch exposes none of it.
Snapshot-aware readers pin that generation for their operation, so concurrent
publication cannot give them a mixture of the old and new release.

This strengthens pnpr's existing journaled batch publication. It builds on
immutable release records and composes with resolution snapshots. An initial
implementation can restrict batches to one organization and one configured
hosted catalog; it does not promise transactions across independent servers.

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

## Detailed Explanation

### Transaction boundary

A hosted catalog is the authority for a declared set of packages under one
organization. It has a current reference to an immutable catalog generation.
Each generation identifies package release records and mutable selections,
including tags and selected revisions.

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
2. Validate all package records, artifact integrities, and requested selections.
   Stage immutable objects durably without making them discoverable as releases.
3. Read the current catalog generation and verify name/version conflicts and
   other mutation preconditions against it.
4. Build and durably store a new immutable generation referencing the prepared
   objects and the proposed selections.
5. Recheck commit authorization against the authoritative policy generation.
6. Compare-and-swap the current catalog reference from the expected generation
   to the new generation. This is the publication's commit point.
7. Return a durable transaction result identifying the committed generation.

Every object referenced by the new generation must be readable and verified
before the reference can advance. A successful response cannot precede that
durability boundary. A backend unable to provide the required conditional-write
and read-after-write guarantees cannot advertise atomic publication support.

The commit operation must bind the authorization check to the policy generation
it checked. A policy change racing the commit causes a retry or rejection;
rechecking permissions immediately before an unrelated catalog write is not
sufficient. The coordination mechanism is a ratification requirement.

### Concurrent writers and retries

If another writer advances the catalog first, a stale conditional write fails
without exposing any part of the losing batch. The server can rebuild against
the new generation only after revalidating the complete batch and its
authorization. A conflicting version fails the entire transaction.

Existing releases never change their bytes. An identical request retry may
return its earlier result, but a new transaction cannot quietly adopt another
publisher's competing release to make its batch appear successful.

Clients supply an idempotency key scoped to the organization and publisher
identity. Bind that key to a digest of the complete request. Reusing it with a
different request is a conflict. The request identity is scoped to a stable
principal, not a bearer-token string that changes during token rotation.

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

### Crash recovery and garbage collection

Before the reference advances, staged objects and candidate generations are
uncommitted and may eventually be collected. After it advances, publication is
committed even if the response is lost. Recovery inspects the durable catalog
and transaction state, rather than treating a local journal's disappearance as
proof of rollback.

Uncommitted candidates are protected while a transaction can still commit.
Garbage collection must coordinate with active transactions and retained
generations. Old generations remain reachable while a snapshot or retention
reference needs them. Deleting a tag never makes a retained artifact eligible
for accidental collection.

Recovery and idempotency information required to decide a committed outcome
must be in durable shared storage for replicated backends. A replica-local
journal may assist staging, but cannot be the only evidence of commit.

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

### Storage and performance

Do not copy every package record for every commit. Use a persistent structure
that shares unchanged data between generations and supports bounded lookup.
The structure and commit coordinator must be selected before ratification.

The catalog reference is a contention point. Initially scope it to an
organization and hosted catalog, reuse existing artifact storage, and benchmark
realistic concurrent publishers before expanding the transaction boundary.
Partitioning is acceptable only when it preserves the advertised boundary;
splitting a batch across shards without coordination is not atomic publication.

## Rationale and Alternatives

1. **Keep journaled per-package updates.** Provides useful recovery and remains
   the baseline, but cannot hide intermediate state from concurrent readers or
   prevent a late conflict from producing a partial batch.
2. **Use a distributed lock around package updates.** Serializes writers but
   does not prevent unlocked readers from seeing intermediate state. Making
   every registry read participate would add coordination to the hot path.
3. **Use a transactional database.** Can implement the required catalog and
   authorization commit semantics. It is a valid implementation option, but
   requires an explicit deployment decision; an object-store-only deployment
   still needs an equivalent safe commit design or must decline the capability.

## Implementation

Target pnpr, with new publishing and snapshot-read support in pnpm v12 only.

1. Define a catalog generation, conditional reference, transaction identity,
   and durable transaction state in `pnpr-storage`. Specify backend guarantees.
2. Route all hosted mutations through the same commit path. Reuse existing
   validation, artifact staging, conditional writes, and immutable revisions.
3. Make npm and other ecosystem views read the catalog. Remove any competing
   authoritative update path before advertising support for that ecosystem.
4. Add idempotent transaction status and capability-negotiated batch semantics.
5. Connect server-side resolution and retained snapshots to pinned generations.

Required validation includes concurrent publications of overlapping versions,
non-overlapping writers, conflicting tag updates, and policy revocation racing
commit. Inject crashes before and after each durability boundary, including a
lost successful response. Exercise a restarted replica with an empty local
cache and journal. Readers pinned to old and new generations must each see a
complete batch. Garbage collection must never delete an active transaction's
objects or a retained generation. Test existing clients separately from
snapshot-aware clients and report their actual consistency guarantees.

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

- What catalog structure and commit coordinator preserve conditional updates,
  policy-generation checks, and durable outcomes on each supported backend?
- Is one organization and hosted catalog the right initial transaction scope?
- How should clients name transactions, query outcomes, and pin read generations?
- How long should idempotency records and unreferenced generations survive?
- Should the first implementation cover npm only, or every ecosystem whose
  hosted mutation paths can share the catalog authority?
- What contention and storage measurements should gate making this the default
  hosted publication path?

---
Written by an agent (Codex, GPT-6).
