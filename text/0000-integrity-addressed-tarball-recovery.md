# Integrity-addressed recovery for mutable registry tarballs

## Summary

pnpm should send a registry package's expected lockfile integrity when it
fetches the canonical tarball URL. A pnpr registry implementing integrity-
addressed patch revisions can use that value to return the exact historical
revision in the first response, even after the same `name@version` has acquired
a newer current artifact.

If a registry or intermediary ignores the request field and the download fails
integrity verification, pnpm should discard the response, refresh the package
document, and retry through the registry's immutable integrity-addressed URL
only when the document advertises the complete locked integrity for that exact
package version. The retry verifies the original lockfile integrity again.
pnpm never adopts the response's checksum or the registry's current checksum
implicitly.

This is the client half of
[`pnpr/text/0000-integrity-addressed-patch-revisions.md`](../pnpr/text/0000-integrity-addressed-patch-revisions.md).

## Motivation

pnpm omits canonical registry tarball URLs from most lockfile entries. It stores
the integrity and reconstructs the URL from registry, package name, and version:

```yaml
packages:
  ejs@2.7.4:
    resolution:
      integrity: sha512-<echo-r1>
```

That makes lockfiles compact and portable across registry base URLs. It also
means a later fetch requests:

```text
https://registry.example/ejs/-/ejs-2.7.4.tgz
```

with no indication that the lockfile expects revision 1 rather than the
registry's current revision 2. If the canonical URL is mutable, pnpm downloads
revision 2 and correctly fails the revision 1 integrity check.

The missing selector already exists in the lockfile. Sending it to the registry
lets the server return revision 1 while the full client-side SRI verification
retains exactly the same security boundary.

## Detailed Explanation

### First request: send the complete expected integrity

For a canonical HTTP(S) registry tarball resolution with a valid SRI string,
pnpm adds:

```http
Pnpr-Expected-Integrity: sha512-<complete-base64-digest>
```

The complete SRI value is sent, not a prefix. It is roughly one hundred bytes
for sha512 and is already available in memory. A prefix provides no meaningful
request-size benefit, can be ambiguous, and would create a second identifier
that is weaker than the value pnpm ultimately verifies.

The digest is not a secret or bearer capability. It already appears in the
package document, lockfile, revision history, and integrity-addressed URL.
pnpm sends it only to the registry origin from which it is about to request the
tarball, under the same HTTPS and authentication rules as that request.

It does reveal the exact historical revision in use to the registry and CDN,
which can imply patch posture for a private project. They need that selector to
serve the revision, and a unique prefix would reveal the same fact. Registry
operators should treat the field as package-access metadata and may redact it
from routine request logs.

The request field contains exactly one canonical, complete hash expression.
When an integrity set contains several algorithms, pnpm chooses the strongest
supported expression. It does not forward unbounded SRI options or an
attacker-controlled multi-value field. The patch-revision protocol should
require sha512 rather than treating a complete but collision-weakened legacy
digest as adequate artifact identity.

Registries that do not understand the field ignore it and behave exactly as
today. pnpm still checks the downloaded bytes against the expected SRI.

The field is added only for registry tarballs whose resolution has an expected
integrity. It is not added to arbitrary direct URLs, GitHub release assets, or
local tarballs. Auth selection and redirect credential behavior remain
unchanged.

### Successful negotiated response

A supporting pnpr registry looks up:

```text
(registry identity, package name, version, expected integrity)
```

and either returns the matching bytes or redirects to the immutable revision
URL. pnpm follows the redirect and performs its ordinary full integrity check.
The request field is a performance and representation-selection hint; it does
not reduce verification.

The server response varies on the request field. If an intermediary implements
HTTP `Vary` correctly, each requested integrity has a distinct cache entry.

The recommended CDN shape does not cache these negotiated canonical responses
at all. It forwards the field to pnpr, receives a small
`Cache-Control: private, no-store` redirect, and caches the redirected
integrity-qualified tarball by URL with an immutable TTL. This avoids placing
large bodies behind a high-cardinality header cache key.

CDNs generally require explicit configuration for this behavior. For example,
CloudFront cache/origin policies determine which viewer headers reach the
origin, and a positive minimum TTL can override origin `no-store`; Cloudflare
requires Cache Rules configuration to process arbitrary `Vary` fields; Fastly
supports `Vary` but still needs the canonical route to be pass/private for this
recommended profile. The registry RFC provides the concrete deployment rules
and links to the vendor documentation.

If the field is stripped, pnpm receives current bytes, rejects them against the
lockfile, and enters the recovery path below. CDN misconfiguration therefore
costs a request but cannot change the integrity pnpm accepts.

### Recovery after an integrity failure

An integrity failure normally remains a hard supply-chain error. The recovery
path is enabled only when all of these conditions hold:

1. The resolution is an npm registry tarball with a canonical URL derived from
   a known registry, package name, and version.
2. The lockfile contains a valid full integrity value.
3. The downloaded response fails that exact integrity check.
4. A cache-bypassing metadata refresh of the same package and registry exposes
   a supported revision-history field.
5. The history contains an exact full-integrity match under the same version.

pnpm then derives or reads the immutable revision URL, fetches it once, and
verifies the response against the original locked SRI. The first mismatched
response is never added to the content-addressed store and its discovered hash
is not written anywhere.

Pseudocode:

```ts
async function fetchRegistryTarball(resolution, pkg, registry) {
  const expected = resolution.integrity
  try {
    return await download(resolution.tarball, {
      expected,
      headers: {
        'pnpr-expected-integrity': expected,
      },
    })
  } catch (error) {
    if (!isTarballIntegrityError(error)) throw error
    if (!isCanonicalRegistryTarball(resolution.tarball, pkg, registry)) {
      throw error
    }

    const metadata = await fetchPackument(pkg.name, {
      registry,
      cache: 'reload',
    })
    const revision = findExactRevision(
      metadata.versions[pkg.version]?.dist?.revisions,
      expected
    )
    if (revision == null) throw error

    const revisionUrl = getIntegrityAddressedTarballUrl(
      resolution.tarball,
      expected
    )
    return download(revisionUrl, { expected })
  }
}
```

Any of the following preserves or produces the original integrity error:

- the metadata cannot be authenticated or fetched;
- the version or revision history is absent;
- only a digest prefix matches;
- the history belongs to another registry, package, or version;
- the historical endpoint returns `403`, `404`, or another failure;
- the second response does not match the full locked integrity.

There is no retry with `dist.integrity` and no automatic equivalent of
`--update-checksums`.

### Why check the package document before fallback?

The full lockfile integrity alone is enough to authenticate matching bytes, so
pnpm could safely try a deterministic digest URL immediately. Requiring the
package document has two additional properties:

- the registry explicitly advertises support for historical revisions instead
  of pnpm probing a made-up path on every integrity error;
- the registry affirms that the digest belongs to this exact package version
  and remains within its retention and authorization policy.

This metadata check cannot make pnpm accept bad bytes. A malicious package
document can cause denial of service or point at unavailable content, but the
second full integrity verification still gates store insertion and execution.

The initial request header avoids this metadata round trip in the normal pnpr
case. The fallback exists for stale or non-conforming intermediaries.

### No special lockfile URL is required

The integrity-qualified URL is a transport fallback, not the logical
resolution. pnpm keeps the canonical resolution and expected integrity in the
lockfile. It does not replace `resolution.tarball` with the deployment-specific
fallback URL.

This preserves pnpm's existing behavior:

- canonical URLs can be reconstructed against the currently configured
  registry;
- the lockfile does not acquire a pnpr host;
- the integrity distinguishes artifact revisions in the content-addressed
  store;
- a frozen lockfile never changes its expected bytes.

### Interaction with tarball URL verification

When pnpm verifies that a lockfile tarball URL is the one advertised by the
registry, it continues to compare the logical canonical URL with
`versions[version].dist.tarball`. The integrity-qualified request is an
internal retrieval path and does not turn into a lockfile-provided alternate
origin.

This distinction is important: a tampered lockfile cannot use the recovery
feature to authorize an arbitrary tarball host. The fallback stays on the same
registry origin and is derived from a URL that already passed canonical
registry checks.

### Existing retry behavior

pnpm currently retries a tarball request after an integrity failure in case a
proxy or network cache served stale or corrupted content. Integrity-addressed
recovery should occur after the ordinary same-URL retry budget, unless the
server's response explicitly indicates that it does not have the requested
revision at the canonical representation.

The final error should explain both attempts:

```text
The canonical tarball did not match the integrity recorded in pnpm-lock.yaml.
The registry advertised that integrity as a historical revision, but the
integrity-addressed tarball also failed verification.
```

This must remain a high-signal supply-chain failure, not a warning.

### Refreshing selected patch revisions

Recovery preserves a locked artifact. It does not adopt the registry's current
artifact.

A separate explicit command should refresh artifact revisions without changing
semver selections:

```text
pnpm update --patches
```

For each registry package already selected in the lockfile, pnpm resolves the
same exact `name@version`. If current `dist.integrity` differs and the registry
identifies it as a compatible patch revision, pnpm updates the resolution
integrity. No dependency range is reselected.

The exact command name and whether ordinary non-frozen installs automatically
adopt a new revision remain open policy choices.

### Scope and coexistence

The current pnpm lockfile key still permits only one artifact revision of a
registry `name@version` in a dependency graph. This RFC supports a registry
moving that identity globally from one revision to another while different
lockfiles remain reproducible. It does not allow revision 1 and revision 2 to
coexist in one lockfile graph under the same key.

Likewise, this RFC does not solve collision between the same `name@version`
from different named registries. Registry identity in package keys remains a
separate lockfile change.

## Rationale and Alternatives

### Fail immediately on integrity mismatch

This remains correct for every registry that has not advertised revision
history. It is insufficient for a registry that intentionally has several
retained, integrity-addressed representations of the same version: the
lockfile already says exactly which representation it wants.

### Download current first, then recover

This requires no request field, but every historical cache miss downloads and
hashes a tarball known to be wrong before retrieving the desired one. Sending
the expected integrity makes the common path one request. The failure-triggered
lookup remains as compatibility insurance.

### Put the integrity only in the URL

Persisting an integrity-qualified URL would make revision identity explicit,
but pnpm would retain the non-canonical absolute URL in the lockfile and pin
the registry deployment host. Deriving it only when needed preserves canonical
URL reconstruction and registry portability.

### Use `Want-Content-Digest` or `Want-Repr-Digest`

[RFC 9530](https://www.rfc-editor.org/rfc/rfc9530.html) defines these fields to
request a digest in the response and express algorithm preferences. They do
not ask the server for a representation matching a digest the client already
knows, so overloading them would give an existing standard field new and
surprising semantics.

### Send an integrity prefix

Rejected. pnpm already has the full value, the bandwidth saving is negligible,
and prefix matching introduces ambiguity and collision handling. Full response
verification would keep a sufficiently long prefix from directly accepting
bad bytes, but there is no reason to weaken the selection key.

The full value also discloses no new registry information: it is already in
the package document and the deterministic revision URL. pnpm should instead
strictly bound and parse the complete header value.

## Implementation

1. Extend registry tarball download options with the package identity,
   registry identity, canonical-URL status, and full expected integrity needed
   for negotiation and recovery.
2. Add `Pnpr-Expected-Integrity` to eligible requests without forwarding
   registry credentials to a new origin.
3. Preserve current same-URL retries and catch only the typed final tarball
   integrity failure for the recovery path.
4. Fetch full or abbreviated metadata without stale cache reuse and parse the
   revision history defensively.
5. Require an exact SRI match and derive a same-origin, URL-safe revision URL.
6. Download once through the existing tarball fetcher and full integrity
   verifier; never mutate the logical resolution or lockfile as part of
   recovery.
7. Add reporting that distinguishes successful historical recovery from a
   second integrity failure.
8. Add an explicit revision-refresh operation.

Tests should cover:

- a header-aware registry returning the requested old revision on the first
  request;
- a normal registry ignoring the field with no behavior change;
- a proxy stripping the field, canonical integrity failure, metadata
  confirmation, and successful immutable-URL recovery;
- a CDN forwarding the field but bypassing cache for the canonical redirect,
  followed by a cacheable immutable revision request;
- duplicate, oversized, multi-value, option-bearing, and abbreviated request
  integrity values being rejected or never emitted;
- history absent, malformed, wrong version, wrong package, wrong registry,
  abbreviated digest, ambiguous digest, and unsupported algorithm;
- the first response never reaching the content-addressed store;
- the second response being checked against the original lockfile SRI;
- no fallback for direct tarball, git, file, or non-canonical URLs;
- authentication and redirect behavior staying on the registry origin;
- frozen lockfile content remaining byte-for-byte unchanged;
- `--offline` succeeding only when the locked integrity already exists in the
  content-addressed store;
- revision refresh changing integrity but not selected version;
- clear errors when retrieval is forbidden by registry policy.

## Prior Art

- The [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
  allows clients to retrieve a repository manifest by tag or digest. The
  request-integrity header makes an npm canonical URL act like a tag request
  carrying the desired digest, while the fallback URL is a direct digest
  reference.
- [Subresource Integrity](https://www.w3.org/TR/SRI/) already defines the
  lockfile checksum format and verification model used by npm tooling.
- HTTP [`Vary`](https://www.rfc-editor.org/rfc/rfc9110.html#name-vary)
  provides the standard cache-key semantics for a response selected by a
  request field.
- [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530.html) provides optional
  response `Content-Digest` and `Repr-Digest` fields, complementary to pnpm's
  npm SRI verification.

## Unresolved Questions and Bikeshedding

- Should the request field be `Pnpr-Expected-Integrity`,
  `Package-Expected-Integrity`, or a field proposed for broader registry use?
- Should pnpm send the field on every canonical registry request with SRI, or
  only after a registry advertises protocol support? Frozen installs cannot
  rely on a fresh packument to discover support.
- Should a successful historical recovery be silent, debug-logged, or shown in
  the install summary?
- Should recovery begin only after current same-URL retries are exhausted, or
  may a supporting server signal the historical redirect immediately?
- What is the final name and update policy for `pnpm update --patches`?
