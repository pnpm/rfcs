# Locked build environments for git-hosted dependencies

## Summary

When pnpm installs a git-hosted dependency that has to be built, it recreates the dependency's own development workflow: it installs the dependency's `devDependencies` with the package manager the dependency asks for (npm, Yarn, Bun, or pnpm itself) and runs its `prepare` scripts. Which version of that package manager runs is decided per machine — often from a floating spec sniffed from the dependency's lockfile — and the build always runs on whatever Node.js the host has. This RFC proposes to (1) resolve the build tools of a git-hosted dependency at install time like any other dependency and record them in the consuming project's lockfile — their integrity in ordinary `packages` entries, and the edges from each git-hosted package to the tools that build it in a new top-level `buildSnapshots` map, a build graph kept apart from the install graph in `snapshots` — and make that record authoritative on subsequent installs; and (2) provision the Node.js runtime a dependency pins via `devEngines.runtime`, the same way the package manager is already provisioned from `devEngines.packageManager` / `packageManager`.

The lockfile change is additive. The breaking part is behavioral: a frozen install of a build-requiring git dependency with no record fails instead of resolving its tools live, so the change ships in a pnpm major.

## Motivation

### The package manager used for the build is pinned per machine, not per project

pnpm v12 decides which package manager prepares a git-hosted dependency from three sources, in order:

1. A pin in the dependency's `devEngines.packageManager` or `packageManager` field — an exact version or a range (e.g. `yarn@^4`).
2. A sniff of the lockfile the dependency ships: `yarn.lock` implies Yarn `1` or `>=2` (depending on whether the file carries Berry's `__metadata` block), `bun.lock` implies Bun, and so on — with no version constraint beyond the Yarn line.
3. npm, when nothing else applies.

When the dependency pins a version, or the host cannot satisfy what it needs, pnpm provides the package manager itself: the npm-published ones are resolved through the trusted package-manager registries and verified against npm's signature, Yarn 6 and Bun come from release archives pinned by a publisher checksum, and the resolved version is remembered in a per-package-manager environment lockfile under the pnpm home. Otherwise, the host's copy prepares the dependency, whatever its version.

Only an exact pin is deterministic across machines, and only transitively: the commit SHA in the consumer's lockfile pins the manifest, and the manifest pins the version. A range, a sniffed spec, or a host copy resolves to whatever that machine resolved first or happens to have installed. Most real git-hosted dependencies fall into one of those cases, since most packages don't declare `packageManager`. The pin in the pnpm home doesn't travel with the project, and nothing in the project records what was used.

The consequences:

- **Builds are not reproducible across machines or time.** Two machines preparing the same git dependency can build with different Yarn versions, whose hoisting and dedupe differences produce different dependency trees for the build — and potentially different build output. Because git resolutions carry no integrity checksum for the built result, the divergence is undetectable. This is a classic source of "works locally, fails in CI."
- **`--frozen-lockfile` is not actually frozen.** A frozen install on a fresh CI machine resolves the package manager live from the registry and then *executes* it, with no lockfile diff for anyone to review. A malicious release matching `>=2` that is signed by a compromised publisher account reaches script execution on CI. Approving the dependency in `allowBuilds` doesn't help: the approval is for the dependency's scripts, not for the package manager that runs them. This contradicts the promise a frozen install makes, and it is a weaker guarantee than pnpm gives every other package it downloads.
- **The host path bypasses verification entirely.** When the host's copy satisfies an unpinned spec, the build runs on an executable pnpm never verified, at a version nobody chose.

### The build runs on an arbitrary Node.js

The prepare currently runs with the host's Node, unconditionally. If the dependency's authors develop and test on Node 24 and the host runs Node 20, the build can fail outright (build tooling with `engines` floors is common) or quietly produce different output. pnpm already knows how to resolve, download, and verify any Node version, and already records one in a lockfile as a `node@runtime:<version>` package with per-platform integrity — it just doesn't apply that in the one place where pnpm runs *someone else's* development workflow.

The expected outcome of this RFC: preparing a git-hosted dependency uses the same package manager and runtime on every machine that shares the lockfile, those tools are integrity-verified downloads named by the lockfile, a frozen install performs no live resolution of executables, and a git dependency that pins its dev runtime builds with that runtime instead of failing on hosts that don't have it.

## Detailed Explanation

### 1. A build graph: `buildSnapshots`

The lockfile already separates *what* is pinned from *how it is used*: `packages` holds each artifact's resolution and integrity, and `snapshots` places those artifacts in the install graph — the graph materialized into `node_modules`. This RFC adds a second graph over the same `packages`: `buildSnapshots`, the graph of tools pnpm runs to prepare git-hosted dependencies.

```yaml
packages:

  '@yarnpkg/cli-dist@4.9.2':
    resolution: {integrity: sha512-…}
    hasBin: true

  example@git+https://github.com/org/example.git#5c8f9a…:
    resolution: {type: git, repo: …, commit: 5c8f9a…}

  node@runtime:24.6.0:
    resolution:
      type: variations
      variants:
        - resolution:
            archive: tarball
            bin:
              node: bin/node
            integrity: sha256-…
            type: binary
            url: https://nodejs.org/download/release/v24.6.0/node-v24.6.0-darwin-arm64.tar.gz
          targets:
            - cpu: arm64
              os: darwin
        # … one variant per platform

snapshots:

  example@git+https://github.com/org/example.git#5c8f9a…:
    dependencies: …

buildSnapshots:

  '@yarnpkg/cli-dist@4.9.2': {}

  example@git+https://github.com/org/example.git#5c8f9a…:
    dependencies:
      node: runtime:24.6.0
      yarn: '@yarnpkg/cli-dist@4.9.2'

  node@runtime:24.6.0: {}
```

- **A root per git-hosted package that needs a build.** Its key is the package's key in `packages`, and its `dependencies` are the tools that prepare it. The build belongs to the package, not to any one peer-resolved snapshot of it, so the root carries no peer suffix.
- **A node per tool.** Each tool has a `packages` entry for its resolution and a `buildSnapshots` entry for its edges, exactly as a package in the install graph has a `packages` entry and a `snapshots` entry. Most tools have no edges — npm bundles its dependencies, `@yarnpkg/cli-dist` is a single bundle, and runtimes and archive-shipped package managers are single archives — so their node is `{}`, the way `node@runtime:<version>: {}` already appears in `snapshots`. A tool that does have dependencies uses the same edges an install snapshot does: a downloaded pnpm is `@pnpm/exe`, whose binary comes through platform-specific `optionalDependencies`. Edges in `buildSnapshots` point only at other `buildSnapshots` nodes.
- **No peers.** Build tools never take part in peer resolution, so a tool has exactly one build snapshot and its key is its package key.

**Why a separate graph.** Every walk over `snapshots` — materialization, hoisting, bin linking, the current lockfile — means "install this into `node_modules`". Build tools must never be installed there: they are fetched only when a build runs and executed from the store. Keeping them out of `snapshots` means none of those walks can reach them, without any of them having to learn an exception. A separate graph also leaves the git package's own install snapshot untouched, so its `dependencies` keep meaning what they mean everywhere else (see Rationale and Alternatives).

**Why `packages` is shared.** A tool's resolution and integrity are facts about a published artifact, independent of which graph uses it, so they live where every other artifact's do:

- **Integrity comes for free.** A package manager published to npm is locked by its tarball integrity, exactly as a regular dependency is. The record doesn't trust the registry to serve the same bytes for the same version any more than pnpm does for a leaf dependency.
- **Platform binaries are covered.** A runtime, and a package manager that ships as a platform archive (Bun, Yarn 6), is locked by the existing `variations` resolution, which pins every platform's archive by integrity. A single integrity on a wrapper package would pin only the wrapper and leave the binary that actually runs unpinned. Because every platform is recorded at resolution time, the record is the same no matter which machine wrote it.
- **One entry per artifact.** Ten git dependencies prepared with Yarn 4.9.2 share one `packages` entry, one build node, and one store copy. A runtime that is also the project's own `devEngines.runtime` has one `packages` entry used by both graphs.
- **The alias form is the existing one.** Yarn Berry is published as `@yarnpkg/cli-dist`, so it is recorded as `yarn: '@yarnpkg/cli-dist@4.9.2'`, the way any npm alias is.

This changes what a `packages` entry means: no longer "a node of the install graph", but "an artifact the lockfile pins", which one or both graphs use. Code that walks `snapshots` is unaffected. Code that iterates `packages` wholesale as a stand-in for the install graph has to iterate what `snapshots` reaches instead; the Implementation section lists the places that do.

**Lifetime.** A `packages` entry is kept while either graph references it. A build root is kept while its git-hosted package is in `packages` and needs a build; when the package leaves the lockfile, its root goes with it, then any tool node nothing references any more, then the `packages` entries only those nodes used.

**Fetching.** Tools are fetched into the content-addressable store only when a git dependency actually has to be prepared — its built artifact is missing from the store and from the side-effects cache — and run from there, the way `pnpm dlx` runs a package manager today. A warm install downloads nothing for them.

The build graph names only what pnpm itself downloads and executes to prepare the dependency. The dependency's own `devDependencies`, installed by that package manager inside the checkout, are governed by the lockfile the dependency ships at that commit and are out of scope.

### 2. Resolution and reuse rules for the package manager

**First resolution** (the git dependency enters the lockfile, or is re-resolved by `pnpm update`): pnpm reads the wanted spec from the dependency's manifest or lockfile, exactly as today, and resolves it with pnpm's normal rule: the `latest` dist-tag if it satisfies the spec, otherwise the highest satisfying version. The result is recorded in the package's build root even when the host has a copy that would satisfy the spec, so the record never depends on which machine wrote it. When the dependency's `packageManager` field carries a Corepack hash (`yarn@1.22.22+sha512.…`), the hash is checked against the artifact being recorded where it covers the same bytes, and a mismatch fails resolution.

**Subsequent installs**: the record is authoritative.

- If the host has the package manager on `PATH` **at exactly the recorded version** (probed via `--version`, as today), the host copy is used.
- Otherwise pnpm provisions the recorded version and verifies it against the recorded integrity. Today's rule — the host's copy prepares an unpinned dependency whenever it satisfies the spec — is dropped, because it silently reintroduces cross-machine divergence.
- A **frozen install** never resolves live: it uses a matching host copy, uses a copy already in the store, or downloads exactly the recorded version and verifies it. A git-hosted package that requires a build but has no build root is treated like any other lockfile staleness under `--frozen-lockfile`: the install fails with an error telling the user to run a regular install to update the lockfile.

**Stickiness** follows the git resolution itself: the record refreshes only when the git dependency re-resolves (`pnpm update`), exactly like the commit SHA. Ordinary installs cause no lockfile churn.

**A release committed to the repository.** A Yarn Berry repository may commit its release under `.yarn/releases/` and point `yarnPath` at it in `.yarnrc.yml`; the release that runs is then pinned by the commit, and the recorded Yarn only launches it. The record is still written, because the launcher is still code pnpm downloads and executes, but whether the build is reproducible no longer depends on it.

### 3. Runtime selection

The dependency's manifest gets a say in which runtime runs its build:

1. **`devEngines.runtime` names a runtime pnpm can provision** → pnpm resolves it, records it as a `<name>@runtime:<version>` edge of the package's build root, and runs the inner install and the `prepare` scripts on it. This is symmetric with `devEngines.packageManager`: the field describes the dependency's development environment, which the prepare recreates.
2. **Otherwise** → the host runtime is used, and nothing is recorded.

Only an explicit dev pin selects the runtime, so the record depends on the dependency's manifest at the locked commit and never on the host. A dependency whose `engines.node` the host fails is not provisioned for: provisioning only on hosts that fail the range would record a runtime on some machines and not others, and two contributors would produce different lockfiles for the same dependency. Such a build fails as it does today, with a hint naming the range. `engines.node` is a consumer-facing compatibility range (typically `>=18`); treating it as a build pin on every host would mean "always latest Node," which nobody intends.

**Scope boundary — ABI.** The provisioned runtime applies only to the *inner* workflow inside the git checkout: the dependency's own dependency install and its `prepare`/`prepack` scripts. The dependency's `install`/`postinstall` scripts, which run later in the consuming project, keep the project's runtime, because their output (typically compiled native addons) must match the ABI of the Node that will load it. A git dependency that compiles native code *during prepare* has an ABI mismatch problem with or without this RFC (the built result is cached and reused regardless of the consumer's Node); this RFC deliberately does not attempt to solve it, and the boundary above at least never makes it worse.

### 4. Store and cache keying

The store-index key of a built git-hosted dependency, today derived from the package ID and whether it was built, also covers its build root: the tools it reaches in `buildSnapshots` and their integrity. `pnpm update` can keep the commit and move the recorded package manager; without the record in the key, the artifact built by the old environment would be reused and the lockfile would describe a build that never happened. The same applies to the input key of the shared side-effects cache ([RFC 0007](./0007-shared-side-effects-cache.md)): an artifact built with Yarn 4.9.2 is not the artifact built with Yarn 4.10.0.

### 5. Failure behavior

Everything in the build graph was demanded either by the dependency or by the lockfile, so failing to provide it fails the install with an actionable error, as a pinned package manager does today. Nothing falls back to the host silently. Offline installs succeed whenever the recorded tools are already in the store or on the host at the exact version.

### 6. Compatibility and migration

The lockfile change is additive: a new top-level key, plus `packages` entries of a shape the lockfile already has. It ships with a minor `lockfileVersion` bump, not a major one. The breaking part is behavioral, so the change ships in a pnpm major:

- **Frozen installs of an upgraded project fail until the lockfile is rewritten.** Every existing lockfile lacks the build graph, so a frozen install of a project with a build-requiring git dependency fails with the staleness error from section 2. Run `pnpm install` once and commit the result. Lockfiles without build-requiring git dependencies are unaffected.
- **An older pnpm drops the record loudly, not silently.** An older pnpm ignores `buildSnapshots` when reading and drops it when it rewrites the lockfile, pruning the tool entries in `packages` that nothing in `snapshots` references. The deletion shows up in the lockfile diff, and the next frozen install with a current pnpm fails with the staleness error. Nothing is lost that a regular install doesn't restore.
- **Hosts that used to prepare with their own package manager may now download one.** A host whose Yarn satisfies the spec but not the recorded version gets a provisioned copy. This is the point of the RFC, and it goes in the release notes.

## Rationale and Alternatives

1. **Do nothing; rely on `packageManager` exact pins (the Corepack model).** Works only for dependencies that pin, which most don't; the sniffed and range cases keep floating. It also leaves the frozen-install hole open, which is the most serious of the problems.

2. **Keep the status quo: pin per machine in the environment lockfile under the pnpm home.** This is what pnpm v12 does today. It gives per-machine stability, but it doesn't travel with the project: CI and a new laptop resolve fresh, and there is no reviewable diff when the version changes. The project lockfile is the only artifact with the right ownership and review workflow.

3. **Write the build tools into the git package's existing snapshot `dependencies`.** This needs no new key: `engines.runtime` entries are already turned into `node: runtime:<version>` dependencies there. But a dependency in that field means something different — it is linked into the package's `node_modules`, and a runtime there is the runtime the package runs on, including its `postinstall`. Recording a dev-only runtime there would compile native addons for the wrong ABI, and a frozen install, which links from the lockfile without reading manifests, has no way to tell a build tool from a real dependency.

4. **Put the tools in the install graph behind a new edge kind.** A `buildDependencies` field on the git package's snapshot, pointing at tool entries in `packages` and `snapshots`. Every walk over `snapshots` would then have to learn that this edge retains entries without materializing them. A separate graph gets the same record with no exception in any of those walks.

5. **A self-contained map for the tools, outside `packages`.** A top-level map holding both the tools' resolutions and their edges, with the git package's tool list on its `packages` entry. None of the code that iterates `packages` would see the tools. But it is a second catalog of artifacts beside `packages`, with its own entry shape, validation, and integrity checks, and an artifact used by both graphs (a runtime pinned by the project and by a git dependency) would be recorded twice. Keeping one catalog and adding a graph is the smaller conceptual change; the price is the handful of places that treat `packages` as the install graph.

6. **The env lockfile document, or a separate file.** The env document holds pnpm's own tooling, read early to select the pnpm version; a transitive dependency's build tools don't belong there, and its keys would have to point into the other document. A separate lockfile file would avoid touching `pnpm-lock.yaml`, but every tool that handles the lockfile — Renovate and Dependabot, merge drivers, git-branch lockfiles, `pnpm deploy` — would have to learn a second file, and the two could drift.

7. **Lock the version but not the integrity.** Cheaper to implement and closes the reproducibility gap. But it still trusts the registry to serve the same bytes for the same version forever — a guarantee pnpm refuses to extend to any ordinary dependency. The executable that runs build scripts deserves at least the scrutiny of a leaf dependency.

8. **Lock the integrity of the built output instead of the environment.** This would detect divergence however it arose, without caring what ran the build. It fails in practice because builds are rarely byte-reproducible: timestamps, absolute paths, and nondeterministic bundler output would make the check fail on machines that built correctly. Locking the inputs is the part pnpm controls.

9. **Always provision; never use a host copy.** Maximally hermetic and simpler to reason about, but it downloads package managers even on hosts that have the exact version installed, and it penalizes the common case for no determinism gain — an exact-version host probe is equally deterministic. The proposed rule (host allowed only at exactly the recorded version) keeps the guarantee at lower cost. This remains a reasonable fallback position if exact-version probing proves unreliable in practice.

The proposed design is the only one of these that makes `--frozen-lockfile` mean what it says, keeps builds convergent across machines, and stays within pnpm's existing trust model: everything downloaded and executed at install time is named and integrity-pinned by the lockfile.

## Implementation

pnpm v12 only, per the version policy: the TypeScript v11 CLI receives no changes.

- `crates/git-fetcher` — `preferred_pm` keeps detecting the wanted spec; `prepare_package` accepts the resolved tools from the caller instead of resolving them itself, drops the "host satisfies the range" path in favor of the exact-version probe, and runs the inner workflow on the recorded runtime; `pm_shims` forwards to the recorded exact version and integrity rather than to a range.
- `crates/resolving-git-resolver` / `crates/resolving-deps-resolver` — the first-resolution path resolves the package manager and the `devEngines.runtime` pin through the existing package-manager and runtime resolvers (`engine-pm-yarn-resolver`, `engine-runtime-node-resolver`, `engine-runtime-bun-resolver`, `engine-runtime-deno-resolver`), so the record exists before the build runs.
- `crates/lockfile` and `crates/lockfile-verification` — the `buildSnapshots` map, the minor `lockfileVersion` bump, pruning that keeps a `packages` entry while either graph references it, and frozen-install validation that every build-requiring git package has a build root and that every build edge resolves to a `buildSnapshots` node with a `packages` entry.
- `crates/store-dir` — `git_hosted_store_index_key` covers the recorded build dependencies.
- Code that iterates `packages` as a stand-in for the install graph switches to what `snapshots` reaches, or skips entries only the build graph uses:
  - `crates/package-manager` tarball prefetch (`tarball_prefetch/lockfile_entries.rs`) — the speculative prefetch and the store fetch download every registry entry in `packages`, which would fetch build tools on every cold install;
  - `crates/deps-restorer` package map (`package_map.rs`) — a `packages` entry with no snapshot gets a map entry pointing at a virtual-store slot that will never exist;
  - `crates/package-manager` modules-state metadata merge (`install/modules_state/merge_metadata.rs`);
  - `crates/deps-inspection` (`graph.rs`, `pkg_info.rs`) — `pnpm list`/`pnpm why` show build tools only when asked for them.

  Walks over `snapshots` — `materialization_closure`, `filter_by_importers`, the current lockfile — need no change: the build graph is not in `snapshots`.

**Effects and risks:**

- First resolution gains one resolution per distinct package-manager spec and runtime pin (cached; in practice one or two per project).
- Recording every platform variant of a runtime or archive-shipped package manager makes those `packages` entries large. They are shared by every dependency that uses them, and the format is the one `devEngines.runtime` already writes for a project's own runtime.

## Prior Art

- **Corepack / the `packageManager` field** pins a project's *own* package manager, with a hash. It established that "which package manager, at which version" is lockable metadata — but it only protects projects that opt in, says nothing about a consumer's transitive git dependencies, and Corepack's hash covers its own download path rather than flowing through the consumer's lockfile.
- **pnpm's own `devEngines.runtime`** already records a project's runtime in the lockfile as a `node@runtime:<version>` package with per-platform integrity. This RFC extends the same record to the runtime a git dependency builds with.
- **npm** side-steps the problem by always preparing git dependencies with npm itself, regardless of what the dependency uses — deterministic, but simply broken for dependencies whose builds require Yarn or Bun (their lockfiles are ignored, their `prepare` tooling may not resolve). pnpm's provisioning is the more correct behavior; this RFC removes the nondeterminism it introduced.
- **Volta** pins per-project toolchains (`node`, `yarn`) in `package.json` and provisions them transparently — the same "the declared toolchain is what runs" philosophy, applied at project scope rather than per git dependency.
- **Nix / Bazel–style hermetic builds** pin the entire build environment by content hash. This RFC is a narrow, pnpm-native slice of that idea: pin exactly the environment inputs pnpm itself downloads and executes.

## Unresolved Questions and Bikeshedding

- **Key name.** `buildSnapshots` vs `prepareSnapshots`. "Snapshots" signals the shape and the pairing with `packages`; "build" matches `allowBuilds` and the store's built/not-built split, while "prepare" names the lifecycle step exactly.
- **Yarn 6 key format.** Yarn 6 ships as platform archives rather than an npm package. It fits the `variations` resolution, but needs a `packages` key, presumably mirroring `runtime:` (e.g. `yarn@pm:6.0.0`), that the lockfile doesn't define yet.
- **`engines.node` fallback provisioning.** Excluded from this iteration because it makes the record host-dependent. A later iteration could provision a deterministic choice (e.g. the newest LTS satisfying the range, resolved once and recorded on every host) if failing on incompatible hosts proves to be a real pain point.
- **An escape hatch.** Whether a setting is needed to disable provisioning and recording entirely (air-gapped registries that don't mirror the package managers, e.g.), and its name if so.
- **Bun/Deno runtimes.** `devEngines.runtime` can name `bun` or `deno`; the same mechanism applies through their resolvers, but whether the first iteration supports them or errors is open.
