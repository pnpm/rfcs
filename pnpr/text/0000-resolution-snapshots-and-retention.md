# Resolution snapshots and retained dependency sets for pnpr

> Status: draft for discussion. No implementation or protocol version is approved.

## Summary

Let pnpr resolve against an immutable snapshot of package observations and
produce a receipt that binds the resolution inputs, resolver version, policy,
and selected artifacts. An explicit retention operation makes the artifacts
needed by an approved dependency set durable for a declared period. Current
authorization and security policy still apply when that set is restored.

This builds on pnpr's server-side resolver and authorization-aware cache.
The companion immutable release records draft supplies verified package
records and the SQL metadata storage contract; the atomic publication draft
describes consistent hosted release visibility. Snapshot selection and retention
are separate operations.

## Motivation

A lockfile records selected packages and their integrities, but does not
preserve every candidate or policy input used to select them. Repeating a
dependency update can therefore produce another result after a tag moves,
an upstream publishes a version, or a cached document is refreshed.

For example, a team approves an update to `@example/dashboard`. A developer
and CI should be able to resolve that update against the same observed
candidate set. Later, an upstream deletion should not make the approved
lockfile unrestorable if the team explicitly retained its artifacts.

A disposable proxy cache cannot promise this: eviction is part of its contract.
The current hosted/cache storage distinction provides a starting point for
making retained upstream artifacts authoritative local data.

## Detailed Explanation

### Snapshot contents and scope

A snapshot is an immutable manifest of package observations. For every included
origin and package it binds:

- The candidate versions and their observed manifests or verified release records.
- The tag and selected-revision mappings used for resolution.
- Publication timestamps used by policy, upstream validators when available,
  and observation times.
- The origin's declared namespace and route identity.

A snapshot has an algorithm-qualified digest and a separately authorized
reference. Private snapshots and their dependency names are private data.

Store snapshot manifests, observations, receipts, and their memberships in the
authoritative SQL metadata database under the
[release-record storage contract](https://github.com/pnpm/rfcs/pull/44).
Membership references shared immutable records; creating a snapshot must not
copy an entire catalog or duplicate every member's canonical bytes. A SQL read
transaction can give hosted metadata reads a consistent view. A sealed snapshot
requires persisted membership to remain readable after that transaction ends.

The scope is finite and declared. Initial support should capture the packages
observed while resolving a request, rather than mirror every upstream version
of every package. One observation of each package is fixed on first use and
reused throughout that capture. The resulting candidate set is sealed only
after resolution succeeds. Subsequent requests can use it only if all metadata
they require is present.

For a snapshot-pinned request, a missing package observation, uncaptured direct
dependency, or unsupported resolver input is an explicit failure. It must not
silently consult live metadata. Extending a capture creates a new snapshot.
Missing metadata is different from an unsatisfied dependency range; report
which condition occurred without revealing unauthorized package names.

### What consistency means

An upstream snapshot is a capture of what pnpr observed, not a claim that all
upstream packages existed together at one wall-clock instant. Sequential
upstream reads cannot establish that stronger property without upstream support.

Hosted registries can expose one committed catalog generation. Once atomic
publication exists, capture uses that generation throughout the operation.
Until then, hosted capture has the same observed-state limitation and must say so.

Replaying a sealed snapshot uses the same immutable candidate observations.
It does not refresh a package because its TTL expired or silently switch to a
different registry. A changed source mapping requires a new snapshot or an
explicitly approved authority rebinding.

### Resolution receipt

The resolver produces an immutable receipt binding:

| Input | Purpose |
| --- | --- |
| Snapshot digest | Fixes candidate metadata and tag/revision selections. |
| Manifests and input lockfile | Fixes the requested workspace and update baseline. |
| Resolver and protocol versions | Fixes resolution semantics. |
| Overrides, peers, and resolution settings | Fixes graph-selection behavior. |
| Target environments | Fixes platform and engine-dependent selection. |
| Policy digest and evaluation time | Fixes the eligibility rules used for this decision. |
| Output lockfile digest and artifact references | Identifies the accepted result. |

Time-dependent policy, including release-age rules, uses an explicit evaluation
time. The server chooses and records it; an untrusted caller cannot backdate
an approval. Every semantic input used by the resolver must be included or the
request must be rejected as unsupported for replay.

The receipt can explain selected versions and rejected candidates using
structured reason codes. It must not serialize upstream credentials. Returned
reason details and private footprints remain subject to package authorization.
A receipt signature authenticates the server's decision, not a proof that an
arbitrary remote resolver executed correctly. Clients retain artifact integrity
verification, and the protocol must define which metadata bindings they verify.

Resolution replay with the recorded implementation and inputs should produce
the same graph, or a specific unsupported-version error. Byte-identical
lockfile serialization is a separate requirement that must be versioned if offered.

### Current policy remains current

Separate historical resolution from permission to install now:

- The receipt records why a dependency set was accepted at its decision time.
- Current access rules gate all snapshot, receipt, record, and artifact reads.
- Current security policy can refuse a retained artifact after a new advisory
  or screening result.

Such a refusal must identify the blocked dependency to authorized callers and
explain the relevant policy. It must not silently select another release or
rewrite the historical receipt. Any operator exception is explicit, authorized,
and audited. This composes with the existing package-screening proposal rather
than defining another verdict store.

### Explicit retention

An authorized retention request names an approved receipt or exact lockfile,
the required target environments, and a retention period. It is complete only
after pnpr has:

1. Enumerated all package resources required to restore that dependency set.
2. Downloaded and verified every resource not already durably stored.
3. Stored the records, metadata, and evidence needed to interpret them.
4. Committed retention references that keep these objects out of cache eviction
   and garbage collection for the declared period.

Return a completeness report. Inaccessible, unsupported, or unverifiable
dependencies prevent a complete status; a partially filled cache is not success.
Direct git, URL, workspace, or local-file dependencies require explicit handling.
Initial support may cover registry artifacts only, but cannot call a set
complete while skipping unsupported external sources. Workspace source files
remain the build system's responsibility and are listed as such in the report.

Retention preserves package inputs, not arbitrary network downloads performed
by lifecycle scripts or all compiler and operating-system inputs. It promises
restorability of the declared package resources, not a reproducible build.

Reference changes, expiry, and garbage collection must be race-safe. A retained
object cannot be collected between verification and committing its reference.
Restore checks current authorization even when another organization retains
identical bytes. Physical deduplication does not share retention permissions.

Download, verify, and durably stage artifacts outside SQL write transactions.
Protect staging from garbage collection while acceptance is pending. Commit
retention membership and completeness in one metadata transaction only after
every required object is durable. Object storage does not participate in that
SQL transaction: a failed commit leaves protected staging for retry or later
cleanup, and never a complete set referencing an unfinished upload. Retention
expiry and collection coordinate through the same authoritative references.

Security blocks do not automatically destroy retained bytes. Deletion for a
legal, privacy, or explicit administrative requirement can override retention;
record the resulting incomplete state and report it on restore. No retention
policy can make an unconditional promise against privileged deletion or storage loss.

### Client and protocol behavior

Add capability-negotiated snapshot, receipt, and retention operations to the
existing pnpr surface. Endpoint names and configuration syntax remain open.
A client requesting these guarantees must fail when they are unsupported;
falling back to live resolution would violate its request.

Frozen installs keep the existing inexpensive local path when they have the
required artifacts and current policy verification. Snapshot support must not
introduce a metadata request per package into every frozen install. A client
that explicitly requires a fresh server verdict pays for that requested check.

Existing npm clients can install through compatibility registry views, but
ordinary live endpoints do not acquire snapshot semantics automatically. A
snapshot-specific base URL is a possible adapter only if every metadata and
artifact path stays in that context and current authorization is enforced.

### Cache and performance

Extend the existing cache key with snapshot and semantic-input digests. Retain
private-footprint access checks on hits. Neither a public-looking snapshot ID
nor a guessed receipt digest can unlock a private cached resolution.

Store snapshot metadata by reference and reuse immutable records. Bound the
package count, metadata bytes, capture duration, retained bytes, and active
capture count per organization. Snapshot capture and retention are explicit
work; they must not turn a normal install into a full upstream mirror.

Batch indexed SQL reads for package observations and snapshot membership.
Serving immutable records can use caches or replicas; completeness decisions,
current authorization checks, and reference deletion require the authoritative
consistency defined by the storage contract. Ordinary installs must not create
a new durable snapshot or retention transaction unless explicitly requested.

Measure initial capture, replay, retention transfer, metadata storage, and warm
frozen restores separately. No performance gain is assumed by this draft.

## Rationale and Alternatives

1. **Rely on lockfiles and an ordinary proxy cache.** Covers selected versions
   while the cache happens to retain them, but not candidate replay or durable
   availability. Keep this as the ordinary low-cost mode.
2. **Resolve as of a timestamp.** Convenient syntax, but publication times do
   not reconstruct historical tags, revisions, deletions, or observations. A
   captured manifest makes the available evidence explicit.
3. **Mirror an entire registry.** Provides broader historical coverage at a
   much higher storage and operational cost. Finite captures meet the project
   use case without requiring an ecosystem-wide mirror.

## Implementation

Target pnpr and capability-negotiated support in pnpm v12 only.

1. Add a resolver metadata source that captures and seals observations or reads
   a sealed snapshot without live fall-through. Reuse the existing resolver.
2. Define the versioned receipt and replay-input contract. Preserve the existing
   authorization-aware cache classification and gates.
3. Store snapshots, receipts, memberships, and retention references in the shared
   SQL metadata layer, with organization authorization and limits. Keep artifact
   retention separate from snapshot metadata retention.
4. Add verified artifact promotion from disposable cache into retained storage,
   with race-safe retention references and completeness reporting.
5. Expose negotiated operations and update pnpr and pnpm documentation.

Validation must demonstrate replay after tag, version, revision, and TTL changes;
missing-observation failures without network fallback; resolver-version and
policy-time binding; access revocation on cache hits; and concurrent capture,
retention, expiry, and garbage collection. Test restore with the upstream
unavailable and the disposable cache removed. A policy block must preserve the
receipt, and an administrative deletion must invalidate completeness. Run the
same backend conformance cases for concurrent retain/expire/collect operations,
lagging replicas, and database restart. A backend export/import must preserve
snapshot and receipt digests, membership, expiry, and verified blob availability.

## Prior Art

- [Authorization-aware resolution cache](./0000-auth-aware-resolution-cache.md).
- [Package screening proposal](https://github.com/pnpm/rfcs/pull/15).
- [Existing install acceleration](https://github.com/pnpm/pnpm/blob/7535582cd7c42df60f61ea6edaaf3cc49d861316/pnpr/docs/install-acceleration.md).
- [Hosted and disposable storage](https://github.com/pnpm/pnpm/blob/7535582cd7c42df60f61ea6edaaf3cc49d861316/pnpr/docs/storage.md).
- [npm unpublishing behavior](https://docs.npmjs.com/unpublishing-packages-from-the-registry/).

## Unresolved Questions and Bikeshedding

- What names and configuration expose snapshot capture, replay, and retention
  without adding overlapping install modes?
- What is the minimum replay-input schema, and how long should pnpr retain
  compatible resolver implementations?
- Should initial snapshots require verified release records, or allow clearly
  marked upstream metadata observations until retention verifies them?
- Which target environments and external dependency sources are required for
  an initial complete retention set?
- How should signed receipts, authority rebinding, and retention references be
  exported between deployments without exporting access rights?
- What quotas and retention defaults make the storage obligation predictable?
- What membership indexes and retention/collection coordination meet the shared
  SQL backend contract without catalog-wide copies or long write transactions?

---
Written by an agent (Codex, GPT-6).
