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
- zero or more **variants**, each an archive, a **tag** that names it, and the
  list of **platforms** it runs on.

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
variant matches, with an error that lists the published platforms, and skips
the package when it is an optional dependency. npm clients get a primary that
pnpr generates: a `package.json` with the version's name, version, and the
union of the variants' `os` and `cpu`, so npm reports an unsupported platform
instead of installing an empty package.

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
  `peerDependencies`, `peerDependenciesMeta`) equal the primary's, after
  removing from each archive the names it bundles (`bundleDependencies`), so
  the dependency graph does not depend on the platform;
- no variant requires a build (`dist.pnpr.requiresBuild` is false), so a
  variant never needs an `allowBuilds` entry;
- every variant's `os`, `cpu`, and `libc` fields, if present, include each of
  its platforms;
- no two variants have the same tag, and no two variants list the same
  platform with the same floors, so no host can tie between them.

A variant may differ from the primary in its files, bundled dependencies,
`bin`, `main`, `exports`, and `engines`. The primary may require a build: a
source build is a fallback, and it needs an `allowBuilds` entry only on hosts
that link the primary.

Bundling is how a fallback tier carries a runtime the other tiers do not need.
napi-rs `wasm32-wasi` packages depend on `@napi-rs/wasm-runtime` and `@emnapi/*`
(about 4 MB unpacked); `@tailwindcss/oxide-wasm32-wasi` already bundles them,
while `@rspack/binding-wasm32-wasi` and `@unrs/resolver-binding-wasm32-wasi`
declare them as dependencies. sass-embedded's fallback depends on the
pure-JavaScript `sass` (6 MB). Declared as dependencies of every archive, those
would be installed on every host. Bundled into the `wasm32` variant or the
primary, they are installed only where that archive is linked.

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

A variant's tag is its name in routes and the lockfile. For a variant with one
platform it is that platform's tag; for one with several, the publisher picks a
name in the same character set, such as `linux-x64-static` or
`darwin-universal`.

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
of guessing. pnpr's [shared side-effects cache](https://pnpm.io/pnpr/shared-side-effects-cache#platform-compatibility)
already applies the same rule to its `pnpm:v1:` tags. The two vocabularies
should be one specification with shared test vectors.

### Selection

For a host, the client keeps the variants with a platform whose every field
matches, then picks one in this order, comparing each variant by its best
matching platform:

1. Variants with more matched fields, without `wasm32`.
2. Among those, the highest `libcVersion`, then the highest `osVersion`, then
   the highest `napi`.
3. A `wasm32` variant.
4. The primary archive, if the version has one.

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
          "tag": "linux-x64-manylinux_2_17",
          "platforms": [
            { "os": "linux", "cpu": "x64", "libc": "glibc", "libcVersion": "2.17", "napi": 8 }
          ],
          "integrity": "sha512-<variant tar digest>",
          "tarball": "https://registry.example/fast-thing@1.4.0/variants/linux-x64-manylinux_2_17",
          "unpackedSize": 9123456,
          "fileCount": 4,
          "bins": { "fast-thing": { "path": "bin/fast-thing" } }
        },
        {
          "tag": "darwin-universal",
          "platforms": [
            { "os": "darwin", "cpu": "x64", "osVersion": "11.0" },
            { "os": "darwin", "cpu": "arm64", "osVersion": "11.0" }
          ],
          "integrity": "sha512-<variant tar digest>",
          "tarball": "https://registry.example/fast-thing@1.4.0/variants/darwin-universal",
          "unpackedSize": 18234567,
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
uncompressed tar, as for every pnpr archive. The abbreviated document omits
`tarball`, which the client derives from the route below.

The package's own document grows by what the variant documents carried for
the same versions, minus their repeated names and manifests. Rebuilding
today's data in this shape, gzipped: `esbuild` gains about 0.2 MB in place of
1.0 MB across 26 documents, `@swc/core` 0.6 MB in place of 2.8 MB across 12,
and rollup 0.3 MB in place of 0.9 MB across 27.

A package with per-ABI addons multiplies that. better-sqlite3 12.x attached 145
prebuilds to each GitHub release (105 for Electron ABIs, 40 for Node.js). Such
a package should move to N-API before it moves to variants.

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
        - tag: linux-x64-musl
          resolution:
            integrity: sha512-<linux-x64 musl digest>
          targets:
            - os: linux
              cpu: x64
              libc: musl
        - tag: linux-x64-manylinux_2_17
          resolution:
            integrity: sha512-<linux-x64 glibc digest>
          targets:
            - os: linux
              cpu: x64
              libcVersion: '2.17'
        - tag: darwin-universal
          resolution:
            integrity: sha512-<darwin universal digest>
          targets:
            - os: darwin
              cpu: x64
            - os: darwin
              cpu: arm64
      fallback:
        integrity: sha512-<primary digest>
```

The lockfile is the same on every machine. No optional node exists for an
installer to drop, and a frozen install on a new platform needs no
resolution. The archive route is rebuilt from the package key and the
variant's `tag`, so entries stay integrity-only like other registry entries.
A target without `libc` on Linux means glibc, as in the existing type; a
static build lists a glibc target and a musl target.

Three changes to the existing type are needed:

- A variant gains `tag`.
- `targets` gain the floor and ABI fields from the tag table.
- A `fallback` resolution holds the primary archive. It is absent when the
  version has no primary.

`VariationsResolution` and its targets reject unknown fields today, so
pnpm v12 releases that predate this RFC fail to parse such a lockfile instead
of installing the wrong archive. That holds only under `--frozen-lockfile`.
Without it, those releases report the lockfile as broken, resolve again, link
the primary, and write the lockfile back without variants, so one teammate on
an older pnpm silently removes every variant from the repository. The
lockfile must carry a marker that older releases reject in every mode, such
as a new `lockfileVersion`.

### Install

- The installer computes the host's platform, selects a variant, and extracts
  it as the package. `bin` entries point directly at the variant's executable,
  so CLIs need no spawning shim.
- `supportedArchitectures` decides which variants are fetched into the store,
  for `pnpm fetch`, Docker images, and Lambda bundles: every variant that
  matches a platform in its cross product of `os`, `cpu`, and `libc`, and the
  primary when one of those platforms matches none. Only one variant can be
  linked under a package's name. The linked one is the host's, or, if
  `supportedArchitectures` lists exactly one platform and omits `current`,
  that platform's.
- The linked variant depends on the host, so the virtual store directory of a
  package with variants is keyed by the selected archive's integrity, not only
  by name and version. A global virtual store shared by an arm64 Node.js and an
  x64 Node.js under Rosetta, or by two Node.js versions that select different
  `nodeAbi` variants, then keeps both.
- The installer records which archive it linked for each package with
  variants, and relinks the package when the selection changes: a change to
  `supportedArchitectures`, a `node_modules` mounted into a musl container,
  or a new host Node.js for a selection that depends on `napi` or `nodeAbi`.
  The lockfile and the package key do not change in any of these cases, so
  nothing else tells the installer that the linked archive is stale.
- The primary's `dist.pnpr` facts, such as `requiresBuild`, `bins`, and its
  sizes, describe only the primary. A client uses each variant's own entry
  for the archive it links.
- Some tools need another platform's build next to the host's: Prisma's
  `binaryTargets`, electron-builder packaging for another OS, a Lambda bundle
  built on macOS. They can read the variants fetched for
  `supportedArchitectures` from the store through an API that this RFC leaves
  open; they are not linked under the package's name.
- Server-side resolution through pnpr already produces platform-neutral
  lockfiles. It writes `variations` entries without change to its protocol.

### Publication

Each archive is uploaded in its own request and recorded in a stage, and all
archives of a version become visible in one transaction: the
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

One request cannot carry a whole release. esbuild's 26 variants are about
140 MB once base64-encoded in an npm publish body, over a 100 MB limit, and
biome's 10 are 310 MB before encoding. Limits apply to each archive, not to
the release.

Each archive carries its own provenance statement, from the job that built it.
Approval rejects the set unless every statement names the same source
repository and commit, so a matrix job cannot attach a build of other source.
The trust level of the version is the weakest among its archives, as the
[`tool:` protocol](../../text/0000-tool-protocol.md) defines for tools that
ship as platform archives.

### Clients that do not read variants

npm, Yarn, and Bun ignore `dist.pnpr` and install the primary archive. For
esbuild or biome that primary has no binary, so those users get a package
that fails when it runs. A publisher whose users are not all on pnpm needs
either a primary that works everywhere, such as a wasm build, or a generated
compatibility view: the primary version's npm-visible
`optionalDependencies` lists one reserved name per variant, and each reserved
name serves a single version with the variant's `os`, `cpu`, `libc`, and
archive. Reserved names cannot be published directly. The primary archive
then needs the loader that esbuild-style packages ship today. pnpm ignores
the generated entries when it reads `dist.pnpr.variants`.

A publisher that needs the view cannot migrate until pnpr serves it, so it
ships with the rest, not after it.

## Migration

Popular packages, checked against their October 2026 releases on npm:

| Package | Today | As variants |
| --- | --- | --- |
| esbuild, `@biomejs/biome`, oxlint, dprint, `@ast-grep/cli`, `@typescript/native-preview`, `@tauri-apps/cli`, `@openai/codex`, deno, bun, workerd | 6–26 platform packages and a `bin` shim; several have a postinstall that swaps the shim for the executable | Variants whose `bin` is the executable. The postinstall goes away. The JavaScript is under 3 MB, so each variant can carry it. |
| `@swc/core`, lightningcss, oxc-parser, `@rspack/binding`, `@napi-rs/canvas`, `@parcel/watcher` | napi-rs or prebuild addon packages; a `try`/`catch` loader | Variants with the addon, selected by `napi`. The `wasm32-wasi` packages of `@rspack/binding` and `unrs-resolver` must bundle their wasm runtime. |
| rollup, rolldown, `@tailwindcss/oxide` | Addon packages, plus a separate `@rollup/wasm-node` or a `wasm32-wasi` package | Variants. rollup's JavaScript (3.0 MB) is larger than its addon (2.7 MB), so a separate binding package with variants avoids doubling each download. |
| next | `@next/swc-*`, up to 110 MB each, under a 192 MB package | Variants of a `@next/swc` package that next depends on. next itself stays one archive. |
| sharp | `@img/sharp-*` addons, each depending on `@img/sharp-libvips-*`, plus `wasm32` packages | Two packages with variants: sharp, and a libvips package with an empty primary, so libvips keeps its separate license and release cadence. The `wasm32` variant bundles `@emnapi/runtime`. |
| esbuild, `@bufbuild/buf`, `@vscode/ripgrep`, turbo, `@sentry/cli` | Linux packages without `libc`, which npm installs on musl too | One variant listing a glibc and a musl platform, plus `android` for turbo. |
| sass-embedded | Platform packages, and fallback packages selected by negated `os` and `cpu` that depend on `sass` | Variants, and a primary that bundles `sass`. |
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
  since these projects do not maintain one. The repackaging from `.zip`,
  `.tar.xz`, or a bare executable belongs in the publisher command.

Two limits remain. Versions must be semver, so tools such as jq (`jq-1.8.2`)
need a mapping. And several tools ship no Windows or macOS arm64 build, so
those hosts would need emulation, which is an unresolved question below.

## Prototype

A prototype implemented the pnpr publication checks, `dist.pnpr.variants`, the
variant route, and the pnpm resolution, lockfile, and install parts, without
staged publishing, the floor and ABI fields, `wasm32`, or the compatibility
view. Real packages from npm were converted into a primary and variants and
published to a local pnpr, then installed by real projects:

| Scenario | Result |
| --- | --- |
| esbuild 0.28.2 (26 variants), `@biomejs/biome` 2.6.4 (10), typst 0.15.1 from GitHub releases (5) | `bin` links the native executable directly. The JavaScript shims, platform packages, and esbuild's postinstall are gone. |
| A Vite 8 app (rolldown with 15 variants, lightningcss) and a Tailwind 4 CLI app (`@tailwindcss/oxide`, lightningcss) | Both build. The napi-rs loaders of rolldown, oxide, and lightningcss already try a local `.node` file first, so their variants needed no JavaScript change. esbuild needed one line. |
| The same lockfile, frozen, with `supportedArchitectures` set to glibc, musl, darwin-arm64, win32-x64, freebsd-x64, and an unknown OS | Each gets its variant; the unknown OS gets the primary. The lockfile does not change. |
| Resolution with a different `supportedArchitectures`, and resolution by pnpr's server-side resolver | The lockfile is byte-identical to a local resolution on the host. |
| The musl variants and the static Linux variants in Alpine containers | biome, typst, esbuild, and the Tailwind build run. typst's musl-only build serves the glibc host through one variant with two platforms. |
| `@tailwindcss/oxide` with its `wasm32-wasi` build as the primary and the wasm runtime bundled | An unmatched host scans through the wasm build. Native hosts install none of the runtime packages. |
| Publishing variants that break each invariant | Each is rejected with a reason. A bundled extra dependency is accepted. A variant added to a published version is rejected. |

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
- The primary's `dist.pnpr.bins` and `requiresBuild` would have been applied
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
   as its main known risk. Here a variant is an archive of the same version, so
   the registry can require that it shares the primary's dependencies, and that
   risk disappears.
3. **One archive with every platform inside**, the NuGet `runtimes/<rid>/`
   approach. It needs no registry change, but every install downloads every
   platform's binaries. For packages such as `@swc/core` that multiplies the
   download by more than ten.
4. **Variants as overlays on the primary archive.** It saves duplicating the
   JavaScript in each variant, but makes extraction a merge with conflict rules.
   Duplicated JavaScript costs little next to the native binary, and where it
   does not, as in next and rollup, the native code can move to its own
   package with variants.

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
6. pnpr: the compatibility view for other clients, before any package whose
   users are not all on pnpm migrates.
7. pnpm: a lockfile marker that older releases reject without
   `--frozen-lockfile` too.

Tests must cover selection order with floors and `wasm32`, an unknown tag
field falling back to the primary, a frozen install on a platform that did not
resolve the lockfile, a variant whose dependencies differ from the primary's
(rejected), a `wasm32` variant whose extra dependencies are bundled
(accepted), a variant that requires a build (rejected), a static Linux variant
selected on a musl host, a version without a primary on an unmatched host, two
variants that list the same platform (rejected), a stage approved with a
missing variant or with provenance from another commit, and variants under a
registry revision.

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
- Should a variant be allowed extra dependencies? Every case in
  [Migration](#migration) is covered by bundling or by a separate package with
  variants (sharp's libvips), but a package whose native library is shared by
  several dependents would install it on every platform.
- Should a host select a variant for an emulated CPU, such as an `x64`
  executable on macOS `arm64` under Rosetta or on Windows `arm64`? It works for
  executables but never for addons, which load into the host's Node.js
  process, so the variant would have to declare that it is an executable.
- Should the vocabulary gain an accelerator dimension (CUDA, ROCm, Vulkan,
  Metal), as Python's WheelNext wheel variants proposal does? Such a
  dimension cannot be detected as reliably as the OS and CPU.
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
