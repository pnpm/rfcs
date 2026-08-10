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

### The build-input problem

**The dependency-state key is not a function of everything a build depends on, and for arbitrary npm packages it cannot be made one.**

This is the central constraint, and it is easy to understate. `calcDepState` captures the dependency graph, patches, OS, arch, and Node major. A lifecycle script may additionally depend on environment variables, compiler and linker versions, the C++ standard library, Python, platform SDKs, system headers and shared libraries, CPU feature detection, network responses, the wall clock, and arbitrary files on the host. libc is one known omission; it is not the only one, and enumerating the rest is not a finite task.

The consequence is that two honest builders can produce materially different outputs under the same key. A cache that assumes otherwise is not merely inefficient — it silently serves the wrong artifact. This is precisely where the Nix analogy breaks down: Nix can treat a store path as a pure function of its inputs because it *models* the inputs and builds in a restricted environment. npm install scripts do neither.

### Bounding the problem instead of closing it

The input set cannot be closed for arbitrary npm packages, so this RFC does not try to. It bounds the blast radius instead, by three means — none of which is cache-key fragmentation.

**A trust domain, which is where the residual risk is accepted.** The cache is scoped to one organization, and that organization owns the correctness of artifacts within it, exactly as it already owns the correctness of its own CI output. If two of Acme's builders disagree about how to compile a package, that is Acme's inconsistency to fix, and Acme is the party positioned to notice and fix it. This is the same bargain Turborepo, Nx, and moon offer, and it is why they need no environment model at all. It is also why this RFC declines an *anonymous, multi-builder* public cache, where anyone may contribute an artifact for anyone's package: there the residual risk has no owner, and the mechanisms that would be needed instead — reproducibility or attestation — are a different and much larger project. A public cache with a named owner is a different proposition, and is treated as a deliberate future direction below rather than as something excluded.

**Compatibility constraints, which are the hard gate.** What actually has to be enforced per artifact is not "was this built in an environment I approve of" but "will this run correctly here". That is what the compatibility tags in the next section do, and they do it as floors, so an artifact built against an older libc stays valid on newer systems — including when the builder's own image is patched.

**A package eligibility contract.** Sharing is opt-in per package, not blanket, and the default for a package nobody has vouched for is to build locally. Eligibility may be asserted by the publisher, by the registry operator for packages they have evaluated, or by the consuming organization for its own dependencies. This is where packages that are inherently unshareable are excluded — those whose builds read the clock, the network, or the host's mutable state — and no key design substitutes for it.

**The builder profile is provenance, not a key component.** An artifact records the environment it was produced in: a container image digest, the target architecture baseline, and the build environment as a **canonical name-to-value map**, with every undeclared variable denied. Names alone are insufficient — `CFLAGS=-march=x86-64` and `CFLAGS=-march=native` satisfy the same allowlist and produce artifacts that are not interchangeable. Variables carrying secrets are denied rather than recorded, since provenance is served to clients and a name allowlist that merely permits them would leak them.

A flag such as `-march=native` makes the produced artifact's real requirements unknowable from the environment alone, which is a *compatibility tag* problem rather than a profile one: such a build must either declare a correspondingly narrow architecture tag or be ineligible. The profile itself travels in the signed block for debugging, incident response, and optional organizational policy.

It deliberately does **not** participate in the input key. Doing so would fragment the cache along an axis that mostly does not affect correctness: a routine base-image security patch changes the digest and would invalidate every artifact built under it, in precisely the CI scenario the feature exists to serve. It is also a producer-side claim rather than a verifiable property — nothing in an artifact proves where it was built — so treating it as a correctness mechanism would overstate what it can carry. The signature makes the claim attributable; the compatibility tags are themselves a signed producer assertion, constraining where an artifact may be used rather than establishing that it is safe. Safety rests on the trusted builder and the eligibility contract.

**Sharing requires behavioural equivalence, not bit-identity.** Any artifact for input key K must work correctly on any machine matching its compatibility constraints. Embedded timestamps, link order, and hash-iteration nondeterminism are irrelevant. This is a materially lower bar than Nix's, and it is why the problem is tractable at all.

Finally, a *hermetic* sandbox with declared inputs — restricting filesystem reads, environment, network, clocks, and randomness — is the one thing that would genuinely close the input set rather than bound it. Plain sandboxing is not sufficient: [pnpm/pnpm#13772](https://github.com/pnpm/pnpm/issues/13772) is primarily about restricting authority and leaves network policy open, and a sandbox permitting network access closes nothing. Network is the largest gap in practice, since `node-gyp` fetches headers and `prebuild-install` fetches binaries, so hermeticity means `--network=none` plus pre-seeded caches — what Nix does, and what breaks packages unprepared for it. CPU feature detection is the second, which is why the architecture baseline is recorded. Nothing here depends on hermetic sandboxing shipping; it would allow the trust domain to widen later, and a hermetically sandboxed build is the strongest form of an eligible one.

### Compatibility tags are not the lookup key

An artifact built against glibc 2.17 should serve a glibc 2.39 machine. That is a floor, not an identity — and a floor cannot be expressed by hashing the consumer's own platform into the key it looks up. A 2.39 client would request a key containing 2.39 and never find the 2.17 artifact.

Two distinct concepts are therefore required:

- **An input key** identifying what was built: the dependency graph and applied patches. This is `calcDepState` with the platform components removed, and it is what an entry is stored under.
- **Compatibility constraints** the artifact advertises: the platform tags it is valid on, expressed as floors where the underlying property admits one.

The client presents an ordered set of tags it supports and selects the best offered artifact, exactly as Python installers select among wheels rather than hashing the interpreter's exact identity into a filename ([PyPA platform compatibility tags](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/#use)). Two properties follow: an artifact serves many systems rather than one, and a tag the client does not understand is a miss, never a guess.

### Fetching

**Requests are batched per registry and issued in parallel.** Artifacts belong to the registry a package resolved from, and an install may draw from several. The client groups candidates by concrete registry and sends each registry only its own candidates — both because there is no single origin that could answer for all of them, and because broadcasting the full candidate set would leak an organization's cross-registry dependency graph to every registry it talks to.

Within a registry the lookup is one round trip for the whole install, mirroring how `POST /-/pnpr/v0/resolve` takes an entire workspace in a single request. This matters because the lookup sits immediately before the builds that are the last thing blocking install completion. Where the client is already talking to pnpr, the lookup should fold into the resolve request that is already happening, so a warm install pays no additional round trip at all.

Only packages marked `requiresBuild` are candidates — those declaring `preinstall`/`install`/`postinstall`, plus the implicit cases `pkgRequiresBuild` detects from `binding.gyp` or a `.hooks/` directory. When such a package is also patched, the patch hash is part of its input key. Packages that are *only* patched are deliberately excluded: applying a patch overlay locally is cheap, and including them would require inventing build-approval semantics for packages that have no build to approve, contradicting the `allowBuild` requirement below.

A response carries, per candidate, the *set* of variants the registry holds: each with the `added` file list as `(path, digest, mode)` triples, the `deleted` path list, the compatibility tags, and the signed provenance described below. The client selects among variants on both compatibility and signer trust, so a registry holding several builders' output for one input key is a normal case rather than a conflict. Variants per candidate and total response size are capped by the client, since the response is untrusted input parsed before any signature is checked. Sending a diff rather than a built tree keeps responses proportional to what a build changed rather than to the size of the package.

**Clients MUST recompute the digest of every downloaded blob** before it enters the content-addressable store or the importer; the signed manifest attests the digests, but nothing attests that the bytes served match them. A mismatch is a cache miss and a diagnostic, and the offending entry is quarantined so the same poisoned artifact is not re-fetched on every subsequent install.

### Remote artifacts stay labelled once stored

A verified artifact must not become indistinguishable from a locally built one the moment it lands in the store. The store is global and shared across every project on a machine, so an unlabelled remote entry would be silently reused in situations where it should not be: after its signing key was revoked; after remote artifacts were disabled; by a different project or tenant that happens to share the store; or for a package that now resolves through a different registry entirely — which would defeat the tenant namespacing this design otherwise depends on.

The existing `sideEffectsMaps` type carries only `added` and `deleted`, so remote entries cannot simply be fed into it and forgotten. Remote artifacts therefore retain **persistent origin metadata**: the trust domain and registry they came from, the signer key identifier, the builder profile, the signed envelope, and their verification status.

**Trust policy is applied at every reuse, not only at download.** An artifact whose key has since been revoked, whose registry is no longer configured for remote artifacts, or whose trust domain does not match the current project's, is not reused for this install, and the package builds locally. It is not deleted: the store is global, and another project may still legitimately trust the same artifact. Whether this is implemented as fields on the existing entries or as a separate local index for remote artifacts is an implementation choice; the requirement is that origin survives storage and is consulted on every use.

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
- **Provenance is signed, not merely recorded.** A signature binds the protocol version, the source tarball integrity, the input key, the builder profile, the compatibility constraints, the manifest digest, the builder identity, and the tenant/trust-domain identifier — the last so that an artifact signed for one domain cannot be replayed into another. Unauthenticated provenance fields are useful for debugging and useless against substitution; only the binding makes an artifact attributable.
- **The signing key must be independent of the registry.** This is the load-bearing constraint, not a deployment detail. The threat this section exists to address is a registry — or an uploader it authorized — substituting built files. A registry that signs artifacts itself defends against nothing it is not already trusted for, and reduces the signature to a transport checksum. The signer is therefore a **pluggable trust root**, and the protocol must not assume which kind. V1 defines exactly one kind — **an organization-configured CI signing key per registry, held outside the registry server**, with key identifiers, rotation, and revocation defined from the start rather than retrofitted. Publisher identity is the second kind and is deliberately not precluded (see below); baking the organizational key into the protocol shape would force a breaking change to add it.
- **Several builders per input key is the normal case.** Two CI pipelines can legitimately produce entries for the same package, profile, and tags. Builder identity therefore participates in variant identity: the registry stores the variants and the client picks one signed by a key it trusts, rather than the registry silently choosing on the client's behalf.
- **The existing allowlist gate is preserved.** A remote artifact is accepted only for a package `allowBuild` already permits. Fetching a prebuilt artifact avoids *rebuilding* something you approved; it is not a route around approving it.

### Publishing

Writing artifacts is a separate grant from publishing packages, with its own policy verb. "May publish `foo`" and "may publish a build of `foo`" differ in kind: a package is reviewed source, a build is the output of an install script. Publication is opt-in per registry and off by default; the expected deployment is a team's own registry populated by that team's CI.

### Publisher-published artifacts: the intended second step

The natural extension is for a package's own publisher to ship its built forms alongside its source, exactly as PyPI serves wheels next to sdists. This RFC does not specify it, but it is the intended direction and nothing here may preclude it.

It is worth being clear that this is *more* defensible than the team cache rather than less. The trust relationship already exists: a consumer who installs a package already executes that publisher's install script, and accepting a prebuilt artifact signed by the same identity is a smaller grant than arbitrary code execution during install — the artifact is inert data placed in a directory. It also needs no new trust root, since the identity authenticating the artifact is the one already authenticating the tarball, for which npm provenance and OIDC-based trusted publishing exist. And as the Prior Art section notes, the ecosystem already does this through `prebuild-install` and `node-pre-gyp`, without integrity pinning, signatures, or offline support.

Two things make it the second step rather than the first:

- **It depends on registry adoption pnpm does not control.** A team cache ships with pnpm and pnpr alone. Publisher-published artifacts for the public ecosystem require `registry.npmjs.org` to accept and serve them, which is a sequencing constraint rather than a design objection.
- **Compatibility tags must be exact.** A team cache targets machines its operator knows and can correct in an afternoon; a publisher targets every consumer, so tag floors have to be right the first time. This is manylinux's problem at full difficulty and deserves the round the tag format is already owed.

The V1 decision that keeps the door open is the pluggable trust root in the trust section: publisher identity must be addable as a second signer kind without a protocol break.

### What pnpr already provides

The design reuses pnpr's routing and storage model rather than adding a parallel one.

- **Private-entry namespacing.** `RoutePolicy` already namespaces private resolution-cache entries under an HMAC keyed on `resolution_cache_secret` so they are not correlatable offline. The mechanism carries over — but **its input does not.** Source visibility must not determine artifact scope: a public source package built by one organization still yields an artifact that can reveal builder identity, embed organization-specific output, or simply differ from another organization's build of the same public package. Artifacts are therefore namespaced by tenant and signing trust domain — which for this design are the same boundary — and publishing an artifact beyond that namespace is a separate explicit decision, never a consequence of the source tarball having been public.
- **Durability.** Artifacts are regenerable derived data: obtainable again from the tarball plus a build, safe to wipe, self-healing. (Regenerable, not reproducible — a rebuild need only be behaviourally equivalent.) That is the contract of pnpr's disposable `cache` root as distinct from the authoritative `storage` root, so artifacts need no new storage tier. The existing S3 block remains available for operators who want them shared across replicas.
- **Per-package policy.** "Serve artifacts for `@ourco/**`, never for `**`" is a `packages:` pattern rule using the existing specificity selection.
- **Existence is not leaked.** `access` already gates reads with denied callers masked as not-found, which matters because artifact existence would otherwise reveal an organization's dependency graph and input keys.

## Rationale and Alternatives

**An external package provider ([pnpm/pnpm#13639](https://github.com/pnpm/pnpm/pull/13639)).** Delegate materialization of every package to a third-party executable, optionally backed by the Nix store. This delivers shared build outputs, at the cost of a permanent stdio protocol, a second install mode incompatible with the global virtual store, the pnpr server, and the hoisted linker, and the silent voiding of `onlyBuiltDependencies`, `--ignore-scripts`, and `approve-builds` because the provider decides what builds. It also serves only users who have already adopted Nix. Notably, it does *not* escape the build-input problem — it relocates it into the provider, where Nix's restricted build environment is what actually solves it. This RFC bounds the problem instead, via a trust domain, compatibility constraints, and eligibility.

**A standalone cache URL, in the shape of Turborepo's Remote Cache.** A `sideEffectsCacheUrl` pointing at a dedicated service. Rejected because it duplicates the entire registry authentication surface — tokens, helpers, certificates, proxies — and introduces a wholly new principal permitted to write into `node_modules`, without even the partial containment of the registry relationship. Its cost is that it does nothing for users installing directly from `registry.npmjs.org`; the answer there is a proxying pnpr, which is already how such teams deploy.

**Adopt the Bazel Remote Execution API v2 instead of defining a protocol.** REAPI is a real standard with off-the-shelf servers (bazel-remote, Buildbarn, BuildBuddy), and moon adopted it after sunsetting its own hosted cache. It covers more than it first appears: `Platform` properties are explicitly intended for "hardware, operating system, or compiler toolchain" and are implicitly part of the Action digest; `ExecutedActionMetadata` records the producing worker; `GetActionResult`/`UpdateActionResult` work without the Execution service; and because `Command` declares which outputs are captured, an `ActionResult` need not carry a full tree — pnpm's added-file set could be declared directly as outputs.

It is declined on a single objection, which is sufficient on its own: **its lookup model is exact-digest, and this design requires compatibility selection.** `GetActionResult` answers "is there a result for precisely this Action digest". It cannot answer "give me the best artifact compatible with this ordered tag set, signed by a key I trust" — which is the question a floor-based tag model with multiple builders must ask. Emulating it means the client enumerating one candidate digest per supported tag in preference order and issuing a lookup for each, multiplying request count by tag-set size, in a service with no batched `ActionCache` read at all (`GetActionResultRequest` carries a single `action_digest`; only the CAS service batches). `Platform` property matching does not rescue this: it governs execution placement, not cache lookup, which stays exact-digest regardless.

One narrower mismatch is worth recording but is not load-bearing: `Action` requires a `command_digest` referencing a `Command` in CAS, while pnpm has no command, so the key would derive from a synthesized `Command` no worker can execute, making its canonicalization an unintended compatibility surface. Deletions, by contrast, are *not* an obstacle — a pnpm manifest carrying its own `deleted` list can be stored as a single opaque `ActionResult` output blob, since CAS contents are uninterpreted.

Raw round-trip count is deliberately *not* claimed as the deciding factor. gRPC multiplexes concurrent RPCs over HTTP/2, so N lookups are not N serialized round trips, and whether batching wins on wall-clock is a question for measurement rather than assertion. A hybrid — REAPI's CAS and action cache with a small batched-lookup extension — is a coherent option and is the most likely route back to this alternative, along with a batched `ActionCache` appearing upstream.

**Do nothing.** Native module builds stay per-machine. Survivable, and it is the status quo, but it leaves a real cost on every CI run and leaves the Nix-shaped proposals as the only answer for teams that care.

## Implementation

**`pnpm/pnpm` (TypeScript CLI and pacquet).** Split the dependency-state key into an input key and advertised compatibility constraints — this is the largest client change and the one everything else depends on. Record the builder profile as signed provenance, deliberately outside the key. Then a remote tier on the side-effects lookup, grouped per registry. This cannot simply feed the existing `sideEffectsMaps` path unchanged: that type carries only `added` and `deleted`, so remote entries need persistent origin metadata alongside them (or a separate local index), and the reuse path needs a trust-policy check it does not have today. Manifest validation must sit at the boundary, before any remote-supplied path reaches the importer. The `allowBuild` gate at `pnpm11/building/during-install/src/index.ts:111` must be confirmed to cover the remote path, not just the local one. Both stacks, per the repository's parity rule.

**`pnpr`.** The batched endpoint on the existing resolver surface alongside `/-/pnpr/v0/resolve`, capability-advertised through the `/-/pnpr` handshake, ideally answerable within a resolve response. Storage in the disposable `cache` root. Reuse of `RoutePolicy` and `packages:` rules. A publish path with its own policy verb and signature verification.

**Documentation.** The protocol, the tag format, and the provenance block are a public contract from the moment they ship and must be specified so registries other than pnpr can implement them.

Ordering: the input-key/compatibility split is the prerequisite. Nothing may cross a machine boundary before it exists, because until then the cache cannot state what an artifact is valid for.

## Prior Art

**`prebuild-install`, `node-pre-gyp`, and `prebuildify`** are the closest prior art of all, because npm already has a publisher-published binary cache — implemented as postinstall scripts that download artifacts from GitHub Releases or S3. It works well enough to be ubiquitous, which is the strongest available evidence that the demand is real. It also has no lockfile integrity pinning for what it downloads, no signatures, a hard requirement for network access at install time, and it fails under `--ignore-scripts` and offline installs. A first-class mechanism would replace this rather than add to it.

**Turborepo and Nx** cache first-party task outputs over a plain HTTP API anyone can self-host, with optional HMAC-SHA256 artifact signing and verification failure treated as a cache miss. Two lessons carry over: a documented HTTP spec is sufficient for a healthy ecosystem of implementations, and treating verification failure as a miss rather than an error is the right default.

Most instructive for this design: **none of the three models the build environment at all.** They can omit it because their caches are team-scoped and their artifacts first-party — the organization already owns their correctness, and no one is defending against another organization having built something wrong. That is the precedent for keeping the builder profile out of the input key here, and for scoping V1 to a single trust domain rather than to an anonymous public cache the environment model would then have to carry.

The analogy breaks in two places. Their threat model is a compromised cache operator or transport, which signing answers because the key never leaves the user's machines; ours additionally includes an artifact that is the legitimate output of a third-party install script nobody read. And their cache keys are user-declared in configuration, so a wrong key is user misconfiguration costing a stale build — pnpm computes the key on the user's behalf, which is why the residual build-input risk is pnpm's to bound rather than the user's to notice.

**moon** ran its own hosted cache service, Moonbase, then sunset it and implemented REAPI so users could run off-the-shelf servers. This is a JS-ecosystem tool that built a bespoke protocol and walked it back, which is why the REAPI alternative is argued rather than dismissed.

**Python wheels** are the closest analogue: the one mainstream ecosystem that made the built form of a third-party dependency a first-class distributable, served from the same index under the same authentication as source. Its compatibility-tag model — the artifact advertises what it is valid on, the installer selects from an ordered set of tags it supports — is the direct model for the tag design here, and manylinux's encoded glibc floor is the worked example of getting a floor right.

**Nix binary caches and Homebrew bottles** achieve stronger guarantees by making builds reproducible and sandboxed by construction, and by signing substituter output. pnpm can assume neither of npm install scripts. That is why this RFC bounds rather than closes the problem — and why it stays inside a single trust domain, which is the scope in which bounding is honest.

## Unresolved Questions and Bikeshedding

- **Who runs the builds.** If a team's CI builds and uploads, this RFC's scope holds. If pnpr builds, pnpr becomes a build service — that is remote *execution*, declared a non-goal below, and it is REAPI's actual domain. The distinction should be decided deliberately rather than arrived at.
- **Who asserts eligibility, in practice?** Publisher, registry operator, and consuming organization are all plausible, with different incentives. A publisher-asserted flag is the easiest to ship and the easiest to get wrong.
- **How much of the publisher path to specify now.** The trust root is pluggable so publisher identity can be added without a protocol break, but it is an open question whether the tag format and eligibility assertion should be designed against the publisher case from the start — where they are much harder — or allowed to be shaped by the team cache and generalized later, at the risk of a format that does not stretch.
- **Beyond libc.** Compiler and C++ standard library identity may matter for some native modules and not others. Since the profile is not a gate, the question is whether any of it needs to become a compatibility tag, or whether it is eligibility's job.
- **Non-relocatable builds.** Some builds embed absolute paths or host state and are never shareable. Detection, a `package.json` marker, or eligibility-by-default-off? [pnpm/pnpm#5002](https://github.com/pnpm/pnpm/issues/5002) asks for the local form of this.
- **Sandboxed and unsandboxed artifacts.** If dependency scripts eventually run sandboxed ([pnpm/pnpm#13772](https://github.com/pnpm/pnpm/issues/13772)), provenance records which produced an entry — but whether a client should accept the other kind is a policy question.
- **Key distribution.** V1 fixes the shape — one organization-configured CI key per registry, held outside the registry server, with identifiers, rotation, and revocation. How a client learns and pins those keys, and whether revocation needs to be online, is not settled. That a registry may *not* sign on a builder's behalf is settled, and is stated in the trust section rather than here.
- **Batching versus concurrent lookups.** The batched design should be benchmarked against concurrent per-package lookups over HTTP/2 rather than assumed faster, including at small candidate counts where a single request may not pay for itself.
- **Inline small files.** Whether small files ride inline in the batch response instead of costing a second blob fetch is measurable and should be measured.
