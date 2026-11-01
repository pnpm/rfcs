# Release records and upstream verification for pnpr

> Status: draft for discussion. Split from
> [pnpm/rfcs#44](https://github.com/pnpm/rfcs/pull/44). Builds on
> [publish-time manifest verification](./0000-publish-manifest-verification.md),
> [SQL metadata storage](./0000-sql-metadata-storage.md), and the release
> record that [split packuments](./0000-split-packuments.md) serves.

## Summary

pnpr already serves each version's release record, an immutable document
at `<registry-base>/<name>/<version>+r<revision>` with the version's
archive facts and archive-derived manifest
([split packuments](./0000-split-packuments.md)). This RFC makes that record
verifiable: it adds the authority, interpretation profile, and input digests
that bind it, fixes its bytes so its digest identifies it, lets a configured
signer endorse it, keeps publication evidence beside it, and lets pnpr
compare upstream metadata with archive-derived records for proxied packages
under an explicit policy.

## Motivation

Publish-time manifest verification makes pnpr serve archive-derived metadata
for hosted packages. Clients still have to trust that the registry did so,
and proxied upstream packages are not verified at all.

A release record is a long-lived unit that can be signed, cached, audited, and
pinned by later features (resolution snapshots,
[atomic workspace publication](./0000-atomic-workspace-publication.md))
independently of mutable package state such as tags and deprecations.

### Threat model

This section must be settled before ratification. The record's value depends on
who can produce an endorsement and who verifies it:

- Against a network attacker, TLS and `dist.integrity` already protect the
  archive. Records add nothing there.
- Against a compromised serving registry, a record endorsed by a key held by
  that same registry adds nothing; it can forge both metadata and record.
  Moving the key to a separate signing service helps only if that service
  validates what it signs. A service that signs any record the serving
  registry submits is as compromised as the registry.
- Records help third parties that receive data from a mirror, cache, or export
  and need to verify it without contacting the authority.
- Records give snapshots and receipts a stable identity to pin.
- Against a stolen publisher credential, records add nothing by themselves.
  This is the attack seen in practice: eslint-scope in 2018
  ([npm/npm#21202](https://github.com/npm/npm/issues/21202)) and axios in 2026
  ([npm/rfcs#860](https://github.com/npm/rfcs/issues/860),
  [npm/cli#9242](https://github.com/npm/cli/issues/9242), 68 👍). The
  attacker publishes a well-formed archive whose metadata matches it. What
  catches it is how the version was published: axios had been publishing
  with provenance attestations, and the malicious versions were published
  manually from the maintainer's account without them. See
  [Publication evidence](#publication-evidence).

npm's registry signatures sign only `${name}@${version}:${integrity}`. They
bind the archive bytes but not the dependencies, scripts, or publisher that
clients resolve from. A record binds the archive-derived manifest, which closes
the first gap; publication evidence addresses the second.

The draft should state which of these it targets and size the mechanism to
match. For a signer separate from the serving registry, it must specify:

- **Who validates the binding.** The signer reads the archive by integrity from
  artifact storage and derives the manifest under the named profile itself. It
  does not sign a manifest supplied by the serving registry.
- **Who authorizes signing.** Which publication or acceptance decision the
  signer requires before it endorses a record, and how that decision is
  authenticated to it.
- **Which trust roots clients already hold.** A key fetched from the same
  registry it vouches for adds nothing against that registry. The draft must
  say how clients obtain and rotate trust roots out of band.

## Detailed Explanation

### Record contents

The record is the one pnpr serves at
`<registry-base>/<name>/<version>+r<revision>`, one per revision, with the
fields that bind it added:

| Field | Contract |
| --- | --- |
| `schema` | Record format version. |
| `authority` | The configured origin and its namespace. |
| `name`, `version`, `revision` | The package identity, with the revision the route names. |
| `interpretation` | The manifest interpretation profile. |
| `inputs` | Exact byte digests of the archive manifest and shrinkwrap, if present. |
| Archive facts | `integrity` (the digest of the uncompressed tar, as for every pnpr archive), `fileCount`, `unpackedSize`, `requiresBuild`, `bins`, and `variants`. |
| `manifest` | Install-critical metadata derived from the archive. |

A version with [platform variants](./0000-platform-variants.md) has one
record covering every archive: each variant entry carries its own tar
`integrity`, sizes, `bins`, platforms, and the fields of its manifest that
differ from the version's. A version without a primary has no top-level
`integrity`.

The record is a deterministic function of the authority, package identity,
interpretation profile, and archive bytes. It contains no timestamps,
signatures, provenance, transport locations, deprecation, or `_npmUser`; the
served record already leaves those out.

```json
{
  "schema": "pnpr-release-record/1",
  "authority": "example-hosted-npm",
  "name": "@example/widgets",
  "version": "1.2.0",
  "revision": 0,
  "interpretation": "npm-install-manifest/1",
  "inputs": {
    "packageJson": { "path": "package/package.json", "integrity": "sha512-<bytes>" },
    "shrinkwrap": null
  },
  "integrity": "sha512-<tar digest>",
  "fileCount": 12,
  "unpackedSize": 4231,
  "requiresBuild": false,
  "manifest": {
    "dependencies": { "@example/core": "^2.0.0" }
  }
}
```

The fields this RFC adds are additive: a client that reads the served
record today ignores them. An artifact digest identifies bytes; a record
binds them to a package and authority. Identical bytes from separate
registries do not establish shared ownership. Rebinding an authority to a
different source requires an explicit operator action.

### Encoding and transport

These are two separate decisions.

**Encoding** is the deterministic production rule. Independent parties that
derive a record from the same archive and profile must produce the same bytes,
for example a separate signer or an auditor re-deriving records. The strawman
is [RFC 8785 JCS](https://www.rfc-editor.org/rfc/rfc8785), with
`recordIntegrity` = `sha512-` + base64(`SHA512(JCS_UTF8(record))`). JCS input
rejects duplicate keys, invalid Unicode, and integers outside the JSON safe
range.

**Transport** is how clients receive the record. pnpr stores the exact
encoded bytes when it accepts the version and serves them unchanged at the
record route; the route is immutable, so the bytes never change. Clients
hash the received bytes, then parse them, and never re-serialize. Clients
then do not need a JCS implementation. Only producers and auditors that
rebuild records need one. pnpr may compress the response; the digest covers
the decoded bytes.

The record is addressed by version and revision, not by its digest:
[split packuments](./0000-split-packuments.md) chose that so the package
document carries no incompressible digest per version. A client that needs
the digest computes it from the bytes it received.

### Endorsements and evidence

An endorsement authenticates a record's integrity under a release-record
signing context. Evidence carries source signatures, provenance, publication
statements, and source-to-tar derivation links, each keeping its original
subject. They are served apart from the record, at
`<registry-base>/<name>/<version>+r<revision>/evidence`, as
`{ "recordIntegrity", "endorsements": [], "evidence": [] }`. Adding evidence
or rotating signers changes that envelope, not the record or its integrity,
so the envelope is not immutable: its HTTP validator includes its evidence
generation. It follows the record's access rules, including
`private, no-store` for a caller-gated package.

A registry endorsement means the registry accepted the manifest-to-archive
binding. It is not publisher provenance and does not prove a package is benign.

### Publication evidence

At acceptance, the registry writes a publication statement as detached
evidence for the record. Only the registry produces it; publisher-supplied
fields with the same names are stripped. It is the evidence form of the
version's [publication](./0000-atomic-workspace-publication.md#publications),
which already records the publishing identity, a workload's repository and
commit, and the commit time, and it names:

- The principal kind: a user token, an interactive session, or an OIDC
  workload.
- For a workload, the issuer, subject, and the claims the workload binding
  matched, such as repository and workflow.
- Whether the version was staged, and the principal that approved it.
- The acceptance time.

Clients evaluating `trustPolicy` read this statement instead of inferring trust
from `_npmUser`, and pnpr computes the `trust` rank of each entry in its package
document ([split packuments](./0000-split-packuments.md)) from it. npm
overloaded `_npmUser` when it added staged publishing, so the approver replaced
the workload and tools lost the trusted-publishing signal
([npm/cli#9397](https://github.com/npm/cli/issues/9397)).

pnpr records no such evidence today. A workload credential resolves to an
ordinary user identity named by the binding's `username`, and publication
writes only `_npmUser.name`. A keyless workload publish is therefore
indistinguishable from a token publish by the same name, and `trustPolicy`
cannot protect pnpr-hosted packages. The npm-compatible view should also carry
`_npmUser.trustedPublisher` for workload publications, so existing clients
work.

### Interpretation profiles

A correction to derivation creates new records under a new profile; historical
records are never rehashed. Clients must explicitly support a profile, and an
unsupported profile is a capability error rather than a silent
reinterpretation. Revision numbers do not change when only the interpretation
changes.

### Upstream observations and policy

For proxied packages, fetched metadata is an unverified observation. A
per-version observation holds one version's profile-interpreted install claims
and claimed artifact integrity; it excludes unrelated versions, tags, and
signatures, so unrelated upstream changes do not invalidate it. Comparisons
between observations and archive-derived records are keyed by both
integrities.

| Mode | Resolution contract |
| --- | --- |
| Compatibility | Resolves from observed upstream metadata; reports it as unverified. |
| Verify and match | Requires a verified archive with matching install-critical metadata; disagreements fail closed. |
| Archive authoritative | Explicitly accepts the archive-derived manifest despite a recorded disagreement. |

- An archive-authoritative approval binds the package, the complete artifact
  integrity, and the interpretation profile. It does not carry over to another
  artifact, version, or profile. It never waives archive integrity, archive
  safety, or package identity checks.
- When verification is required and fails, resolution fails. It never falls
  back to compatibility mode.
- Archive-authoritative acceptance changes the metadata used for resolution:
  the graph is resolved from the archive manifest, never resolved from upstream
  metadata and patched afterwards.

Verification costs downloads. Range selection can nominate candidates from
observed metadata, but each selected candidate's archive must be fetched before
its graph contribution is accepted, and rejected candidates can require more
fetches. Lockfile-only resolution becomes much more expensive. A corpus study
of upstream mismatches should precede choosing defaults.

A survey compared registry metadata with the archive's `package.json` for
8,620 versions: the 1,674 registry packages in pnpm/pnpm's lockfile, the latest
version of the 5,000 most-downloaded packages, and a random older version of
each of the top 2,000. Before normalization, 273 differ (about 3%). All 273
are explained by the publishing client's normalization, mostly `bin` syntax
and `optionalDependencies` copied into `dependencies`. The full table is in
[publish-time manifest verification](./0000-publish-manifest-verification.md).
No other differences were found. Once the profile defines those
normalizations, "verify and match" is
viable without frequent failures, and its cost is the archive downloads rather
than operator overrides.

### Client protocol

Extend the `GET /-/pnpr` capability document:

```json
{
  "records": [1],
  "manifestProfiles": ["npm-install-manifest/1"]
}
```

- **Records.** Clients read records where they read them today: an entry of
  pnpr's package document (`application/vnd.pnpr.install-v2+json`) that
  carries its manifest is the record's content inline, and a picked version
  without one is fetched from its record route. A client that verifies
  records fetches the record route for every version it locks, since only
  those bytes are the record, and caches them on disk forever.
- **Evidence.** A client whose policy needs endorsements or evidence fetches
  the version's envelope from its evidence route.

Paths are relative to the effective registry base (`/npm/` or `/npm/~<name>/`).
A digest is never a read capability: every record, evidence, and artifact
read is authorized through its package and registry, and unauthorized
callers must not learn whether a private record exists.

When record support is required, clients do not fall back to ordinary
metadata. A policy block is an explicit refusal, never a skipped version. If
the endorsements and evidence the active policy requires exceed response
limits, the response is an explicit error, never a truncated envelope.

### Caching

Records are the per-version fragments of
[SQL metadata storage](./0000-sql-metadata-storage.md), without their mutable
state, and are served immutable. Envelopes are cached by the evidence and
endorsement generation, which joins the fragment-cache key for any view
that carries them. A record's integrity is not an ETag for its envelope,
because evidence can change without changing the record.

## Rationale and Alternatives

1. **Client-side install-time check only.** pnpm compares the extracted
   `package.json` with the resolved manifest. Covers every registry with no
   protocol change, but detects mismatches after resolution and gives no
   portable, signable unit.
2. **Sign the whole package document.** Requires re-signing on every tag or
   annotation change and complicates historical reads.
3. **OCI artifact manifests.** A standard digest envelope, but does not settle
   npm manifest interpretation, revision selection, or package authorization.
4. **A separate record from the served one.** An earlier draft defined its
   own record shape, served at a digest-addressed route and inline in an
   extended package document. Two records of one release would have to be
   kept consistent, and split packuments already serves an immutable record
   for every version; signing that one costs a few fields.
5. **A candidate-lookup endpoint.** Returning bounded candidates with inline
   records for a range would spare cold resolution separate fetches. Split
   packuments already inlines the likely picks and fetches only the picked
   versions' records.

## Implementation

Target pnpr and pnpm v12. Do not add a feature to pnpm v11.

1. Settle the threat model, encoding, authority mapping, and signer trust
   configuration. Add cross-implementation fixtures.
2. Produce records during publication acceptance, add the binding fields to
   the record pnpr already serves, and store the exact bytes in the SQL
   metadata layer.
3. Serve evidence envelopes, and add client verification.
4. Add upstream observations, comparisons, and policy modes.

Benchmark cold and warm resolution, and the extra downloads required by
verified lockfile-only resolution. Test that tag changes, deprecations,
delayed attestations, and re-signing preserve record integrities, and that
private records cannot be read through digest paths.

## Prior Art

- [Split packuments](./0000-split-packuments.md), which serves the record
  this RFC signs.
- [Platform variants](./0000-platform-variants.md) and
  [atomic workspace publication](./0000-atomic-workspace-publication.md),
  whose variants and publications the record and its evidence describe.
- [Registry mounts and declared namespaces](./0000-pnpr-registry-mounts.md).
- [Integrity-addressed patch revisions](./0000-integrity-addressed-patch-revisions.md).
- [Tar integrity for every source](../../text/0000-tar-integrity-for-every-source.md).
- [npm registry signatures](https://docs.npmjs.com/about-registry-signatures/).
- [JCS](https://www.rfc-editor.org/rfc/rfc8785).
- [pnpm agent proposal](https://github.com/pnpm/rfcs/pull/9), for file-level
  transfer, which this draft does not cover.

## Unresolved Questions and Bikeshedding

- Which threat does a record endorsement defend against, and who holds the
  signing key?
- Is JCS the right production rule, and should any client re-serialize?
- How are authority identities made portable without embedding deployment
  hostnames in lockfiles?
- Which signatures should clients require, and how are trust roots configured
  and rotated?
- What does a corpus study show about upstream mismatches and verification
  downloads, and which upstream mode should be the default? The survey above
  found only normalization differences across 8,620 versions. Download cost
  still needs measuring.
- Should publication evidence be its own RFC, together with a package-level
  publish policy that the registry enforces at acceptance, such as "only OIDC
  from this repository"?
- Should the fields that bind a record be served in every record, or only
  where a registry signs records?
- The record route is immutable, so it serves the record of the profile a
  version was accepted under. Where is a record re-derived under a later
  interpretation profile served, and how does a client ask for it?
