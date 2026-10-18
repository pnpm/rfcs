# Integrity-addressed registry tarballs

> Status: proposed amendment to the experimental revision protocol;
> implementation pending.

## Summary

pnpm should support registries that serve immutable, integrity-addressed
tarball URLs:

```text
<registry-base>/-/tarballs/<algorithm>/<base64url-digest>
```

and represent registry artifact revisions with a `revision` field in the
lockfile's resolution object:

```yaml
packages:
  ejs@2.7.4:            # original artifact, revision zero
    resolution:
      integrity: sha512-<original-tar-digest>
      revision: 0
  lodash@4.17.21:       # replacement revision 1
    resolution:
      integrity: sha512-<first-patch-digest>
      revision: 1
```

Every revision-aware entry records its revision explicitly, including
`revision: 0` for the original. All revisions use the uncompressed tar's
integrity and the same digest route, independent of registry format.
An entry without `revision` is an ordinary registry resolution outside this
protocol and retains its existing retrieval convention.

An entry with `revision: N` must be fetched by digest. A frozen install
converts the complete digest to base64url and makes one request to:

```text
<effective-registry-base>/-/tarballs/sha512/<first-patch-digest-base64url>
```

The digest names the revision's uncompressed tar, not its `.tgz`. HTTP
content coding may change without changing that URL or its integrity.

No tarball URL, provider alias, custom request header, redirect, integrity
failure, metadata lookup, or capability probe is required. The field itself is
the per-entry signal.

The same revision number also gives workspaces explicit revision selection: a
spec of the form `<version>+rN` — the revision number carried as semver build
metadata — pins a dependency to that exact version and registry revision `N`,
usable as an override target or a directly declared dependency. `+r0` keeps the
original; a positive revision number adopts or freezes a specific replacement.

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

That compact representation implies a name-and-version request. A
revision-aware resolution needs an explicit marker to select the digest route
and tar integrity. The same convention applies to the original and every
replacement, so selecting revision zero does not change fetch semantics.

Storing an absolute digest URL would pin the lockfile to a deployment
hostname. Storing a relative tarball path would duplicate information already
present in the integrity. Registry configuration already determines the
origin, and the complete SRI digest already determines the immutable object.
A `revision` field carries the one missing bit — which retrieval convention
applies — and doubles as the human-readable explanation: a lockfile diff
showing `revision: 1` → `revision: 2` beside the integrity change says why
the integrity changed without exposing a provider-specific package name or
version.

The revisions feature is experimental. This proposal replaces its earlier
contract without a compatibility or migration requirement for experimental
registry state or lockfiles.

Direct integrity URLs retain the performance and caching properties of
ordinary immutable URLs:

- one CDN request;
- no custom request header;
- only the standard `Vary: Accept-Encoding` for HTTP content coding;
- no redirect;
- no deliberate failed download;
- no metadata recovery round trip.

## Detailed Explanation

### Registry metadata

For every revision-aware version, the registry advertises
the digest URL, a standard SRI integrity, and the revision number:

```jsonc
{
  "name": "ejs",
  "version": "2.7.4",
  "dist": {
    "tarball": "https://registry.example/-/tarballs/sha512/AbCd...",
    "integrity": "sha512-AbCd...",
    "revision": 2
  }
}
```

`dist.integrity` is an ordinary Subresource Integrity value with no
extensions. `dist.revision` is present for every revision-aware version:
`0` selects the original and `N ≥ 1` selects a replacement. Its `dist.tarball`
always names the tar by digest, including for revision zero. A version outside
this protocol omits both `dist.revision` and `dist.revisions` and follows the
registry's ordinary resource convention.

When `dist.revision` is present, pnpm validates during resolution:

1. `dist.tarball` is on the selected registry origin and beneath its
   integrity-tarball route;
2. the URL algorithm is supported and is sha512 for this protocol;
3. decoding the URL's base64url digest produces exactly the digest in
   `dist.integrity`;
4. the digest is complete, well formed, and unambiguous;
5. `dist.revision` is a safe nonnegative integer within the range defined
   under *The revision field*;
6. `dist.revisions` includes revision zero and a record matching the selected
   revision number and tar integrity. The history and selected marker must both
   be present; incomplete or contradictory revision metadata is rejected.

A digest-route `dist.tarball` without `dist.revision` is not valid revision
metadata. pnpm must not infer a revision or normalize that URL onto a canonical
route. Ordinary non-revision URLs retain their existing handling.

Clients receiving revision resources must remove HTTP content coding before
SRI verification and accept a plain tar archive. Following `dist.tarball` and
ignoring `dist.revision` alone is not sufficient; support in npm, Yarn, and
other clients requires client-matrix tests.

### Revision resources and their integrity

Every revision, including zero, is a tar resource on every registry. Its URL
digest, `dist.integrity`, `dist.revisions[].integrity`, and
`resolution.integrity` all hash the same uncompressed tar. This does not depend
on `lockfileIntegrity` or on a registry's `tarballs` setting.

The tar is the whole gzip-decoded source artifact, as defined by
[the tar registry RFC](./0000-zstd-registry-tarballs.md#the-tar-and-its-integrity).
It is not re-packed or normalized. pnpr verifies the source `.tgz` against its
source digest before deriving the tar hash. That source digest remains evidence
for signatures and attestations; it is not the client's content address.
Provider sources additionally require the signed manifest verification defined
in the registry half of this RFC.

A revision response has `Content-Type: application/x-tar`. Its
`Content-Encoding` is `gzip`, `zstd`, or absent, negotiated through
`Accept-Encoding`, with `Vary: Accept-Encoding`. After content decoding,
pnpm hashes the tar and checks it against the locked digest before indexing
the package. The decoded-output budget and decoder window limits of the tar
registry RFC apply. There is no fallback to a compressed-byte hash.

### The revision field

Rules:

- `N` is a nonnegative integer bounded by `0 ≤ N ≤ 2^53 − 1`
  (JavaScript's `Number.MAX_SAFE_INTEGER`). Producers serialize it as a
  canonical base-10 integer: no leading zeros, sign, fraction, or exponent.
  Clients validate the parsed value as a safe nonnegative integer; they need
  not recover the original numeric token from JSON or YAML. Strings,
  fractional values, negative values, and values outside the range are rejected;
- revision numbering is scoped to one registry and one `name@version`;
- the original artifact is recorded as `revision: 0`; absence of the field
  means an ordinary resolution outside the revision protocol;
- revision numbers are never reassigned, and an already recorded artifact
  retains its revision number if selected again;
- the field matches the numbering of the structured `dist.revisions` record,
  which counts the original as revision `0`.

The revision number is deliberately not the content address. Editing `revision:
1` to `revision: 2` cannot make different bytes pass verification. It is a
human-readable position in the revision history and a retrieval marker;
structured registry history remains authoritative about provider, provenance,
fixes, and policy.

### Lockfile representation

For every validated revision, including zero, pnpm records the integrity and
revision number and omits `resolution.tarball`:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-AbCd...
      revision: 2
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

The representation is self-describing with two states:

- no `revision` field means the ordinary registry retrieval convention;
- `revision: N`, including `revision: 0`, means the tar digest endpoint and
  registry revision `N`.

### Fetch and verification

For an entry carrying `revision: N`, pnpm:

1. converts the complete base64 digest to base64url;
2. resolves `-/tarballs/<algorithm>/<digest>` against the effective registry
   base;
3. sends one ordinary authenticated GET;
4. removes HTTP content coding and verifies the tar against the complete SRI
   digest before recording the package in the content-addressed store index.

If URL construction fails or the response does not match, installation fails.
pnpm must not fetch current package metadata, change the checksum, retry the
canonical URL, or fall forward to another revision. The locked tar digest and
fetch route are authoritative, including for revision zero.

The endpoint is same-origin with the configured registry, so ordinary registry
authentication and credential-scoping behavior applies. Redirects — to another
origin or the same origin — are not part of this protocol; every 3xx response
fails the fetch. One GET means one GET: every extra hop is another log that
sees the URL, and a deployment that wants a CDN puts it at the registry
origin. Knowledge of a digest is not authorization to retrieve it.

The `revision` field is excluded from:

- URL and content-store keys;
- byte verification and equality;
- authorization;
- advisory or VEX policy.

A revision-only difference does not create another content-store object or
require another download. The structured registry record decides whether the
recorded revision number is truthful.

### Feature detection is advisory, never load-bearing

The fetch convention is determined entirely by the lockfile: an entry with
`revision: N`, including zero, uses the digest route. Correctness never depends
on detecting what the registry supports.

A pnpr deployment's discovery surface (the `/-/pnpr/v0/*` namespace of the
resolution-cache design) may improve diagnostics after a digest fetch fails.
It may report that the registry does not advertise integrity-addressed tarballs
rather than a bare `404`. It cannot change the locked resource or fetch route.

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

Revision numbers are registry-scoped, not global. The same `name@version` may
have different revision histories on different registries, so re-resolving a
`+rN` spec against a different registry can select different bytes, a different
revision number for the same bytes, or fail. A frozen install stays safe
regardless — it fetches by complete digest and verifies the bytes — but
portability across registry configuration guarantees integrity, never
availability or identical revision-number meaning. A revision-addressed spec is
an assertion about the currently effective registry.

### Historical lockfile verification

A historical lockfile may pin revision `1` while current metadata selects
revision `2`:

```text
lockfile:       ejs@2.7.4, integrity sha512-<patch-1>, revision 1
current dist:   ejs@2.7.4, integrity sha512-<patch-2>, revision 2
```

Fetching does not require current metadata: the locked digest URL is immutable.
If a policy separately verifies the lockfile against registry metadata, it
accepts the resolution when the same registry, name, version, complete digest,
and revision number appear either in current `dist` or in that version's
append-only `dist.revisions`.

A syntactically valid digest route alone is not enough to prove that the
registry associated those bytes with the package.

### Existing and older lockfiles

The experimental revision protocol has no backward compatibility or migration
requirement. Earlier experimental registry state and lockfiles do not constrain
this design. A frozen install must still fail on unsupported state rather than
silently reinterpret a compressed digest as a tar digest.

Lockfiles that use this protocol require a format version that understands
`revision: 0` and tar integrity for all revisions. A client that cannot interpret
that version must reject the lockfile before fetching. Lockfiles without
revision entries follow the ordinary registry protocol.

### CDN and installation performance

Every revision, including the original, fetches from an ordinary immutable
CDN cache key:

```text
one resolved package → one digest URL → one tarball GET
```

A warm CDN answers without contacting the registry application. A miss reaches
one content-addressed lookup. There is no redirect or selector request before
the cacheable object.

The full sha512 digest is small relative to a tarball body and is already
present in metadata and the lockfile. A revision resource is decoded and hashed
once on the client, as a tar resource; compression variants share the same
content-store identity.

If the tarball is already in pnpm's content-addressed store, installation
remains network-free.

### Selecting revisions through overrides

A workspace can pin the artifact revision of a picked version with an ordinary
override whose target carries the revision number as semver build metadata:

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
   pinned entry's own `manifest` drives subtree resolution — never the current
   projected version's top-level fields.

Rules:

- `+r0` keeps the original — the opt-out. A positive revision number adopts a
  replacement the registry advertises but has not selected, or freezes one it
  has — the explicit opt-in.
- A revision number the registry does not advertise is a hard resolution error;
  pnpm must not fall forward to the selected revision.
- Version-changing targets compose: `"ejs@2.7.4": "2.7.5+r1"` rewrites the
  spec to version `2.7.5`, then selects its revision `1`.
- If the resolved version is not revision-aware, `+r0` is trivially satisfied
  by the only artifact and a positive revision number is an error.
- On a revision-aware registry the `r<digits>` build-metadata namespace is
  reserved for revision selection; specs carrying any other build metadata are
  ordinary specs.
- Rewriting applies once: a `+rN` result — including a version-changing
  target such as `2.7.5+r1` — is never re-matched against overrides, so
  revision selection cannot chain or cycle.
- Two specs demanding different revisions of the same `name@version` in one
  graph (for example, parent-scoped overrides pinning `+r0` and `+r1`) are an
  explicit resolution error until integrity becomes part of the package key;
  pnpm must not silently unify them.

Like any exact-selector override, a `+rN` override **pins the version**: a
dependency declared `^2.7.0` stays on `2.7.4+r0` after upstream publishes a
fixed `2.7.5`, until the override is updated or removed. That is ordinary,
visible override behavior, and audit reporting of overrides that pin
superseded or vulnerable versions applies to it unchanged.

Every revision-aware selection records tar `integrity` plus `revision: N`.
A `+r0` selection records `revision: 0`; switching from a replacement to the
original changes the revision number and integrity together. For a version
outside the revision protocol, `+r0` leaves its ordinary resolution unchanged.
The override lives in the lockfile's existing overrides record with its existing
change detection. Removing the override re-resolves the affected entries;
frozen installs are untouched either way.

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
bytes, revision numbers are stable by the registry's invariant, and structured
`dist.revisions` history is authoritative if a recorded revision number is
disputed.

### Refreshing revisions

Fetching preserves the locked revision; it does not adopt the registry's
current selection.

A separate operation can refresh artifact revisions without changing package
versions:

```text
pnpm update --patches
```

For each locked registry package, pnpm resolves current metadata for the same
exact `name@version`. If the selected artifact changed, pnpm updates the
integrity, the `revision` field, and the package snapshot atomically.
Dependency metadata may differ between artifact revisions, so changing only
the checksum would be incorrect. Packages whose revision an override pins are
skipped.

Whether an ordinary non-frozen install adopts a new revision automatically is a
separate pnpm policy choice.

### One revision per package key

Current pnpm package keys permit only one artifact revision of a registry
`name@version` in a graph. This RFC lets different lockfiles reproducibly pin
different revisions and lets a registry move its selected revision. It does
not allow `r1` and `r2` to coexist under one package key. Overrides or specs
demanding different revisions of one `name@version` therefore fail resolution
with an explicit conflict rather than silently unifying.

Adding integrity and registry identity to package keys would be a separate
lockfile format change.

## Rationale and Alternatives

### Encode the revision as an SRI option

An earlier draft of this RFC carried the revision number inside the integrity
value, using the SRI option grammar: `sha512-<digest>?r2`, with `?r0` marking
originals. SRI reserves `?options` for exactly this kind of extension, and the
option is self-contained: it travels wherever the integrity string is copied,
so no new field has to be threaded through resolution types, fetcher options,
snapshot converters, or cache entries. Rejected nonetheless:

- pnpm's Rust stack deserializes `resolution.integrity` into an
  `ssri::Integrity`, whose parser has no option grammar at all: it splits on
  `-` and takes the remainder as the digest, so `sha512-<digest>?r2` parses
  *successfully* with the option absorbed into the digest string. It
  round-trips through `Display`, so the corruption survives a read and write
  cycle intact and surfaces far from the parse — as a verification failure, or
  as a base64 decode panic in `to_hex`. Silent absorption is a worse failure
  mode than rejection.
- integrity-string equality is byte equality throughout pnpm today. The package
  requester decides whether to force a refetch by comparing the previous and
  freshly resolved integrity strings, and the same string is persisted beside a
  fetched tarball for later comparison. A revision number inside the value
  makes a revision-only annotation difference read as a byte change at every
  such site. As a separate field, the invariant that the revision is excluded
  from equality, store keys, and verification holds for free; as an option,
  every call site in both stacks has to strip it first.
- the revision number is buried at the end of a ninety-character line, so a
  lockfile diff hides exactly the information the marker exists to show; a
  separate `revision` line is legible on its own.

### Encode the revision in the lockfile package key

pnpm's dep-path grammar already carries non-semver information in the version
slot — `foo@file:...`, `node@runtime:...`, and the registry-qualified
`foo@work:1.0.0` keys of [pnpm/pnpm#13528] — so the revision number could ride
there too, as `ejs@2.7.4+r2`, reusing the `+rN` spec syntax instead of adding a
resolution field. That would also let `r1` and `r2` coexist in one graph, which
the `revision` field cannot (see *One revision per package key*).

Rejected because a dep path is repeated once per dependent. In pnpm's own
lockfile `semver@7.8.5` appears as a snapshot dependency ref fifty times beside
its `packages:` and `snapshots:` keys, so one revision bump would rewrite
roughly fifty lines of renaming and bury the two lines that describe the
artifact change. pnpm has already run this experiment: `patchedDependencies`
encodes `(patch_hash=...)` into the dep path, so editing a patch file rewrites
every reference to the patched package. The `revision` field changes exactly
two lines, integrity and revision number, and `pnpm update --patches` is meant
to be run repeatedly.

The difference from #13528 is not that a key may not carry extra data, but
which data belongs there. A package's registry is effectively immutable for
the life of a dependency; a revision is designed to change. Encoding a mutable
attribute in an identity key turns every update into a graph-wide rename, and
the key also names the virtual store directory, so a revision bump would
relink every dependent instead of re-materializing one package's files.

[pnpm/pnpm#13528]: https://github.com/pnpm/pnpm/pull/13528

### Mark original artifacts in the lockfile

Recording `revision: 0` gives originals and replacements one representation:
all revision entries use the tar digest route. Selecting the original does not
require a format-dependent checksum or canonical-route exception. The
experimental protocol does not preserve unmarked revision-zero pins.

### Detect the digest endpoint instead of recording anything

The lockfile records the fetch convention for each revision explicitly,
including revision zero. A capability probe would add a network dependency and
make frozen retrieval sensitive to cached discovery state. Detection is kept
advisory for diagnostics.

### Store a relative tarball path

An earlier design stored:

```yaml
resolution:
  integrity: sha512-<digest>
  revision: 2
  tarball: -/tarballs/sha512/<digest>
```

This is host-portable and explicit, but the tarball path duplicates the
algorithm and digest. Once the `revision` field durably distinguishes the
retrieval convention, pnpm can synthesize the same path without lockfile
redundancy.

### Request digest URLs for every unmarked integrity

pnpm could reinterpret every existing `{ integrity }` resolution as a digest
request. Registries that do not implement the endpoint would fail or require a
404 followed by the canonical request. That would either break compatibility
or slow the normal registry path. The unmarked representation keeps its
existing, universally supported meaning instead.

### Keep the absolute integrity URL

Keeping `dist.tarball` works and is what pnpm does with non-canonical URLs
today. It pins the registry deployment hostname in the lockfile even though the
endpoint is defined relative to a registry. The `revision` field preserves
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

### Use `v` or `b` as the spec-syntax letter

The revision number is a bare integer everywhere structured (`dist.revision`,
`resolution.revision`, `dist.revisions[].revision`); a letter appears only in
the inline `+rN` spec syntax, where build metadata needs a marker. `v` reads as
"version" — the association this design exists to avoid, because a revision is
deliberately not a version. `b` matches the semver carrier ("build metadata")
but means the wrong thing — Debian's `+b1` is a rebuild with no source change —
and `b` followed by digits is valid hexadecimal, so a reserved `b<digits>`
namespace could collide with truncated-hash build metadata. `r` abbreviates
*registry revision*, matches `dist.revisions`, and follows Alpine and Gentoo,
which number downstream revisions of an unchanged upstream version `-r0`,
`-r1`, and so on.

The syntax is npm-specific; the portable rule is that the revision signal
lives in resolution metadata, never in the version string.

### Put provider identity in the revision record

A value such as `revision: echo-r2` exposes implementation and provider
naming in resolution metadata. It couples lockfiles to a provider and
duplicates structured registry metadata, and a provider rename would look
like an artifact change. Neutral integers explain artifact changes without
making provider identity part of package resolution.

## Implementation

1. Recognize `dist.revision` alongside a digest-route `dist.tarball` during
   resolution and validate the pair (origin, algorithm, digest equality,
   nonnegative integer revision number).
2. Record `revision: N` in the resolution object and gate the field with an
   appropriate lockfile version; lockfiles without the field keep the current
   version. Ship the field's type in `@pnpm/lockfile-types` and document it
   in the same release that first emits it — many tools parse
   `pnpm-lock.yaml`, and a typed field on day one beats each of them
   reverse-engineering it.
3. Require explicit `dist.revision: 0` and tar integrity for originals in
   revision-aware metadata. Reject incomplete revision metadata.
4. During lockfile hydration of a `revision` entry, convert the digest to
   base64url and resolve `-/tarballs/<algorithm>/<digest>` against the
   effective registry base, preserving registry path prefixes such as
   `/~main/`.
5. Fetch all revisions as tar resources, removing HTTP content coding before
   SRI verification under the tar registry RFC's limits. Do not add redirect,
   selector-header, or recovery behavior; on failure, optionally consult the
   registry handshake to report missing endpoint support distinctly from a
   missing object.
6. Extend optional lockfile verification to accept current and historical
   digest/revision pairs from `dist.revisions`.
7. Resolve `<version>+rN` specs — override targets or declared dependencies —
   by resolving the version half normally, then selecting the revision from
   `dist.revisions` with its own resolution metadata, failing on unadvertised
   revision numbers.
8. Add an explicit revision-refresh operation that updates integrity,
   `revision`, and snapshot atomically, skipping override-pinned packages.

Tests should cover:

- ordinary entries without `revision` retaining their existing fetch behavior;
- `revision: N` entries making one digest request with no canonical retry,
  metadata fetch, or fall-forward;
- revision zero using the same tar digest route as positive revisions;
- digest-route revision metadata without `dist.revision` rejected;
- malformed `revision` values (negative, fractional, strings, values above
  `2^53 − 1`) rejected at resolution and at lockfile parsing;
- producers serializing revision numbers as canonical decimal integers;
- base64/base64url conversion;
- malformed, abbreviated, unsupported, and mismatched digests;
- registry path-prefix preservation;
- changed configured registry bases without lockfile rewrites;
- same-origin authentication and rejection of every redirect (3xx),
  same-origin included;
- a lockfile without `revision` entries remaining readable by older pnpm, and
  older pnpm rejecting a lockfile that contains the gated field;
- historical verification through `dist.revisions`;
- a `+r0` override applying to an intersecting declared range and pinning the
  original, and a positive `+rN` override adopting an advertised replacement;
- a `+rN` spec declared directly as a dependency;
- a revision-aware `+r0` opt-out recorded with tar integrity and `revision: 0`;
- rewriting applied once: no re-matching of `+rN` results, no chains or
  cycles through version-changing targets;
- conflicting revision demands for one `name@version` failing with an
  explicit error;
- an override naming an unadvertised revision number failing resolution;
- a pinned revision resolving with its own dependency metadata;
- revision refresh updating integrity, `revision`, and snapshot together, and
  skipping override-pinned packages;
- identical tar hashes and digest URLs for two gzip encodings of one tar;
- gzip, zstd, and identity responses verifying against the same revision pin,
  including revision zero, with tampered or truncated responses failing before a package is indexed;
- tar integrity for every revision on `.tgz` and tar registries with
  `lockfileIntegrity` unset or set to `tar`;
- switching the selected artifact back to zero preserving the tar digest
  convention and updating its manifest atomically;
- a tar-hashed revision never normalized to a canonical `.tgz` URL;
- no extra request, redirect, or encoded-response integrity check;
- offline installation from the content-addressed store.

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
- What lockfile version should introduce the `revision` field?
- What is the exact shape of the advisory handshake (endpoint, fields, cache
  lifetime)?
- What is the final name and adoption policy for `pnpm update --patches`?
- What should the global posture setting that prefers originals wholesale be
  called, with per-package `+rN` overrides as the exceptions?
- Should `pnpm audit --fix` write `+rN` overrides when a registry advertises a
  replacement fixing a reported advisory?
- Should the lockfile `revision` value admit an optional annotation suffix
  (for example `2-echo_r2`) that pnpm ignores but human readers see, or does
  that re-couple lockfiles to the provider naming that `dist.revisions`
  already records?
- How should named registry identity be represented so two registries can
  resolve the same `name@version` without a lockfile collision?
