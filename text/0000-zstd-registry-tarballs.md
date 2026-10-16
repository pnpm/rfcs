# zstd tarballs through tar integrity

## Summary

The lockfile can pin the integrity of a package's uncompressed tar instead of its `.tgz`:

```yaml
packages:
  lodash@4.17.21:
    resolution:
      tarIntegrity: sha512-<tar digest>
```

The tar is the gzip-decoded content of the published `.tgz`, so its hash is a deterministic function of the publisher's artifact. Compression becomes a transport detail. A registry that declares the `.tar` endpoint serves the tar with standard HTTP content coding, zstd for clients that accept it:

```yaml title="pnpm-workspace.yaml"
registries:
  https://pnpr.corp.example/:
    scopes: ["@"]
    tarballFormat: tar
```

```text
GET /lodash/-/lodash-4.17.21.tar
Accept-Encoding: zstd, gzip

200 OK
Content-Type: application/x-tar
Content-Encoding: zstd
```

The same lockfile entry installs from any registry. pnpm fetches the `.tar` from a registry that declares it, and fetches the `.tgz` and decompresses it everywhere else. Either way it verifies the tar against `tarIntegrity`. pnpr serves the `.tar` endpoint for any registry that enables it.

## Motivation

Tarballs are most of what a cold-store install downloads. A `.tgz` is gzip output, and npm's `dist.integrity` hashes those compressed bytes. That binds the compression format into every lockfile, registry signature, and provenance attestation, so no registry can serve a package with better compression without breaking every hash that names it. pnpm/pnpm#16585 made pnpm and pnpr use zstd for metadata responses, but tarballs cannot follow while the hash covers the gzip output.

A prototype measured what zstd would gain (details in [pnpm/tasks#135]). The fixture is a project with 30 popular direct dependencies, whose macOS install fetches 1,183 tarballs:

| encoding | total size | vs gzip |
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

The prototype served `.tar.zst` files. They carry the same bytes on the wire as a level-19 `Content-Encoding: zstd` response for the same tar.

The installs that benefit are cold-store installs over a constrained link: CI without a restored store, Docker builds, developer machines on home or office connections, and remote runners pulling from a registry in another region. Where the crossover lies on Linux, which creates files faster, is not measured yet.

[pnpm/tasks#135]: https://github.com/pnpm/tasks/issues/135

## Detailed Explanation

### The tar and its integrity

The **tar** of a version is the byte sequence a gzip decoder produces from the whole published `.tgz`. A `.tgz` made of several gzip members decodes to their concatenation. A `.tgz` with data that is not a gzip member has no tar under this RFC. Such a version keeps `integrity`, and the registry serves no `.tar` for it.

**`tarIntegrity`** is the Subresource Integrity value of the tar, using sha512.

Because the tar is fully determined by the `.tgz`:

- anyone holding the `.tgz` can compute `tarIntegrity`, including a client that verified the `.tgz` against the publisher's `integrity`;
- a registry signature or provenance attestation over the `.tgz` covers the tar by derivation;
- the compressed bytes on the wire can differ between registries, requests, and compression levels without affecting the hash.

### Lockfile

A package entry pins either `integrity` (the `.tgz`, as today) or `tarIntegrity` (the tar):

```yaml
packages:
  lodash@4.17.21:
    resolution:
      tarIntegrity: sha512-<tar digest>
  left-pad@1.3.0:
    resolution:
      integrity: sha512-<tgz digest>
```

- **When pnpm writes it:** pnpm records `tarIntegrity` for packages resolved from a registry that declares `tarballFormat: tar`. Other entries keep `integrity`.
- **What it promises:** an entry with `tarIntegrity` is installable from any registry that serves the version, not just the declaring one (see [Fetching](#fetching)).
- **Configuration changes:** declaring or removing `tarballFormat: tar` never makes an existing entry invalid. Both kinds of entry install from either kind of registry. pnpm may convert entries when it re-resolves them, but no install is forced to rewrite the lockfile.
- **Older pnpm versions:** an entry without `integrity` could be read by an older pnpm as an entry with no integrity at all. A lockfile that contains a `tarIntegrity` entry therefore carries a lockfile version that older pnpm versions reject.

### The setting

`tarballFormat` is a field of an npm entry in the [`registries`](https://pnpm.io/registries) setting.

| value | meaning |
|---|---|
| `tgz` (default) | Today's behavior: pnpm downloads `.tgz`, and new lockfile entries pin `integrity`. |
| `tar` | The registry serves the `.tar` endpoint. pnpm downloads `.tar` with `Accept-Encoding: zstd, gzip`, and new lockfile entries pin `tarIntegrity`. |

Like `serverType` and `supportsTimeField`, the field describes the server, and pnpm reads it only from `pnpm-workspace.yaml`. It is refused for Cargo and PyPI entries.

### Registry protocol

**Tarball URL.** The tar of a version lives at the canonical npm-layout path with the `.tar` extension:

```text
<registry>/<name>/-/<scopeless-name>-<version>.tar
```

pnpm reconstructs it the way it reconstructs `.tgz` URLs today, with the same `serverType` layout rules, so lockfiles keep omitting tarball URLs.

**Response.** The response body, after content decoding, is the tar:

- `Content-Type: application/x-tar`;
- `Content-Encoding` chosen from the request's `Accept-Encoding`: `zstd`, `gzip`, or none;
- `Vary: Accept-Encoding`, so caches keep one stored variant per encoding.

This is ordinary HTTP content coding, so CDNs and proxies can cache and serve it without knowing about this RFC. A zstd response follows RFC 9659: its window is at most 8 MiB, which every level up to 19 respects without `--long`. pnpm rejects larger windows, so the memory each concurrent extraction can claim stays bounded. The existing ceiling on the unpacked size applies unchanged.

**Metadata.** A version's `dist` may carry the tar's integrity next to the existing fields:

```jsonc
{
  "dist": {
    "tarball": "https://pnpr.corp.example/lodash/-/lodash-4.17.21.tgz",
    "integrity": "sha512-<tgz digest>",
    "tarIntegrity": "sha512-<tar digest>",
    "signatures": [ /* unchanged, over the .tgz integrity */ ]
  }
}
```

`tarIntegrity` is a second hash of the same artifact, as `shasum` is today. Every other field keeps its meaning, so npm-compatible clients are unaffected, and registry signatures and attestations stay valid.

`dist.tarIntegrity` is optional, even on a registry that declares `tar`. A registry learns the tar's hash only once it holds the `.tgz`, so a proxy may not know it for upstream versions it has not fetched yet.

### Resolution

When pnpm resolves a version from a `tar` registry, it needs the version's `tarIntegrity`:

- **Present in metadata:** pnpm uses it, trusting the registry's metadata exactly as it trusts `dist.integrity` today.
- **Missing from metadata:** pnpm downloads the `.tgz`, verifies it against the hash it checks a `.tgz` against today (`dist.integrity`, or `dist.shasum` for versions that have no `integrity`), and computes `tarIntegrity` from the decoded bytes. The lockfile entry is then derived from the publisher's hash, with no claim from the registry beyond what it makes today. An install needs the tarball anyway. A `--lockfile-only` resolution pays for downloads it would otherwise skip, and only for versions the registry does not describe yet.
- **Verification policies:** when a policy that vouches for the publisher's artifact is enabled, such as `trustPolicy`, pnpm derives `tarIntegrity` from the verified `.tgz` even when the metadata carries one. It records the result only if the two agree, and fails resolution otherwise.

### Fetching

For an entry with `tarIntegrity`:

- **From a `tar` registry:** pnpm requests the `.tar` URL with `Accept-Encoding: zstd, gzip`. The HTTP client decodes the content coding. pnpm hashes the tar as it arrives and extracts it as a plain tar stream.
- **From any other registry:** pnpm requests the `.tgz`, decodes the gzip, hashes the decoded tar, and extracts it.

In both cases pnpm verifies the tar against `tarIntegrity` the way it verifies `integrity` today, and a mismatch fails the install. An entry with `integrity` is fetched exactly as today.

A decoded body has no reliable length, so pnpm takes `dist.unpackedSize` or the size limit into account when it decides whether to extract while downloading.

The content-addressed store holds files, so a package fetched as `.tgz` in one project and as `.tar` in another shares its file content. Store index rows keyed by the integrity value differ between the two kinds of entry.

### pnpr

Any pnpr registry enables the `.tar` endpoint and `dist.tarIntegrity` with one field:

```yaml title="pnpr.yaml"
registries:
  local:
    type: hosted
    tar: true
  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    tar: true
  main:
    type: router
    sources: [local, npmjs]
```

A router serves the `.tar` endpoint for the packages of the sources that enable it.

- **Tar integrity:** pnpr computes `tarIntegrity` when it first holds a `.tgz`: at publish for hosted packages, and on first fetch for upstream ones. It verifies an upstream `.tgz` against its upstream integrity first. The value is deterministic, so it can live in the disposable cache and be recomputed.
- **Serving:** pnpr serves the `.tar` from the stored `.tgz`, encoded for the request. To avoid compressing large tarballs per request, it may store encoded variants, such as a level-19 zstd encoding produced in the background, and serve a fast level until it exists. Variants can be evicted and regenerated at any time, because no hash depends on their bytes.
- **No new signing:** the `.tgz`, its `integrity`, and its signatures are unchanged, so pnpr needs no signing key for this RFC.

### Trust

The trust model does not change:

- **Hosted and upstream metadata:** a client that takes `dist.tarIntegrity` from metadata trusts the registry as it trusts `dist.integrity` today.
- **Verified derivation:** a client that derives `tarIntegrity` from a `.tgz` it verified relies only on the publisher's hash.
- **Signatures and provenance:** they cover the `.tgz`, and through the derivation, the tar.

## Rationale and Alternatives

**Registry-produced `.tar.zst` with its own integrity.** An earlier draft of this RFC had registries transcode each `.tgz` into a `.tar.zst` and lockfiles pin the hash of the compressed bytes. That makes the compressed bytes part of the artifact's identity, which costs:

- **write-once storage,** because a later transcoding with another library version or level produces different bytes and breaks every lockfile that pinned the first;
- **a trust shift,** because the hash comes from the registry's transcoding, and registry signatures and provenance no longer cover what the lockfile pins;
- **lockfile portability,** because the pinned bytes exist only on the registry that produced them;
- **a slow first request,** because a level chosen once is fixed forever, so the first request waits for the highest level;
- **a separate registry URL** for zstd metadata, to keep `dist` describing the served artifact.

Hashing the tar removes all of these.

**Per-file hashes,** as JSR publishes for its packages. A manifest of file hashes would also make compression a transport detail and would let clients skip files they already store. It needs a new manifest endpoint and a different store ingestion path. The tar hash keeps today's one-artifact-per-version model and could later be complemented by a manifest.

**`Content-Encoding: zstd` on the `.tgz` URL.** This saves nothing, because gzip output does not compress further. Sending `zstd(tar)` under that URL would violate HTTP semantics, since the decoded body would not be the `.tgz` the URL names.

**Publisher-produced `.tar.zst`.** This needs registry support for a second artifact and helps only packages published after adoption. With the tar hash, any registry can serve any already-published version compressed with zstd.

**Brotli.** On npm metadata it compresses about as well as zstd. Its decoder is slower, and pnpm and pnpr already ship zstd since pnpm/pnpm#16585. The `.tar` endpoint can still offer it later through content negotiation, without a lockfile change.

## Implementation

pnpm v12 only. pnpm v11 is maintenance-only and is not changed.

**pnpm:**

- `config`: the `tarballFormat` field on npm registry entries, with validation and the workspace-only rule.
- `registry`, `resolving-npm-resolver`: parse `dist.tarIntegrity`. Derive it from a verified `.tgz` when it is missing or when a verification policy asks for the chain. Rebuild `.tar` URLs.
- `lockfile`: the `tarIntegrity` resolution field and the lockfile version gate.
- `tarball`:
  - Extract a plain tar stream on the `.tar` path, where the HTTP client has already decoded the content coding.
  - Hash the decoded tar on both the `.tar` and `.tgz` paths when the entry pins `tarIntegrity`.
  - Enforce the 8 MiB zstd window limit.
  - Choose streaming extraction without a reliable body length.
- Store index and global virtual store: confirm that entries keyed by `tarIntegrity` reuse file content and that slot hashes stay correct.
- Docs: `pnpm/docs/registries.md`.

**pnpr:**

- Config: `tar` per registry.
- Publish and upstream fetch: compute `tarIntegrity`, after verifying an upstream `.tgz`.
- Metadata: add `dist.tarIntegrity` for versions whose `.tgz` pnpr holds.
- Routes: serve `.tar` with content negotiation. Optionally store and evict encoded variants.
- Docs: `pnpr/docs`.

## Prior Art

- **JSR:** publishes per-file checksums in a version manifest and serves files individually. Its npm compatibility layer generates `.tgz` for npm clients.
- **OCI images:** layers are addressed by the digest of their compressed bytes, and the config separately records the digest of each uncompressed layer (`diff_ids`). The same filesystem content can exist as gzip and zstd layers.
- **HTTP content coding:** RFC 8878 registers `zstd`, and RFC 9659 fixes its window for HTTP.

## Unresolved Questions and Bikeshedding

- **Names:** `tarIntegrity`, `tarballFormat: tar`, the `.tar` path, and pnpr's `tar: true`.
- **All registries:** whether pnpm should eventually record `tarIntegrity` for every registry, so every lockfile is independent of compression. A registry that never serves `.tar` would still work, through the `.tgz` path.
- **Lockfile version:** the exact gate, and whether a lockfile without `tarIntegrity` entries keeps today's version.
- **Upstream hashes:** whether pnpr should fetch the `.tgz` of the latest versions in the background, so `--lockfile-only` resolutions rarely download.
- **Dictionaries:** whether a zstd dictionary trained on npm tarballs is worth a later content coding. It mainly helps small tarballs, which are most tarballs by count.
- **Linux crossover:** to be measured before ratification, because it decides how many installs see the gain.
