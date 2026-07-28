# Integrity-addressed registry tarballs

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
  ejs@2.7.4:            # original artifact — today's representation, unchanged
    resolution:
      integrity: sha512-<original-digest>
  lodash@4.17.21:       # replacement revision 1
    resolution:
      integrity: sha512-<first-patch-digest>
      revision: 1
```

An entry without `revision` keeps exactly today's meaning: pnpm reconstructs
the canonical name-and-version tarball URL. The companion registry RFC pins
that URL to the original upstream bytes forever, so **original artifacts need
no marker at all** — a lockfile that has adopted no replacements is
byte-identical to a current lockfile.

An entry with `revision: N` must be fetched by digest. A frozen install
converts the complete digest to base64url and makes one request to:

```text
<effective-registry-base>/-/tarballs/sha512/<first-patch-digest-base64url>
```

No tarball URL, provider alias, custom request header, redirect, integrity
failure, metadata lookup, or capability probe is required. The field itself is
the per-entry signal.

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

That compact representation implies a name-and-version request. A replacement
revision cannot use it: the canonical URL deliberately keeps serving the
original bytes, so a replacement needs a durable, per-entry signal that its
bytes live at the digest route instead.

Storing an absolute digest URL would pin the lockfile to a deployment
hostname. Storing a relative tarball path would duplicate information already
present in the integrity. Registry configuration already determines the
origin, and the complete SRI digest already determines the immutable object.
A `revision` field carries the one missing bit — which retrieval convention
applies — and doubles as the human-readable explanation: a lockfile diff
showing `revision: 1` → `revision: 2` beside the integrity change says why
the integrity changed without exposing a provider-specific package name or
version.

Originals need no signal, because the registry-side invariant already
guarantees them: the canonical URL serves revision zero forever. An old
lockfile written years before a package gained its first patch keeps
requesting the canonical URL and keeps receiving the original bytes. Nothing
about an unpatched resolution changes, which means:

- a workspace that has adopted no replacements produces a lockfile
  indistinguishable from today's — no format gate, no growth, readable by
  older pnpm;
- pointing a lockfile at a registry without the digest endpoint keeps the
  entire unpatched graph installable; only `revision` entries fail, and they
  fail as genuine unavailability, not as ambiguity.

Direct integrity URLs retain the performance and caching properties of
ordinary immutable URLs:

- one CDN request;
- no custom request header;
- no CDN `Vary` configuration;
- no redirect;
- no deliberate failed download;
- no metadata recovery round trip.

## Detailed Explanation

### Registry metadata

A registry whose selected artifact for a version is a replacement advertises
the digest URL, a standard SRI integrity, and the revision ordinal:

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
extensions, so every strict SRI consumer parses it. `dist.revision` is
present exactly when the selected artifact is a replacement (`N ≥ 1`). A
version whose selected artifact is the original serves ordinary, unmarked
metadata; its `dist.tarball` may be the canonical URL or the digest URL, at
the registry's choice.

When `dist.revision` is present, pnpm validates during resolution:

1. `dist.tarball` is on the selected registry origin and beneath its
   integrity-tarball route;
2. the URL algorithm is supported and is sha512 for this protocol;
3. decoding the URL's base64url digest produces exactly the digest in
   `dist.integrity`;
4. the digest is complete, well formed, and unambiguous;
5. `dist.revision` is a positive integer.

When a digest-route `dist.tarball` appears *without* `dist.revision` — an
original served through the registry's CDN route — pnpm normalizes: it
records a plain integrity-only entry rather than storing the absolute URL,
because the canonical URL serves the same bytes and an absolute URL would pin
the deployment hostname in the lockfile.

Existing npm-compatible clients follow `dist.tarball` directly, verify the
standard integrity, and ignore the unknown `dist.revision` field.

### The revision field

Rules:

- `N` is a positive base-10 integer;
- revision numbering is scoped to one registry and one `name@version`;
- the original artifact is revision zero and is never recorded — the absence
  of the field is the durable signal for the canonical fetch convention,
  which the registry invariant pins to revision zero;
- ordinals are never reassigned, and an already recorded artifact retains its
  ordinal if selected again;
- the field matches the numbering of the structured `dist.revisions` record,
  which counts the original as revision `0`.

The revision number is deliberately not the content address. Editing
`revision: 1` to `revision: 2` cannot make different bytes pass verification.
It is a human-readable ordinal and a retrieval marker; structured registry
history remains authoritative about provider, provenance, fixes, and policy.

### Lockfile representation

For a validated replacement, pnpm records the integrity and ordinal and omits
`resolution.tarball`:

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

- no `revision` field means the canonical URL convention — a legacy entry and
  a deliberate revision-zero entry are the same thing, because the canonical
  route serves revision zero by invariant;
- `revision: N` means the digest endpoint and replacement revision `N`.

### Fetch and verification

For an entry carrying `revision: N`, pnpm:

1. converts the complete base64 digest to base64url;
2. resolves `-/tarballs/<algorithm>/<digest>` against the effective registry
   base;
3. sends one ordinary authenticated GET;
4. verifies the response against the complete SRI digest before admitting it
   to the content-addressed store.

If URL construction fails or the response does not match, installation fails.
pnpm must not fetch current package metadata, change the checksum, retry the
canonical URL, or fall forward to another revision. Falling back to the
canonical URL can never succeed anyway — it serves revision zero, whose bytes
cannot match a replacement's integrity — so the failure surfaces immediately
rather than after a wasted download.

The endpoint is same-origin with the configured registry, so ordinary registry
authentication and credential-scoping behavior applies. Redirects to another
origin are not part of this protocol. Knowledge of a digest is not
authorization to retrieve it.

The `revision` field is excluded from:

- URL and content-store keys;
- byte verification and equality;
- authorization;
- advisory or VEX policy.

A revision-only difference does not create another content-store object or
require another download. The structured registry record decides whether the
recorded ordinal is truthful.

### Feature detection is advisory, never load-bearing

The fetch convention is determined entirely by the lockfile: no `revision`
field means canonical, `revision: N` means the digest route. Correctness never
depends on detecting what the registry supports.

A pnpr deployment's discovery surface (the `/-/pnpr/v0/*` namespace of the
resolution-cache design) can still improve two things:

- **diagnostics** — when a digest fetch fails, pnpm can consult the
  handshake to report "this registry does not advertise integrity-addressed
  tarballs; the locked replacement cannot be fetched from it" instead of a
  bare `404`;
- **optimization** — a registry advertising the capability may let pnpm fetch
  originals through the digest route as well (shared CDN cache with fresh
  npm-client resolutions). The canonical route remains the specified default
  for originals, so an install must behave identically with detection
  unavailable.

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

Revision ordinals are registry-scoped, not global. The same `name@version`
may have different revision histories on different registries, so
re-resolving a `+rN` spec against a different registry can select different
bytes, a different ordinal for the same bytes, or fail. A frozen install
stays safe regardless — it fetches by complete digest and verifies the bytes
— but portability across registry configuration guarantees integrity, never
availability or identical ordinal meaning. An ordinal-addressed spec is an
assertion about the currently effective registry.

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
and revision ordinal appear either in current `dist` or in that version's
append-only `dist.revisions`.

A syntactically valid digest route alone is not enough to prove that the
registry associated those bytes with the package.

### Existing and older lockfiles

The registry-side design keeps the original canonical URL immutable:

- legacy entries and deliberate revision-zero entries share one
  representation — integrity only — and reconstruct the canonical URL, which
  continues serving the original bytes on every registry, revision-aware or
  not;
- an entry with `revision: N` requests replacement `N` from the digest
  endpoint;
- a frozen install never consults current metadata merely to choose an
  artifact.

No mismatch recovery path is required because the canonical URL never changes.

A lockfile containing no `revision` entries is byte-identical to the current
format and needs no gating. The `revision` field itself must be gated by
lockfile version: an older pnpm that ignored the unknown field would
reconstruct the canonical URL and fail integrity verification with a
misleading error, so it must reject the lockfile instead.

### CDN and installation performance

Originals fetch exactly as today: one canonical URL, one GET. Replacements
fetch from an ordinary immutable CDN cache key:

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
   pinned entry's own `manifest` drives subtree resolution — never the current
   projected version's top-level fields.

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

Nothing new is recorded beyond the field itself. An adopted replacement
stores `integrity` plus `revision: N`; an opt-out stores the original's
integrity as a plain entry with no `revision` field — the lockfile diff shows
the revision line appearing or disappearing beside the integrity change. The
override lives in the lockfile's existing overrides record with its existing
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
bytes, ordinals are stable by the registry's invariant, and structured
`dist.revisions` history is authoritative if a recorded ordinal is disputed.

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

An earlier draft of this RFC carried the ordinal inside the integrity value,
using the SRI option grammar: `sha512-<digest>?r2`, with `?r0` marking
originals. The option is self-contained — it travels wherever the integrity
string is copied — but it was rejected for the field:

- it assigns protocol semantics to a generic web-spec extension point, and
  npm metadata is parsed by many strict SRI consumers that may reject or
  normalize options;
- preserving options through SRI libraries and serializers becomes
  load-bearing protocol state, an entire class of tooling fragility a plain
  YAML/JSON field does not have;
- the ordinal is buried at the end of a ninety-character line, so a lockfile
  diff hides exactly the information the marker exists to show; a separate
  `revision` line is legible on its own.

### Mark original artifacts in the lockfile

The same earlier draft marked *every* entry from a revision-aware registry —
originals as `?r0` (later `revision: 0`) — so the lockfile itself recorded
the digest-endpoint capability before any patch existed. Rejected:

- the canonical URL already guarantees originals: the registry invariant pins
  it to revision zero forever, so an unmarked entry keeps fetching correct
  bytes through any registry, and a patch appearing later changes nothing for
  existing lockfiles;
- marking everything hard-binds the whole lockfile to revision-aware
  registries — switching the effective registry to one without the endpoint
  would break every entry instead of only the replacements that genuinely do
  not exist there;
- it puts a marker line (or suffix) on every entry and gates every lockfile
  behind a new version, making the common case — no adopted patches — pay
  for the rare one. With unmarked originals, a lockfile without replacements
  is byte-identical to today's format.

Absence of the field is itself durable: it does not mean "unknown", it means
revision zero, because the canonical route is pinned to revision zero by the
registry invariant.

### Detect the digest endpoint instead of recording anything

Registry feature detection (the pnpr handshake) could tell pnpm to fetch
everything by digest, with no lockfile change at all for originals — but
replacements still need the per-entry ordinal and integrity, so the field is
required regardless, and making the fetch path depend on a probe would put a
network round trip and a cached-capability staleness question in front of
every install. Detection is kept advisory (diagnostics, optional
original-by-digest optimization); the lockfile field alone determines the
fetch convention.

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

The ordinal is a bare integer everywhere structured (`dist.revision`,
`resolution.revision`, `dist.revisions[].revision`); a letter appears only in
the inline `+rN` spec syntax, where build metadata needs a marker. `v` reads
as "version" — the association this design exists to avoid, because a
revision is deliberately not a version. `b` matches the semver carrier
("build metadata") but means the wrong thing — Debian's `+b1` is a rebuild
with no source change — and `b` followed by digits is valid hexadecimal, so a
reserved `b<digits>` namespace could collide with truncated-hash build
metadata. `r` abbreviates *registry revision*, matches `dist.revisions`, and
follows Alpine and Gentoo, which number downstream revisions of an unchanged
upstream version `-r0`, `-r1`, and so on.

### Put provider identity in the revision record

A value such as `revision: echo-r2` exposes implementation and provider
naming in resolution metadata. It couples lockfiles to a provider and
duplicates structured registry metadata, and a provider rename would look
like an artifact change. Neutral integers explain artifact changes without
making provider identity part of package resolution.

## Implementation

1. Recognize `dist.revision` alongside a digest-route `dist.tarball` during
   resolution and validate the pair (origin, algorithm, digest equality,
   positive integer ordinal).
2. Record `revision: N` in the resolution object and gate the field with an
   appropriate lockfile version; lockfiles without the field keep the current
   version.
3. Normalize digest-route tarballs without `dist.revision` to plain
   integrity-only entries (no hostname in the lockfile).
4. During lockfile hydration of a `revision` entry, convert the digest to
   base64url and resolve `-/tarballs/<algorithm>/<digest>` against the
   effective registry base, preserving registry path prefixes such as
   `/~main/`.
5. Fetch through the existing remote-tarball fetcher and SRI verifier without
   redirect, header, or recovery behavior; on failure, optionally consult the
   registry handshake to report missing endpoint support distinctly from a
   missing object.
6. Extend optional lockfile verification to accept current and historical
   digest/revision pairs from `dist.revisions`.
7. Resolve `<version>+rN` specs — override targets or declared dependencies —
   by resolving the version half normally, then selecting the revision from
   `dist.revisions` with its own resolution metadata, failing on unadvertised
   ordinals.
8. Add an explicit revision-refresh operation that updates integrity,
   `revision`, and snapshot atomically, skipping override-pinned packages.

Tests should cover:

- entries without `revision` fetching the canonical URL unchanged, including
  against revision-aware registries;
- `revision: N` entries making one digest request with no canonical retry,
  metadata fetch, or fall-forward;
- digest-route tarballs without `dist.revision` normalized to integrity-only
  entries;
- malformed `revision` values (zero, negative, non-integer, strings) rejected
  at resolution and at lockfile parsing;
- base64/base64url conversion;
- malformed, abbreviated, unsupported, and mismatched digests;
- registry path-prefix preservation;
- changed configured registry bases without lockfile rewrites;
- same-origin authentication and cross-origin redirect rejection;
- a lockfile without `revision` entries remaining readable by older pnpm, and
  older pnpm rejecting a lockfile that contains the gated field;
- historical verification through `dist.revisions`;
- a `+r0` override applying to an intersecting declared range and pinning the
  original, and a positive `+rN` override adopting an advertised replacement;
- a `+rN` spec declared directly as a dependency;
- a `+r0` opt-out recorded as a plain entry with no `revision` field;
- rewriting applied once: no re-matching of `+rN` results, no chains or
  cycles through version-changing targets;
- conflicting revision demands for one `name@version` failing with an
  explicit error;
- an override naming an unadvertised ordinal failing resolution;
- a pinned revision resolving with its own dependency metadata;
- revision refresh updating integrity, `revision`, and snapshot together, and
  skipping override-pinned packages;
- no extra request, redirect, or integrity pass;
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
  lifetime), and should pnpm use it to fetch originals by digest?
- What is the final name and adoption policy for `pnpm update --patches`?
- What should the global posture setting that prefers originals wholesale be
  called, with per-package `+rN` overrides as the exceptions?
- Should `pnpm audit --fix` write `+rN` overrides when a registry advertises a
  replacement fixing a reported advisory?
- How should named registry identity be represented so two registries can
  resolve the same `name@version` without a lockfile collision?
