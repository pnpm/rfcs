# Integrity-addressed patch revisions for pnpr

## Summary

pnpr should be able to project a provider's patched build directly over the
original package's `name@version`, without exposing a provider alias or
inventing a version suffix. Every accepted artifact is stored and served from
an immutable, registry-scoped URL identified by its complete sha512 digest:

```text
https://registry.example/-/tarballs/sha512/<base64url-digest>
```

The selected version's `dist.tarball` points directly at that URL.
`dist.integrity` contains the corresponding SRI value plus a compact registry
revision option:

```text
sha512-<original-digest>?       original artifact
sha512-<first-patch-digest>?v1  first replacement
sha512-<second-patch-digest>?v2 second replacement
```

The empty option is the original baseline. Positive `vN` values number
replacement artifacts. The package document also advertises an append-only
history of the original artifact and every accepted replacement for that
`name@version`.

The option has two client-facing purposes:

- it tells pnpm that the integrity-addressed URL can be constructed from the
  effective registry and complete digest;
- it makes an integrity change understandable in a lockfile diff.

The option is not part of the content address or byte verification. The
complete algorithm and digest remain the only artifact identity.

The ordinary canonical npm URL
`<registry>/<name>/-/<name>-<version>.tgz` never changes: it continues to serve
the original upstream artifact. Consequently, legacy lockfiles remain
reproducible. Fresh resolutions in pnpm, npm, and Yarn receive the selected
artifact directly from the immutable URL advertised in `dist.tarball`.

This makes the identity split explicit:

> `name@version` selects the registry's current approved artifact.
> The complete integrity identifies immutable bytes.
> The SRI option exposes the registry revision and retrieval convention.

The companion pnpm RFC specifies how marked integrities replace tarball URLs in
the pnpm lockfile and how pnpm refreshes artifact revisions without changing
package versions.

## Motivation

### Patch providers preserve the original version

Vendors such as Echo produce security-remediated builds of existing package
versions. An application asked for `ejs@2.7.4`; the patched artifact still
represents `ejs@2.7.4`, and semver ranges, peer dependencies, and runtime
version checks should continue to see `2.7.4`.

Encoding the provider into a consumer-visible alias gives registry policy a
large client surface:

```jsonc
{
  "pnpm": {
    "overrides": {
      "ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"
    }
  }
}
```

The provider scope must then be routed and authenticated on every client. A
different package name enters lockfiles, SBOMs, license tools, advisory
databases, and runtime metadata. A second provider patch also requires another
provider-side package identity or version even though the provider's delivery
scheme is not part of the application's dependency identity.

For a registry-wide remediation policy, the intended operation is simpler:
when this registry resolves `ejs@2.7.4`, its current approved artifact is
Echo's second replacement of `ejs@2.7.4`.

### One upstream version can have several artifact revisions

A provider can issue one replacement for a vulnerability and another when a
later vulnerability is found in the same old release line:

```text
ejs@2.7.4 + sha512-<upstream>?  original artifact
ejs@2.7.4 + sha512-<echo-r1>?v1 first replacement
ejs@2.7.4 + sha512-<echo-r2>?v2 current replacement
```

Every accepted artifact must remain reproducible. Encoding these revisions as
names creates one package document per patch. Encoding them as versions
introduces prerelease semantics and changes the version reported by the
package.

Integrity already authenticates npm tarballs and distinguishes their bytes. A
small neutral revision ordinal can explain the sequence without becoming
artifact identity or exposing the provider's own naming system.

### Mark the original before a patch exists

A lockfile is often created before a package needs remediation. If only patched
artifacts receive a marker, that old lockfile cannot know that the registry
later gained an integrity-addressed historical endpoint.

pnpr therefore emits a revision-aware integrity and integrity URL for every
package version served through this protocol, including an unmodified original:

```text
sha512-<upstream>?
```

pnpm can immediately lock the original by digest. If `v1` is selected later,
the existing lockfile still requests the original digest while a newly resolved
lockfile records the replacement digest and `?v1`.

### Immutable URLs preserve lockfiles and make CDNs fast

Serving new bytes from an old canonical URL breaks lockfiles and makes shared
caches ambiguous. Sending an integrity selector in a request header could
recover old content, but it requires special CDN forwarding and either a
varying cache key or an extra redirect.

A digest in the URL needs neither. Every representation has one ordinary,
immutable cache key. Package managers resolving current metadata follow
`dist.tarball`, while revision-aware pnpm lockfiles construct the same route
directly from registry plus integrity.

## Detailed Explanation

### Terminology

- An **original identity** is an npm package `name@version`, for example
  `ejs@2.7.4`.
- An **artifact** is one exact tarball for that identity, identified by its
  complete sha512 digest.
- A **registry revision** is the stable ordinal assigned to an accepted
  artifact for that identity. The original is revision zero and is encoded by
  an empty SRI option. Replacements are encoded as `v1`, `v2`, and so on.
- A **provider revision** is an optional provider-specific identifier such as
  `echo-r2`. It is provenance metadata and is not exposed in the SRI option.
- The **selected revision** is the artifact advertised by the current version
  document.
- The **canonical URL** is the ordinary npm tarball URL derived from registry,
  name, and version.
- The **integrity URL** is a registry-scoped content address containing the
  complete digest.

### Invariants

1. **Full sha512 is artifact identity.** A truncated digest or SRI option is
   never an object key, authorization input, or security decision.
2. **Every tarball URL is immutable.** The canonical URL remains on the
   original artifact, and an integrity URL always returns the bytes named by
   its digest.
3. **Every protocol-aware integrity is marked.** The original ends in `?`;
   replacements end in canonical `?vN`.
4. **Revision ordinals are stable.** A distinct accepted replacement receives
   the next positive integer. Numbers are never reassigned. Selecting a
   previously accepted artifact again uses its existing ordinal.
5. **History is append-only.** Selecting a revision never removes or mutates an
   older artifact record or object.
6. **Content addressing is registry-scoped.** A digest locates an object inside
   one addressed registry; it does not bypass authorization or expose a global
   content store.
7. **One selected revision per identity.** Within one projected registry,
   exactly one revision of a `name@version` is current. Provider conflicts are
   configuration errors, not implicit precedence.
8. **Metadata is revision-bound.** Projected version metadata is derived from
   the same verified artifact as `dist.integrity`; a revision update replaces
   integrity and dependency metadata together.
9. **The lockfile wins.** Existing resolutions keep their digest until an
   explicit resolution or revision-refresh operation changes them.

### Provider input remains integrity-pinned

Patch providers publish a signed manifest mapping an original identity to an
immutable artifact. The artifact may be stored as a provider package or an
opaque tarball; that delivery encoding is not exposed to consumers:

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

pnpr verifies the manifest signature, downloads the artifact, verifies its
complete integrity, and stores it in the selected registry's
content-addressed store before making it visible.

The verified tarball must contain the original `name` and `version`. A provider
may use aliases or revision identifiers in its distribution system, but the
artifact offered for projection must identify itself as `ejs@2.7.4`. pnpr does
not rewrite `package.json` after verification because that would produce bytes
different from the provider's signed integrity.

Illustrative configuration:

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

Provider credentials and artifact locations remain server-side. Consumers
configure only the `main` registry; they do not configure an `@echo-patch`
scope or named Echo registry.

### Integrity URL

The content-addressed route is relative to the addressed npm registry:

```text
<registry-base>/-/tarballs/<algorithm>/<base64url-digest>
```

Examples:

```text
https://registry.example/-/tarballs/sha512/AbCd...
https://pnpr.example/~main/-/tarballs/sha512/AbCd...
```

The complete sha512 digest uses base64url without padding. Raw SRI base64 is not
embedded because `/`, `+`, and `=` have awkward URL and intermediary
semantics. Package metadata uses standard SRI base64 plus the revision option:

```text
sha512-<base64-digest>?
sha512-<base64-digest>?v2
```

The option is excluded when deriving the URL. Two marked SRI strings with the
same algorithm and digest therefore locate the same immutable object even if
their displayed ordinals differ.

The package name and version are absent from the URL. The digest locates the
object in the registry's content-addressed index, deduplicates identical
tarballs, and keeps the endpoint independent of provider naming.

The registry base remains load-bearing for security. A request to
`/~main/-/tarballs/...` is authorized in `main` and succeeds only if that
registry has an allowed reference to the digest. pnpr may deduplicate physical
storage globally, but it must not expose an object merely because another
organization or registry stored the same digest. Knowledge of a digest is not
a bearer credential.

Successful public responses are ordinary immutable CDN objects:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
ETag: "<full-digest>"
Content-Digest: sha-512=:<base64>:
```

Private registries use cache directives compatible with their authorization
model. pnpr verifies content against the digest before storing or serving it.

### Revision-aware SRI options

The
[Subresource Integrity specification](https://www.w3.org/TR/sri/#the-integrity-attribute)
allows every hash expression to carry zero or more options:

```text
hash-with-options = hash-expression *("?" option-expression)
option-expression = *VCHAR
```

The empty option is valid. This RFC assigns neutral registry revision semantics
to two forms:

```text
<hash-expression>?     original artifact
<hash-expression>?vN   replacement revision N
```

`N` is a positive decimal integer without leading zeroes. The original is
conceptually revision zero, but omitting the number keeps `v1` aligned with the
first actual replacement.

pnpr applies these rules:

- it emits the empty option for original artifacts served through the digest
  protocol;
- it emits `v1` for the first distinct accepted replacement and monotonically
  increasing values for later replacements;
- it never uses the provider name or provider revision in the option;
- it preserves the option in `dist.integrity`, `dist.revisions`, audit output,
  and logs;
- it excludes the option from object lookup, equality, authorization, VEX, and
  integrity verification;
- it treats structured revision metadata as authoritative if an option is
  forged or inconsistent;
- an option-only change cannot create a new stored artifact.

Conforming SRI consumers ignore unrecognized options. pnpm additionally
recognizes the empty option and `vN` as a durable signal to construct the digest
URL. pnpm parses the raw SRI form rather than relying on generic tooling to
preserve an empty option.

### Projected package document

For a covered version, pnpr projects the selected artifact under the original
name and version. `dist.tarball` points directly to the immutable integrity URL.
`dist.revisions` advertises the original artifact and every accepted provider
replacement:

```jsonc
{
  "name": "ejs",
  "versions": {
    "2.7.4": {
      "name": "ejs",
      "version": "2.7.4",
      // metadata derived from the selected echo-r2 tarball
      "dependencies": {},
      "dist": {
        "tarball": "https://registry.example/-/tarballs/sha512/<echo-r2-base64url>",
        "integrity": "sha512-<echo-r2-base64>?v2",
        "revisions": [
          {
            "revision": 0,
            "integrity": "sha512-<upstream>?",
            "tarball": "https://registry.example/-/tarballs/sha512/<upstream-base64url>",
            "source": "upstream",
            "publishedAt": "2020-04-03T00:00:00.000Z"
          },
          {
            "revision": 1,
            "integrity": "sha512-<echo-r1>?v1",
            "tarball": "https://registry.example/-/tarballs/sha512/<echo-r1-base64url>",
            "source": "echo",
            "providerRevision": "echo-r1",
            "publishedAt": "2026-05-10T00:00:00.000Z",
            "fixes": ["GHSA-..."],
            "attestations": ["https://..."]
          },
          {
            "revision": 2,
            "integrity": "sha512-<echo-r2>?v2",
            "tarball": "https://registry.example/-/tarballs/sha512/<echo-r2-base64url>",
            "source": "echo",
            "providerRevision": "echo-r2",
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

The selected revision is the entry whose algorithm, complete digest, and
registry revision equal `dist.integrity`. Existing npm-compatible clients
ignore `dist.revisions` and fetch selected `dist.tarball` normally.

The history provides:

- discoverability through `npm view` and registry APIs;
- provenance and advisory/VEX metadata for exact bytes;
- an auditable sequence when a provider reissues a patch;
- the allow-list used when a verifier checks a historical digest;
- retention and withdrawal state if the schema grows it later.

The full packument may carry all provenance fields. Abbreviated metadata should
keep current `dist` plus the integrity and digest URL of historical revisions
needed for verification. An unpatched version does not need a
`dist.revisions` array, but its `dist.tarball` still uses the digest URL and its
`dist.integrity` still ends in `?`.

### The canonical URL remains the original artifact

The canonical npm tarball route is never repointed:

```text
https://registry.example/ejs/-/ejs-2.7.4.tgz
```

It continues returning the original upstream bytes forever. Current package
metadata may advertise another artifact through `dist.tarball`, just as npm
package documents may already advertise non-canonical tarball URLs.

This provides backward compatibility:

- a legacy lockfile reconstructs the canonical URL and receives original bytes;
- a revision-aware original lockfile constructs the original digest URL;
- a patched lockfile constructs its replacement digest URL;
- changing the selected revision changes only mutable package metadata;
- no CDN URL ever refers to two bodies.

There is no need to use an integrity mismatch as protocol negotiation.

### Manifest metadata across revisions

A patch may update dependencies as well as source files. Revisions are not
required to have identical dependency metadata.

pnpr reads the selected artifact's `package.json` only after the tarball passes
integrity verification and projects its resolution-relevant fields into the
current version document. When a revision becomes selected, the entire
projected version entry and `dist` move atomically.

An old lockfile retains its old digest and dependency snapshot. A pnpm revision
refresh must replace both together; changing only the checksum would be
incorrect.

Two lockfiles can therefore hold different dependency graphs for the same
`name@version`. Current pnpm package keys cannot represent both revisions
simultaneously in one graph. Supporting that would require integrity in the
package key as a separate lockfile-format change.

### CDN and installation performance

The normal data path remains:

```text
packument resolution → one tarball URL → one CDN request
```

A frozen revision-aware pnpm install skips packument resolution for artifact
selection and constructs that same URL from its lockfile.

There is no custom header, `Vary` cache key, redirect, metadata recovery
request, or intentionally failed download. A warm CDN serves the object without
contacting pnpr. On a miss, pnpr performs one content-addressed lookup.

The URL is longer by one sha512 digest. Patched packuments contain a small
revision list. Both costs are negligible relative to tarball bodies. pnpr
should precompute projected packuments and index objects directly by binary
digest rather than scanning revision arrays on the fetch path.

### Selecting and updating revisions

A frozen install never changes revision. An ordinary install may continue to
prefer its lockfile, just as it does for package versions.

pnpm should provide an explicit operation to refresh only artifact revisions:

```text
pnpm update --patches
```

For every locked registry package, pnpm resolves the same exact
`name@version`. If `dist.integrity` changed, it replaces the marked integrity
and package snapshot atomically. It does not rerun semver selection or change
package versions.

Whether non-frozen `pnpm install` automatically adopts a new revision remains a
pnpm policy question.

### Caching and atomic provider refresh

pnpr indexes artifact content by:

```text
(concrete registry identity, algorithm, complete digest)
```

Physical bytes may be deduplicated beneath that authorization index. A provider
refresh that accepts a distinct artifact follows this order:

1. Download and verify the tarball.
2. Store it under its immutable integrity URL.
3. Derive and validate projected version metadata from the verified tarball.
4. Assign the next stable registry revision.
5. Append its structured revision metadata.
6. Atomically update the projected version entry and `dist`.
7. Invalidate projected full and abbreviated packuments.

No tarball cache object is invalidated or overwritten. Metadata never advertises
an integrity URL before it is available.

### Authorization, retention, and enforcement

Both current and historical integrity URLs apply the addressed registry's
access policy. History and availability remain separate: an operator may
retain a revision record for audit while a legal or security policy refuses its
bytes. Such a request returns a policy-specific `403`.

This matters for a vulnerable original. A reproducibility policy may retain it
indefinitely; strict enforcement may deliberately block it even though an old
lockfile then cannot install through that registry. Patch selection must not
silently weaken a registry's separate screening policy.

### Auditing a revision

Advisories apply to the original `name@version`; provider VEX statements apply
to one exact integrity. `dist.revisions[].fixes` and attestations preserve that
mapping without changing package identity.

An audit request carrying only `name@version` is evaluated conservatively
against the base version. An integrity-aware pnpm audit extension can identify
the installed revision and subtract only fixes declared for those exact bytes.
A later provider revision does not retroactively add fixes to an older locked
revision.

## Rationale and Alternatives

### Provider aliases and `_pnprPatch`

The alternative patch-provider RFC exposes a provider package such as
`@echo-patch/ejs` and asks the workspace to write an alias override or adds a
registry annotation instructing pnpm to do it.

That representation keeps package publication immutable and makes provider
provenance conspicuous, but turns registry policy into client-level package
substitution. It requires provider-registry configuration, creates separate
provider package documents, and depends on pnpm-specific annotation or override
behavior.

Integrity-addressed revisions keep provenance in registry metadata and present
ordinary `dist` entries to npm-compatible clients.

### Encode the revision in the version

Versions such as `2.7.4-echo.2` are prereleases under semver. Ordinary ranges
containing `2.7.4` do not necessarily select them, peer ranges may reject them,
and the package no longer reports the version the application was tested
against. Build metadata is unsuitable because semver ignores it for precedence
and equality.

### Encode the revision in a package name

Names such as `@echo-patch/ejs-r1` and `@echo-patch/ejs-r2` avoid prerelease
semantics but create one package document per patch and expose provider
internals throughout consumer tooling.

### Use a named provider registry

A specifier such as `echo:ejs@2.7.4` keeps the original name and version, but
every consumer must configure the `echo` registry locally. Current pnpm
lockfiles also cannot safely represent two named registries serving different
bytes for the same `name@version` in one graph.

Named registries remain coherent when the provider is intentionally the origin
for all packages. They are a poor encoding for a central registry adopting
selected patch artifacts.

### Mutate the canonical URL and recover by integrity

The registry could replace the bytes at the canonical name-and-version URL. An
old pnpm could download the replacement, observe an integrity mismatch, and
retry a digest endpoint using its locked checksum.

That costs a complete failed download for old lockfiles, breaks other existing
clients, and makes one CDN URL refer to several bodies over time. Keeping the
canonical URL on the original bytes and selecting current content through
`dist.tarball` avoids all three problems.

### Select integrity through a request header

A complete integrity request header could select a historical representation,
but CDNs must forward it and either vary large cached bodies on a
high-cardinality field or return an extra redirect.

Direct integrity URLs use ordinary cache keys and require one request.

### Include package name and version in the integrity URL

A format such as `<canonical-url>+sha512.<digest>` makes the relationship
human-readable and allows routing without a digest-to-package index. It also
duplicates identical content under several URLs and makes content addressing
depend on package naming.

The digest-only route deduplicates naturally and lets pnpm construct the URL
from registry plus integrity. Registry-scoped authorization provides the
package-policy boundary omitted from the path.

### Use only a digest prefix

Rejected. A full sha512 value is small relative to an HTTP request or tarball.
Prefixes introduce ambiguity, collision work, and an arbitrary minimum length.
The complete digest is package metadata, not a credential.

### Use `?v0` for the original

`?v0` would work, but the empty option gives the original a visibly distinct
baseline while allowing the first actual replacement to be `v1`:

```text
?    original
?v1  first replacement
```

The SRI grammar permits an empty option. pnpm parses it directly rather than
delegating preservation of protocol state to a generic SRI library.

### Put provider identity in the SRI option

An option such as `?pnpr-patch=echo-r2` leaks the implementation and provider
scheme into a generic integrity field. It also makes a provider rename look
like an artifact change.

Neutral `vN` ordinals are sufficient for lockfile visibility. Structured
`dist.revisions` and signed provider manifests remain authoritative for
provenance, fixes, attestations, and policy.

### Omit structured revision history

The SRI option alone cannot authenticate provider identity, fixes, or
attestations. It also cannot tell a verifier whether a historical digest was
ever associated with a particular `name@version`.

The compact suffix complements but does not replace `dist.revisions`.

## Implementation

### pnpr

1. Add provider manifest configuration, signature verification, pinned refresh,
   and content-addressed ingestion.
2. Validate every artifact's complete integrity and original `name@version`,
   then derive projected version metadata from its verified `package.json`.
3. Add a registry-scoped digest index that separates physical deduplication
   from authorization.
4. Serve immutable `<registry-base>/-/tarballs/<algorithm>/<digest>` routes
   with range request, CDN, and access-policy support.
5. Expose original upstream artifacts through the digest route even before any
   replacement exists.
6. Emit an empty SRI option for originals and stable `vN` options for
   replacements.
7. Project selected digest URLs and append-only `dist.revisions` into full and
   abbreviated packuments.
8. Keep original canonical tarball routes immutable.
9. Make provider refresh atomic and audit every selected-revision transition.
10. Extend audit handling to understand integrity-scoped fixes and
    attestations.

Tests should cover:

- unpatched originals exposed through a digest URL with a trailing `?`;
- original, first replacement, and second replacement retrievable by digest;
- `v1` assigned to the first replacement and stable monotonic ordinals;
- malformed revision options rejected from protocol metadata;
- `dist.tarball` selecting the latest revision without changing name/version;
- canonical URLs continuing to return original bytes after every refresh;
- full and abbreviated metadata exposing required revision history;
- SRI options excluded from digest lookup, byte verification, and policy;
- forged revision options unable to change accepted bytes;
- provider revisions represented only in structured metadata;
- the same physical digest referenced by two authorized registries without
  cross-registry disclosure;
- a known digest requested through an unauthorized registry returning no
  bytes;
- immutable CDN caching, range requests, and warm-CDN behavior;
- atomic refresh with no metadata-visible-but-unavailable interval;
- selected metadata derived from the verified tarball;
- a dependency-changing revision updating snapshot and integrity together;
- provider conflict rejection;
- retention and screening policy on historical objects;
- audit/VEX evaluation tied to exact integrity.

### pnpm

The companion RFC,
[`text/0000-integrity-addressed-registry-tarballs.md`][pnpm-rfc],
contains the client and lockfile changes.

[pnpm-rfc]: ../../text/0000-integrity-addressed-registry-tarballs.md

## Prior Art

- The [OCI Distribution Specification][oci-distribution]
  permits retrieval by mutable tag or immutable digest. This RFC applies the
  same current-pointer/content-identity split to npm artifacts.
- npm lockfiles already use
  [Subresource Integrity](https://www.w3.org/TR/SRI/) to authenticate tarballs.
- Content-addressed package stores in pnpm and systems such as Nix demonstrate
  that logical package identity and physical content identity can be separate.
- [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530.html) provides optional
  `Content-Digest` response fields complementary to npm SRI verification.

[oci-distribution]: https://github.com/opencontainers/distribution-spec/blob/main/spec.md

## Unresolved Questions and Bikeshedding

- Is `/-/tarballs/<algorithm>/<digest>` the right registry-neutral endpoint?
- Is `dist.revisions` the right package-document field name?
- Which projected manifest fields should historical entries retain?
- Does `pnpm update --patches` update every artifact revision, only
  security-motivated revisions, or accept provider and advisory filters?
- How long must a pnpr deployment retain historical content, and how should it
  communicate a policy shorter than lockfile lifetime?
- Should vulnerable historical artifacts remain retrievable by default or
  require an explicit reproducibility policy?
- Should registry revision ordinals remain stable across provider changes?
