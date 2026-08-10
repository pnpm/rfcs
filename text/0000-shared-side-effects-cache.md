# Shared side-effects cache

## Summary

pnpm already caches the result of a dependency's build so it does not have to run again. That cache is local to one machine. This RFC makes it shareable for the subset of packages where sharing is sound: a registry may additionally serve the *built* form of a package it already serves, and pnpm may fetch that instead of running the package's install scripts. Artifacts are fetched from the registry the package resolved from, over an openly documented protocol, gated by an explicit per-registry opt-in, with pnpr as the reference implementation.

## Motivation

Every CI runner, every fresh container, and every new laptop re-runs `node-gyp` for the same native module. The work is usually redundant, and pnpm already detects the common case: it computes a dependency-state key and skips the rebuild whenever the same machine has seen those inputs before. The result is simply not portable.

This is the one capability a Nix binary cache offers that pnpm has no answer to. It surfaced concretely in [pnpm/pnpm#13639](https://github.com/pnpm/pnpm/pull/13639), which proposed delegating package materialization to an external `packageProvider` executable in order to reach it (closing [pnpm/pnpm#11703](https://github.com/pnpm/pnpm/issues/11703)). That approach obtains shared build outputs at the cost of a second install mode in which pnpm's build allowlist, global virtual store, hoisting, and the Rust fast path are all bypassed, and in which a new stdio protocol becomes a permanent compatibility surface.

The underlying need does not require any of that. pnpm already has the cache, a key, and the payload format. What it lacks is a transport, a trust model, and — the substance of this RFC — a defensible answer to when a build output may be shared at all.

Tracking issue: [pnpm/pnpm#13771](https://github.com/pnpm/pnpm/issues/13771).

### What already exists

- **A dependency-state key.** `calcDepState` (`pnpm11/deps/graph-hasher/src/index.ts`) produces `${platform};${arch};node${major}` plus `;deps=<graph hash>` and `;patch=<hash>`. It captures the dependency graph, applied patches, OS/arch, and the Node major that will spawn the script.
- **A payload that is already a diff over content-addressed files.** `sideEffectsMaps?: Map<string, { added?: FilesMap, deleted?: string[] }>` (`pnpm11/store/cafs-types/src/index.ts:28`). Every added file is an ordinary CAS entry, so a manifest is `(path, digest, mode)` triples plus deleted paths.
- **Separate read and write settings.** `sideEffectsCacheRead` and `sideEffectsCacheWrite` (`pnpm11/config/reader/src/Config.ts:148-149`).
- **A build allowlist that already covers cached builds.** Since [pnpm/pnpm#11039](https://github.com/pnpm/pnpm/pull/11039), `allowBuild` is consulted for packages that would otherwise be skipped as already built (`pnpm11/building/during-install/src/index.ts:111,141`). This RFC must preserve that gate, not reintroduce it.

## Detailed Explanation

### The build-input closure problem

**The dependency-state key is not a function of everything a build depends on, and for arbitrary npm packages it cannot be made one.**

This is the central constraint, and it is easy to understate. `calcDepState` captures the dependency graph, patches, OS, arch, and Node major. A lifecycle script may additionally depend on environment variables, compiler and linker versions, the C++ standard library, Python, platform SDKs, system headers and shared libraries, CPU feature detection, network responses, the wall clock, and arbitrary files on the host. libc is one known omission; it is not the only one, and enumerating the rest is not a finite task.

The consequence is that two honest builders can produce materially different outputs under the same key. A cache that assumes otherwise is not merely inefficient — it silently serves the wrong artifact. This is precisely where the Nix analogy breaks down: Nix can treat a store path as a pure function of its inputs because it *models* the inputs and builds in a restricted environment. npm install scripts do neither.

Sharing therefore requires closing the input set by one of two means, and this RFC proposes both:

**A declared builder profile.** An artifact records the environment it was produced in — a versioned, named profile (base image or equivalent, toolchain versions, libc) rather than an open-ended property bag. A client accepts an artifact only if its profile is one the client is configured to trust. This does not make builds reproducible; it makes the environment an explicit, comparable part of the entry instead of an unstated assumption.

**A package eligibility contract.** Sharing is opt-in per package, not blanket. The default for a package nobody has vouched for is to build locally. Eligibility may be asserted by the publisher, by the registry operator for packages they have evaluated, or by the consuming organization for its own dependencies. A package whose build reads the clock, the network, or the host's mutable state is not eligible, and no key design fixes that.

Sandboxed lifecycle scripts ([pnpm/pnpm#13772](https://github.com/pnpm/pnpm/issues/13772)) are the mechanism that makes this rigorous rather than advisory: a sandbox is how the input set is actually *closed* rather than merely described. This RFC does not depend on sandboxing shipping first, but the two compose, and a sandboxed build is the strongest form of an eligible one.

### Compatibility tags are not the lookup key

An artifact built against glibc 2.17 should serve a glibc 2.39 machine. That is a floor, not an identity — and a floor cannot be expressed by hashing the consumer's own platform into the key it looks up. A 2.39 client would request a key containing 2.39 and never find the 2.17 artifact.

Two distinct concepts are therefore required:

- **An input key** identifying what was built: the dependency graph, patches, and the builder profile. This is `calcDepState` extended, and it is what an entry is stored under.
- **Compatibility constraints** the artifact advertises: the platform tags it is valid on, expressed as floors where the underlying property admits one.

The client presents an ordered set of tags it supports and selects the best offered artifact, exactly as Python installers select among wheels rather than hashing the interpreter's exact identity into a filename ([PyPA platform compatibility tags](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/#use)). Two properties follow: an artifact serves many systems rather than one, and a tag the client does not understand is a miss, never a guess.

### Fetching

**Requests are batched per registry and issued in parallel.** Artifacts belong to the registry a package resolved from, and an install may draw from several. The client groups candidates by concrete registry and sends each registry only its own candidates — both because there is no single origin that could answer for all of them, and because broadcasting the full candidate set would leak an organization's cross-registry dependency graph to every registry it talks to.

Within a registry the lookup is one round trip for the whole install, mirroring how `POST /-/pnpr/v0/resolve` takes an entire workspace in a single request. This matters because the lookup sits immediately before the builds that are the last thing blocking install completion. Where the client is already talking to pnpr, the lookup should fold into the resolve request that is already happening, so a warm install pays no additional round trip at all.

Only packages with install scripts are candidates, which keeps requests small.

Each response entry carries the `added` file list as `(path, digest, mode)` triples, the `deleted` path list, the compatibility tags, and the signed provenance described below. Sending a diff rather than a built tree keeps the response proportional to what the build changed rather than to the size of the package.

Support is negotiated through the existing `/-/pnpr` handshake. A registry that does not implement the endpoint says so and the client builds locally, so the feature degrades correctly against `registry.npmjs.org` and every other registry that will never implement it.

### Manifest validation is mandatory

A remote manifest supplies file paths that reach `importIndexedDir` (`pnpm11/fs/indexed-pkg-importer/src/importIndexedDir.ts`). Locally produced side-effect paths come from walking a directory pnpm itself created; remote paths carry no such guarantee. The existing `sanitizeFilenames` helper is a compatibility retry for badly-named files in tarballs — it runs only after an import failure and sanitizes per path segment — and must not be mistaken for a security boundary.

Before any remote manifest is acted on, both clients must reject: absolute paths; any `..` segment; platform-specific separators; NUL and control characters; duplicate paths and paths colliding under case-insensitive filesystems; symlink or special-file entries not explicitly modelled; modes outside an allowed set; and file counts or sizes beyond configured limits. A manifest failing any check is a cache miss and the package builds locally.

This matters more than the equivalent check on tarballs because artifact-publish authority is deliberately weaker and more widely delegated than package-publish authority.

### Trust

The registry that served a package's tarball is already trusted to hand over the code its install script would execute, so serving the built form of that same package is a smaller step than trusting an unrelated cache service. But the step is not zero, and the RFC should not claim it is.

**In a frozen install the lockfile pins the tarball's integrity, so a compromised registry cannot substitute source bytes undetected. No such pin exists for an artifact.** A registry — or anyone it has authorized to upload artifacts — can therefore substitute arbitrary built files without violating any integrity the lockfile records. That is strictly more authority than a registry has today, and `onlyBuiltDependencies` does not cover it: approving a package authorizes running *its* script, not accepting arbitrary output attributed to it by a third party.

This RFC resolves that as follows:

- **Remote artifacts are a separate, per-registry opt-in.** `sideEffectsCacheRead` continues to mean the local cache and is not silently widened. A user enables remote artifacts for a named registry deliberately, and that act is the grant of the additional authority described above.
- **Provenance is signed, not merely recorded.** A signature binds the protocol version, the source tarball integrity, the input key, the compatibility constraints, the manifest digest, and the builder identity. Unauthenticated provenance fields are useful for debugging and useless against substitution; only the binding makes an artifact attributable.
- **The existing allowlist gate is preserved.** A remote artifact is accepted only for a package `allowBuild` already permits. Fetching a prebuilt artifact avoids *rebuilding* something you approved; it is not a route around approving it.

### Publishing

Writing artifacts is a separate grant from publishing packages, with its own policy verb. "May publish `foo`" and "may publish a build of `foo`" differ in kind: a package is reviewed source, a build is the output of an install script. Publication is opt-in per registry and off by default; the expected deployment is a team's own registry populated by that team's CI.

### What pnpr already provides

The design reuses pnpr's routing and storage model rather than adding a parallel one.

- **Public vs private classification.** `RoutePolicy` already classifies fetches as public (anonymous, globally shared) or private, namespacing private resolution-cache entries under an HMAC keyed on `resolution_cache_secret` so they are not correlatable offline. Side-effects entries reuse it directly.
- **Durability.** Artifacts are derived data: reproducible from the tarball plus the build, safe to wipe, self-healing. That is the contract of pnpr's disposable `cache` root as distinct from the authoritative `storage` root, so artifacts need no new storage tier. The existing S3 block remains available for operators who want them shared across replicas.
- **Per-package policy.** "Serve artifacts for `@ourco/**`, never for `**`" is a `packages:` pattern rule using the existing specificity selection.
- **Existence is not leaked.** `access` already gates reads with denied callers masked as not-found, which matters because artifact existence would otherwise reveal an organization's dependency graph and input keys.

## Rationale and Alternatives

**An external package provider ([pnpm/pnpm#13639](https://github.com/pnpm/pnpm/pull/13639)).** Delegate materialization of every package to a third-party executable, optionally backed by the Nix store. This delivers shared build outputs, at the cost of a permanent stdio protocol, a second install mode incompatible with the global virtual store, the pnpr server, and the hoisted linker, and the silent voiding of `onlyBuiltDependencies`, `--ignore-scripts`, and `approve-builds` because the provider decides what builds. It also serves only users who have already adopted Nix. Notably, it does *not* escape the build-input closure problem — it relocates it into the provider, where Nix's restricted build environment is what actually solves it. This RFC addresses that problem directly instead, via builder profiles and eligibility.

**A standalone cache URL, in the shape of Turborepo's Remote Cache.** A `sideEffectsCacheUrl` pointing at a dedicated service. Rejected because it duplicates the entire registry authentication surface — tokens, helpers, certificates, proxies — and introduces a wholly new principal permitted to write into `node_modules`, without even the partial containment of the registry relationship. Its cost is that it does nothing for users installing directly from `registry.npmjs.org`; the answer there is a proxying pnpr, which is already how such teams deploy.

**Adopt the Bazel Remote Execution API v2 instead of defining a protocol.** REAPI is a real standard with off-the-shelf servers (bazel-remote, Buildbarn, BuildBuddy), and moon adopted it after sunsetting its own hosted cache. It covers more than it first appears: `Platform` properties are explicitly intended for "hardware, operating system, or compiler toolchain" and are implicitly part of the Action digest; `ExecutedActionMetadata` records the producing worker; `GetActionResult`/`UpdateActionResult` work without the Execution service; and because `Command` declares which outputs are captured, an `ActionResult` need not carry a full tree — pnpm's added-file set could be declared directly as outputs.

It is nonetheless declined, primarily because **its lookup model is exact-digest and this design requires compatibility selection**. `GetActionResult` answers "is there a result for precisely this Action digest". It cannot answer "give me the best artifact compatible with this ordered tag set", which is the question a floor-based tag model must ask. Emulating it means the client enumerating one candidate digest per supported tag in preference order and issuing a lookup for each — multiplying request count by tag-set size, in a protocol that has no batched `ActionCache` read at all (`GetActionResultRequest` carries a single `action_digest`; only the CAS service batches). Three narrower mismatches compound it: `Action` requires a `command_digest` referencing a `Command` in CAS, but pnpm has no command, so the key would derive from a synthesized `Command` no worker can execute, making its canonicalization an unintended compatibility surface; `ActionResult` outputs are purely additive, with no representation for the `deleted` paths a side-effects map carries; and whether a platform property matches exactly or as a range is explicitly server-dependent, which is workable for one operator's cluster but not for a format many independent registries must interpret identically.

Raw round-trip count is deliberately *not* claimed as the deciding factor. gRPC multiplexes concurrent RPCs over HTTP/2, so N lookups are not N serialized round trips, and whether batching wins on wall-clock is a question for measurement rather than assertion. A hybrid — REAPI's CAS and action cache with a small batched-lookup extension — is a coherent option and is the most likely route back to this alternative, along with a batched `ActionCache` appearing upstream.

**Do nothing.** Native module builds stay per-machine. Survivable, and it is the status quo, but it leaves a real cost on every CI run and leaves the Nix-shaped proposals as the only answer for teams that care.

## Implementation

**`pnpm/pnpm` (TypeScript CLI and pacquet).** Split the dependency-state key into an input key and advertised compatibility constraints, and add the builder profile — this is the largest client change and the one everything else depends on. Then a remote tier on the side-effects lookup, grouped per registry, feeding the existing `sideEffectsMaps` path so nothing downstream changes. Manifest validation must sit at the boundary, before any remote-supplied path reaches the importer. The `allowBuild` gate at `pnpm11/building/during-install/src/index.ts:111` must be confirmed to cover the remote path, not just the local one. Both stacks, per the repository's parity rule.

**`pnpr`.** The batched endpoint on the existing resolver surface alongside `/-/pnpr/v0/resolve`, capability-advertised through the `/-/pnpr` handshake, ideally answerable within a resolve response. Storage in the disposable `cache` root. Reuse of `RoutePolicy` and `packages:` rules. A publish path with its own policy verb and signature verification.

**Documentation.** The protocol, the tag format, and the builder-profile vocabulary are a public contract from the moment they ship and must be specified so registries other than pnpr can implement them.

Ordering: the input-key/compatibility split and the builder profile are prerequisites. Nothing may cross a machine boundary before they exist, because until then the cache cannot state what an artifact is valid for.

## Prior Art

**Turborepo and Nx** cache first-party task outputs over a plain HTTP API anyone can self-host, with optional HMAC-SHA256 artifact signing and verification failure treated as a cache miss. Two lessons carry over: a documented HTTP spec is sufficient for a healthy ecosystem of implementations, and treating verification failure as a miss rather than an error is the right default.

The analogy breaks in two places. Their threat model is a compromised cache operator or transport, which signing answers because the key never leaves the user's machines; ours additionally includes an artifact that is the legitimate output of a third-party install script nobody read. And their cache keys are user-declared in configuration, so a wrong key is user misconfiguration costing a stale build — pnpm computes the key on the user's behalf, which is why the input-closure problem is pnpm's liability rather than the user's.

**moon** ran its own hosted cache service, Moonbase, then sunset it and implemented REAPI so users could run off-the-shelf servers. This is a JS-ecosystem tool that built a bespoke protocol and walked it back, which is why the REAPI alternative is argued rather than dismissed.

**Python wheels** are the closest analogue: the one mainstream ecosystem that made the built form of a third-party dependency a first-class distributable, served from the same index under the same authentication as source. Its compatibility-tag model — the artifact advertises what it is valid on, the installer selects from an ordered set of tags it supports — is the direct model for the tag design here, and manylinux's encoded glibc floor is the worked example of getting a floor right.

**Nix binary caches and Homebrew bottles** achieve stronger guarantees by making builds reproducible and sandboxed by construction, and by signing substituter output. pnpm can assume neither of npm install scripts, which is exactly why builder profiles, eligibility, and signed provenance stand in for reproducibility here.

## Unresolved Questions and Bikeshedding

- **How coarse should a builder profile be?** Too fine and nothing ever hits; too coarse and it stops predicting the artifact. A named, versioned profile is proposed over an open property bag, but the vocabulary is undesigned.
- **Who asserts eligibility, in practice?** Publisher, registry operator, and consuming organization are all plausible, with different incentives. A publisher-asserted flag is the easiest to ship and the easiest to get wrong.
- **Beyond libc.** Compiler and C++ standard library identity may matter for some native modules and not others. Whether that belongs in the tag, the profile, or eligibility is open.
- **Non-relocatable builds.** Some builds embed absolute paths or host state and are never shareable. Detection, a `package.json` marker, or eligibility-by-default-off? [pnpm/pnpm#5002](https://github.com/pnpm/pnpm/issues/5002) asks for the local form of this.
- **Sandboxed and unsandboxed artifacts.** If dependency scripts eventually run sandboxed ([pnpm/pnpm#13772](https://github.com/pnpm/pnpm/issues/13772)), provenance records which produced an entry — but whether a client should accept the other kind is a policy question.
- **Signature trust roots.** Who signs, how keys are distributed and rotated, and whether a registry may sign on a builder's behalf.
- **Batching versus concurrent lookups.** The batched design should be benchmarked against concurrent per-package lookups over HTTP/2 rather than assumed faster, including at small candidate counts where a single request may not pay for itself.
- **Inline small files.** Whether small files ride inline in the batch response instead of costing a second blob fetch is measurable and should be measured.
