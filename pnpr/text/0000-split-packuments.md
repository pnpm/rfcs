# Split packuments: keep the versions nobody picks out of large package documents

## Summary

pnpr serves a large package document in a split form to clients that ask for
it. Each version keeps only what picking a version reads: its `deprecated`
notice, its trust flags, and its provenance. The rest of the version
(dependencies, engines, bins, integrity, signatures, variant archives) moves
to the version's immutable document `<registry-base>/<name>/<version>`, and
the version is marked `dist.pnpr.details: "external"`. The versions a fresh
resolution almost always picks stay whole, so most packages still resolve
with one request. Every response stays a plain GET that a CDN caches for
everyone. Small documents, and clients that do not opt in, get
today's abbreviated document unchanged.

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
and provenance, as the split form below does, though it moves every version,
where the split form keeps a few whole. Moving only `dist` leaves next at
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

A document served under the new type may be whole. A client reads the marker
on each version, never the media type, to know whether a version is all
there.

### The split document

When the abbreviated document is larger than the threshold (see
[Threshold](#threshold)), each version that is not a likely pick keeps only
what picking a version reads:

```json
{
  "name": "fast-thing",
  "dist-tags": { "latest": "2.1.0" },
  "modified": "2026-10-01T10:00:00.000Z",
  "time": { "1.4.0": "2025-03-02T00:00:00.000Z", "2.1.0": "2026-10-01T00:00:00.000Z" },
  "versions": {
    "1.4.0": {
      "name": "fast-thing",
      "version": "1.4.0",
      "deprecated": "1.x is no longer maintained",
      "_npmUser": { "trustedPublisher": true },
      "dist": {
        "attestations": {
          "url": "https://registry.example/-/npm/v1/attestations/fast-thing@1.4.0",
          "provenance": { "predicateType": "https://slsa.dev/provenance/v1" }
        },
        "pnpr": { "details": "external" }
      }
    },
    "2.1.0": {
      "name": "fast-thing",
      "version": "2.1.0",
      "dependencies": { "detect-libc": "^2.0.0" },
      "engines": { "node": ">=18" },
      "dist": {
        "integrity": "sha512-<primary tar digest>",
        "pnpr": { "variants": ["..."] }
      }
    }
  }
}
```

An external version keeps:

- `name` and `version`.
- `deprecated`, because pnpm skips a deprecated version when a range also
  matches one that is not.
- `_npmUser.trustedPublisher` and `dist.attestations`, because a trust policy
  such as [`trustPolicy: no-downgrade`](https://pnpm.io/settings#trustpolicy)
  compares how versions were published before it picks one. The attestations
  are a URL and a predicate type, which compress well.
- `dist.pnpr.details: "external"`, the marker.

The top-level `time` map stays too, because a minimum release age and
time-based resolution filter versions by it.

Everything else is in the version document: `dependencies`,
`peerDependencies`, `optionalDependencies`, `engines`, `os`, `cpu`, `libc`,
`bin`, `hasInstallScript`, the whole `dist` with its integrity, signatures,
and bins, and each platform variant with its tag, platforms, and archive.
All of them are read only after a version is picked, and a picked external
version needs its version document for the integrity anyway, so moving them
costs no extra request.

### The marker

`dist.pnpr.details: "external"` is the only signal that a version is
elsewhere. A missing `integrity` cannot be the signal: it already means
something, a version whose integrity the client computes while it fetches the
tarball, as pnpm does today for registries that omit it. A client that saw a
missing `integrity` would not know whether to fetch the version document or
the tarball.

The marker is per version, so one document can split some of its versions and
not others. That is what keeps the likely picks inline, and it lets a registry
split by any rule without a new media type.

### The version document

The version document is the existing per-version route,
`<registry-base>/<name>/<version>` (`<registry-base>/@scope%2fname/<version>`
for a scoped name). npm serves the same route today, so a client needs no new
URL scheme. Asked for with the new media type, pnpr serves the version as the
abbreviated document would have listed it, with `Vary: Accept`; asked for
with `application/json`, it serves the full manifest as today.

For an exact version, pnpr serves it with
`Cache-Control: public, max-age=31536000, immutable`. A published version's
facts never change: a [revision](https://github.com/pnpm/rfcs/pull/42) has a
route of its own, and an unpublish removes the version from the package
document, so no client asks for it again. The route with a dist-tag in place
of a version moves, and keeps its usual caching.

A client checks that the version document names the version it picked.

### Likely picks

Every dist-tag target and the highest release of each major (of each minor
below 1.0, where a caret range stays within one minor) stay whole. A fresh resolution almost always picks one of these: `^1.2.3` picks
the highest 1.x, and `latest` picks its target. A lockfile update picks them
too. An older version is picked only by a narrower range, by an `overrides`
pin, or under
[time-based resolution](https://pnpm.io/settings#resolutionmode), and those
cost one extra GET each.

When the prototype first split every version, a resolution of the benchmark
app fetched 180 version documents and was slower than whole documents on
localhost (3.1 s against 1.8 s), because pnpm waited for a version document
before it resolved that package's dependencies. With the likely picks inline,
it fetches 22.

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

1. Picks a version from the document as today.
2. If the picked version is marked external, fetches its version document,
   once per URL per install, sharing an in-flight request between the
   resolutions that need it, and reads the version from it, dependencies
   included.
3. Writes the lockfile exactly as today. The lockfile records the integrity,
   so a frozen install reads no metadata at all and never sees the split.

Version documents never change, so the client caches them on disk with no
expiry, keyed by registry, name, and version.

A client keeps split documents in a metadata cache of its own, apart from the
abbreviated documents it cached before. An older release that shares the
cache directory must never read a version whose dependencies and integrity
it cannot find. The prototype hit exactly that: an older pnpm on `PATH`, started by the version
switch, read a split document the new pnpm had cached and failed.

### CDN caching

Every response is a GET addressed by its URL and the `Accept` header. There
are no range queries, `?since=` cursors, or POST lookups, because each of
those makes a cache key of its own, or none. The split document is cached for
everyone who sends the new type, and a version document for everyone who picks
that version.

A CDN keys the package document by `Accept` through `Vary: Accept`. CDNs that
do not honor `Vary` need the `Accept` header normalized into the cache key, or
a URL of its own for the split form; see
[Unresolved Questions](#unresolved-questions-and-bikeshedding).

### Relation to other RFCs

- [SQL metadata storage](https://github.com/pnpm/rfcs/pull/48): a version
  document is one of its per-version fragments, rendered, and the split
  document is its package view with the large fragments cut down. Its open
  question, whether fragments become a client-facing read path, is answered
  here with a narrow yes.
- [Platform variants](https://github.com/pnpm/rfcs/pull/51) relies on this RFC
  to keep variant archives out of large documents. Variant selection happens
  after a version is picked, so the variants move with the rest of the
  version.
- [Release records](https://github.com/pnpm/rfcs/pull/49): a version document
  and a release record both describe one version; the record stays the signed
  statement, and the version document the cacheable read path.
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
documents too. Both split forms keep the likely picks inline. Medians of 3
runs:

| Network | Whole | Split, `dist` only | Split, whole versions |
| --- | --- | --- | --- |
| 10 ms, 200 Mbit/s, resolve | 1.32 s | 0.47 s | 0.43 s |
| 40 ms, 50 Mbit/s, resolve | 5.12 s | 1.54 s | 1.41 s |
| 100 ms, 10 Mbit/s, resolve | 24.64 s | 6.68 s | 5.86 s |
| 40 ms, 50 Mbit/s, full install | 28.45 s | 24.86 s | 24.65 s |
| Packument bytes (gzip) | 30.3 MB | 7.6 MB | 6.6 MB |
| Version documents | 0 | 22 (12 KB) | 22 (12 KB) |

With whole versions moved, resolution is 67% to 76% faster, and a full
install 13% faster, as tarballs dominate it. The three lockfiles are
identical.

Moving whole versions saves less here than in the 15 packages measured in
[Motivation](#motivation), because of what the split leaves alone: about
half of the remaining 6.6 MB is in documents under the threshold, and 1.6 MB
is the `time` maps.

The prototype found the cache-sharing problem described in
[Client behavior](#client-behavior), and that the version a version switch
picks bypassed the npm resolver, so the version document is now fetched in the
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
5. **Mark external versions by a missing `integrity`.** It is already a valid
   state with another meaning; see [The marker](#the-marker).
6. **Incremental updates of the document**, sending only what changed since the
   client's copy. This is complementary and stays cacheable only with
   Cargo-style conditional GETs of the whole document, or with append-only
   segments where every closed segment is immutable. It helps warm caches; this
   RFC helps cold ones.

## Implementation

1. pnpr: serve the split form under the new media type with `Vary: Accept`,
   with the likely-picks rule and a configurable threshold, and record that a
   package is split.
2. pnpr: serve exact version documents with immutable caching.
3. pnpm: send the new media type, read the marker, fetch version documents for
   picked external versions, cache them on disk, and keep split documents in a
   metadata cache directory of their own.
4. pnpm: when the user points pnpm at a registry that does not serve the type,
   nothing changes, since it answers with the abbreviated document.

Tests must cover a version marked external with and without `integrity` in
the version document, a version without `integrity` and without the marker
(computed while fetching the tarball, as today), a document that splits only
some versions, a pick of a likely pick (no second request), a pick under
`overrides` and under time-based resolution (one request), an older pnpm
sharing the home directory, and a frozen install that reads no metadata.

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
- Should more versions stay inline, such as the highest release of each
  minor, to save the request for `~` ranges?
- The media type's name.
