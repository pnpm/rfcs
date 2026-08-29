# pnpm agent — Server-Side Resolution for Faster Installs

## Summary

A pnpm agent server that resolves dependencies server-side, computes which files the client is missing at the individual file level, and streams only those files. On warm server, install is consistently faster than standard pnpm — especially on cold OS file caches and in CI environments where the npm CDN edge cache may not be warm.

## Motivation

`pnpm install` with a cold store is slow because it must:

1. Download metadata from the npm registry for every package (hundreds of HTTP requests)
2. Download gzipped tarballs for every package
3. Decompress and extract each tarball
4. Compute SHA-512 hashes for every extracted file
5. Write files to the content-addressable store (CAFS)
6. Link packages into `node_modules`

Steps 1-5 involve significant network I/O, CPU work (decompression, hashing), and disk I/O. In CI environments, these steps run on every build from scratch.

A centralized agent server that has already performed steps 1-5 can serve pre-resolved, pre-extracted files directly — skipping metadata fetches, tarball downloads, decompression, and rehashing entirely. The client only needs to download the files it's missing and link them.

**Use cases:**

- **CI pipelines**: A shared agent server on the same network eliminates npm registry latency. Every CI run benefits from the server's warm cache.
- **Team development**: Multiple developers installing the same dependencies get instant resolution from the server instead of each hitting npm independently.
- **Air-gapped environments**: The agent server can be the sole source of packages, with no need for npm registry access from client machines.

## Detailed Explanation

### Configuration

Add to `pnpm-workspace.yaml`:

```yaml
agent: http://agent.company.com:4873
```

When set, `pnpm install` delegates resolution to the agent server instead of resolving locally.

### Install Flow

1. Client reads integrity hashes from its local store index (SQLite key-only query, no msgpack decode)
2. Sends `POST /v1/install` with dependencies, `storeIntegrities`, overrides, and `minimumReleaseAge`
3. Server resolves the full dependency tree using pnpm's `install({ lockfileOnly: true })` with a SQLite-backed metadata cache
4. As each package resolves, the server immediately streams file digest lines (`D` messages) to the client via NDJSON
5. Client reads the stream line by line; as digest batches accumulate (4000 per batch), it dispatches worker threads to `POST /v1/files`
6. After resolution completes, the server sends store index entries (`I` messages) and the lockfile (`L` message)
7. Client writes pre-packed msgpack index entries directly to its store SQLite and runs headless install for linking
8. The wrapped `fetchPackage` in headless install calls `readPkgFromCafs` with `verifyStoreIntegrity: false` — files from the agent are trusted

### Protocol

**`POST /v1/install`** — NDJSON streaming response:

```
D\t<digest>\t<size>\t<executable>\n    (file the client is missing)
D\t<digest>\t<size>\t<executable>\n
...
I\t<integrity>\t<pkgId>\t<base64-msgpack>\n    (store index entry)
I\t<integrity>\t<pkgId>\t<base64-msgpack>\n
...
L\t{"lockfile":{...},"stats":{...}}\n    (resolved lockfile)
```

`D` lines stream during resolution (overlap with file downloads). `I` and `L` lines come after resolution completes.

**`POST /v1/files`** — gzip-compressed binary stream:

```
[4 bytes: JSON header length][JSON: {}]
[64B digest + 4B size + 1B mode + content]...
[64 zero bytes: end marker]
```

The entire response is piped through `createGzip(level: 1)` on the server and `createGunzip()` on the client. Workers parse and write files to CAFS as decompressed data arrives — no buffering.

### File-Level Dedup

The server computes which individual file digests the client is missing, not just which packages. Even for packages the client doesn't have, many files already exist in the store from other packages:

- `lodash@4.17.20` and `4.17.21` share 595 of 600 files
- Many packages share identical `LICENSE`, `README.md` files
- Version upgrades download only changed files

### Server Architecture

- **Multi-process**: Node.js `cluster` module (default: CPU cores - 1 workers)
- **Resolution**: pnpm's `install()` with `lockfileOnly: true`
- **SQLite metadata cache**: Stores npm registry metadata as blobs, keyed by package name. One indexed DB lookup per package instead of reading multi-MB `.jsonl` files from disk. Resolution drops from ~3.4s to ~0.9s.
- **SQLite file store**: Stores CAFS file contents for consistent read performance regardless of OS file cache state.
- **Streaming resolution**: A wrapped `storeController.requestPackage` intercepts each resolved package and emits digest lines immediately.

### What the Client Skips

Compared to a normal `pnpm install`, the agent client skips:

| Step | Normal pnpm | With agent |
|------|-------------|------------|
| Metadata download | HTTP to npm per package | Skipped (server cached) |
| Version resolution | Local CPU | Skipped (server-side) |
| Tarball download | HTTP to npm per package | Single batched `/v1/files` |
| Tarball decompression | gunzip per tarball | Skipped (pre-extracted on server) |
| File hashing | SHA-512 per file | Skipped (server-provided digest) |
| Store integrity check | stat + hash per file | Skipped (`verifyStoreIntegrity: false`) |
| Temp file dance | write temp → rename | Direct `writeFileSync` with `O_CREAT\|O_EXCL` |

## Rationale and Alternatives

### Alternative 1: Shared network store (NFS/EFS mount)

Mount the CAFS store from a shared filesystem. All machines read/write to the same store.

**Drawbacks**: File locking issues with SQLite over NFS, slow random I/O over network filesystems, no resolution caching, requires infrastructure setup.

### Alternative 2: npm proxy/cache (Verdaccio, Artifactory)

A caching proxy that stores npm tarballs locally.

**Drawbacks**: Still requires per-client resolution, tarball extraction, and file hashing. The proxy only caches the download — all CPU-intensive work still happens on the client. No file-level dedup.

### Alternative 3: Pre-populated store in Docker/CI cache

Cache the `node_modules` or pnpm store directory between CI runs.

**Drawbacks**: Cache invalidation is complex (which files changed?), cache size grows unbounded, doesn't help with resolution, doesn't work across different projects.

### Why agent is better

The agent combines resolution caching, file-level dedup, and pre-extracted file serving in one solution. The client does minimal work — just receive files and link them. The server amortizes the expensive work (resolution, fetching, extraction, hashing) across all clients.

## Implementation

Reference implementation: [pnpm/pnpm#11251](https://github.com/pnpm/pnpm/pull/11251)

### New packages

- `@pnpm/agent.server` — HTTP server with SQLite-backed caches
- `@pnpm/agent.client` — streaming NDJSON client with worker-thread file downloads

### Modified packages

- `@pnpm/installing.deps-installer` — `installFromPnpmRegistry()` function, `agent` config option
- `@pnpm/resolving.npm-resolver` — `metaCache` option in `ResolverFactoryOptions`
- `@pnpm/store.index` — `checkpoint()` method for WAL visibility
- `@pnpm/worker` — `setImportConcurrency()`, streaming `fetchAndWriteCafs`
- Config packages — `agent` setting in workspace YAML

### Server deployment

The server is a standalone Node.js process. Start with `node agent/server/lib/bin.js`. Configuration via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | `4873` | Listen port |
| `PNPM_AGENT_STORE_DIR` | `./store` | Content-addressable store |
| `PNPM_AGENT_CACHE_DIR` | `./cache` | npm metadata cache |
| `PNPM_AGENT_UPSTREAM` | `https://registry.npmjs.org/` | Upstream registry |
| `PNPM_AGENT_WORKERS` | `CPU cores - 1` | Cluster workers |

## Prior Art

- **Yarn Zero Installs**: Vendors packages in the repository (`.yarn/cache`). Avoids network entirely but increases repo size. The pnpm agent takes a different approach — packages live on a shared server, not in the repo.
- **Turborepo / nx Remote Cache**: Caches build outputs on a remote server. Similar concept (shared server amortizes work) but for build artifacts, not package installation.
- **Git packfiles**: Git servers build custom packfiles containing exactly the objects the client needs, delta-compressed. Our `/v1/files` endpoint is analogous — a custom archive of exactly the files the client is missing.
- **Rush build cache**: Stores build outputs in cloud storage for reuse across machines. Similar motivation but different domain.

## Unresolved Questions and Bikeshedding

- **Naming**: The setting is `agent` in `pnpm-workspace.yaml`. Alternatives considered: `pnpmRegistry` (confusing with npm registries), `remoteResolver`, `storeServer`, `depot`. "Agent" was chosen because the server acts on the client's behalf.
- **Store integrity**: Currently `verifyStoreIntegrity: false` is used for all packages when the agent is active, including pre-existing files that may have been corrupted. A more granular approach would only skip verification for files just written by the agent.
- **Serverless deployment**: The agent works best as a long-running server with warm caches. Serverless (Lambda) is not a good fit because the caches are ephemeral and the response sizes exceed typical API Gateway limits.
- **Authentication**: The agent server has no authentication. For production use, it should support tokens or mTLS.
- **Multi-project support**: Currently the server resolves for one project at a time. Workspace support (multiple importers) is not implemented.
