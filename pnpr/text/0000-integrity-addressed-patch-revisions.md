# Integrity-addressed patch revisions for pnpr

## Summary

pnpr should be able to project a provider's patched build directly over the
original package's `name@version`, without exposing a provider package alias or
inventing a version suffix. The projected package document keeps the ordinary
canonical tarball URL and identifies the currently selected build through
`dist.integrity`. It also advertises an append-only history of every artifact
revision previously served for that `name@version`.

The canonical URL is a movable pointer for clients that have not resolved the
package before. Every underlying artifact is immutable and is additionally
available from an integrity-addressed URL derived from the canonical URL.
A client that already has an integrity pin can send that full SRI value in a
`Pnpr-Expected-Integrity` request header, and pnpr returns or redirects to that
exact historical revision. A companion pnpm change specifies this request
header and a metadata-assisted fallback when an intermediary ignores it.

This makes the distinction explicit:

> `name@version` selects the registry's current artifact revision.
> `name@version+integrity` identifies immutable bytes.

Fresh resolutions receive the current patched build. Updated pnpm clients can
keep existing lockfiles on exactly the bytes they pinned. The lockfile
integrity is never silently replaced.

## Motivation

### Patch providers deliberately preserve the original version

Vendors such as Echo produce security-remediated builds of existing package
versions. The application asked for `ejs@2.7.4`, the patched package still
reports `ejs@2.7.4`, and ordinary semver ranges and peer dependency checks
should continue to see `2.7.4`.

Encoding the provider into a package alias gives this operation a large client
surface:

```jsonc
{
  "pnpm": {
    "overrides": {
      "ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"
    }
  }
}
```

The provider scope must be routed and authenticated on every client. The
provider package appears as a different package in lockfiles, SBOMs, license
tools, advisory databases, and runtime package metadata. A second patch of the
same upstream version also needs another provider-side package identity or
version, even though those are delivery details rather than application
dependency identities.

For a registry-wide remediation policy, the intended operation is simpler:
when this registry resolves `ejs@2.7.4`, its current approved artifact is
Echo's second patch of `ejs@2.7.4`.

### A patched build can itself need another patch

There is not necessarily one patched artifact per upstream version. A provider
may issue revision 1 for one vulnerability and revision 2 when another
vulnerability is discovered in the same old release line. Both builds must
remain reproducible:

```text
ejs@2.7.4 + sha512-<upstream>    original artifact
ejs@2.7.4 + sha512-<echo-r1>     first provider revision
ejs@2.7.4 + sha512-<echo-r2>     current provider revision
```

Making each revision a package name creates one packument per revision.
Encoding it into the version introduces prerelease semantics and makes the
version reported by the package untrue. Integrity is already the value npm
lockfiles use to distinguish and authenticate the actual bytes, so it is the
natural revision key.

### The old integrity must remain authoritative

Changing the bytes returned by the canonical npm tarball URL breaks an old
lockfile unless the registry can learn which bytes that lockfile expected.
The HTTP request normally carries only the URL; the expected SRI value remains
inside the client.

The client can provide that missing dimension. With the expected integrity in
the request, the canonical URL becomes equivalent to a mutable OCI tag while
the integrity-qualified URL is equivalent to an immutable digest reference.
The server can select the correct historical representation before sending a
tarball, and the client still verifies the complete response against its
lockfile.

## Detailed Explanation

### Terminology and invariants

- An **original identity** is an npm package `name@version`, for example
  `ejs@2.7.4`.
- An **artifact revision** is one exact tarball for that identity, identified
  by its complete SRI value.
- The **current revision** is the artifact returned to a client that resolves
  the version now and sends no previously locked integrity.
- The **canonical URL** is the ordinary npm tarball URL derived from registry,
  name, and version:
  `https://registry.example/ejs/-/ejs-2.7.4.tgz`.
- A **revision URL** is the canonical URL with a URL-safe integrity suffix:
  `https://registry.example/ejs/-/ejs-2.7.4.tgz+sha512.<base64url>`.

The following invariants are required:

1. **Full integrity is artifact identity.** pnpr stores and looks up revisions
   by the complete SRI value. A truncated digest is never an identity or a
   security decision.
2. **Revision URLs are immutable.** Once a revision URL has returned bytes, it
   may never return different bytes.
3. **History is append-only.** Selecting a new current revision does not
   remove or mutate older revision records.
4. **The lockfile wins.** The request header selects bytes; it never authorizes
   pnpm to replace the integrity already in its lockfile.
5. **Unknown integrity fails closed.** If a request names an integrity that is
   not retained for that exact registry, package name, and version, pnpr does
   not fall back to the current revision.
6. **One selected revision per identity.** Within one projected registry,
   exactly one revision of a `name@version` is current. Conflict between
   provider policies is a configuration error, not an implicit precedence
   rule.
7. **Resolution metadata is revision-bound.** The projected version metadata
   comes from the same integrity-verified artifact as its tarball, and a
   lockfile revision update replaces the integrity and dependency snapshot
   together.

### Provider input remains integrity-pinned

Patch providers publish a signed manifest that maps an original identity to an
immutable artifact. The artifact may be stored as a provider package or as an
opaque tarball; that internal encoding is not exposed to package consumers:

```jsonc
{
  "schema": "pnpr-patch-manifest/1",
  "entries": {
    "ejs@2.7.4": {
      "revision": "echo-r2",
      "tarball": "https://npm.echo.example/artifacts/ejs/2.7.4/r2.tgz",
      "integrity": "sha512-<echo-r2>",
      "fixes": ["GHSA-...", "CVE-2026-..."],
      "attestations": ["https://npm.echo.example/attestations/..."]
    }
  }
}
```

pnpr verifies the manifest signature, fetches the artifact, verifies its full
integrity, and stores it in the selected registry's content-addressed cache
before making it current. A manifest refresh is pinned and applied atomically:
packument metadata must not advertise a revision before its immutable tarball
is available.

The verified tarball must contain the original `name` and `version`. A provider
may use scoped package names or revision numbers inside its own distribution
system, but the artifact it offers for projection must still identify itself
as `ejs@2.7.4`. pnpr does not rewrite `package.json` after verification, because
doing so would create different bytes from the provider's signed integrity.

The provider remains an uncontrolled origin. Its declared integrity is useful
only because pnpr enforces it at ingestion and serves the verified bytes from
the pnpr registry. A compromised provider may refuse to serve a new revision,
but it cannot replace an already accepted revision with different bytes.

An illustrative configuration is:

```yaml
registries:
  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true

  main:
    type: router
    sources: [npmjs]
    patching:
      providers: [echo]
      minSeverity: high

patchProviders:
  echo:
    manifest:
      url: https://npm.echo.example/pnpr-manifest.json
      publicKeys:
        - ed25519:AAAA...
      refreshInterval: 1h
    auth:
      tokenEnv: ECHO_PATCH_TOKEN
```

The provider credential and its artifact namespace are server-side details.
A consumer configures only the `main` registry; it does not configure an
`@echo-patch` scope or a named Echo registry.

### Projected package document

For a covered version, pnpr serves the original package name and version with
the selected revision's integrity. The canonical tarball URL remains in
`dist.tarball`. `dist.revisions` advertises the original artifact and every
accepted provider revision:

```jsonc
{
  "name": "ejs",
  "versions": {
    "2.7.4": {
      "name": "ejs",
      "version": "2.7.4",
      // resolution metadata from the selected echo-r2 artifact
      "dependencies": {},
      "dist": {
        "tarball": "https://registry.example/ejs/-/ejs-2.7.4.tgz",
        "integrity": "sha512-<echo-r2>",
        "revisions": [
          {
            "integrity": "sha512-<upstream>",
            "source": "upstream",
            "publishedAt": "2020-04-03T00:00:00.000Z"
          },
          {
            "integrity": "sha512-<echo-r1>",
            "source": "echo",
            "revision": "echo-r1",
            "publishedAt": "2026-05-10T00:00:00.000Z",
            "fixes": ["GHSA-..."],
            "attestations": ["https://..."]
          },
          {
            "integrity": "sha512-<echo-r2>",
            "source": "echo",
            "revision": "echo-r2",
            "publishedAt": "2026-07-28T00:00:00.000Z",
            "fixes": ["GHSA-...", "CVE-2026-..."],
            "attestations": ["https://..."]
          }
        ]
      }
    }
  }
}
```

The current revision is the entry whose integrity equals `dist.integrity`.
There is no `_pnprPatch` instruction and no alternate package specifier for a
client to interpret. Unaware npm clients ignore `dist.revisions` and install
the current revision exactly like an ordinary package.

The history provides:

- discoverability through `npm view` and registry APIs;
- provenance and advisory/VEX metadata for each exact set of bytes;
- the allow-list used by integrity-aware retrieval;
- an auditable sequence when a provider re-issues a patch;
- retention and withdrawal state, if the schema grows it later.

The field name is bikesheddable. Its semantics are not: the list is
append-only, keyed by full integrity, and includes the original artifact.

### Manifest metadata across revisions

Package managers resolve the dependency graph from packument metadata before
they download the tarball. A security patch may legitimately update a
dependency as well as source files, so revisions are not required to have
identical dependency metadata.

Instead, pnpr reads the selected artifact's `package.json` only after the
tarball passes integrity verification and projects its resolution-relevant
fields into the current version document. When a revision becomes current, the
whole projected version document and `dist.integrity` move atomically.

An old lockfile retains both the old integrity and its old dependency snapshot,
so fetching its historical tarball remains coherent. A revision refresh in
pnpm must replace the integrity and package snapshot together; updating only
the checksum would be incorrect.

This deliberately means that two lockfiles can record different dependency
graphs for the same `name@version` when they pin different integrities. The
current pnpm lockfile cannot represent both revisions simultaneously in one
graph, consistent with the one-selected-revision invariant. If simultaneous
revisions are required, integrity must become part of the lockfile package key
in a separate format change.

### Integrity negotiation on the canonical tarball URL

A pnpm client with a lockfile pin sends the complete SRI value:

```http
GET /ejs/-/ejs-2.7.4.tgz HTTP/1.1
Host: registry.example
Pnpr-Expected-Integrity: sha512-<echo-r1>
```

If that integrity exists in `ejs@2.7.4`'s retained revisions and the caller is
authorized, pnpr returns those bytes or redirects to the immutable revision
URL:

```http
HTTP/1.1 307 Temporary Redirect
Location: /ejs/-/ejs-2.7.4.tgz+sha512.<base64url>
Vary: Pnpr-Expected-Integrity
Cache-Control: private, no-store
```

Without the header, pnpr selects the revision named by the current
`dist.integrity`. With an unknown or malformed value, it returns an error; it
must never silently return current bytes for a request that named another
integrity.

`Want-Content-Digest` and `Want-Repr-Digest` from
[RFC 9530](https://www.rfc-editor.org/rfc/rfc9530.html) are not suitable for
this request: they ask a server to include a digest of its response and express
algorithm preference, but do not select a representation by an already-known
digest. This RFC therefore proposes a dedicated request field. Its name can be
standardized separately.

Responses selected by a request field must use
[`Vary`](https://www.rfc-editor.org/rfc/rfc9110.html#name-vary), so conforming
shared caches do not return one revision for a request that named another.
However, pnpr deployments must not assume every CDN forwards an unknown request
field or enables arbitrary `Vary` processing by default. Redirecting the
canonical request is preferred: the mutable, negotiated response is small and
not cached, while the tarball body is served only from an immutable URL.

The server may also send `Content-Digest` for HTTP-level observability, but the
npm SRI value from the lockfile remains the client's security check.

### Integrity-addressed revision URL

Every revision is available from a deterministic URL:

```text
<canonical-url>+<algorithm>.<base64url-digest>
```

For example:

```text
https://registry.example/ejs/-/ejs-2.7.4.tgz+sha512.AbCd...
```

The URL form uses the SRI algorithm and a base64url encoding of the complete
digest. Raw SRI base64 is not appended directly because `/`, `+`, and `=` have
awkward URL and intermediary semantics.

The endpoint:

- applies the same registry identity and access policy as the canonical URL;
- verifies that the digest belongs to this exact `name@version`;
- never redirects or falls through to a different digest;
- returns immutable cache headers and a strong digest-derived validator;
- remains available for as long as pnpr promises lockfile reproducibility.

Example:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
ETag: "<full-digest>"
Content-Digest: sha-512=:<base64>:
```

The URL also gives clients a fallback when a proxy strips the negotiation
header, ignores `Vary`, or has cached old content at the canonical URL.

### CDN deployment profile

The protocol works with a CDN when the mutable selector and immutable content
are treated as two cache classes:

1. The CDN forwards `Pnpr-Expected-Integrity` on canonical tarball requests.
2. The canonical endpoint bypasses the CDN cache and returns a small redirect.
   `Cache-Control: private, no-store` expresses this at HTTP level, and the CDN
   configuration must not override it with a positive minimum TTL.
3. The integrity-qualified endpoint is cached by its complete URL with a long
   immutable TTL. It does not vary on the request header.
4. The packument remains ordinary mutable metadata and is purged or
   revalidated when the selected revision changes.

This does not put high-cardinality integrity values into a tarball-body cache
key. Every artifact body has one stable URL and therefore the same CDN behavior
as any other content-addressed asset. Only the inexpensive canonical redirect
reaches pnpr.

CDN configuration is not uniform:

- CloudFront can forward the field through an origin request policy without
  caching on it. If redirects are cached, the cache policy must also put the
  field in the cache key. Its minimum TTL can override origin `no-store`, so
  the canonical behavior must use a zero-TTL or disabled cache policy.
- Cloudflare requires its Cache Rules `Vary` support or a custom cache key for
  arbitrary varying headers. The simpler recommended rule is to bypass cache
  for canonical tarball redirects and forward the field.
- Fastly supports `Vary`, but the canonical route should still be pass/private
  so all large cacheable objects live at revision URLs.

If a CDN strips the field despite that configuration, correctness is
preserved: pnpm rejects the current revision against its old lockfile
integrity and uses the package-document plus revision-URL recovery path. The
penalty is an extra failed download, not acceptance of wrong bytes. A
deployment should exercise this failure mode before enabling same-version
revision changes.

The relevant vendor documentation is:

- [CloudFront cache policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cache-key-understand-cache-policy.html)
- [Cloudflare `Vary` cache behavior](https://developers.cloudflare.com/cache/concepts/vary/)
- [Fastly `Vary` behavior](https://www.fastly.com/documentation/reference/http/http-headers/Vary/)

### The integrity value is not a secret

The request sends the full digest because it is an identifier, not a
credential. The same value is already present in the authorized package
document, the consumer's lockfile, the revision history, and the immutable
revision URL. TLS protects it from observers outside the client, CDN, and
registry, while those services necessarily see the package metadata and
tarball anyway.

Possession of a digest grants no additional access. pnpr checks the ordinary
registry authorization and verifies that the digest belongs to the requested
package version before returning bytes.

The value is still usage metadata: it tells the registry and its CDN exactly
which historical revision a consumer is installing, potentially revealing
that the consumer is behind the current security patch. Those services
necessarily need that information to return the requested revision. Operators
should protect or redact this field in access logs as they would private
package names. A uniquely identifying prefix would have the same privacy
property, so truncation does not solve it.

The field accepts exactly one supported, canonical SRI hash expression and has
a small fixed maximum length. If a lockfile integrity set contains several
algorithms, pnpm selects the strongest mutually supported complete hash. pnpr
rejects duplicate fields, SRI option metadata, unsupported algorithms, and
oversized values before any cache lookup. These parsing rules prevent the
field from becoming an unbounded cache or logging input. Patch revisions should
require sha512; a legacy checksum whose algorithm is no longer collision
resistant is not made strong merely by sending all its characters.

A prefix would reveal almost the same information while making selection
weaker. Two retained revisions can share a short prefix accidentally, and an
untrusted provider can deliberately search for one. Full client verification
would prevent such a collision from becoming arbitrary code execution, but the
prefix could still select the wrong download or cause denial of service. Full
sha512 avoids that ambiguity at negligible request cost.

### Client behavior and compatibility

The companion pnpm RFC gives pnpm three paths:

1. Send the locked integrity on the initial canonical tarball request. A pnpr
   server returns the right revision immediately.
2. If an intermediary ignores the header but happens to return bytes matching
   the lockfile, accept them normally.
3. If integrity fails, discard the response, refresh the package document,
   require the exact locked integrity to appear in `dist.revisions`, and retry
   through the deterministic revision URL. Verify the full integrity again.

At no point is the integrity from current registry metadata copied over the
locked value. A missing history entry, unavailable artifact, second mismatch,
or unsupported algorithm is a hard failure.

Compatibility is intentionally asymmetric:

| Client and state | Result |
| --- | --- |
| Any npm-compatible client making a fresh resolution | Installs the current revision from `dist` |
| New pnpm with an old lockfile | Retrieves the locked revision through the header or revision URL |
| Old pnpm, npm, or Yarn with an old lockfile after current changes | Fails integrity verification |
| Any client with the tarball already in its verified content store | Uses the stored revision |

This proposal improves pnpm reproducibility; it cannot teach already released
clients how to retrieve a second representation from one canonical URL. A
deployment that requires historical lockfiles to work with every existing npm
client must keep the canonical URL immutable and publish the current revision
under a non-canonical immutable `dist.tarball` URL instead.

### Updating patches without changing versions

A frozen install never changes revision. An ordinary install may continue to
prefer its lockfile, just as it does for package versions.

pnpm should provide an explicit operation to refresh only artifact revisions,
for example:

```text
pnpm update --patches
```

For every locked registry package, it resolves the exact locked
`name@version`, compares `dist.integrity` with the lockfile, and updates the
resolution integrity and snapshot only when the registry selected another
compatible revision. This does not re-run semver selection and does not update
package versions.

A normal resolution caused by another lockfile change also records the current
revision. Whether a non-frozen `pnpm install` should adopt newly selected
revisions automatically is a pnpm policy question; the registry protocol does
not require silent lockfile mutation.

### Caching and atomicity

pnpr must not cache artifact bodies only by canonical URL. Its cache key is:

```text
(concrete registry identity, package name, version, full integrity)
```

The packument cache and canonical redirect are mutable projections. A provider
refresh that selects a new revision follows this order:

1. Verify and retain the new tarball under its integrity key.
2. Derive and validate its version metadata from the verified tarball.
3. Append its revision metadata.
4. Atomically update the projected version document and `dist.integrity`.
5. Invalidate projected full and abbreviated packuments and the canonical
   redirect.

This ordering prevents metadata from selecting bytes the tarball endpoint
cannot yet serve. Prior revision objects and URLs remain untouched.

The canonical response must not be served with npm tarballs' customary
long-lived immutable caching unless the cache key includes
`Pnpr-Expected-Integrity`. A short, revalidated redirect plus immutable
revision URLs avoids that ambiguity.

For predictable behavior across CDN implementations, the default is stronger:
the canonical redirect is `private, no-store` and the CDN is configured to
bypass it. `Vary` remains required protocol metadata and permits an operator to
cache redirects only after verifying the CDN's exact header forwarding and
cache-key behavior.

### Authorization, retention, and enforcement

Knowledge of an integrity is not authorization. Both retrieval forms apply the
same registry access rules as the package document and canonical tarball.

History and availability are separate concepts. An operator may retain a
revision record for audit while a legal or security policy refuses its bytes.
In that case the revision request returns a policy-specific `403`, and a
lockfile that pins it cannot install through that registry.

This matters for the vulnerable original artifact. Reproducibility policy may
retain it so an old lockfile works; strict enforcement may deliberately block
it. The patch mechanism must not silently weaken a registry's separate
screening or refusal policy.

### Auditing a revision

Advisories apply to the original `name@version`; provider VEX statements apply
to one exact integrity. `dist.revisions[].fixes` and attestations preserve that
mapping without changing package identity.

An audit request that carries only `name@version` is evaluated conservatively
against the base version. An integrity-aware pnpm audit extension can identify
the installed revision and subtract only the fixes declared for those exact
bytes. A later provider revision does not retroactively add fixes to an older
locked one.

## Rationale and Alternatives

### Provider package aliases and `_pnprPatch`

The alternative patch-provider RFC exposes a provider package such as
`@echo-patch/ejs` and either asks the workspace to write an alias override or
adds a registry annotation that instructs pnpm to do it.

That representation makes provider provenance conspicuous and keeps
`name@version` immutable, but it makes a registry policy into a client-level
package substitution. It requires provider-registry configuration to resolve
the alias, creates separate packuments for provider revision names, and depends
on pnpm-specific annotation or override behavior. This proposal keeps
provenance in revision metadata and makes unaware clients install the current
registry policy.

### Encode the patch revision in the version

Versions such as `2.7.4-echo.2` provide a distinct npm identity, but they are
prereleases under semver. Ordinary ranges containing `2.7.4` do not necessarily
select them, peer ranges may reject them, and the package no longer reports
the upstream version the application was tested against. Build metadata is
not a substitute: semver deliberately ignores it for precedence and equality.

### Encode the revision in a package alias name

Names such as `@echo-patch/ejs-r1` and `@echo-patch/ejs-r2` avoid prerelease
semantics but create a separate package document for every patch. They also
surface a provider-internal revision scheme throughout lockfiles and tooling.

### Use a named provider registry

A specifier such as `echo:ejs@2.7.4` keeps the original name and version, but
every consumer must configure the `echo` registry locally. Current pnpm
lockfiles also key registry packages primarily by `name@version`, so two named
registries serving different artifacts for the same identity can collide in
one graph until registry identity becomes part of the package key.

Named registries remain a coherent choice when the provider is intentionally
the origin for all packages. They are a poor encoding for a central registry
that wants to adopt only selected patch artifacts.

### Keep the canonical URL immutable and rewrite `dist.tarball`

The strongest cross-client compatibility design keeps the original canonical
URL on the original bytes forever and changes current packument metadata to
point at a non-canonical, integrity-addressed patch URL. Old lockfiles continue
to work in pnpm, npm, and Yarn without a client change.

Its cost in pnpm today is that non-canonical absolute tarball URLs are retained
in the lockfile, pinning the deployment host and losing the portability that
canonical URL reconstruction provides. This RFC chooses a canonical movable
pointer plus integrity negotiation to retain compact, host-free pnpm lockfiles.
The compatibility table above is the price of that choice.

### Retry only after an integrity failure

Downloading the current tarball, hashing it, and only then requesting the
historical revision is safe but wasteful. Sending the expected integrity on
the first request lets pnpr select correctly in one round trip. The
post-failure package-document lookup remains necessary as a robust fallback
for intermediaries that do not preserve the request field or `Vary` behavior.

### Send only a digest prefix

Rejected. A full sha512 SRI value is small compared with even the HTTP headers
of a tarball request. Prefixes introduce accidental ambiguity, allow deliberate
collision work against the lookup key, and require an arbitrary minimum length.
The client would still need the full digest for verification, so truncation
saves almost nothing. The full digest is already advertised in package
metadata and is neither an authentication secret nor an authorization
capability.

## Implementation

### pnpr

1. Add provider manifest configuration, signature verification, pinned refresh,
   and a content-addressed ingestion path.
2. Validate each artifact's integrity and original `name@version`, then derive
   the projected version metadata from its verified `package.json`.
3. Store revisions by concrete registry identity, package, version, and full
   integrity; retain append-only provenance metadata.
4. Project the selected revision into full and abbreviated packuments,
   including `dist.revisions`.
5. Add integrity negotiation to canonical tarball routes and emit a correct
   `Vary` header.
6. Add immutable integrity-qualified tarball routes with the same authorization
   and screening policy as the canonical package.
7. Make provider refresh atomic and audit every current-revision transition.
8. Extend audit handling to understand revision-scoped fixes and attestations.

Tests should cover:

- original, first patch, and second patch all retrievable by full integrity;
- a request without a header receiving the current revision;
- a request header for an older revision receiving or redirecting to exactly
  that revision;
- malformed, abbreviated, unknown, wrong-package, and wrong-version integrity
  values failing without current-revision fallback;
- full SRI verification both at provider ingestion and client response;
- `Vary`, cache invalidation, immutable revision caching, and a proxy that
  ignores the request header;
- CloudFront-, Cloudflare-, and Fastly-shaped cache configurations in which
  canonical redirects bypass cache and revision bodies remain cacheable;
- rejection of duplicate, oversized, multi-value, unsupported, and abbreviated
  integrity request fields;
- atomic refresh with no metadata-visible-but-unavailable interval;
- selected revision metadata being derived from the verified tarball;
- a revision that changes dependencies updating its lockfile snapshot together
  with its integrity;
- provider conflict rejection;
- access and screening policy on historical URLs;
- audit/VEX evaluation tied to the exact artifact integrity.

### pnpm

The companion RFC,
[`text/0000-integrity-addressed-tarball-recovery.md`](../../text/0000-integrity-addressed-tarball-recovery.md),
contains the client implementation and tests.

## Prior Art

- The [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
  permits a manifest reference to be a mutable tag or an immutable digest.
  This proposal applies the same current-pointer/content-identity split to npm
  package artifacts.
- npm lockfiles already use
  [Subresource Integrity](https://www.w3.org/TR/SRI/) to authenticate fetched
  tarballs. This proposal uses the existing full SRI value as the revision key
  rather than inventing another hash format.
- HTTP [`Vary`](https://www.rfc-editor.org/rfc/rfc9110.html#name-vary)
  defines the cache behavior needed when a request field selects among
  representations.
- [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530.html) standardizes
  `Content-Digest` and `Repr-Digest` for response integrity. Those fields are
  useful response metadata here, though their `Want-*` request fields do not
  perform exact representation selection.

## Unresolved Questions and Bikeshedding

- Is `dist.revisions` the right package-document field name, or should it be
  explicitly namespaced?
- Is `Pnpr-Expected-Integrity` the right request field name? If other registries
  want the protocol, should it be proposed as a vendor-neutral HTTP field?
- Should the canonical endpoint return the selected body directly or always
  redirect to the immutable revision URL? This draft recommends the redirect
  for CDN behavior.
- Which manifest fields must pnpr project from the selected tarball, and should
  historical revision entries retain a normalized snapshot for inspection?
- Does `pnpm update --patches` update all selected revisions, only
  security-motivated ones, or accept provider and advisory filters?
- How long must a pnpr deployment retain historical revision bytes, and how
  should an operator communicate a retention policy that is shorter than
  lockfile lifetime?
- Should historical vulnerable revisions remain retrievable by default, or
  require an explicit reproducibility policy?
