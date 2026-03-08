# RFC: Config Dependencies Overhaul & `pnpm-config-lock.yaml`

## Background

Config dependencies (`configDependencies` in `pnpm-workspace.yaml`) let a workspace pull in
npm packages that provide pnpm configuration — hooks, shared patches, catalog presets, etc.
The mechanism works, but the current design has several limitations that become more apparent
as the scope grows:

1. **Integrity is inlined into the declaration.** The version specifier embeds a hash using
   a `version+sha512-…` convention (`"@my-org/cfg": "1.2.0+sha512-XYZ"`). This makes
   `pnpm-workspace.yaml` noisy and forces a non-standard value format that surprises users.

2. **No platform-specific variants.** Config dependencies are always pure-JS npm packages.
   There is no mechanism to express that a dependency has a platform-specific binary
   (like `@pnpm/exe`) with different tarballs and integrity hashes per OS/arch.

3. **No transitive dependencies.** Each config dep must be fully self-contained; it cannot
   depend on other packages.

4. **pnpm's own version is not a config dependency.** The `packageManager` field in
   `package.json` declares the required pnpm version, but the integrity of the pnpm binary
   is never recorded anywhere. There is no tamper-detection, no reproducibility guarantee
   for the toolchain itself.

5. **The version switching bootstrapping problem.** `switchCliVersion` runs at process
   startup, before the project's `pnpm-lock.yaml` is read. Even if we stored pnpm's
   integrity in the main lockfile, we could not use it to verify or locate the right
   pnpm binary before executing it.

---

## Proposed Design

### 1. Separate lockfile: `pnpm-config-lock.yaml`

A new, workspace-level file committed alongside `pnpm-workspace.yaml`. It records resolved
versions, tarballs, and integrity hashes for all config dependencies (including pnpm itself).

**This file is:**
- **Committed** to version control (provides reproducibility and auditability across the team).
- **Read early** at process startup, before `pnpm-lock.yaml`, so pnpm's own version and
  integrity can be verified before the correct binary is executed.
- **Uses the same format as `pnpm-lock.yaml`** — the same `packages` and `snapshots`
  structure, so existing parsing and tooling can be reused.
- **Updated** by `pnpm install` and `pnpm self-update`.

```yaml
# pnpm-config-lock.yaml
lockfileVersion: '9.0'

importers:
  .:
    configDependencies:
      pnpm:
        specifier: 10.6.0
        version: 10.6.0
      '@my-org/pnpm-config':
        specifier: ^2.0.0
        version: 2.1.0

packages:
  '@my-org/pnpm-config@2.1.0':
    resolution: {integrity: sha512-abc...}

  pnpm@10.6.0:
    resolution: {integrity: sha512-xyz...}

  '@pnpm/exe@10.6.0':
    resolution: {integrity: sha512-def...}
    optionalDependencies:
      '@pnpm/exe-linux-x64': 10.6.0
      '@pnpm/exe-linux-arm64': 10.6.0
      '@pnpm/exe-darwin-x64': 10.6.0
      '@pnpm/exe-darwin-arm64': 10.6.0
      '@pnpm/exe-win32-x64': 10.6.0

  '@pnpm/exe-darwin-arm64@10.6.0':
    resolution: {integrity: sha512-klm...}
    os: [darwin]
    cpu: [arm64]

  # ... other platform packages ...

snapshots:
  '@my-org/pnpm-config@2.1.0': {}
  pnpm@10.6.0: {}
  '@pnpm/exe@10.6.0':
    optionalDependencies:
      '@pnpm/exe-darwin-arm64': 10.6.0
```

The format is exactly the same as `pnpm-lock.yaml` — no special sections. Both `pnpm`
(JS) and `@pnpm/exe` (standalone) are recorded in the lockfile so the project works for
all team members regardless of their environment. Platform-specific binaries for
`@pnpm/exe` are resolved through the standard `optionalDependencies` mechanism, the same
way packages like `esbuild` or `turbo` handle platform variants.

Platform entries accumulate over time as developers on different operating systems run
`pnpm install` or `pnpm self-update` — exactly the pattern used in `pnpm-lock.yaml`.

### 2. Clean declaration in `pnpm-workspace.yaml`

Integrities are removed from `pnpm-workspace.yaml`. Declarations become plain version
specifiers, similar to regular `dependencies`:

```yaml
# pnpm-workspace.yaml
configDependencies:
  '@my-org/pnpm-config': '^2.0.0'
```

`pnpm add --config <pkg>` resolves the specifier, installs the package, writes the exact
version + integrity to `pnpm-config-lock.yaml`, and writes the specifier (without the inline
hash) to `pnpm-workspace.yaml`.

### 3. pnpm as a config dependency

The pnpm binary is declared as a config dependency in `pnpm-workspace.yaml`, just like
any other config dep. There are two packages that provide pnpm:

| Package | Description |
|---|---|
| `@pnpm/exe` | Standalone executable per platform (no Node.js required) |
| `pnpm` | JS npm package (needs Node.js) |

The lockfile always contains **both** packages, so the project works regardless of which
variant a given developer uses. At runtime, pnpm picks whichever package is appropriate
for the current environment.

```yaml
# pnpm-workspace.yaml
configDependencies:
  pnpm: '10.6.0'
  '@my-org/pnpm-config': '^2.0.0'
```

Only one specifier is needed in `pnpm-workspace.yaml` — `pnpm install` resolves both the
`pnpm` JS package and the `@pnpm/exe` standalone binary for the declared version and
writes both into `pnpm-config-lock.yaml`.

`@pnpm/exe` uses `optionalDependencies` to ship platform-specific binaries (e.g.
`@pnpm/exe-darwin-arm64`, `@pnpm/exe-linux-x64`), which are resolved through the
standard lockfile mechanism. Platform entries accumulate as developers on different
operating systems run `pnpm install`.

`pnpm install` and `pnpm self-update` resolve the declared version and write the result
into `pnpm-config-lock.yaml` using the standard lockfile format.

> **Migration**: the existing `packageManager: "pnpm@x.y.z"` field in `package.json`
> and `devEngines.packageManager` continue to be recognised for backwards compatibility.
> They trigger a deprecation notice suggesting the user add pnpm to `configDependencies`.

### 4. Startup sequence

```
process start
  │
  ├── read pnpm-config-lock.yaml (fast — small file)
  │     ├── find pnpm in configDependencies
  │     ├── pick @pnpm/exe (standalone) or pnpm (JS) based on environment
  │     ├── resolve platform-specific optional dep if using @pnpm/exe
  │     └── get { version, integrity }
  │
  ├── check $PNPM_HOME/.tools/pnpm/
  │     ├── if version matches and hash matches → use it
  │     └── if not → install from store (fast) or fetch from registry (slow, first time)
  │
  ├── switchCliVersion → spawn correct pnpm binary
  │
  └── (new pnpm process) read pnpm-lock.yaml → normal install flow
```

`$PNPM_HOME/.tools/pnpm/` holds **one** active installation, not a per-version cache.
The pnpm store (content-addressed) provides fast re-linking when switching between
versions that have already been downloaded. A version already in the store is re-linked
without a network request; only a first-time use requires a download.

### 5. More powerful config dependencies

Removing the inline integrity constraint opens the door to richer config dependency
capabilities:

- **Transitive dependencies**: config deps can declare their own `dependencies`. All
  packages (config deps and their transitive deps) are stored in the global content-
  addressable store and linked via the global virtual store (`<store-path>/links/`),
  the same mechanism used when `enableGlobalVirtualStore` is enabled for regular
  dependencies. Their resolutions are recorded in the `packages` and `snapshots`
  sections of `pnpm-config-lock.yaml`, using the same format as `pnpm-lock.yaml`.

- **Platform-specific config deps**: any config dep that ships OS/arch-specific binaries
  (e.g. a custom fetcher that wraps a native binary) can use `optionalDependencies` with
  per-platform packages, the same way `@pnpm/exe`, `esbuild`, and `turbo` do.

- **Richer plugin discovery**: lifting the `pnpm-plugin-*` naming restriction; any package
  listed in `configDependencies` may export a pnpmfile via its `main` entry point.

---

## File layout summary

```
project-root/
  pnpm-workspace.yaml          # declarations (specifiers only, no hashes)
  pnpm-config-lock.yaml        # committed lockfile for config deps + pnpm itself
  pnpm-lock.yaml               # unchanged — project dependency lockfile
  node_modules/
    .pnpm-config/
      @my-org/
        pnpm-config → <store-path>/links/<hash>  # symlink to global virtual store

<store-path>/                  # global pnpm store (pnpm store path)
  v3/                          # content-addressable store (unchanged)
  links/                       # global virtual store
    <hash>/                    # dependency-graph-hash-based directory
      node_modules/
        @my-org/
          pnpm-config/         # hard-linked from content-addressable store
```

Config dependencies always use the global virtual store (`<store-path>/links/`), the same
mechanism as `enableGlobalVirtualStore`. The project-local `node_modules/.pnpm-config/`
directory contains only symlinks into the global virtual store — no hard links or copies.
This means multiple projects sharing the same config dep version share a single copy on
disk.

---

## What stays the same

- `pnpm-lock.yaml` is unchanged. Project dependencies are unaffected.
- Config deps are installed via the global virtual store (`<store-path>/links/`), with
  `node_modules/.pnpm-config/` containing only symlinks — the same mechanism as
  `enableGlobalVirtualStore`.
- `configDependencies` can still reference private registries and custom tarballs.
- `pnpm add --config <pkg>` is the user-facing command to add a config dep.

---

## Migration path

1. `pnpm install` on a project with the old inline-hash format automatically migrates:
   - strips hashes from `pnpm-workspace.yaml` values
   - writes `pnpm-config-lock.yaml` with the extracted integrity data

2. Projects without `pnpm-config-lock.yaml` and without `configDependencies` are unaffected.

3. The `packageManager` and `devEngines.packageManager` fields are still read for backwards
   compatibility but trigger a deprecation notice suggesting the user add `pnpm` to
   `configDependencies`.

