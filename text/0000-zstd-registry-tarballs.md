# zstd-compressed registry tarballs

## Summary

A registry entry in the `registries` setting can declare that the registry serves package tarballs compressed with zstd:

```yaml title="pnpm-workspace.yaml"
registries:
  https://pnpr.corp.example/~zstd/:
    scopes: ["@"]
    tarballCompression: zstd
```

For packages resolved from that registry, pnpm downloads `.tar.zst` instead of `.tgz`, and the integrity recorded in `pnpm-lock.yaml` is the integrity of the `.tar.zst`. Registries that do not declare it keep today's behavior.

A zstd registry is its own registry URL, with its own metadata. In that metadata, `dist.tarball` and `dist.integrity` describe the `.tar.zst`. A registry that offers both formats does so at two URLs, so every client reads ordinary npm metadata in the shape it expects.

pnpr can serve `.tgz`, `.tar.zst`, or both, by exposing registries in either format over the same packages. It produces the `.tar.zst` from the `.tgz` at publish time for hosted packages, and on first request for upstream packages. Every `.tar.zst` it produces is stored write-once.

## Motivation

Tarballs are most of what a cold-store install downloads. A `.tgz` is gzip output, so response compression cannot shrink it further: pnpm/pnpm#16585 taught pnpm and pnpr to use zstd for metadata responses, and pnpr deliberately leaves tarballs uncompressed. The saving needs a different artifact, the same tar compressed with zstd.

A prototype measured the gain (details in [pnpm/tasks#135]). The fixture is a project with 30 popular direct dependencies, whose macOS install fetches 1,183 tarballs:

| format | total size | vs gzip |
|---|---|---|
| gzip, as published | 228.9 MB | |
| zstd level 3 | 198.8 MB | -13% |
| zstd level 19 | 143.5 MB | -37% |

The install was frozen-lockfile with a cold store and a warm metadata cache, on an M1 Max. The figures are medians of 5 runs, comparing gzip with zstd level 19:

| link | wall time | user CPU |
|---|---|---|
| loopback | 48.5 -> 49.8 s, no difference | 3.29 -> 2.70 s |
| 100 Mbit/s, 40 ms RTT | 46.3 -> 49.4 s, inconclusive | 3.49 -> 2.78 s |
| 25 Mbit/s, 60 ms RTT | 84.8 -> 56.3 s, -33% | 4.59 -> 3.26 s |

- **Slow links:** zstd shortens the install by up to the time saved on the download.
- **Fast links:** the install is bound by creating files in the store. On this machine that floor is about 45 s, almost all system time, and a faster download hides behind it.
- **CPU:** decoding takes 18-30% less CPU at every link speed.

The installs that benefit are cold-store installs over a constrained link: CI without a restored store, Docker builds, developer machines on home or office connections, and remote runners pulling from a registry in another region. Where the crossover lies on Linux, which creates files faster, is not measured yet.

[pnpm/tasks#135]: https://github.com/pnpm/tasks/issues/135

## Detailed Explanation

### The setting

`tarballCompression` is a field of an npm entry in the [`registries`](https://pnpm.io/registries) setting.

| value | meaning |
|---|---|
| `gzip` (default) | Today's behavior: pnpm downloads `.tgz`, and the lockfile pins its integrity. |
| `zstd` | pnpm downloads `.tar.zst`, and the lockfile pins its integrity. |

Like `serverType` and `supportsTimeField`, the field describes the server and the lockfile depends on it. pnpm therefore reads it only from `pnpm-workspace.yaml` and refuses it in the global `config.yaml`. It is refused for Cargo and PyPI entries.

The declaration is per registry, not per package. A registry that declares `zstd` serves a `.tar.zst` for every version it serves, so a lockfile entry never needs to say which format it pins.

### Registry protocol

**Tarball URL.** The `.tar.zst` of a version lives at the canonical npm-layout path with the `.tar.zst` extension:

```text
<registry>/<name>/-/<scopeless-name>-<version>.tar.zst
```

pnpm reconstructs it the same way it reconstructs `.tgz` URLs today, with the same `serverType` layout rules. A lockfile then carries no host-specific URL for these packages.

**Metadata.** A zstd registry serves its own metadata. In each version's `dist`, the standard fields describe the `.tar.zst`:

```jsonc
{
  "dist": {
    "tarball": "https://pnpr.corp.example/~zstd/lodash/-/lodash-4.17.21.tar.zst",
    "integrity": "sha512-<tar.zst digest>",
    "unpackedSize": 1412415
  }
}
```

- **Unchanged fields:** fields that describe the unpacked package, such as `unpackedSize` and `fileCount`, stay as they are, because both formats hold the same tar.
- **Replaced fields:** `shasum` is the SHA-1 of the `.tgz`, so it is left out. The upstream's `signatures` sign `name@version:integrity` with the `.tgz` integrity, so they are replaced by the zstd registry's own signatures over the `.tar.zst` integrity (see [Registry signatures](#registry-signatures)).
- **`attestations`:** kept, because they describe the published package. Their subject is the `.tgz` digest, as described under [Trust](#trust).

A registry that offers both formats serves them at two URLs, for example `https://pnpr.corp.example/` for `.tgz` and `https://pnpr.corp.example/~zstd/` for `.tar.zst`. Each URL has one document per package, so no response varies by request header, and a CDN caches each as it caches metadata today. The URL a project keys in `registries` decides which format it gets.

`dist.integrity` may be missing from a zstd registry's metadata. A registry that transcodes upstream packages lazily does not know the integrity of a `.tar.zst` it has not produced yet. When a resolved version has no `dist.integrity`:

- pnpm downloads the `.tar.zst` from the canonical URL. The registry produces it on that request.
- pnpm hashes the received bytes and records that integrity in the lockfile.
- The download is not wasted, because an install needs the tarball anyway. A `--lockfile-only` resolution pays for downloads it would otherwise skip, and only for versions the registry has not transcoded yet.

When `dist.integrity` is present, pnpm verifies the downloaded bytes against it, as it does today.

The `tarballCompression` declaration is still needed. Metadata alone does not tell pnpm how to rebuild an omitted tarball URL, and the lockfile records the declaration (see [Lockfile](#lockfile)). Like `serverType`, the format is declared, never inferred from the metadata.

**Immutability.** A registry must serve the same `.tar.zst` bytes for a version forever, once it has served them or advertised their integrity. Lockfiles pin those bytes. A registry that produced different bytes on a later request, for example after upgrading its zstd library or changing the compression level, would break every lockfile that pinned the old ones. Regenerating on demand is therefore not allowed. The first produced artifact is stored and served from then on.

**Frame limits.** A `.tar.zst` is a single zstd frame with a window of at most 8 MiB, the limit RFC 9659 sets for HTTP content coding. That covers every standard level up to 19 without `--long`. pnpm rejects larger windows, so the memory each concurrent extraction can claim stays bounded. The existing ceiling on the unpacked size applies unchanged.

### Lockfile

An entry resolved from a `zstd` registry records the integrity of the `.tar.zst` in the existing field:

```yaml
packages:
  lodash@4.17.21:
    resolution:
      integrity: sha512-<tar.zst digest>
```

The lockfile records which registries were declared `zstd`, in `settings`. The key appears only when some registry declares `zstd`, so every other lockfile is unchanged:

```yaml
settings:
  autoInstallPeers: true
  excludeLinksFromLockfile: false
  tarballCompression:
    https://pnpr.corp.example/~zstd/: zstd
```

Freshness checks treat a difference between this map and the configuration like any other changed setting:

- **Frozen install:** pnpm fails with an error that names the registry and the declaration. Without this check, a removed declaration would surface as integrity mismatches on every package.
- **Other installs:** pnpm re-resolves the affected packages and writes their new integrities. Declaring or removing `zstd` produces a one-time lockfile diff, as declaring a `serverType` does.

### Fetching and verification

pnpm downloads the `.tar.zst` and verifies its bytes against the locked integrity before it writes any file to the store. It then decodes zstd where it decodes gzip today, both on the streaming path during the download and on the buffered path. A mismatch or a missing artifact fails the install. pnpm never falls back to the `.tgz`, because the lockfile pins no hash it could verify the `.tgz` against.

The content-addressed store holds files, not tarballs, so the same package installed from `.tgz` in one project and from `.tar.zst` in another shares its file content in the store. The store index rows are keyed by the integrity, so they differ.

### Trust

The `.tgz` integrity is the publisher's. The `.tar.zst` integrity is the registry's: the registry decompresses the publisher's tarball and compresses it again. A project that declares `zstd` therefore trusts that registry to transcode faithfully, in addition to the trust it already places in it to serve the metadata the lockfile was resolved from. That fits a registry the organization operates, such as its own pnpr. It is why the declaration is per registry, and why pnpm never applies it to a registry the user did not declare.

Two consequences:

- **Registry signatures.** The upstream's signatures cover the `.tgz` integrity. A zstd registry signs the `.tar.zst` integrities with its own key instead, described below.
- **Provenance.** Sigstore provenance names the `.tgz` digest as its subject. The link from the attested `.tgz` to the installed `.tar.zst` is the registry's claim.

pnpr, in turn, must verify an upstream `.tgz` against its upstream integrity before it transcodes it.

#### Registry signatures

npm registry signatures sign `name@version:integrity` with the registry's key, published at `/-/npm/v1/keys`, and `pnpm audit signatures` verifies them. The registry that produces a `.tar.zst` is the party that can vouch for it, so a zstd registry signs its own artifacts:

- Each version's `dist.signatures` holds signatures over `name@version:<.tar.zst integrity>`, made with the zstd registry's key.
- The registry serves that key at its own `/-/npm/v1/keys`, for example `/~zstd/-/npm/v1/keys`.
- `pnpm audit signatures` against the zstd registry then works unchanged.

What a signature vouches for depends on where the package lives:

- **Hosted packages:** the registry accepted the publish and produced the `.tar.zst` from the published `.tgz`. Its signature covers both steps.
- **Upstream packages:** the registry vouches for its own transcoding of the upstream `.tgz`, which it verified against the upstream integrity. That is weaker than the upstream's signature over the published `.tgz`.

A version the registry has not transcoded yet has no `.tar.zst` integrity, so its metadata carries no signature until it is transcoded.

### pnpr

Any pnpr registry can serve its packages as zstd with a `tarballs` field. The value is `tgz` (the default) or `zstd`. Formats are exposed as registries, so a deployment offers zstd next to `.tgz` by adding a router over the same sources:

```yaml title="pnpr.yaml"
registries:
  local:
    type: hosted
    packages:
      '@acme/*':
        publish: $authenticated
  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true
  main:
    type: router
    sources: [local, npmjs]
  zstd:
    type: router
    sources: [local, npmjs]
    tarballs: zstd

defaultRegistry: main
```

Here `https://pnpr.corp.example/` serves `.tgz` and `https://pnpr.corp.example/~zstd/` serves `.tar.zst`. The deployment shapes:

| deployment | exposed registries |
|---|---|
| `.tgz` only | Today's configuration. |
| both | A `.tgz` registry and a zstd registry over the same sources, as above. npm, Yarn, and projects without the declaration use the first. Projects that declare `zstd` use the second. |
| zstd only | Only zstd registries, for example `defaultRegistry: zstd`. Only pnpm with the declaration can install from it. |

A `.tar.zst` belongs to the hosted or upstream registry that holds the package, not to the router that serves it. Two zstd registries over the same source serve the same bytes. Publishing still accepts a `.tgz`, and pnpr keeps it, because the `.tgz` is what the publisher's integrity and attestations cover.

**When pnpr transcodes:**

- **Hosted registries:** at publish. The `.tar.zst` and its integrity exist before the first client resolves the version.
- **Upstream registries:** on the first `.tar.zst` request for a version. pnpr fetches and verifies the `.tgz`, transcodes it, stores the result, and only then serves it. Concurrent first requests for one version must produce one artifact.
- **Storage:** the stored `.tar.zst` lives in durable storage, not in the disposable upstream cache that pnpr may evict. Evicting it would let a later request produce different bytes.

pnpr compresses at level 19. Transcoding happens once per version, so a high level costs nothing per request.

## Rationale and Alternatives

**One metadata document with both artifacts**, for example a `dist.zstd` object next to the `.tgz` fields. One document would serve every client. But every version would then describe two artifacts, the standard `dist` fields would describe the one a declaring client ignores, and every later format would add another sibling field.

**Negotiating metadata by request header**, for example a parameter on `Accept`. This keeps one URL, but every metadata response would vary on that header, and caches would have to store a variant per format. A separate registry URL gives the same documents without negotiation and fits how `registries` is already keyed.

**`Content-Encoding: zstd` on the `.tgz` URL.** This needs no lockfile change, but saves nothing, because gzip output does not compress further. Serving `zstd(tar)` under `Content-Encoding` instead would violate HTTP semantics, since the decoded body would no longer be the `.tgz` the URL names.

**A per-entry marker in the lockfile** (`resolution.compression: zstd`). This would let one registry serve both formats per version and survive configuration changes. It adds a field to every entry, and the per-registry declaration already carries the information, because a `zstd` registry serves zstd for every version. The `settings` record gives frozen installs the same protection at one line per registry.

**Publisher-produced `.tar.zst`.** `pnpm publish` would upload both artifacts and provenance would cover both. That moves the trust back to the publisher, but it needs registry support for a second artifact per version. It only helps packages published after adoption, which leaves out almost every package an install fetches today. Registry transcoding works for the whole graph. Publisher-produced artifacts can later replace the transcoded ones in the same zstd registry.

**Integrity-addressed URLs** (`-/tarballs/sha512/<digest>`, from `0000-integrity-addressed-registry-tarballs.md`). They make a `.tar.zst` immutable by construction and CDN-friendly. They cannot address an artifact before its digest exists, so they cannot carry lazy upstream transcoding, which needs a name-and-version URL. A registry implementing both RFCs can advertise the digest route for transcoded versions.

**Brotli.** On npm metadata it compresses about as well as zstd. Its decoder is slower, and pnpm and pnpr already ship zstd since pnpm/pnpm#16585.

## Implementation

pnpm v12 only. pnpm v11 is maintenance-only and is not changed.

**pnpm:**

- `config`: the `tarballCompression` field on npm registry entries, with validation and the workspace-only rule.
- `resolving-npm-resolver`, `registry`: accept a missing `dist.integrity` from a `zstd` registry. Rebuild omitted tarball URLs with the `.tar.zst` extension.
- `lockfile`: the `settings.tarballCompression` record and its freshness check.
- `tarball`:
  - Decode zstd at every site that decodes gzip today: the eager decode, the buffered streaming decode, and the decode fed from the download channel.
  - Accept the zstd magic where the download path rejects bodies that do not start with the gzip magic.
  - Enforce the 8 MiB window limit.
  - When the metadata carries no `dist.integrity`, record the integrity computed from the received bytes.
- Store index and global virtual store: confirm that entries keyed by a `.tar.zst` integrity reuse file content and that slot hashes stay correct.
- Docs: `pnpm/docs/registries.md`.

The prototype behind the measurements covers the decoding sites in `pnpm/crates/tarball`. It skipped integrity verification, which this RFC defines.

**pnpr:**

- Config: `tarballs` on every registry type.
- Publish path: transcode and store the `.tar.zst`.
- Upstream path: transcode on first request, with single-flight deduplication, and store the result write-once in durable storage.
- Metadata: in zstd registries, rewrite `dist.tarball` and `dist.integrity` to the `.tar.zst`, drop `shasum`, and replace `signatures` with the registry's own. Leave `dist.integrity` and `signatures` out for versions not transcoded yet.
- Signing: a registry signing key, served at `/-/npm/v1/keys` for the registries it signs. pnpr has no key of its own today; it proxies the upstream's keys.
- Routes: serve `.tar.zst` under zstd registries.
- Docs: `pnpr/docs`.

## Prior Art

- **OCI images:** the image spec defines zstd layers (`application/vnd.oci.image.layer.v1.tar+zstd`) next to gzip layers. Each blob is addressed by the digest of its compressed bytes, and the same filesystem content can exist in both forms.
- **conda:** the `.conda` package format replaced `.tar.bz2` and stores package contents compressed with zstd.
- **Linux distributions:** Arch Linux and Fedora moved their binary packages to zstd, citing much faster decompression.

## Unresolved Questions and Bikeshedding

- **Names:** `tarballCompression` and `gzip`/`zstd`, or `tarballFormat` with `tgz`/`tar.zst`. In pnpr: `tarballs`. The extension: `.tar.zst`, the common convention, or `.tzst`, the counterpart of `.tgz`.
- **Signing upstream packages:** whether a zstd registry should sign its transcodings of upstream packages, or only hosted ones. Unsigned upstream versions would fail `pnpm audit signatures`, while signed ones carry the weaker guarantee described under [Registry signatures](#registry-signatures).
- **Level and parameters:** whether the RFC fixes level 19, or only the window limit and leaves the level to the registry.
- **Dictionaries:** whether a zstd dictionary trained on npm tarballs is worth a later extension. It mainly helps small tarballs, which are most tarballs by count.
- **Lockfile-only resolution:** whether pnpr should transcode the latest versions of a package eagerly when it caches its metadata, so `--lockfile-only` resolutions rarely download.
- **Linux crossover:** to be measured before ratification, because it decides how many installs see the gain.
