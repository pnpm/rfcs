# Locked build environments for git-hosted dependencies

## Summary

When pnpm installs a git-hosted dependency that has to be built, it recreates the dependency's own development workflow: it installs the dependency's `devDependencies` with the package manager the dependency asks for (Yarn, npm, Bun, or pnpm itself) and runs its `prepare` scripts. The package manager is resolved from the registry *at prepare time*, from a spec that is often a floating range, and the build always runs on whatever Node.js the host happens to have. This RFC proposes to (1) record the resolved build environment — package manager and runtime, with exact versions and integrity — in the consuming project's lockfile and make that record authoritative on subsequent installs, and (2) provision the Node.js runtime a dependency pins via `devEngines.runtime`, the same way the package manager is already provisioned from `devEngines.packageManager` / `packageManager`.

## Motivation

### The package manager used for the build is not locked anywhere

pnpm decides which package manager prepares a git-hosted dependency from three sources, in order:

1. An exact pin in the dependency's `packageManager` or `devEngines.packageManager` field.
2. A range in those same fields (e.g. `yarn@^4`).
3. A sniff of the lockfile the dependency ships: `yarn.lock` implies Yarn `1` or `>=2` (depending on whether the file carries Berry's `__metadata` block), `bun.lock` implies Bun, and so on — with no version constraint beyond the Yarn line.

Only case 1 is deterministic, and only transitively: the git commit SHA in the consumer's lockfile pins the manifest, and the manifest pins the version. Cases 2 and 3 — the overwhelming majority of real git-hosted dependencies, since most packages don't declare `packageManager` — resolve to *whatever is latest-matching at that moment*. Nothing records what was used.

The consequences:

- **Builds are not reproducible across machines or time.** Two machines preparing the same git dependency on different days can build with different Yarn versions, whose hoisting and dedupe differences produce different dependency trees for the build — and potentially different build output. Because git resolutions carry no integrity checksum, the divergence is undetectable. This is a classic source of "works locally, fails in CI."
- **`--frozen-lockfile` is not actually frozen.** A frozen install of a project with such a git dependency will download and *execute* an executable resolved live from the registry, with no lockfile diff for anyone to review. A compromised `yarn@latest` (or a malicious release matching `>=2`) reaches script execution on CI. This contradicts the promise a frozen install makes, and it is a weaker guarantee than pnpm gives every other package it downloads.
- **Even the exact-pin case trusts the registry on faith.** The commit SHA pins the version *string*, but nothing pins the *content* served for that version. Every regular dependency in a pnpm lockfile carries an integrity hash; the package manager that runs the build — with full script execution rights — carries none.

### The build runs on an arbitrary Node.js

The prepare currently runs with the host's Node, unconditionally. If the dependency's authors develop and test on Node 24 and the host runs Node 20, the build can fail outright (build tooling with `engines` floors is common) or quietly produce different output. pnpm already knows how to resolve, download, and verify any Node version (the `devEngines.runtime` / `pnpm env` machinery, verified against the official SHASUMS files) — it just doesn't apply that knowledge in the one place where pnpm runs *someone else's* development workflow.

The expected outcome of this RFC: preparing a git-hosted dependency uses the same package manager and runtime versions on every machine that shares the lockfile, those versions are integrity-verified downloads, a frozen install performs no live resolution of executables, and a git dependency that pins its dev runtime builds with that runtime instead of failing on hosts that don't have it.

## Detailed Explanation

### 1. A `prepared` record on git-dependency lockfile snapshots

The lockfile snapshot of a git-hosted dependency that requires a build gains an additive field describing the build environment pnpm chose:

```yaml
'example@git+https://github.com/org/example.git#5c8f9a…':
  resolution: { … }
  prepared:
    packageManager:
      name: yarn
      version: 4.9.2
      integrity: sha512-…
    runtime:
      name: node
      version: 24.6.0
```

- `packageManager.version` is the **exact** version the spec resolved to at first resolution, and `integrity` is the integrity of that package's tarball as served by the registry (the same value the registry's packument advertises, i.e. what a regular lockfile entry would carry).
- `runtime` is present only when pnpm provisioned the runtime (see section 3). Its integrity is implicit: Node downloads are already verified against the release line's SHASUMS file, so the version alone is sufficient to pin content.

The field is additive and optional; lockfile parsers that don't know it ignore it, so no lockfile-format major bump is required.

### 2. Resolution and reuse rules for the package manager

**First resolution** (the git dependency enters the lockfile, or is re-resolved by `pnpm update`): pnpm resolves the wanted spec (`yarn@^4`, `1`, `>=2`, or no constraint) against the registry to an exact version, fetches its integrity from the packument, and records both. The build uses that exact version.

**Subsequent installs**: the recorded version is authoritative.

- If the host has the package manager on `PATH` **at exactly the recorded version** (probed via `--version`, as today), the host copy is used.
- Otherwise pnpm provisions the recorded version itself and verifies the downloaded tarball against the recorded integrity. Today's behavior — accepting any host copy that merely satisfies the range — is dropped, because it silently reintroduces cross-machine divergence.
- A **frozen install** never resolves live: it either uses a matching host copy, uses a cached provisioned copy, or downloads exactly the recorded version and verifies it. A git-dependency snapshot that requires a build but lacks a `prepared` record is treated like any other lockfile staleness under `--frozen-lockfile`: the install fails with an error telling the user to run a regular install to update the lockfile.

**Stickiness** follows the git resolution itself: the `prepared` record refreshes only when the git dependency re-resolves (`pnpm update`), exactly like the commit SHA. This keeps lockfile churn at zero for ordinary installs.

### 3. Runtime selection

The dependency's manifest gets a say in which Node (or Bun/Deno, via the same mechanism) runs its build:

1. **`devEngines.runtime` pins a version pnpm can provision** → pnpm provisions it, records it in `prepared.runtime`, and the inner install + `prepare` run on it. This is symmetric with `devEngines.packageManager`: the field is a statement about the dependency's development environment, which the prepare recreates.
2. **No dev pin, but the host runtime fails the dependency's `engines.node` range** → pnpm provisions a satisfying version (see Bikeshedding for which one) rather than running a build that is declared to be unsupported. Recorded the same way.
3. **Otherwise** → the host runtime is used, and no `runtime` entry is recorded. `engines.node` is a consumer-facing compatibility range (typically `>=18`); treating it as a build pin would mean "always latest Node," which nobody intends.

**Scope boundary — ABI.** The provisioned runtime applies only to the *inner* workflow inside the git checkout: the dependency's own dependency install and its `prepare`/`prepack` scripts. The dependency's `install`/`postinstall` scripts, which run later in the consuming project, keep the project's runtime, because their output (typically compiled native addons) must match the ABI of the Node that will load it. A git dependency that compiles native code *during prepare* has an ABI mismatch problem with or without this RFC (the built result is cached and reused regardless of the consumer's Node); this RFC deliberately does not attempt to solve it, and the boundary above at least never makes it worse.

### 4. Failure behavior

Provisioning follows the existing pinned/unpinned split: if the environment was demanded by the dependency (a `devEngines`/`packageManager` pin) or by the lockfile record and cannot be provided, the install fails with an actionable error; a best-effort inference that cannot be provided falls back to the host with a warning. Offline installs succeed whenever the recorded versions are already in the store or on the host.

### 5. Parity

The change is a lockfile-format and behavior change, so it lands in both stacks — the TypeScript CLI and the Rust CLI — together, with matching tests.

## Rationale and Alternatives

1. **Do nothing; rely on `packageManager` exact pins (the Corepack model).** Works only for dependencies that pin, which most don't; the lockfile-sniff and range cases keep floating, and even a pinned version has no content integrity. It also leaves the frozen-install hole open, which is the most serious of the problems.

2. **Lock the version but not the integrity.** Cheaper to implement (no packument lookup at record time), and closes the reproducibility gap. But it still trusts the registry to serve the same bytes for the same version forever — a guarantee pnpm refuses to extend to any ordinary dependency. The executable that runs build scripts deserves at least the scrutiny of a leaf dependency.

3. **Pin in a machine-global cache instead of the lockfile** (e.g. make the dlx-style cache remember its first resolution per spec). This gives per-machine stability but doesn't travel with the project: CI and a new laptop still resolve fresh, and there is no reviewable diff when the version changes. The lockfile is the only artifact with the right ownership and review workflow.

4. **Always provision; never use a host copy.** Maximally hermetic and simpler to reason about, but it downloads package managers even on hosts that have the exact version installed, and it penalizes the common case for no determinism gain — an exact-version host probe is equally deterministic. The proposed rule (host allowed only at exactly the locked version) keeps the hermetic guarantee at lower cost. This remains a reasonable fallback position if exact-version probing proves unreliable in practice.

The proposed design is the only one of these that makes `--frozen-lockfile` mean what it says, keeps builds convergent across machines, and stays within pnpm's existing trust model (everything downloaded at install time is named and integrity-pinned by the lockfile).

## Implementation

**TypeScript CLI:**

- `exec/prepare-package` — currently selects the package manager via `preferred-pm` and always runs with the host toolchain; grows the spec-detection order (`devEngines` → `packageManager` → lockfile sniff), the exact-version host probe, provisioning of PM and runtime, and acceptance of a pre-recorded environment from the caller.
- `resolving/git-resolver` / the fetcher pipeline — the first-resolution path resolves the PM spec to an exact version + integrity so the record exists before the build runs, and threads it to prepare.
- `lockfile/*` (types, serialization, verification) — the additive `prepared` field on git snapshots; frozen-install validation that a build-requiring git snapshot carries the record.
- `env/*` — reuse of the Node download/verify machinery for provisioning the build runtime.

**Rust CLI:** the counterparts in `crates/git-fetcher` (`preferred_pm`, `pm_shims`, `prepare_package`), `crates/lockfile`, and `crates/engine-runtime-node-resolver`. The shim mechanism already forwards to a "resolve, verify, cache, run" path; it changes from carrying a range to carrying the locked exact version plus expected integrity.

**Effects and risks:**

- Lockfile change is additive; old readers ignore the key. Writing it does not require a lockfile major.
- The strictened host probe (exact match instead of range match) is the one behavior change existing users may notice: hosts whose Yarn satisfies the range but not the recorded exact version start getting a provisioned copy. This is the point of the RFC, but it deserves a release note.
- First resolution gains one packument fetch per distinct package-manager spec (cached; in practice one or two per project).
- The store key for built git dependencies may want to incorporate the recorded environment so that a changed record triggers a rebuild rather than reusing a stale artifact; see Unresolved Questions.

## Prior Art

- **Corepack / the `packageManager` field** pins a project's *own* package manager, with a hash. It established that "which package manager, at which version" is lockable metadata — but it only protects projects that opt in, says nothing about a consumer's transitive git dependencies, and Corepack's hash covers its own download path rather than flowing through the consumer's lockfile.
- **npm** side-steps the problem by always preparing git dependencies with npm itself, regardless of what the dependency uses — deterministic, but simply broken for dependencies whose builds require Yarn or Bun (their lockfiles are ignored, their `prepare` tooling may not resolve). pnpm's provisioning is the more correct behavior; this RFC removes the nondeterminism it introduced.
- **Volta** pins per-project toolchains (`node`, `yarn`) in `package.json` and provisions them transparently — the same "the declared toolchain is what runs" philosophy, applied at project scope rather than per git dependency.
- **Nix / Bazel–style hermetic builds** pin the entire build environment by content hash. This RFC is a narrow, pnpm-native slice of that idea: pin exactly the two environment inputs pnpm itself downloads and executes.

## Unresolved Questions and Bikeshedding

- **Field naming and shape.** `prepared` vs `buildEnv` vs folding into `resolution`. Whether `packageManager` should be a structured object (as drafted) or a `name@version` string plus a sibling integrity key, mirroring the manifest field.
- **Which version does an open-ended spec resolve to?** For `>=2`-style sniffed specs and absent specs: highest stable, or the default dist-tag (`latest`)? Dist-tags and semver-highest can disagree.
- **Recording host-prepared builds.** When the host's package manager at a matching version does the first build, should its version be recorded and enforced on other machines (drafted: yes — that is what makes builds converge), or should host-prepared builds stay unrecorded and unlocked?
- **`engines.node` fallback provisioning** (rule 2 of runtime selection): in scope, or should the first iteration ship `devEngines.runtime` only and keep failing on incompatible hosts? And when provisioning to satisfy a range: lowest satisfying, highest satisfying, or newest satisfying LTS?
- **Store/cache keying.** Should the store-index key for built git dependencies include the `prepared` environment, so a changed record rebuilds instead of reusing the artifact built by the old environment?
- **An escape hatch.** Whether a setting is needed to disable provisioning/locking entirely (air-gapped registries that don't mirror the package managers, e.g.), and its name if so.
- **Bun/Deno runtimes.** `devEngines.runtime` can name `bun` or `deno`; the same provisioning machinery applies, but whether the first iteration supports them or errors is open.
