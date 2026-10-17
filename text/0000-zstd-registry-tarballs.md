# zstd tarballs through tar integrity

## Summary

A registry can serve a package's tarball as an uncompressed tar with standard HTTP content coding, zstd for clients that accept it. Its metadata then describes the tar, in the same fields as today:

```jsonc
"dist": { "integrity": "sha512-<tar digest>" }
```

The tar is the gzip-decoded content of the published `.tgz`, so its hash is a deterministic function of the publisher's artifact. Compression becomes a transport detail: the registry may encode the tar at any level, re-encode it, or let a CDN choose, because no hash depends on the encoded bytes.

A project opts in by using the tar registry's URL. The only declaration is pnpr's URL layout, so that lockfiles can keep omitting tarball URLs:

```yaml title="pnpm-workspace.yaml"
registries:
  https://pnpr.corp.example/~zstd/:
    scopes: ["@"]
    serverType: pnpr
```

```text
GET /~zstd/lodash@4.17.21
Accept-Encoding: zstd, gzip

200 OK
Content-Type: application/x-tar
Content-Encoding: zstd
```

The lockfile keeps its `integrity` field; for packages from a tar registry it pins the tar's hash. A registry offers both formats at two URLs. pnpr exposes a tar registry over any of its sources with one setting.

## Motivation

Tarballs are most of what a cold-store install downloads. A `.tgz` is gzip output, and npm's `dist.integrity` hashes those compressed bytes. That binds the compression format into every lockfile, registry signature, and provenance attestation, so no registry can serve a package with better compression without breaking every hash that names it. pnpm/pnpm#16585 made pnpm and pnpr use zstd for metadata responses, but tarballs cannot follow while the hash covers the gzip output.

A prototype measured what zstd would gain (full tables in [pnpm/tasks#135]). The fixture is a project with 30 popular direct dependencies; a cold-store install fetches about 1,180 tarballs, 229-236 MB as published, depending on the platform's optional dependencies. Re-compressed with zstd, the same tars are 12-13% smaller at level 3 and 36-37% smaller at level 19.

Each run was a frozen-lockfile install with a cold store and a warm metadata cache, so tarballs were the only network cost, behind an emulated link. Medians of 5 runs, gzip against zstd level 19:

| machine | install floor on loopback | zstd clearly faster below | at 25 Mbit/s, 60 ms RTT |
|---|---|---|---|
| Linux, Ryzen 9 9950X3D, btrfs on NVMe | 1.3 s | 2-4 Gbit/s | 75.7 -> 48.4 s (-36%) |
| Windows 10, i7-3770, SATA SSD, Defender exclusion | 21 s | 50-100 Mbit/s | 80.0 -> 51.6 s (-36%) |
| macOS, M1 Max | 45 s | 25-100 Mbit/s | 84.8 -> 56.3 s (-33%) |

- **Below the crossover** the install is bandwidth-bound and the gain equals the size reduction: 28-36% on Linux from 1 Gbit/s down, 32-36% on Windows at 50 and 25 Mbit/s. Level 3 gives 12-14% on the same links.
- **Above it** the variants are indistinguishable. The install is bound by creating files in the store, and the download hides behind that work.
- **The crossover is set by the filesystem.** Where creating 48,000 store files takes 1.3 s, zstd helps on any link slower than a few Gbit/s. Where it takes 45 s, only links under about 100 Mbit/s see a difference.
- **Antivirus dominates everything.** With Windows Defender's default real-time scanning, the same install took about 325 s, and zstd made no difference even at 20 Mbit/s.
- **CPU:** decoding takes less user CPU with zstd on every platform (18-30% on macOS), but it never shows in wall time.

The prototype served `.tar.zst` files. They carry the same bytes on the wire as a level-19 `Content-Encoding: zstd` response for the same tar.

The installs that benefit are cold-store installs over a constrained link: CI without a restored store, Docker builds, developer machines on home or office connections, and remote runners pulling from a registry in another region. On Linux that covers most links in use.

[pnpm/tasks#135]: https://github.com/pnpm/tasks/issues/135

## Detailed Explanation

### The tar and its integrity

The **tar** of a version is the byte sequence a gzip decoder produces from the whole published `.tgz`. A `.tgz` made of several gzip members decodes to their concatenation. A `.tgz` with data that is not a gzip member has no tar under this RFC, and a tar registry serves no such version.

The tar's **integrity** is its Subresource Integrity value, using sha512.

Because the tar is fully determined by the `.tgz`:

- anyone holding the `.tgz` can compute the tar's hash, including a client that verified the `.tgz` against the publisher's hash;
- a registry signature or provenance attestation over the `.tgz` covers the tar by derivation;
- the encoded bytes on the wire can differ between registries, requests, and compression levels without affecting the hash.

### One meaning for `integrity`

In metadata and in the lockfile, `integrity` keeps the meaning it has for every tarball source today: the Subresource Integrity value of the resource at the package's tarball URL. SRI is defined over the resource after HTTP content coding is removed, so for a `.tar` URL served with `Content-Encoding: zstd` that value is the hash of the tar. The field is not redefined; only the derived URL changes, as it already does with `serverType`.

A misread can only fail. Whether pnpm treats a value as a `.tgz` hash or a tar hash, the bytes it verifies either match or they do not, and no other content matches without a SHA-512 preimage.

### Lockfile

An entry for a package from a tar registry pins the tar's hash in the existing field, and the tarball URL is omitted when pnpm can rebuild it:

```yaml
packages:
  lodash@4.17.21:
    resolution:
      integrity: sha512-<tar digest>
```

- **Which hash an entry holds** follows from the registry the package resolved from, as the tarball URL does. The lockfile is not annotated.
- **Switching a registry's format** makes its entries' hashes stale, like any other change in what a registry serves. pnpm reports `ERR_PNPM_TARBALL_INTEGRITY`, and `pnpm install --update-checksums` refreshes the entries, as it does today when a registry rewrites its tarballs.
- **Older pnpm versions** reject the unknown `serverType: pnpr` value when they read the configuration, so they never reach a tar hash.

### The layout declaration

No setting selects the tar format. A project opts in by keying a tar registry's URL in `registries`, and pnpm learns everything else from the registry's responses: the encoding from `Content-Encoding`, the format from the first bytes of the decoded body, and the hash from the metadata.

What a frozen install cannot learn from a response is the tarball URL it has to rebuild, because the lockfile omits it. That is what the existing `serverType` field declares, and this RFC adds the value `pnpr`:

| `serverType` | tarball URL layout |
|---|---|
| *undeclared* | `<registry>/<name>/-/<scopeless-name>-<version>.tgz`, strict |
| `npm` | the same, with the npm registry's leniencies |
| `artifactory` | the same, with the scope repeated in the file name |
| `pnpr` | `<registry>/<name>@<version>` |

`pnpr` describes pnpr's layout, not the tar format: pnpr serves that path on every registry it exposes, `.tgz` and tar alike, and pnpm treats whatever comes back by its content. The existing rules for `serverType` apply: it is read only from `pnpm-workspace.yaml`, declared rather than inferred, and refused for Cargo and PyPI entries.

### Registry protocol

A **tar registry** is a registry URL whose metadata and tarballs describe tars. A registry that offers both formats does so at two URLs, for example `https://pnpr.corp.example/` for `.tgz` and `https://pnpr.corp.example/~zstd/` for tars. Each URL serves one document per package, so no response varies by request header, and the URL a project keys in `registries` decides the format.

**Tarball URL.** A tar registry serves the tar of a version at the package identifier under the registry base, with no file extension:

```text
<registry>/<name>@<version>
<registry>/@<scope>/<name>@<version>
```

A package name contains `@` only as its scope prefix, so the path is unambiguous, and it does not collide with the npm version-document route `<registry>/<name>/<version>`. pnpm rebuilds it from the `serverType: pnpr` declaration, so lockfiles keep omitting tarball URLs. GitHub Packages already serves npm tarballs from extensionless URLs, and the npm clients handle them.

**Response.** The response body, after content decoding, is the tar:

- `Content-Type: application/x-tar`;
- `Content-Encoding` chosen from the request's `Accept-Encoding`: `zstd`, `gzip`, or none;
- `Vary: Accept-Encoding`, so caches keep one stored variant per encoding.

This is ordinary HTTP content coding, so CDNs and proxies can cache and serve it without knowing about this RFC, and a registry may add or drop an encoding without any change to metadata or lockfiles. The protocol itself names no encoding; which ones a client accepts and a server offers is a property of each, negotiated per request.

pnpm accepts `zstd` and `gzip`. Each encoding pnpm accepts comes with a limit that keeps decoder memory bounded across the many extractions an install runs in parallel: a zstd response follows RFC 9659, whose window is at most 8 MiB, which every level up to 19 respects without `--long`. pnpm rejects streams beyond the limit. The decoded output is bounded separately, by the [decoded-output budget](#decoded-output-budget).

Enforcing the window means pnpm decodes tar responses with its own decoder rather than the HTTP client's. reqwest's built-in zstd decoding uses libzstd's default 128 MiB window limit and offers no setting. Until metadata responses move to the same decoder, tarball requests therefore go through HTTP clients of their own that leave `Content-Encoding` in place, alongside the clients that decode metadata. Metadata responses have been decoded by reqwest since pnpm/pnpm#16585; putting them behind the 8 MiB-window decoder too would leave one set of clients, but it touches every place pnpm reads a metadata body, so it is a separate change.

Adding an encoding later, brotli for example, means advertising its token in `Accept-Encoding`, adding its decoder and window limit to pnpm, and letting registries offer it. Metadata and lockfiles are untouched, because neither refers to an encoding. No metadata or lockfile field describes an encoded response; Integrity of encoded responses under [Rationale and Alternatives](#rationale-and-alternatives) explains why.

**Metadata.** In a tar registry's documents, each version's `dist` describes the tar:

```jsonc
{
  "dist": {
    "integrity": "sha512-<tar digest>",
    "unpackedSize": 1412415,
    "fileCount": 17
  }
}
```

- **`tarball` is optional.** A registry with a declared layout needs no URL per version, and a tar registry omits it. Measured on the fixture's 1,235 abbreviated packuments, dropping the field removes 11% of the raw JSON (2.7% after gzip, because the URLs compress well), and the registry no longer has to rewrite URLs to its public address. A registry may still set `tarball` for a version served from a non-canonical location, and pnpm records such a URL in the lockfile as it does today.
- **Unchanged fields:** everything that describes the unpacked package, such as `unpackedSize`, `fileCount`, and the attestation markers, stays as it is, because both formats hold the same tar.
- **Removed fields:** `shasum` is the SHA-1 of the `.tgz`, and `signatures` sign `name@version:integrity` with the `.tgz` hash, so neither describes the tar. The tar registry leaves both out.
- **Size:** the document is as large as the `.tgz` registry's. Adding a second hash to every version instead would have grown abbreviated metadata by 38%; see [Rationale and Alternatives](#rationale-and-alternatives).

`integrity` may be missing from a version in a tar registry's metadata. A registry learns the tar's hash only once it holds the `.tgz`, so a proxy may not know it for upstream versions it has not fetched yet.

### Resolution

When pnpm resolves a version from a tar registry:

- **`integrity` present:** pnpm records it, trusting the registry's metadata exactly as it trusts `dist.integrity` today.
- **`integrity` missing:** pnpm downloads the tarball and records the hash of the tar it received. The registry produces the tar from a `.tgz` it verified first (see [pnpr](#pnpr)), so this trusts the registry no more than taking a hash from its metadata does. An install needs the tarball anyway. A `--lockfile-only` resolution pays for downloads it would otherwise skip, and only for versions the registry does not describe yet.

### Fetching

pnpm requests the tarball URL with `Accept-Encoding: zstd, gzip`, as it does for every registry. The HTTP client decodes the content coding. pnpm reads the first bytes of the decoded body: the gzip magic means a `.tgz`, which it handles as today, and anything else is a tar, which it hashes as it arrives, verifies against `integrity` before it records the package in the store index, and extracts as a plain tar stream. One body, one hash, one check: a mismatch fails the install.

The tar is only known after decoding, so pnpm decodes every tar response before it can verify it. Today it verifies a buffered `.tgz` before decoding. Integrity of encoded responses under [Rationale and Alternatives](#rationale-and-alternatives) covers why that is acceptable.

#### Decoded-output budget

pnpm verifies a tar only once it has decoded it, so a hostile response can make pnpm decode, hash, and write files before the mismatch is found. The window limit bounds decoder memory, not the decoded output. pnpm has no output ceiling today either: its 64 MiB limit bounds the buffer it reserves up front, and a larger archive streams through in full. This RFC adds explicit limits on the decoded tar, on both the tar and `.tgz` paths. All of them use checked arithmetic, and none of them can be raised by anything the registry sends.

- **File payload:** the bytes of regular-file entries may total at most `maxTarballUnpackedSize` (name open), a project setting with a default of 4 GiB. For a tar, the version's metadata can only tighten the limit: `unpackedSize` is the exact sum of the file payloads, so a payload above it fails the package. On the fixture's 1,141 versions that carry the field, the sum equals it in every case. A `.tgz` gets the ceiling only: tightening it would break installs that work today wherever a registry or publisher under-reports `unpackedSize`, which the fixture cannot rule out.
- **Everything else:** every decoded byte that is not regular-file payload counts against a separate limit, 256 MiB by default. Each entry costs about 1 KiB of headers and padding, so the default admits a package of roughly 250,000 files; the largest packages on npm have tens of thousands. That covers entry headers, pax extension headers, padding, the end-of-archive marker, and the bodies of entries pnpm skips, such as unsupported entry types, which the parser still has to read past. The two limits together cover every decoded byte. This limit is not derived from `unpackedSize` or `fileCount`, because a valid pax header or a skipped entry can be arbitrarily long without changing any installed file.
- **After the end-of-archive marker:** at most one tar record (10 KiB) of zero bytes. Every tarball on the fixture decodes to exactly the marker and nothing after it. This rule is about decoded bytes; a gzip trailer error after the marker on the `.tgz` path keeps today's handling, which accepts the archive.
- **An empty body** is not an archive and fails the package.
- **On overrun:** pnpm stops reading and fails the package. The files it already wrote stay where they are: they are content-addressed, so a file of the same content may already belong to, or be about to be indexed by, another package, and nothing is indexed for the failed package itself. `pnpm store prune` removes blobs that no index refers to, as it does for every other interrupted install.

The same limits apply when pnpm streams a `.tgz` through the decoder while downloading, which it already does for bodies of 4 MiB and more, so this closes a gap that exists today.

The content-addressed store holds files, so a package fetched as `.tgz` in one project and as a tar in another shares its file content. Store index rows keyed by the integrity value differ between the two kinds of entry.

### pnpr

A pnpr registry with `tarballs: tar` serves tars. Formats are exposed as registries, so a deployment offers tars next to `.tgz` by adding a router over the same sources:

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
    tarballs: tar

defaultRegistry: main
```

Here `https://pnpr.corp.example/` serves `.tgz` and `https://pnpr.corp.example/~zstd/` serves tars. Every pnpr registry serves its tarballs at `<registry>/<name>@<version>`. A `.tgz` registry also serves the npm-layout `.tgz` path and names it in `dist.tarball`, for npm and Yarn; a tar registry serves tars only and omits the field.

| deployment | exposed registries |
|---|---|
| `.tgz` only | Today's configuration. |
| both | A `.tgz` registry and a tar registry over the same sources, as above. npm, Yarn, and projects without the declaration use the first. Projects that declare `tar` use the second. |
| tar only | Only tar registries, for example `defaultRegistry: zstd`. Only pnpm can install from it. |

`tarballs` sets the format a registry serves at its own URL. The tar hash and the encoded variants belong to the hosted or upstream registry that holds the package; a router serves what its sources hold:

- **Tar hash:** a hosted registry computes it at publish. An upstream registry computes it when it first fetches a version's `.tgz`, after verifying the `.tgz` against the upstream hash. The value is deterministic, so it can be recomputed at any time, and replicas that derive it independently agree. pnpr stores it in a reserved namespace of the hosted storage backend, which every replica shares, rather than in a replica-local cache, so every replica's metadata carries it once any replica has derived it. Each package has one table there, from `.tgz` integrity to tar integrity. Every value is a function of its key, so the table is shared across registries and storage namespaces: a lookup only happens for versions in metadata the requester may already read, and reveals nothing they could not compute. A replica whose metadata lacks a hash only costs the client a download during resolution.
- **Serving:** pnpr serves a tar from the stored `.tgz`, encoded for the request:
  - `gzip`: the stored `.tgz` itself, sent with `Content-Encoding: gzip`. A `.tgz` is the gzip encoding of its tar, so this costs no CPU. It also derives no tar hash; an upstream version gets one from its first zstd or identity request.
  - `zstd`: the stored level-19 variant once one exists. Until then pnpr compresses at level 3 as the response streams, which is only about 12% smaller than gzip; the 36% measured in this RFC needs level 19, which is too slow to run per request.
  - neither: the tar, decoded as it streams.
- **Level-19 variants:** the first zstd request for a version keeps a copy of the tar it decoded and queues it for a bounded background worker, which replaces the level-3 encoding with a stored level-19 one. Only that one variant is stored, about 60% of the `.tgz` size again. Variants live in each replica's local cache and are evicted with it, because no hash depends on their bytes: a missing variant only means level 3 again until it is regenerated. When the queue is full, the copy is dropped and a later request queues it again. A level-3 response that an HTTP cache still holds stays valid, because both encodings decode to the same tar.
- **The `.tgz` is kept:** publishing still accepts a `.tgz`, and pnpr keeps it, because the `.tgz` is what the publisher's hash, signatures, and attestations cover. The `.tgz` registry keeps serving it unchanged.

### Trust

- **Metadata:** a client that takes a tar hash from a tar registry's metadata trusts that registry as it trusts `dist.integrity` from any registry today. pnpr only ever derives the hash from a `.tgz` it verified.
- **Signatures:** npm's registry signatures sign `name@version:integrity` with the `.tgz` hash, and only npm can change what it signs. A tar registry's metadata carries no `.tgz` hash and no `signatures`, so `pnpm audit signatures` has nothing to verify against it. A client that wants to check the derivation itself reads the `.tgz` hash from the sibling `.tgz` registry's metadata and verifies the `.tgz` it downloads from there.
- **Provenance:** attestations name the `.tgz` digest as their subject and cover the tar by derivation. An attestation that names the tar directly is not part of this RFC. An in-toto statement may list several subjects, but npm's publish-time verifier rejects a bundle with more than one, and pacote checks only the first, so a second subject is not compatible with npm today. A separate attestation with the tar digest as its single subject is the likely shape; see the open question.
- **Install-time checks** such as `trustPolicy` read provenance markers in metadata, which a tar registry keeps, so they work unchanged.

## Rationale and Alternatives

**A second hash field in one document** (`dist.tarIntegrity` next to `dist.integrity`, served by one registry for both formats). This avoids a second registry URL, but a packument carries every version, and an install resolves a few of them. Measured on the fixture's 1,235 packuments, which hold 202,875 versions: adding a sha512 to every version grows the abbreviated metadata from 35.97 MB to 49.78 MB with gzip (+38%) and the full metadata by 13%, about 68 bytes per version, because a hash does not compress. A cold install would still net a 71 MB saving, but every resolution with a cold metadata cache would pay 14 MB for the 1,183 hashes it uses. Serving the hash only in per-version documents or for dist-tag versions would avoid most of the cost at the price of extra requests. A separate registry costs nothing: its documents are the same size as today's.

**Negotiating metadata by request header**, for example a parameter on `Accept`. This keeps one URL, but every metadata response would vary on that header, and caches would have to store a variant per format. A separate registry URL gives the same documents without negotiation and fits how `registries` is keyed.

**Installing a half-converted lockfile.** An earlier draft let an entry install from a registry of the other format: after a hash mismatch, pnpm would hash the decoded tar of a `.tgz`, or fetch a `.tgz` path that tar registries served for this purpose. That answered an integrity failure by trying again with a different hash, which is the wrong reflex for a check meant to catch tampering. A format switch is a registry serving different bytes, and `--update-checksums` already exists for that.

**A separate lockfile field** (`tarIntegrity`) instead of reusing `integrity`. Entries would be self-describing, and tools that ignore unknown fields would be unaffected. But `integrity` already means the SRI of the tarball URL's resource for every source, which covers tars by SRI's own definition; a wrong reading can only fail, never accept other content; and a second field would leave a permanent split in every lockfile if tar hashes become the norm. Nix uses one `hash` attribute the same way, whether a fetcher hashes a file or an unpacked tree.

**Registry-produced `.tar.zst` with its own integrity.** An earlier draft had registries transcode each `.tgz` into a `.tar.zst` and lockfiles pin the hash of the compressed bytes. That makes the compressed bytes part of the artifact's identity, which costs write-once storage, a trust shift to the transcoding registry, lockfiles that only that registry can serve, and a first request that waits for the highest compression level. Hashing the tar removes all of these.

**Integrity of encoded responses.** Metadata could also carry a hash of each encoded form, such as the bytes of the level-19 zstd encoding. This RFC defines none:

- **pnpm never keeps the encoded bytes.** It streams the response through the content decoder into the tar parser, hashes the tar on the way, and verifies it before it records the package in the store index. A hash of the encoded bytes would verify nothing the tar's hash does not.
- **It would bind the encoding again.** A registry would have to serve exactly the hashed bytes, so it could no longer re-compress at another level, regenerate an evicted variant, or let a CDN choose.
- **Decoding before verification:** without a hash of the encoded bytes, pnpm decodes a response before it can verify it. Today pnpm verifies a buffered `.tgz` before decoding it, and already streams tarballs advertised at 4 MiB or more through the gzip decoder while they download, before their hash is final. Since pnpm/pnpm#16585, the same zstd decoder already processes unverified metadata responses from every registry. The [decoded-output budget](#decoded-output-budget) and the 8 MiB window limit bound what a hostile stream can make the decoder do.
- **A local archive cache would not need it.** If pnpm kept archives on disk, it would pick its own on-disk encoding. Checking a cached archive would mean decoding it and hashing the tar, which extraction does anyway.
- **Transfer integrity belongs in HTTP.** A proxy or mirror that wants to verify a cached encoded response without decoding it can use `Content-Digest` (RFC 9530), a hash of the bytes as transferred.

**Per-file hashes,** as JSR publishes for its packages. A manifest of file hashes would also make compression a transport detail and would let clients skip files they already store. It needs a new manifest endpoint and a different store ingestion path. The tar hash keeps today's one-artifact-per-version model and could later be complemented by a manifest.

**`Content-Encoding: zstd` on the `.tgz` URL.** This saves nothing, because gzip output does not compress further. Sending `zstd(tar)` under that URL would violate HTTP semantics, since the decoded body would not be the `.tgz` the URL names.

**Publisher-produced `.tar.zst`.** This needs registry support for a second artifact and helps only packages published after adoption. With the tar hash, any registry can serve any already-published version compressed with zstd.

**Brotli.** On the fixture's tarballs, brotli quality 11 gives 130.4 MB against zstd 19's 143.5 MB (gzip: 228.9 MB), and quality 5 gives 165.7 MB against zstd 3's 198.8 MB. Its decoder is slower, and pnpm and pnpr already ship zstd since pnpm/pnpm#16585, so this RFC adds no second decoder. It is the test case for the design: a registry can offer `br` on the `.tar` endpoint, and a client can accept it, with no change to metadata, lockfiles, or the fallback. reqwest and tower-http both have brotli features, so the client and pnpr halves are each a feature flag plus a window limit when the time comes.

## Implementation

pnpm v12 only. pnpm v11 is maintenance-only and is not changed.

**pnpm:**

- `config`: the `serverType: pnpr` value and its URL layout.
- `resolving-npm-resolver`: accept a missing `dist.tarball`, and a missing `dist.integrity` from a tar registry, recording the hash of the downloaded tar.
- `tarball`:
  - Tell a tar from a `.tgz` by the first bytes of the decoded body, and extract a tar as a plain stream, hashing it on the way.
  - The decoded-output budget on both paths.
  - Enforce the 8 MiB zstd window limit.
  - Choose streaming extraction without a reliable body length, using `dist.unpackedSize` or the size limit.
- Store index and global virtual store: confirm that entries keyed by a tar hash reuse file content and that slot hashes stay correct.
- Docs: `pnpm/docs/registries.md`.

**pnpr:**

- Config: `tarballs` per registry.
- Publish and upstream fetch: compute the tar hash, after verifying an upstream `.tgz`, and record it in the shared hosted storage.
- Metadata: in tar registries, set `dist.integrity` to the tar's hash, drop `tarball`, `shasum`, and `signatures`, and leave `integrity` out for versions whose `.tgz` pnpr does not hold yet.
- Routes: serve `<name>@<version>` on every registry, as a tar with content negotiation on tar registries and as the `.tgz` elsewhere. Tar registries serve no `.tgz` path. Serve gzip clients the stored `.tgz`, and zstd clients a level-19 variant from the local cache, produced by a bounded background worker.
- Docs: `pnpr/docs`.

## Prior Art

- **Go modules:** `go.sum` records `h1:` hashes of a module's file tree, not of the zip, so the hash does not depend on how the archive was written. The module cache verifies a download by hashing what it extracts.
- **Nix:** `fetchurl` hashes the downloaded file and `fetchzip` hashes the unpacked tree, both through the same `hash` attribute; the fetcher decides which kind applies.
- **OCI images:** a manifest records each layer's digest over the compressed blob, and the image config records `diff_ids`, the digests of the uncompressed tars. The same content can exist as gzip and zstd layers.
- **Yarn Berry:** `yarn.lock` checksums hash the zip Yarn builds in its cache, and their `10c0/` prefix encodes the cache format and compression level. When Yarn 4 changed the default compression level, every checksum changed. That is the coupling the tar hash avoids.
- **JSR:** publishes per-file checksums in a version manifest and serves files individually. Its npm compatibility layer generates `.tgz` for npm clients.
- **HTTP content coding:** RFC 8878 registers `zstd`, and RFC 9659 fixes its window for HTTP.

## Unresolved Questions and Bikeshedding

- **Names:** `serverType: pnpr` and pnpr's `tarballs: tar`.
- **Decoded-output budget:** the defaults of the payload ceiling and the non-payload limit.
- **Metadata decoding:** whether to put metadata responses behind the same 8 MiB-window decoder in the same change, or as a follow-up to pnpm/pnpm#16585.
- **Self-computed hashes for other sources:** pnpm computes `integrity` itself for `tarball:` URLs and `file:` tarballs. Hashing the tar there would make those pins survive a host re-compressing the same content, as GitHub did to its `codeload` archives in January 2023 before reverting. It touches sources this RFC does not otherwise change, so it is proposed as a follow-up.
- **Signing tar hashes:** whether pnpr should sign `name@version:integrity` over tar hashes with its own key, so `pnpm audit signatures` works against a tar registry. pnpr has no signing key today.
- **A tar attestation:** whether `pnpm publish --provenance` should produce a second attestation whose single subject is the tar digest, published next to the `.tgz` one. npm's attestations endpoint already serves several attestations per version, but registry acceptance and the verifiers' handling of an extra bundle need testing first. With that and tar-hash signatures, a hosted tar-only registry could stop keeping the `.tgz`, which this RFC does not allow.
- **Upstream hashes:** whether pnpr should fetch the `.tgz` of the latest versions in the background, so `--lockfile-only` resolutions rarely download.
- **Dictionaries:** whether a zstd dictionary trained on npm tarballs is worth a later content coding. It mainly helps small tarballs, which are most tarballs by count.
