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

### When it runs

The check runs after a package's contents are available and before any
dependency code executes: before lifecycle scripts, and before the package is
linked where its code could be loaded. It applies whether the contents were
just downloaded or come from the content-addressable store. For store hits,
`package.json` is read from its content-addressed file, not from metadata cached
alongside the store entry.

### On mismatch

The install fails, naming the package, version, and differing fields. An
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
installs. Measure how often mismatches appear across a sample of popular
packages before choosing the default.

## Prior Art

- [The massive hole in the npm ecosystem](https://blog.vlt.sh/blog/the-massive-hole-in-the-npm-ecosystem),
  the public description of manifest confusion.
- [Publish-time manifest verification for pnpr](https://github.com/pnpm/rfcs/pull/47).

## Unresolved Questions and Bikeshedding

- Should a mismatch fail by default, given that historical mismatches exist on
  the public npm registry? A corpus measurement should decide this.
- What is the setting name and shape for the per-package allowance?
- Which install-critical fields does the lockfile need to record so frozen
  installs can compare all of them?
