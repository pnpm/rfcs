# Publish-time manifest verification for pnpr

> Status: draft for discussion. Split from
> [pnpm/rfcs#44](https://github.com/pnpm/rfcs/pull/44).

## Summary

When a package is published to a pnpr-hosted registry, derive its
install-critical metadata from the `package.json` inside the archive and compare
it with the manifest submitted alongside the archive. Reject the publication on
any disagreement. After acceptance, serve only the archive-derived metadata for
that version.

## Motivation

The npm publish format carries version metadata separately from the archive.
The resolver reads the submitted metadata; the installer extracts the archive.
Nothing requires the two to agree, and a tarball checksum alone does not
establish that separately served dependency metadata describes that tarball.
This is commonly called "manifest confusion".

For example, a publisher uploads `@example/widgets@1.2.0`. The submitted
manifest declares `@example/core: ^2.0.0`, while the archive's `package.json`
declares `@example/core: ^1.0.0` and a `postinstall` script the submitted
manifest omits. Clients resolve one dependency graph and install code that
declares another, and a lifecycle script that metadata-based policy never saw.

pnpr controls publication for its hosted registries, so it can close this gap
for every package it hosts without any client or protocol change.

### Evidence

The npm trackers report the symptom rather than the term:

| Report | Year | 👍/comments | What disagreed |
| --- | --- | --- | --- |
| [npm/cli#626](https://github.com/npm/cli/issues/626) | 2019 | 6/1 | `fsevents@1.2.11`: metadata lists `node-pre-gyp` in `dependencies`; the archive lists it only in `bundledDependencies`. |
| [npm/feedback#797](https://github.com/npm/feedback/discussions/797) | 2022 | 2/1 | `node-rdkafka@2.14.0`: the archive's `package.json` says version `v2.14.2-1.HEAD`. "Allowing mismatches could lead to a security issue if e.g. `scripts` is mismatched." |
| [npm/feedback#1057](https://github.com/npm/feedback/discussions/1057) | 2023 | 1/2 | `esbuild@0.19.10`: metadata copies every `optionalDependencies` entry into `dependencies`. |
| [npm/cli#8714](https://github.com/npm/cli/issues/8714) | 2025 | 1/1 | `fsevents@2.3.3`: metadata has `"install": "node-gyp rebuild"`; the archive has no install script and no `binding.gyp`, so npm runs a build that fails. |
| [npm/feedback#804](https://github.com/npm/feedback/discussions/804) | 2022 | 1/0 | "npm registry should not add install scripts into package.json". |
| [npm/cli#2606](https://github.com/npm/cli/issues/2606) | 2021 | 30/23 | `hasInstallScript` missing, so `npm ci` skips a `postinstall`. |
| [npm/cli#4583](https://github.com/npm/cli/issues/4583) | 2022 | 8/5 | `_hasShrinkwrap` missing on third-party registries, so the shrinkwrap is ignored. |
| [npm/cli#9837](https://github.com/npm/cli/issues/9837) | 2026 | 1/2 | `better-sqlite3` sets `gypfile: false`; npm still reports `install: node-gyp rebuild`. |

Most of these mismatches are produced by the publishing client, not by an
attacker; see [Publisher-side normalization](#publisher-side-normalization).

## Detailed Explanation

### Acceptance

For a hosted npm publication, pnpr:

1. Verifies the submitted archive integrity and validates the archive under
   bounded decoding and entry-count limits.
2. Reads the manifest that the supported installer would select. Rejects
   duplicate or ambiguous manifest entries and unsafe archive paths.
3. Derives install-critical fields under a named interpretation profile.
4. Normalizes both the submitted manifest and the archive manifest under the
   profile, then compares their install-critical fields. A mismatch rejects
   the publication with the package and differing fields named, except for
   the one benign direction defined under
   [Publisher-side normalization](#publisher-side-normalization).
5. Stores the archive-derived manifest as the accepted metadata for the
   version, together with the exact byte digests of the archive manifest and
   shrinkwrap, if present.

Metadata responses for the accepted version are generated from the stored
archive-derived manifest, not from the submitted document.

### Install-critical fields

Install-critical fields include:

- `name` and `version`. A mismatch places the package in the wrong slot
  ([npm/feedback#797](https://github.com/npm/feedback/discussions/797)).
- `dependencies`, `optionalDependencies`, `peerDependencies`, and
  `peerDependenciesMeta`.
- `bundleDependencies` / `bundledDependencies`.
- `os`, `cpu`, `libc`, and `engines`.
- `bin` and `directories.bin`.
- Lifecycle scripts (`preinstall`, `install`, `postinstall`) and derived
  `hasInstallScript` (true when any of those scripts is present, or when a root
  `binding.gyp` is in the archive without those scripts and without
  `gypfile: false`). pnpr's `requiresBuild` already derives this from the
  verified archive.
- Shrinkwrap presence and contents.

The exact field set, and how absent and empty values compare, must be fixed
before ratification.

Generated metadata always carries the derived `hasInstallScript` and
`_hasShrinkwrap`. A registry that omits them makes clients skip scripts or
shrinkwraps ([npm/cli#2606](https://github.com/npm/cli/issues/2606),
[npm/cli#4583](https://github.com/npm/cli/issues/4583)).

### Publisher-side normalization

`npm publish` does not upload the raw `package.json`. It uploads the output of
[`@npmcli/package-json`'s `prepare()`](https://github.com/npm/package-json/blob/main/lib/normalize.js),
while the archive keeps the raw file. A strict comparison would reject ordinary
`npm publish` uploads. The steps that change install-critical fields:

| Step | Submitted manifest | Archive `package.json` |
| --- | --- | --- |
| `binRefs`, `binDir` | `"bin": {"c8": "bin/c8.js"}` | `"bin": "./bin/c8.js"` |
| `gypfile` | `scripts.install: "node-gyp rebuild"`, `gypfile: true`, set when any `*.gyp` file is in the source directory, even if `files` excludes it | no install script |
| `bundleDependencies` | `true` expanded to the names in `dependencies` | `true` |
| `normalizeData` | `version` cleaned, for example `v1.2.3` to `1.2.3`; `name` trimmed | as written |

npm 9 and earlier ran `normalize-package-data`, whose
[`addOptionalDepsToDeps`](https://github.com/npm/normalize-package-data/blob/main/lib/fixer.js)
copied every `optionalDependencies` entry into `dependencies`. Metadata
published that way still exists, so a profile that reads historical versions
must treat a name present in both maps with the same range as optional.

The profile defines each of these as a normalization, applied to both sides
before comparison. The `gypfile` step is the one case where the normalized
sides still differ: the submitted manifest claims a build that the archive
cannot run. That direction is benign. pnpr accepts it, serves the
archive-derived verdict, and reports the difference to the publisher. The
reverse direction, where the archive declares a script or dependency that the
submitted manifest omits, is the attack and is always rejected.

A survey compared each version's registry metadata with its archive's
`package.json` in three samples: the 1,674 registry packages in pnpm/pnpm's
lockfile, the latest version of the 5,000 most-downloaded packages (from
[`npm-high-impact`](https://www.npmjs.com/package/npm-high-impact); 4 were no
longer on the registry), and one randomly chosen older version of each of the
top 2,000 (1,950 had one). Publishers range from npm 1 to npm 12 and lerna.

| Difference | Lockfile (1,674) | Top 5,000, latest (4,996) | Top 2,000, older (1,950) |
| --- | --- | --- | --- |
| `bin` syntax only (`./` prefix, string versus object) | 43 | 130 | 37 |
| `optionalDependencies` copied into `dependencies` | 3 | 10 | 15 |
| Empty map versus absent field | 0 | 5 | 14 |
| `version` build metadata stripped (lerna canary builds) | 0 | 0 | 10 |
| `engines: {"node": "*"}` added | 1 | 2 | 6 |
| `directories.bin` expanded into `bin` | 0 | 3 | 0 |
| `gypfile` install script added | 1 | 1 | 2 |
| Git shorthand expanded to `github:` | 0 | 0 | 1 |
| Anything else | 0 | 0 | 0 |
| Packages that differ | 46 (2.7%) | 146 (2.9%) | 81 (4.2%) |

The `gypfile` row is benign in two of its cases (`sharp@0.17.1`,
`environment@0.0.1`): the archive contains `binding.gyp`, so the added script
is what an installer runs anyway. The only version whose metadata claims a
build its archive cannot run is `fsevents@2.3.3`, which appears in two
samples.

Packages published with current npm (10 to 12) still differ in `bin` syntax,
for example `c8@12.0.0`, `eslint@10.13.0`, and `cross-env@10.1.0`.

The profile therefore also normalizes:

- An empty dependency map, `bin` object, or `engines` object equals an absent
  field, as does `engines: {"node": "*"}` added by `normalize-package-data`.
- `directories.bin` expands into `bin` entries.
- `version` compares without semver build metadata, which npm's version
  cleaning drops. pnpm already reserves `+rN` build metadata for
  [registry revisions](./0000-integrity-addressed-patch-revisions.md), so
  whether pnpr should accept a publisher's build metadata at all is an open
  question.
- A hosted-git shorthand dependency equals its `github:` form.

Registry-generated data such as the publisher identity, `dist.tarball`, and
`_`-prefixed fields is not publisher-authored and is not compared. Deprecation
messages remain mutable annotations. A dependency correction uses a new version
or an explicit registry revision; it never rewrites an accepted manifest.

### Versioned interpretation

`npm-install-manifest/1` names a protocol contract, not whatever normalization
the current pnpm release happens to implement. Publish shared fixtures of
input bytes and expected derived fields. pnpr and the pnpm v12 client can share
the Rust implementation, while the fixtures define the behavior.

A later profile that changes derivation does not silently reinterpret existing
versions. pnpr re-derives from the stored archive under the new profile and
stores the result as a new immutable accepted manifest beside the existing one.
It never overwrites accepted metadata. Which profile a response uses is an
explicit selection, and a client that does not support it gets a capability
error.

### Existing hosted versions

Versions published before this check have not been verified. pnpr can
re-derive their manifests from the stored archives and report disagreements to
operators. Whether such versions are then served from archive-derived metadata,
blocked, or left as-is is an operator decision.

## Rationale and Alternatives

1. **Client-side check at install time.** pnpm already extracts the archive and
   could compare its `package.json` with the manifest the resolver used. This
   protects against every registry, including the public npm registry, and is
   proposed in [pnpm/rfcs#50](https://github.com/pnpm/rfcs/pull/50). It
   complements this RFC: it detects the problem after resolution, while
   publish-time verification prevents it from being published.
2. **Silently replace the submitted manifest with the archive-derived one.**
   Avoids rejecting publications, but hides publisher tooling bugs and makes
   the published metadata differ from what the publisher saw.
3. **Warn instead of rejecting.** Lower friction, but leaves the disagreement
   in the registry and pushes the decision to every consumer.

## Implementation

Target pnpr. Do not add a feature to pnpm v11.

1. Fix the install-critical field set and comparison rules. Add fixtures.
2. Extend pnpr's publication staging and archive-reading code to derive and
   compare manifests, reusing the pnpm manifest implementation.
3. Store the archive-derived manifest and input digests with the version.
4. Generate version metadata from the stored archive-derived manifest.

Tests must cover mismatched dependencies, a script present only in the
archive, ambiguous archive manifests, unsafe paths, bounded decoding, and
shrinkwrap presence. They must also cover each normalization above, using
publish documents produced by `npm publish`, `pnpm publish`, and lerna, so an
ordinary publication from a supported client is never rejected.

## Prior Art

- [The massive hole in the npm ecosystem](https://blog.vlt.sh/blog/the-massive-hole-in-the-npm-ecosystem),
  the public description of manifest confusion.
- [npm publication implementation](https://github.com/npm/cli/blob/latest/workspaces/libnpmpublish/lib/publish.js).
- [Integrity-addressed patch revisions](./0000-integrity-addressed-patch-revisions.md),
  which already keeps revision-specific manifests.
- [PEP 658](https://peps.python.org/pep-0658/): PyPI serves the `METADATA`
  file extracted from the wheel, so the served metadata is the artifact's own.
- crates.io generates index entries from the uploaded `.crate`'s `Cargo.toml`.

## Unresolved Questions and Bikeshedding

- What is the complete install-critical field set, including absent-versus-empty
  rules and shrinkwrap treatment?
- Should verification be mandatory immediately, or opt-in per hosted registry
  for an initial period? The survey above found no difference outside the
  listed normalizations across 8,620 versions, which argues for mandatory once
  they are in the profile.
- Does the installer-selected build file stay `binding.gyp`, while npm's
  `gypfile` step matches any `*.gyp`?
- What diagnostics help publishers repair a rejected package?
- How should operators handle existing hosted versions that fail re-derivation?
