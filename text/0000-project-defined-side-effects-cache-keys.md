# Project-defined side-effects cache keys

## Summary

A project can tell pnpm what its dependency builds are built for:

```yaml title="pnpm-workspace.yaml"
sideEffectsCache:
  engine: darwin;arm64;electron43
  key: <digest of the build inputs>
```

`engine` replaces the host part of the side-effects cache key, `<platform>;<arch>;node<major>`. pnpm hashes `key` and appends it to the key. pnpm doesn't interpret either value. Builds made for different targets then get their own cache entries and global virtual store slots in one store. A project whose builds target something other than the Node.js that runs pnpm can then keep the cache on. With neither field set, no key changes. Both fields are pnpm v12 only.

The configuration shape and the requirements follow the maintainer's reply in [discussion 16039](https://github.com/orgs/pnpm/discussions/16039#discussioncomment-19592074).

## Motivation

### The key names the host

pnpm stores a dependency's build under this key:

```text
<platform>;<arch>;node<major>;format=2[;deps=<hash>][;patch=<hash>]
```

The host part describes the machine and the Node.js that run the install. A build script can target something else, and then two different builds get the same key. [RFC 0007](./0007-shared-side-effects-cache.md#the-build-input-problem) describes the general case: the key isn't a function of everything a build depends on. For a project that knows its targets, the gap has a concrete cost.

VS Code has three install roots whose native modules are built for different runtimes. The root workspace targets Electron, `remote` targets the server's Node.js, and `build` targets the Node.js that runs the install. The `updateConfig` hook of each root passes the target to install scripts through `extraEnv` (`npm_config_runtime`, `npm_config_target`, `npm_config_disturl`, `npm_config_arch`), which is how node-gyp and prebuild-install learn what to build for. All three roots compute the same key. With pnpm 12.16.0, one store and the cache on, `remote` ran no install scripts and got the root's Electron build of `node-pty` byte for byte, under `darwin;arm64;node24;format=2;deps=<hash>`. Its `config.gypi` says `runtime: electron`.

So all three roots set `sideEffectsCache: false`, and every fresh checkout compiles the same native modules again. Any project that sets `npm_config_runtime=electron`, as [Electron's documentation](https://www.electronjs.org/docs/latest/tutorial/using-native-node-modules#using-npm) suggests for npm-style installs, has the same problem.

### What the cache would save

As a workaround, each root's `updateConfig` hook returned a store directory per target, for example `<store dir>/electron-43.8.6-arm64`, with the cache on. On macOS arm64, the median install in a fresh worktree went from about 46 seconds to about 9 seconds once that target's store had been warmed by one install.

The workaround doesn't hold up:

- The hook has to seed each new store (link `files`, copy `index.db` without the side-effects rows), or pnpm downloads every package again.
- `pnpm store prune` doesn't run the hook and only prunes the default store.
- A new Electron or Node.js version leaves the old store behind.
- `--store-dir` on the command line wins over the hook's `storeDir` but not over its `sideEffectsCache`, so the hook has to detect that case and turn itself off.

### Only the project knows its build inputs

The runtime is not the only input. VS Code's Linux CI sets `CC`, `CXX`, `CFLAGS`, `CXXFLAGS` and `LDFLAGS` per target: sysroot compilers, libc++ and LTO flags for the Electron root, and other sysroot compilers for the server root. Other projects use toolchains pnpm can't anticipate. The values hold absolute paths of the workspace and the sysroot, so one toolchain gives different values on two machines. A new compiler or sysroot at the same path changes the build without changing any variable.

pnpm can't pick these inputs or normalize them. The project can, at install time, from whatever its builds depend on.

## Detailed Explanation

### The settings

```yaml title="pnpm-workspace.yaml"
sideEffectsCache:
  engine: darwin;arm64;electron43   # replaces <platform>;<arch>;node<major>
  key: <digest of the build inputs> # appended as ;key=<hash>
```

| Field | Effect on the key | Cost of a wrong value |
|---|---|---|
| `engine` | Replaces the host part, used as written. | Can merge entries that must stay apart. |
| `key` | Appends `;key=<hash>`. pnpm hashes the value, so paths in it never reach the store index or a server. | Can only split entries. Costs an extra build. |

- pnpm doesn't interpret either value.
- A project that sets `engine` owns the platform and the architecture in it. A value that leaves them out shares builds between machines with different platforms. The docs must say so.
- Both fields come from the same sources as `sideEffectsCache.exclude`: `pnpm-workspace.yaml`, the environment, and the result of an `updateConfig` hook.
- A value is a string or a map of package patterns, the patterns `exclude` uses. The first implementation accepts the string form only. A map fails with an error that says the per-package form isn't supported yet. See [What this RFC does not do](#what-this-rfc-does-not-do).

### Setting the values from `updateConfig`

The hook of each install root computes the values at install time. pnpm v12 already applies a `sideEffectsCache` object returned by `updateConfig` (tested with `exclude`), so no new hook is needed.

```js title=".pnpmfile.mjs"
import { createHash } from 'node:crypto'

const buildInputs = ['npm_config_runtime', 'npm_config_target', 'npm_config_arch', 'CC', 'CXX', 'CFLAGS', 'CXXFLAGS', 'LDFLAGS']

export const hooks = {
  updateConfig: config => {
    const env = { ...process.env, ...config.extraEnv }
    const key = createHash('sha256')
      .update(JSON.stringify(buildInputs.map(name => [name, env[name] ?? null])))
      .digest('hex')
    return { ...config, sideEffectsCache: { ...config.sideEffectsCache, key } }
  },
}
```

A real hook would remove machine-specific paths from the values first, and could add a compiler's `--version` output or a checksum of the sysroot. Unresolved question 1 covers what this example does when the cache is off.

### The local cache key

```text
<engine>;format=2[;deps=<hash>][;patch=<hash>][;key=<hash>]
```

- `<engine>` is `sideEffectsCache.engine` when it is set. Otherwise pnpm resolves it as it does today: the root project's `node@runtime:` pin, then `nodeVersion`, then `node --version`.
- `;key=<hash>` is present only when `key` is set.
- pnpm keeps owning `format`, `deps` and `patch`.
- With `engine` set, pnpm can key a build without detecting Node.js.
- Every path that computes an engine uses this one result, including the frozen-lockfile install, the deferred host probe and the remote restore. [Implementation](#implementation) lists them.

### The global virtual store

- A package in the build-required set hashes the configured `engine` and a `key` field into its slot. The set holds every package that builds and every package that depends on one, which is the rule the engine follows today. Dependents are in it because their slot links to the built package's slot.
- Every other slot hashes as today. Packages that need no build keep sharing slots across targets.
- The layout cache that saves computed slot paths between installs includes `key` in its fingerprint, as it includes the engine. Otherwise an install with a new `key` reuses the slot paths computed for the old one.

### The remote tier

- The remote input key, `dependency-side-effects:v1:deps=<hash>[;patch=<hash>]`, leaves the engine out on purpose. [RFC 0007](./0007-shared-side-effects-cache.md#compatibility-tags-are-not-the-lookup-key) matches the platform through compatibility tags instead. When a field is set, pnpm appends `;engine=<hash>`, `;key=<hash>`, or both. An artifact built for one target is then never restored for another.
- The platform check at restore time still runs against the host.
- A restored artifact goes into the local cache under the key from [The local cache key](#the-local-cache-key), configured values included. Under the host key, a root that sets neither field would find it.

### Commands

Every command that computes a cache key or a slot path sees the same `updateConfig` result as `pnpm install`.

- `pnpm rebuild` already runs the hooks.
- `pnpm store status` doesn't, so it would look for the wrong slots. A hook-set `sideEffectsCache.exclude` has the same gap today, so the fix ships on its own, as a bug fix to a released setting.
- An optimistic repeat install records both fields, as it records `exclude`. A changed value forces a full install.
- pnpm's download of itself keeps the host engine. Unresolved question 6 covers `pnpm dlx`.

### Unset fields

With neither field set, every cache key, slot path and remote input key is byte-identical to today. Existing stores keep working.

### Older pnpm versions

pnpm v11 gets bug fixes only and doesn't read either field. A v11 install of a project that sets them behaves as if they were unset, with today's host keys and today's mix-ups between targets. A project that relies on the fields should require pnpm v12, for example through `packageManager`.

### Trust

A cloned repository controls `pnpm-workspace.yaml` and its pnpmfile, so it controls both values. The values only select cache entries and grant no trust. The signing material for the remote tier stays in global config, and the signature and platform checks still run on restore. Any project on the machine can already write builds under the host key, and `engine` doesn't change that.

### What this RFC does not do

- The map form. It needs each package's `engine` and `key` folded into the `deps=` graph hash, which covers a dependency's id but not its values.
- A package-declared field. It can later provide a per-package default for projects that configure nothing.
- Hashing node-gyp's `npm_config_*` variables by default. See [Rationale and Alternatives](#hashing-node-gyps-variables-by-default).
- The other ideas in the discussion's opening post: a message when pnpm restores a build from the cache, and a flag that forces rebuilds.

## Rationale and Alternatives

### `key` alone

The first proposal in the discussion had only `key`. The host values aren't only incomplete, they can be wrong. In the VS Code setup, `node24` names the Node.js that runs the install, but the build targets Electron or the server's runtime. With `key` alone, one Electron build made under two host Node.js versions lands under two keys. `engine` lets the project replace the host values. The cost is that a wrong `engine` can merge entries that must stay apart, while a wrong `key` only splits them.

### A list of environment variable names

An earlier proposal listed variables whose names and values pnpm would hash, as an install script would see them:

```yaml
sideEffectsCache:
  env:
    - npm_config_runtime
    - npm_config_target
    - npm_config_arch
    - npm_config_disturl
```

Values with absolute paths hash differently on two machines for one toolchain, and a toolchain change at the same path changes no variable. Normalizing them takes knowledge only the project has. An opaque string lets the hook do it.

### Hashing node-gyp's variables by default

pnpm could hash the well-known `npm_config_*` variables without any configuration. That would make Electron projects correct out of the box, but pnpm core would then name variables. The compiler variables matter as much and vary between projects, so a list in core is never complete. Adding a variable to it later would change the keys of every project that sets that variable.

### One store per target

The workaround from [What the cache would save](#what-the-cache-would-save) works with no change to pnpm. It needs every store seeded by hand, `pnpm store prune` doesn't reach it, old stores stay behind, and it has to detect `--store-dir`.

### A field declared by the package

The opening post proposed a `package.json` field in which a package names the settings that change its build. The project sets the variables, so it knows they change the build, whatever the packages declare. A package-declared field can follow as a default for projects that configure nothing, and can feed the per-package form.

### Compatibility tags instead of the input key

[RFC 0007](./0007-shared-side-effects-cache.md#bounding-the-problem-instead-of-closing-it) keeps the build environment out of the remote input key. It records the builder profile as provenance and matches platforms through compatibility tags. A producer's claim about where an artifact was built would fragment the cache without making it correct. `engine` and `key` are different. They describe what the project builds for, and the producer and the consumer read them from the same project configuration. An artifact built for another target is wrong, not just built elsewhere, so a mismatch has to be a miss. Compatibility tags can't express that: pnpm interprets them as a closed vocabulary of host properties, and these values are opaque to it.

## Implementation

All changes are in the pnpm v12 CLI, under `pnpm/` in pnpm/pnpm. pnpm v11 doesn't change.

**Hashing** (`pnpm/crates/graph-hasher`). `calc_dep_state` takes the engine string as today and an optional `key`. `calc_dep_state_input_key` takes an optional `engine` and `key`. `calc_graph_node_hash` takes an optional `key` for build-required packages.

**Engine resolution.** One function returns the effective engine, and a configured `engine` short-circuits the runtime pin, the host detection and the deferred probe. Several paths compute the engine today, and each one has to use that result:

- `resolve_engine_name` and `DeferredEngineName::resolve_for_build` in `pnpm/crates/deps-restorer/src/materialization_plan.rs`.
- `plan_engine_name` and `settle_engine_name` in `pnpm/crates/deps-restorer/src/install_frozen_lockfile/planning.rs`. When host detection is still pending, the frozen-lockfile install fills the engine from the host without calling `resolve_engine_name`.
- `settle_engine_name` in `pnpm/crates/package-manager/src/install_with_fresh_lockfile/on_disk.rs`.
- `virtual_store_layout_for_lockfile` in `pnpm/crates/deps-restorer/src/virtual_store_layout.rs`.
- `package_map_engine_name` in `pnpm/crates/package-manager/src/install/apply_materialization/selection.rs`.
- `DepStateHasher` in `pnpm/crates/deps-restorer/src/shared_side_effects/planning.rs`, which keys a restored remote artifact in the local cache. Its platform detection for the compatibility tags stays host-based.

**Global virtual store layout.** `GvsHasher::fingerprint` in `pnpm/crates/deps-restorer/src/virtual_store_layout/graph_hash.rs` adds `key`. The slot hash there prefers a snapshot's own `node@runtime:` pin over the install's engine, which unresolved question 2 covers.

**Configuration** (`pnpm/crates/config`). `SideEffectsCacheSettings` gains `engine` and `key`, typed so that a map can be added later. Both flow through applying, resetting, the resolved form and the environment overlay, as `exclude` does. `apply_side_effects_cache` decides what an omitted `read` or `write` means (unresolved question 1).

**Commands.** The `store` dispatcher in `pnpm/crates/cli/src/cli_args/dispatch_query/maintenance.rs` applies `updateConfig`. The install state in `pnpm/crates/package-manager/src/install_state/settings.rs` records both fields.

**Tests.**

- `pnpm-graph-hasher`: unset fields give today's strings, `engine` replaces the prefix, and `key` appends one segment. The input key and a slot hash change only when a field is set, and a slot hash only for build-required packages.
- `pnpm-config`: parsing, reset, the resolved form, the environment, and the map-form error.
- An `updateConfig` hook that returns `key` reaches the config and leaves `read` and `write` as they were.
- The layout cache fingerprint changes when `key` changes. A reinstall with a new `key` after the layout cache is warm gets new slots.
- CLI tests with two projects sharing one store:
  - Different values make the second install run the build script. Equal values restore the build.
  - Under the global virtual store, only built packages and their dependents get new slots.
  - `pnpm rebuild` and `pnpm store status` agree with `pnpm install` on a hook-set value.
  - A restored remote artifact lands in the local cache under the configured key.
- Repeat-install drift for both fields.

**Docs and release.** Sections for both fields in `pnpm/docs/settings/build.md`, with an `updateConfig` example and the warning about `engine`. A `pacquet` changeset with a minor bump, and a separate patch changeset for the `pnpm store status` fix.

The implementation is done when:

1. Two install roots that share a store and set different `key` values each build a native package once, then restore their own build on later installs.
2. With the same `engine`, builds made under two host Node.js versions share one cache entry.
3. With neither field set, no cache key, slot path or remote input key changes.
4. `pnpm install`, `pnpm rebuild` and `pnpm store status` compute the same keys and slots for a value set by `updateConfig`.

## Prior Art

- [prebuild-install](https://github.com/prebuild/prebuild-install) picks a prebuilt binary by target: `--runtime` (`node`, `napi` or `electron`), `--target`, `--arch`, `--platform` and `--libc`, or the matching `npm_config_*` variables. The target is an input separate from the Node.js that runs the installer, which is what `engine` gives the cache.
- [ccache](https://ccache.dev/manual/latest.html)'s `compiler_check` setting accepts `string:value`, for "another string that the build system generates to identify the compiler", or a command whose output ccache hashes. That is the role of `key`: the build system says what identifies its toolchain, and the cache hashes it without interpreting it.
- [`actions/cache`](https://github.com/actions/cache#creating-a-cache-key) takes its `key` from the workflow, commonly built from the runner's OS and a hash of the lockfiles.
- [RFC 0007's prior art](./0007-shared-side-effects-cache.md#prior-art) notes that Turborepo, Nx and moon take their keys from user configuration, so a wrong key is the user's misconfiguration. This RFC makes that trade for two values only, and `key` can only split entries.

## Unresolved Questions and Bikeshedding

1. `read` and `write` in hook output. When the cache is off, a hook sees `sideEffectsCache: false`, and the pattern `{ ...config.sideEffectsCache, key }` then returns `{ key }`. `apply_side_effects_cache` defaults a missing `read` and `write` to `true`, so the hook turns the cache back on and overrides `--no-side-effects-cache`. Proposal: in hook output, an omitted `read` or `write` keeps its current value.
2. `engine` and a package's own `node@runtime:` pin. The slot hash prefers a package's own pin over the install's engine. Proposal: a configured `engine` wins, since the project owns the value. The runtime dependency still changes `deps=`.
3. Which slots get `key`. The maintainer's reply says "only of a built package". This RFC reads that as the build-required set, dependents included. Gating on the built package alone would let two targets share a dependent's slot whose links point at different builds.
4. Empty strings. Proposal: reject them, so that a hook bug can't silently fall back to shared keys.
5. Project-local `node_modules`. Without the global virtual store, should changing a value re-import built packages that are already linked, or is `pnpm rebuild` the answer?
6. `pnpm dlx`. It loads the project's `pnpm-workspace.yaml` without the hooks, so a value set there can reach the tool's install. Proposal: `dlx` ignores both fields, since it installs a tool for the host and not for the project's target.
