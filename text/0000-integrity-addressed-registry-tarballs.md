# Integrity-addressed registry tarballs

## Summary

pnpm should recognize registry-scoped, integrity-addressed tarball URLs:

```text
<registry-base>/-/tarballs/<algorithm>/<base64url-digest>
```

When a package document advertises such a URL in `dist.tarball`, pnpm fetches it
directly like any other registry tarball. In the lockfile, pnpm stores the path
relative to the registry base instead of retaining an absolute deployment URL:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-<base64>?pnpr-patch=echo-r2
      tarball: -/tarballs/sha512/<base64url>
```

Frozen installs resolve that path against the currently configured registry and
make one ordinary CDN request. The URL digest and hash expression in
`resolution.integrity` must agree; an optional SRI revision hint is preserved
but ignored for that comparison. Historical locked URLs remain valid even when
the same `name@version` has another selected revision in current package
metadata.

This is the pnpm half of
[`pnpr/text/0000-integrity-addressed-patch-revisions.md`](../pnpr/text/0000-integrity-addressed-patch-revisions.md).

## Motivation

pnpm normally omits a canonical registry tarball URL from the lockfile and
reconstructs it from registry, package name, and version:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-<upstream>
```

This is compact and registry-portable because canonical npm tarball URLs are
predictable. A content-addressed URL is also predictable, but from registry and
integrity rather than registry, name, and version.

Without special handling, pnpm retains a non-canonical `dist.tarball` as an
absolute URL. A pnpr deployment moving from one hostname or registry base to
another would then require lockfile rewrites. Storing the digest path relative
to the registry preserves portability while making the immutable URL explicit
to old and new pnpm fetch paths.

Direct integrity URLs also avoid the cost and complexity of content negotiation
on a mutable canonical URL:

- no custom request header;
- no CDN `Vary` configuration;
- no redirect;
- no failed integrity download followed by metadata recovery;
- no extra round trip.

## Detailed Explanation

### Resolution

A supporting registry advertises an immutable URL as ordinary package metadata:

```jsonc
{
  "name": "ejs",
  "version": "2.7.4",
  "dist": {
    "tarball": "https://registry.example/-/tarballs/sha512/AbCd...",
    "integrity": "sha512-AbCd...?pnpr-patch=echo-r2"
  }
}
```

The npm resolver validates:

1. the tarball URL is on the selected registry origin and beneath its
   integrity-tarball route;
2. the URL algorithm is supported and is sha512 for patch revisions;
3. decoding its base64url digest produces exactly the digest in the strongest
   hash expression in `dist.integrity`;
4. neither hash value is abbreviated, malformed, or ambiguous.

SRI options do not participate in steps 2 or 3. pnpm preserves a recognized,
bounded `pnpr-patch=<provider-revision>` option for display but treats it like
any other unknown SRI option during byte verification.

On success, the resolver returns the ordinary tarball resolution. No
provider-specific alias, annotation, or resolver substitution occurs.

An unaware npm-compatible client also follows `dist.tarball` directly and
verifies `dist.integrity`, so fresh resolution requires no pnpm-specific
behavior.

### Lockfile representation

For an integrity URL on the selected registry base, pnpm strips that base and
stores a relative path:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-AbCd...?pnpr-patch=echo-r2
      tarball: -/tarballs/sha512/AbCd...
```

The path deliberately has no leading slash. Resolving `-/tarballs/...` against:

```text
https://pnpr.example/~main/
```

produces:

```text
https://pnpr.example/~main/-/tarballs/...
```

A root-relative `/-/tarballs` path would incorrectly discard the named
registry segment.

This uses the lockfile's existing relative-tarball concept rather than adding a
provider hostname or duplicating registry configuration. On install, pnpm
resolves the path against the effective registry exactly as it already does for
relative tarball resolutions.

The explicit path is preferable to storing only `{ integrity }`: that compact
form currently means "reconstruct the canonical name-and-version URL." A
relative path keeps the retrieval protocol self-describing and lets clients
that understand existing relative tarballs fetch it without a capability
probe.

### Registry identity

Resolving a relative tarball requires the same registry identity used during
package resolution. For ordinary default and scope registries, pnpm can recover
that base through its existing registry mapping.

The known lockfile gap for named registries remains: current package keys do not
distinguish identical `name@version` values from different named registries,
and the named registry identity is not always retained for fetching. This RFC
does not make that collision worse, but integrity URLs do not solve it. The
registry-identity lockfile follow-up must cover the relative integrity path as
well.

Until then, a lockfile can contain only one selected registry and artifact for a
given `name@version`, and deployment-portable relative paths are guaranteed only
where pnpm can recover the effective registry unambiguously.

### Fetch and verification

pnpm resolves the relative path against the registry, sends one ordinary GET,
and runs its existing complete SRI verification before adding files to the
content-addressed store.

The digest appears twice intentionally:

- the URL selects the immutable CDN object;
- `resolution.integrity` independently tells pnpm which bytes it may accept.

If they disagree syntactically before fetching, pnpm fails. If the response
does not hash to the full locked integrity, pnpm follows its ordinary retry and
hard-failure behavior. It never tries current `dist`, changes the checksum, or
falls back to another revision.

An SRI option such as `?pnpr-patch=echo-r2` is not hashed and does not change
which bytes match. pnpm preserves it in serialization for human visibility,
but it must not use the option for URL construction, cache identity,
authorization, advisory suppression, or patch-selection policy. Structured,
authenticated registry metadata remains authoritative.

An option-only difference is not an integrity change and must not trigger a
tarball download or create another content-store entry. A later explicit
resolution may update the displayed hint in the lockfile while retaining the
same artifact.

The endpoint is same-origin with the configured registry, so ordinary registry
authentication and credential-scoping behavior applies. Redirects to another
origin are not part of this protocol.

### Historical lockfile verification

pnpm can optionally verify lockfile tarball URLs against current registry
metadata. A historical lockfile may point to revision 1 while the current
version document selects revision 2:

```text
lockfile:       ejs@2.7.4 + sha512-r1
current dist:   ejs@2.7.4 + sha512-r2
```

Exact comparison with only current `dist.tarball` would incorrectly reject the
historical immutable URL. For integrity-addressed registries, verification
succeeds when either:

1. URL and integrity equal the current `dist`; or
2. the exact URL and full integrity appear together in that version's
   append-only `dist.revisions`.

The registry must affirm the same package name, version, registry identity,
URL, and integrity. Merely recognizing a syntactically valid digest route is
not enough to authorize a lockfile-supplied object.

If metadata cannot be fetched, history is absent, or any component differs,
verification fails closed under the same policy as other unverifiable lockfile
tarball URLs.

### Existing lockfiles

The registry-side design keeps the original canonical URL immutable.
Consequently:

- old lockfiles containing only the original integrity reconstruct the
  canonical URL and continue working without a pnpm change;
- old pnpm versions resolving a patched package retain its absolute integrity
  URL and can reproduce it, although the lockfile is host-pinned;
- new pnpm versions store the relative integrity URL and remain portable;
- frozen installs never consult current `dist` merely to choose an artifact.

No integrity request header or mismatch recovery path is needed.

### CDN and installation performance

An integrity URL is an ordinary immutable CDN cache key. A cold install makes
the same number of requests as today:

```text
one resolved package → one tarball GET
```

A warm CDN answers without contacting pnpr. A miss reaches one direct
content-addressed lookup at pnpr. There is no redirect or selector request
before the body.

The request URL is roughly one sha512 digest longer than a canonical npm
tarball URL, which is insignificant relative to the body. pnpm already hashes
the response for SRI and indexes its local store by content, so CPU and local
store behavior do not gain another verification pass.

If the tarball is already in pnpm's content-addressed store, installation
remains entirely local.

### Refreshing patch revisions

Fetching preserves the locked revision; it does not adopt the registry's
currently selected artifact.

A separate operation refreshes artifact revisions without changing versions:

```text
pnpm update --patches
```

For each registry package already selected in the lockfile, pnpm resolves the
same exact `name@version`. If current `dist.integrity` changed, pnpm updates the
relative tarball path, integrity, and package snapshot together. Dependency
ranges are not reselected.

Whether an ordinary non-frozen install adopts a new revision automatically is a
separate policy choice.

### Coexistence

Current pnpm package keys still permit only one revision of a registry
`name@version` in a graph. This RFC lets different lockfiles reproducibly pin
different revisions and lets a registry move its selected revision globally.
It does not allow revision 1 and revision 2 to coexist under one package key.

Adding integrity and registry identity to package keys would be a separate
lockfile format change.

## Rationale and Alternatives

### Store only integrity and synthesize the URL

When pnpm knows a registry supports the endpoint, it could construct
`/-/tarballs/<algorithm>/<digest>` from `resolution.integrity` and store no
tarball path.

That is a valid future optimization. It needs a durable registry capability
signal on frozen installs so `{ integrity }` is not confused with today's
canonical name-and-version reconstruction. The relative path is already
self-describing, host-independent, and compatible with the existing lockfile
resolution shape, so this RFC uses it first.

### Keep the absolute integrity URL

This works and is what pnpm does with non-canonical tarball URLs today. It pins
the registry deployment hostname in the lockfile even though the endpoint is
defined relative to a registry. Stripping only the verified registry base
retains the useful path while preserving portability.

### Select integrity through a request header

A request header lets one canonical URL return several revisions, but CDNs must
forward the field and include it in cache selection. Serving a redirect adds
another round trip; caching bodies by the header adds deployment-specific cache
configuration.

Direct URLs are standard cache keys, use one request, and work for unaware npm
clients through ordinary `dist.tarball`.

### Add the integrity to the canonical package URL

`<canonical-url>+<integrity>` is workable and retains a visible package
relationship. A digest-only route is shorter, deduplicates identical artifacts
across package aliases inside the same registry, and can be derived from
registry plus integrity alone. Registry scoping supplies the authorization
boundary.

### Use a digest prefix

Rejected. The complete sha512 digest is already in package metadata and the
lockfile, is not a credential, and is small relative to the request. A prefix
adds collision ambiguity and an arbitrary security parameter while saving
negligible bandwidth.

### Encode the revision as an SRI option

The
[current SRI grammar](https://www.w3.org/TR/sri/#the-integrity-attribute)
permits:

```text
sha512-<base64-digest>?pnpr-patch=echo-r2
```

and requires unknown options to be ignored. npm's
[`ssri`](https://github.com/npm/ssri) library preserves the option while
verifying only the underlying digest. This is a useful redundant lockfile hint,
but it is not revision identity: changing the option does not change the
content, and the option is not authenticated by the hash.

This RFC permits pnpm to round-trip a bounded namespaced hint while retaining
the structured `dist.revisions` record for provider, revision, fixes, and
provenance. Support remains optional until compatibility is tested across
package managers and third-party SRI consumers.

## Implementation

1. Add parsing and validation for same-registry
   `/-/tarballs/<algorithm>/<digest>` URLs.
2. Verify that the decoded URL digest exactly matches the strongest supported
   hash expression in `dist.integrity`, excluding SRI options.
3. In lockfile serialization, strip the effective registry base from a
   validated integrity URL and retain the registry-relative path.
4. In lockfile hydration, resolve that path against the effective registry,
   preserving registry path prefixes such as `/~main/`.
5. Extend tarball URL verification to accept exact URL/integrity pairs from
   `dist.revisions`, not only current `dist`.
6. Fetch through the existing remote-tarball fetcher and SRI verifier without
   redirect, header, or recovery behavior.
7. Preserve bounded SRI revision hints through resolution and lockfile
   serialization without using them for content, change detection, or policy
   identity.
8. Add an explicit patch-revision refresh operation that updates resolution
   and snapshot atomically.

Tests should cover:

- an integrity URL resolved and fetched directly in one request;
- URL digest and SRI agreement, including base64/base64url conversion;
- malformed, abbreviated, unsupported, and mismatched digests;
- a namespaced SRI option round-tripping while verification uses only the
  complete digest;
- unrecognized SRI options remaining verification-compatible and policy-inert;
- an option-only change causing no download or new content-store object;
- same-origin enforcement and cross-origin rejection;
- absolute packument URL becoming a registry-relative lockfile path;
- a named registry path prefix surviving lockfile serialization and hydration;
- changing the configured registry base without changing the lockfile;
- an old canonical lockfile continuing to fetch original bytes;
- historical URL verification through an exact `dist.revisions` pair;
- current URL verification through ordinary `dist`;
- failure for a digest URL absent from both current and historical metadata;
- no extra network request, redirect, or integrity pass;
- offline install from the existing content-addressed store;
- revision refresh changing integrity and snapshot but not package version.

## Prior Art

- The [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
  retrieves content by immutable digest as well as mutable tag.
- [Subresource Integrity](https://www.w3.org/TR/SRI/) already supplies the
  digest format and verification model used by npm lockfiles.
- pnpm already stores non-canonical tarball URLs and resolves relative lockfile
  tarball paths against the configured registry.
- pnpm's content-addressed store already separates package resolution identity
  from physical file content.

## Unresolved Questions and Bikeshedding

- Should the route be `/-/tarballs/` or a pnpr-specific `/-/pnpr/tarballs/`?
- Should a later lockfile format replace the relative path with a compact
  content-addressed marker once registry capability is durable?
- Should the client require sha512 exclusively or support future algorithms
  through an allow-list?
- What is the final name and adoption policy for `pnpm update --patches`?
- Should pnpm emit or merely preserve `pnpr-patch=<provider-revision>` while
  SRI option semantics remain unstandardized?
