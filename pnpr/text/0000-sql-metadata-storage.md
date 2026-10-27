# SQL metadata storage for pnpr

> Status: draft for discussion. Split from
> [pnpm/rfcs#44](https://github.com/pnpm/rfcs/pull/44).

## Summary

Store pnpr's authoritative registry metadata in a transactional SQL database,
keeping archives and large evidence bundles in the filesystem or object
storage. Treat npm package documents as generated, cached views over immutable
per-version metadata and separately stored mutable state. Start with embedded
SQLite; add a shared backend only after it passes the same conformance suite.

## Motivation

An npm package document combines version manifests with tags, deprecations,
and other mutable package state. A tag change and a new release invalidate the
same document, although no accepted version's contents changed. Storing the
document as the authoritative record makes every mutation a read-modify-write
of the whole package.

pnpr also needs relationships that object storage does not coordinate well:
uniqueness of published versions, concurrent tag and policy changes, durable
publication outcomes after a lost response, and the snapshot and publication
features drafted in [pnpm/rfcs#45](https://github.com/pnpm/rfcs/pull/45) and
[pnpm/rfcs#46](https://github.com/pnpm/rfcs/pull/46).

## Detailed Explanation

### Storage split

| Data | Storage |
| --- | --- |
| Accepted version manifests and artifact references | SQL |
| Tags, selected revisions, deprecations, and policy bindings | SQL |
| Publication outcomes and idempotency records | SQL |
| Archives and large evidence bundles | Filesystem or object storage |
| Generated npm metadata JSON | Disposable cache |

Accepted version manifests are insert-only. Tags, revision selections,
deprecations, and access rules change independently and never update an
accepted manifest row.

### Domain operations

pnpr handlers and resolvers depend on domain operations such as accepting a
release, committing a publication, changing a tag, and reading a package view.
Each backend owns its queries, migrations, and transaction mechanics. Remote
SQL connections and vendor-specific types do not leak into registry handlers.

### Backend requirements

Every supported backend must provide:

- Atomic metadata transactions and enforced uniqueness of version bindings.
- Commit-time conflict detection for concurrent publication and policy changes.
  A transaction that reads an old permission and writes a different row is
  insufficient unless the backend detects or serializes that conflict.
- Durable, queryable publication outcomes after a lost response or restart.
- Defined read consistency, including a path for current authorization checks.
  Replica reads are usable only where their freshness meets the operation's
  requirements; asynchronous replica writes cannot acknowledge a publication.
- A consistent export/import procedure and the shared conformance tests.

All statements in one domain transaction share one database transaction
through a pinned connection or a documented session handle. Serialization
failures retry the whole transaction, preserving idempotency.

Embedded SQLite is the initial target for single-instance deployments.
PostgreSQL and Turso are candidates for shared deployments, subject to the
conformance suite and workload tests. SQL syntax alone does not establish
equivalent isolation, durability, or freshness.

### Relational keys sketch

`package_key` abbreviates `(registry, ecosystem, name)`. `revision_key` is
`ordinary` outside the revision protocol, or `rN` for an explicit registry
revision, including `r0`; it is an internal key, not new version syntax.

| Relation | Primary key | Notes |
| --- | --- | --- |
| `versions` | `(package_key, version, revision_key)` | Insert-only accepted manifest and artifact integrity. |
| `tags` | `(package_key, tag)` | Points to a version; changes are versioned. |
| `annotations` | `(package_key, version)` | Deprecations and other mutable annotations. |
| `publication_outcomes` | `(registry, principal, idempotency_key)` | Request integrity and committed result. |

All key columns are non-null, so a nullable revision column cannot weaken
duplicate-publication protection. Tables for snapshots, retention, and
workspace publication belong to their own RFCs and build on these operations.

### Generated views and caching

The npm full, abbreviated, and exact-version responses are views over accepted
versions and mutable state.

- Cache per-version rendered fragments keyed by version, representation,
  renderer version, and annotation generation. A tag change reuses all
  fragments; a new release renders one new fragment; a deprecation re-renders
  one fragment.
- Cache assembled full and abbreviated responses keyed by representation and
  committed package generation, plus any authorization context the existing
  cache contract requires. Coalesce concurrent misses.
- Give cached views bounded freshness and revalidate their generation on
  expiry or startup, so a lost invalidation cannot pin an old view forever.

Returning a full package document still costs its emitted bytes. This reduces
repeated database reads and rendering work; it does not make large documents
cheap to send.

### Backend replacement

Moving between supported backends requires a consistent logical export or a
quiesced transfer, verification of referenced blobs, and a controlled cutover.
Importing data must not grant access; operator-approved policy bindings are
re-established at the destination. pnpr is experimental, so alpha format
changes continue to be consolidated in place without migration fallbacks.

## Rationale and Alternatives

1. **Keep the package document authoritative.** Fewest changes, but immutable
   contents and mutable state keep sharing one mutation and cache boundary.
2. **Keep all metadata in object storage.** Preserves an object-store-only
   deployment, but needs custom coordination for uniqueness, tag changes, and
   publication outcomes. SQL makes those commit boundaries explicit at the cost
   of operating a database and coordinating its backup with the blob store.
3. **Start with a shared backend.** Supports multi-instance deployments sooner,
   but commits to a backend before workload measurements exist.

## Implementation

1. Split `pnpr-storage` metadata operations from artifact storage behind the
   domain interface.
2. Implement the interface on embedded SQLite. Retain existing artifact
   validation and staging.
3. Generate registry responses and historical revision views from SQL with
   fragment caching.
4. Write the backend conformance suite: concurrent publication and policy
   revocation, lost commit responses, crash recovery, replica lag, cache
   invalidation loss, and export/import.
5. Add a shared backend only after it passes that suite.

Measure cold and warm full/abbreviated generation, rows read, and rendering
work for packages with thousands of versions, and verify that a tag-only
update reuses unchanged fragments.

## Prior Art

- [SQLite deployment guidance](https://www.sqlite.org/whentouse.html).
- [PostgreSQL transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
  and [explicit locking](https://www.postgresql.org/docs/current/explicit-locking.html).
- [Turso's Rust backend options](https://docs.turso.tech/sdk/rust/reference).
- [npm package metadata](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md).

## Unresolved Questions and Bikeshedding

- What schema, indexes, and isolation strategy meet the domain contract on
  SQLite and the first shared backend?
- Which shared backend ships first, and in which deployment mode?
- What logical export format and coordinated metadata/blob backup process
  should operators use?
