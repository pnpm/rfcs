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

## Detailed Explanation

### Acceptance

For a hosted npm publication, pnpr:

1. Verifies the submitted archive integrity and validates the archive under
   bounded decoding and entry-count limits.
2. Reads the manifest that the supported installer would select. Rejects
   duplicate or ambiguous manifest entries and unsafe archive paths.
3. Derives install-critical fields under a named interpretation profile.
4. Compares the submitted and archive-derived install-critical fields. A
   mismatch rejects the publication with the package and differing fields named.
5. Stores the archive-derived manifest as the accepted metadata for the
   version, together with the exact byte digests of the archive manifest and
   shrinkwrap, if present.

Metadata responses for the accepted version are generated from the stored
archive-derived manifest, not from the submitted document.

### Install-critical fields

Install-critical fields include:

- `dependencies`, `optionalDependencies`, `peerDependencies`, and
  `peerDependenciesMeta`.
- `bundleDependencies` / `bundledDependencies`.
- `os`, `cpu`, `libc`, and `engines`.
- `bin` and `directories.bin`.
- Lifecycle-script presence (`hasInstallScript`) and the scripts that imply it.
- Shrinkwrap presence and contents.

The exact field set, and how absent and empty values compare, must be fixed
before ratification.

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
   worth proposing separately. It complements this RFC: it detects the problem
   after resolution, while publish-time verification prevents it from being
   published.
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
shrinkwrap presence.

## Prior Art

- [The massive hole in the npm ecosystem](https://blog.vlt.sh/blog/the-massive-hole-in-the-npm-ecosystem),
  the public description of manifest confusion.
- [npm publication implementation](https://github.com/npm/cli/blob/latest/workspaces/libnpmpublish/lib/publish.js).
- [Integrity-addressed patch revisions](./0000-integrity-addressed-patch-revisions.md),
  which already keeps revision-specific manifests.

## Unresolved Questions and Bikeshedding

- What is the complete install-critical field set, including absent-versus-empty
  rules and shrinkwrap treatment?
- Should verification be mandatory immediately, or opt-in per hosted registry
  for an initial period?
- What diagnostics help publishers repair a rejected package?
- How should operators handle existing hosted versions that fail re-derivation?
