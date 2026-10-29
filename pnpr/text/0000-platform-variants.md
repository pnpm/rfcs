# Platform variants: one version, many artifacts

> Status: draft for discussion. No implementation or protocol version is approved.

## Summary

Let one package version carry a primary archive plus platform-specific variant
archives, published together and described in the version's metadata. pnpm
locks every variant in a platform-neutral lockfile entry and links the variant
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
- zero or more **variants**, each an archive plus a **platform tag**.

A variant archive is a complete package for its platform, not an overlay on
the primary. Its `package.json` has the same `name` and `version` as the
primary. Linking a variant means extracting its archive as the package.

The variant set is part of the immutable version. It is fixed when the version
is published, and a [registry revision](./0000-integrity-addressed-patch-revisions.md)
replaces it as a whole.

### Invariants the registry enforces

At publication, pnpr derives each archive's manifest under the
[manifest verification profile](https://github.com/pnpm/rfcs/pull/47) and
rejects the version unless:

- every variant's `name` and `version` equal the primary's;
- every variant's dependency fields (`dependencies`, `optionalDependencies`,
  `peerDependencies`, `peerDependenciesMeta`, `bundleDependencies`) equal the
  primary's, so the dependency graph does not depend on the platform;
- no variant requires a build (`dist.pnpr.requiresBuild` is false), so a
  variant never needs an `allowBuilds` entry;
- no two variants have the same platform tag.

A variant may differ from the primary in its files, `bin`, `main`, `exports`,
and `engines`.

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

Examples: `linux-x64`, `linux-x64-musl`, `linux-x64-manylinux_2_28`,
`darwin-arm64`, `win32-x64`. The string form of the floors and ABI fields is
an unresolved question.

The vocabulary is versioned. A tag with a field the client does not know is
not selectable, so an older client falls back to the primary archive instead
of guessing. pnpr's [shared side-effects cache](https://pnpm.io/pnpr/shared-side-effects-cache#platform-compatibility)
already applies the same rule to its `pnpm:v1:` tags. The two vocabularies
should be one specification with shared test vectors.

### Selection

For a host, the client keeps the variants whose every field matches, then
picks one in this order:

1. Variants with more matched fields, without `wasm32`.
2. Among those, the highest `libcVersion`, then the highest `osVersion`, then
   the highest `napi`.
3. A `wasm32` variant.
4. The primary archive.

If two variants still tie, publication should have rejected the version. The
client fails rather than pick one.

### Metadata

Each version object carries its variants under the pnpr namespace that npm
clients ignore. pnpr computes every field from the verified archive and strips
publisher-supplied `dist.pnpr` fields, as it does today:

```json
{
  "name": "fast-thing",
  "version": "1.4.0",
  "dist": {
    "integrity": "sha512-<primary tar digest>",
    "tarball": "https://registry.example/fast-thing@1.4.0",
    "pnpr": {
      "requiresBuild": false,
      "variants": [
        {
          "platform": { "os": "linux", "cpu": "x64", "libc": "glibc", "libcVersion": "2.17", "napi": 8 },
          "tag": "linux-x64-manylinux_2_17",
          "integrity": "sha512-<variant tar digest>",
          "tarball": "https://registry.example/fast-thing@1.4.0/variants/linux-x64-manylinux_2_17",
          "unpackedSize": 9123456,
          "fileCount": 4,
          "bins": { "fast-thing": { "path": "bin/fast-thing" } }
        }
      ]
    }
  }
}
```

The full and abbreviated documents both carry `dist.pnpr.variants`, so
resolution needs one document per package. `integrity` is the digest of the
uncompressed tar, as for every pnpr archive.

### Archive routes

A variant is served from an immutable route next to the version's own:

```text
<registry-base>/<name>@<version>/variants/<tag>
<registry-base>/<name>@<version>+r<revision>/variants/<tag>
```

Responses follow the revision route's rules: a tar with `gzip` or `zstd`
content coding, no redirects, and integrity checked by the client.

### Lockfile

pnpm records a package with variants using the existing `variations`
resolution, ordered by the selection rule so that its first-match semantics
pick the best variant:

```yaml
packages:
  fast-thing@1.4.0:
    resolution:
      type: variations
      variants:
        - resolution:
            integrity: sha512-<linux-x64 musl digest>
          targets:
            - os: linux
              cpu: x64
              libc: musl
        - resolution:
            integrity: sha512-<linux-x64 glibc digest>
          targets:
            - os: linux
              cpu: x64
              libcVersion: '2.17'
      fallback:
        integrity: sha512-<primary digest>
```

The lockfile is the same on every machine. No optional node exists for an
installer to drop, and a frozen install on a new platform needs no
resolution. The archive route is rebuilt from the package key and the
target's tag, so entries stay integrity-only like other registry entries.

Two changes to the existing type are needed:

- `targets` gain the floor and ABI fields from the tag table.
- A `fallback` resolution holds the primary archive.

`VariationsResolution` and its targets reject unknown fields today, so
pnpm v12 releases that predate this RFC fail to parse such a lockfile instead
of installing the wrong archive.

### Install

- The installer computes the host's platform, selects a variant, and extracts
  it as the package. `bin` entries point directly at the variant's executable,
  so CLIs need no spawning shim.
- `supportedArchitectures` decides which variants are fetched into the store,
  for `pnpm fetch`, Docker images, and Lambda bundles. Only one variant can be
  linked under a package's name. The linked one is the host's, or, if
  `supportedArchitectures` lists exactly one platform and omits `current`,
  that platform's.
- Server-side resolution through pnpr already produces platform-neutral
  lockfiles. It writes `variations` entries without change to its protocol.

### Publication

All archives of a version are published in one transaction: the
[journaled batch publish](https://pnpm.io/pnpr/ecosystems#one-publish-for-a-workspace-that-spans-ecosystems)
today, and the [atomic workspace publication](https://github.com/pnpm/rfcs/pull/46)
transaction when it lands. Variants are usually built by separate CI matrix
jobs, so staged publishing collects them:

```sh
# one job: stage the primary archive
pnpm stage publish
# each matrix job: attach its variant to the stage
pnpm stage add-variant <stage-id> --platform linux-x64-musl ./fast-thing-1.4.0-linux-x64-musl.tgz
# final job: publish the primary and every variant at once
pnpm stage approve <stage-id>
```

Command names are placeholders. Approval runs the invariants above against
the complete set, so a missing or mismatched variant fails the release before
anything is visible.

### Clients that do not read variants

npm, Yarn, and Bun ignore `dist.pnpr` and install the primary archive. A
publisher that wants those clients to get native code can ask pnpr for a
generated compatibility view: the primary version's npm-visible
`optionalDependencies` lists one reserved name per variant, and each reserved
name serves a single version with the variant's `os`, `cpu`, `libc`, and
archive. Reserved names cannot be published directly. The primary archive
then needs the loader that esbuild-style packages ship today. pnpm ignores
the generated entries when it reads `dist.pnpr.variants`.

This view is optional and can ship after the rest.

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
   as its main known risk. Here a variant is an archive of the same version, so
   the registry can require that it shares the primary's dependencies, and that
   risk disappears.
3. **One archive with every platform inside**, the NuGet `runtimes/<rid>/`
   approach. It needs no registry change, but every install downloads every
   platform's binaries. For packages such as `@swc/core` that multiplies the
   download by more than ten.
4. **Variants as overlays on the primary archive.** It saves duplicating the
   JavaScript in each variant, but makes extraction a merge with conflict rules.
   Duplicated JavaScript costs little next to the native binary.

## Implementation

Target pnpr and pnpm v12. Do not add a feature to pnpm v11.

1. Write the platform tag specification with test vectors, shared with the
   shared side-effects cache.
2. pnpr: store variants with the version, enforce the invariants, compute
   `dist.pnpr.variants`, and serve the variant routes, including under
   revisions.
3. pnpr: accept variants in batch publish and staged publishing, and add the
   attach-to-stage operation.
4. pnpm: read `dist.pnpr.variants`, write `variations` entries with the new
   target fields and `fallback`, select at install, and fetch variants for
   `supportedArchitectures`.
5. pnpm: publisher commands to pack a variant and attach it to a stage.
6. Later: the compatibility view for other clients.

Tests must cover selection order with floors and `wasm32`, an unknown tag
field falling back to the primary, a frozen install on a platform that did not
resolve the lockfile, a variant whose dependencies differ from the primary's
(rejected), a variant that requires a build (rejected), a stage approved with a
missing variant, and variants under a registry revision.

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

- The string form of floors and ABI fields in tags, for example
  `darwin-arm64-macos_11_0` or `linux-x64-manylinux_2_17-napi8`.
- Should a variant be allowed extra dependencies, such as sharp's separate
  `libvips` packages, or must every native library be inside the variant
  archive or be its own package with variants?
- Can a variant be added to a published version for a platform not yet
  covered, for example a new Electron ABI, without a revision? It would need
  the same publisher identity and a lockfile refresh rule.
- How should `pnpm deploy` and `node_modules` copied into a container choose
  the linked variant when several platforms are prepared?
- Should large assets that are not platform builds, such as a Chromium
  download, use the same mechanism or a separate attached-assets RFC?
- Can pnpr fold the existing esbuild-style pattern of proxied npm packages into
  a variant view to save the metadata fan-out, even though the linked layout
  must stay unchanged for those packages?
