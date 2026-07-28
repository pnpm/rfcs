# Integrity-addressed patch revisions for pnpr

## Summary

pnpr should be able to project a provider's patched build directly over the
original package's `name@version`, without exposing a provider alias or
inventing a version suffix. Every accepted artifact is stored and served from
an immutable, registry-scoped URL identified by its complete sha512 digest:

```text
https://registry.example/-/tarballs/sha512/<base64url-digest>
```

The selected version's `dist.tarball` points directly at that URL and
`dist.integrity` contains the corresponding SRI value. The package document
also advertises an append-only history of the original artifact and every patch
revision previously selected for that `name@version`.

Patched integrities may carry an optional, namespaced SRI option as a
human-readable revision hint:

```text
sha512-<base64-digest>?pnpr-patch=echo-r2
```

The option is preserved in metadata and lockfiles but is not part of the
content address or integrity comparison.

The ordinary canonical npm URL
`<registry>/<name>/-/<name>-<version>.tgz` never changes: it continues to serve
the original upstream artifact. Consequently, old lockfiles remain
reproducible in every existing npm-compatible client. Fresh resolutions in
pnpm, npm, and Yarn receive the selected patched artifact directly from its
immutable URL in one CDN request.

This makes the identity split explicit:

> `name@version` selects the registry's current approved revision.
> The complete integrity identifies immutable bytes.

The companion pnpm RFC specifies a host-portable relative lockfile encoding for
these URLs, historical URL verification, and an operation that refreshes
artifact revisions without changing package versions.

## Motivation

### Patch providers preserve the original version

Vendors such as Echo produce security-remediated builds of existing package
versions. An application asked for `ejs@2.7.4`; the patched artifact still
represents `ejs@2.7.4`, and semver ranges, peer dependencies, and runtime
version checks should continue to see `2.7.4`.

Encoding the provider into a consumer-visible alias gives that registry policy
a large client surface:

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
provider-side package identity or version, although the provider's delivery
scheme is not part of the application's dependency identity.

For a registry-wide remediation policy, the intended operation is simpler:
when this registry resolves `ejs@2.7.4`, its current approved artifact is
Echo's second patch of `ejs@2.7.4`.

### One upstream version can have several patch revisions

A provider can issue revision 1 for one vulnerability and revision 2 when
another vulnerability is found in the same old release line. Every accepted
artifact must remain reproducible:

```text
ejs@2.7.4 + sha512-<upstream>    original artifact
ejs@2.7.4 + sha512-<echo-r1>     first provider revision
ejs@2.7.4 + sha512-<echo-r2>     current provider revision
```

Encoding revisions as names creates one package document per patch. Encoding
them as versions introduces prerelease semantics and changes the version
reported by the package. Integrity already authenticates npm tarballs and
distinguishes their actual bytes, so it is the natural revision identifier.

### Immutable URLs preserve lockfiles and make CDNs fast

Serving new bytes from an old canonical URL breaks lockfiles and makes shared
caches ambiguous. Sending an integrity selector in a request header can recover
old content, but it requires special CDN forwarding and either a varying cache
key or an extra redirect.

A digest in the URL needs neither. Every representation has one ordinary,
immutable cache key. pnpm already knows the integrity before fetching, and all
npm clients already follow the `dist.tarball` URL from the package document.
The registry can therefore select the current revision in metadata while
keeping every tarball route immutable.

## Detailed Explanation

### Terminology and invariants

- An **original identity** is an npm package `name@version`, for example
  `ejs@2.7.4`.
- An **artifact revision** is one exact tarball for that identity, identified
  by its complete sha512 SRI value.
- The **selected revision** is the artifact advertised by the current version
  document.
- The **canonical URL** is the ordinary npm tarball URL derived from registry,
  name, and version.
- The **integrity URL** is a registry-scoped content address containing the
  complete digest.

The following invariants are required:

1. **Full sha512 is artifact identity.** A truncated digest or SRI option is
   never an object key, authorization input, or security decision.
2. **Every tarball URL is immutable.** The canonical URL remains on the
   original artifact, and an integrity URL always returns the bytes named by
   its digest.
3. **History is append-only.** Selecting a new revision never removes or
   mutates an older revision record or object.
4. **Content addressing is registry-scoped.** A digest locates an object inside
   one addressed registry; it does not bypass that registry's authorization or
   expose a global content store.
5. **One selected revision per identity.** Within one projected registry,
   exactly one revision of a `name@version` is current. Provider conflicts are
   configuration errors, not implicit precedence.
6. **Metadata is revision-bound.** The projected version metadata is derived
   from the same verified artifact as `dist.integrity`, and a revision update
   replaces the integrity and dependency snapshot together.
7. **The lockfile wins.** Existing resolutions keep their integrity URL until
   an explicit resolution or patch-refresh operation changes them.

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

pnpr verifies the manifest signature, downloads the artifact, verifies its full
integrity, and stores it in the selected registry's content-addressed store
before making it visible. A compromised provider can refuse to deliver a new
artifact, but it cannot replace bytes pnpr already accepted under an integrity.

The verified tarball must contain the original `name` and `version`. A provider
may use aliases or revision identifiers in its distribution system, but the
artifact offered for projection must still identify itself as `ejs@2.7.4`.
pnpr does not rewrite `package.json` after verification because that would
produce bytes different from the provider's signed integrity.

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

The content-addressed route is relative to the npm registry root:

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
semantics. The corresponding package metadata continues to use standard SRI:

```text
sha512-<base64-digest>?pnpr-patch=echo-r2
```

The optional suffix is excluded when deriving the URL. Two SRI strings with
the same algorithm and digest but different options therefore identify the
same immutable object.

The package name and version are intentionally absent from the integrity URL.
The digest alone locates the object in the registry's content-addressed index,
deduplicates identical tarballs, and keeps the endpoint independent of any
provider revision naming scheme.

The registry base remains load-bearing for security. A request to
`/~main/-/tarballs/...` is authorized in `main` and succeeds only if that
registry has an allowed reference to the digest. pnpr may deduplicate physical
storage globally, but it must not expose a global `/-/tarballs/...` object merely
because another organization or registry stored the same digest. Knowledge of
a digest is not a bearer credential.

Successful responses are ordinary immutable CDN objects:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
ETag: "<full-digest>"
Content-Digest: sha-512=:<base64>:
```

Private registries use cache directives compatible with their authorization
model. The response body is always verified against the digest before pnpr
stores or serves it.

### Projected package document

For a covered version, pnpr projects the selected artifact under the original
name and version. Its `dist.tarball` points directly to the immutable integrity
URL. `dist.revisions` advertises the original artifact and every accepted
provider revision:

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
        "integrity": "sha512-<echo-r2-base64>?pnpr-patch=echo-r2",
        "revisions": [
          {
            "integrity": "sha512-<upstream>",
            "tarball": "https://registry.example/-/tarballs/sha512/<upstream-base64url>",
            "source": "upstream",
            "publishedAt": "2020-04-03T00:00:00.000Z"
          },
          {
            "integrity": "sha512-<echo-r1>?pnpr-patch=echo-r1",
            "tarball": "https://registry.example/-/tarballs/sha512/<echo-r1-base64url>",
            "source": "echo",
            "revision": "echo-r1",
            "publishedAt": "2026-05-10T00:00:00.000Z",
            "fixes": ["GHSA-..."],
            "attestations": ["https://..."]
          },
          {
            "integrity": "sha512-<echo-r2>?pnpr-patch=echo-r2",
            "tarball": "https://registry.example/-/tarballs/sha512/<echo-r2-base64url>",
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

The selected revision is the entry whose complete integrity metadata equals
`dist.integrity`.
There is no `_pnprPatch` instruction and no alternate package specifier for a
client to interpret. Existing npm-compatible clients ignore `dist.revisions`
and fetch the selected `dist.tarball` normally.

The history provides:

- discoverability through `npm view` and registry APIs;
- provenance and advisory/VEX metadata for each exact set of bytes;
- an auditable sequence when a provider re-issues a patch;
- the allow-list used when a client or verifier requests a historical digest;
- retention and withdrawal state, if the schema grows it later.

The full packument may carry all provenance fields. Abbreviated metadata should
keep the representation compact: current `dist`, plus the integrity and
integrity URL of historical revisions needed for lockfile verification. Most
packages have no patch revisions and pay no metadata cost.

The field name is bikesheddable. Its semantics are not: history is append-only,
keyed by complete integrity, and includes the original artifact.

### SRI options as a revision hint

The current
[Subresource Integrity specification](https://www.w3.org/TR/sri/#the-integrity-attribute)
allows each hash expression to carry zero or more `?option-expression`
suffixes:

```text
hash-with-options = hash-expression *("?" option-expression)
```

No standard option meanings are defined yet, and conforming consumers must
ignore unrecognized options. npm's
[`ssri`](https://github.com/npm/ssri) implementation already parses,
serializes, and verifies values such as:

```text
sha512-<base64-digest>?pnpr-patch=echo-r1
```

pnpr may use one option as a display hint:

```text
pnpr-patch=<provider-revision>
```

Rules:

- provider and revision use a restricted URL-safe character set and bounded
  length;
- pnpr preserves the option in `dist.integrity`, `dist.revisions`, audit
  output, and logs;
- pnpm preserves it in the lockfile so a reviewer can see `echo-r1`;
- SRI byte verification ignores it, as required for unknown options;
- integrity URLs, content-store keys, byte equality, authorization, VEX, and
  policy decisions use only algorithm plus complete digest;
- an option-only metadata change never creates a new artifact revision;
- authenticated structured revision metadata wins if the hint disagrees.

The hint is deliberately redundant with `dist.revisions[].revision`. It
improves visibility in tools that record only `dist.integrity`, while the
structured entry carries provenance and policy semantics.

This option is not required for protocol correctness. Registries may omit it,
and compatibility must be tested across pnpm, npm, Yarn, lockfile serializers,
and third-party SRI tooling before it is enabled by default. The SRI document
currently specifying options is a Working Draft and explicitly leaves their
semantics undefined.

### The canonical URL remains the original artifact

The canonical npm tarball route is never repointed:

```text
https://registry.example/ejs/-/ejs-2.7.4.tgz
```

It continues to return the original upstream bytes and integrity forever. The
current package document may advertise another artifact through
`dist.tarball`, just as npm package documents already may advertise
non-canonical tarball URLs.

This rule provides universal backward compatibility:

- a lockfile created before patch integration reconstructs the canonical URL
  and still receives its original bytes;
- a lockfile created after patch integration records an immutable integrity
  URL and keeps receiving that patch revision;
- changing the selected revision changes only mutable package metadata;
- CDN objects and package-manager caches never have one URL for two bodies.

### Manifest metadata across revisions

A patch may update dependencies as well as source files. Revisions are
therefore not required to have identical dependency metadata.

pnpr reads the selected artifact's `package.json` only after the tarball passes
integrity verification and projects its resolution-relevant fields into the
current version document. When a revision becomes selected, the entire
projected version entry and `dist` move atomically.

An old lockfile retains its old integrity URL and dependency snapshot. A pnpm
revision refresh must replace both together; changing only the checksum would
be incorrect.

This means two lockfiles can hold different dependency graphs for the same
`name@version` when they pin different integrities. The current pnpm lockfile
cannot represent both revisions simultaneously in one graph. If simultaneous
revisions are required, integrity must become part of the package key in a
separate lockfile format change.

### CDN and installation performance

The normal data path remains the same length as today:

```text
packument resolution → one tarball URL → one CDN request
```

There is no custom request header, `Vary` cache key, canonical redirect,
metadata recovery request, or intentionally failed download. A warm CDN serves
the tarball without contacting pnpr. On a CDN miss, pnpr performs one
content-addressed lookup and streams the object, equivalent to its existing
tarball cache path.

The URL is longer by one sha512 digest, and patched packuments contain a small
revision list. Those costs are negligible compared with a tarball body, and
the list exists only for patched versions. pnpr should precompute projected
packuments and index objects directly by binary digest rather than scanning
revision arrays on the fetch path.

Local installation is unchanged after pnpm already has the integrity in its
content-addressed store. On a cold install, pnpm makes the same number of
tarball requests and performs the same SRI verification as today.

### Selecting and updating revisions

A frozen install never changes revision. An ordinary install may continue to
prefer its lockfile, just as it does for package versions.

pnpm should provide an explicit operation to refresh only artifact revisions:

```text
pnpm update --patches
```

For every locked registry package, pnpm resolves the same exact
`name@version`. If `dist.integrity` changed, it replaces the integrity URL,
integrity, and package snapshot atomically. It does not rerun semver selection
or update package versions.

A normal resolution caused by another lockfile change records the currently
selected revision. Whether non-frozen `pnpm install` automatically adopts a new
revision remains a pnpm policy question; the registry protocol does not require
silent lockfile mutation.

### Caching and atomic provider refresh

pnpr indexes artifact content by:

```text
(concrete registry identity, algorithm, complete digest)
```

Physical bytes may be deduplicated beneath that authorization index. A provider
refresh that selects a new revision follows this order:

1. Download and verify the new tarball.
2. Store it under its immutable integrity URL.
3. Derive and validate projected version metadata from the verified tarball.
4. Append its revision metadata.
5. Atomically update the projected version entry and `dist`.
6. Invalidate projected full and abbreviated packuments.

No tarball cache object is invalidated or overwritten. Metadata never advertises
an integrity URL before that URL is available.

### Authorization, retention, and enforcement

Both current and historical integrity URLs apply the addressed registry's
access policy. History and availability remain separate: an operator may
retain a revision record for audit while a legal or security policy refuses
its bytes. Such a request returns a policy-specific `403`.

This matters for the vulnerable original. A reproducibility policy may retain
it indefinitely; strict enforcement may deliberately block it even though an
old lockfile then cannot install through that registry. Patch selection must
not silently weaken a registry's separate screening policy.

### Auditing a revision

Advisories apply to the original `name@version`; provider VEX statements apply
to one exact integrity. `dist.revisions[].fixes` and attestations preserve that
mapping without changing package identity.

An audit request carrying only `name@version` is evaluated conservatively
against the base version. An integrity-aware pnpm audit extension can identify
the installed revision and subtract only the fixes declared for those exact
bytes. A later provider revision does not retroactively add fixes to an older
locked revision.

## Rationale and Alternatives

### Provider aliases and `_pnprPatch`

The alternative patch-provider RFC exposes a provider package such as
`@echo-patch/ejs` and either asks the workspace to write an alias override or
adds a registry annotation instructing pnpm to do it.

That representation keeps `name@version` immutable and makes provider
provenance conspicuous, but turns a registry policy into a client-level package
substitution. It requires provider-registry configuration, creates separate
provider package documents, and depends on pnpm-specific annotation or override
behavior. Integrity-addressed revisions keep provenance in registry metadata
and work as ordinary `dist` entries in every npm-compatible client.

### Encode the revision in the version

Versions such as `2.7.4-echo.2` are prereleases under semver. Ordinary ranges
containing `2.7.4` do not necessarily select them, peer ranges may reject them,
and the package no longer reports the version the application was tested
against. Build metadata is unsuitable because semver ignores it for precedence
and equality.

### Encode the revision in a package name

Names such as `@echo-patch/ejs-r1` and `@echo-patch/ejs-r2` avoid prerelease
semantics but create one package document per patch and expose a
provider-internal revision scheme throughout consumer tooling.

### Use a named provider registry

A specifier such as `echo:ejs@2.7.4` keeps the original name and version, but
every consumer must configure the `echo` registry locally. Current pnpm
lockfiles also cannot safely represent two named registries serving different
bytes for the same `name@version` in one graph.

Named registries remain coherent when the provider is intentionally the origin
for all packages. They are a poor encoding for a central registry adopting only
selected patch artifacts.

### Mutate the canonical URL and negotiate by request header

Rejected as the normal path. A complete integrity request header can safely
select a historical representation, but CDNs must forward it and either vary
large cached bodies on a high-cardinality field or return an extra redirect.
Clients without the protocol still break on old lockfiles.

Direct integrity URLs use ordinary CDN cache keys, require one request, and let
the canonical URL remain immutable for every old client.

### Include the package name and version in the integrity URL

A format such as
`<canonical-url>+sha512.<digest>` makes the relationship human-readable and
allows routing without a digest-to-package index. It is valid, but duplicates
identical content under several URLs and makes content addressing depend on
package naming.

The digest-only route deduplicates naturally and lets pnpm construct or store
the URL from registry plus integrity alone. Registry-scoped authorization
provides the package-policy boundary that the omitted name otherwise supplied.

### Use only a digest prefix

Rejected. A full sha512 value is small relative to an HTTP request or tarball.
Prefixes introduce accidental ambiguity, deliberate collision work, and an
arbitrary minimum length. The full digest is package metadata, not a credential
or bearer capability.

### Put the revision only in an SRI option

An SRI option is useful visibility for lockfiles and generic metadata
consumers, but it is intentionally ignored by integrity verification and has no
standardized patch semantics. It cannot replace the structured revision
record, signed provider manifest, or digest URL. This RFC uses it only as a
redundant hint.

## Implementation

### pnpr

1. Add provider manifest configuration, signature verification, pinned refresh,
   and content-addressed ingestion.
2. Validate every artifact's integrity and original `name@version`, then derive
   the projected version metadata from its verified `package.json`.
3. Add a registry-scoped digest index that separates physical deduplication
   from authorization.
4. Serve immutable `/-/tarballs/<algorithm>/<digest>` routes with range request,
   CDN, and access-policy support.
5. Project selected integrity URLs and append-only `dist.revisions` into full
   and abbreviated packuments.
6. Parse and emit the optional bounded `pnpr-patch` SRI hint while excluding
   all options from object lookup and security policy.
7. Keep original canonical tarball routes immutable.
8. Make provider refresh atomic and audit every selected-revision transition.
9. Extend audit handling to understand integrity-scoped fixes and attestations.

Tests should cover:

- original, first patch, and second patch all retrievable by full integrity;
- `dist.tarball` selecting the latest revision without changing name/version;
- original canonical URLs continuing to return original bytes after every
  provider refresh;
- full and abbreviated metadata exposing the required revision history;
- malformed, abbreviated, unsupported, unknown, and non-canonical digests;
- SRI revision options round-tripping while leaving the object key unchanged;
- changed or forged SRI options remaining policy-inert;
- the same physical digest referenced by two authorized registries without
  cross-registry data disclosure;
- a known digest requested through an unauthorized registry returning no
  bytes;
- immutable CDN caching, range requests, and a warm CDN avoiding pnpr;
- atomic refresh with no metadata-visible-but-unavailable interval;
- selected metadata being derived from the verified tarball;
- a dependency-changing revision updating snapshot and integrity together;
- provider conflict rejection;
- retention and screening policy on historical objects;
- audit/VEX evaluation tied to exact integrity.

### pnpm

The companion RFC,
[`text/0000-integrity-addressed-registry-tarballs.md`](../../text/0000-integrity-addressed-registry-tarballs.md),
contains the client and lockfile changes.

## Prior Art

- The [OCI Distribution Specification](https://github.com/opencontainers/distribution-spec/blob/main/spec.md)
  permits retrieval by a mutable tag or immutable digest. This RFC applies the
  same current-pointer/content-identity split to npm artifacts.
- npm lockfiles already use
  [Subresource Integrity](https://www.w3.org/TR/SRI/) to authenticate tarballs.
  This proposal uses the existing full value as the content key.
- Content-addressed package stores in pnpm and systems such as Nix demonstrate
  that logical package identity and physical content identity can be separate.
- [RFC 9530](https://www.rfc-editor.org/rfc/rfc9530.html) provides optional
  `Content-Digest` response fields complementary to npm SRI verification.

## Unresolved Questions and Bikeshedding

- Is `/-/tarballs/<algorithm>/<digest>` the right registry-relative endpoint,
  or should it be namespaced explicitly under `/-/pnpr/`?
- Is `dist.revisions` the right package-document field name, or should it be
  namespaced?
- Which projected manifest fields should historical revision entries retain
  for inspection?
- Does `pnpm update --patches` update all artifact revisions, only
  security-motivated revisions, or accept provider and advisory filters?
- How long must a pnpr deployment retain historical content, and how should it
  communicate a policy shorter than lockfile lifetime?
- Should historical vulnerable artifacts remain retrievable by default, or
  require an explicit reproducibility policy?
- Should `pnpr-patch=echo-r2` be enabled by default while SRI option semantics
  remain undefined, and is that the right namespaced syntax?
