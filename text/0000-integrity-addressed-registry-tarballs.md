# Integrity-addressed registry tarballs

## Summary

pnpm should recognize revision-aware Subresource Integrity values emitted by a
registry that supports immutable, integrity-addressed tarball URLs:

```text
<registry-base>/-/tarballs/<algorithm>/<base64url-digest>
```

The SRI option distinguishes revision-aware resolutions from legacy registry
resolutions:

```text
sha512-<original-digest>?r0     original artifact
sha512-<first-patch-digest>?r1  first replacement
sha512-<second-patch-digest>?r2 second replacement
```

`?r0` is the revision-aware original — revision zero. `?rN`, where `N` is a
positive decimal integer, identifies replacement revision `N`. Every marked
form tells pnpm to construct the integrity URL directly from the configured
registry and the complete digest.

The lockfile therefore needs only the marked integrity:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-<second-patch-digest>?r2
```

A frozen install strips the SRI option for content addressing, converts the
complete digest to base64url, and makes one request to:

```text
<effective-registry-base>/-/tarballs/sha512/<second-patch-digest-base64url>
```

No tarball URL, provider alias, custom request header, redirect, integrity
failure, metadata lookup, or capability probe is required.

An integrity with no option retains today's meaning: pnpm reconstructs the
canonical name-and-version tarball URL. This preserves existing lockfiles. The
companion registry RFC requires that canonical URL to keep serving the original
artifact.

The same ordinal also gives workspaces explicit revision selection: a spec of
the form `<version>+rN` — the ordinal carried as semver build metadata — pins
a dependency to that exact version and registry revision `N`, usable as an
override target or a directly declared dependency. `+r0` keeps the original;
a positive ordinal adopts or freezes a specific replacement.

This is the pnpm half of
[`pnpr/text/0000-integrity-addressed-patch-revisions.md`][pnpr-rfc].

[pnpr-rfc]: ../pnpr/text/0000-integrity-addressed-patch-revisions.md

## Motivation

pnpm normally omits a canonical registry tarball URL from the lockfile and
reconstructs it from registry, package name, and version:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-<upstream>
```

That compact representation currently implies a name-and-version request. It
cannot also imply an integrity-addressed request without a durable signal in
the lockfile.

Storing an absolute digest URL would pin the lockfile to a deployment hostname.
Storing a relative tarball path would duplicate information already present in
the integrity. Registry configuration already determines the origin, and the
complete SRI digest already determines the immutable object. The SRI option can
carry the one missing bit: which retrieval convention pnpm should use.

Including the marker on the original artifact is important. A lockfile may be
created before any provider revision exists. If it records:

```text
sha512-<original>?r0
```

it already uses the integrity endpoint. A patch added years later does not
change that lockfile's request:

```text
old lockfile  → /-/tarballs/sha512/<original>
new lockfile  → /-/tarballs/sha512/<patched>
```

The visible transition from `?r0` to `?r1` also explains why the integrity
changed without exposing a provider-specific package name or version.

Direct integrity URLs retain the performance and caching properties of ordinary
immutable URLs:

- one CDN request;
- no custom request header;
- no CDN `Vary` configuration;
- no redirect;
- no deliberate failed download;
- no metadata recovery round trip.

## Detailed Explanation

### Registry metadata

A supporting registry advertises its selected artifact through an immutable
digest URL and a revision-aware integrity:

```jsonc
{
  "name": "ejs",
  "version": "2.7.4",
  "dist": {
    "tarball": "https://registry.example/-/tarballs/sha512/AbCd...",
    "integrity": "sha512-AbCd...?r2"
  }
}
```

For an original artifact with no replacement, the same registry advertises:

```jsonc
{
  "dist": {
    "tarball": "https://registry.example/-/tarballs/sha512/Original...",
    "integrity": "sha512-Original...?r0"
  }
}
```

The marker applies to every package version exposed through this retrieval
protocol, not only versions that already have replacements. Consequently, every
new pnpm lockfile created against the registry is prepared for later revisions.

During resolution, pnpm validates:

1. `dist.tarball` is on the selected registry origin and beneath its
   integrity-tarball route;
2. the URL algorithm is supported and is sha512 for this protocol;
3. decoding the URL's base64url digest produces exactly the digest in
   `dist.integrity`;
4. the digest is complete, well formed, and unambiguous;
5. the selected hash expression ends in one canonical `rN` option.

The option is not part of steps 2 through 4. It describes registry revision
semantics and selects pnpm's fetch convention, but the algorithm and complete
digest alone identify the bytes.

Conforming SRI consumers ignore unrecognized options. An npm-compatible client
that does not implement this pnpm optimization can follow `dist.tarball`
directly and verify the underlying hash.

### Revision option syntax

The
[Subresource Integrity grammar](https://www.w3.org/TR/sri/#the-integrity-attribute)
defines:

```text
hash-with-options = hash-expression *("?" option-expression)
option-expression = *VCHAR
```

This RFC assigns the following registry/package-manager semantics to one
canonical form:

```text
<hash-expression>?rN   revision-aware artifact, registry revision N
```

Rules:

- `N` is a non-negative base-10 integer with no leading zeroes;
- `r0` is the original artifact; the first distinct replacement is `r1`;
- revision numbering is scoped to one registry and one `name@version`;
- later distinct replacements receive increasing numbers that are never
  reassigned;
- an already recorded artifact retains its revision if selected again;
- a suffix other than one canonical `rN` option is not this protocol's
  capability marker;
- pnpm removes all SRI options before decoding or verifying the digest.

The letter abbreviates *registry revision* and matches the numbering of the
structured `dist.revisions` record, which counts the original as revision `0`.
The revision number is deliberately not the content address. Editing `r1` to
`r2` cannot make different bytes pass verification. It is a human-readable
ordinal and a retrieval marker; structured registry history remains
authoritative about provider, provenance, fixes, and policy.

### Lockfile representation

When the validated `dist` pair uses the integrity endpoint and a recognized
revision option, pnpm records the integrity and omits `resolution.tarball`:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-AbCd...?r2
```

This remains portable when a registry deployment moves between hostnames or
when the user changes registry configuration. It also retains named-registry
path prefixes. Resolving the digest route against:

```text
https://registry.example/~main/
```

produces:

```text
https://registry.example/~main/-/tarballs/sha512/AbCd...
```

The endpoint path is relative to the registry base. It is not root-relative,
because a leading slash would discard `/~main/`.

The marked integrity is self-describing:

- no option means the legacy canonical URL convention;
- `?r0` means the digest endpoint and the original artifact;
- `?rN` with positive `N` means the digest endpoint and replacement revision
  `N`.

### Fetch and verification

For a recognized revision-aware integrity, pnpm:

1. selects the supported hash expression;
2. records its revision ordinal `N`;
3. removes the option;
4. converts the complete base64 digest to base64url;
5. resolves `-/tarballs/<algorithm>/<digest>` against the effective registry
   base;
6. sends one ordinary authenticated GET;
7. verifies the response against the complete SRI digest before admitting it
   to the content-addressed store.

If URL construction fails or the response does not match, installation fails.
pnpm must not fetch current package metadata, change the checksum, retry the
canonical URL, or fall forward to another revision.

The endpoint is same-origin with the configured registry, so ordinary registry
authentication and credential-scoping behavior applies. Redirects to another
origin are not part of this protocol. Knowledge of a digest is not
authorization to retrieve it.

The option is excluded from:

- URL and content-store keys;
- byte verification and equality;
- authorization;
- advisory or VEX policy;
- patch selection.

An option-only difference does not create another content-store object or
require another download. The structured registry record decides whether the
displayed ordinal is truthful.

### Registry identity

Constructing the digest endpoint requires the same registry identity used
during package resolution. For ordinary default and scope registries, pnpm can
recover that base through its existing registry mapping.

The known lockfile gap for named registries remains: current package keys do not
distinguish identical `name@version` values from different named registries,
and the named registry identity is not always retained for fetching. This RFC
does not make that collision worse, but content addressing does not solve it.

Until registry identity is represented unambiguously, one lockfile can contain
only one selected registry and artifact for a given `name@version`.

### Historical lockfile verification

A historical lockfile may contain `?r1` while current metadata selects `?r2`:

```text
lockfile:       ejs@2.7.4 + sha512-<patch-1>?r1
current dist:   ejs@2.7.4 + sha512-<patch-2>?r2
```

Fetching does not require current metadata: the locked digest URL is immutable.
If a policy separately verifies the lockfile against registry metadata, it
accepts the resolution when the same registry, name, version, complete digest,
and revision ordinal appear either in current `dist` or in that version's
append-only `dist.revisions`.

A syntactically valid digest route alone is not enough to prove that the
registry associated those bytes with the package.

### Existing and older lockfiles

The registry-side design keeps the original canonical URL immutable:

- a legacy lockfile with no SRI option reconstructs the canonical URL and
  continues receiving the original bytes;
- a revision-aware lockfile containing `?r0` requests those same original
  bytes from the digest endpoint;
- a revision-aware lockfile containing `?rN` requests replacement `N` from the
  digest endpoint;
- a frozen install never consults current metadata merely to choose an
  artifact.

No mismatch recovery path is required because the canonical URL never changes.

An older pnpm that resolves fresh metadata can retain and follow the absolute
`dist.tarball` URL. However, an older pnpm reading a new lockfile does not know
that `?rN` changes URL reconstruction. The lockfile format must therefore gate
this representation so older pnpm versions reject it instead of silently
requesting the canonical URL and failing integrity verification.

### CDN and installation performance

An integrity URL is an ordinary immutable CDN cache key:

```text
one resolved package → one digest URL → one tarball GET
```

A warm CDN answers without contacting the registry application. A miss reaches
one content-addressed lookup. There is no redirect or selector request before
the cacheable object.

The full sha512 digest is small relative to a tarball body and is already
present in metadata and the lockfile. pnpm already hashes the response for SRI
verification, so the protocol adds no cryptographic pass.

If the tarball is already in pnpm's content-addressed store, installation
remains network-free.

### Selecting revisions through overrides

A workspace can pin the artifact revision of a picked version with an ordinary
override whose target carries the ordinal as semver build metadata:

```jsonc
{
  "pnpm": {
    "overrides": {
      "ejs@2.7.4": "2.7.4+r0",        // always the original artifact
      "lodash@4.17.21": "4.17.21+r1"  // freeze on the first replacement
    }
  }
}
```

Build metadata is the correct carrier. Semver ignores it for precedence and
equality, so the `+rN` half can never affect which version a spec selects; it
addresses an artifact *within* the selected version — exactly the
version/revision split this design rests on.

No new override semantics are required. pnpm's exact-version selectors already
match any declared range that intersects them — `"ejs@2.7.4"` fires for a
dependency declared `^2.7.0` — rewriting the spec before resolution, behavior
documented in [pnpm/pnpm#13470] and retained for parity with npm's overrides.
What is new is only the target syntax: `<version>+rN` is a
**revision-addressed spec**, meaningful anywhere a spec appears — an override
target or a directly declared dependency.

[pnpm/pnpm#13470]: https://github.com/pnpm/pnpm/issues/13470

Resolving a `<version>+rN` spec, pnpm:

1. resolves the version half normally (build metadata is ignored for version
   matching);
2. selects registry revision `N` from the resolved entry's `dist.revisions`
   instead of the selected `dist`: its integrity, its digest URL, and its
   `manifest` record. Revisions may legally differ in dependencies, optional
   and peer dependencies, `bin`, engines, and install-script posture, so the
   pinned entry's own `manifest` drives subtree resolution — never the
   selected revision's fields.

Rules:

- `+r0` keeps the original — the opt-out. A positive ordinal adopts a
  replacement the registry advertises but has not selected, or freezes one it
  has — the explicit opt-in.
- An ordinal the registry does not advertise is a hard resolution error; pnpm
  must not fall forward to the selected revision.
- Version-changing targets compose: `"ejs@2.7.4": "2.7.5+r1"` rewrites the
  spec to version `2.7.5`, then selects its revision `1`.
- If the resolved version is not revision-aware, `+r0` is trivially satisfied
  by the only artifact and a positive ordinal is an error.
- On a revision-aware registry the `r<digits>` build-metadata namespace is
  reserved for revision selection; specs carrying any other build metadata are
  ordinary specs.

Like any exact-selector override, a `+rN` override **pins the version**: a
dependency declared `^2.7.0` stays on `2.7.4+r0` after upstream publishes a
fixed `2.7.5`, until the override is updated or removed. That is ordinary,
visible override behavior, and audit reporting of overrides that pin
superseded or vulnerable versions applies to it unchanged.

Nothing new is recorded. The resolved entry stores the marked integrity —
`sha512-<original>?r0` for an opt-out — exactly as any revision-aware
resolution does, and the override lives in the lockfile's existing overrides
record with its existing change detection. Removing the override re-resolves
the affected entries; frozen installs are untouched either way.

Degradation elsewhere is predictable: npm's overrides share the
intersection-matching behavior, and node-semver strips build metadata, so npm
treats the target as plain `2.7.4` — the version is pinned identically, but
the artifact is the registry's selected revision. Degradation is a pin
without revision selection; revision addressing is a revision-aware-client
feature, and receiving registry policy is the intended fallback for unaware
clients.

Client selection remains a preference, not a security boundary. A registry
whose policy refuses a revision's bytes answers its digest URL with the policy
`403`; a pinned install then fails loudly, naming the policy and the
advertised revisions. A workspace can choose failure over substitution; it
cannot obtain bytes the registry refuses.

A full-integrity pin is deliberately not part of this grammar: base64
characters are not legal in build metadata, the lockfile already pins exact
bytes, ordinals are stable by the registry's invariant, and structured
`dist.revisions` history is authoritative if a displayed ordinal is disputed.

### Refreshing revisions

Fetching preserves the locked revision; it does not adopt the registry's
current selection.

A separate operation can refresh artifact revisions without changing package
versions:

```text
pnpm update --patches
```

For each locked registry package, pnpm resolves current metadata for the same
exact `name@version`. If `dist.integrity` changed, pnpm updates the marked
integrity and package snapshot atomically. Dependency metadata may differ
between artifact revisions, so changing only the checksum would be incorrect.
Packages whose revision an override pins are skipped.

Whether an ordinary non-frozen install adopts a new revision automatically is a
separate pnpm policy choice.

### One revision per package key

Current pnpm package keys permit only one artifact revision of a registry
`name@version` in a graph. This RFC lets different lockfiles reproducibly pin
different revisions and lets a registry move its selected revision. It does
not allow `r1` and `r2` to coexist under one package key.

Adding integrity and registry identity to package keys would be a separate
lockfile format change.

## Rationale and Alternatives

### Store a relative tarball path

An earlier design stored:

```yaml
resolution:
  integrity: sha512-<digest>?r2
  tarball: -/tarballs/sha512/<digest>
```

This is host-portable and explicit, but the tarball path duplicates the
algorithm and digest. Once the SRI option durably distinguishes the retrieval
convention, pnpm can synthesize the same path without lockfile redundancy.

### Request digest URLs for every unmarked integrity

pnpm could reinterpret every existing `{ integrity }` resolution as a digest
request. Registries that do not implement the endpoint would fail or require a
404 followed by the canonical request. That would either break compatibility
or slow the normal registry path.

The SRI option makes support explicit and durable without a probe.

### Keep the absolute integrity URL

Keeping `dist.tarball` works and is what pnpm does with non-canonical URLs
today. It pins the registry deployment hostname in the lockfile even though the
endpoint is defined relative to a registry. The marked integrity preserves
portability.

### Select integrity through a request header

A request header lets one canonical URL return several revisions, but CDNs must
forward the field and include it in cache selection. A redirect adds another
round trip; caching bodies by the header adds deployment-specific cache
configuration.

Digest URLs are ordinary cache keys, use one request, and work for unaware npm
clients through `dist.tarball`.

### Add integrity to the canonical package URL

`<canonical-url>+<integrity>` is workable and retains a visible package
relationship. A digest-only route is shorter, deduplicates identical artifacts
inside the same registry, and can be constructed from registry plus integrity.
Registry scoping supplies the authorization boundary.

### Use a digest prefix

Rejected. The complete sha512 digest is already present, is not a credential,
and is small relative to the request. A prefix introduces collision ambiguity
and an arbitrary security parameter while saving negligible bandwidth.

### Use an empty option for the original

An earlier design marked the original with a bare `?` — grammatically valid —
and numbered only replacements. Rejected: an empty option is precisely what a
generic SRI serializer is most likely to normalize away, forcing raw-string
parsing just to preserve protocol state; build metadata cannot be empty, so
override targets would need an asymmetric `+r0` ⇔ `?` translation rule; and
the structured `dist.revisions` record already numbers the original as
revision `0`. An explicit `r0` costs two characters per lockfile entry and
removes all three problems.

### Use `v` or `b` as the ordinal letter

`v` reads as "version" — the association this design exists to avoid, because
a revision is deliberately not a version. `b` matches the semver carrier
("build metadata") but means the wrong thing — Debian's `+b1` is a rebuild
with no source change — and `b` followed by digits is valid hexadecimal, so a
reserved `b<digits>` namespace could collide with truncated-hash build
metadata. `r` abbreviates *registry revision*, matches `dist.revisions`, and
follows Alpine and Gentoo, which number downstream revisions of an unchanged
upstream version `-r0`, `-r1`, and so on.

### Put provider identity in the option

A value such as `?pnpr-patch=echo-r2` exposes implementation and provider
naming in a generic integrity field. It is longer, couples lockfiles to a
provider, and duplicates structured registry metadata.

Neutral numeric revisions explain artifact changes without making provider
identity part of package resolution.

## Implementation

1. Parse the selected SRI hash expression and recognize canonical `rN`
   options.
2. Validate during resolution that the digest URL, complete hash expression,
   and revision-aware option agree with registry metadata.
3. Store only the marked integrity in the lockfile and gate the representation
   with an appropriate lockfile version.
4. During lockfile hydration, remove the option, convert the digest to
   base64url, and resolve `-/tarballs/<algorithm>/<digest>` against the
   effective registry base.
5. Preserve registry path prefixes such as `/~main/`.
6. Fetch through the existing remote-tarball fetcher and SRI verifier without
   redirect, header, or recovery behavior.
7. Extend optional lockfile verification to accept current and historical
   digest/revision pairs from `dist.revisions`.
8. Resolve `<version>+rN` specs — override targets or declared dependencies —
   by resolving the version half normally, then selecting the revision from
   `dist.revisions` with its own resolution metadata, failing on unadvertised
   ordinals.
9. Add an explicit revision-refresh operation that updates integrity and
   snapshot atomically, skipping override-pinned packages.

Tests should cover:

- `?r0` round-tripping as a distinct state from no option;
- `?r1` and later canonical revision ordinals;
- malformed values such as `?r01`, `?r-1`, and `?rfoo`;
- unrecognized options such as `?v1` treated as unmarked legacy integrity;
- options surviving lockfile serialization and SRI-library round trips;
- URL derivation using the complete digest but excluding the option;
- base64/base64url conversion;
- malformed, abbreviated, unsupported, and mismatched digests;
- one direct request for both the original and replacements;
- no `resolution.tarball` for a revision-aware integrity;
- registry path-prefix preservation;
- changed configured registry bases without lockfile rewrites;
- same-origin authentication and cross-origin redirect rejection;
- legacy unmarked lockfiles continuing to fetch canonical original bytes;
- older pnpm rejecting the gated lockfile representation;
- historical verification through `dist.revisions`;
- a `+r0` override applying to an intersecting declared range and pinning the
  original, and a positive `+rN` override adopting an advertised replacement;
- a `+rN` spec declared directly as a dependency;
- an override naming an unadvertised ordinal failing resolution;
- a pinned revision resolving with its own dependency metadata;
- revision refresh skipping override-pinned packages;
- no extra request, redirect, or integrity pass;
- offline installation from the content-addressed store;
- revision refresh changing integrity and snapshot but not package version.

## Prior Art

- The [OCI Distribution Specification][oci-distribution]
  retrieves content by immutable digest as well as mutable tag.
- [Subresource Integrity](https://www.w3.org/TR/SRI/) already supplies the
  digest format and verification model used by npm lockfiles.
- pnpm's content-addressed store already separates package resolution identity
  from physical file content.

[oci-distribution]: https://github.com/opencontainers/distribution-spec/blob/main/spec.md

## Unresolved Questions and Bikeshedding

- Should the route be `/-/tarballs/` or another registry-neutral path?
- Should sha512 be required initially or should algorithms be allow-listed?
- What lockfile version should introduce revision-aware integrity fetching?
- What is the final name and adoption policy for `pnpm update --patches`?
- What should the global posture setting that prefers originals wholesale be
  called, with per-package `+rN` overrides as the exceptions?
- Should `pnpm audit --fix` write `+rN` overrides when a registry advertises a
  replacement fixing a reported advisory?
- How should named registry identity be represented so two registries can
  resolve the same `name@version` without a lockfile collision?
