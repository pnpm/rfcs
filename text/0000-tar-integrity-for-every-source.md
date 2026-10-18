# Tar integrity for every source

## Summary

A project can pin the hash of the uncompressed tar for every package in its lockfile, whatever registry or URL the package comes from and however it is compressed on the wire:

```yaml title="pnpm-workspace.yaml"
lockfileIntegrity: tar
```

With the setting on, `integrity` in `pnpm-lock.yaml` is the tar's hash for every entry. pnpm derives it from a `.tgz` it verified against the publisher's hash, or takes it from a tar registry's metadata, and verifies every download by decoding it and hashing the tar. The lockfile then no longer changes when a registry re-compresses its tarballs, starts serving tars, or is swapped for a mirror that does either.

This is the follow-up that `0000-zstd-registry-tarballs.md` left open. That RFC made the hash kind follow the registry: tar registries pin tar hashes, every other source pins the `.tgz`. This one makes it a property of the project instead.

## Motivation

Pinning the compressed bytes couples a lockfile to one encoding of each package:

- **A registry cannot change how it compresses.** Switching a `.tgz` registry to a tar registry, or moving from npmjs to a pnpr in front of it, invalidates every hash and needs `--update-checksums`. The tar RFC accepted that cost because the hash kind followed the registry.
- **A host re-compressing the same content breaks installs.** In January 2023 GitHub changed how it compressed `codeload` archives, and checksums broke across Bazel, Homebrew, and Nix until it reverted. Tools that hashed the unpacked content were unaffected. pnpm pins `tarball:` URLs by their compressed bytes and would have broken.
- **Two kinds of hash in one lockfile** is a permanent split if tar registries spread. An entry's meaning depends on which registry it came from, and a reader has to know the registry to interpret it.

Hashing the tar removes the coupling: the hash is a function of the published content, and compression becomes a transport detail everywhere, not only on tar registries. Go's `go.sum` and Nix's `fetchzip` already pin content this way; see Prior Art.

## Detailed Explanation

### The setting

`lockfileIntegrity` is a project setting in `pnpm-workspace.yaml`.

| value | meaning |
|---|---|
| `tarball` (default) | Today's behavior: an entry pins the hash of the resource at its tarball URL, a `.tgz` for most registries and a tar for tar registries. |
| `tar` | Every package tarball entry pins the tar's hash, from every source (see [Which entries](#which-entries)). |

The lockfile records the setting under `settings`, as it records other settings that change how entries are read:

```yaml
settings:
  autoInstallPeers: true
  lockfileIntegrity: tar
```

The key appears only when the setting is `tar`. A lockfile without it reads exactly as today.

### Which entries

The setting covers every entry whose resource is a package tarball: registry packages, `tarball:` URLs, and `file:` tarballs. Other kinds keep the integrity semantics they have today:

- **Binary resolutions** (`BinaryResolution`: Node, Deno, and Bun runtime archives) pin the downloaded archive, which may be a zip. They are not package tarballs and have no tar in this RFC's sense.
- **Custom fetchers** own their integrity semantics and are not touched.
- **Git and directory dependencies** have no `integrity`.

### Which hash pnpm computes

With `lockfileIntegrity: tar`, pnpm hashes the decoded tar of every download:

- **A tar body** is hashed as it arrives, as on tar registries today.
- **A `.tgz` body** is decoded and the tar hashed as it streams. The `.tgz` bytes themselves are not compared against anything the lockfile holds.

Both run under the decoded-output budget the tar RFC defined, so a hostile body is bounded the same way on both paths. This moves the `.tgz` path to decode-before-verify, which the tar RFC's "Integrity of encoded responses" section discusses; the budget is what makes it acceptable.

### Where the hash comes from at resolution

- **Tar registries:** from `dist.integrity`, as today.
- **`.tgz` registries, including npmjs:** metadata carries only the `.tgz` hash. pnpm downloads the `.tgz`, verifies it against `dist.integrity` (or `dist.shasum` for versions without it), decodes it, and records the tar's hash. The lockfile entry is derived from the publisher's hash, so no trust moves to the registry.
- **`tarball:` URLs and `file:` tarballs:** pnpm already downloads and hashes these at resolution; it hashes the decoded tar instead of the file.
- **Git and directory dependencies:** unchanged; they have no `integrity`.

### The cost, and the store as its cache

Deriving a tar hash needs the tarball. The costs by command:

- **`pnpm install`** downloads every tarball it does not have in the store anyway. Hashing the decoded tar adds about three times the `.tgz` bytes to the hashing work: for the fixture project of the tar RFC, roughly 0.7 GB per cold install, under a second of CPU across threads.
- **`pnpm update`** installs what it selects, so the tarballs of new versions are downloaded either way. Versions it keeps need a hash only if the store cannot supply one.
- **`pnpm install --lockfile-only`** and **`pnpm update --lockfile-only`** download no tarballs today. With the setting on, they download the `.tgz` of every version from a `.tgz` registry whose tar hash is not already known: 229 MB for the fixture project on a cold machine.

Two things keep the download from being the steady state:

- **The store index records the mapping, and is readable by either hash.** Whenever pnpm extracts a `.tgz` it verified, it records `tgz integrity -> tar integrity` in the store index row and makes the row reachable under the tar hash as well. Store lookups are keyed by the locked `integrity`, so without the second key a frozen install under the setting would miss every row written under a `.tgz` hash and take the cold path for packages whose files are already in the store. A resolution of a version whose row carries the mapping takes the tar hash from it and downloads nothing. The mapping cannot be rebuilt from the stored files, because the store holds the files and not the tar, so a row written before pnpm recorded mappings has none: the first resolution or install under the setting downloads that version's `.tgz` once to derive the hash, even though its files are already in the store. On a developer machine or a CI runner whose store was populated by a pnpm that records mappings, nearly every version has one.
- **Tar registries carry the hash in metadata.** A project behind a pnpr tar registry never pays the download.

What remains is one download per version this machine has never installed from a `.tgz` registry, or installed before mappings were recorded. For an install, that is the download it would do anyway; for a lockfile-only resolution, it is new.

### Switching the setting

Turning `lockfileIntegrity: tar` on or off changes what every `integrity` means, so pnpm treats a lockfile whose recorded setting differs from the configured one like any other outdated setting: a frozen install fails with an error that names the setting, and a non-frozen install re-resolves and rewrites the hashes. The conversion is one install and one lockfile diff, in which every entry changes. Converting in the other direction behaves the same.

### Older pnpm versions

A lockfile with `lockfileIntegrity: tar` holds tar hashes that an older pnpm would compare against `.tgz` bytes, and it must not get the chance: it ignores the unknown settings key, warns about the unknown `pnpm-workspace.yaml` setting, and would then fail with an integrity mismatch that reads like tampering.

The lockfile version is the only gate every released reader honors, and only a new major works:

- pnpm v12 accepts any `lockfileVersion` whose major is 9, and also major 12 (`LockfileVersion::is_compatible`). A minor bump such as `9.1` gates nothing.
- pnpm v11 accepts any version with the same major, and only warns that the lockfile "was generated by a newer version of pnpm" when the minor is higher, then proceeds.

A lockfile written with `lockfileIntegrity: tar` therefore carries a lockfile major that no released pnpm accepts. A pnpm that implements this RFC accepts that major and reads `settings.lockfileIntegrity` to know how to verify. A lockfile with the setting off keeps today's version, so projects that do not opt in see no change. Which major number to reserve is an open question, because v12 already treats 12 as compatible.

What the gate guarantees differs by reader:

- **pnpm v12** rejects the lockfile when it parses it, with "The lockfileVersion of X is incompatible with this version of pnpm". There is no bypass.
- **pnpm v11** rejects it the same way in a frozen install and in a plain `pnpm install`. With `--force`, or in CI without frozen mode, v11 deliberately ignores an incompatible lockfile, re-resolves, and rewrites it as a `9.0` lockfile with `.tgz` hashes. That is a silent downgrade, not a misread: the rewritten lockfile is valid for the pnpm that wrote it, and the next pnpm that implements this RFC sees the setting mismatch and converts it back on a non-frozen install. Projects that pin pnpm through `packageManager` do not meet this case.

### Trust

- **Resolution** trusts the publisher's hash, not the registry's compression: the tar hash is computed from a `.tgz` that matched `dist.integrity`, except on tar registries, where it comes from metadata as the tar RFC already allows.
- **Registry signatures** sign `name@version:<.tgz integrity>`. `pnpm audit signatures` keeps verifying them against the `.tgz` hash in metadata; to tie that hash to the lockfile's tar hash it uses the store index mapping when the package is in the store, and otherwise downloads the `.tgz`, verifies it, and derives. The command reports which entries it could tie and which it could not.
- **Provenance** names the `.tgz` digest and covers the tar by derivation, as in the tar RFC.
- **A wrong reading can only fail.** Whether a value is read as a `.tgz` hash or a tar hash, the bytes either match or they do not, and no other content matches without a SHA-512 preimage.

## Rationale and Alternatives

**Keep the hash kind per registry** (the tar RFC as merged). Simplest, and right as a first step, but a lockfile is then valid only against registries of the kinds it was resolved from, every registry format switch needs `--update-checksums`, and two kinds of hash coexist indefinitely.

**Make tar hashes the default now.** The download cost on first resolution falls on every project at once, including bots that run `--lockfile-only` per pull request on machines with no store. An opt-in lets projects that are behind a tar registry, or that keep a store, take the benefit first, and lets the store-index cache prove itself before a default changes.

**Record both hashes per entry.** OCI records a compressed digest per layer and `diff_ids` for the uncompressed tars. In a lockfile that is a third more size for the fixture project, two values per entry that must agree, and on the tar path only one of them can be checked. Metadata keeps the `.tgz` hash, and the store index keeps the mapping, so the lockfile needs only one.

**A self-describing value** (a prefix on the SRI string, as Go's `h1:` or Yarn's `10c0/`). It would make an entry's kind visible, at the cost of breaking every SRI parser that reads lockfiles. The lockfile-level setting carries the same information for every entry at once.

**A separate field** (`tarIntegrity`). Rejected in the tar RFC: `integrity` already means the SRI of the resource, and a second field would split every lockfile.

## Implementation

pnpm v12 only.

- `config`: the `lockfileIntegrity` setting; `pnpm-workspace.yaml` only.
- `lockfile`: the `settings.lockfileIntegrity` record, its freshness check, and the lockfile version gate.
- `tarball`, `store-dir`: hash the decoded tar of a `.tgz` body when the setting is `tar`, on the buffered, streamed, and local-file paths, under the decoded-output budget. Record the `tgz integrity -> tar integrity` mapping in the store index on every verified extraction, with the setting on or off, so the cache fills before a project opts in, and make the row reachable under both hashes.
- Resolution: derive the tar hash from the verified `.tgz` for `.tgz` registries, `tarball:` URLs and `file:` tarballs; consult the store index first; take tar registries' metadata as today.
- `audit signatures`: tie tar hashes to signed `.tgz` hashes through the store index, or by downloading.
- Tests: a frozen install with the setting on, a store populated by `.tgz` installs, an empty `node_modules`, and `--offline` must succeed without a download; and the reverse direction.
- Docs: the setting, the switch procedure, the lockfile-only cost, and the v11 `--force` downgrade.

## Prior Art

- **Go modules:** `go.sum` hashes a module's file tree, not the zip, so the hash survives any re-archiving. The module cache verifies downloads by hashing what it extracts.
- **Nix:** `fetchzip` hashes the unpacked tree, which is why Nix packages survived GitHub's 2023 `codeload` change while `fetchurl` pins did not.
- **OCI images:** `diff_ids` in the image config are the digests of the uncompressed layer tars, next to the compressed digests in the manifest.
- **Yarn Berry:** `yarn.lock` checksums hash the zip Yarn builds in its cache, and changing the cache's compression level in Yarn 4 changed every checksum. That is the coupling this RFC removes.
- **`0000-zstd-registry-tarballs.md`:** defines the tar, its hash, the decoded-output budget, and tar registries; this RFC builds on all of them.

## Unresolved Questions and Bikeshedding

- **Names:** `lockfileIntegrity` with `tarball` and `tar`, or a boolean such as `tarIntegrity: true`.
- **Lockfile major to reserve:** v12 readers already accept major 12, so the gate needs another number, and it should be shared with any other pending lockfile change that needs one.
- **Default:** whether and when `tar` becomes the default. The store-index cache and the spread of registries that publish tar hashes decide how much the first-resolution download costs in practice.
- **Store index mapping:** whether to record it only when the setting is on, or always, so a later opt-in finds a warm cache. The RFC proposes always; the cost is one hash per extraction, which the extraction already streams past.
- **`audit signatures` without the store:** whether a download to tie a signature to a tar hash should be the default or need a flag.
