# Split packuments: a package document that leaves out the versions nobody picks

## Summary

pnpr serves install clients that ask for it a package document in a format
of its own. Every version has a short entry with what picking it reads and
what may change after publication: its deprecation notice, trust rank, and
selected revision where it has any, and its publish time if it is recent.
The versions a fresh resolution almost always picks also carry their
manifest; for a large document, every other version's manifest is its
release record, an immutable document at
`<registry-base>/<name>/<version>+r<revision>`, which the client fetches
only for a version it picks. Every response stays a plain GET, which a CDN
caches for everyone when the package is public. Clients that do not ask for
the format get today's documents unchanged.

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
and provenance, and moves every version; the format below keeps a few
manifests inline and less of the other versions (see
[The document](#the-document)). Moving only `dist` leaves next at 379 KB,
almost all of it dependency lists. npm's abbreviated form has no `time`
map; pnpr's has one, for the minimum release age, and the new format keeps
the recent times (see [Publish times](#publish-times)).

[Platform variants](./0000-platform-variants.md) make it worse. Each
variant archive brings a digest of its own, so `esbuild`'s document grows from
90 KB to 361 KB gzipped and `@swc/core`'s from 190 KB to 764 KB. The tags and
platforms of the variants repeat and cost almost nothing; the digests are
nearly all of the growth.

## Detailed Explanation

### Opting in

pnpm asks for the new format only from a registry declared
`serverType: pnpr`, the declaration
[tar registries](https://github.com/pnpm/rfcs/blob/main/text/0000-zstd-registry-tarballs.md)
already require so that pnpm can rebuild pnpr's tarball URLs. It sends the
format's media type ahead of the abbreviated one:

```
Accept: application/vnd.pnpr.install-v2+json; q=1.0, application/vnd.npm.install-v1+json; q=0.9, application/json; q=0.8, */*
```

pnpr answers with `Content-Type: application/vnd.pnpr.install-v2+json`.
Every representation of a package document carries `Vary: Accept`, the new
format and the abbreviated and full documents alike, so a cache never hands
one client the form another asked for. npm, Yarn, Bun, and older pnpm releases do
not send the type and get today's abbreviated or full document, so nothing
changes for them.

Every other registry, and a pnpr that is not declared, gets npm's media
types and answers with npm's documents. Third-party registries and their
CDNs never see the new type, and the choice is declared rather than
inferred, as for the tarball layout. pnpr identifies itself with a
`pnpr-registry` response header on package documents, and pnpm warns when a
registry sends it without being declared: the install still works from npm's
documents, but without variants, pnpr's facts about the archives, or
revision overrides.

Only a client that asks for the type reads the document, so it does not
follow npm's format; see [Why a format of its own](#why-a-format-of-its-own).
A proxy in front of pnpr, such as a registry that mirrors it, must not
forward the type unless it understands the format: it could not rewrite the
document's tarball URLs, as such proxies do for npm's documents.

### The document

```json
{
  "name": "fast-thing",
  "distTags": { "latest": "2.1.0" },
  "modified": "2026-10-01T10:00:00.000Z",
  "timesSince": "2026-09-01T10:00Z",
  "versions": {
    "1.4.0": {},
    "1.4.1": { "deprecated": "1.x is no longer maintained" },
    "1.5.0": { "trust": "provenance", "revision": 2 },
    "1.9.3": { "integrity": "sha512-...", "manifest": { "...": "..." } },
    "2.1.0": {
      "time": "2026-10-01T10:00Z",
      "trust": "trustedPublisher",
      "integrity": "sha512-<primary tar digest>",
      "requiresBuild": false,
      "variants": ["..."],
      "manifest": { "dependencies": { "detect-libc": "^2.0.0" } }
    }
  }
}
```

`distTags` are npm's dist-tags, and `modified` is when the package last
changed. Each version's entry is an object with any of these fields:

- `time`: the publish time, only when the version was published at or after
  `timesSince` (see [Publish times](#publish-times)), or `null` when the
  registry has none. A `null` time is unknown, not old: the client handles
  it as it handles a registry without publish times today, which pnpm
  decides with `minimumReleaseAgeIgnoreMissingTime`. Only an entry without a
  `time` field is known to be older than `timesSince`.
- `deprecated`: the deprecation message. pnpm skips a deprecated version when
  a range also matches one that is not, and shows the message when it does
  install one. A deprecation can be added or changed after publication, so it
  lives here and never in the manifest.
- `trust`: the strongest trust evidence the version has, as the trust policy
  ranks it: `provenance`, `trustedPublisher`, or `stagedPublish`. A trust
  policy such as
  [`trustPolicy: no-downgrade`](https://pnpm.io/settings#trustpolicy)
  compares how versions were published before it picks one, and it reads
  only this rank.
- `revision`: the selected
  [registry revision](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-integrity-addressed-patch-revisions.md)
  of a revision-aware version, including `0`, and absent for an ordinary
  version. It decides the tarball URL, `<name>@<version>+r<revision>` or the
  canonical one, and with it which bytes `integrity` describes. Selecting a
  patched revision changes which release record the version is. The
  revision history is not in the document: every revision in it has its own
  release record.
- `integrity`, `fileCount`, and `unpackedSize`: the archive's digest, file
  count, and unpacked size, which npm's documents give in `dist`.
- `requiresBuild` and `bins`: whether the archive needs a build, and its
  executables with their interpreters, which pnpr derives from the verified
  archive.
- `variants`: the version's
  [platform variants](./0000-platform-variants.md), if it has any.
- `manifest`: the publisher's manifest, as
  [manifest verification](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-publish-manifest-verification.md)
  derives it, in its abbreviated form, without `dist`, `name`, and
  `version`, and without what may change after publication.

Each fact about the archives is left out when it does not apply or is not
known: `bins` when the archive has no executables, `variants` when the
version has none, `fileCount`, `unpackedSize`, and `requiresBuild` when pnpr
has not derived them, and `integrity` as described below. The examples here
show only some of them.

The entry keeps the publisher's manifest apart from what pnpr writes: the
mutable state (`deprecated`, `trust`, `revision`), the publish time, and the
facts pnpr derives from the archives sit at the entry's top level. A field
name there is pnpr's alone, so there is no `dist` and no `pnpr` namespace.

The facts about the archives and the `manifest` are the version's release
record (see [The release record](#the-release-record)), and an entry either
has all of them or none. Every entry has them while the document is under
the threshold (see [Threshold](#threshold)). Above it, only the likely picks
do (see [Likely picks](#likely-picks)), and every other version's are in its
release record, fetched when the version is picked. Dependencies, engines,
platform fields, bins, the integrity, and the platform variants are all read
only after a version is picked, and a picked version needs its integrity
anyway, so leaving them out costs no extra request. A complete entry is told
by its `manifest`, not by its `integrity`.

### Publish times

A minimum release age filters versions by their publish time before it
picks one, but it only needs to know whether a version is younger than the
cutoff. So an entry carries its time only if the version was published within
a window before the package's last publish, 30 days by default, and
`timesSince` says where that window starts. An entry without `time` was
published before `timesSince`; this is an upper bound, not its publication
timestamp. The client uses the bound only when it suffices for the cutoff,
and otherwise fetches exact times as described below.

A registry can make the window longer, which costs bytes, but not shorter
than 30 days. This is a compression choice, not a limit on client policy.
For a cutoff at or after `timesSince`, every omitted time is old enough.
For an earlier cutoff, an omitted time is an upper bound, not an exact time;
the client must obtain exact times before deciding which versions qualify.
The same rule applies to time-based resolution. Substituting `timesSince`
for an unknown older time can change the dependency graph, so it is not an
acceptable approximation.

Clients needing earlier times fetch the existing full package document's
`time` map, with ordinary conditional GET caching and access rules. Use only
its timestamps; continue resolving from the split document and verified
records under the active policy. For hosted packages, the full and split
views must identify the same committed package generation (and catalog when
snapshot reads are in use). For proxied packages, both views must derive from
the same cached upstream document. Both responses MUST carry
`PNPR-Source-Document: sha256-<base64>`, where the value is the SHA-256 digest
of that cached upstream document's exact bytes after HTTP content decoding,
encoded as standard padded Base64. The two views MUST carry the same value
when used together. This source digest is distinct from each representation's
own ETag; full and split representations need not have equal ETags. Upstream
validators such as ETags still revalidate the source cache; using a digest on
the client-facing path also covers missing or weak upstream validators.

Capture the header value from the source snapshot used to render each view,
not from the latest upstream state when serving a cached response. Caches
retain the header beside the body. A 304 response MUST carry the source
identity of the validated representation, which the client saves with its
cached body. Clients compare identities only within the same declared
registry and package. On a mismatch they MUST revalidate both views and
retry, without evaluating older cutoffs from the mismatched pair. A missing
or malformed header, or a pair that still differs after one revalidation
attempt, is an explicit exact-time capability or consistency error; never
combine the timestamps or silently approximate them. A server without a
consistent exact-time path returns an explicit capability error for that
lookup. An actually unknown publish time remains unknown and follows
`minimumReleaseAgeIgnoreMissingTime`.
Do not globally reject `minimumReleaseAge` values above 30 days, including
when using other registries. Counting the window from the last publish rather
than the current time keeps the document unchanged until the next publish,
so it caches as well as today's.

The times are most of the size of the entries without a manifest. For 8
popular packages (react, typescript, `@types/node`, next, eslint, webpack,
lodash, and `@babel/core`), those entries took 105.2 KB gzipped with a time
in every entry and 49.4 KB with the window, and the whole document 198.6 KB
and 142.8 KB. An object in every entry costs nothing measurable over a bare
value, since most entries are `{}`. Keeping the times in the entries, so
each version string appears once, mattered too: in an earlier draft with a
time for every version, the times alone took 83.8 KB, the times and a
separate map of flags 126.1 KB, and the two merged 87.6 KB.

### Why a format of its own

npm, Yarn, Bun, and older pnpm releases never ask for the type, and pnpm
keeps these documents in a metadata cache of its own (see
[Client behavior](#client-behavior)), so nothing that expects npm's format
reads one. The format can then say what it means directly: an entry without
a manifest is a version whose manifest is elsewhere, and deprecation, trust,
and revision are written the same way for every version. npm's format has
no place for either. An earlier draft kept npm's format and moved the other
versions to a map of their own, so that a client that did not know the split
saw only complete versions; with the type as the only way in, that client
does not exist.

npm's documents served by pnpr carry none of pnpr's facts: no `dist.pnpr`,
no variants, and none of the revision protocol's fields. A selected revision
appears there as npm clients read any version, by its `+r<revision>` tarball
URL and its integrity. A pnpm release that read `dist.pnpr.requiresBuild` and
`dist.pnpr.bins` from them falls back to what it does with any registry,
reading both from the archive after it fetches it. pnpr is experimental, so
its documents and clients change together.

A missing `integrity` in a complete entry keeps its meaning: a version whose
integrity the client computes while it fetches the tarball, as pnpm does
today for registries that omit it.

### The release record

A version's release record is served at
`<registry-base>/<name>/<version>+r<revision>`
(`<registry-base>/@scope%2fname/<version>+r<revision>` for a scoped name),
and it holds the facts about the archives and the `manifest` of the
version's entry, with the version's `name` and `version`:

```json
{
  "name": "fast-thing",
  "version": "1.5.0",
  "integrity": "sha512-<primary tar digest>",
  "requiresBuild": false,
  "manifest": { "dependencies": { "detect-libc": "^2.0.0" } }
}
```

An ordinary version, outside the revision protocol, has one artifact, its
original, and its record is at `+r0`: on this route, revision 0 names the
original artifact whether or not the version is revision-aware. The entry's
`revision` decides only the tarball URL, so an ordinary version's record
describes the archive at its canonical URL, and a revision-aware original's
the tar at `<name>@<version>+r0`.

It is a metadata endpoint, separate from the revision route of
[integrity-addressed patch revisions](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-integrity-addressed-patch-revisions.md),
`<name>@<version>+r<revision>`, which serves the revision's tar. The record
extends the version-document route `<name>/<version>`, which pnpr and npm
already serve, with the same `+r<revision>` suffix, so a scoped name keeps
the same escaping on both routes. It leaves out what may change after
publication and what
[release records](./0000-release-records.md) keep as separate
evidence: no `deprecated`, no `_npmUser`, no `tarball`, and no registry
signatures or attestations. What remains never changes, so
pnpr serves it with `Cache-Control: public, max-age=31536000, immutable`
when anyone may read the package. A package closed to anonymous callers gets
the headers of every caller-gated response
([registry mounts](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-pnpr-registry-mounts.md)):
`Cache-Control: private, no-store` and `Vary: Authorization`, for its
release records and its package document alike. The client still caches
the records on disk for the caller that fetched them.

A record that was served publicly stays in shared caches for its lifetime,
like the public tarball of the same version. So when a package goes from
public to caller-gated, pnpr purges its records from the CDN together with
its tarballs and documents, as the operator's CDN integration does for any
visibility change. Without a purge integration, pnpr refuses that
transition for a package whose records were served publicly.

The address carries the revision because `<name>/<version>` does not name
fixed content: it is the registry's currently selected revision, and
selecting a patched revision changes its integrity and can change its
dependencies. That route keeps its meaning and its usual caching. Addressing
the record by a hash of its content would also be immutable, but it would put
an incompressible digest per version back into the package document.

A client checks that the record names the version it picked, and takes the
deprecation and trust evidence from the package document.

### Likely picks

Above the threshold, every dist-tag target and the highest release of each
major (of each minor below 1.0, where a caret range stays within one minor)
keep their manifests. A fresh
resolution almost always picks one of these: `^1.2.3` picks
the highest 1.x, and `latest` picks its target. A lockfile update picks them
too. An older version is picked only by a narrower range, by an `overrides`
pin, or under
[time-based resolution](https://pnpm.io/settings#resolutionmode), and those
cost one extra GET each.

When the prototype first split every version, a resolution of the benchmark
app fetched 180 version documents and was slower than whole documents on
localhost (3.1 s against 1.8 s), because pnpm waited for a version document
before it resolved that package's dependencies. With the likely picks'
manifests inline, it fetches 22 release records.

The rule depends only on the document, so every pnpr instance and every CDN
node serves the same bytes for the same document.

### Threshold

Every document under the new type has the same format. Above a threshold
on the size of its abbreviated form, only the likely picks keep their
manifests. Below it, every entry has one: leaving manifests out saves little
there and would cost a request for every package whose resolution picks an
older version. The prototype used 64 KiB of serialized JSON.

Once a package's document crosses the threshold, it stays split, even if it
shrinks below it again (after an unpublish, for example). A document that
flips between the two near the threshold would change for clients on every
publish, for no saving. pnpr records the decision with the package.

### Client behavior

A client that opts in:

1. Reads every entry into its list of versions, with their deprecations
   and trust ranks, and picks a version as today. An entry without a `time`
   field is known to predate `timesSince`; that bound suffices only for a
   cutoff at or after it. Earlier cutoffs fetch exact times as above. An entry
   with `time: null` has an unknown
   publish time and goes through `minimumReleaseAgeIgnoreMissingTime`, as a
   missing time does today.
2. If the picked version has no manifest in the document, fetches its
   release record, once per URL per install, sharing an in-flight request
   between the resolutions that need it, and reads the version from it,
   dependencies included. An override that pins another revision (`+rN`)
   fetches that revision's record the same way.
3. Writes the lockfile as today: an ordinary version records its integrity,
   and a revision-aware version its integrity and `revision`, including `0`,
   from which a frozen install rebuilds `<name>@<version>+r<revision>`. A
   frozen install reads no metadata at all and never sees the split.

Release records never change, so the client caches them on disk with no
expiry, keyed by registry, name, version, and revision.

A client keeps these documents in a metadata cache of its own, apart from
the abbreviated documents it caches for other registries (pnpm uses
`v12/metadata-pnpr` beside `v12/metadata`). An older release that shares the
cache directory could not read them. The prototype hit an earlier form of
this: an older pnpm on `PATH`, started by the version switch, read a split
document the new pnpm had cached and failed.

### CDN caching

A document in the new format is a generated view of the package, like the
abbreviated and full documents, and follows the same cache contract as they
do in
[SQL metadata storage](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-sql-metadata-storage.md):
pnpr caches it keyed by representation, the package's committed generation,
and any authorization context, gives it bounded freshness, and revalidates
the generation when that expires. A tag move, a deprecation, a trust
change, or a newly selected revision is a committed mutation, so it advances
the generation and the next response reflects it. Release records are the
exception: they never change, so they need no revalidation.

Every response is a GET addressed by its URL and the `Accept` header. There
are no range queries, `?since=` cursors, or POST lookups, because each of
those makes a cache key of its own, or none. For a package anyone may read,
the document is cached for everyone who sends the new type, and a
release record for everyone who picks that version. A caller-gated package
stays out of shared caches, as today.

A CDN keys the package document by `Accept` through `Vary: Accept`. CDNs that
do not honor `Vary` need the `Accept` header normalized into the cache key, or
a URL of its own for the new format; see
[Unresolved Questions](#unresolved-questions-and-bikeshedding).

### Relation to other RFCs

- [SQL metadata storage](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-sql-metadata-storage.md) keeps
  accepted manifests insert-only and stores tags, revision selections, and
  deprecations as mutable state beside them. The new format exposes that
  division to clients: a manifest is one of its per-version fragments,
  rendered, and the rest of each entry is the mutable state. Its
  open question, whether fragments become a client-facing read path, is
  answered here with a narrow yes, and this RFC removes it from there.
- [Platform variants](./0000-platform-variants.md) relies on this RFC
  to keep variant archives out of large documents. Variant selection happens
  after a version is picked, so the variants move with the rest of the
  version.
- [Release records and upstream verification](./0000-release-records.md)
  keeps endorsements and evidence apart from the immutable record, as the
  release record here leaves out signatures and attestations. Its signed
  record and the one served here describe the same release; the one served
  here is the cacheable read path.
- [Server-side resolution](https://github.com/pnpm/rfcs/pull/9) does not need
  this: the server reads its own storage, and the client receives a lockfile.

## Prototype

Implemented in [pnpm/pnpm#17522](https://github.com/pnpm/pnpm/pull/17522), in
pnpr's `split_packuments.rs` and pnpm's `split_index.rs` and
`version_document.rs`. The benchmark resolves an app of 787 packages (next,
vite, vitest, eslint, typescript, storybook, and others) proxied from npmjs.
It runs through a proxy that gzips and caches like a CDN, adds a round trip
per request, and caps bandwidth, and every run starts with an empty pnpm
cache and store. The registry is declared with `serverType: pnpr`, so pnpm
reads optional dependencies from abbreviated documents too. It compares
npm's abbreviated documents, the new format with every manifest inline (no
threshold), and the new format as proposed. Medians of 3 runs:

| Network | npm's abbreviated | New format, every manifest | New format, 64 KiB threshold |
| --- | --- | --- | --- |
| 10 ms, 200 Mbit/s, resolve | 1.35 s | 0.76 s | 0.36 s |
| 40 ms, 50 Mbit/s, resolve | 5.07 s | 2.60 s | 0.79 s |
| 100 ms, 10 Mbit/s, resolve | 24.68 s | 12.38 s | 2.62 s |
| 40 ms, 50 Mbit/s, full install | 28.50 s | 25.99 s | 24.03 s |
| Package documents (gzip) | 30.3 MB | 14.9 MB | 2.5 MB |
| Release records | 0 | 0 | 22 (7 KB) |

Resolution is 73% to 89% faster than with npm's documents, and a full
install 16% faster, as tarballs dominate it. Every form produces the same
lockfile.

The format alone halves the bytes, before any manifest moves out: it leaves
out npm's registry signatures, which do not compress, the attestations and
`_npmUser`, which become a `trust` rank, the publish times older than the
window, and the repeated `name` and `version`. So documents under the
threshold, which are never split, shrink too. An earlier draft that kept
npm's format, with the other versions in a map of their own, transferred
4.3 MB and resolved in 0.32 s, 1.01 s, and 4.03 s on the same three
networks, in a separate run; most of its 4.3 MB was in documents under the
threshold.

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
   on the way. But the other versions are rare picks, and dependency
   lists pinned to the package's own version do not compress: next stays at
   379 KB gzipped, against 21 KB with whole versions moved. In the
   benchmark, that draft transferred 7.6 MB and resolved in 0.45 s, 1.51 s,
   and 6.67 s on the three networks of the [Prototype](#prototype).
5. **Keep npm's format**, with the other versions in `versions` behind a
   marker such as `dist.pnpr.details: "external"`, or in a map of their own
   beside `versions`, as earlier drafts did. Either keeps a document that npm
   clients could read, but none of them asks for the type. The marker cost
   more too: for 16 popular packages, such versions, with `name`, `version`,
   the `_npmUser` flags, `dist.attestations`, and the marker, took 157 KB
   gzipped, against 68 KB for the same flags in short entries. Marking them
   by a missing `integrity` is worse still, since that is already a valid
   state with another meaning.
6. **Incremental updates of the document**, sending only what changed since the
   client's copy. This is complementary and stays cacheable only with
   Cargo-style conditional GETs of the whole document, or with append-only
   segments where every closed segment is immutable. It helps warm caches; this
   RFC helps cold ones.

## Implementation

1. pnpr: serve the new format under the new media type with `Vary: Accept`,
   with the likely-picks rule, a configurable threshold, and a configurable
   publish-time window of at least 30 days (30 by default), and record that a
   package is split.
2. pnpr: serve release records at `<name>/<version>+r<revision>` for every
   accepted revision, with immutable caching, and remove `dist.pnpr`,
   `variants`, any variant declarations pnpr stores with a version,
   `dist.revision`, and `dist.revisions` from npm's documents; identify
   itself with the `pnpr-registry` header.
3. pnpm: send the new media type to registries declared `serverType: pnpr`,
   read the format, fetch the release records of picked versions without a
   manifest and of `+rN` overrides, cache them on disk, and keep these
   documents in a metadata cache directory of their own. Warn about an
   undeclared registry that sends `pnpr-registry`.
4. pnpm: fetch consistent exact publish times when a release-age or time-based
   cutoff predates `timesSince`, without imposing a global 30-day policy limit.
5. pnpm: when the user points pnpm at a registry that does not serve the type,
   nothing changes, since it answers with the abbreviated document.

Tests must cover an entry without a manifest, with and without flags; npm's
documents of the same package (no pnpr facts); one with a time
inside the window (filtered by it), one without a time (passes a cutoff at or
after `timesSince`, and fetches
exact times for an earlier cutoff), and one with a `null` time (handled as a
missing time, passing
or failing with `minimumReleaseAgeIgnoreMissingTime`); a caller-gated package
(its records and document are `private, no-store`); a public package made
caller-gated (its records are purged, or the change is refused); a
deprecation, tag move, and revision selection each changing the next
document; a scoped package's release record fetched through a CDN; a
revision-aware version at revision 0 (rebuilt `+r0` URL) and at a patched
revision, installed through pnpr's format and through npm's documents; a
`+rN` override of a non-selected revision (its record is fetched), and the
same override against npm's documents (fails, naming the revision); an
undeclared pnpr (npm's documents, and the warning); a version
deprecated after its record was cached (the new message shows), a selected
patch revision (its record is fetched, not the original's), a version without
`integrity` in `versions` (computed while fetching the tarball, as today), a
pick of a likely pick (no second request), a pick under `overrides` and under
time-based resolution (one request), a trust downgrade detected across
versions with and without a manifest, a document under the threshold (every
version has its manifest), an older pnpm sharing the home directory, and a frozen
install that reads no metadata. Also test a 60-day release-age policy,
time-based resolution with an older cutoff, and mismatched generations while
fetching exact times; the graph must equal the unsplit representation's graph.
For proxied packages, test an upstream refresh between the two reads, distinct
representation ETags derived from one source validator, and a source document
without an upstream validator. Check the `PNPR-Source-Document` header on
both representations and on 304 responses, retention beside cached bodies,
missing and malformed headers, and retry exhaustion under concurrent upstream
refreshes. Mismatched source identities must be retried without mixing views.

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
- Should the full document (`application/json`) have a split form too?
  Install clients do not read it, so the prototype left it alone.
- Should more versions keep their manifests, such as the highest release of
  each minor, to save the request for `~` ranges?
- The route of a release record. `<name>/<version>+r<revision>` mirrors the
  tarball route, but `+r<revision>` could be mistaken for build metadata of a
  version that has some.
- The media type's name.
