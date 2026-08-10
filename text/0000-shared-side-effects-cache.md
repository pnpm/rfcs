# Shared side-effects cache

## Summary

pnpm already caches the result of a dependency's build so it does not have to run again. That cache is local to one machine. This RFC makes it shareable: a registry may additionally serve the *built* form of a package it already serves, and pnpm may fetch that instead of running the package's install scripts. The artifact is fetched from the registry the package resolved from, over an openly documented endpoint, with pnpr as the reference implementation.

## Motivation

Every CI runner, every fresh container, and every new laptop re-runs `node-gyp` for the same native module with byte-identical inputs. The work is already known to be redundant — pnpm computes a key that proves it, and skips the rebuild whenever the same machine has seen those inputs before. The result is simply not portable.

This is the one capability a Nix binary cache offers that pnpm has no answer to. It surfaced concretely in [pnpm/pnpm#13639](https://github.com/pnpm/pnpm/pull/13639), which proposed delegating package materialization to an external `packageProvider` executable in order to reach it (closing [pnpm/pnpm#11703](https://github.com/pnpm/pnpm/issues/11703)). That approach obtains shared build outputs at the cost of a second install mode in which pnpm's build allowlist, global virtual store, hoisting, and the Rust fast path are all bypassed, and in which a new stdio protocol becomes a permanent compatibility surface.

The underlying need does not require any of that. pnpm already has the cache, the key, and the payload format. What it lacks is a transport and a trust model, and it already has a registry protocol and a registry server to carry both.

Tracking issue: [pnpm/pnpm#13771](https://github.com/pnpm/pnpm/issues/13771).

### What already exists

Three pieces are in place today, and none of them need redesigning:

- **The key.** `calcDepState` (`pnpm11/deps/graph-hasher/src/index.ts`) produces `${platform};${arch};node${major}` plus `;deps=<graph hash>` and `;patch=<hash>`. It already captures the dependency graph, applied patches, and the runtime that will spawn the script.
- **The payload is already a diff over content-addressed files.** `sideEffectsMaps?: Map<string, { added?: FilesMap, deleted?: string[] }>` (`pnpm11/store/cafs-types/src/index.ts:28`). Every added file is an ordinary CAS entry, so the wire format is a manifest of `(path, digest, mode)` plus a list of deleted paths. The bytes themselves are objects the store already knows how to fetch and verify.
- **Read and write are already separate settings.** `sideEffectsCacheRead` and `sideEffectsCacheWrite` (`pnpm11/config/reader/src/Config.ts:148-149`), so remote reads can be enabled without granting publish.

## Detailed Explanation

### Artifacts belong to the registry that served the package

An artifact is fetched from the registry the package resolved from — not from a separate cache service configured out of band.

This is the central design decision, and it is what makes the trust story tractable. The registry that served you `foo@1.0.0`'s tarball is *already* trusted to hand you the code that its install script would execute. A registry that additionally serves the built form of that same package introduces no new principal into the install. A standalone cache URL would: it would be a new party granted permission to write files into `node_modules`, authenticated separately, with its own compromise surface.

Three practical consequences follow:

**Authentication and transport configuration come free.** `_authToken`, `tokenHelper`, CA certificates, proxies, and `npmrc-auth-file` already resolve per registry. A user with a working private registry needs no new configuration to also read artifacts from it.

**Routing is already recorded.** Since packages resolved from a named registry are keyed `<name>@<registry>:<version>` in the lockfile, the lockfile already says which origin to ask. There is no discovery mechanism to design.

**Cross-registry contamination is structurally impossible.** If two registries serve different bytes for the same name and version, their integrity differs, so `fullPkgId` differs, so the side-effects key differs and the entries cannot collide. If they serve identical bytes, sharing between them is correct anyway.

### Fetching

**Cache-hit determination is one round trip for the whole install.** `POST /-/pnpr/v0/side-effects` carries every candidate `(pkgKey, sideEffectsKey)` pair the install might use and returns the manifests that hit, mirroring how `POST /-/pnpr/v0/resolve` takes the entire workspace in a single request. This is a hard requirement, not an optimization: the lookup sits at the tail of an install, immediately before the builds that are the last thing blocking completion, and a per-package design would put tens of serialized round trips on the critical path and scale them with workspace size.

Where the client is already talking to pnpr, the lookup should fold into the resolve request that is already happening, so a warm install pays no additional round trip at all.

Each manifest carries the `added` file list as `(path, digest, mode)` triples and the `deleted` path list, plus the provenance block described below. Sending the *diff* rather than the built tree matters for the same reason: a native module typically adds a handful of files to a package containing hundreds, and a full-tree manifest would scale the response with the size of packages the build never touched.

File contents are fetched as ordinary CAS blobs and verified against their digests exactly as store content is today, reusing the existing concurrent fetch path. A digest mismatch is a cache miss, never a hard failure.

Support is negotiated through the existing `/-/pnpr` handshake rather than assumed. A registry that does not implement the endpoint reports so, and the client builds locally. This makes the feature degrade silently and correctly against `registry.npmjs.org` and every other registry that will never implement it — which is the common case and must not be a failure mode.

Remote reads are off unless enabled. `sideEffectsCacheRead` gains a remote tier rather than being repurposed, so a user who wants the local cache but not remote artifacts keeps that.

### Platform tags

**This is the blocker, and it is independent of everything else in this document.**

`engineName` is `${process.platform};${process.arch};node${major}` and contains no libc. On a single machine that is harmless. Across machines it is not: a musl (Alpine) runner and a glibc (Debian) runner compute the *same* key for a from-source `node-gyp` build, so one would happily install the other's `.node` binary. libc enters the hash today only for `variations` resolutions (`pnpm11/deps/graph-hasher/src/index.ts:486-491`), not for ordinary packages.

Before any artifact crosses a machine boundary, the key must capture everything that changes the artifact. At minimum that means libc family and a version floor. Python's manylinux tags (`cp312-cp312-manylinux_2_17_x86_64`) are the worked example: an entire ecosystem built a tagging standard around exactly this gap, and encoding the glibc floor in the tag is what makes a wheel safe to publish.

Two properties matter more than the specific format:

1. **A tag must be a floor, not an identity.** `manylinux_2_17` means "needs glibc 2.17 or newer", which lets one artifact serve many systems. A tag that pins an exact toolchain version produces a cache that never hits.
2. **An unknown tag must be a miss, not a guess.** A client that does not understand a tag it is offered builds locally.

Compiler and toolchain identity for from-source builds is a harder question and is left open below.

### Provenance

A registry serving an artifact it built itself is making a claim it does not make when relaying a tarball. Relayed bytes are reproducible from upstream; a build is *produced*. Same trust principal, new assertion.

Each entry therefore carries a provenance block recording what produced it: the platform tag it was built for, the toolchain identity where known, whether the build ran sandboxed ([pnpm/pnpm#13772](https://github.com/pnpm/pnpm/issues/13772)), and the builder's identity. This is metadata for policy and debugging, not a security mechanism on its own — but without it, a client cannot express any policy about artifacts at all, and an operator cannot answer "where did this come from" after an incident.

### Interaction with the build allowlist

Today a cached side-effect bypasses the build-policy check: a package that arrives already built never reaches the `allowBuild` filter, so it produces no `ignoredBuilds` entry and does not trip `strictDepBuilds` ([pnpm/pnpm#11035](https://github.com/pnpm/pnpm/issues/11035)). Locally this is a policy annoyance, since the build it is reusing is one this machine already ran.

With remote artifacts it stops being an annoyance. An artifact built elsewhere would install as "already built", with no allowlist check, for a package the user never approved. **A remote artifact must therefore be gated by the same `onlyBuiltDependencies` decision as running the build would be.** Fetching a prebuilt artifact is not a way to avoid approving a package; it is a way to avoid *rebuilding* one you already approved.

### Publishing

Writing artifacts is a separate grant from publishing packages. "May publish `foo`" and "may publish a build of `foo`" are different authorizations: a package is reviewed source, a build is the attacker-controlled output of an install script. The two should not share a policy verb even though they share a registry.

Publication is opt-in per registry and off by default. The expected deployment is a team's own registry, populated by that team's CI.

### What pnpr already provides

The design leans on pnpr's existing routing and storage model rather than adding a parallel one.

- **Public vs private classification.** pnpr's `RoutePolicy` already classifies fetches as public (fetched anonymously, globally shared) or private, and namespaces private resolution-cache entries under an HMAC keyed on `resolution_cache_secret` so they are not correlatable offline. Side-effects entries reuse this directly. The team-cache versus public-cache distinction is not a new axis; it is the existing route classification.
- **Durability.** Artifacts are derived data — reproducible from the tarball plus the build, safe to wipe, self-healing. That is precisely the contract of pnpr's disposable `cache` root, as opposed to the authoritative `storage` root that operators back up. Artifacts inherit it without a new storage tier, and the existing S3 block remains available for operators who want them shared across replicas.
- **Per-package policy.** "Serve artifacts for `@ourco/**`, never for `**`" is a `packages:` pattern rule using the existing specificity selection, in vocabulary operators already know.
- **Existence is not leaked.** `access` already gates reads with denied callers masked as not-found. This matters more than it appears: artifact existence would otherwise reveal what an organization depends on and what its dependency-graph hash is.

## Rationale and Alternatives

**An external package provider ([pnpm/pnpm#13639](https://github.com/pnpm/pnpm/pull/13639)).** Delegate materialization of every package to a third-party executable, which may back it with the Nix store. This does deliver shared build outputs. It costs a permanent stdio protocol, a second install mode incompatible with the global virtual store, the pnpr server, and the hoisted linker, and it silently voids `onlyBuiltDependencies`, `--ignore-scripts`, and `approve-builds` because the provider decides what builds. It also serves only users who have already adopted Nix. The present proposal reaches the same outcome for every pnpm user without a second install path.

**A standalone cache URL, in the shape of Turborepo's Remote Cache.** A `sideEffectsCacheUrl` setting pointing at a dedicated service. This is the obvious design and was the initial sketch. It was rejected because it duplicates the entire registry authentication surface — tokens, helpers, certificates, proxies — and because it introduces a new principal permitted to write into `node_modules`. Registry attachment avoids both. The cost is that it does nothing for users installing directly from `registry.npmjs.org`; the answer there is a proxying pnpr, which is already how such teams deploy.

**Adopt the Bazel Remote Execution API v2 instead of defining an endpoint.** REAPI is a real standard with off-the-shelf servers (bazel-remote, Buildbarn, BuildBuddy), and moon adopted it after sunsetting its own hosted cache. Its data model is genuinely close: a content-addressable blob store plus an action cache mapping a digest to a result manifest. It also covers more of this proposal than it first appears — `Platform` properties are explicitly intended for "hardware, operating system, or compiler toolchain" and are implicitly part of the Action digest, which is the platform-tag role; `ExecutedActionMetadata` records the producing worker, which is the provenance role; and `GetActionResult`/`UpdateActionResult` work standalone without the Execution service. This is the strongest alternative and the decision is close.

It is declined primarily on **performance**. `ActionCache` exposes exactly two RPCs, and `GetActionResultRequest` carries a single `action_digest` — there is no batched cache-hit lookup. Its CAS service batches blob I/O (`FindMissingBlobs`, `BatchReadBlobs`), but the part on the critical path is determining *which* packages hit, and REAPI can only answer that one package per round trip. pnpm and pnpr have deliberately moved the other way: `POST /-/pnpr/v0/resolve` takes an entire workspace in one request. Adopting REAPI would reintroduce per-package chatter precisely where it was engineered out, at the tail of the install, and would scale it with workspace size. Folding the lookup into an existing resolve round trip — the design above — is structurally unavailable to a separate service speaking a separate protocol.

Two further costs compound it. `ActionResult` carries a full output tree, so a native module adding three files to an 800-file package transfers 800 manifest entries: response size scales with the size of packages the build never touched. And the TypeScript client would need `@grpc/grpc-js` bundled, or REAPI's HTTP binding, which is less widely supported and forfeits the HTTP/2 multiplexing that would otherwise partly offset the round-trip count.

Two non-performance objections also stand. `Action` requires a `command_digest` referencing a `Command` in CAS, but pnpm has no command — only "this package, these dependencies, this platform" — so the key would have to be derived from a synthesized `Command` that no worker can execute, making its exact canonicalization an unintended compatibility surface. And whether a platform property matches exactly or as a range is explicitly server-dependent, which is workable for one operator's cluster but not for a format many independent registries are meant to serve, where a floor semantic like manylinux's must mean the same thing everywhere.

Worth revisiting if REAPI gains a batched `ActionCache` lookup, or if this endpoint drifts toward REAPI's shape anyway.

**Do nothing.** Native module builds stay per-machine. This is survivable — it is the status quo — but it leaves a real cost on every CI run, and it leaves the Nix-shaped proposals as the only answer for teams that care.

## Implementation

**`pnpm/pnpm` (TypeScript CLI and pacquet).** Platform tags in `calcDepState` and `engineName` land first and independently; they are a correctness fix to the existing local cache regardless of whether the rest ships. Then a remote tier on the side-effects lookup in the store controller, feeding the existing `sideEffectsMaps` path so nothing downstream of the cache changes. The `allowBuild` gate must be moved so it is consulted before a remote artifact is accepted, which also fixes [pnpm/pnpm#11035](https://github.com/pnpm/pnpm/issues/11035) for the local case. Both stacks, per the repository's parity rule.

**`pnpr`.** The batched endpoint on the existing resolver surface alongside `/-/pnpr/v0/resolve`, capability-advertised through the `/-/pnpr` handshake, and ideally answerable as part of a resolve response so a warm install adds no round trip. Storage in the disposable `cache` root. Reuse of `RoutePolicy` for public/private classification and of `packages:` rules for per-package policy. A publish path with its own policy verb.

**Documentation.** The endpoint and the platform tag format are a public contract from the moment they ship, and must be specified as such so registries other than pnpr can implement them.

Platform tags are a prerequisite for everything else. Nothing may cross a machine boundary before they are correct.

## Prior Art

**Turborepo and Nx** cache first-party task outputs, keyed on declared inputs, over a plain HTTP API that anyone can self-host. Turborepo additionally signs artifacts (`remoteCache.signature: true` with `TURBO_REMOTE_CACHE_SIGNATURE_KEY`, HMAC-SHA256, verification failure treated as a cache miss). Two lessons: a documented HTTP spec is enough for a healthy ecosystem of implementations, and treating verification failure as a miss rather than an error is the right default.

Where the analogy breaks: their threat model is a compromised cache operator or transport, which signing answers, because the key never leaves the user's machines. Ours includes an artifact that is the *legitimate* output of a third-party install script nobody read. Signing establishes provenance, not safety — hence the allowlist gate above.

Their cache keys are also user-declared in configuration, so a wrong key is user misconfiguration that costs a stale build. pnpm computes the key invisibly on the user's behalf, which is why the platform tag gap is pnpm's liability and why it must be fixed first.

**moon** ran its own hosted cache service, Moonbase, then sunset it and implemented the Bazel Remote Execution API v2 so users could run off-the-shelf servers instead. This is a JS-ecosystem tool that built a bespoke protocol and walked it back — the reason the REAPI alternative is treated seriously above rather than dismissed.

**Python wheels** are the closest analogue: the one mainstream ecosystem that made the built form of a third-party dependency a first-class distributable. `pip install` used to run `setup.py build` on every machine. PyPI serves sdists and wheels from one index under one authentication story, which is the registry-attached model this RFC proposes. The manylinux tag system exists precisely because of the libc problem, and is the direct model for platform tags here.

**Nix binary caches and Homebrew bottles** solve the same problem with per-package derivations and signed substituters. They achieve stronger guarantees than this proposal by making builds reproducible and sandboxed by construction. pnpm cannot assume either of npm install scripts, which is why provenance metadata and the allowlist gate substitute for reproducibility.

## Unresolved Questions and Bikeshedding

- **Toolchain identity for from-source builds.** libc is necessary but may not be sufficient: a native module built with a different compiler or C++ standard library may not be interchangeable. Does the tag need a toolchain component, or is that over-fitting? manylinux gets away without one; whether npm's build ecosystem is as forgiving is unknown.
- **Non-relocatable builds.** Some builds embed absolute paths or other machine state and are not shareable at all. Per-package opt-out, a `package.json` marker, or detection? [pnpm/pnpm#5002](https://github.com/pnpm/pnpm/issues/5002) asks for the local form of this.
- **Sandboxed and unsandboxed builds.** If dependency scripts eventually run sandboxed ([pnpm/pnpm#13772](https://github.com/pnpm/pnpm/issues/13772)), are the artifacts interchangeable? Provenance records which produced an entry; it does not say whether a client should accept the other.
- **Signing.** Turborepo-style HMAC is straightforward for a team cache where one key is shared. Whether anything stronger is warranted depends on whether a public cross-organization cache is ever in scope — this RFC deliberately does not propose one.
- **Endpoint shape and naming.** The batching requirement is settled; the path and payload names are not. Whether small files ride inline in the batch response instead of costing a second blob fetch is a measurable question and should be measured rather than guessed.
- **Which packages to ask about.** Only packages with install scripts are candidates, which keeps the request small — but that set is known from the lockfile at slightly different points on the frozen and non-frozen paths, and the request should not delay either.
- **Cache key visibility.** Users currently cannot see why an artifact missed. Some `pnpm why`-shaped diagnostic may be needed once the key has more components.
