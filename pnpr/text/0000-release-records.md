# Release records and upstream verification for pnpr

> Status: draft for discussion. Split from
> [pnpm/rfcs#44](https://github.com/pnpm/rfcs/pull/44). Builds on publish-time
> manifest verification and SQL metadata storage.

## Summary

Represent each accepted package version as an immutable, content-addressed
release record that binds an authority, a package identity, an interpretation
profile, the archive-derived install manifest, and the archive integrity.
Let a configured signer endorse records, let clients fetch and verify them, and
let pnpr compare upstream metadata with archive-derived records for proxied
packages under an explicit policy.

## Motivation

Publish-time manifest verification makes pnpr serve archive-derived metadata
for hosted packages. Clients still have to trust that the registry did so,
and proxied upstream packages are not verified at all.

A release record is a long-lived unit that can be signed, cached, audited, and
pinned by later features (resolution snapshots, workspace publication)
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

| Field | Contract |
| --- | --- |
| Schema | Record format version. |
| Authority | The configured origin and its namespace. |
| Package | Ecosystem, name, version, and revision when applicable. |
| Interpretation | The manifest interpretation profile. |
| Inputs | Exact byte digests of the archive manifest and shrinkwrap, if present. |
| Manifest | Install-critical metadata derived from the archive. |
| Artifact | Algorithm, digest, size, and kind of the accepted archive. |

The record is a deterministic function of the authority, package identity,
interpretation profile, and archive bytes. It contains no timestamps,
signatures, provenance, or transport locations.

```json
{
  "record": {
    "schema": "pnpr-release-record/1",
    "authority": "example-hosted-npm",
    "package": { "ecosystem": "npm", "name": "@example/widgets", "version": "1.2.0" },
    "interpretation": "npm-install-manifest/1",
    "inputs": {
      "packageJson": { "path": "package/package.json", "integrity": "sha512-<bytes>" },
      "shrinkwrap": null
    },
    "manifest": {
      "dependencies": { "@example/core": "^2.0.0" },
      "hasInstallScript": false
    },
    "artifact": { "kind": "tgz", "integrity": "sha512-<archive>", "size": 4231 }
  },
  "recordIntegrity": "sha512-<record-bytes>",
  "endorsements": [],
  "evidence": []
}
```

An artifact digest identifies bytes; a record binds them to a package and
authority. Identical bytes from separate registries do not establish shared
ownership. Rebinding an authority to a different source requires an explicit
operator action.

### Encoding and transport

These are two separate decisions.

**Encoding** is the deterministic production rule. Independent parties that
derive a record from the same archive and profile must produce the same bytes,
for example a separate signer or an auditor re-deriving records. The strawman
is [RFC 8785 JCS](https://www.rfc-editor.org/rfc/rfc8785), with
`recordIntegrity` = `sha512-` + base64(`SHA512(JCS_UTF8(record))`). JCS input
rejects duplicate keys, invalid Unicode, and integers outside the JSON safe
range.

**Transport** is how clients receive the record. pnpr stores the exact encoded
bytes and transmits them unchanged. Clients hash the received bytes, then
parse them, and never re-serialize. Clients then do not need a JCS
implementation. Only producers and auditors that rebuild records need one.

Either way, `recordIntegrity`, `endorsements`, and `evidence` are outside the
hashed input, and storage row IDs never enter the record.

### Endorsements and evidence

An endorsement authenticates a record's integrity under a release-record
signing context. Evidence carries source signatures, provenance, and
source-to-tar derivation links, each keeping its original subject. Adding
evidence or rotating signers changes the envelope, not the record integrity,
so the envelope's HTTP validator includes its evidence generation.

A registry endorsement means the registry accepted the manifest-to-archive
binding. It is not publisher provenance and does not prove a package is benign.

### Publication evidence

At acceptance, the registry writes a publication statement as detached
evidence for the record. Only the registry produces it; publisher-supplied
fields with the same names are stripped. It names:

- The principal kind: a user token, an interactive session, or an OIDC
  workload.
- For a workload, the issuer, subject, and the claims the workload binding
  matched, such as repository and workflow.
- Whether the version was staged, and the principal that approved it.
- The acceptance time.

Clients evaluating `trustPolicy` read this statement instead of inferring trust
from `_npmUser`. npm overloaded `_npmUser` when it added staged publishing, so
the approver replaced the workload and tools lost the trusted-publishing signal
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

A first survey covered the 1,674 registry packages in pnpm/pnpm's lockfile.
Before normalization, 46 differ between registry metadata and the archive's
`package.json`. All 46 are explained by the publishing client's normalization:
`bin` syntax (43), `optionalDependencies` copied into `dependencies` (3), the
`gypfile` install-script injection (1), and an added `engines` default (1).
No other differences were found. Once the profile defines those
normalizations
([pnpm/rfcs#47](https://github.com/pnpm/rfcs/pull/47)), "verify and match" is
viable without frequent failures, and its cost is the archive downloads rather
than operator overrides.

### Client protocol

Extend the `GET /-/pnpr` capability document:

```json
{
  "records": [1],
  "recordCandidates": [1],
  "manifestProfiles": ["npm-install-manifest/1"]
}
```

- **Extended metadata.** An opt-in representation
  (`application/vnd.pnpr.install-v1+json`) includes each accepted version's
  complete envelope as `pnprRecord`, with `Vary: Accept`.
- **Candidate lookup.** `POST <registry-base><package>/-/record-candidates`
  with a range, profiles, and a candidate limit returns bounded candidates with
  inline envelopes, so cold resolution needs no separate record fetches. Paging
  cursors bind the captured package index and access context. Responses are
  private to the caller.
- **Standalone retrieval.**
  `GET <registry-base><package>/-/records/sha512/<base64url-digest>` for audit.

Paths are relative to the effective registry base (`/npm/` or `/npm/~<name>/`).
A digest is never a global read capability: every record, evidence, and
artifact read is authorized through its package and registry, and
unauthorized callers must not learn whether a private record exists.

When record support is required, clients do not fall back to ordinary
metadata. A policy block is an explicit refusal, never an empty candidate list
or a skipped candidate. If the endorsements and evidence the active policy
requires exceed response limits, the response is an explicit error, never a
truncated envelope.

### Caching

Generated views that carry envelopes extend the fragment-cache key from SQL
metadata storage, which already includes the interpretation profile, with the
evidence/endorsement generation. A record's integrity is not an ETag for its
envelope, because evidence can change without changing the record.

## Rationale and Alternatives

1. **Client-side install-time check only.** pnpm compares the extracted
   `package.json` with the resolved manifest. Covers every registry with no
   protocol change, but detects mismatches after resolution and gives no
   portable, signable unit.
2. **Sign the whole package document.** Requires re-signing on every tag or
   annotation change and complicates historical reads.
3. **OCI artifact manifests.** A standard digest envelope, but does not settle
   npm manifest interpretation, revision selection, or package authorization.

## Implementation

Target pnpr and pnpm v12. Do not add a feature to pnpm v11.

1. Settle the threat model, encoding, authority mapping, and signer trust
   configuration. Add cross-implementation fixtures.
2. Produce records during publication acceptance and store them in the SQL
   metadata layer.
3. Add negotiated record retrieval and client verification.
4. Add upstream observations, comparisons, and policy modes.

Benchmark cold and warm resolution, and the extra downloads required by
verified lockfile-only resolution. Test that tag changes, deprecations,
delayed attestations, and re-signing preserve record integrities, and that
private records cannot be read through digest paths.

## Prior Art

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
  downloads, and which upstream mode should be the default? The first survey
  above found only normalization differences. Download cost still needs
  measuring.
- Should publication evidence be its own RFC, together with a package-level
  publish policy that the registry enforces at acceptance, such as "only OIDC
  from this repository"?
- Are the inline envelopes, candidate request and cursor schema, and retrieval
  paths the right initial protocol?
