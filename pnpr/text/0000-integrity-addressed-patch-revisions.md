# Integrity-addressed patch revisions for pnpr

> Status: proposed amendment to the experimental revision protocol;
> implementation pending.

## Summary

pnpr should be able to project a provider's patched build directly over the
original package's `name@version`, without exposing a provider alias or
inventing a version suffix. Every accepted artifact is stored and served from
an immutable, registry-scoped URL identified by its package name, version, and
revision number. For every revision, including the original, `dist.integrity`
is the hash of its uncompressed tar, independent of the registry's default
format and the client's lockfile setting:

```text
https://registry.example/<name>@<version>+r<revision>
```

The selected version's `dist.tarball` points directly at that URL.
`dist.integrity` contains the corresponding standard SRI value, and a selected
artifact advertises its registry revision number:

```jsonc
"dist": {
  "tarball": "https://registry.example/ejs@2.7.4+r2",
  "integrity": "sha512-<second-patch-digest>",
  "revision": 2
}
```

The original is explicitly marked `revision: 0` and uses the same
`<name>@<version>+r<revision>` route as every replacement. The package document
advertises an append-only history of the original artifact and every accepted
replacement for that `name@version`. The protocol is experimental and has no
backward compatibility or migration requirement for earlier experimental state
or lockfiles.

The revision has three client-facing purposes:

- it tells pnpm that the artifact's tar must be fetched from the
  `<name>@<version>+r<revision>` URL, constructed from the effective registry,
  package name, version, and revision number;
- it makes an integrity change understandable in a lockfile diff;
- it gives workspace overrides a stable revision number to pin (the companion
  RFC's `+rN` build-metadata specs).

The revision is not part of byte verification. The complete algorithm and
digest remain the artifact identity.

Fresh resolutions receive the selected artifact directly from the immutable
URL advertised in `dist.tarball`. A client must support HTTP content decoding
before SRI verification and extraction of a plain tar; npm and Yarn support
requires client-matrix tests.

This makes the identity split explicit:

> `name@version` selects the registry's current approved artifact.
> The complete integrity identifies immutable bytes.
> `dist.revision` exposes the registry revision and retrieval convention.

The companion pnpm RFC specifies how a lockfile `revision` field replaces
tarball URLs in the pnpm lockfile and how pnpm refreshes artifact revisions
without changing package versions.

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
ejs@2.7.4  revision 0  sha512-<upstream>  original artifact
ejs@2.7.4  revision 1  sha512-<echo-r1>   first replacement
ejs@2.7.4  revision 2  sha512-<echo-r2>   current replacement
```

Every accepted artifact must remain reproducible. Encoding these revisions as
names creates one package document per patch. Encoding them as versions
introduces prerelease semantics and changes the version reported by the
package.

Integrity already authenticates npm tarballs and distinguishes their bytes. A
small neutral revision number can explain the sequence without becoming
artifact identity or exposing the provider's own naming system.

### One convention for every revision

Every revision-aware artifact uses the `<name>@<version>+r<revision>` route.
Revision zero is explicitly marked so clients need no exception for original
checksums or canonical URLs. Restoring the original changes selection to
`revision: 0` while preserving retrieval and verification semantics.

Versions outside this protocol retain ordinary registry metadata and fetch
behavior. Introducing revisions for a version does not reinterpret any existing
ordinary lockfile resolution.

### Immutable URLs preserve lockfiles and make CDNs fast

Serving new bytes from an old canonical URL breaks lockfiles and makes shared
caches ambiguous. Sending an integrity selector in a request header could
recover old content, but it requires special CDN forwarding and either a
varying cache key or an extra redirect.

A revision suffix in the URL needs neither. Every representation has one
ordinary, immutable cache key. Package managers resolving current metadata
follow `dist.tarball`, while revision-aware pnpm lockfiles construct the same
route directly from the registry, package name, version, and revision number.

## Detailed Explanation

### Terminology

- An **original identity** is an npm package `name@version`, for example
  `ejs@2.7.4`.
- An **artifact** is one package resource for that identity, identified by its
  complete sha512 digest over the uncompressed tar, including for originals.
- A **registry revision** is an accepted artifact's entry in the identity's
  append-only history. Its position is its stable **revision number**: the
  original is revision zero, the selected artifact's number is always
  advertised in `dist.revision`, and the inline spec syntax writes it
  as `+rN`.
- A **provider revision** is an optional provider-specific identifier such as
  `echo-r2`. It is opaque provenance metadata — not a number, not ordered,
  never parsed — and is not exposed in `dist.revision`.
- The **selected revision** is the artifact advertised by the current version
  document.
- The **canonical URL** serves the original resource: the npm tarball URL on
  a `.tgz` registry, or `<registry>/<name>@<version>` on a tar registry.
- The **revision URL** is `<registry>/<name>@<version>+r<revision>`.

### Invariants

1. **Full sha512 is artifact identity.** A truncated digest or revision number
   is never a byte-verification input or security decision.
2. **Every tarball URL is immutable.** The canonical URL remains on the
   original artifact, and a revision URL always returns the resource bound to
   that revision in the identity's append-only history, verified against its
   complete sha512 digest after HTTP content decoding. Re-encoding a revision
   tar changes neither its identity nor its revision number.
3. **Every revision is explicitly marked.** Revision-aware metadata advertises
   `dist.revision`, including zero, and a `<name>@<version>+r<revision>`
   `dist.tarball`. Absence of the marker means ordinary metadata outside this
   protocol.
4. **Revision numbers are stable and uniquely allocated.** A distinct accepted
   replacement receives the next positive integer. Numbers are never
   reassigned. Selecting a previously accepted artifact again uses its existing
   revision number. Allocation serializes on `(registry, name, version)` and
   holds two uniqueness constraints within that identity: each revision number
   is assigned at most once, and each digest maps to exactly one revision
   number. The registry owns the numbering: provider renames, replacements, or
   removals never renumber existing revisions, so a recorded `rN` can never
   silently change meaning.
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
  "sequence": 42,
  "entries": {
    "ejs@2.7.4": {
      "revision": "echo-r2",
      "tarball": "https://npm.echo.example/artifacts/ejs/2.7.4/r2.tgz",
      "integrity": "sha512-<echo-r2-tgz>",
      "fixes": ["GHSA-...", "CVE-2026-..."],
      "attestations": ["https://npm.echo.example/attestations/..."]
    }
  }
}
```

pnpr verifies the manifest signature, downloads the `.tgz`, and verifies its
complete integrity against the manifest before decoding it. It then derives
the tar hash and stores the artifact and the verified source-to-tar binding
before making the replacement visible. The manifest's `integrity` pins the
provider's `.tgz`; the projected `dist.integrity` pins its tar. Those two
values are not interchangeable. The fetch itself is
constrained: https only, artifact origins declared in provider configuration
the way the manifest origin is, private and link-local addresses rejected,
no redirects, provider credentials sent only to declared origins, and
download size bounded before and after decompression.

The signed document carries a monotonic `sequence`. pnpr persists, per
provider, the highest accepted sequence together with the canonical digest of
the manifest that carried it, and applies one rule set:

- a higher sequence is processed and advances the checkpoint;
- an exact redelivery — same sequence, same manifest digest — is an
  idempotent no-op, so a retried or duplicated delivery of an already
  accepted manifest never reads as an attack;
- the same sequence with a different manifest digest is rejected and flagged
  as provider equivocation: two contradictory documents signed at one
  sequence indicate a compromised or misbehaving provider, not a duplicate;
- a lower sequence is rejected: a valid signature over an older document must
  not be able to replay earlier selections and reselect a vulnerable
  revision.

Moving selection backward — withdrawing a bad patch, restoring the original —
therefore always requires either a new, higher-sequence manifest from the
provider or an explicit, separately authorized operator operation, never a
replayed document. The checkpoint advances atomically with — never before —
the durable acceptance of the manifest's work: a crash mid-ingestion leaves
the previous checkpoint in place, so the same sequence can be retried rather
than becoming permanently unacceptable.

The verified tarball must contain the original `name` and `version`. A provider
may use aliases or revision identifiers in its distribution system, but the
artifact offered for projection must identify itself as `ejs@2.7.4`. pnpr does
not rewrite `package.json` after verification because that would produce bytes
different from the provider's signed artifact. Provider attestations and
signatures retain that source `.tgz` as their subject; they are not relabeled
as direct signatures over the derived tar. Audit/VEX translation from a
source digest to a tar digest requires the verified binding. A provider
recompressing an identical tar supplies new source evidence for the same
artifact, not a new revision. Source evidence is retained per signed input
so a later upload cannot replace the evidence attached to an accepted input.
Additional source evidence is append-only and separate from the immutable
revision record; re-compression never rewrites that record's fixes or
attestations.

Every projected revision's `dist`, including zero, omits `shasum` and
signatures over a source `.tgz`, even on a `.tgz` registry: neither authenticates
the tar resource. Attestation links and provenance markers may remain, with
their source subjects and verified bindings intact. Ordinary metadata outside
the revision protocol keeps the fields appropriate to its own resource.

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

### Revision tar resources

Every revision, including zero, uses the tar defined by
[the tar registry RFC](../../text/0000-zstd-registry-tarballs.md#the-tar-and-its-integrity).
The whole source `.tgz` must decode successfully under that RFC's byte limits
before a tar hash or revision is published. Concatenated gzip members are
decoded together; a `.tgz` with invalid trailing data has no tar and is rejected.
The client's legacy `.tgz` extraction exception for gzip trailer errors after
the tar end marker does not apply to deriving a tar resource. No re-packing,
metadata rewriting, or tar normalization is allowed.

The registry stores each verified source `.tgz` and its source digest as
evidence, including for revision zero. The tar hash keys revision history,
digest URLs, projected metadata, and client lockfiles. Uniqueness and concurrent
allocation compare tar hashes, so differently compressed copies of one tar
share a revision. Provider input matching the original tar remains revision
zero. Distinct source attestations remain bound to their own signed inputs.

The verified original tar and its source-to-tar binding must be materialized
before a version becomes revision-aware. Ordinary upstream metadata remains
outside the protocol until that succeeds. Publishing revision zero and its
history is atomic with activating revision-aware metadata, including when the
first selected artifact is a replacement.

### Revision URL

The revision route is relative to the addressed npm registry:

```text
<registry-base>/<name>@<version>+r<revision>
```

Examples:

```text
https://registry.example/ejs@2.7.4+r2
https://pnpr.example/~main/@scope/pkg@1.0.0+r0
```

The revision number uses canonical decimal digits without leading zeros (except
`0` itself), matching the `+rN` build-metadata spec syntax; servers reject
non-canonical variants such as `+r01`. Package metadata uses standard SRI
base64 with no extensions, and the selected artifact's revision number in
`dist.revision` matches the `+r<revision>` suffix in `dist.tarball`.

Because the package name and version are part of the URL, the route routes and
authorizes directly by `(registry, name, version)` without requiring a separate
digest-to-package reverse index, while keeping the endpoint independent of
provider naming.

The registry base remains load-bearing for security. A request to
`/~main/<name>@<version>+r<revision>` is authorized in `main` against that
package's access rules. pnpr may deduplicate physical storage globally, but it
must not expose an object merely because another organization or registry
stored the same digest.

Successful public responses are ordinary immutable CDN objects:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
Content-Type: application/x-tar
Content-Encoding: gzip
Vary: Accept-Encoding
```

This example is a revision resource served with gzip content coding. Its HTTP
resource is the tar; the stored source `.tgz` can be sent unchanged as its
gzip encoding. Clients accepting zstd receive the level-19 cached variant,
or a level-3 stream until it exists; other clients receive the identity tar.
Use the tar registry RFC's decoder limits, bounded variant worker, caching
policy, and response-header rules. No `ETag` derived only from the tar digest
is emitted for differently encoded responses. `Content-Digest`, if emitted,
hashes the encoded message content and therefore differs by encoding; the
URL and `dist.integrity` always name the decoded resource. Partial-content
responses for revisions are outside this amendment; their range requests
receive the full body.

Private registries use cache directives compatible with their authorization
model. The immutable object must have been verified before publication.
A response that still checks a cached source must fail on a mismatch before
completing, omit `Content-Length`, and evict the stale source. No derived
hash or variant may be published from mismatched bytes.

### Intermediaries and the revision route

The `<name>@<version>+r<revision>` URL tells any layer between client and
registry — a corporate proxy, a mirror, a registry firewall — which package,
version, and revision a request is for, without exposing a provider name:

- **The effective registry is the base the client addresses.** When a proxy
  fronts pnpr, the proxy's base URL is the registry from the client's point of
  view: revision numbers are scoped to that base and credentials are scoped
  same-origin to it, exactly as for any registry base. A proxy projecting
  several upstream registries must present them as distinct bases (for example,
  distinct path prefixes), because one base implies one revision numbering and
  one credential scope.
- **`dist.revisions` binds each revision URL to its tar digest.** A proxy or
  client verifies the downloaded tar against the `dist.integrity` recorded for
  that revision in the package's metadata or lockfile.

### Advertising the selected revision

Every revision-aware version carries the selected revision number as a plain
integer field beside an ordinary SRI integrity:

```jsonc
"dist": {
  "tarball": "https://registry.example/ejs@2.7.4+r2",
  "integrity": "sha512-<digest-base64>",
  "revision": 2
}
```

`dist.integrity` remains a standard Subresource Integrity value with no
extensions, so every strict SRI consumer parses it. The numbering matches the
structured `dist.revisions` record, which counts the original as revision
`0`.

pnpr applies these rules:

- it emits `dist.revision` for every revision-aware version, including
  `0` when the original is selected;
- it assigns `1` to the first distinct accepted replacement and monotonically
  increasing values to later replacements, within the canonical integer range
  the companion RFC fixes (`0 ≤ N ≤ 2^53 − 1`), serialized as canonical
  decimal integers;
- it never uses the provider name or provider revision in the field;
- it preserves revision numbers in `dist.revisions`, audit output, and logs;
- it excludes the field from object lookup, equality, authorization, VEX, and
  integrity verification;
- it validates the field under one rule, identical for full and abbreviated
  packuments: an invalid parsed value (negative, fractional, a string, or
  outside the safe-integer range) or a revision number that does not match the
  `dist.revisions` entry whose integrity equals `dist.integrity` is a protocol
  error — pnpr never emits it, and a client rejects the version's metadata
  instead of reconciling the mismatch from either side;
- a field-only change cannot create a new stored artifact: the integrity and
  structured history remain authoritative for bytes and provenance, so a forged
  revision number can misdescribe a revision but never change what is fetched
  or verified.

Tar-aware clients ignore the unknown `dist.revision` field and follow
`dist.tarball`; HTTP content decoding must precede SRI verification. pnpm records the revision
number in its lockfile as the durable signal to construct the digest URL, as the companion RFC specifies.

### Projected package document

For a covered version, pnpr projects the selected artifact under the original
name and version. `dist.tarball` points directly to the immutable
`<name>@<version>+r<revision>` URL. `dist.revisions` advertises the original
artifact and every accepted provider replacement:

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
        "tarball": "https://registry.example/ejs@2.7.4+r2",
        "integrity": "sha512-<echo-r2-base64>",
        "revision": 2,
        "revisions": [
          {
            "revision": 0,
            "integrity": "sha512-<upstream>",
            "tarball": "https://registry.example/ejs@2.7.4+r0",
            "source": "upstream",
            "publishedAt": "2020-04-03T00:00:00.000Z",
            "manifest": { "dependencies": {} }
          },
          {
            "revision": 1,
            "integrity": "sha512-<echo-r1>",
            "tarball": "https://registry.example/ejs@2.7.4+r1",
            "source": "echo",
            "providerRevision": "echo-r1",
            "publishedAt": "2026-05-10T00:00:00.000Z",
            "fixes": ["GHSA-..."],
            "attestations": ["https://..."],
            "manifest": { "dependencies": {} }
          },
          {
            "revision": 2,
            "integrity": "sha512-<echo-r2>",
            "tarball": "https://registry.example/ejs@2.7.4+r2",
            "source": "echo",
            "providerRevision": "echo-r2",
            "publishedAt": "2026-07-28T00:00:00.000Z",
            "fixes": ["GHSA-...", "CVE-2026-..."],
            "attestations": ["https://..."],
            // the patch replaced a vulnerable transitive dependency
            "manifest": { "dependencies": { "safe-parse": "^1.2.0" } }
          }
        ]
      }
    }
  }
}
```

The selected revision is the entry whose complete digest equals
`dist.integrity` and whose revision number equals the required
`dist.revision`. Every history entry carries a tar digest, including zero.
Tar-aware clients ignore `dist.revisions` and fetch the selected `dist.tarball`.

The history provides:

- discoverability through `npm view` and registry APIs;
- provenance and advisory/VEX metadata for exact bytes;
- an auditable sequence when a provider reissues a patch;
- the allow-list used when a verifier checks a historical digest;
- per-entry status: optional `withdrawn` and `supersededBy` fields record
  whether a revision was retired and which revision replaced it. Revision
  numbers only order the history; two live revisions with different `fixes`
  sets may legally sit alongside each other, so audit needs the status fields
  to tell "r2 replaces r1" from "r2 coexists with r1".

Each revision entry carries a `manifest` object holding the resolution-relevant
fields of its artifact — the same per-version subset the abbreviated packument
serves: `dependencies`, `optionalDependencies`, `peerDependencies`,
`peerDependenciesMeta`, `bundledDependencies`, `bin`, `engines`, `os`, `cpu`,
`libc`, and `hasInstallScript`. Revisions may legally differ in any of them,
and the companion RFC lets a client resolve a non-selected revision — an
override pinning `+rN` — without first downloading its tarball. These fields
are always derived from the verified tarball's `package.json`, exactly as the
selected projection is; a provider manifest cannot declare metadata that
diverges from its artifact's bytes. Registry-managed mutable metadata is
deliberately excluded from the subset: `deprecated` can change after
publication through `npm deprecate`, so it lives on the projected version entry
as ordinary mutable registry metadata, never inside an immutable revision
record. The only registry-managed annotations a history entry itself may carry
are the status fields `withdrawn` and `supersededBy`: they describe the
record's standing, never its artifact, and updating them touches no
artifact-derived field, byte, or revision number — append-only means records
are never removed or renumbered, not that their status can never be annotated.
The full packument may additionally carry all provenance fields. Abbreviated
metadata should keep current `dist` plus the `revision` number, integrity,
digest URL, and `manifest` of each historical revision — `+rN` resolution maps
a revision number to its history record using the metadata pnpm actually
fetches, so the revision number must survive abbreviation. A revision-aware
version includes both fields even when zero is its only revision. A version
outside the protocol omits both fields and uses ordinary metadata.

### The canonical URL remains the original artifact

Ordinary name-and-version routes keep serving the original resource in the
registry's configured format. They do not participate in revision selection.
Every revision-aware metadata record and lockfile entry uses the tar digest
route, including revision zero. A registry's `.tgz` canonical route and revision
zero's digest route can therefore expose different resource formats derived
from the same verified original source.

Clients must follow the advertised digest URL, remove HTTP content coding, and
verify the tar integrity. A client that reconstructs a canonical `.tgz` URL
cannot install such metadata. A deployment must validate its supported client
matrix before enabling revisions. Revision management remains pnpm-specific
(host-free lockfile entries, `+rN` pinning, revision refresh).

No route silently substitutes a selected replacement for the original, and an
integrity mismatch is never protocol negotiation.

### Manifest metadata across revisions

A patch may update dependencies as well as source files. Revisions are not
required to have identical dependency metadata.

pnpr reads the selected artifact's `package.json` only after the tarball passes
integrity verification and projects its resolution-relevant fields into the
current version document. When a revision becomes selected, the entire
projected version entry and `dist` move atomically, carrying forward
registry-managed mutable fields such as `deprecated`.

Before activating revision-aware metadata, pnpr MUST materialize revision zero
from the verified original artifact, including its tar integrity and
resolution-relevant manifest. This applies whether the first selected revision
is zero or a replacement.
Every later selection rebuilds the version document's artifact-derived fields
from the selected revision's manifest. Selecting revision zero restores the
original artifact-derived fields and sets `dist.revision: 0`, while retaining
the complete revision history and tar digest URL. A change of upstream source
MUST NOT redefine revision zero; an artifact with a different digest is either
accepted as a new revision or rejected by registry policy.

The selected revision therefore needs no per-field override: the projected
version entry *is* that revision's metadata. Non-selected revisions expose
theirs through `dist.revisions[].manifest`, so a client pinning an older or
newer revision resolves with the dependency set, peer requirements, and
install-script posture that artifact actually has.

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

There is no selector header, redirect, metadata recovery request, or
intentionally failed download. Standard `Vary: Accept-Encoding` separates
transport encodings of one tar resource. A warm CDN serves the object without
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
`name@version`. If the selected artifact changed, it replaces the integrity,
the recorded revision, and the package snapshot atomically. It does not rerun
semver selection or change package versions.

The companion RFC also gives a workspace explicit revision selection: a spec of
the form `<version>+rN` — as an override target or a declared dependency — pins
the dependency to that exact version and registry revision `N`. `+r0` keeps the
original, a positive revision number adopts or freezes a specific replacement.
`pnpm update --patches` skips pinned packages, and a revision whose bytes this
registry's policy refuses fails the install explicitly instead of falling
forward to another artifact.

Whether non-frozen `pnpm install` automatically adopts a new revision remains a
pnpm policy question.

### Caching and atomic provider refresh

pnpr indexes artifact content by:

```text
(concrete registry identity, algorithm, complete digest)
```

Physical bytes may be deduplicated beneath that authorization index. A provider
refresh that accepts a distinct artifact follows this order:

1. Download the provider `.tgz` and verify its signed source digest.
2. Decode it in full, derive the tar hash, and store the source-to-tar binding
   and the artifact before exposing its tar-addressed URL.
3. Derive and validate projected version metadata from the verified tar.
4. Assign the next stable registry revision.
5. Append its structured revision metadata.
6. Atomically update the projected version entry and `dist`.
7. Invalidate projected full and abbreviated packuments.

Steps 4 through 6 execute as one transaction that serializes on `(registry,
name, version)`, with uniqueness enforced separately on the revision number and
on the digest mapping within that identity. A retry after a partial failure
observes the existing digest mapping instead of allocating again, and two
concurrent refreshes carrying different digests cannot both receive the same
next revision number.

No immutable tar resource is overwritten. Encoded variants are disposable
cache entries and may be evicted or regenerated without changing history,
revision numbers, or client pins. Metadata never advertises an integrity URL
before its verified resource is available.

### Authorization, retention, and enforcement

Both current and historical integrity URLs apply the addressed registry's
access policy. History and availability remain separate: an operator may
retain a revision record for audit while a legal or security policy refuses its
bytes. Such a request returns a policy-specific `403`.

This matters for a vulnerable original. A reproducibility policy may retain it
indefinitely; strict enforcement may deliberately block it even though an old
lockfile then cannot install through that registry. Patch selection must not
silently weaken a registry's separate screening policy.

Immutability is a statement about bytes, not availability: a successful
response for a digest URL always decodes to the same resource bytes, but
policy may stop the URL from answering successfully. A transition to refusal
is effective only once the edge stops serving, so deployments that publish
long public cache lifetimes must purge affected objects on such transitions, and a
registry whose policy may require immediate refusal should prefer
authorization-aware (non-shared) caching for the affected class of artifacts
over year-long shared caching. Policy applies to the artifact, never the
route: the canonical URL and the digest URL must give the same answer for the
same artifact. Source and tar addresses are joined by their verified binding
for this check, so neither route nor resource format can bypass policy.

### Auditing a revision

Advisories apply to the original `name@version`; provider VEX statements apply
to one exact integrity. `dist.revisions[].fixes` and attestations preserve that
mapping without changing package identity.

An audit request carrying only `name@version` is evaluated conservatively
against the base version. An integrity-aware pnpm audit extension can identify
the installed revision and subtract only fixes declared for those exact bytes.
A later provider revision does not retroactively add fixes to an older locked
revision. The per-entry status fields make the history's shape explicit to
audit: `supersededBy` marks a retired revision, while live revisions with
disjoint `fixes` sets legitimately coexist — the revision number alone never
encodes which of the two situations holds.

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
ordinary `dist` entries to clients that support the tar resource contract.

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

### Digest-only URLs (`/-/tarballs/sha512/<digest>`)

Earlier drafts addressed revisions by digest alone (`/-/tarballs/sha512/<base64url-digest>`),
omitting the package name and version from the URL. While a digest-only URL
deduplicates identical tarballs across package names at the URL layer, it
requires a reverse digest-to-package index on hosted registries (`O(N)`
storage entries under `.revisions/sha512/<digest>/index.json`) to enforce
per-package access and OSV policy, and obscures package identity in proxy logs.

Because `<registry>/<name>@<version>` is already the canonical tarball URL on
pnpr, appending `+r<revision>` (`<registry>/<name>@<version>+r<revision>`)
matches the `+rN` override and dependency selector syntax, routes directly by
package and version without a reverse index, and remains provider-neutral and
immutable per `(registry, name, version, revision)`. Content-hash verification
still rests on `dist.integrity` and the lockfile's `resolution.integrity`.

### Use only a digest prefix

Rejected. A full sha512 value is small relative to an HTTP request or tarball.
Prefixes introduce ambiguity, collision work, and an arbitrary minimum length.
The complete digest is package metadata, not a credential.

### Mark revisions inside the SRI integrity value

Earlier drafts carried the revision number as an SRI option —
`sha512-<digest>?r2`, with `?r0` (or a bare `?`) marking originals — so the
marker traveled inside the integrity string itself. Rejected in favor of the
separate `dist.revision` field:

- SRI reserves `?options` for extensions, but real npm-ecosystem SRI parsers
  do not implement the grammar. pnpm's Rust `ssri` splits on `-` and takes the
  remainder as the digest, so an option parses *successfully* into a corrupted
  digest and round-trips intact, failing far from the parse. Making option
  preservation load-bearing protocol state is a class of fragility a plain
  JSON field does not have;
- integrity-string equality is used as byte equality across the ecosystem, so a
  revision number inside the value makes a revision-only difference read as a
  content change. The companion pnpm RFC records where this bites a client;
- a plain integer field is ordinary JSON everywhere, and the revision number
  appears on its own line in a lockfile diff instead of at the end of a
  ninety-character integrity string.

The `r` letter survives only in the inline `+rN` spec syntax, where build
metadata needs a compact marker: `v` reads as "version" — the association
this design exists to avoid; `b` means a source-identical rebuild in Debian
(`+b1`) and collides with hexadecimal digest fragments; Alpine and Gentoo's
`-rN` downstream revision numbering is the exact precedent.

### Put provider identity in the revision field

A value such as `revision: "echo-r2"` leaks the provider's naming scheme into
resolution metadata and makes a provider rename look like an artifact change.

Neutral integers are sufficient for lockfile visibility. Structured
`dist.revisions` and signed provider manifests remain authoritative for
provenance, fixes, attestations, and policy.

### Omit structured revision history

A bare revision number cannot authenticate provider identity, fixes, or
attestations. It also cannot tell a verifier whether a historical digest was
ever associated with a particular `name@version`.

The compact `dist.revision` field complements but does not replace
`dist.revisions`.

## Implementation

### pnpr

1. Add provider manifest configuration, signature verification, pinned refresh,
   and content-addressed ingestion.
2. Verify the provider's `.tgz` digest and original `name@version`, decode it
   in full under the tar registry RFC's limits, and retain the verified
   source-to-tar binding. Key all revisions and revision allocation by tar
   digest, including revision zero, then derive projected metadata from the
   verified `package.json`.
3. Serve every revision's `<name>@<version>+r<revision>` route as a tar
   resource with gzip, zstd, or identity content coding, standard
   encoding-aware CDN caching, and the existing access policy.
4. Materialize verified original tar resources before activating revision-aware
   metadata, including versions with no replacement yet.
5. Emit `dist.revision` with stable nonnegative integer revision numbers,
   including zero for the original.
6. Project selected `<name>@<version>+r<revision>` URLs and append-only
   `dist.revisions` into full and abbreviated packuments.
7. Keep original canonical tarball routes immutable.
8. Make provider refresh atomic and audit every selected-revision transition.
9. Extend audit handling to understand integrity-scoped fixes and
   attestations.

Tests should cover:

- revision-aware originals explicitly marked zero with tar integrity and a
  `<name>@<version>+r0` URL, including before any replacement exists;
- ordinary versions remaining outside the protocol until their original tar
  and history are materialized atomically;
- original, first replacement, and second replacement retrievable by revision
  URL;
- `1` assigned to the first replacement and stable monotonic revision numbers;
- malformed `dist.revision` values rejected from protocol metadata;
- `dist.tarball` selecting the latest revision without changing name/version;
- canonical URLs continuing to return original bytes after every refresh;
- full and abbreviated metadata exposing required revision history, each
  entry's `revision` number included in the abbreviated form;
- revision entries carrying `manifest` fields equal to their verified
  artifacts' `package.json` (declared provider metadata cannot diverge);
- the `dist.revision` field excluded from digest lookup, byte verification,
  and policy;
- a forged `dist.revision` unable to change accepted bytes;
- provider revisions represented only in structured metadata;
- the same physical digest referenced by two authorized registries without
  cross-registry disclosure;
- a known digest requested through an unauthorized registry returning no
  bytes;
- provider `.tgz` integrity checked before the tar hash is derived;
- source signatures, attestations, and VEX bound to the verified source-to-tar
  mapping rather than asserted to sign the tar directly;
- every projected revision, including zero, omitting source `shasum` and
  signatures on either registry format;
- two gzip encodings of an identical tar allocating only one revision, and
  input matching the original tar remaining revision zero;
- all revision encodings, including zero, yielding the same tar hash and
  digest URL;
- encoded variant regeneration preserving pins and revision numbers;
- invalid gzip members, gzip trailers, or decoded trailing tar data rejected
  before publication, including errors after the tar end marker;
- stale-source failures over real HTTP for gzip, zstd, and identity, including
  a gzip response whose declared length would otherwise hide a late error;
- immutable CDN caching, `Vary: Accept-Encoding`, ignored range requests,
  and warm-CDN behavior;
- atomic refresh with no metadata-visible-but-unavailable interval;
- selected metadata derived from the verified tarball;
- a dependency-changing revision updating snapshot and integrity together;
- provider conflict rejection;
- a replayed lower-sequence manifest rejected despite a valid signature;
- an exact redelivery of an accepted `(sequence, manifest digest)` pair a
  no-op, and the same sequence with different content rejected as provider
  equivocation;
- a crash between ingestion and checkpoint advance leaving the same sequence
  retryable;
- concurrent refreshes and retries allocating exactly one revision number per
  digest, and never the same number to two digests;
- revision numbers surviving provider rename, replacement, and removal
  unrenumbered;
- `withdrawn`/`supersededBy` status annotated without removing records or
  renumbering revisions;
- digest requests evaluated against the principal's package-level access, not
  mere registry membership;
- policy refusal answered identically on canonical and digest routes, with
  edge purge or non-shared caching on the transition;
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

- Is `<name>@<version>+r<revision>` the right revision endpoint?
- Is `dist.revisions` the right package-document field name?
- Beyond the resolution-relevant fields required for revision selection, which
  projected manifest fields should historical entries retain?
- Does `pnpm update --patches` update every artifact revision, only
  security-motivated revisions, or accept provider and advisory filters?
- Which clients both honor a non-conventional `dist.tarball` and remove HTTP
  content coding before verifying SRI and extracting a plain tar (npm, Yarn
  classic, Yarn Berry, Bun)? This needs client-matrix tests, including frozen
  installs, redirects, and missing or unsupported content codings.
- Should pnpr expose a batch revision endpoint — POST the locked
  `(name@version, revision)` pairs, receive each entry's currently selected
  revision, integrity, and fixes — so `pnpm update --patches` and
  integrity-aware audit cost one request per lockfile instead of one
  packument read per package? Advisory only: the lockfile still decides what
  is fetched.
- How long must a pnpr deployment retain historical content, and how should it
  communicate a policy shorter than lockfile lifetime?
- Should vulnerable historical artifacts remain retrievable by default or
  require an explicit reproducibility policy?
