# Release records and upstream verification for pnpr

> Status: draft for discussion. Split from
> [pnpm/rfcs#44](https://github.com/pnpm/rfcs/pull/44). Builds on
> [publish-time manifest verification](./0000-publish-manifest-verification.md),
> [SQL metadata storage](./0000-sql-metadata-storage.md), and the release
> record that [split packuments](./0000-split-packuments.md) serves.

## Summary

pnpr already serves each version's release record, an immutable document at
`<registry-base>/<name>/<version>+r<revision>` with the version's archive facts
and archive-derived manifest ([split packuments](./0000-split-packuments.md)).
This RFC makes that record verifiable with the standards other ecosystems
already use, instead of a scheme of pnpr's own:

- Each record is appended to the registry's **transparency log**, in the
  formats Go's checksum database uses, so a registry, mirror, or cache cannot
  serve one client a different record than it serves everyone else.
- Endorsements and publication statements are **in-toto Statements in DSSE
  envelopes**, the format npm provenance, Sigstore, and SLSA use. They sign the
  exact bytes served, so nothing is canonicalized.
- Publisher provenance stays **Sigstore** evidence, as npm provenance is today.
- Clients obtain a registry's keys through **TUF** root metadata, which handles
  rotation and revocation.
- A registry is identified by its **log's verifier key**, not by a hostname, and
  a package by its **Package URL**.
- For proxied packages, pnpr serves **archive-derived metadata**, as PyPI's
  index serves metadata extracted from each wheel, and verification is **on by
  default and fails closed**, as Go's checksum verification is.

## Motivation

Publish-time manifest verification makes pnpr serve archive-derived metadata
for hosted packages. Clients still have to trust that the registry did so, and
proxied upstream packages are not verified at all.

A release record is a long-lived unit that can be signed, logged, cached,
audited, and pinned by later features (resolution snapshots,
[atomic workspace publication](./0000-atomic-workspace-publication.md))
independently of mutable package state such as tags and deprecations.

npm's registry signatures sign only `${name}@${version}:${integrity}` with a key
the registry publishes at `/-/npm/v1/keys`. They bind the archive bytes but not
the dependencies, scripts, or publisher that clients resolve from, and a key
fetched from the registry it vouches for does not protect against that
registry.

### Threat model

| Attack | Defense |
| --- | --- |
| A network attacker alters a response. | TLS, and the archive's integrity in the lockfile. Records add nothing. |
| A registry, mirror, or cache serves a client metadata that does not match the archive. | The record binds the archive-derived manifest to the archive's integrity. |
| A compromised registry serves one client a forged record, or rewrites a record after the fact. | The transparency log: a client accepts a record only with a proof that it is in the log, and witnesses check that the log only ever grows. A forged record must be logged, where monitors see it. This is the protection Go's checksum database gives against a compromised proxy. |
| A stolen publisher credential publishes a well-formed malicious version. | Publication evidence. In practice this is the attack seen: eslint-scope in 2018 ([npm/npm#21202](https://github.com/npm/npm/issues/21202)) and axios in 2026 ([npm/rfcs#860](https://github.com/npm/rfcs/issues/860), [npm/cli#9242](https://github.com/npm/cli/issues/9242), 68 👍). axios had been publishing with provenance, and the malicious versions were published manually without it. A trust policy that compares how versions were published catches it. |
| A registry's key is stolen or must be replaced. | TUF root metadata, signed by offline keys, rotates it, and clients stop accepting the old key. |

Records do not prove a package is benign. A registry endorsement means the
registry accepted the manifest-to-archive binding; it is not publisher
provenance.

## Detailed Explanation

### Record contents

The record is the one pnpr serves at
`<registry-base>/<name>/<version>+r<revision>`, one per revision, with the
fields that bind it added:

| Field | Contract |
| --- | --- |
| `schema` | Record format version. |
| `purl` | The version's [Package URL](https://github.com/package-url/purl-spec), such as `pkg:npm/%40example/widgets@1.2.0`. |
| `revision` | The revision the route names. |
| `interpretation` | The manifest interpretation profile. |
| `inputs` | Exact byte digests of the archive manifest and shrinkwrap, if present. |
| Archive facts | `integrity` (the digest of the uncompressed tar, as for every pnpr archive), `fileCount`, `unpackedSize`, `requiresBuild`, `bins`, and `variants`. |
| `manifest` | Install-critical metadata derived from the archive. |

The served record's `name` and `version` stay for the clients that read them
today. A version with [platform variants](./0000-platform-variants.md) has one
record covering every archive: each variant entry carries its own tar
`integrity`, sizes, `bins`, platforms, and the fields of its manifest that
differ from the version's. A version without a primary has no top-level
`integrity`.

The record is a deterministic function of the package identity,
interpretation profile, and archive bytes. It names no registry: the log that
holds it does. It contains no timestamps, signatures, provenance, transport
locations, deprecation, or `_npmUser`; the served record already leaves those
out.

```json
{
  "schema": "pnpr-release-record/1",
  "purl": "pkg:npm/%40example/widgets@1.2.0",
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

The fields this RFC adds are additive: a client that reads the served record
today ignores them.

### Bytes, not canonical JSON

pnpr stores a record's bytes when it accepts the version and serves them
unchanged at the record route, which is immutable. Its digest is the SHA-512 of
those bytes; everything that refers to the record, the log entry and every
statement, uses that digest. Clients hash the bytes they received, then parse
them, and never re-serialize. This is how DSSE signs a payload, and why: no
canonicalization scheme has to be agreed on or implemented. pnpr may compress
the response; the digest covers the decoded bytes.

An auditor that re-derives a record from the archive compares the manifest it
derives with the record's `manifest` under the record's profile, not the bytes.

The record is addressed by version and revision, not by its digest:
[split packuments](./0000-split-packuments.md) chose that so the package
document carries no incompressible digest per version.

### Transparency log

Each registry runs an append-only log of its records, in the formats Go's
checksum database uses, now specified by
[C2SP](https://github.com/C2SP/C2SP): checkpoints signed as
[signed notes](https://github.com/C2SP/C2SP/blob/main/signed-note.md), the log
served as [tiles](https://github.com/C2SP/C2SP/blob/main/tlog-tiles.md), and
[witnesses](https://github.com/C2SP/C2SP/blob/main/tlog-witness.md) that
co-sign checkpoints after checking that the log only grew.

- A log entry is the record's PURL, revision, profile, and digest. pnpr appends
  it in the same transaction that accepts the version.
- A second entry for the same PURL, revision, and profile with another digest is
  a forged or rewritten record. Clients reject it, and monitors that read the
  log report it.
- With each record, pnpr serves an inclusion proof against a recent checkpoint.
  A client verifies the proof and the checkpoint's signatures, the log's and the
  witnesses' it requires, before it accepts the record.
- Clients keep the latest checkpoint they have seen and check that a newer one
  is consistent with it, so a registry cannot show different clients different
  logs.

The log holds hosted versions and the versions pnpr derived records for while
proxying an upstream. It holds digests and identities, not records, so a
private package's log entries reveal its name and versions; a private registry
keeps its log, like its records, behind its access rules.

Each deployment runs its own log, and its verifier key is its identity. A
shared log, as Go's ecosystem shares `sum.golang.org`, would need an operator
every deployment trusts and would expose private names; deployments that want
one can agree on a log later without changing the record or the protocol.

Witnesses are what protect a client against a registry that shows different
clients different logs. A public registry's TUF root names the witnesses its
clients require. A private registry may name none: its clients still detect a
rewritten entry, since each keeps the latest checkpoint it saw and checks every
newer one against it, and a CI job that records checkpoints, or a second pnpr
deployment, can witness the log where an organization needs more.

### Statements: in-toto in DSSE

Everything signed about a record is an
[in-toto Statement](https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md)
wrapped in a [DSSE](https://github.com/secure-systems-lab/dsse) envelope, the
format of npm provenance and SLSA. The Statement's subject is the record, by its
PURL and digest; its `predicateType` says what is claimed:

- **Acceptance**, `https://pnpm.io/pnpr/acceptance/v1`: the registry accepted
  the binding of this manifest to this archive under this profile. The log
  entry already proves the record existed; an acceptance statement lets a
  record travel with a claim that does not need the log to be reachable.
- **Publication**, `https://pnpm.io/pnpr/publication/v1`: how the version was
  published (see [Publication evidence](#publication-evidence)).

Publisher provenance keeps its own subject. An npm provenance bundle is a
Sigstore bundle whose SLSA Statement names the `.tgz` digest; pnpr keeps it as
it was published and links it to the record through the `.tgz`-to-tar
derivation it recorded at acceptance.

All of a version's statements and bundles are served beside the record, at
`<registry-base>/<name>/<version>+r<revision>/evidence`, with the inclusion
proof:

```json
{
  "record": "sha512-<record digest>",
  "inclusion": { "checkpoint": "<signed note>", "proof": ["<hash>", "..."] },
  "statements": ["<DSSE envelope>", "..."],
  "sigstore": ["<Sigstore bundle>", "..."]
}
```

Adding a statement, a newer checkpoint, or a re-signature changes this
response, not the record, so it is not immutable: its HTTP validator includes
the evidence generation. It follows the record's access rules, including
`private, no-store` for a caller-gated package.

### Publication evidence

At acceptance, the registry signs a publication statement. Only the registry
produces it; publisher-supplied fields with the same names are stripped. Its
predicate is the version's
[publication](./0000-atomic-workspace-publication.md#publications), which
already records the publishing identity, a workload's repository and commit,
and the commit time, plus:

- the principal kind: a user token, an interactive session, or an OIDC
  workload;
- for a workload, the issuer, subject, and the claims the workload binding
  matched, such as repository and workflow;
- whether the version was staged, and the principal that approved it.

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

### Keys and identity

A registry's keys are distributed as [TUF](https://theupdateframework.io/) root
metadata, as PyPI specified in
[PEP 458](https://peps.python.org/pep-0458/). The root, signed by offline keys,
names the log's checkpoint key, the key that signs statements, and the
witnesses the registry's clients require. Rotating a key is a new root version
signed by the previous one; a client follows the chain and stops accepting the
old key.

- **Public registries.** pnpm ships the TUF roots of the public registries it
  knows, as Go ships the key of `sum.golang.org`.
- **Private registries.** The workspace declares the registry's root, by the
  digest of its first root version, beside the registry's `serverType: pnpr`.
  pnpm does not trust a root fetched from the registry it vouches for unless the
  workspace pinned it.

A registry is identified by its log's verifier key, written as Go writes one,
`<name>+<key hash>+<key>`. The name is an identifier, not a transport location:
a mirror or another hostname serving the same log is the same authority, and a
lockfile that records the authority does not change when the registry moves.
Rebinding an authority to a different log requires an explicit operator action.
Identical archive bytes in two logs do not establish shared ownership.

### Interpretation profiles

A correction to derivation creates new records under a new profile; historical
records are never rehashed. Clients must explicitly support a profile, and an
unsupported profile is a capability error rather than a silent
reinterpretation. Revision numbers do not change when only the interpretation
changes.

A record's identity is its PURL, revision, and profile, as its log entry is.
The record route `<name>/<version>+r<revision>` serves the record under the
profile the version was accepted with, and keeps serving it. A record
re-derived under another profile is served at
`<name>/<version>+r<revision>/records/<profile>`, also immutable, and the
accepted profile's record is at that route too. A client that requires a
profile reads that route and caches records by all three parts, never by
revision alone.

### Upstream verification

For proxied packages, upstream metadata is an unverified claim. pnpr fetches
each version's archive, derives its record under the profile, and serves that,
as PyPI's index extracts each wheel's metadata and serves it with its hash
([PEP 658](https://peps.python.org/pep-0658/)). pnpr pays the download once,
for every client; the client reads archive-derived metadata without fetching
archives to resolve.

| Mode | Contract |
| --- | --- |
| Verify (default) | Resolves from the archive-derived record. A version whose upstream metadata disagrees with its archive in an install-critical field fails closed. |
| Archive authoritative | An operator accepts a recorded disagreement for one package, artifact integrity, and profile, and resolves from the archive-derived record. It does not carry over to another artifact, version, or profile, and never waives archive integrity, safety, or identity checks. |
| Unverified | Resolves from upstream metadata, reported as unverified. Configured for registries or package patterns, as Go's `GONOSUMDB` exempts private modules. |

- When verification is required and fails, resolution fails. It never falls back
  to the unverified mode.
- Archive-authoritative acceptance changes the metadata used for resolution: the
  graph is resolved from the archive manifest, never resolved from upstream
  metadata and patched afterwards.

A survey compared registry metadata with the archive's `package.json` for 8,620
versions: the 1,674 registry packages in pnpm/pnpm's lockfile, the latest
version of the 5,000 most-downloaded packages, and a random older version of
each of the top 2,000. Before normalization, 273 differ (about 3%). All 273 are
explained by the publishing client's normalization, mostly `bin` syntax and
`optionalDependencies` copied into `dependencies`. The full table is in
[publish-time manifest verification](./0000-publish-manifest-verification.md).
No other differences were found. With those normalizations in the profile,
verification on by default fails no version in the survey.

### Client protocol

Extend the `GET /-/pnpr` capability document:

```json
{
  "records": [1],
  "log": "registry.example+1a2b3c4d+<key>",
  "manifestProfiles": ["npm-install-manifest/1"]
}
```

- **Records.** An entry of pnpr's package document
  (`application/vnd.pnpr.install-v2+json`) that carries its manifest is a
  projection of the record: the same manifest and archive facts, beside the
  version's mutable `trust`, selected `revision`, publish time, and
  deprecation, which are not part of the record. A client that does not verify
  resolves from it, as today. A client that verifies records fetches the
  record route for every version it locks, since only those bytes have the
  logged digest, resolves from the record, and caches it on disk forever.
- **Evidence and proofs.** A verifying client fetches each locked version's
  evidence and checks its inclusion proof. Statements are fetched only when the
  active policy needs them.
- **Log.** The log's checkpoints and tiles are served at
  `<registry-base>/-/log/`, as C2SP's tiles specify, for monitors and
  witnesses.

Paths are relative to the effective registry base (`/npm/` or `/npm/~<name>/`).
A digest is never a read capability: every record, evidence, and artifact read
is authorized through its package and registry, and unauthorized callers must
not learn whether a private record exists.

When record support is required, clients do not fall back to ordinary metadata.
A policy block is an explicit refusal, never a skipped version. If the evidence
the active policy requires exceeds response limits, the response is an explicit
error, never a truncated response.

### Caching

Records are the per-version fragments of
[SQL metadata storage](./0000-sql-metadata-storage.md), without their mutable
state, and are served immutable. Evidence is cached by the evidence
generation, which joins the fragment-cache key for any view that carries it.
Log tiles are immutable once full, as C2SP's tiles specify.

## Rationale and Alternatives

1. **A signing service separate from the registry, with no log.** Helps only if
   that service validates what it signs, and a client still cannot tell when a
   compromised registry and signer serve it something no one else sees. A log
   makes that visible without a trusted second party.
2. **RFC 8785 JCS canonicalization.** Lets independent parties produce
   identical bytes, but every producer and verifier must implement it. Signing
   the stored bytes, as DSSE does, needs neither, and an auditor compares the
   derived manifest instead.
3. **A record of pnpr's own, apart from the served one.** An earlier draft
   defined its own record shape, served at a digest-addressed route and inline
   in an extended package document. Two records of one release would have to be
   kept consistent; split packuments already serves an immutable record for
   every version.
4. **A candidate-lookup endpoint.** Returning bounded candidates with inline
   records for a range would spare cold resolution separate fetches. Split
   packuments already inlines the likely picks and fetches only the picked
   versions' records.
5. **Client-side install-time check only.** pnpm compares the extracted
   `package.json` with the resolved manifest. Covers every registry with no
   protocol change, but detects mismatches after resolution and gives no
   portable, verifiable unit.
6. **Sign the whole package document.** Requires re-signing on every tag or
   annotation change and complicates historical reads.
7. **OCI artifact manifests.** A standard digest envelope, but does not settle
   npm manifest interpretation, revision selection, or package authorization.

## Implementation

Target pnpr and pnpm v12. Do not add a feature to pnpm v11.

1. pnpr: add the binding fields to the record it serves, store the exact bytes
   in the SQL metadata layer, and append each record to a C2SP log in the
   acceptance transaction.
2. pnpr: sign acceptance and publication statements as in-toto Statements in
   DSSE envelopes, keep Sigstore bundles, and serve evidence with inclusion
   proofs.
3. pnpr: publish TUF root metadata, and serve the log's checkpoints and tiles.
4. pnpm: verify records, proofs, checkpoint consistency, and the statements the
   trust policy needs; ship the roots of known public registries, and read a
   pinned root for a private one.
5. pnpr: derive records for proxied versions and enforce the upstream modes.

Use existing implementations rather than new ones: Go's `golang.org/x/mod/sumdb`
packages and their Rust counterparts for notes and tiles, a DSSE library, and a
TUF client.

Benchmark cold and warm resolution with verification, and pnpr's cost of
deriving records for proxied packages on a large mirror. pnpr keeps the archive
of each version it derived a record for as long as it keeps the record, as a
mirror keeps the tarballs it serves; the benchmark decides whether derivation
runs on first request or ahead of it. Test that tag changes, deprecations,
delayed attestations, and re-signing preserve record digests; that a second
record for a logged identity is rejected; that an inconsistent checkpoint is
rejected; that a rotated key is followed and the old one refused; and that
private records cannot be read through digest paths.

## Prior Art

- [Go's checksum database](https://go.dev/ref/mod#checksum-database) and the
  [C2SP](https://github.com/C2SP/C2SP) transparency-log specifications.
- [in-toto attestations](https://github.com/in-toto/attestation),
  [DSSE](https://github.com/secure-systems-lab/dsse),
  [SLSA provenance](https://slsa.dev/), and
  [npm provenance](https://docs.npmjs.com/generating-provenance-statements) on
  [Sigstore](https://www.sigstore.dev/).
- [TUF](https://theupdateframework.io/) and its adoption for PyPI in
  [PEP 458](https://peps.python.org/pep-0458/).
- [PEP 658](https://peps.python.org/pep-0658/), metadata extracted from the
  archive and served with its hash.
- [Package URL](https://github.com/package-url/purl-spec).
- [npm registry signatures](https://docs.npmjs.com/about-registry-signatures/).
- [Split packuments](./0000-split-packuments.md), which serves the record this
  RFC makes verifiable.
- [Platform variants](./0000-platform-variants.md) and
  [atomic workspace publication](./0000-atomic-workspace-publication.md), whose
  variants and publications the record and its evidence describe.
- [Registry mounts and declared namespaces](./0000-pnpr-registry-mounts.md).
- [Integrity-addressed patch revisions](./0000-integrity-addressed-patch-revisions.md).
- [Tar integrity for every source](../../text/0000-tar-integrity-for-every-source.md).

## Unresolved Questions and Bikeshedding

None. A package-level publish policy that the registry enforces at
acceptance, such as "only OIDC from this repository", reads the publication
evidence defined here and belongs in an RFC of its own.
