# Platform variants: one version, many artifacts

> Status: draft for discussion. No implementation or protocol version is approved.

## Summary

Let one package version carry a primary archive plus platform-specific variant
archives, published together and described in pnpr's own
[package document format](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-split-packuments.md).
npm's documents do not list variants, so npm clients see only the primary
archive. pnpm locks every variant in a platform-neutral lockfile entry and links the variant
that matches the host *as* the package itself. Native packages then no longer
need one separately named package per platform wired through
`optionalDependencies`, and no longer need install scripts that download
binaries.

## Motivation

The npm registry stores one archive per version. A package with native code
works around that in one of two ways, and both fail in practice.

**Separate packages per platform.** esbuild, swc, rollup, sharp, biome,
lightningcss, next, and every napi-rs package publish one package per platform
and list them all in the main package's `optionalDependencies`, each with
`os`, `cpu`, and `libc`. The resolved tree then depends on the machine that
resolved it:

| Report | Year | 👍/comments | What broke |
| --- | --- | --- | --- |
| [npm/cli#4828](https://github.com/npm/cli/issues/4828) | 2022 | 818/307 | Platform-specific optional dependencies missing from `package-lock.json`. The most-upvoted issue in npm's trackers. Open three years. |
| [npm/npm#17722](https://github.com/npm/npm/issues/17722) | 2017 | 336/69 | Linux developers drop `fsevents` from the lockfile and macOS developers add it back. |
| [npm/npm#14042](https://github.com/npm/npm/issues/14042) | 2016 | 167/81 | A shrinkwrap makes npm install an optional dependency on an unsupported platform. |
| [npm/cli#558](https://github.com/npm/cli/issues/558) | 2019 | 50/38 | `npm ci` installs the Linux variant on macOS and the reverse. |
| [npm/cli#8805](https://github.com/npm/cli/issues/8805) | 2025 | 6/4 | "Missing: fsevents@2.3.3 from lock file" after the 2025 fix. |
| [npm/cli#10043](https://github.com/npm/cli/issues/10043) | 2026 | 0/8 | Still open: "Missing: @napi-rs/nice-android-arm-eabi@ from lock file". |
| [npm/rfcs discussion 120](https://github.com/npm/rfcs/discussions/120) | 2020 | 15/4 | "Platform-Specific Dependencies". Of esbuild: "the fact that npm doesn't natively support something like this seems like madness". |

The pattern has costs beyond lockfile churn:

- **Metadata fan-out.** Locking every variant means fetching every variant's
  package document. In October 2026, abbreviated and gzipped, that was 26 extra
  documents and 1.0 MB for `esbuild`, and 12 documents and 2.8 MB for
  `@swc/core`.
- **Releases are not atomic.** Each variant is a separate publication, so a
  failed or late variant is visible to installers as a broken lockfile.
- **More names to take over.** Each variant name is a package with its own
  owners, and a target for squatting or takeover.
- **Two meanings of optional.** `optionalDependencies` means both "may fail to
  install" and "only for some platforms". A tree with every optional
  dependency omitted is valid, so nothing can require exactly one variant.
- **Loader shims.** The main package must `try`/`catch` its way through
  `require` calls to find the installed variant, and CLIs need a JavaScript
  `bin` shim that spawns the real executable.
- **Install scripts anyway.** Of 32 popular packages that use the pattern
  (October 2026, see [Migration](#migration)), 14 still ship an install
  script: esbuild, `@swc/core`, deno, bun, `@anthropic-ai/claude-code`,
  workerd, dprint, `@ast-grep/cli`, `@bufbuild/buf`, `@sentry/cli`,
  `unrs-resolver`, `@parcel/watcher`, lmdb, and `msgpackr-extract`. Most of
  them replace the JavaScript `bin` shim with the native executable, or
  install the platform package that npm dropped. Each one asks for an
  `allowBuilds` entry in pnpm.

**Install-script downloads.** node-pre-gyp, prebuild-install, sharp before
0.33, puppeteer, and electron download binaries in an install script. Those
bytes have no lockfile integrity, no proxy or offline support, and they do not
arrive when install scripts are disabled
([npm/npm#1891](https://github.com/npm/npm/issues/1891),
[npm/feedback discussion 80](https://github.com/npm/feedback/discussions/80)).
These packages are a large part of why build allowlists exist.

pnpm v12 already fixes the client half of the first problem: it locks every
optional variant whatever the host, filters at link time with
[`supportedArchitectures`](https://pnpm.io/settings/dependency-resolution#supportedarchitectures),
and pnpr's server-side resolution returns a platform-neutral lockfile. What
remains is the publisher half, which only a registry can fix: a version that
can hold more than one archive.

## Detailed Explanation

### Model

A version has:

- one **primary** archive, installed when no variant matches. It can contain a
  portable implementation (JavaScript or wasm), a source build, or only an error
  that names the supported platforms;
- zero or more **variants**, each an archive and the list of **platforms** it
  runs on, which also names it.

A platform is one combination of OS, CPU, and C library, with the fields in
[Platform tags](#platform-tags), and a variant lists its platforms one by one.
It does not use the `os`, `cpu`, and `libc` arrays of `package.json`, which
match every combination of their values: a build for glibc x64 and musl
arm64 declared that way also claims musl x64 and glibc arm64.

A variant usually has one platform. It lists several when one build serves
several hosts:

- A statically linked Linux executable runs on glibc and musl hosts. Go tools
  (esbuild, `@typescript/native-preview`, `@bufbuild/buf`, gh, golangci-lint)
  ship one Linux build per CPU, and typst and ripgrep ship only musl builds
  for x64, which glibc hosts use too.
- A macOS universal binary, such as protoc's and CMake's, serves `x64` and
  `arm64`.
- `turbo` and `@sentry/cli` already declare one Linux build for `linux` and
  `android` (and `freebsd`) in their `os` field.

A version with variants may omit the primary. That fits tools with no portable
implementation, such as a Go or Rust CLI. pnpm then fails on a host that no
variant matches, with an error that lists the published platforms, and skips the
package when it is an optional dependency. In pnpr's package document, top-level
`integrity` is absent. For npm's documents, pnpr generates and stores a minimal
primary archive at publication acceptance containing a `package.json` with the
version's name, version, and the union of the variants' `os`, `cpu`, and Linux
`libc` (a Linux platform without `libc` counts as `glibc`), so npm reports an
unsupported platform instead of installing an empty package. npm checks `libc`
only on Linux, so the field does not exclude other operating systems. Those
fields match every combination of their values, so a host whose OS and CPU each
appear in some variant, but not together, still gets the empty package (see
[Clients that do not read variants](#clients-that-do-not-read-variants)). pnpr
builds the generated archive deterministically from those fields, so every
replica and every rebuild produces the same integrity. A revision that replaces
the variant set gets its own generated primary, built from its variants and
served at its `<name>@<version>+r<revision>` route like any revision's archive;
the canonical route keeps the original's bytes, and npm's documents point at the
selected revision's archive as for any other revision. It is not a subject of
the version's provenance statement and does not affect the version's trust
level.

A variant archive is a complete package for its platform, not an overlay on
the primary. Its `package.json` has the same `name` and `version` as the
primary. Linking a variant means extracting its archive as the package.

A variant's `package.json` declares the variant, so the archive describes
itself and nothing about it is supplied beside the upload:

```json
{
  "name": "fast-thing",
  "version": "1.4.0",
  "variant": {
    "platforms": ["linux-x64", "linux-x64-musl"]
  },
  "bin": { "fast-thing": "bin/fast-thing" }
}
```

`platforms` lists [platform tags](#platform-tags), each one combination.
pnpr reads the declaration from the verified archive and
derives the version's `variants` from it, as it derives `requiresBuild` and
`bins`, so the archive's provenance statement covers where the build claims
to run. The primary archive has no `variant` field. The field is
top-level rather than under a pnpr key: it describes the package, not the
registry, and pnpr rejects an archive whose `variant` does not parse, so a
field npm might add under the same name would fail publication instead of
being misread.

The variant set is part of the immutable version. It is fixed when the version
is published, and a [registry revision](./0000-integrity-addressed-patch-revisions.md)
replaces it as a whole.

### Invariants the registry enforces

At publication, pnpr derives each archive's manifest under the
[manifest verification profile](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-publish-manifest-verification.md) and
rejects the version unless:

- every archive's `name` and `version` equal the version's;
- every archive's `peerDependencies` and `peerDependenciesMeta` are equal across
  the primary (when present) and all variants, so the package's peers, and with
  them its key in the lockfile, do not depend on the platform;
- no variant requires a build (its `requiresBuild` is false), so a
  variant never needs an `allowBuilds` entry;
- every variant archive declares a `variant` field with at least one
  platform, and no `os`, `cpu`, or `libc` field, so a variant has one
  statement of where it runs and it cannot disagree with another;
- no two variants list the same platform with the same floors, so no host
  can tie between them and no two variants get the same tag.

A variant may differ from the primary in its files, its dependencies,
including their specifiers and whether they are optional, bundled
dependencies, `bin`, `main`, `exports`, and `engines`. The primary may require
a build: a source build is a fallback, and it needs an `allowBuilds` entry
only on hosts that link the primary.

Platform packages today declare their own dependencies, and variants keep
that. sharp's `@img/sharp-linux-x64` depends on `@img/sharp-libvips-linux-x64`.
napi-rs `wasm32-wasi` packages depend on `@napi-rs/wasm-runtime` and `@emnapi/*`
(about 4 MB unpacked). sass-embedded's fallback depends on the pure-JavaScript
`sass` (6 MB). Requiring every archive to declare the same dependencies would
install those on every host, or force them to be bundled, or force the
dependency to migrate to variants in the same release. A dependency that only
some archives declare is installed only where one of those archives is
linked, and it is required there, not optional: unlike an optional
dependency, a failure to install it fails the install.

Peer dependencies stay equal because pnpm keys a package by its resolved
peers. A peer that only the Linux variant declared would give the package a
different key on Linux, and the lockfile would depend on the platform again.

### Platform tags

A platform tag uses the syntax of `supportedArchitectures` platform lists:
`<os>-<cpu>`, optionally followed by a C library, plus the additional parts
below. The structured form in metadata has one field per part:

| Field | Values | Matching |
| --- | --- | --- |
| `os` | Node.js `process.platform` values | Exact. |
| `cpu` | Node.js `process.arch` values | Exact. |
| `libc` | `glibc`, `musl`; absent outside Linux | Exact. Absent on Linux means glibc. |
| `libcVersion` | glibc floor, `manylinux` semantics, such as `2.28` | Host version at least the floor. |
| `osVersion` | macOS or Windows floor, such as `11.0` | Host version at least the floor. |
| `napi` | Lowest N-API version the addon needs, such as `8` | Host N-API version at least the value. |
| `nodeAbi` | `node-<NODE_MODULE_VERSION>` or `electron-<abi>`, for addons outside N-API | Exact. |
| `wasm32` | `true` | Matches every host, ranked last. |

A platform's tag is its fields written in this order, joined with `-`, with
absent fields left out:

| Field | String form |
| --- | --- |
| `os`, `cpu` | the value, such as `linux-x64` |
| `libc` | `musl`; glibc is written as no part |
| `libcVersion` | `manylinux_<major>_<minor>`, as in PEP 600 |
| `osVersion` | `macos_<major>_<minor>` on `darwin`, `windows_<major>_<minor>` on `win32` |
| `napi` | `napi<N>` |
| `nodeAbi` | `node<N>` or `electron<N>` |
| `wasm32` | the whole tag is `wasm32` |

Versions are written as two decimal numbers without leading zeros, so `11`
and `11.0` both give `11_0`. Examples: `linux-x64`, `linux-x64-musl`,
`linux-x64-manylinux_2_28-napi8`, `darwin-arm64-macos_11_0`, `win32-x64`.
The form of a field never changes once specified: a later vocabulary only
adds fields, which appear only in the tags of platforms that use them, so a
tag computed today stays the same under every later vocabulary.

A variant's tag is its name in routes and the lockfile. pnpr derives it, and
pnpm derives the same string from the lockfile's `targets`: the variant's
platform tags, sorted by their bytes and joined with `+`, a character platform
tags do not use. A variant with one platform is named by that platform, such
as `linux-x64-musl`; a static Linux build is `linux-x64+linux-x64-musl`, and a
macOS universal binary `darwin-arm64+darwin-x64`. Overlapping platforms are
rejected, so derived tags are unique within a version, and the publisher has
no name to choose or keep stable.

`process.arch` reports `arm` for both ARMv6 and ARMv7. napi-rs builds `arm`
as ARMv7 (`gnueabihf`, `musleabihf`), and the vocabulary keeps that meaning. gh,
golangci-lint, and gitleaks also publish ARMv6 builds, which would need an
`armVersion` floor added to the vocabulary later.

`nodeAbi` is matched against the runtime the addon loads into, not the Node.js
that runs pnpm. An Electron app sets that runtime and version in its
configuration, as `@electron/rebuild` takes them today. Without it, the host
Node.js is used.

The vocabulary is versioned. A tag with a field the client does not know is
not selectable, so an older client falls back to the primary archive instead
of guessing. When the version has no primary, no other variant matches, and
the package is not optional, the install fails as for any unmatched host. pnpr's [shared side-effects cache](https://pnpm.io/pnpr/shared-side-effects-cache#platform-compatibility)
already applies the same rule to its `pnpm:v1:` tags. The two vocabularies
should be one specification with shared test vectors.

### Selection

For a host, the client keeps the variants with a platform whose every field
matches, then picks one in this order, comparing each variant by its best
matching platform:

1. Native targets rank above `wasm32`. Among native targets, prefer the
   greatest number of explicit constraints after normalization. Count `os`,
   `cpu`, Linux `libc` (including implicit glibc), each version floor, and the
   ABI constraint once each; do not count absent fields.
2. Among equal counts, compare `libcVersion`, then `osVersion`, with absent
   floors below every present floor. Compare numeric `(major, minor)` pairs,
   never strings.
3. Among remaining ties, prefer an exact `nodeAbi` match over a `napi` match,
   and a `napi` match over no ABI constraint. For two `napi` matches, prefer
   the greatest numeric N-API floor. `napi` and `nodeAbi` are mutually
   exclusive within one target. Two matching `nodeAbi` values must be equal
   because the host has one runtime family and ABI.
4. A `wasm32` variant ranks below every matching native target, and above the
   primary archive. The primary is selected only when no variant matches.

A variant's score is the maximum of its matching target scores under this
lexicographic rule. When choosing the archive to link, an unknown floor or ABI
value cannot satisfy an explicit constraint: linking an archive the host may not
run is worse than linking the primary. A platform named in
`supportedArchitectures` states its floors and ABI in the platform tag syntax,
such as `linux-x64-manylinux_2_26-node127`; any it does not state are unknown.
Fetching is looser (see [Install](#install)). Targets normalize implicit Linux
glibc before counting or comparing fields, so spelling it explicitly cannot
change specificity. Reject a publication if distinct variants can share the
highest score on any host. Clients also reject ties in malformed metadata rather
than use array order or a tag as an arbitrary tie-breaker.

### Metadata

Variants are described only in pnpr's own package document,
`application/vnd.pnpr.install-v2+json`, which pnpm asks for only from
registries declared with `serverType: pnpr` and npm clients never do. npm's
abbreviated, full, and version documents describe the version as npm clients
read it, without variants. That keeps npm's documents the size they are today
and leaves the variant metadata free to follow pnpr's format alone.

In pnpr's document, `variants` is a field of the version's entry, next to the
primary archive's `integrity`, `requiresBuild`, and `bins`, and it is in the
version's release record at `<registry-base>/<name>/<version>+r<revision>`.
pnpr computes every field from the verified archives and ignores
publisher-supplied ones:

```json
{
  "versions": {
    "1.4.0": {
      "trust": "provenance",
      "integrity": "sha512-<primary tar digest>",
      "requiresBuild": false,
      "variants": [
        {
          "tag": "linux-x64-manylinux_2_17-napi8",
          "platforms": [
            { "os": "linux", "cpu": "x64", "libc": "glibc", "libcVersion": "2.17", "napi": 8 }
          ],
          "integrity": "sha512-<variant tar digest>",
          "unpackedSize": 9123456,
          "fileCount": 4,
          "bins": { "fast-thing": { "path": "bin/fast-thing" } },
          "manifest": { "bin": { "fast-thing": "bin/fast-thing" } }
        },
        {
          "tag": "darwin-arm64-macos_11_0+darwin-x64-macos_11_0",
          "platforms": [
            { "os": "darwin", "cpu": "x64", "osVersion": "11.0" },
            { "os": "darwin", "cpu": "arm64", "osVersion": "11.0" }
          ],
          "integrity": "sha512-<variant tar digest>",
          "unpackedSize": 18234567,
          "fileCount": 4,
          "bins": { "fast-thing": { "path": "bin/fast-thing" } },
          "manifest": { "bin": { "fast-thing": "bin/fast-thing" } }
        }
      ],
      "manifest": { "bin": { "fast-thing": "bin/fast-thing.js" } }
    }
  }
}
```

`integrity` is the digest of the uncompressed tar, as for every pnpr archive.
There is no `tarball`: the client derives each archive's URL from the routes
below.

A variant entry carries a `manifest`, matching the
`{ <archive facts>, manifest }` shape of the version entry itself, with only
the fields whose value differs from the version's `manifest`, such as
`dependencies`, `optionalDependencies`, `bundleDependencies`, and `engines`. A
field present there replaces the version's field as a whole, and a field the
version's `manifest` has but the variant's does not is given as `null`. A
variant whose manifest equals the version's has `"manifest": {}`. The
`variant` field is not part of it: the entry's `platforms` and `tag` describe
it. When the version has a primary, the version's `manifest` is the primary's
manifest; when it has no primary, the version's `manifest` holds the shared
`peerDependencies` and `peerDependenciesMeta` plus the `dependencies` and
`optionalDependencies` that every variant declares with the same specifier
(`{}` when none). pnpr derives each `manifest` from the verified archive. A
resolver therefore sees every archive's dependencies in the one entry it
already reads.

Inline, the variants would make a package's document several times larger.
Rebuilt from today's platform packages, `esbuild`'s document would grow from
90 KB to 361 KB gzipped and `@swc/core`'s from 190 KB to 764 KB. The variant
integrities cause nearly all of it: a digest does not compress, while tags
and platforms repeat from version to version.

pnpr's format leaves them out of large documents, as
[the split packuments RFC](https://github.com/pnpm/rfcs/blob/main/pnpr/text/0000-split-packuments.md)
does for all packages. Every version has a short entry with its deprecation,
trust rank, selected revision, and its publish time if that is recent, and
the likely picks, every dist-tag target and the highest release of each
major (of each minor below 1.0), also carry their manifest and archive
facts. For every other version, its dependencies, its variants, and every
integrity are in its immutable release record
`<registry-base>/<name>/<version>+r<revision>`, which the resolver fetches
for the versions it picks. Variant selection happens after a version
is picked, so the variants move with the rest of the version. Split,
`esbuild`'s document with variants is 29 KB gzipped, most of it the 29
manifests it keeps inline, one per 0.x minor, and `@swc/core`'s is 7 KB,
both smaller than today's 90 KB and 190 KB without variants.

The prototype measured this on an app of 787 packages proxied from npmjs,
through a proxy that gzips and caches like a CDN, with an empty pnpm cache in
every run. Resolution took 0.36 s instead of 1.35 s at 10 ms and 200 Mbit/s,
0.79 s instead of 5.07 s at 40 ms and 50 Mbit/s, and 2.6 s instead of
24.7 s at 100 ms and 10 Mbit/s. Package documents shrank from 30.3 MB to
2.5 MB gzipped, and 22 release records were fetched. A full install at
40 ms and 50 Mbit/s took 24.0 s instead of 28.5 s, as tarballs dominate it.

A [release record](./0000-release-records.md) of a version with
variants covers every archive of the version, the primary and each variant,
so the record, like the version document, describes the version as one
release.

A package with per-ABI addons multiplies the variant count. better-sqlite3
12.x attached 145 prebuilds to each GitHub release (105 for Electron ABIs, 40
for Node.js). Such a package should move to N-API before it moves to variants.

### Archive routes

A variant is served from an immutable route next to the version's own:

```text
<registry-base>/<name>@<version>/variants/<tag>
<registry-base>/<name>@<version>+r<revision>/variants/<tag>
```

`<name>` uses the same form as the tarball and revision routes
(`<registry-base>/@<scope>/<name>@<version>/variants/<tag>` for a scoped
package), and `<tag>` is a single path segment of `[a-z0-9._+-]+` (never `.` or
`..`, and containing no `/` or `%`). Responses follow the revision route's
rules: a tar with `gzip` or `zstd` content coding, no redirects, and integrity
checked by the client. Archives are content-addressed in storage, so a variant
unchanged across revisions is stored once.

### Lockfile

pnpm records a package with variants using the existing `variations`
resolution shape, extended with the selection rule above. Serialize a
package's variants in byte order of their derived tags for a platform-neutral
lockfile; array order does not choose its archive. At install time, evaluate
every variant's best matching target for the actual host and rank those
matches. A runtime's `binary` variants keep their written order, which their
first-match selection depends on.

The current first-match implementation cannot express this rule for all hosts.
For example, variant A can list `linux-x64` and `darwin-arm64-napi9`, while B
lists `linux-x64-napi8` and `darwin-arm64`. Linux with N-API 9 must prefer B;
macOS arm64 with N-API 9 must prefer A. No single ordering of A and B works.
Reuse the resolution type and matching utilities, but extend the selector for
these records. The selector chooses its rule from each variant's resolution: a
runtime's `binary` archives keep first-match selection, and a package's
registry or tarball variants are ranked. It needs no version check.

For example:

```yaml
packages:
  fast-thing@1.4.0:
    resolution:
      type: variations
      variants:
        - resolution:
            integrity: sha512-<darwin universal digest>
          targets:
            - os: darwin
              cpu: arm64
            - os: darwin
              cpu: x64
        - resolution:
            integrity: sha512-<linux-x64 glibc digest>
          targets:
            - os: linux
              cpu: x64
              libcVersion: '2.17'
              napi: 8
        - resolution:
            integrity: sha512-<linux-x64 musl digest>
          targets:
            - os: linux
              cpu: x64
              libc: musl
      fallback:
        integrity: sha512-<primary digest>
```

The lockfile is the same on every machine. No optional node exists for an
installer to drop, and a frozen install on a new platform needs no
resolution. The archive route is rebuilt from the package key, the
tag derived from the variant's `targets`, and the resolution's `revision` when
present (including `0`, for `<name>@<version>+r<revision>/variants/<tag>` and
`<name>@<version>+r<revision>` for `fallback`), so entries stay integrity-only
like other registry entries and variant items keep the `{ resolution, targets }`
shape used by runtime dependencies. A target without `libc` on Linux means
glibc, as in the existing type; a static build lists a glibc target and a musl
target.

A variant's external dependencies are its `dependencies` and
`optionalDependencies` without the names in its `bundleDependencies` (every name
in `dependencies` when it is `true`), which pnpm already leaves out of
snapshots. pnpm resolves them for every variant into the snapshot the resolver
would write for that archive alone, with `dependencies`, `optionalDependencies`,
and `transitivePeerDependencies`, references including peer suffixes. When that
snapshot differs in any field from the version's `snapshots` entry, `snapshots`
records it in full for that variant at `<name>@<version>/variants/<tag>` (with
any peer suffix at the end, as for ordinary snapshot keys):

```yaml
snapshots:
  sharp@0.35.0:
    dependencies:
      detect-libc: 2.1.0
      semver: 7.7.0
  sharp@0.35.0/variants/darwin-arm64:
    dependencies:
      '@img/sharp-libvips-darwin-arm64': 1.3.0
      detect-libc: 2.1.0
      semver: 7.7.0
  sharp@0.35.0/variants/linux-x64:
    dependencies:
      '@img/sharp-libvips-linux-x64': 1.3.0
      detect-libc: 2.1.0
      semver: 7.7.0
  sharp@0.35.0/variants/wasm32:
    dependencies:
      '@emnapi/runtime': 1.7.0
      detect-libc: 2.1.0
      semver: 7.7.0
```

A variant whose resolved snapshot is identical to `<name>@<version>`'s needs no
separate entry in `snapshots`; the installer looks up
`<name>@<version>/variants/<tag>` and falls back to `<name>@<version>`. The
lockfile is still the same on every machine, because every archive's
dependencies are resolved on every machine, as every platform package is in pnpm
today.

Six changes to the lockfile and its interpretation are needed:

- `targets` gain the floor and ABI fields from the tag table.
- The selector ranks the best matching target of every variant at install
  time, independently of serialization order, instead of stopping at the first
    match. Runtime entries, whose variants are `binary` archives, keep
  first-match selection.
- A `fallback` resolution holds the primary archive. It is absent when the
  version has no primary.
- `VariationsResolution` gains an optional `revision` (including `0` for
  revision-aware versions), matching ordinary registry resolutions.
- `snapshots` records `<name>@<version>/variants/<tag>` entries for variants
  whose resolved snapshot differs in any field from `<name>@<version>`'s.
- Snapshot keys gain the `/variants/<tag>` part, before any peer suffix, as
  in `sharp@0.35.0/variants/linux-x64(react@18.3.1)`. A version has no `/`,
  so the part is unambiguous, but every parser of snapshot keys must accept
  it.

Tools that walk the lockfile, such as `pnpm why`, `pnpm audit`,
`pnpm licenses`, and SBOM generation, visit the variant snapshots too, so a
dependency that only one variant declares is reported.

A lockfile that records variants keeps `lockfileVersion: '9.0'`. Only pnpr's
package variants would need a new version, so none is added.
`VariationsResolution` and its targets reject unknown fields today, so pnpm
v12 releases that predate this RFC refuse to read an entry with a `fallback`
field, or any other field they do not know, instead of installing the wrong
archive. Under `--frozen-lockfile` the install stops. Without it, those
releases report the lockfile as broken and resolve again. pnpm v11 does not
refuse the entry. Either one can link the primary on a fresh resolve and
write the lockfile back without variants, so one teammate on an older pnpm
can remove every variant from the repository. That is accepted while pnpr is
experimental.

### Install

- The installer computes the host's platform, selects a variant, and extracts
  it as the package. `bin` entries point directly at the variant's executable,
  so CLIs need no spawning shim.
- `supportedArchitectures` decides which variants are fetched into the store,
  for `pnpm fetch`, Docker images, and Lambda bundles: every variant that
  matches one of its platforms, and the primary when no variant can be linked
  for one of those platforms under the linking rule below. For a platform other
  than the host, a variant matches when its `os`, `cpu`, and `libc` match,
  whatever its floors and ABI, unless the platform states them: fetching an
  archive the eventual host cannot use only costs disk, while skipping the one
  it needs makes an offline install fail. Its list form names each platform; its
  `os`, `cpu`, and `libc` form stands for every combination of its values. Only
  one variant can be linked under a package's name. The linked one is the
  host's, or, if `supportedArchitectures` lists exactly one platform and omits
  `current`, that platform's, chosen by the linking rule: a variant whose floor
  or ABI that platform does not state is not linked, so a Lambda bundle built on
  another machine names its target's floors, such as `linux-x64-manylinux_2_26`.
- For a package with variants, the installer walks the selected variant's
  snapshot (`<name>@<version>/variants/<tag>`, falling back to
  `<name>@<version>`), or `<name>@<version>` when the primary archive is
  selected. A package reachable only through archives that were not selected is
  neither fetched nor linked, the same walk that skips another platform's
  optional packages today. `supportedArchitectures` fetches the dependencies of
  every archive it fetches.
- The linked variant depends on the host, so the virtual store directory of a
  package with variants includes the selected archive's integrity in addition
  to the existing resolved dependency-graph and peer-context identity. Archive
  integrity alone does not identify dependencies: the same archive's ranges
  may resolve differently in two projects or after an update. A global virtual
  store shared by an arm64
  Node.js and an x64 Node.js under Rosetta, or by two Node.js versions that
  select different `nodeAbi` variants, then keeps both.
- The installer records which archive it linked for each package with
  variants, and relinks the package when the selection changes: a change to
  `supportedArchitectures`, a `node_modules` mounted into a musl container,
  or a new host Node.js for a selection that depends on `napi` or `nodeAbi`.
  The lockfile and the package key do not change in any of these cases, so
  nothing else tells the installer that the linked archive is stale.
- The entry's facts about the primary, such as `requiresBuild`, `bins`, and
  its sizes, describe only the primary. A client uses each variant's own entry
  for the archive it links.
- Some tools need another platform's build next to the host's: Prisma's
  `binaryTargets`, electron-builder packaging for another OS, a Lambda bundle
  built on macOS. They can read the variants fetched for
  `supportedArchitectures` from the store through an API that this RFC leaves
  open; they are not linked under the package's name.
- Server-side resolution through pnpr already produces platform-neutral
  lockfiles. It returns the variants it resolved in the same lockfile
  version, so no format negotiation or capability is needed. Never silently
  downgrade to the primary.

### Publication

A version and its variants are published together in one request, a
single-package publish or a commit to pnpr's batch publish endpoint, so all
archives of the version become visible in one transaction. Publishing
variants depends on [atomic workspace publication](./0000-atomic-workspace-publication.md): today's
[journaled batch publish](https://pnpm.io/pnpr/ecosystems#one-publish-for-a-workspace-that-spans-ecosystems)
carries every archive's bytes in its body, which cannot hold a release of
this size, and a commit request that refers to archives uploaded before it
is part of that RFC.

One request body cannot carry a whole release: esbuild's 26 variants are about
140 MB once base64-encoded in an npm publish body, over a 100 MB limit, and
biome's 10 are 310 MB before encoding. As specified in
[atomic workspace publication](./0000-atomic-workspace-publication.md), each
archive is uploaded in its own request to the content-addressed blob endpoint
(`PUT /-/pnpr/v0/artifacts/blob?integrity=...`, with `HEAD` skipping blobs
already stored) before the batch commit request. Limits apply to each archive,
not to the release. The batch publish commit request references the primary and
variant archives by integrity without carrying archive bytes in the commit body,
and runs the invariants above against the complete set before committing the
release. In CI, each matrix job uploads the archive it built from its own
runner, and a final release job sends the commit request that references them
all. No archive moves between jobs, and no upload is visible before the
commit.
[Publishing with pnpm](#publishing-with-pnpm) describes how pnpm builds and
sends these requests.

When the release job publishes with provenance, it signs one statement for
the version, with the digest of every archive it publishes as a subject; SLSA
statements allow several subjects. pnpr rejects a version with a statement
unless the statement names every archive the publisher sent, so no archive
enters the version outside it. A version published without provenance has no
statement to check.
This is what npm provenance proves today: that a workflow of a given
repository and commit published these bytes. It does not prove that a job of
that commit built each archive, the same gap npm has for any package built in
one job and published in another. The version's trust level comes from that
one statement, and a generated primary, which pnpr builds itself, is not a
subject of it.

A build attestation from each matrix job, naming its archive's digest and the
same repository and commit, would close that gap. It is optional evidence
that a later revision of this protocol can accept and rank, not a
requirement. Build jobs already upload their archives to pnpr, so they
could upload their attestations beside them.

### Publishing with pnpm

pnpm publishes a version and its variants from one package directory. The
publisher declares every archive of the version in `package.json`, under
`publishConfig.variants`: one entry per platform variant, and one entry that
targets `any` for the archive that every other host installs. From that
declaration `pnpm pack` and `pnpm publish` build one archive per entry, write
each platform variant's `variant` field, and run the checks pnpr runs.
`pnpm publish` then sends all the archives in one publication. When the
archives are built on several machines, such as by the jobs of a CI matrix,
each machine uploads its own archive, and a final `pnpm publish` commits them
all.

#### Declaring variants

`publishConfig.variants` is an array with one entry per archive. Each entry
lists where its archive runs, as `targets`, and says what the archive adds to
the package:

```json
{
  "name": "example-native-addon",
  "version": "2.3.0",
  "main": "index.js",
  "files": ["index.js"],
  "publishConfig": {
    "variants": [
      {
        "targets": ["any"],
        "files": ["lib/**"]
      },
      {
        "targets": ["linux-x64-manylinux_2_17-napi8"],
        "files": ["example-native-addon.linux-x64-gnu.node"]
      },
      {
        "targets": ["linux-x64-musl-napi8"],
        "files": ["example-native-addon.linux-x64-musl.node"]
      },
      {
        "targets": ["darwin-arm64-macos_11_0-napi8"],
        "files": ["example-native-addon.darwin-arm64.node"]
      },
      {
        "targets": ["win32-x64-napi8"],
        "files": ["example-native-addon.win32-x64-msvc.node"]
      }
    ]
  }
}
```

The archives of this version contain:

| Archive | Files |
| --- | --- |
| `any` | `index.js`, `lib/**` |
| `linux-x64-manylinux_2_17-napi8` | `index.js`, `example-native-addon.linux-x64-gnu.node` |
| `linux-x64-musl-napi8` | `index.js`, `example-native-addon.linux-x64-musl.node` |
| `darwin-arm64-macos_11_0-napi8` | `index.js`, `example-native-addon.darwin-arm64.node` |
| `win32-x64-napi8` | `index.js`, `example-native-addon.win32-x64-msvc.node` |

Each also contains `package.json` and the README and license files, which
npm and pnpm always pack.

An entry has three fields:

- `targets`: a non-empty array. Each target is the tag of one platform, as
  [Platform tags](#platform-tags) defines it, such as `linux-x64-musl` or
  `darwin-arm64-macos_11_0`, or the reserved target `any`. The publisher
  never writes a variant's tag. pnpm derives it as this RFC does, by sorting
  the targets by their bytes and joining them with `+`. That tag is the
  variant's name in routes, in the lockfile, in the publish request, and in
  the file names of packed archives (see [Packing](#packing)).
- `files`: patterns in the syntax of the `files` field of `package.json`,
  relative to the directory that is packed (`publishConfig.directory` when it
  is set).
- `manifest`: fields that replace the field of the same name in the base
  manifest (described below) as a whole. A `null` value removes the field.
  This is the rule that the `manifest` of a variant entry in pnpr's document
  follows.

The entry that targets `any` describes the version's primary archive: the
archive that a host installs when no platform variant matches it. It is also
the archive that npm's documents give as `dist.tarball`, so npm, Yarn, Bun,
and every client without variant support installs it. `any` is reserved: it
is never a platform tag, and the vocabulary will not define one by that
name. At most one entry targets `any`, and that entry has no other target.

A declaration without an `any` entry publishes a version with no primary
archive. Such a version has only platform variants. pnpm fails on a host that
none of them matches, with an error that lists the published platforms, as an
OCI image index does for a platform it lacks, and pnpr generates the minimal
archive that npm's documents point to (see [Model](#model)). That archive
installs nothing that runs. pnpm therefore warns when a declaration has no
`any` entry: a package that should stay installable everywhere declares one.

One target names its archive wherever pnpm takes one on the command line:
`pnpm pack --variant linux-x64-musl-napi8`,
`pnpm publish --upload-variant linux-x64-musl-napi8`, and
`--variant-integrity linux-x64-musl-napi8=<integrity>`. Any one of an
entry's targets names it, because no target appears in two entries. `any`
names the `any` entry in the same way, as in `pnpm pack --variant any`. pnpm
prints a platform variant's derived tag beside the target in its output and
in its JSON. The `any` archive has no derived tag, so its output and JSON
name it by the target `any` alone. An error about an entry shows the entry by
its targets as written.

The package's top-level `files` field keeps its usual meaning, and with
variants it lists what every archive shares. Each entry's `files` adds that
archive's own files. pnpm lists each archive's files as `pnpm pack` does
today, from the top-level `files`, and adds every file that a `files` pattern
of an entry matches, even when the top-level `files` leaves the file out. A
file that one of these patterns matches goes only into the archives of the
entries whose patterns match it. Every file that no pattern claims is shared
and goes into every archive, the `any` archive included. A pattern that
matches no file fails the pack of its archive, so a missing build output
never publishes a variant without its binary.

As npm does, every archive also contains `package.json`, the README and
license files, and the files that its own manifest's `main`, `browser`, and
`bin` name. pnpm reads those fields from the archive's manifest, after its
entry's `manifest` is applied, so a variant whose `bin` names its executable
does not also get the shim that the base manifest's `bin` names.

What the top-level `files` lists depends on the package:

- A native addon with a shared loader lists the loader, as
  `example-native-addon` lists `index.js`.
- A package of native executables shares nothing. It sets `"files": []`,
  which npm and pnpm both read as "only what is always packed", and gives
  any JavaScript shim to its `any` entry.

Without a top-level `files`, a package packs every file that is not ignored,
and with variants every such file that no entry claims ships in every
archive: a build directory, test fixtures, or another platform's binary that
a pattern missed. pnpm warns when a declaration has platform variants and
the package has no top-level `files` field. The rule depends only on the
manifest, so the same package warns, or does not, on every machine and in
every CI job. A rule that warned about large unclaimed files would depend on
what the build left in the directory and on a size threshold, and a file
just under it would ship once per platform without a word. The warning
protects disk use and download size for every installer, and costs the
publisher one field, which also documents what the archives share.

The **base manifest** is the manifest that pnpm publishes for the package
today: `publishConfig` overrides hoisted, `workspace:` and `catalog:`
specifiers replaced, publish lifecycle scripts stripped.
`publishConfig.variants` is removed from it. Each archive's `package.json` is
the base manifest with its entry's `manifest` applied. A platform variant's
also gets a `variant` field that lists its entry's targets in tag order. The
`any` archive gets none, as the primary archive has none.

Specifiers in a `manifest` go through the same `workspace:` and `catalog:`
replacement as the base manifest. pnpm rejects the declaration before it packs
anything when:

- a `manifest` sets `name`, `version`, `peerDependencies`,
  `peerDependenciesMeta`, `variant`, or `publishConfig`. Every archive shares
  the first four (see [Invariants](#invariants-the-registry-enforces)), pnpm
  writes `variant` from the entry's targets, and `publishConfig` holds the
  declaration, which no archive publishes;
- the `manifest` of an entry other than `any` sets `os`, `cpu`, or `libc`;
- the base manifest has `os`, `cpu`, or `libc`, including fields hoisted from
  `publishConfig`. Every platform variant would inherit the fields, which
  pnpr rejects. An `any` archive that npm clients should only install on some
  platforms declares them in its own entry's `manifest`.

pnpm does not remove these fields for the publisher. A platform package's
`os` and `cpu` that are left in its manifest after a migration are reported
as an error, not dropped without a notice.

A version with variants usually needs no manifest difference at all. An
addon whose loader tries a local `.node` file first works unchanged in every
variant, as the [Prototype](#prototype) found for napi-rs packages. In the
example above, `index.js` loads the host's `.node` file and falls back to
`lib/`, which only the `any` archive contains. A CLI shares nothing and
points `bin` at its executable:

```json
{
  "name": "example-cli",
  "version": "4.0.0",
  "bin": { "example-cli": "bin/example-cli.js" },
  "files": [],
  "publishConfig": {
    "variants": [
      {
        "targets": ["any"],
        "files": ["bin/**", "lib/**"]
      },
      {
        "targets": ["linux-x64", "linux-x64-musl"],
        "files": ["native/linux-x64/example-cli"],
        "manifest": { "bin": { "example-cli": "native/linux-x64/example-cli" } }
      },
      {
        "targets": ["darwin-arm64", "darwin-x64"],
        "files": ["native/darwin-universal/example-cli"],
        "manifest": { "bin": { "example-cli": "native/darwin-universal/example-cli" } }
      },
      {
        "targets": ["win32-x64"],
        "files": ["native/win32-x64/example-cli.exe"],
        "manifest": { "bin": { "example-cli": "native/win32-x64/example-cli.exe" } }
      }
    ]
  }
}
```

The second entry is one static Linux build for glibc and musl hosts, and pnpm
names it `linux-x64+linux-x64-musl`; the third is a macOS universal binary,
`darwin-arm64+darwin-x64`. `"files": []` shares nothing, so each native
archive holds its executable, `package.json`, and the README and license
files. The `any` entry claims the JavaScript shim `bin/example-cli.js` and
its portable implementation in `lib/`, so only the `any` archive has them.
Without the `any` entry, the version would have no primary, and the shim
would be in no archive at all: no entry would claim it, `"files": []` would
not share it, and every archive's `bin` would name its executable.

#### Checks before upload

pnpm runs pnpr's checks on every archive before it uploads anything. pnpm and
pnpr share one implementation of platform tags, so the two cannot disagree on
what parses. pnpr can still reject a publication that passes these checks,
but only for what the client cannot see, such as ownership or a version that
already exists.

For each entry of `publishConfig.variants`, pnpm checks that:

- `targets` is a non-empty array, and each target is `any` or parses as a
  platform tag. Only the canonical form parses: fields in the vocabulary's
  order, glibc written as no part, floors written as `<major>_<minor>`
  without leading zeros, and no field that this pnpm does not know. The
  error spells out the canonical form when there is one, such as `linux-x64`
  for the napi-rs triple `linux-x64-gnu`. A tag with a field from a later
  vocabulary is rejected and the error asks for a newer pnpm, because a
  client that cannot parse the tag cannot check it;
- no target appears twice in the entry, and an entry that targets `any` has
  no other target;
- the derived tag of a platform variant is at most 128 bytes long. pnpr
  refuses a longer tag, because a tag is one segment of a route and part of
  an attachment's file name. pnpm refuses it before packing; a variant that
  needs more platforms than fit is split into several variants, never
  shortened or hashed into another name.

Across the entries, pnpm checks that no target appears in two entries, so at
most one entry targets `any`, and that no two platform variants can share the
highest score on any host under [Selection](#selection).

For each archive that pnpm packs, in the command that publishes or in a job
that uploads one archive, pnpm checks that:

- its `name` and `version` are the version's;
- its `peerDependencies` and `peerDependenciesMeta` equal the base
  manifest's, as every archive's must;
- the `any` archive declares no `variant`. Each platform variant declares a
  `variant` whose platforms are its entry's targets, and no `os`, `cpu`, or
  `libc`;
- no platform variant requires a build. pnpm decides this as pnpr does: from
  install scripts and from a `binding.gyp` that is not ignored. The `any`
  archive may require one. A `binding.gyp` is shared unless an entry's
  pattern claims it, so a package that keeps a source build as its fallback
  lists `binding.gyp` and the sources in its `any` entry's `files`, and its
  `install` script in that entry's `manifest`.

#### Packing

`pnpm pack` writes the `any` archive under the file name it uses today for a
package's archive. It writes each platform variant next to it, with `~<tag>`
before `.tgz`, as in `example-native-addon-2.3.0~linux-x64-musl-napi8.tgz` or
`example-cli-4.0.0~linux-x64+linux-x64-musl.tgz`. The publish request uses
the same name for the variant's attachment, so the derived tag is the only
name a variant has, and a retry derives the same names. A long package name
and a long tag can together exceed the 255 bytes that common file systems
allow for one file name. pnpm checks the whole name before it packs and
refuses a variant whose file name would be longer, naming the package and the
entry's targets.

`pnpm pack --variant <target>` packs only the named archives, and can be
repeated. The patterns of entries that are not packed do not have to match
anything, so a build job of a CI matrix packs its own archive without the
other platforms' binaries. Their files are still left out of the archive it
packs. With `--json`, `pnpm pack` lists each archive with its tag, targets,
integrity, size, and files. `prepack` and `postpack` run once around all the
archives of the package.

Packing is deterministic, as today: the same files give the same bytes, and
so the same integrities, on every machine.

#### Sending a version with variants

Variants are published only to a registry declared with `serverType: pnpr`.
pnpm builds the publish document of the version as today, with the `any`
archive as the version's primary archive: its attachment is the usual
`<basename>-<version>.tgz`, its manifest is `versions[<version>]`, and pnpr
records it as the primary and serves it as `dist.tarball`. The `any` archive
has no `~` suffix: `any` is a name in the declaration only, and never
appears in the request. pnpm adds one entry in `_attachments` per platform
variant, named `<basename>-<version>~<tag>.tgz` after the primary's
attachment name, with the variant's derived tag, `+` included. The entry
carries the archive's `integrity` and `length` and no `data`.

These attachments are the whole declaration of the variants. The version
entry has no field that lists them. pnpr finds a version's variants from its
`~<tag>` attachments. It verifies each archive against the attachment's
integrity: a staged archive by the integrity that the attachment references,
and an inline one, which pnpm never sends but another client may, by hashing
its `data`. It reads each archive's platforms from its `variant` field and
rejects the version when they do not give the tag in the attachment's name.
It also rejects a `~<tag>` attachment whose name gives a version that the
document does not publish.

Attachments belong to their package's document, so a batch commit carries
each version's variants in that version's document, and the batch stays one
transaction. The whole-request digest that pnpr keys idempotent retries by
covers the attachments, so a retry with the same archives is still the same
request. pnpr's own package document keeps its `variants` field as described
in [Metadata](#metadata): pnpr derives it, and the publish request never
sends it.

pnpm stages every archive it sends, the `any` archive included, by its
integrity before it sends the document. The archive that pnpr generates for a
version without an `any` entry is not sent, so it is not staged and does not
count toward the limits below. It sends `HEAD`, then `PUT` to
`/-/pnpr/v0/artifacts/blob?integrity=…` when the archive is not staged yet.
Each attachment then references its own staged archive. Variant archives are
never sent inline: the limit of one request body is why staging exists. pnpr
holds at most 256 staged archives, and 2 GiB, for one account, so one request
can reference at most that many. pnpm fails before it uploads when a package,
or a batch, has more than 256 archives or more than 2 GiB of them in total.
Archives that the account staged earlier also count, which pnpm cannot see,
so pnpr can still refuse an upload; a failed staging publishes nothing. pnpm
does not fall back to inline archives as it does for a batch without
variants.

- `pnpm publish` of one package sends the document in its usual
  `PUT <registry>/<name>` request. pnpr accepts staged archives there too.
- `pnpm publish -r --batch` puts every package's document into the batch
  commit to `/-/pnpr/v0/publish`. Every version of the release becomes
  visible together with all of its variants. The batch is one transaction of
  one organization's hosted catalog, and pnpr rejects a batch that spans
  organizations, as in
  [atomic workspace publication](./0000-atomic-workspace-publication.md).
- `pnpm publish -r` without `--batch` sends one request per package. Each
  version still becomes visible together with its variants, but the release
  as a whole is not atomic.

For a version without an `any` entry, `versions[<version>]` is the base
manifest with no `dist`, and the document has no primary attachment. pnpr
builds the generated archive itself.

#### Uploading variants from several machines

Variants are usually built on several machines, one per platform. Each
machine uploads the archive it built, and one final job commits the version.
No archive moves between jobs, and nothing is visible until the commit.

`pnpm publish --upload-variant <target>` runs on the machine that built the
variant:

1. It packs the variant that `<target>` names, exactly as
   `pnpm pack --variant <target>` would.
2. It runs every check of [Checks before upload](#checks-before-upload) that
   the one archive allows, against the package's `publishConfig.variants`.
3. It applies the rule of
   [Registries that do not serve variants](#registries-that-do-not-serve-variants).
4. It uploads the archive with `PUT /-/pnpr/v0/artifacts/blob?integrity=…`.
5. It prints `<target>=<integrity>`, the value that the commit takes, one
   line per archive, with the target it was given. Its standard output holds
   only these lines, so a CI step can capture it, and its progress output
   names a platform variant's derived tag. With `--json`, it prints the
   package's name and version, the targets, and the integrity of each
   archive, and the derived tag of each platform variant.

The option can be repeated, for a machine that builds several archives, and
`--upload-variant any` uploads the `any` archive like any other. The command always sends the `PUT`,
without asking `HEAD` first. Uploading bytes that are already staged only
renews them, so a rerun of the job is safe and restarts the archive's
expiry. The upload publishes nothing: a staged archive is not readable
through any registry route, and it joins a version only when a commit claims
it. With `--dry-run`, the command packs, checks, and prints the integrity,
and uploads nothing.

The option is part of `pnpm publish`, not of `pnpm pack`, because it needs
what publishing resolves: the registry from `publishConfig`, the credentials
for it, and the OIDC exchange of trusted publishing. It is named for what it
does, an upload. It is not called `--stage-variant`, because
`pnpm stage publish` already means npm's staged publishing, where a version
waits for approval. A staged archive is not such a version.

The commit runs in a release job after every upload job:

```sh
pnpm publish --variant-integrity linux-x64-musl-napi8=sha512-… …
```

`--variant-integrity <target>=<integrity>` is repeated once per variant, and
any one of the variant's targets names it, `any` included. With `-r`, each
value names its package first, as `<name>~<target>=<integrity>` or
`<name>~any=<integrity>`. Targets never contain `~`, so the last `~`
separates the two.

The values are short strings, so CI job outputs carry them from the upload
jobs to the release job without artifacts (see [Example](#example)). Before
it sends the commit, pnpm checks that:

- every entry of `publishConfig.variants`, the `any` entry included, has
  exactly one integrity, and no value names a target that no entry declares.
  Two values that name the same entry by different targets are an error. A
  matrix job that failed or was skipped is caught here, by name. A package
  with any `--variant-integrity` takes every archive from them, and pnpm
  packs none of its archives, so the `any` archive is uploaded by a job of
  its own, or by a step of the release job, like every other;
- each archive is still staged. It sends `HEAD` on the staging endpoint for
  each integrity, and uses the `Content-Length` that pnpr returns as the
  attachment's `length`. pnpr reports an archive that expires within the
  hour as missing. A missing archive fails the publish before the commit
  with an error that names the variant by its targets, its integrity, and
  the job to run again.

pnpm attaches each staged platform variant under its entry's derived tag,
and the `any` archive as the primary attachment, with its integrity as
`dist.integrity`.

The commit is then the request of
[Sending a version with variants](#sending-a-version-with-variants), and it
references every archive by integrity. The version appears complete or not
at all. pnpr reads each staged archive's `variant` field, name, version, and
peers, and runs every invariant on the complete set before it commits. An
archive packed for another package or version, or one that declares other
platforms than its entry's targets, is therefore rejected even though the release job
never saw its bytes. Nothing in an archive names the source commit it was
built from, so an archive of the right version built from another commit
passes these checks; only build provenance, discussed below, could bind it to
a commit.

The release job does not have the `any` archive's bytes either. It derives
`versions[<version>]` from the declaration as packing would, and sends only
`dist.integrity`. pnpr's
[manifest verification](./0000-publish-manifest-verification.md) checks that
manifest against the staged archive, as it does for every publish.

Staged archives belong to the account that uploaded them, and pnpr keeps one
for 24 hours after its last upload. The upload jobs and the release job must
therefore publish as the same account, and the release must commit within a
day of the oldest upload. With trusted publishing, every job of the workflow
exchanges its OIDC token for the same workload binding, so they are one
account. When the accounts differ, `HEAD` finds no archive for the release
job's account, and pnpm fails before the commit. The error says that staged
archives belong to the account that uploaded them, and that the upload jobs
must use the release job's credentials. If an archive expires between the
check and the commit, pnpr refuses the commit as referencing an archive that
is not staged, and nothing is published.

When every variant can be built on one machine, as for a Go or Rust tool
that cross-compiles, or for a release job that has all the binaries, plain
`pnpm publish` packs, stages, and commits in one command, as
[Sending a version with variants](#sending-a-version-with-variants)
describes.

#### Registries that do not serve variants

Unless `--no-variants` is given, pnpm refuses to publish a package that
declares or is given variants to a registry that is not declared with
`serverType: pnpr`. This includes a pnpr
deployment that the configuration does not declare as one. The error names
the registry and the setting.

Publishing only the `any` archive instead would be worse. The variant set is
part of the immutable version, so a version published without its variants
cannot get them later from any registry, and the publisher would have to
publish a new version to repair the release. Until then, every user of that
version gets the `any` archive: a slower portable build, a source build that
needs `allowBuilds`, or only an error. A publisher who means to publish the
`any` archive alone says so with `--no-variants`. An example is publishing to npm during a
migration, beside the separate platform packages that npm clients still
install (see [Clients that do not read variants](#clients-that-do-not-read-variants)).
`--no-variants` without an `any` entry is an error, since there is nothing
to publish.

#### Dry runs, provenance, and retries

`pnpm publish --dry-run` packs every archive it would pack, runs every
check above that needs no request, computes the integrities, and applies the registry rule above. It then
prints each archive's tag, platforms, size, and integrity. It sends no
request, not even `HEAD`, and signs nothing. Variants given with
`--variant-integrity` are listed by tag and integrity only. The JSON summary gains a
`variants` list for each package with the same fields. A dry run fails
wherever the publish would fail before its first request, except for the
checks that need one: whether each archive given with `--variant-integrity`
is still staged, belongs to the account, and has not expired is checked only
by the publish, with `HEAD`.

With `--provenance`, pnpm signs one statement for each version it publishes,
as [Publication](#publication) requires. Its subjects are that version's
archives: the `any` archive's `.tgz`, as the primary, named by the version's
canonical npm PURL as
npm provenance names it (a scope's `@` written `%40`, as in
`pkg:npm/%40example/widgets@1.2.0`), and each variant's `.tgz`, named by the
same PURL with a `variant` qualifier whose value is the variant's tag, as in
`pkg:npm/example-native-addon@2.3.0?variant=linux-x64-musl`. PURL qualifier
values are percent-encoded, and `+` is always written `%2B`, never left as is,
because a query string can read a bare `+` as a space:
`linux-x64+linux-x64-musl` is
`pkg:npm/example-cli@4.0.0?variant=linux-x64%2Blinux-x64-musl`. Without an
`any` entry, the subjects are the platform variants. The release record's `purl` and its
variants' `purl` values are built the same way, so each names an archive by
the same PURL (see [release records](./0000-release-records.md#npm)). The
statement goes in the version's usual inline `.sigstore` attachment.

The job that commits signs the statement, so it covers every archive by
digest, including archives that other jobs built and uploaded. For those, the
subject's digest is the integrity given with `--variant-integrity`, which is
the `.tgz` SHA-512 that npm provenance names. The statement attests that this
workflow run published these archives, not that the committing job built
them. npm provenance makes the same claim for a package built in one job and
published in another. Staging narrows the claim: a commit can only claim
archives that its own account uploaded. Under trusted publishing, that account
is the workflow's binding, so every archive came from a job that holds the
workflow's identity. pnpr checks the contents of each archive against the
version before it commits.

A matrix job could also sign SLSA build provenance on its own runner, with
`--upload-variant --provenance`. That statement would name the variant's
`.tgz` and the job that built it, would be uploaded beside the archive, and
would be attached to the version at the commit. That is the build attestation
that [Publication](#publication) leaves to a later revision. This RFC
recommends the single statement of the committing job as the requirement for
a version's trust level. It is what npm clients and pnpr check today, and it
needs nothing new from either. Per-variant build statements stay optional
evidence: they need a staging route for statements, a field of the commit
that names them, and a rank in the trust model. Until then,
`--upload-variant` refuses `--provenance`.

`pnpm publish --batch` accepts `--provenance`. It signs one statement per
version of the batch, naming only that version's primary and variants, and
puts it in the `.sigstore` attachment of that package's document, so each
statement is checked against its own version as it would be in a
single-package publish.

A publish can be retried by running the same command again:

- When staging fails, nothing has been published. The next run uploads only
  the archives that pnpr does not hold staged yet. An upload job that runs
  again uploads its archive again, which renews it.
- When the response to the commit is lost, an identical request gets pnpr's
  recorded outcome (see
  [atomic workspace publication](./0000-atomic-workspace-publication.md)).
  A retry is identical when its archives are byte-identical, which
  deterministic packing and archives given by integrity ensure, and when it
  signs nothing.
- A retry that signs provenance again sends a new statement, so it is a new
  request, and pnpr reports the version as already published. pnpm then reads
  the version's release record and its publication evidence. It reports the
  version as published only when all of these hold:
  - the record has the same set of variant tags as the request, and the
    `.tgz` that it names as the input of the primary (the `any` archive) and
    of each variant is
    the one the request would have sent;
  - when this run signs provenance, the version has a provenance statement
    whose subjects are the primary's `.tgz`, when the version has a primary,
    and every variant's `.tgz`. Each subject is named by that archive's PURL
    and carries the digest of the `.tgz` that the record names as the
    archive's input. This is the comparison that pnpr runs at publication
    (see [Publication](#publication)), and a client verifying the record
    runs it as [release records](./0000-release-records.md#statements-in-toto-in-dsse)
    describe;
  - the dist-tag the request would set points at the version.

  For a version without an `any` entry, the record has no top-level
  `integrity` and names no primary input, as
  [release records](./0000-release-records.md#npm) specify. The generated
  archive that npm's documents point to is built from the variants, so pnpm
  compares the variants only. In any other case pnpm
  fails and names what differs: an archive, a missing statement, or the
  dist-tag. Rerunning a release job is safe this way, with or without
  provenance.

#### Migrating to variants

The declaration is built so that the layouts that packages use today become
variants by moving files and adding entries:

- **One package per platform** (esbuild, `@swc/core`, napi-rs packages). The
  release job already gathers every platform's build, for napi-rs with
  `napi artifacts` into `npm/<platform>/`. It instead puts the builds into the
  main package's directory and declares one entry per platform package. The
  entry's `targets` restate that package's `os`, `cpu`, and `libc` as
  platform tags, one per combination it actually supports.
  The `optionalDependencies` on the platform packages and the platform
  packages' `os` and `cpu` go away. napi-rs loaders keep working because they
  try the local file first. A CLI sets `bin` to its executable and drops the
  postinstall that swapped the shim. During the migration, the platform
  packages can still be published to npm, and the `any` archive with
  `--no-variants`.
- **prebuildify and node-gyp-build.** `prebuilds/<platform>-<arch>/` stays
  where it is, and each entry claims its directory, or its files when one
  directory holds a glibc and a musl build. node-gyp-build finds the one
  prebuild that the variant contains. The source build becomes the fallback:
  an entry that targets `any` claims `binding.gyp` and the sources, and its
  `manifest` holds the `install` script.
- **node-pre-gyp and prebuild-install.** The binaries they download from a
  release host are built by the same CI matrix. The matrix packs them as
  variants instead of uploading them, and the install-time download goes
  away. An `any` entry keeps the source build, as above.

#### Example

`example-native-addon` 2.3.0 is an N-API addon built with napi-rs. It has a
pure-JavaScript fallback for every other host. Its `package.json` is the one
in [Declaring variants](#declaring-variants). Its `index.js` loads
`./example-native-addon.<triple>.node` for the host and falls back to the
pure-JavaScript `lib/` when that file is missing. Five CI jobs each pack one
archive and upload it from their own runner: four build a platform variant,
and one packs the `any` archive, which needs no build. A release job then
commits the version:

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - target: any
            runner: ubuntu-latest
          - target: linux-x64-manylinux_2_17-napi8
            runner: ubuntu-latest
          - target: linux-x64-musl-napi8
            runner: ubuntu-latest
          - target: darwin-arm64-macos_11_0-napi8
            runner: macos-latest
          - target: win32-x64-napi8
            runner: windows-latest
    name: build (${{ matrix.target }})
    runs-on: ${{ matrix.runner }}
    permissions:
      id-token: write
    outputs:
      any: ${{ steps.upload.outputs.any }}
      linux-x64-manylinux_2_17-napi8: ${{ steps.upload.outputs.linux-x64-manylinux_2_17-napi8 }}
      linux-x64-musl-napi8: ${{ steps.upload.outputs.linux-x64-musl-napi8 }}
      darwin-arm64-macos_11_0-napi8: ${{ steps.upload.outputs.darwin-arm64-macos_11_0-napi8 }}
      win32-x64-napi8: ${{ steps.upload.outputs.win32-x64-napi8 }}
    steps:
      - uses: actions/checkout@v5
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - if: matrix.target != 'any'
        run: pnpm run build:native ${{ matrix.target }}
      - id: upload
        shell: bash
        run: |
          value=$(pnpm publish --upload-variant ${{ matrix.target }})
          echo "${{ matrix.target }}=$value" >> "$GITHUB_OUTPUT"
  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - uses: actions/checkout@v5
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: >-
          pnpm publish --provenance
          --variant-integrity "${{ needs.build.outputs.any }}"
          --variant-integrity "${{ needs.build.outputs['linux-x64-manylinux_2_17-napi8'] }}"
          --variant-integrity "${{ needs.build.outputs['linux-x64-musl-napi8'] }}"
          --variant-integrity "${{ needs.build.outputs['darwin-arm64-macos_11_0-napi8'] }}"
          --variant-integrity "${{ needs.build.outputs['win32-x64-napi8'] }}"
```

Each matrix leg is named by its variant's target, and sets only the output of
that name, so the outputs of the five jobs do not overwrite one another.
Platform tags are valid output names. Each value is `<target>=<integrity>`, as
`--upload-variant` prints it, and is split at its first `=`, because platform
tags never contain one. The musl variant is cross-compiled on the Ubuntu runner, as
napi-rs builds it, so every job can use `bash`. The jobs exchange their OIDC
tokens for the same trusted-publisher binding, so the archives they upload
belong to the account that the release job commits as. A matrix job that
fails stops the workflow before the release job, which `needs` every one of
them, so nothing is committed. If the release step names an output that no
matrix job sets, as when a matrix entry is removed but its line is not, the
value is empty, and pnpm fails before the commit, naming the variant that has
no integrity.

The release job checks with `HEAD` that the five archives are still staged,
then sends one publish request with a provenance statement that has five
subjects. The `any` archive, which holds `index.js`, `lib/`, and
`package.json` without any `.node` file, is the version's primary archive.
The document references the archives only through its attachments:

```json
{
  "_attachments": {
    "example-native-addon-2.3.0.tgz": { "integrity": "sha512-<tgz digest>", "length": 4210 },
    "example-native-addon-2.3.0~linux-x64-musl-napi8.tgz": { "integrity": "sha512-<tgz digest>", "length": 812345 }
  }
}
```

(The other variants' attachments and the `.sigstore` attachment are left out.)
pnpr reads each variant's `variant` field from the staged archive and derives
its tag. pnpm on an Alpine host then links the musl variant, and pnpm on
FreeBSD links the `any` archive. npm, which reads npm's documents, installs
the `any` archive as `dist.tarball` and gets the JavaScript fallback.

The same package can also be published from one machine that has all four
`.node` files: `pnpm publish` packs the five archives itself.

### Clients that do not read variants

npm, Yarn, and Bun read npm's documents, which list no variants, and install
the primary archive (or, when the version has no primary, the generated minimal
primary archive of the selected revision, at the URL and integrity that
`dist.tarball` and `dist.integrity` give). So does pnpm with a registry that does
not serve pnpr's format. For esbuild or biome that primary has no binary, so
those users get a package that fails when it runs. Until something else exists,
a publisher whose users are not all on pnpm and pnpr needs a primary that works
everywhere, such as a wasm or JavaScript build, or keeps publishing separate
platform packages.

A compatibility view for those clients is out of scope for now. It could
come later without changing pnpr's format: npm's documents would list
generated platform packages under reserved names in the primary's
`optionalDependencies`, each serving one variant's archive with its `os`,
`cpu`, and `libc`, the layout npm clients read today.

## Migration

Popular packages, checked against their October 2026 releases on npm:

| Package | Today | As variants |
| --- | --- | --- |
| esbuild, `@biomejs/biome`, oxlint, dprint, `@ast-grep/cli`, `@typescript/native-preview`, `@tauri-apps/cli`, `@openai/codex`, deno, bun, workerd | 6–26 platform packages and a `bin` shim; several have a postinstall that swaps the shim for the executable | Variants whose `bin` is the executable. The postinstall goes away. The JavaScript is under 3 MB, so each variant can carry it. |
| `@swc/core`, lightningcss, oxc-parser, `@rspack/binding`, `@napi-rs/canvas`, `@parcel/watcher` | napi-rs or prebuild addon packages; a `try`/`catch` loader | Variants with the addon, selected by `napi`. The `wasm32-wasi` variants of `@rspack/binding` and `unrs-resolver` keep depending on their wasm runtime; `@tailwindcss/oxide`'s already bundles it. |
| rollup, rolldown, `@tailwindcss/oxide` | Addon packages, plus a separate `@rollup/wasm-node` or a `wasm32-wasi` package | Variants. rollup's JavaScript (3.0 MB) is larger than its addon (2.7 MB), so a separate binding package with variants avoids doubling each download. |
| next | `@next/swc-*`, up to 110 MB each, under a 192 MB package | Variants of a `@next/swc` package that next depends on. next itself stays one archive. |
| sharp | `@img/sharp-*` addons, each depending on `@img/sharp-libvips-*`, plus `wasm32` packages | Variants of sharp, each depending on its existing `@img/sharp-libvips-*` package, so libvips keeps its separate license and release cadence and need not migrate in the same release. It can later become a package with variants and an empty primary. The `wasm32` variant depends on `@emnapi/runtime`. |
| esbuild, `@bufbuild/buf`, `@vscode/ripgrep`, turbo, `@sentry/cli` | Linux packages without `libc`, which npm installs on musl too | One variant listing a glibc and a musl platform, plus `android` for turbo. |
| sass-embedded | Platform packages, and fallback packages selected by negated `os` and `cpu` that depend on `sass` | Variants, and a primary that depends on `sass`, which only hosts that no variant matches install. |
| better-sqlite3 12.x, canvas, sqlite3, bcrypt, re2, node-pty | prebuild-install or node-gyp-build downloads or bundles, with a source build as fallback | Variants by `napi`, or by `nodeAbi` for packages not on N-API; the primary keeps the source build behind `allowBuilds`. |
| onnxruntime-node | One 307 MB archive with every platform, plus a postinstall for GPU providers | CPU builds as variants, so a host downloads only its own. GPU providers do not fit; see below. |
| electron | A platform build downloaded outside the package | Variants of the platform build. Packaging for another OS needs the store API above. |

Some do not fit, and should keep separate packages:

- **Builds for the same platform that differ in features.** node-llama-cpp
  ships CPU, CUDA, Vulkan, and Metal builds for one OS and CPU, and picks at
  run time. Hugo ships standard and `extended` builds. The vocabulary has no
  accelerator dimension, and a variant cannot be chosen at run time.
- **Assets versioned apart from the package.** Puppeteer's Chrome and
  Playwright's browsers are pinned by browser build, shared across projects in
  a user cache, and hundreds of megabytes each. A browser could be its own
  package with variants, but that is a change in those tools, not a migration.

### Tools outside npm

Executables now distributed only as GitHub release archives match the model:
gh, uv, ruff, golangci-lint, shellcheck, actionlint, jq, ripgrep, fd,
protoc, helm, typst, hugo, gitleaks, trivy, k6, hadolint, and shfmt. Each
release has one archive per platform, built by a CI matrix. Publishing to pnpr
would replace the per-tool definitions of the
[`tool:` protocol](../../text/0000-tool-protocol.md), and give them one
publication, integrity in the lockfile, provenance, and a proxy.

What they need from this RFC:

- variants with several platforms, since most of their Linux builds are static
  and protoc and CMake ship macOS universal binaries;
- versions with no primary;
- a variant archive with only an executable and a generated `package.json`,
  since these projects do not maintain one. A small package directory whose
  variants claim the extracted executables, as in
  [Publishing with pnpm](#publishing-with-pnpm), already gives that. Extracting
  `.zip`, `.tar.xz`, or a bare executable is left to the release job.

Two limits remain. Versions must be semver, so tools such as jq (`jq-1.8.2`)
need a mapping. And several tools ship no Windows or macOS arm64 build, so
those hosts would need emulation, which is an unresolved question below.

## Prototype

A prototype implemented the pnpr publication checks, `variants` in pnpr's
package document and release records, the variant route, and the pnpm
resolution, lockfile, and install parts, without per-archive blob upload, the
floor and ABI fields, `wasm32`, or dependencies that differ between archives:
it enforced an earlier, stricter rule that every archive declares the same
dependencies. It also took each variant's tag and platforms from the publish
request rather than from the `variant` field, recorded the tag on each
lockfile variant and the dependencies of all archives in one snapshot rather
than per variant, and checked the variant's `os`, `cpu`, and `libc` against
them. Real packages from npm were converted into a primary and variants and
published to a local pnpr, then installed by real projects:

| Scenario | Result |
| --- | --- |
| esbuild 0.28.2 (26 variants), `@biomejs/biome` 2.6.4 (10), typst 0.15.1 from GitHub releases (5) | `bin` links the native executable directly. The JavaScript shims, platform packages, and esbuild's postinstall are gone. |
| A Vite 8 app (rolldown with 15 variants, lightningcss) and a Tailwind 4 CLI app (`@tailwindcss/oxide`, lightningcss) | Both build. The napi-rs loaders of rolldown, oxide, and lightningcss already try a local `.node` file first, so their variants needed no JavaScript change. esbuild needed one line. |
| The same lockfile, frozen, with `supportedArchitectures` set to glibc, musl, darwin-arm64, win32-x64, freebsd-x64, and an unknown OS | Each gets its variant; the unknown OS gets the primary. The lockfile does not change. |
| Resolution with a different `supportedArchitectures`, and resolution by pnpr's server-side resolver | The lockfile is byte-identical to a local resolution on the host. |
| The musl variants and the static Linux variants in Alpine containers | biome, typst, esbuild, and the Tailwind build run. typst's musl-only build serves the glibc host through one variant with two platforms. |
| `@tailwindcss/oxide` with its `wasm32-wasi` build as the primary and the wasm runtime bundled | An unmatched host scans through the wasm build. Native hosts install none of the runtime packages. |
| esbuild, biome, typst, the Tailwind app, and server-side resolution through pnpr's package document, with only large documents split and with every document split | Each installs, builds, and runs with the same lockfile, and the darwin-arm64 frozen install gets its variant. With every document split, the variants come from 16 release records. npm's documents list no variants, so pnpm without pnpr's format links the primary. |
| Split packuments, resolving a 787-package app with an empty cache | 73% to 89% faster resolution, and the same lockfile; see [Metadata](#metadata). |
| Publishing variants that break each invariant | Each is rejected with a reason, including a variant with an extra dependency under the earlier rule. A bundled extra dependency is accepted. A variant added to a published version is rejected. |

The prototype found these problems, now addressed above:

- Two projects that selected different variants of one version shared one
  global virtual store slot, so the darwin project got the Linux binary. The
  slot hash now includes the selected archive's integrity.
- A frozen install after a change to `supportedArchitectures` kept the
  previously linked variant in a local virtual store.
- A pnpm release that predates the lockfile change rewrote the lockfile
  without variants on a non-frozen install.
- npm installed the primary, and esbuild and biome failed at run time.
- `pnpm fetch` with two platforms fetched only the host's variant, so an
  offline install for the other platform failed.
- A single publish request carrying every variant exceeded the body limit,
  and biome's timed out.
- The primary's `bins` and `requiresBuild` would have been applied
  to the variant on a fresh install.

## Rationale and Alternatives

1. **Lock every optional variant (the status quo in pnpm).** It fixes the
   lockfile, but keeps the metadata fan-out, the non-atomic releases, the extra
   names, the loader shims, and the double meaning of optional. It cannot help
   packages that download in install scripts.
2. **Distributions that point to other packages**, as in npm's
   [Package Distributions RFC](https://github.com/npm/rfcs/pull/519) and Yarn's
   [package variants proposal](https://github.com/yarnpkg/berry/issues/2751).
   Both select another *package* by condition and link it in place. That keeps
   one package per platform, so it keeps separate owners, separate publications,
   and the fan-out. npm's RFC also listed dependencies of distribution packages
   as its main known risk: a client learned them only after fetching the
   package it selected. Here every archive's dependencies are in the version's
   own document, checked by the registry, and resolved together into the
   lockfile.
3. **One archive with every platform inside**, the NuGet `runtimes/<rid>/`
   approach. It needs no registry change, but every install downloads every
   platform's binaries. For packages such as `@swc/core` that multiplies the
   download by more than ten.
4. **Variants as overlays on the primary archive.** It saves duplicating the
   JavaScript in each variant, but makes extraction a merge with conflict rules.
   Duplicated JavaScript costs little next to the native binary, and where it
   does not, as in next and rollup, the native code can move to its own
   package with variants.
5. **Declaring variants in a top-level `package.json` field.** Installers and
   other tools read top-level fields, and pnpr reads the archive's own
   `variant` field. A `variants` field next to it would ship in every archive
   with publisher paths in it and be easy to confuse with `variant`.
   `publishConfig` already holds what only the publisher reads, and pnpm
   removes the declaration from what it publishes.
6. **Declaring variants in a separate file**, such as a
   `pnpm-variants.yaml` beside `package.json`. It would be one more file that
   `pnpm publish -r` has to discover in each package, with a second format for
   facts about one package. It would also sit apart from the `files` and
   `publishConfig` settings that decide what goes into the archives.
7. **Declaring variants with CLI flags only.** Flags cannot say which package
   of a recursive publish each variant belongs to, and the release command
   becomes the only record of what a version contains. Flags remain for what
   only the release job knows: the integrities that the upload jobs reported
   (`--variant-integrity`).
8. **A directory per variant**, with its own `package.json`, like today's
   `npm/<platform>/` packages. Each directory would repeat the shared files or
   need overlay rules, and each `package.json` would repeat `name`,
   `version`, and the peers that must stay equal. Claiming files with
   patterns keeps one package root and one manifest. prebuildify's
   `prebuilds/` layout and napi-rs's `.node` files fit it without moving
   anything.
9. **A map keyed by the variant's tag**, such as
   `"variants": { "linux-x64+linux-x64-musl-napi8": { … } }`, with a reserved
   `primary` key for the fallback. Keys cannot repeat, but the publisher has to write the
   registry's identifier: the targets sorted by their bytes, joined with `+`,
   with every floor spelled out. That is easy to get wrong for a variant with
   several targets, and a reordering is an error rather than the same
   variant. An array of entries lets the publisher write the targets, while
   pnpm derives the tag and checks for overlaps as pnpr does.
10. **A separate field for the primary archive**, such as
    `publishConfig.primary` beside the variants, with `"primary": false` for
    a version without one. The primary archive has the same `files` and
    `manifest` as any variant and is packed, checked, uploaded, and named on
    the command line the same way, so a separate field duplicates the entry
    kind and needs its own name in every command, its own "absent" and
    "false" cases, and its own rules in every check. An entry that targets
    `any` expresses all of it as one more entry: no `any` entry is a version
    without a primary, and `--variant any` names it as any target names its
    variant. Other ecosystems model the fallback the same way, as one more
    artifact in the same list: a Python release lists its `py3-none-any`
    wheel beside its platform wheels, a RubyGems version has a `ruby`
    platform gem beside its native gems, NuGet resolves the `any` runtime
    identifier as the last fallback of its runtime graph, and conda publishes
    `noarch` packages in the same channel as platform builds.
11. **A main artifact with attached artifacts**, as Maven publishes one main
    jar and attaches classified artifacts to it. The main artifact is a
    different kind from the attachments, so it has to exist even when it is
    only an error, and the attachments are found by classifier strings that
    the publisher chooses. A version with only platform builds, such as a Go
    or Rust CLI, has no honest main artifact. Here the `any` entry is
    optional, and every other archive is named by the platforms it runs on.
12. **A field of the version entry that lists the variants**, such as a map
    from each variant's tag to the integrity of its `.tgz`. Every variant is
    already an attachment named by its tag, which carries its integrity, and
    every archive's `package.json` already declares its targets in its
    `variant` field. The map repeats both, so a request could disagree with
    itself, and pnpr would have to reject the disagreement. A field in the
    version entry also shares its namespace with the publisher's own
    `package.json` fields, so a package whose manifest happened to use the
    name would collide with the declaration. The attachments say everything
    the map said.
13. **Each machine publishing its own variant into the version.** A version
    would be live, and installable, before its last variant arrived. An
    install in that window would lock the primary or another variant for a
    host whose variant was still missing. The same `name@version` would then
    resolve to different archives before and after, which the lockfile and
    the immutable version are meant to rule out. It would also contradict
    pnpr's rule that variants are accepted only together with their version's
    primary, and the immutable release record, which covers every archive of
    the version once. Staging from each machine and committing once keeps the
    uploads parallel and the publication atomic.
14. **Moving archives between jobs as CI artifacts** to a release job that
    uploads them all, with an option such as `--variant-archives <glob>`.
    The archives cross the CI provider's storage
    twice for nothing, the release job has to trust files it did not build,
    and every release pays for the transfer of the largest archives. With
    staging, each archive goes from the runner that built it straight to
    pnpr, and the release job passes only short integrities. A build machine
    that cannot reach the registry can still hand its build output, the
    binary, to a machine that can. Packing is deterministic, so that machine
    produces the same archive, and `--variant-archives` adds nothing.
15. **Publishing only the `any` archive to a registry without variants.** This
   produces a version that can never get its variants. See
   [Registries that do not serve variants](#registries-that-do-not-serve-variants).

## Implementation

Target pnpr and pnpm v12. Do not add a feature to pnpm v11.

1. Write the platform tag specification, including the derivation of tags,
   with test vectors, shared with the
   shared side-effects cache.
2. pnpr: store variants with the version, enforce the invariants, compute
   `variants` in its own package document and release records,
   and serve the variant routes, including under revisions.
3. pnpr: accept variants in batch publish (referencing archives uploaded by
   integrity).
4. pnpm: read `variants` from pnpr's package document, write
   `variations` entries with the new
   target fields and `fallback`, select at install, and fetch variants for
   `supportedArchitectures`.
5. pnpm: read `publishConfig.variants`, with its reserved `any` target for
   the primary archive, pack and check each archive, and
   publish a version with its variants through the single-package and batch
   publish requests, staging every archive, as in
   [Publishing with pnpm](#publishing-with-pnpm). Add `pnpm pack --variant`,
   `pnpm publish --upload-variant`, `--variant-integrity` with its checks
   that every declared variant is given and still staged, `--no-variants`,
   the provenance
   subjects of variants with the `variant` PURL qualifier, provenance in
   `pnpm publish --batch` with one statement per version, and the retry
   check against pnpr's document.
6. pnpr: accept a version with variants and no primary archive, generating its
   primary, and check that the provenance statement names every archive.
   Accept the variant's tag, `+` included and at most 128 bytes, as its
   attachment label (`<basename>-<version>~<tag>.tgz`), and find a version's
   variants from those attachments instead of from the `pnprVariants` field
   of the version entry, which goes away. Reject a version whose archive's
   `variant` field gives another tag than the one that names it, and a
   `~<tag>` attachment whose name gives a version that the document does not
   publish. Add each variant's `purl` to the
   release record. Let workload credentials of trusted publishing upload to
   and query the staging endpoint and send the batch commit; today they may
   only send `PUT` to a configured package. Accept a staged primary with only
   `dist.integrity` in its version document.

Tests must cover selection order with floors and `wasm32`, an unknown tag field
falling back to the primary, a frozen install on a platform that did not resolve
the lockfile, a variant with a dependency the primary lacks (linked only where
that variant is selected), a name declared with two specifiers by two archives
(each variant's snapshot resolves its own), a variant whose peer dependencies
differ (rejected), a variant that requires a build (rejected), a variant archive
without a `variant` field or with an `os`, `cpu`, or `libc` field (rejected),
npm's documents of a version with variants (no variants listed, the primary
installed), a static Linux variant selected on a musl host, a version without a
primary on an unmatched host, two variants that list the same platform
(rejected), a batch publish with a missing variant blob or with a provenance
statement that does not name every archive (rejected), a variant that bundles a
dependency the primary installs (a frozen install links no external package for
it), a variant-only dependency with a peer dependency (the variant's snapshot
records its own `transitivePeerDependencies`), and variants under a registry
revision (including a frozen install rebuilding `+r<revision>` variant and
fallback routes).

Test the publisher with a variant pattern that matches no file (rejected before
upload), a file claimed by the `any` entry and left out of every variant, an
`any` entry combined with another target or declared twice (rejected), a
declaration without an `any` entry (a warning, and a version without a primary),
platform variants without a top-level `files` field (a warning), `"files": []`
with a `bin` that a variant replaces (each native archive holds its executable
and no shim), `pnpm pack --variant any` (the archive npm clients install, with
no `variant` field), a publish request whose variants are only `~<tag>`
attachments (accepted with no field in the version entry), a `~<tag>` attachment
whose tag differs from its archive's `variant` field or whose version the
document does not publish (rejected), a retried request with the same
attachments (the recorded outcome), a batch whose documents each carry their own
variants (one transaction), a non-canonical target (rejected with the canonical
spelling), a target that appears twice in one entry or in two entries (rejected,
the error naming the entries by their targets), targets written in any order
(the same derived tag), `--upload-variant` and `--variant-integrity` naming a
variant by any one of its targets, two `--variant-integrity` values naming one
entry (rejected), `os` in the base manifest (rejected), a variant that inherits
an `install` script or a `binding.gyp` (rejected), `--upload-variant` on its own
(one archive staged, nothing published, the same integrity printed on a rerun,
which renews the archive), `--variant-integrity` values that miss a declared
variant or name an undeclared one (rejected before the commit), a staged archive
that has expired or was uploaded by another account (rejected before the commit
with an error naming the variant), an `any` archive given by integrity, a
publish to a registry not declared with `serverType: pnpr` (refused, also in a
dry run), a derived tag over 128 bytes (refused before packing), `--provenance`
with `--batch` (one statement per version, naming only its archives, with `+` in
a tag written `%2B`), `--no-variants` to such a registry for a package that
declares variants (only the `any` archive is published), a dry run that sends no
request, and a rerun after a lost commit response with and without provenance
(reported as published when the integrities match).

Also test the A/B multi-target example above on both hosts after reversing
lockfile array order, and two projects selecting identical archive bytes but
different versions of a ranged dependency. Their virtual store entries must
preserve the different resolved graphs. Cover equal constraint counts with an
absent versus present libc floor, numeric floor ordering, exact ABI versus N-API
overlaps, implicit versus explicit glibc, and rejected score ties. Also cover
`supportedArchitectures` naming another platform without floors: every variant
matching its `os`, `cpu`, and `libc` is fetched, none with a floor is linked
for it, and the primary is fetched and linked when no variant without a floor
matches it.

## Prior Art

- PyPI: one release holds many wheels named with
  [platform tags](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/)
  (PEP 425, 600, 656). The installer orders the tags the host supports and
  `pylock.toml` lists every wheel with its hash. pnpm's Python support already
  works this way.
- RubyGems: platform gems under one version, and a `PLATFORMS` section in
  `Gemfile.lock`.
- Maven classifiers attach several artifacts to one version.
- [pnpm runtimes](https://pnpm.io/cli/runtime) already use the `variations`
  lockfile resolution for per-platform Node.js archives.
- [npm Package Distributions RFC](https://github.com/npm/rfcs/pull/519), closed
  unmerged in 2023.

## Unresolved Questions and Bikeshedding

- Should a host select a variant for an emulated CPU, such as an `x64`
  executable on macOS `arm64` under Rosetta or on Windows `arm64`? It works for
  executables but never for addons, which load into the host's Node.js
  process, so the variant would have to declare that it is an executable.
- Should the vocabulary gain an accelerator dimension (CUDA, ROCm, Vulkan,
  Metal), as Python's WheelNext wheel variants proposal does? Such a
  dimension cannot be detected as reliably as the OS and CPU.
- How should `pnpm deploy` and `node_modules` copied into a container choose
  the linked variant when several platforms are prepared?
- Should large assets that are not platform builds, such as a Chromium
  download, use the same mechanism or a separate attached-assets RFC?
- Can pnpr fold the existing esbuild-style pattern of proxied npm packages into
  a variant view to save the metadata fan-out, even though the linked layout
  must stay unchanged for those packages?
