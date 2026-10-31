# Install-time manifest consistency check

## Summary

Before any dependency code runs, pnpm compares the install-critical fields of
each package's `package.json`, as read from the package contents, with the
metadata that resolution used. A disagreement stops the install with the
package and differing fields named.

## Motivation

npm registries serve version metadata separately from the archive. pnpm
resolves the dependency graph and decides on lifecycle scripts from that
metadata, then installs the archive contents. Nothing ensures the two agree.
This is commonly called "manifest confusion".

For example, registry metadata for `@example/widgets@1.2.0` declares
`@example/core: ^2.0.0` and no install script. The archive's `package.json`
declares `@example/core: ^1.0.0` and a `postinstall` script. pnpm installs a
graph the package does not declare, and runs a script that policy based on
the metadata never saw.

Publish-time verification in a registry, such as the
[pnpr draft](https://github.com/pnpm/rfcs/pull/47), prevents this for packages
that registry hosts. Most pnpm users install from registries they do not
control. A client-side check protects them regardless of the registry.

This check establishes consistency between metadata and contents. It does not
establish that either is trustworthy or that the package is benign.

## Detailed Explanation

### What is compared

For each package installed from a registry, pnpm reads `package.json` from the
package contents and derives the install-critical fields that resolution and
installation depend on:

- `dependencies`, `optionalDependencies`, `peerDependencies`, and
  `peerDependenciesMeta`.
- `bundleDependencies` / `bundledDependencies`.
- `os`, `cpu`, `libc`, and `engines`.
- `bin` and `directories.bin`.
- Lifecycle-script presence.

It compares them with the metadata resolution used: the registry metadata
during a resolving install, or the lockfile entry during a frozen install.
Fields the lockfile does not record are compared only where the lockfile is
the source of the decision they affect. The derivation and comparison rules
follow the same versioned interpretation as the pnpr draft, so both sides use
one definition and share test fixtures.

### Normalizations and direction

Registry metadata for the public npm registry is the output of the publishing
client's normalization, while the archive keeps the raw `package.json`. The
interpretation applies the same normalizations to both sides before comparing.
The pnpr draft lists them under
[publisher-side normalization](https://github.com/pnpm/rfcs/pull/47).

A survey compared each version's registry metadata with its archive's
`package.json` in three samples: the 1,674 registry packages in pnpm/pnpm's
lockfile, the latest version of the 5,000 most-downloaded packages (from
[`npm-high-impact`](https://www.npmjs.com/package/npm-high-impact); 4 were no
longer on the registry), and one randomly chosen older version of each of the
top 2,000 (1,950 had one). Publishers range from npm 1 to npm 12 and lerna.

| Difference | Lockfile (1,674) | Top 5,000, latest (4,996) | Top 2,000, older (1,950) | Examples |
| --- | --- | --- | --- | --- |
| `bin` syntax only (`./` prefix, string versus object) | 43 | 130 | 37 | `eslint@10.13.0`, `glob@11.1.0` |
| `optionalDependencies` copied into `dependencies` | 3 | 10 | 15 | `jest-haste-map@29.7.0` |
| Empty map versus absent field | 0 | 5 | 14 | `assert-plus@latest` |
| `version` build metadata stripped | 0 | 0 | 10 | lerna canary builds |
| `engines: {"node": "*"}` added | 1 | 2 | 6 | `nopt@1.0.10` |
| `directories.bin` expanded into `bin` | 0 | 3 | 0 | `esbuild-wasm@latest` |
| `gypfile` install script added | 1 | 1 | 2 | `fsevents@2.3.3` ([npm/cli#8714](https://github.com/npm/cli/issues/8714)) |
| Git shorthand expanded to `github:` | 0 | 0 | 1 | `table@2.1.0` |
| Anything else | 0 | 0 | 0 | |
| Packages that differ | 46 (2.7%) | 146 (2.9%) | 81 (4.2%) | |

Without normalization, about 3% of packages would fail, among them `eslint`,
`glob`, and `fsevents`. With the normalizations, none of the 8,620 versions
fail.

The direction of a difference matters:

- **The archive declares more than the metadata**: a dependency, a lifecycle
  script, or a `binding.gyp` that resolution did not see. This is the attack.
  The install fails.
- **The metadata declares more than the archive**: `fsevents@2.3.3` claims a
  `node-gyp rebuild` that its archive cannot run. Nothing hidden executes.
  pnpm follows the archive and does not fail. A dependency that resolution
  added but the archive does not declare is extra, not hidden. pnpm reports
  it as a warning.

### When it runs

The check runs after a package's contents are available and before any
dependency code executes: before lifecycle scripts, and before the package is
linked where its code could be loaded. It applies whether the contents were
just downloaded or come from the content-addressable store. For store hits,
`package.json` is read from its content-addressed file, not from metadata cached
alongside the store entry.

### On mismatch

For a difference in the failing direction, the install fails, naming the
package, version, and differing fields. An
explicit per-package allowance accepts a known mismatch. It binds the package
name, version, and artifact integrity, and resolution then uses the
archive-derived fields. It never patches the resolved graph after the fact.

### Cost

The check reads and parses one small file per package and makes no network
request. Where pnpm already reads `package.json` from package contents, the
check reuses that read.
Frozen installs keep their fast path apart from that comparison.

## Rationale and Alternatives

1. **Registry-side verification only.** Prevents the problem at the source,
   but only for registries that implement it. The public npm registry does not.
2. **Always trust the archive and re-resolve.** Treating `package.json` as the
   truth would make resolution depend on downloading every candidate archive,
   which is much slower. Failing makes the disagreement visible.
3. **Warn instead of failing.** Lower friction, but a warning printed after a
   script has run is too late, and warnings during installs are easy to miss.

## Implementation

Target pnpm v12. Do not add a feature to pnpm v11.

1. Share the install-critical field derivation with pnpr as a versioned
   interpretation with common fixtures.
2. Add the comparison where pnpm reads a package's manifest from fetched or
   store contents, before the build phase.
3. Add the per-package allowance setting and diagnostics.

Tests must cover a dependency present only in the archive, a script present
only in the archive, mismatched platform fields, store hits, and frozen
installs. Each normalization needs a fixture taken from a real package in the
survey, and `fsevents@2.3.3` must install.

## Prior Art

- [The massive hole in the npm ecosystem](https://blog.vlt.sh/blog/the-massive-hole-in-the-npm-ecosystem),
  the public description of manifest confusion.
- [Publish-time manifest verification for pnpr](https://github.com/pnpm/rfcs/pull/47).

## Unresolved Questions and Bikeshedding

- Should a mismatch fail by default, given that historical mismatches exist on
  the public npm registry? The survey found no difference outside the listed
  normalizations across 8,620 versions, including the most-downloaded
  packages and older versions of them, which supports failing by default.
- What is the setting name and shape for the per-package allowance?
- Which install-critical fields does the lockfile need to record so frozen
  installs can compare all of them?
