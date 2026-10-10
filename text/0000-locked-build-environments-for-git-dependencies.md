# Locked build environments for git-hosted dependencies

## Summary

When pnpm installs a git-hosted dependency that has to be built, it recreates the dependency's own development workflow: it installs the dependency's `devDependencies` with the package manager the dependency asks for (npm, Yarn, Bun, or pnpm itself) and runs its `prepare` scripts. Which version of that package manager runs is decided per machine — often from a floating spec sniffed from the dependency's lockfile — and the build always runs on whatever Node.js the host has. This RFC proposes to (1) resolve the build tools of a git-hosted dependency at install time like any other dependency and record them in the consuming project's lockfile as ordinary `packages`/`snapshots` entries, reached from the git-hosted package's snapshot through a new `buildDependencies` group that no install ever materializes — the way `pnpm install --prod` leaves `devDependencies` in the lockfile but out of `node_modules` — and make that record authoritative, provisioning exactly the recorded, integrity-verified tools on subsequent installs; and (2) provision the Node.js runtime a dependency pins via `devEngines.runtime`, the same way the package manager is already provisioned from `devEngines.packageManager` / `packageManager`.

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

### 1. A `buildDependencies` group on git-hosted snapshots

pnpm already has a way to keep packages in the lockfile without installing them: dependency groups. `pnpm install --prod` leaves every `devDependencies` entry, and everything only it reaches, in `packages` and `snapshots`, and materializes none of it. The walks that decide what gets installed take the selected groups and do not follow edges from the others; the pruning of the wanted lockfile follows every group, so nothing is lost.

Build tools are a dependency group of the same kind, with one difference: no install ever selects it. The snapshot of a git-hosted package that requires a build gains a `buildDependencies` field, shaped like `dependencies`, and the tools are ordinary `packages` and `snapshots` entries:

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

  '@yarnpkg/cli-dist@4.9.2': {}

  example@git+https://github.com/org/example.git#5c8f9a…:
    dependencies: …
    buildDependencies:
      node: runtime:24.6.0
      yarn: '@yarnpkg/cli-dist@4.9.2'

  node@runtime:24.6.0: {}
```

- **The group is never materialized.** Every walk that decides what goes into `node_modules` — materialization, hoisting, bin linking, the current lockfile, the install-time prefetch — skips `buildDependencies` edges the way `--prod` skips `devDependencies`. A build tool never gets a virtual-store slot and never has bins linked into the project. A package reached both through `buildDependencies` and through an installed group (the project's own `node@runtime:24.6.0`, say) is installed through the installed group, and the two share one entry.
- **The group is always retained.** Pruning the wanted lockfile follows every group, so the tools stay as long as a git-hosted package references them, and go when nothing does.
- **The group is fetched only when a build runs.** The tools are fetched into the content-addressable store when the git dependency actually has to be prepared — its built artifact is missing from the store and from the side-effects cache — and run from there, the way `pnpm dlx` runs a package manager today. A warm install downloads nothing for them. `pnpm fetch`, which fills the store with every group, fetches them too, so a later offline build has them.
- **Tools are ordinary entries.** Most have no edges — npm bundles its dependencies, `@yarnpkg/cli-dist` is a single bundle, and runtimes and archive-shipped package managers are single archives — so their snapshot is `{}`, as `node@runtime:<version>: {}` already is. A tool that does have dependencies uses ordinary edges: a downloaded pnpm is `@pnpm/exe`, whose binary comes through platform-specific `optionalDependencies`. Build tools take no part in peer resolution.

Reusing ordinary entries is what makes the record cheap:

- **Integrity comes for free.** A package manager published to npm is locked by its tarball integrity, exactly as a regular dependency is. The record doesn't trust the registry to serve the same bytes for the same version any more than pnpm does for a leaf dependency.
- **Platform binaries are covered.** A runtime, and a package manager that ships as a platform archive (Bun, Yarn 6), is locked by the existing `variations` resolution, which pins every platform's archive by integrity. A single integrity on a wrapper package would pin only the wrapper and leave the binary that actually runs unpinned. Because every platform is recorded at resolution time, the record is the same no matter which machine wrote it.
- **One entry per artifact.** Ten git dependencies prepared with Yarn 4.9.2 share one entry and one store copy.
- **The alias form is the existing one.** Yarn Berry is published as `@yarnpkg/cli-dist`, so it is recorded as `yarn: '@yarnpkg/cli-dist@4.9.2'`, the way any npm alias is.

**What is new** is a dependency group on a package snapshot. Groups have so far existed only on importers, so group selection is decided where the walk starts. With `buildDependencies`, the walks that install also have to skip a group on every snapshot edge they follow. That rule lives in one place — the reachability walk that already takes the group selection — rather than in each consumer.

The group is on the snapshot because that is where edges live. It depends only on the package, so a git-hosted package resolved in several peer contexts carries the same `buildDependencies` on each of its snapshots.

`buildDependencies` names only what pnpm itself downloads and executes to prepare the dependency. The dependency's own `devDependencies`, installed by that package manager inside the checkout, are governed by the lockfile the dependency ships at that commit and are out of scope.

### 2. Resolution and provisioning rules

**First resolution** (the git dependency enters the lockfile, or is re-resolved by `pnpm update`): pnpm reads the wanted spec from the dependency's manifest or lockfile, exactly as today, and resolves it with pnpm's normal rule: a version already locked that satisfies the spec, else the `latest` dist-tag if it satisfies the spec, otherwise the highest satisfying version. Preferring a locked version is the rule regular dependencies already follow, and keeps git dependencies converging on few tools. The result is recorded in `buildDependencies` even when the host has a copy that would satisfy the spec, so the record never depends on which machine wrote it. When the dependency's `packageManager` field carries a Corepack hash (`yarn@1.22.22+sha512.…`), the hash is checked against the artifact being recorded where it covers the same bytes, and a mismatch fails resolution.

**Subsequent installs: pnpm provisions exactly what is recorded.** The build runs on the recorded package manager and runtime, fetched from the store or downloaded and verified against the recorded integrity. A copy on the host's `PATH` is not used by default, even at the recorded version: it is an executable pnpm never verified, and since a verified copy is downloaded once and reused from the store afterwards, using the host's copy saves little. Today's rule — the host's copy prepares an unpinned dependency whenever it satisfies the spec — is dropped.

**A frozen install** never resolves live. A git-hosted package that requires a build but has no `buildDependencies` is treated like any other lockfile staleness under `--frozen-lockfile`: the install fails with an error telling the user to run a regular install to update the lockfile.

**Stickiness** follows the git resolution itself: the record refreshes only when the git dependency re-resolves (`pnpm update`), exactly like the commit SHA. Ordinary installs cause no lockfile churn.

**A release committed to the repository.** A Yarn Berry repository may commit its release under `.yarn/releases/` and point `yarnPath` at it in `.yarnrc.yml`; the release that runs is then pinned by the commit, and the recorded Yarn only launches it. The record is still written, because the launcher is still code pnpm downloads and executes, but whether the build is reproducible no longer depends on it.

**Choosing a different tradeoff.** Some environments cannot provision: an air-gapped registry that doesn't mirror the package managers, or a sandbox that forbids downloading executables. A setting (drafted as `gitBuildTools`) makes the tradeoff explicit:

| Value | Package manager and runtime used for the build | Verified | Deterministic |
|---|---|---|---|
| `locked` (default) | the recorded version, provisioned | yes | yes |
| `host-exact` | the host's copy when it reports exactly the recorded version, otherwise the recorded version, provisioned | host copy: no | yes |
| `host` | the host's copy when it satisfies the dependency's spec, otherwise the recorded version, provisioned | host copy: no | no |

The setting changes only which executable runs, never what is recorded: the lockfile is the same whichever value wrote it, so a team can mix values across machines without lockfile churn.

### 3. Runtime selection

The dependency's manifest gets a say in which runtime runs its build:

1. **`devEngines.runtime` names a runtime pnpm can provision** → pnpm resolves it, records it in `buildDependencies` as a `<name>@runtime:<version>` package, and runs the inner install and the `prepare` scripts on it. This is symmetric with `devEngines.packageManager`: the field describes the dependency's development environment, which the prepare recreates.
2. **Otherwise** → the host runtime is used, and nothing is recorded.

Only an explicit dev pin selects the runtime, so the record depends on the dependency's manifest at the locked commit and never on the host. A dependency whose `engines.node` the host fails is not provisioned for: provisioning only on hosts that fail the range would record a runtime on some machines and not others, and two contributors would produce different lockfiles for the same dependency. Such a build fails as it does today, with a hint naming the range. `engines.node` is a consumer-facing compatibility range (typically `>=18`); treating it as a build pin on every host would mean "always latest Node," which nobody intends.

**Scope boundary — ABI.** The provisioned runtime applies only to the *inner* workflow inside the git checkout: the dependency's own dependency install and its `prepare`/`prepack` scripts. The dependency's `install`/`postinstall` scripts, which run later in the consuming project, keep the project's runtime, because their output (typically compiled native addons) must match the ABI of the Node that will load it. A git dependency that compiles native code *during prepare* has an ABI mismatch problem with or without this RFC (the built result is cached and reused regardless of the consumer's Node); this RFC deliberately does not attempt to solve it, and the boundary above at least never makes it worse.

### 4. Store and cache keying

The store-index key of a built git-hosted dependency, today derived from the package ID and whether it was built, also covers its `buildDependencies`: the recorded tools and their integrity. `pnpm update` can keep the commit and move the recorded package manager; without the record in the key, the artifact built by the old environment would be reused and the lockfile would describe a build that never happened. The same applies to the input key of the shared side-effects cache ([RFC 0007](./0007-shared-side-effects-cache.md)): an artifact built with Yarn 4.9.2 is not the artifact built with Yarn 4.10.0.

### 5. Failure behavior

Everything in `buildDependencies` was demanded either by the dependency or by the lockfile, so failing to provide it fails the install with an actionable error, as a pinned package manager does today. Nothing falls back to the host silently; the error names `gitBuildTools` for environments that have to use the host's tools. Offline installs succeed whenever the recorded tools are already in the store — after `pnpm fetch`, for instance.

### 6. Compatibility and migration

The lockfile change is additive: a new field on snapshots, plus `packages` and `snapshots` entries of shapes the lockfile already has. It ships with a minor `lockfileVersion` bump, not a major one. The breaking part is behavioral, and provisioning git-dependency build tools is not marked experimental, so the change ships in a pnpm major:

- **Frozen installs of an upgraded project fail until the lockfile is rewritten.** Every existing lockfile lacks the record, so a frozen install of a project with a build-requiring git dependency fails with the staleness error from section 2. Run `pnpm install` once and commit the result. Lockfiles without build-requiring git dependencies are unaffected.
- **An older pnpm drops the record loudly, not silently.** An older pnpm ignores `buildDependencies` when reading and drops it when it rewrites the lockfile, then prunes the tool entries nothing else reaches. The deletion shows up in the lockfile diff, and the next frozen install with a current pnpm fails with the staleness error. Nothing is lost that a regular install doesn't restore.
- **Hosts that used to prepare with their own package manager now download one.** This is the point of the RFC, and it goes in the release notes, together with `gitBuildTools` for environments that cannot download.

### 7. Consolidation point

Dependency groups on snapshot edges, one of which is never materialized, are a general way to express "pinned by the lockfile, run by pnpm, not installed into `node_modules`". Two existing records fit that description and are kept separate by this RFC:

- the per-package-manager environment lockfiles under the pnpm home, which pin the package managers `pnpm dlx` and global shims run — this RFC already replaces them for git-dependency builds;
- `packageManagerDependencies` in the env lockfile document, which pins pnpm itself.

Neither is moved now: the env document is read before the project's lockfile to select the pnpm version, and the home lockfiles have no project to belong to. If a later change unifies them, this group is the intended shared abstraction.

## Rationale and Alternatives

1. **Do nothing; rely on `packageManager` exact pins (the Corepack model).** Works only for dependencies that pin, which most don't; the sniffed and range cases keep floating. It also leaves the frozen-install hole open, which is the most serious of the problems.

2. **Keep the status quo: pin per machine in the environment lockfile under the pnpm home.** This is what pnpm v12 does today. It gives per-machine stability, but it doesn't travel with the project: CI and a new laptop resolve fresh, and there is no reviewable diff when the version changes. The project lockfile is the only artifact with the right ownership and review workflow.

3. **Write the build tools into the git package's existing snapshot `dependencies`.** This needs no new field: `engines.runtime` entries are already turned into `node: runtime:<version>` dependencies there. But a dependency in that field means something different — it is linked into the package's `node_modules`, and a runtime there is the runtime the package runs on, including its `postinstall`. Recording a dev-only runtime there would compile native addons for the wrong ABI, and a frozen install, which links from the lockfile without reading manifests, has no way to tell a build tool from a real dependency.

4. **A separate build graph (`buildSnapshots`) over the shared `packages`.** Tool resolutions in `packages`, but the edges from each git package to its tools in a new top-level map beside `snapshots`. The install walks could not reach the tools at all. But it is a second graph with its own entry type, validation, and pruning, built to provide what dependency groups already provide: "in the lockfile, not installed" is what `--prod` does to `devDependencies`. Extending groups to snapshot edges reuses the owning capability instead.

5. **A self-contained map for the tools, outside `packages`.** A top-level map holding both the tools' resolutions and their edges. It is a second catalog of artifacts beside `packages`, with its own entry shape and integrity checks, and an artifact used both for building and at runtime would be recorded twice.

6. **The env lockfile document, or a separate file.** The env document holds pnpm's own tooling, read early to select the pnpm version; a transitive dependency's build tools don't belong there, and its keys would have to point into the other document. A separate lockfile file would avoid touching `pnpm-lock.yaml`, but every tool that handles the lockfile — Renovate and Dependabot, merge drivers, git-branch lockfiles, `pnpm deploy` — would have to learn a second file, and the two could drift.

7. **Lock the version but not the integrity.** Cheaper to implement and closes the reproducibility gap. But it still trusts the registry to serve the same bytes for the same version forever — a guarantee pnpm refuses to extend to any ordinary dependency. The executable that runs build scripts deserves at least the scrutiny of a leaf dependency.

8. **Lock the integrity of the built output instead of the environment.** This would detect divergence however it arose, without caring what ran the build. It fails in practice because builds are rarely byte-reproducible: timestamps, absolute paths, and nondeterministic bundler output would make the check fail on machines that built correctly. Locking the inputs is the part pnpm controls.

9. **Use the host's copy by default when it reports exactly the recorded version.** Saves a download on hosts that have the tool, and is as deterministic as provisioning. But it runs an executable pnpm never verified, which gives up the security guarantee by default to save a download that happens once per tool and machine. It remains available as `gitBuildTools: host-exact`.

The proposed design is the only one of these that makes `--frozen-lockfile` mean what it says, keeps builds convergent across machines, and stays within pnpm's existing trust model: everything downloaded and executed at install time is named and integrity-pinned by the lockfile.

## Implementation

pnpm v12 only, per the version policy: the TypeScript v11 CLI receives no changes.

- `crates/git-fetcher` — `preferred_pm` keeps detecting the wanted spec; `prepare_package` accepts the resolved tools from the caller instead of resolving them itself, uses the host's copy only as `gitBuildTools` allows, and runs the inner workflow on the recorded runtime; `pm_shims` forwards to the recorded exact version and integrity rather than to a range.
- `crates/resolving-git-resolver` / `crates/resolving-deps-resolver` — the first-resolution path resolves the package manager and the `devEngines.runtime` pin through the existing package-manager and runtime resolvers (`engine-pm-yarn-resolver`, `engine-runtime-node-resolver`, `engine-runtime-bun-resolver`, `engine-runtime-deno-resolver`), so the record exists before the build runs.
- `crates/lockfile` and `crates/lockfile-verification` — the `buildDependencies` snapshot field, the minor `lockfileVersion` bump, and frozen-install validation that every build-requiring git snapshot carries the field and that every edge resolves.
- `crates/deps-restorer` (`current_lockfile/reachability.rs`: `GroupSelection`, `collect_reachable`) and `crates/lockfile` (`filter_by_importers`) — group selection applies on snapshot edges, not only at importers: the install walks never follow `buildDependencies`, and wanted-lockfile pruning always does. Consumers that already receive a group-filtered lockfile — the install-time prefetch, the package map, the current lockfile — need no change of their own.
- `crates/store-dir` — `git_hosted_store_index_key` covers the recorded build dependencies.
- `crates/config` — the `gitBuildTools` setting.
- `crates/deps-inspection` — `pnpm list`/`pnpm why` show build dependencies only when asked for them.

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

- **Names.** `buildDependencies` vs `prepareDependencies` for the group; `gitBuildTools` and its values for the setting. "Build" matches `allowBuilds` and the store's built/not-built split, while "prepare" names the lifecycle step exactly.
- **Yarn 6 key format.** Yarn 6 ships as platform archives rather than an npm package. It fits the `variations` resolution, but needs a `packages` key, presumably mirroring `runtime:` (e.g. `yarn@pm:6.0.0`), that the lockfile doesn't define yet.
- **`engines.node` fallback provisioning.** Excluded from this iteration because it makes the record host-dependent. A later iteration could provision a deterministic choice (e.g. the newest LTS satisfying the range, resolved once and recorded on every host) if failing on incompatible hosts proves to be a real pain point.
- **Bun/Deno runtimes.** `devEngines.runtime` can name `bun` or `deno`; the same mechanism applies through their resolvers, but whether the first iteration supports them or errors is open.
