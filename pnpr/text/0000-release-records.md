# Release records and upstream verification for pnpr

> Status: draft for discussion. Split from
> [pnpm/rfcs#44](https://github.com/pnpm/rfcs/pull/44). Builds on
> [publish-time manifest verification](./0000-publish-manifest-verification.md),
> [SQL metadata storage](./0000-sql-metadata-storage.md), and the release
> record that [split packuments](./0000-split-packuments.md) serves.

## Summary

A release record is the immutable statement of what one released artifact
contains and what a resolver reads from it: its digests and its
archive-derived metadata. This RFC defines records for every ecosystem pnpr
serves, npm, Cargo, Python, and container images, as one core with a profile
per ecosystem. For npm, the record is the one pnpr already serves at
`<registry-base>/<name>/<version>+r<revision>`
([split packuments](./0000-split-packuments.md)). It makes records verifiable
with the standards other ecosystems already use, instead of a scheme of
pnpr's own:

- Each record is appended to the registry's **transparency log**, in the
  formats Go's checksum database uses, so a registry, mirror, or cache cannot
  serve one client a different record than it serves everyone else.
- Endorsements and publication statements are **in-toto Statements in DSSE
  envelopes**, the format npm provenance, Sigstore, and SLSA use. They sign the
  exact bytes served, so nothing is canonicalized.
- Publisher provenance stays **Sigstore** evidence, as npm provenance and PyPI
  attestations are today.
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

### The record

Every record has the same core, whatever its ecosystem:

| Field | Contract |
| --- | --- |
| `schema` | Record format version. |
| `purl` | The [Package URL](https://github.com/package-url/purl-spec) of what the record describes, such as `pkg:npm/%40example/widgets@1.2.0` or `pkg:cargo/serde@1.0.210`. |
| `revision` | The registry revision, in an ecosystem that has revisions. |
| `interpretation` | The profile the metadata was derived under. |
| `inputs` | Exact byte digests of the files the metadata was derived from, such as `package.json` or `Cargo.toml`. |
| Artifact facts | The digests and sizes of the artifacts, in the profile's form. |
| `manifest` | What the ecosystem's resolver reads, derived from the artifacts under the profile. Absent from an OCI tag record, which derives nothing. |

The profile decides which artifacts one record covers, how its `manifest` is
derived, what its log entry is, and where it is served (see
[Ecosystem profiles](#ecosystem-profiles)). Every record is a deterministic
function of the package identity, profile, and artifact bytes. It names no
registry: the log that holds it does. It contains no timestamps, signatures,
provenance, transport locations, or anything that may change after
publication, such as an npm deprecation or a yanked crate.

### Ecosystem profiles

Paths are relative to each ecosystem's registry base (`/npm/`, `/cargo/`,
`/pypi/`, `/oci/`, or a named registry such as `/npm/~<name>/`). Every
profile's record route is immutable, and its evidence is served at the record
route followed by `/evidence`.

| Ecosystem | Profile | One record covers | Record route |
| --- | --- | --- | --- |
| npm | `npm-install-manifest/1` | A version: its primary archive and every platform variant | `<name>/<version>+r<revision>` |
| Cargo | `cargo-index-entry/1` | A crate version: its `.crate` | `api/v1/crates/<name>/<version>/record` |
| Python | `python-core-metadata/1` | One distribution file of a release | `records/<project>/<filename>` |
| OCI | `oci-tag/1` | One setting of a tag to a manifest digest | `-/records/<repository>/<tag>+r<revision>` |

#### npm

The record is the one [split packuments](./0000-split-packuments.md) serves,
with the core fields added; they are additive, so a client that reads it today
ignores them. Its artifact facts are the served ones: `integrity` (the digest of
the uncompressed tar, as for every pnpr archive), `fileCount`, `unpackedSize`,
`requiresBuild`, `bins`, and `variants`. Its `inputs` name the `.tgz` as
published, by the digest npm clients and npm provenance use, beside its
`package.json`: the record binds that digest to the tar's, so anyone can check
the binding by decompressing the `.tgz` and hashing the tar. A version with
[platform variants](./0000-platform-variants.md) has one record covering every
archive: each variant entry carries its own `inputs` (its `.tgz` and its
`package.json`), its tar `integrity`, sizes, `bins`, platforms, and the fields
of its manifest that differ from the version's. A version without a primary has
no top-level `integrity`. The `manifest` is the install manifest that
[publish-time manifest verification](./0000-publish-manifest-verification.md)
derives from `package.json`. The served record's `name` and `version` stay for
the clients that read them today.

```json
{
  "schema": "pnpr-release-record/1",
  "purl": "pkg:npm/%40example/widgets@1.2.0",
  "name": "@example/widgets",
  "version": "1.2.0",
  "revision": 0,
  "interpretation": "npm-install-manifest/1",
  "inputs": {
    "archive": { "kind": "tgz", "integrity": "sha512-<published .tgz>" },
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

#### Cargo

Cargo resolves from the registry's index, not from the crates it downloads, and
does not check one against the other: an index entry can list dependencies the
crate's `Cargo.toml` does not. The record binds them. Its artifact fact is the
`.crate`'s `cksum`, the SHA-256 the index already carries, and its `manifest`
is the index entry Cargo resolves from, derived from the normalized
`Cargo.toml` that `cargo publish` packs into the `.crate`: `deps` (each with
`name`, `req`, `features`, `optional`, `default_features`, `target`, `kind`,
`registry`, and `package`), `features`, `links`, and `rust_version`. The
`yanked` flag changes after publication, so it is not in the record. Crates
have no registry revisions.

pnpr serves the sparse index from the records it holds, so the index entry
Cargo reads is the record's `manifest` with `name`, `vers`, `cksum`, and
`yanked` around it.

#### Python

A Python release is not a fixed set: wheels for new platforms can be uploaded
to it after its first files, though none can be replaced. So one record covers
one distribution file, named by its PURL's `file_name` qualifier, such as
`pkg:pypi/example@1.2.0?file_name=example-1.2.0-py3-none-any.whl`. Its artifact
fact is the file's SHA-256, the digest the Simple API already lists. Its
`manifest` is what a resolver reads from the core metadata: `Requires-Dist`,
`Requires-Python`, and `Provides-Extra`.

- For a wheel, it is derived from the `.dist-info/METADATA` file, whose digest
  is the record's input. That is the file PyPI serves beside the wheel
  ([PEP 658](https://peps.python.org/pep-0658/)), so pnpr serves it too, from
  the record, with the same hash.
- For a source distribution, it is derived from `PKG-INFO` when the metadata
  version is 2.2 or later and none of those fields is declared `Dynamic`
  ([PEP 643](https://peps.python.org/pep-0643/)). Otherwise the metadata is
  known only after a build, and the record says so with `"manifest": null`: it
  binds the file's digest, and a client that requires verified metadata
  refuses to resolve from it.

#### OCI

An image is already content-addressed: its manifest digest identifies it, and
the registry verifies every manifest and blob against its digest. What is not
fixed is the tag: `1.0` can be moved to another image. So the OCI profile
records tag bindings, not derived metadata.

A tag's identity is its `repository` and `tag`, two explicit fields of its
record, since an OCI Package URL names an image by digest rather than a tag.
Each time the tag is set, pnpr writes a record whose `revision` counts the
settings, starting at 0. Its `purl` is the image it points at, in the OCI
type's form: the last segment of the repository as the name, the manifest
digest as the version, and the repository and tag as qualifiers.

```json
{
  "schema": "pnpr-release-record/1",
  "repository": "acme/app",
  "tag": "1.0",
  "revision": 3,
  "interpretation": "oci-tag/1",
  "purl": "pkg:oci/app@sha256%3A<manifest digest>?repository_url=acme/app&tag=1.0"
}
```

Its log entry's identity is the repository, tag, revision, and profile, never
the PURL, which changes with the digest.

Deleting a tag, or deleting the manifest a tag points at, which the OCI
Distribution Specification permits, writes a tombstone revision, a record of
the same profile with no image:

```json
{
  "schema": "pnpr-release-record/1",
  "repository": "acme/app",
  "tag": "1.0",
  "revision": 4,
  "interpretation": "oci-tag/1",
  "purl": null
}
```

Its `purl` is `null`, never omitted, so every tag record has the same fields. A
lookup whose last revision is a tombstone finds no current binding, and setting
the tag again writes the next revision after it. A tombstone is logged, signed,
and served like any other tag record.

For a proxied repository, the log holds what pnpr observed, not the upstream's
history: each time pnpr resolves a tag upstream, it writes a revision if the
digest differs from its last one, and a tombstone if the upstream answers that
the tag does not exist. A deletion pnpr never asked about is not logged.

So a moved tag is a new revision of the same identity, and every move is in the
log; a client that pinned a tag to a digest finds a newer revision by looking
the tag up. Signatures and attestations stay where OCI keeps them, as referrers
of the manifest digest, served at `/v2/<repository>/referrers/<digest>` as
cosign and Notation store them; the record's evidence lists them with pnpr's own
statements.

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

A record is addressed by its identity, such as an npm version and revision, a
crate version, or a Python file name, not by its digest:
[split packuments](./0000-split-packuments.md) chose that so a package
document carries no incompressible digest per version.

### Transparency log

Each registry runs an append-only log of its records, in the formats Go's
checksum database uses, now specified by
[C2SP](https://github.com/C2SP/C2SP): checkpoints signed as
[signed notes](https://github.com/C2SP/C2SP/blob/main/signed-note.md), the log
served as [tiles](https://github.com/C2SP/C2SP/blob/main/tlog-tiles.md), and
[witnesses](https://github.com/C2SP/C2SP/blob/main/tlog-witness.md) that
co-sign checkpoints after checking that the log only grew.

- A log entry is the record's identity (its PURL, or an OCI tag's repository and
  tag), revision, profile, and digest. pnpr sequences it in the SQL metadata
  store, in the same transaction that accepts the version, file, or tag, so a
  record is never accepted without its entry or logged without being accepted.
  The log's tiles and checkpoints are built from the sequenced entries after
  commit, idempotently, as Tessera-style logs integrate them; pnpr serves a
  record only once its entry is integrated and has an inclusion proof.
- A second entry for the same identity, revision, and profile with another
  digest is a forged or rewritten record. An inclusion proof shows only that an
  entry exists, not that no other does, so this is caught as Go's checksum
  database catches it: monitors read the whole witnessed log and report any
  identity logged twice, and a client that locked a record keeps its digest in
  the lockfile and rejects any other. A client that sees a version for the first
  time relies on the monitors. An OCI tag changes its digest only through a new
  revision, so every move is a new entry and none is hidden.
- With each record, pnpr serves an inclusion proof against a recent checkpoint.
  A client verifies the proof and the checkpoint's signatures, the log's and the
  witnesses' it requires, before it accepts the record.
- Clients keep the latest checkpoint they have seen and check that a newer one
  is consistent with it, so a registry cannot rewrite what a client has seen.
  Only witnesses, which co-sign one checkpoint for every client, stop it from
  showing different clients different logs.

Each configured registry has its own log, so `/npm/~internal/` and
`/cargo/~internal/` are separate logs, as they are separate registries. A log
holds hosted versions and the versions pnpr derived records for while proxying
an upstream. It holds digests and identities, not records, so a
private package's log entries reveal its name and versions; a private registry
keeps its log, like its records, behind its access rules.

Each deployment runs its own logs, and a log's verifier key is its identity. A
shared log, as Go's ecosystem shares `sum.golang.org`, would need an operator
every deployment trusts and would expose private names; deployments that want
one can agree on a log later without changing the record or the protocol.

Witnesses are what protect a client against a registry that shows different
clients different logs. A public registry's TUF root names the witnesses its
clients require. A private registry may name none. Its clients then still
detect a rewrite of what each has seen, but not a registry that shows them
different logs: that needs a witness, such as a CI job that co-signs
checkpoints or a second pnpr deployment, which an organization that needs the
guarantee names in its root.

### Statements: in-toto in DSSE

Everything signed about a record is an
[in-toto Statement](https://github.com/in-toto/attestation/blob/main/spec/v1/statement.md)
wrapped in a [DSSE](https://github.com/secure-systems-lab/dsse) envelope, the
format of npm provenance and SLSA. The Statement's subject is the record, by its
digest, named by its PURL, or for an OCI tag record by
`<repository>:<tag>+r<revision>`; its `predicateType` says what is claimed:

- **Acceptance**, `https://pnpm.io/pnpr/acceptance/v1`: the registry accepted
  the binding of this manifest to this archive under this profile. The log
  entry already proves the record existed; an acceptance statement lets a
  record travel with a claim that does not need the log to be reachable.
- **Publication**, `https://pnpm.io/pnpr/publication/v1`: how the version was
  published (see [Publication evidence](#publication-evidence)).

Publisher provenance keeps its own subject and its ecosystem's form, and pnpr
keeps it as it was published:

- npm provenance is a Sigstore bundle whose SLSA Statement names the `.tgz`
  digest. A client accepts it for a record only when that digest is the
  record's `inputs.archive`, or a variant's, which the logged record binds to
  the tar digest it installs.
- Python attestations ([PEP 740](https://peps.python.org/pep-0740/)) are
  Sigstore-signed in-toto Statements naming the distribution file's digest,
  which is the record's artifact digest.
- OCI signatures and attestations are referrers of the manifest digest.

All of a record's statements and bundles are served at its evidence route,
with the inclusion proof:

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

At acceptance, the registry signs a publication statement, in every
ecosystem. Only the registry produces it; publisher-supplied fields with the
same names are stripped. Its predicate is the version's
[publication](./0000-atomic-workspace-publication.md#publications), which
already records the publishing identity, a workload's repository and commit,
and the commit time, plus:

- the principal kind: a user token, an interactive session, or an OIDC
  workload;
- for a workload, the issuer, subject, and the claims the workload binding
  matched, such as repository and workflow;
- whether the version was staged, and the principal that approved it.

It is what a trust policy compares, in every ecosystem: a version published by
a workload of a repository, after earlier versions were, and then one
published with a token is the pattern of a stolen credential whether the
package is an npm package, a crate, or a wheel.

For npm, clients evaluating `trustPolicy` read this statement instead of
inferring trust from `_npmUser`, and pnpr computes the `trust` rank of each
entry in its package document ([split packuments](./0000-split-packuments.md))
from it. npm overloaded `_npmUser` when it added staged publishing, so the
approver replaced the workload and tools lost the trusted-publishing signal
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

A client verifies every signature against the current root only, so history
never depends on a retired key:

- **Statements.** Before a root retires a statement key, pnpr re-signs every
  statement with the new key. That changes evidence, not records, so record
  digests and log entries stay. After a compromise, pnpr re-signs only the
  statements whose records the log shows were integrated before the
  compromise, at a checkpoint the witnesses co-signed.
- **Checkpoints.** A log's checkpoint key signs new checkpoints only. Older
  ones stay valid through consistency: a client checks a new checkpoint,
  signed by the current key, against the one it kept, so an entry proven
  under an old checkpoint is proven under the new one.

- **Public registries.** pnpm ships the TUF roots of the public registries it
  knows, as Go ships the key of `sum.golang.org`.
- **Private registries.** The workspace declares the registry's root, by the
  digest of its first root version, beside the registry's `serverType: pnpr`.
  pnpm does not trust a root fetched from the registry it vouches for unless the
  workspace pinned it.

A registry is identified by its log's name and the digest of its first TUF root
version, such as `registry.example+sha256:<root digest>`, the same digest a
workspace pins. The current checkpoint key, written as Go writes one,
`<name>+<key hash>+<key>`, is verification metadata from the current root, so
rotating it does not change the identity. The name is an identifier, not a
transport location: a mirror or another hostname serving the same log is the
same authority, and a lockfile that records the authority does not change when
the registry moves or rotates a key.
Rebinding an authority to a different log requires an explicit operator action.
Identical archive bytes in two logs do not establish shared ownership.

### Interpretation profiles

A correction to derivation creates new records under a new profile; historical
records are never rehashed. Clients must explicitly support a profile, and an
unsupported profile is a capability error rather than a silent
reinterpretation. Revision numbers do not change when only the interpretation
changes.

A record's identity is its PURL, or an OCI tag's repository and tag, with its
revision and profile, as its log entry is. A profile's record route serves the
record under the profile the artifact was accepted with, and keeps serving it. A
record re-derived under another profile is served at the record route followed
by `/records/<profile>`, such as
`<name>/<version>+r<revision>/records/npm-install-manifest/1` for npm, also
immutable, and the accepted profile's record is at that route too. A profile is
always `<name>/<version>` with each part in `[a-z0-9.-]+`, so it is two path
segments and never needs encoding. A client that requires a profile reads that
route and caches records by the registry's authority, identity, revision, and
profile, never by revision alone: two registries can serve the same PURL with
different records.

### Upstream verification

For proxied packages, upstream metadata is an unverified claim: an npm
package document, a crates.io index entry, or a PyPI page. pnpr fetches each
artifact, derives its record under the profile, and serves the metadata from
it,
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
- A Python source distribution whose metadata is dynamic cannot be verified:
  under the verify mode it resolves only when a wheel of the same release
  provides the metadata, or under an unverified exemption.
- OCI has nothing to derive: pnpr verifies a proxied manifest and its blobs by
  digest, as it does today, and logs the tag binding it observed upstream.
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
verification on by default fails no version in the survey. Cargo and Python
need the same survey, of crates.io index entries against their crates and of
PyPI metadata against its wheels, before their verify mode is released; until
then pnpr records their disagreements without failing on them.

### Client protocol

Extend the `GET /-/pnpr` capability document:

```json
{
  "records": [1],
  "authority": "registry.example+sha256:<first root digest>",
  "manifestProfiles": [
    "npm-install-manifest/1",
    "cargo-index-entry/1",
    "python-core-metadata/1",
    "oci-tag/1"
  ]
}
```

- **Who verifies.** pnpm verifies records for the ecosystems it installs from,
  npm, Cargo, and Python. Native clients, `cargo`, `pip`, and container
  runtimes, read the metadata pnpr serves from records without verifying the
  log; they gain upstream verification from pnpr, and a verifying client or
  monitor can check the same log.
- **Records.** For npm, an entry of pnpr's package document
  (`application/vnd.pnpr.install-v2+json`) that carries its manifest is a
  projection of the record: the same manifest and archive facts, beside the
  version's mutable `trust`, selected `revision`, publish time, and
  deprecation, which are not part of the record. A client that does not verify
  resolves from it, as today. A client that verifies records fetches the
  record route for every version it locks, since only those bytes have the
  logged digest, resolves from the record, and caches it on disk forever,
  keyed by the registry's authority as well as the record's identity. A
  caller-gated record is cached only for the principal that fetched it, as
  [split packuments](./0000-split-packuments.md) caches its records, and a
  cache hit for another principal is a miss. A principal that later loses
  access keeps what it already fetched, as it keeps a tarball already in its
  store.
- **Evidence and proofs.** A verifying client fetches each locked version's
  evidence and checks its inclusion proof. Statements are fetched only when the
  active policy needs them.
- **Log.** Each log's checkpoints and tiles are served at
  `<registry-base>/-/log/`, as C2SP's tiles specify, for monitors and
  witnesses. `<registry-base>/-/log/lookup/<purl>` returns every entry for the
  PURL, each revision and profile, with an inclusion proof against one
  checkpoint, as Go's checksum database serves `/lookup/<module>@<version>`. A
  client selects the entry for the revision and profile it locked. For OCI,
  `<registry-base>/-/log/lookup/<repository>/<tag>` lists a tag's revisions,
  and its last is where the tag points.

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

1. pnpr: add the core fields to the npm record it serves, store the exact bytes
   in the SQL metadata layer, sequence each record's log entry in the
   acceptance transaction, and integrate sequenced entries into a C2SP log.
2. pnpr: sign acceptance and publication statements as in-toto Statements in
   DSSE envelopes, keep Sigstore bundles, and serve evidence with inclusion
   proofs.
3. pnpr: publish TUF root metadata, and serve the log's checkpoints and tiles.
4. pnpm: verify records, proofs, checkpoint consistency, and the statements the
   trust policy needs; ship the roots of known public registries, and read a
   pinned root for a private one.
5. pnpr: derive records for proxied npm versions and enforce the upstream
   modes.
6. pnpr and pnpm: the Cargo and Python profiles, serving the sparse index and
   the Simple API's core metadata from records, after the surveys; and OCI
   tag entries.

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

- [Go's checksum database](https://go.dev/ref/mod#checksum-database), the
  [C2SP](https://github.com/C2SP/C2SP) transparency-log specifications, and
  [Tessera](https://github.com/transparency-dev/tessera), which integrates
  sequenced entries into tiles.
- [in-toto attestations](https://github.com/in-toto/attestation),
  [DSSE](https://github.com/secure-systems-lab/dsse),
  [SLSA provenance](https://slsa.dev/), and
  [npm provenance](https://docs.npmjs.com/generating-provenance-statements) on
  [Sigstore](https://www.sigstore.dev/).
- [TUF](https://theupdateframework.io/) and its adoption for PyPI in
  [PEP 458](https://peps.python.org/pep-0458/).
- [PEP 643](https://peps.python.org/pep-0643/) static source distribution
  metadata, and [PEP 740](https://peps.python.org/pep-0740/) attestations.
- Cargo's [sparse index](https://doc.rust-lang.org/cargo/reference/registry-index.html),
  and [cosign](https://github.com/sigstore/cosign) and
  [Notation](https://notaryproject.dev/) signatures stored as OCI referrers.
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
