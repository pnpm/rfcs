# Split packuments: keep per-version hashes out of large package documents

## Summary

pnpr serves a large package document in a split form to clients that ask for
it. Each version keeps everything resolution reads to pick it, but its `dist`
facts (integrity, sizes, bins, signatures, variant archives) move to the
version's immutable document `<registry-base>/<name>/<version>`, and the
version is marked `dist.pnpr.details: "external"`. The versions a fresh
resolution almost always picks keep their `dist` inline, so most packages
still resolve with one request. Every response stays a plain GET that a CDN
caches for everyone. Small documents, and clients that do not opt in, get
today's abbreviated document unchanged.

## Motivation

Most of a gzipped packument is hashes. Per-version digests and npm registry
signatures do not compress, while names, version ranges, and the rest repeat
from version to version and compress well. I measured npm's abbreviated
documents of 44 popular packages in the form pnpr serves them (without
`npm-signature`, and without `shasum` when `integrity` is present). Leaving
each version's `dist` out cuts the gzipped total from 7.9 MB to 1.3 MB, 83%
less:

| Package | gzip today | gzip without per-version `dist` |
| --- | --- | --- |
| next | 1040 KB | 391 KB |
| typescript | 647 KB | 67 KB |
| react | 473 KB | 33 KB |
| `@types/node` | 366 KB | 16 KB |
| esbuild | 90 KB | 21 KB |

A resolution needs the hash of only the version it picks, usually one per
package, yet it downloads the hash of every version ever published. The cost
grows with the age of the package, not with what the install uses, and it
falls on every cold resolution: in CI, in a fresh container, and on every
`pnpm add` whose metadata has expired.

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
on each version, never the media type, to know whether a version's `dist` is
there.

### The split document

When the abbreviated document is larger than the threshold (see
[Threshold](#threshold)), each version that is not a likely pick keeps every
field of its abbreviated manifest except `dist`:

```json
{
  "name": "fast-thing",
  "dist-tags": { "latest": "2.1.0" },
  "modified": "2026-10-01T10:00:00.000Z",
  "versions": {
    "1.4.0": {
      "name": "fast-thing",
      "version": "1.4.0",
      "dependencies": { "detect-libc": "^2.0.0" },
      "engines": { "node": ">=18" },
      "dist": {
        "attestations": {
          "url": "https://registry.example/-/npm/v1/attestations/fast-thing@1.4.0",
          "provenance": { "predicateType": "https://slsa.dev/provenance/v1" }
        },
        "pnpr": {
          "details": "external",
          "variants": [
            { "tag": "linux-x64-gnu", "platforms": [{ "os": "linux", "cpu": "x64", "libc": "glibc" }] },
            { "tag": "darwin-arm64", "platforms": [{ "os": "darwin", "cpu": "arm64" }] }
          ]
        }
      }
    },
    "2.1.0": {
      "name": "fast-thing",
      "version": "2.1.0",
      "dist": {
        "integrity": "sha512-<primary tar digest>",
        "pnpr": { "variants": ["..."] }
      }
    }
  }
}
```

What stays in an external version's `dist`:

- `pnpr.details: "external"`, the marker.
- `attestations`, because a trust policy such as
  [`trustPolicy: no-downgrade`](https://pnpm.io/settings#trustpolicy) compares
  provenance across versions before it picks one. It is a URL and a predicate
  type, which compress well.
- Each platform variant's `tag`, `platforms`, and the dependencies that differ
  from the primary's, which is all variant selection and resolution read.

Everything else moves to the version document: `integrity`, `shasum`,
`tarball`, `fileCount`, `unpackedSize`, `signatures`, `pnpr.bins`,
`pnpr.requiresBuild`, and each variant's `integrity`, `tarball`, sizes, and
bins.

The fields outside `dist` stay, so a resolver still filters by `engines`,
`os`, `cpu`, `libc`, and `deprecated`, and reads `dependencies`,
`peerDependencies`, and `hasInstallScript` without a second request.

### The marker

`dist.pnpr.details: "external"` is the only signal that a version's `dist` is
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
for a scoped name), with the version's full `dist`. npm serves the same route
today, so a client needs no new URL scheme.

For an exact version, pnpr serves it with
`Cache-Control: public, max-age=31536000, immutable`. A published version's
facts never change: a [revision](https://github.com/pnpm/rfcs/pull/42) has a
route of its own, and an unpublish removes the version from the package
document, so no client asks for it again. The route with a dist-tag in place
of a version moves, and keeps its usual caching.

### Likely picks

Every dist-tag target and the highest release of each major (of each minor
below 1.0, where a caret range stays within one minor) keep their `dist`
inline. A fresh resolution almost always picks one of these: `^1.2.3` picks
the highest 1.x, and `latest` picks its target. A lockfile update picks them
too. An older version is picked only by a narrower range, by an `overrides`
pin, or under
[time-based resolution](https://pnpm.io/settings#resolutionmode), and those
cost one extra GET each.

In the benchmark below, this cut the version documents fetched from 180 to
14, and it removed the slowdown that splitting otherwise caused on a fast
network, where pnpm had to wait for a version document before it could resolve
that package's dependencies.

The rule depends only on the document, so every pnpr instance and every CDN
node serves the same bytes for the same document.

### Threshold

A document is split when its abbreviated form is larger than a threshold.
Below it, splitting saves little and would cost a request for every package
whose resolution picks an older version. The prototype used 64 KiB of
serialized JSON; in the benchmark app, 180 of 711 documents crossed it, and
they held most of the bytes.

Once a package's document crosses the threshold, it stays split, even if it
shrinks below it again (after an unpublish, for example). A document that
flips between the two forms near the threshold would change shape for clients
on every publish, for no saving. pnpr records the decision with the package.

### Client behavior

A client that opts in:

1. Picks a version from the document as today.
2. If the picked version is marked external, fetches its version document,
   once per URL per install, sharing an in-flight request between the
   resolutions that need it.
3. Writes the lockfile exactly as today. The lockfile records the integrity,
   so a frozen install reads no metadata at all and never sees the split.

Version documents never change, so the client caches them on disk with no
expiry, keyed by registry, name, and version.

A client keeps split documents in a metadata cache of its own, apart from the
abbreviated documents it cached before. An older release that shares the
cache directory must never read a version whose integrity it cannot find. The
prototype hit exactly that: an older pnpm on `PATH`, started by the version
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
  to keep variant integrities out of large documents.
- [Release records](https://github.com/pnpm/rfcs/pull/49): a version document
  and a release record both describe one version; the record stays the signed
  statement, and the version document the cacheable read path.
- [Server-side resolution](https://github.com/pnpm/rfcs/pull/9) does not need
  this: the server reads its own storage, and the client receives a lockfile.

## Prototype

Implemented in [pnpm/pnpm#17522](https://github.com/pnpm/pnpm/pull/17522), in
pnpr's `split_packuments.rs` and pnpm's `external_dist.rs`. The benchmark
resolves an app of 787 packages (next, vite, vitest, eslint, typescript,
storybook, and others) proxied from npmjs. It runs through a proxy that gzips
and caches like a CDN, adds a round trip per request, and caps bandwidth, and
every run starts with an empty pnpm cache and store. Medians of 3 runs:

| Network | Whole | Split, every version | Split, likely picks inline |
| --- | --- | --- | --- |
| 10 ms, 200 Mbit/s, resolve | 1.71 s | 1.15 s | 1.21 s |
| 40 ms, 50 Mbit/s, resolve | 6.67 s | 4.10 s | 3.98 s |
| 100 ms, 10 Mbit/s, resolve | 31.89 s | 19.06 s | 19.04 s |
| 40 ms, 50 Mbit/s, full install | 29.89 s | 27.32 s | 27.40 s |
| Packument bytes (gzip) | 39.2 MB | 22.5 MB | 23.0 MB |
| Version documents | 0 | 180 (278 KB) | 14 (20 KB) |

Resolution is 29% to 40% faster. A full install gains less, 8%, because
tarballs dominate it. On localhost with no added latency, splitting every
version made resolution slower than whole documents (3.1 s against 1.8 s),
because the resolver waited for a version document before it walked that
package's dependencies; keeping the likely picks inline removed that.

The 9 `@next/swc-*` platform packages are still about 1 MB each after
splitting, because they list thousands of canary versions. With
[platform variants](https://github.com/pnpm/rfcs/pull/51) they would be
variants of one package and one document.

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
4. **Mark external versions by a missing `integrity`.** It is already a valid
   state with another meaning; see [The marker](#the-marker).
5. **Incremental updates of the document**, sending only what changed since the
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
- A media type or a URL of its own (`/-/v2/<name>`, for example)? A URL needs
  no `Vary` and works with every CDN, but forks the routes; the media type
  keeps one URL per package.
- Should the full document (`application/json`) be split too? Install clients
  do not read it, so the prototype left it alone.
- Should more versions stay inline, such as the highest release of each
  minor, to save the request for `~` ranges?
- The media type's name.
