# Split packuments: keep the versions nobody picks out of large package documents

## Summary

pnpr serves a large package document in a split form to clients that ask for
it. The versions a fresh resolution almost always picks stay whole in
`versions`. Every other version is a short entry in `pnpr.externalVersions`:
its publish time and, where it has any, its deprecation notice, trust rank,
and selected revision, which is what picking a version reads and what may
change after publication. The rest of the version is its release record, an
immutable document at `<registry-base>/<name>/<version>+r<revision>`, which
the client fetches only for a version it picks. Every response stays a plain
GET that a CDN caches for everyone. Small documents, and clients that do not
opt in, get today's abbreviated document unchanged.

## Motivation

A resolution reads the dependencies and the hash of only the version it
picks, usually one per package, yet it downloads those of every version ever
published. The cost grows with the age of the package, not with what the
install uses, and it falls on every cold resolution: in CI, in a fresh
container, and on every `pnpm add` whose metadata has expired.

Most of a gzipped packument is what does not repeat from version to version.
Per-version digests and npm registry signatures do not compress. Neither do
dependencies pinned to the package's own version, as monorepos with
lockstep releases do (next pins `@next/env` and every `@next/swc-*` package
to its exact version, and swc and rollup pin their platform packages), so
each version's dependency list is new. I measured npm's abbreviated documents
of 15 popular packages in the form pnpr serves them (without `npm-signature`,
and without `shasum` when `integrity` is present):

| Package | gzip today | Without per-version `dist` | Without whole versions |
| --- | --- | --- | --- |
| next | 1040 KB | 379 KB | 21 KB |
| typescript | 647 KB | 66 KB | 12 KB |
| react | 473 KB | 33 KB | 20 KB |
| `@types/node` | 366 KB | 16 KB | 8 KB |
| rollup | 199 KB | 49 KB | 3 KB |
| `@swc/core` | 190 KB | 46 KB | 5 KB |
| esbuild | 90 KB | 18 KB | 2 KB |
| All 15 | 4061 KB | 742 KB | 93 KB |

The last column keeps every version's name, version, `deprecated` notice,
and provenance, and moves every version; the split form below keeps a few
versions whole and less of the others (see
[The split document](#the-split-document)). Moving only `dist` leaves next at
379 KB, almost all of it dependency lists. npm's abbreviated form has no
`time` map; pnpr's has one, for the minimum release age, and it stays in the
split form too (see [Prototype](#prototype)).

[Platform variants](https://github.com/pnpm/rfcs/pull/51) make it worse. Each
variant archive brings a digest of its own, so `esbuild`'s document grows from
90 KB to 361 KB gzipped and `@swc/core`'s from 190 KB to 764 KB. The tags and
platforms of the variants repeat and cost almost nothing; the digests are
nearly all of the growth.

## Detailed Explanation

### Opting in

A client asks for the split form with its own media type, ahead of the
abbreviated one:

```
Accept: application/vnd.pnpr.install-v2+json; q=1.0, application/vnd.npm.install-v1+json; q=0.9, application/json; q=0.8, */*
```

pnpr answers with `Content-Type: application/vnd.pnpr.install-v2+json` and
`Vary: Accept`. npm, Yarn, Bun, and older pnpm releases do not send the type
and get today's abbreviated or full document, so nothing changes for them.

A document served under the new type may be whole. A client reads
`pnpr.externalVersions`, not the media type, to know whether some versions
are elsewhere.

### The split document

When the abbreviated document is larger than the threshold (see
[Threshold](#threshold)), only the likely picks stay in `versions` and
`time`, and every other version moves to `pnpr.externalVersions`:

```json
{
  "name": "fast-thing",
  "dist-tags": { "latest": "2.1.0" },
  "modified": "2026-10-01T10:00:00.000Z",
  "time": { "1.9.3": "2026-02-11T09:12Z", "2.1.0": "2026-10-01T10:00Z" },
  "versions": {
    "1.9.3": { "name": "fast-thing", "version": "1.9.3", "...": "..." },
    "2.1.0": {
      "name": "fast-thing",
      "version": "2.1.0",
      "dependencies": { "detect-libc": "^2.0.0" },
      "dist": { "integrity": "sha512-<primary tar digest>", "pnpr": { "variants": ["..."] } }
    }
  },
  "pnpr": {
    "externalVersions": {
      "1.4.0": "2025-03-02T10:00Z",
      "1.4.1": ["2025-03-09T16:40Z", { "deprecated": "1.x is no longer maintained" }],
      "1.5.0": ["2025-05-20T08:00Z", { "trust": "provenance", "revision": 2 }]
    }
  }
}
```

An entry is the version's publish time, or `[time, flags]` when it has any
of these flags:

- `deprecated`: the deprecation message. pnpm skips a deprecated version when
  a range also matches one that is not, and shows the message when it does
  install one. A deprecation can be added or changed after publication, so it
  lives here and never in the release record.
- `trust`: the strongest trust evidence the version has, as the trust policy
  ranks it: `provenance`, `trustedPublisher`, or `stagedPublish`. A trust
  policy such as
  [`trustPolicy: no-downgrade`](https://pnpm.io/settings#trustpolicy)
  compares how versions were published before it picks one, and it reads
  only this rank.
- `revision`: the selected
  [registry revision](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-integrity-addressed-patch-revisions.md),
  when it is not the original (`0`). Selecting a patched revision changes
  which release record the version is.

The publish time is there because a minimum release age and time-based
resolution filter versions by it before they pick one. It is `null` when the
registry has no publish time for the version.

Everything else, dependencies, engines, platform fields, bins, and the whole
`dist` with its integrity and platform variants, is in the release record.
All of it is read only after a version is picked, and a picked external
version needs its record for the integrity anyway, so moving it costs no
extra request.

I measured the external entries of 16 popular packages in other shapes,
gzipped. Entries kept inside `versions` with `name`, `version`, the
`_npmUser` flags, `dist.attestations`, and a per-version marker took 157 KB.
The flags alone, in a map of their own, took 68 KB. Dropping the deprecation
text in favor of a flag would save another 1 KB, since messages repeat across
versions. Merging the entries with the publish times, so each version string
appears once, matters more: for 8 of those packages the publish times alone
take 83.8 KB, the publish times and a separate map of entries 126.1 KB, and
the merged map 87.6 KB.

### Why a map of its own

The external versions are not in `versions`, so a client that does not know
the split form, such as an older pnpm sharing a cache or a tool sent the
media type, sees only complete versions. A version in `versions` without its
dependencies would look valid to such a client, which would write a lockfile
without that package's dependencies. With a map of its own, the worst such a
client does is not find a version, or, when a range matches an older version
held by a dist-tag and a newer external one, pick the tagged one.

A missing `integrity` keeps its meaning too: a version whose integrity the
client computes while it fetches the tarball, as pnpm does today for
registries that omit it.

### The release record

A version's release record is served at
`<registry-base>/<name>/<version>+r<revision>`
(`<registry-base>/@scope%2fname/<version>+r<revision>` for a scoped name),
next to the tarball's `<name>@<version>+r<revision>` route. It is the
abbreviated version object of that revision without what may change after
publication or what [immutable release records](https://github.com/pnpm/rfcs/pull/44)
keep as separate evidence: no `deprecated`, no `_npmUser`, and no
`dist.signatures` or `dist.attestations`. What remains never changes, so
pnpr serves it with `Cache-Control: public, max-age=31536000, immutable`.

The address carries the revision because `<name>/<version>` does not name
fixed content: it is the registry's currently selected revision, and
selecting a patched revision changes its integrity and can change its
dependencies. That route keeps its meaning and its usual caching. Addressing
the record by a hash of its content would also be immutable, but it would put
an incompressible digest per version back into the package document.

A client checks that the record names the version it picked, and takes the
deprecation and trust evidence from the package document.

### Likely picks

Every dist-tag target and the highest release of each major (of each minor
below 1.0, where a caret range stays within one minor) stay whole. A fresh
resolution almost always picks one of these: `^1.2.3` picks
the highest 1.x, and `latest` picks its target. A lockfile update picks them
too. An older version is picked only by a narrower range, by an `overrides`
pin, or under
[time-based resolution](https://pnpm.io/settings#resolutionmode), and those
cost one extra GET each.

When the prototype first split every version, a resolution of the benchmark
app fetched 180 version documents and was slower than whole documents on
localhost (3.1 s against 1.8 s), because pnpm waited for a version document
before it resolved that package's dependencies. With the likely picks whole,
it fetches 22 release records.

The rule depends only on the document, so every pnpr instance and every CDN
node serves the same bytes for the same document.

### Threshold

A document is split when its abbreviated form is larger than a threshold.
Below it, splitting saves little and would cost a request for every package
whose resolution picks an older version. The prototype used 64 KiB of
serialized JSON.

Once a package's document crosses the threshold, it stays split, even if it
shrinks below it again (after an unpublish, for example). A document that
flips between the two forms near the threshold would change shape for clients
on every publish, for no saving. pnpr records the decision with the package.

### Client behavior

A client that opts in:

1. Reads `pnpr.externalVersions` into its list of versions, with their
   publish times, deprecations, and trust ranks, and picks a version as
   today.
2. If the picked version is external, fetches its release record, once per
   URL per install, sharing an in-flight request between the resolutions that
   need it, and reads the version from it, dependencies included.
3. Writes the lockfile exactly as today. The lockfile records the integrity,
   so a frozen install reads no metadata at all and never sees the split.

Release records never change, so the client caches them on disk with no
expiry, keyed by registry, name, version, and revision.

A client keeps split documents in a metadata cache of its own, apart from the
abbreviated documents it cached before. An older release that shares the
cache directory would otherwise resolve from a document without most of its
versions. The prototype hit an earlier form of this: an older pnpm on
`PATH`, started by the version switch, read a split document the new pnpm had
cached and failed.

### CDN caching

Every response is a GET addressed by its URL and the `Accept` header. There
are no range queries, `?since=` cursors, or POST lookups, because each of
those makes a cache key of its own, or none. The split document is cached for
everyone who sends the new type, and a release record for everyone who picks
that version.

A CDN keys the package document by `Accept` through `Vary: Accept`. CDNs that
do not honor `Vary` need the `Accept` header normalized into the cache key, or
a URL of its own for the split form; see
[Unresolved Questions](#unresolved-questions-and-bikeshedding).

### Relation to other RFCs

- [Immutable release records](https://github.com/pnpm/rfcs/pull/44) store
  each release as an immutable record and keep tags, revision selections,
  and deprecations as mutable state beside it. The split form exposes that
  division to clients: the release record is the record, rendered, and the
  external entries are the mutable state.
- [SQL metadata storage](https://github.com/pnpm/rfcs/pull/48): a release
  record is one of its per-version fragments, rendered, and the split
  document is its package view with the large fragments cut down. Its open
  question, whether fragments become a client-facing read path, is answered
  here with a narrow yes.
- [Platform variants](https://github.com/pnpm/rfcs/pull/51) relies on this RFC
  to keep variant archives out of large documents. Variant selection happens
  after a version is picked, so the variants move with the rest of the
  version.
- [Release records and upstream verification](https://github.com/pnpm/rfcs/pull/49):
  the signed record and the served one describe the same release; the
  served one is the cacheable read path.
- [Server-side resolution](https://github.com/pnpm/rfcs/pull/9) does not need
  this: the server reads its own storage, and the client receives a lockfile.

## Prototype

Implemented in [pnpm/pnpm#17522](https://github.com/pnpm/pnpm/pull/17522), in
pnpr's `split_packuments.rs` and pnpm's `version_document.rs`. The benchmark
resolves an app of 787 packages (next, vite, vitest, eslint, typescript,
storybook, and others) proxied from npmjs. It runs through a proxy that gzips
and caches like a CDN, adds a round trip per request, and caps bandwidth, and
every run starts with an empty pnpm cache and store. The registry is declared
with `serverType: pnpr`, so pnpm reads optional dependencies from abbreviated
documents too. Every split form keeps the likely picks whole. Medians of 3
runs:

| Network | Whole | Only `dist` moved | Versions marked in `versions` | External map |
| --- | --- | --- | --- | --- |
| 10 ms, 200 Mbit/s, resolve | 1.34 s | 0.46 s | 0.43 s | 0.37 s |
| 40 ms, 50 Mbit/s, resolve | 5.06 s | 1.51 s | 1.41 s | 1.13 s |
| 100 ms, 10 Mbit/s, resolve | 24.70 s | 6.73 s | 5.86 s | 4.80 s |
| 40 ms, 50 Mbit/s, full install | 28.44 s | 24.82 s | 24.65 s | 24.43 s |
| Packument bytes (gzip) | 30.3 MB | 7.6 MB | 6.6 MB | 5.3 MB |
| Release records | 0 | 22 (7 KB) | 22 (12 KB) | 22 (7 KB) |

The third column, an earlier draft that kept external versions in
`versions` with a marker, comes from a separate run of the same benchmark.
With the external map, resolution is 72% to 81% faster than with whole
documents, and a full install 14% faster, as tarballs dominate it. Every
form produces the same lockfile.

About 3 MB of the remaining 5.3 MB is in documents under the threshold,
which are not split.

The prototype found the cache-sharing problem described in
[Client behavior](#client-behavior), and that the version a version switch
picks bypassed the npm resolver, so the release record is now fetched in the
one function every pick goes through.

## Rationale and Alternatives

1. **Filter the document on the server**, by range, by platform, or by
   `?since=`. Each distinct query is its own cache key, so a CDN caches almost
   nothing, and every cold resolution reaches the origin. Split documents have
   one key per package and one per version, shared by every client.
2. **Always split, or serve only per-version documents.** Go's module proxy
   and RubyGems' compact index list versions apart from their details. For
   npm-sized dependency graphs, a request per package per picked version adds
   a round trip to most resolutions, and a small document saves nothing by
   splitting. A threshold and inline likely picks keep the common case at one
   request.
3. **Better compression**, such as a zstd dictionary shared across documents.
   It shrinks the repeated parts, which already compress well, but cannot
   shrink a random digest.
4. **Move only each version's `dist`**, keeping dependencies, engines, and
   variant platforms inline. An earlier draft did this, so that a resolver
   could walk a picked version's dependencies while its integrity was still
   on the way. But the external versions are rare picks, and dependency
   lists pinned to the package's own version do not compress: next stays at
   379 KB gzipped, against 21 KB with whole versions moved. See
   [Prototype](#prototype) for both forms measured.
5. **Keep external versions in `versions` with a marker** such as
   `dist.pnpr.details: "external"`, as an earlier draft did. A client that
   does not read the marker would take such a version for one without
   dependencies; see [Why a map of its own](#why-a-map-of-its-own). Marking
   them by a missing `integrity` is worse still, since that is already a
   valid state with another meaning.
6. **Incremental updates of the document**, sending only what changed since the
   client's copy. This is complementary and stays cacheable only with
   Cargo-style conditional GETs of the whole document, or with append-only
   segments where every closed segment is immutable. It helps warm caches; this
   RFC helps cold ones.

## Implementation

1. pnpr: serve the split form under the new media type with `Vary: Accept`,
   with the likely-picks rule and a configurable threshold, and record that a
   package is split.
2. pnpr: serve release records at `<name>/<version>+r<revision>` for every
   accepted revision, with immutable caching.
3. pnpm: send the new media type, read `pnpr.externalVersions`, fetch the
   release records of picked external versions, cache them on disk, and keep
   split documents in a metadata cache directory of their own.
4. pnpm: when the user points pnpm at a registry that does not serve the type,
   nothing changes, since it answers with the abbreviated document.

Tests must cover an external version with and without flags, a version
deprecated after its record was cached (the new message shows), a selected
patch revision (its record is fetched, not the original's), a version without
`integrity` in `versions` (computed while fetching the tarball, as today), a
pick of a likely pick (no second request), a pick under `overrides` and under
time-based resolution (one request), a trust downgrade detected across
external versions, an older pnpm sharing the home directory, and a frozen
install that reads no metadata.

## Prior Art

- [Go module proxy](https://go.dev/ref/mod#goproxy-protocol): `@v/list` lists
  versions, and `@v/<version>.info`, `.mod`, and `.zip` are separate,
  immutable files. The checksums are in a separate database.
- [RubyGems compact index](https://guides.rubygems.org/rubygems-org-compact-index-api/):
  a `versions` file appended to over time, and an `info/<gem>` file per gem.
- [Cargo sparse index](https://doc.rust-lang.org/cargo/reference/registry-index.html#sparse-protocol):
  one file per crate, a line per version with its checksum, fetched over plain
  HTTP and refreshed with conditional GETs. Crate files are small because the
  index holds only dependencies and checksums.
- npm's [abbreviated metadata](https://github.com/npm/registry/blob/main/docs/responses/package-metadata.md#abbreviated-metadata-format)
  is the same idea one step earlier: it dropped the readme and the full
  manifests for install clients, opted into by media type.
- PyPI's [JSON simple API](https://peps.python.org/pep-0691/) lists every file
  with its hash, and [PEP 658](https://peps.python.org/pep-0658/) moved each
  file's metadata to a file of its own.

## Unresolved Questions and Bikeshedding

- The threshold: it should be chosen from the size distribution of the
  registry, and measured on gzipped size rather than serialized JSON.
- The `time` map is now a large part of a split document. A minimum release
  age needs only whether a version is older than the cutoff, so coarser
  timestamps, or none for versions older than any reasonable cutoff, would
  shrink it.
- A media type or a URL of its own (`/-/v2/<name>`, for example)? A URL needs
  no `Vary` and works with every CDN, but forks the routes; the media type
  keeps one URL per package.
- Should the full document (`application/json`) be split too? Install clients
  do not read it, so the prototype left it alone.
- Should more versions stay whole, such as the highest release of each
  minor, to save the request for `~` ranges?
- The route of a release record. `<name>/<version>+r<revision>` mirrors the
  tarball route, but `+r<revision>` could be mistaken for build metadata of a
  version that has some.
- The media type's name.
