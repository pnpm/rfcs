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
`dist.integrity` contains the corresponding standard SRI value, and a
selected replacement additionally advertises its registry revision ordinal:

```jsonc
"dist": {
  "tarball": "https://registry.example/-/tarballs/sha512/<digest>",
  "integrity": "sha512-<second-patch-digest>",
  "revision": 2
}
```

The original is revision zero and is never marked: the canonical npm tarball
URL is pinned to it forever, so unmarked metadata — and the unmarked lockfile
entries derived from it — stay correct on every registry. The package
document also advertises an append-only history of the original artifact and
every accepted replacement for that `name@version`.

The revision has three client-facing purposes:

- it tells pnpm that the replacement's bytes must be fetched from the
  integrity-addressed URL, constructed from the effective registry and
  complete digest;
- it makes an integrity change understandable in a lockfile diff;
- it gives workspace overrides a stable ordinal to pin (the companion RFC's
  `+rN` build-metadata specs).

The revision is not part of the content address or byte verification. The
complete algorithm and digest remain the only artifact identity.

The ordinary canonical npm URL
`<registry>/<name>/-/<name>-<version>.tgz` never changes: it continues to serve
the original upstream artifact. Consequently, legacy lockfiles remain
reproducible. Fresh resolutions in pnpm, npm, and Yarn receive the selected
artifact directly from the immutable URL advertised in `dist.tarball`.

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
small neutral revision ordinal can explain the sequence without becoming
artifact identity or exposing the provider's own naming system.

### Originals need no marker

A lockfile is often created before a package needs remediation. That lockfile
does not need to know anything about this protocol, because the canonical
name-and-version URL is pinned to revision zero forever: an unmarked entry
keeps requesting the canonical URL and keeps receiving the original bytes,
even after the registry later selects a replacement.

Only replacements are marked. When `r1` is selected, freshly resolved
lockfiles record the replacement's integrity plus `revision: 1` and fetch
from the digest route; every existing lockfile continues requesting the
original through the canonical route, unchanged. The common case — no
adopted replacements — therefore costs nothing: no marker, no new lockfile
format, no binding to revision-aware registries.

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
  artifact for that identity. The original is revision zero and is never
  marked. A selected replacement's ordinal is advertised in `dist.revision`;
  the inline spec syntax writes it as `+rN`.
- A **provider revision** is an optional provider-specific identifier such as
  `echo-r2`. It is opaque provenance metadata — not a number, not ordered,
  never parsed — and is not exposed in `dist.revision`.
- The **selected revision** is the artifact advertised by the current version
  document.
- The **canonical URL** is the ordinary npm tarball URL derived from registry,
  name, and version.
- The **integrity URL** is a registry-scoped content address containing the
  complete digest.

### Invariants

1. **Full sha512 is artifact identity.** A truncated digest or revision
   ordinal is never an object key, authorization input, or security decision.
2. **Every tarball URL is immutable.** The canonical URL remains on the
   original artifact, and an integrity URL always returns the bytes named by
   its digest.
3. **Replacements are explicitly marked.** A version whose selected artifact
   is a replacement advertises its ordinal in `dist.revision`. Originals
   serve ordinary unmarked metadata: the canonical route is pinned to
   revision zero, so the absence of a marker is itself a durable statement.
4. **Revision ordinals are stable and uniquely allocated.** A distinct
   accepted replacement receives the next positive integer. Numbers are never
   reassigned. Selecting a previously accepted artifact again uses its
   existing ordinal. Allocation serializes on `(registry, name, version)`
   and holds two uniqueness constraints within that identity: each ordinal is
   assigned at most once, and each digest maps to exactly one ordinal. The
   registry owns the ordinal history: provider renames, replacements, or
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

The signed document carries a monotonic `sequence`. pnpr persists the highest
sequence it has accepted per provider and rejects any manifest whose sequence
is not greater: a valid signature over an older document must not be able to
replay earlier selections and reselect a vulnerable revision. Moving selection
backward — withdrawing a bad patch, restoring the original — therefore always
requires either a new, higher-sequence manifest from the provider or an
explicit, separately authorized operator operation, never a replayed
document. The checkpoint advances atomically with — never before — the
durable acceptance of the manifest's work: a crash mid-ingestion leaves the
previous checkpoint in place, so the same sequence can be retried rather
than becoming permanently unacceptable.

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

The complete sha512 digest uses base64url without padding — exactly one
canonical encoding (86 characters for sha512); servers reject padded,
re-encoded, or percent-encoded variants. Raw SRI base64 is not
embedded because `/`, `+`, and `=` have awkward URL and intermediary
semantics. Package metadata uses standard SRI base64 with no extensions; the
selected replacement's ordinal travels separately in `dist.revision`, and the
URL is derived from the digest alone.

The package name and version are absent from the URL. The digest locates the
object in the registry's content-addressed index, deduplicates identical
tarballs, and keeps the endpoint independent of provider naming.

The registry base remains load-bearing for security. A request to
`/~main/-/tarballs/...` is authorized in `main` and succeeds only if that
registry references the digest through at least one package identity the
requesting principal may access. When one registry serves packages under
different access rules, the digest route evaluates the principal against
those package-level rules; "the registry references this digest" alone is
not sufficient. pnpr may deduplicate physical storage globally, but it must
not expose an object merely because another organization or registry stored
the same digest. Knowledge of a digest is not a bearer credential.

Successful public responses are ordinary immutable CDN objects:

```http
HTTP/1.1 200 OK
Cache-Control: public, max-age=31536000, immutable
ETag: "<full-digest>"
Content-Digest: sha-512=:<base64>:
```

Private registries use cache directives compatible with their authorization
model. pnpr verifies content against the digest before storing or serving it.

### Advertising the selected revision

A version whose selected artifact is a replacement carries the ordinal as a
plain integer field beside an ordinary SRI integrity:

```jsonc
"dist": {
  "tarball": "https://registry.example/-/tarballs/sha512/<digest>",
  "integrity": "sha512-<digest-base64>",
  "revision": 2
}
```

`dist.integrity` remains a standard Subresource Integrity value with no
extensions, so every strict SRI consumer parses it. The numbering matches the
structured `dist.revisions` record, which counts the original as revision
`0`.

pnpr applies these rules:

- it emits `dist.revision` exactly when the selected artifact is a
  replacement (`N ≥ 1`); originals serve unmarked metadata;
- it assigns `1` to the first distinct accepted replacement and monotonically
  increasing values to later replacements;
- it never uses the provider name or provider revision in the field;
- it preserves ordinals in `dist.revisions`, audit output, and logs;
- it excludes the field from object lookup, equality, authorization, VEX, and
  integrity verification;
- it treats structured revision metadata as authoritative if the field is
  forged or inconsistent;
- a field-only change cannot create a new stored artifact.

npm-compatible clients ignore the unknown `dist` field and follow
`dist.tarball` normally. pnpm records the ordinal in its lockfile as the
durable signal to construct the digest URL, as the companion RFC specifies.

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
        "integrity": "sha512-<echo-r2-base64>",
        "revision": 2,
        "revisions": [
          {
            "revision": 0,
            "integrity": "sha512-<upstream>",
            "tarball": "https://registry.example/-/tarballs/sha512/<upstream-base64url>",
            "source": "upstream",
            "publishedAt": "2020-04-03T00:00:00.000Z",
            "manifest": { "dependencies": {} }
          },
          {
            "revision": 1,
            "integrity": "sha512-<echo-r1>",
            "tarball": "https://registry.example/-/tarballs/sha512/<echo-r1-base64url>",
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
            "tarball": "https://registry.example/-/tarballs/sha512/<echo-r2-base64url>",
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
`dist.integrity` and whose ordinal equals `dist.revision`. Existing
npm-compatible clients ignore `dist.revisions` and fetch selected
`dist.tarball` normally.

The history provides:

- discoverability through `npm view` and registry APIs;
- provenance and advisory/VEX metadata for exact bytes;
- an auditable sequence when a provider reissues a patch;
- the allow-list used when a verifier checks a historical digest;
- retention and withdrawal state if the schema grows it later.

Each revision entry carries a `manifest` object holding the
resolution-relevant fields of its artifact — the same per-version subset the
abbreviated packument serves: `dependencies`, `optionalDependencies`,
`peerDependencies`, `peerDependenciesMeta`, `bundledDependencies`, `bin`,
`engines`, `os`, `cpu`, `libc`, and `hasInstallScript`.
Revisions may legally differ in any of them, and the companion RFC lets a
client resolve a non-selected revision — an override pinning `+rN` — without
first downloading its tarball. These fields are always derived from the
verified tarball's `package.json`, exactly as the selected projection is; a
provider manifest cannot declare metadata that diverges from its artifact's
bytes. Registry-managed mutable metadata is deliberately excluded from the
subset: `deprecated` can change after publication through `npm deprecate`,
so it lives on the projected version entry as ordinary mutable registry
metadata, never inside an immutable revision record. The full packument may
additionally carry all provenance fields. Abbreviated metadata should keep
current `dist` plus the integrity, digest URL, and `manifest` of historical
revisions, so revision pinning works from the metadata pnpm actually
fetches. An unpatched version does not need a `dist.revisions` array or a
`dist.revision` field; it serves ordinary metadata, and its `dist.tarball`
may use the canonical or the digest URL.

### The canonical URL remains the original artifact

The canonical npm tarball route is never repointed:

```text
https://registry.example/ejs/-/ejs-2.7.4.tgz
```

It continues returning the original upstream bytes forever. Current package
metadata may advertise another artifact through `dist.tarball`, just as npm
package documents may already advertise non-canonical tarball URLs.

This provides backward compatibility:

- any lockfile entry without a recorded revision — legacy or a deliberate
  revision-zero pin — reconstructs the canonical URL and receives original
  bytes on every registry;
- a lockfile entry recording replacement `N` constructs its digest URL;
- changing the selected revision changes only mutable package metadata;
- no CDN URL ever refers to two bodies.

Patch delivery to npm-compatible clients does not depend on the canonical
route changing. A fresh npm or Yarn resolution follows `dist.tarball` — which
points at the selected replacement's digest URL — and records that URL with
the patched integrity in its own lockfile, so registry-selected patches reach
those clients too; what stays pnpm-only is revision *management* (host-free
lockfile entries, `+rN` pinning, revision refresh). A client that ignores
`dist.tarball` and reconstructs the conventional URL from name and version
fails loudly, because original bytes cannot match the advertised replacement
integrity — never a silent wrong install. And existing lockfiles of every
client keep installing exactly what they pin: no registry design can
retroactively patch a resolution that pins the original integrity, because
serving different bytes would only turn reproducible installs into integrity
failures. Canonical immutability makes that fact honest instead of breaking.

There is no need to use an integrity mismatch as protocol negotiation.

### Manifest metadata across revisions

A patch may update dependencies as well as source files. Revisions are not
required to have identical dependency metadata.

pnpr reads the selected artifact's `package.json` only after the tarball passes
integrity verification and projects its resolution-relevant fields into the
current version document. When a revision becomes selected, the entire
projected version entry and `dist` move atomically.

When the first replacement is accepted, pnpr MUST materialize revision zero
from the verified original artifact — including its complete integrity and
resolution-relevant manifest — before atomically projecting the replacement.
Every later selection rebuilds the version document's artifact-derived fields
from the selected revision's manifest. Selecting revision zero restores the
original artifact-derived fields and omits `dist.revision`, while retaining the
complete revision history. A change of upstream source MUST NOT redefine
revision zero; an artifact with a different digest is either accepted as a new
revision or rejected by registry policy.

The selected revision therefore needs no per-field override: the projected
version entry *is* that revision's metadata. Non-selected revisions expose
theirs through `dist.revisions[].manifest`, so a client pinning an older or
newer ordinal resolves with the dependency set, peer requirements, and
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
`name@version`. If the selected artifact changed, it replaces the integrity,
the recorded revision, and the package snapshot atomically. It does not rerun
semver selection or change package versions.

The companion RFC also gives a workspace explicit revision selection: a spec
of the form `<version>+rN` — as an override target or a declared dependency —
pins the dependency to that exact version and registry revision `N`. `+r0`
keeps the original, a positive ordinal adopts or freezes a specific
replacement. `pnpm update --patches` skips pinned packages, and a revision
whose bytes this registry's policy refuses fails the install explicitly
instead of falling forward to another artifact.

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

Steps 4 through 6 execute as one transaction that serializes on
`(registry, name, version)`, with uniqueness enforced separately on the
ordinal and on the digest mapping within that identity. A retry after a
partial failure observes the existing digest mapping instead of allocating
again, and two concurrent refreshes carrying different digests cannot both
receive the same next ordinal.

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

Immutability is a statement about bytes, not availability: a successful
response for a digest URL always carries the same bytes, but policy may stop
the URL from answering successfully. A transition to refusal is effective
only once the edge stops serving, so deployments that publish long public
cache lifetimes must purge affected objects on such transitions, and a
registry whose policy may require immediate refusal should prefer
authorization-aware (non-shared) caching for the affected class of artifacts
over year-long shared caching. Policy applies to the artifact, never the
route: the canonical URL and the digest URL must give the same answer for the
same bytes, so the canonical route can never serve as a policy bypass.

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

### Encode the revision ordinal in the tarball URL

Distinct per-revision URLs such as
`ejs/-/ejs-2.7.4+echo.1.tgz` avoid repointing the canonical route: old URLs
keep serving their bytes, and metadata advances to a new URL on each accepted
artifact. npm clients ignore the build metadata and see the regular version.
The URLs are human-readable, and a proxy log identifies the artifact without a
digest-to-package index.

The scheme shares the append-only intent of this design but the URL only
names the bytes — it does not commit to them. Immutability becomes a policy
the registry must honor rather than a property clients can verify: a buggy or
compromised registry can serve different bytes at the same ordinal URL, and
nothing in the request detects it before integrity verification of a
completed download. The digest route is self-authenticating — the CDN cache
key is the content itself, pnpm constructs the URL from registry plus the
integrity it already stores, and the no-fallback fetch rule holds by
construction. Embedding a provider name in the URL also leaks provider
identity into a route that survives provider changes, which the neutral
ordinal deliberately avoids.

A registry may still expose such a route as a convenience alias for humans
and non-pnpm tooling. The fetch convention pnpm relies on remains the digest
URL.

### Use only a digest prefix

Rejected. A full sha512 value is small relative to an HTTP request or tarball.
Prefixes introduce ambiguity, collision work, and an arbitrary minimum length.
The complete digest is package metadata, not a credential.

### Mark revisions inside the SRI integrity value

Earlier drafts carried the ordinal as an SRI option — `sha512-<digest>?r2`,
with `?r0` (or a bare `?`) marking originals — so the marker traveled inside
the integrity string itself. Rejected in favor of the separate
`dist.revision` field:

- SRI reserves `?options` for extensions, but real npm-ecosystem SRI parsers
  do not implement the grammar. pnpm's Rust `ssri` splits on `-` and takes the
  remainder as the digest, so an option parses *successfully* into a corrupted
  digest and round-trips intact, failing far from the parse. Making option
  preservation load-bearing protocol state is a class of fragility a plain
  JSON field does not have;
- integrity-string equality is used as byte equality across the ecosystem, so
  an ordinal inside the value makes a revision-only difference read as a
  content change. The companion pnpm RFC records where this bites a client;
- marking originals binds every metadata entry and lockfile entry to
  revision-aware registries, although the canonical route already guarantees
  originals without any marker;
- a plain integer field is ordinary JSON everywhere, and the ordinal appears
  on its own line in a lockfile diff instead of at the end of a
  ninety-character integrity string.

The `r` letter survives only in the inline `+rN` spec syntax, where build
metadata needs a compact marker: `v` reads as "version" — the association
this design exists to avoid; `b` means a source-identical rebuild in Debian
(`+b1`) and collides with hexadecimal digest fragments; Alpine and Gentoo's
`-rN` downstream revision numbering is the exact precedent.

### Put provider identity in the revision field

A value such as `revision: "echo-r2"` leaks the provider's naming scheme into
resolution metadata and makes a provider rename look like an artifact change.

Neutral integer ordinals are sufficient for lockfile visibility. Structured
`dist.revisions` and signed provider manifests remain authoritative for
provenance, fixes, attestations, and policy.

### Omit structured revision history

A bare ordinal cannot authenticate provider identity, fixes, or attestations.
It also cannot tell a verifier whether a historical digest was ever
associated with a particular `name@version`.

The compact `dist.revision` field complements but does not replace
`dist.revisions`.

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
6. Emit `dist.revision` with stable integer ordinals for selected
   replacements; serve originals unmarked.
7. Project selected digest URLs and append-only `dist.revisions` into full and
   abbreviated packuments.
8. Keep original canonical tarball routes immutable.
9. Make provider refresh atomic and audit every selected-revision transition.
10. Extend audit handling to understand integrity-scoped fixes and
    attestations.

Tests should cover:

- unpatched originals served with ordinary unmarked metadata, retrievable
  through both canonical and digest routes;
- original, first replacement, and second replacement retrievable by digest;
- `1` assigned to the first replacement and stable monotonic ordinals;
- malformed `dist.revision` values rejected from protocol metadata;
- `dist.tarball` selecting the latest revision without changing name/version;
- canonical URLs continuing to return original bytes after every refresh;
- full and abbreviated metadata exposing required revision history;
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
- immutable CDN caching, range requests, and warm-CDN behavior;
- atomic refresh with no metadata-visible-but-unavailable interval;
- selected metadata derived from the verified tarball;
- a dependency-changing revision updating snapshot and integrity together;
- provider conflict rejection;
- a replayed lower-sequence manifest rejected despite a valid signature;
- a crash between ingestion and checkpoint advance leaving the same sequence
  retryable;
- concurrent refreshes and retries allocating exactly one ordinal per digest,
  and never the same ordinal to two digests;
- ordinals surviving provider rename, replacement, and removal unrenumbered;
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

- Is `/-/tarballs/<algorithm>/<digest>` the right registry-neutral endpoint?
- Is `dist.revisions` the right package-document field name?
- Beyond the resolution-relevant fields required for revision selection, which
  projected manifest fields should historical entries retain?
- Does `pnpm update --patches` update every artifact revision, only
  security-motivated revisions, or accept provider and advisory filters?
- Which npm-compatible clients honor a non-conventional `dist.tarball` (npm,
  Yarn classic, Yarn Berry via `__archiveUrl`, Bun), and which reconstruct
  conventional URLs and therefore hit the loud-failure path? This needs
  verification and client-matrix tests, not assumption.
- How long must a pnpr deployment retain historical content, and how should it
  communicate a policy shorter than lockfile lifetime?
- Should vulnerable historical artifacts remain retrievable by default or
  require an explicit reproducibility policy?
