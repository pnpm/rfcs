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
- **Minimalistic** — only config dependency resolutions, no full dependency graph.
- **Updated** by `pnpm install` and `pnpm self-update`.

```yaml
# pnpm-config-lock.yaml
lockfileVersion: '1'

# Resolved from devEngines.packageManager in package.json
pnpm:
  version: 10.6.0
  resolution:
    linux-x64:
      integrity: sha512-def...
      tarball: https://github.com/pnpm/pnpm/releases/download/v10.6.0/pnpm-linux-x64
    linux-arm64:
      integrity: sha512-efg...
      tarball: https://github.com/pnpm/pnpm/releases/download/v10.6.0/pnpm-linux-arm64
    darwin-x64:
      integrity: sha512-hij...
      tarball: https://github.com/pnpm/pnpm/releases/download/v10.6.0/pnpm-macos-x64
    darwin-arm64:
      integrity: sha512-klm...
      tarball: https://github.com/pnpm/pnpm/releases/download/v10.6.0/pnpm-macos-arm64
    win32-x64:
      integrity: sha512-nop...
      tarball: https://github.com/pnpm/pnpm/releases/download/v10.6.0/pnpm-win-x64.exe

# Resolved from configDependencies in pnpm-workspace.yaml
configDependencies:

  '@my-org/pnpm-config':
    version: 2.1.0
    resolution:
      integrity: sha512-abc...
      tarball: https://registry.npmjs.org/@my-org/pnpm-config/-/@my-org/pnpm-config-2.1.0.tgz
```

Platform entries accumulate over time as developers on different operating systems run
`pnpm install` or `pnpm self-update` — exactly the pattern used for `node@runtime:` entries
in `pnpm-lock.yaml`.

For a pure-JS package (no platform-specific binaries) the resolution is flat:

```yaml
  '@my-org/pnpm-config':
    version: 2.1.0
    resolution:
      integrity: sha512-abc...
      tarball: https://registry.npmjs.org/...
```

For a package that ships platform-specific binaries (executables), the resolution is
keyed by `{os}-{arch}`:

```yaml
  pnpm:
    version: 10.6.0
    resolution:
      darwin-arm64:
        integrity: sha512-...
        tarball: https://...
```

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

### 3. pnpm version via `devEngines.packageManager`

The required pnpm version is declared in `package.json` using the standard
`devEngines.packageManager` field — not in `configDependencies`, since pnpm is a tool/engine,
not a configuration plugin:

```json
{
  "devEngines": {
    "packageManager": {
      "name": "pnpm",
      "version": "10.6.0"
    }
  }
}
```

> **Migration**: the existing `packageManager: "pnpm@x.y.z"` field continues to be
> recognised for backwards compatibility. If both are present, `devEngines.packageManager`
> takes precedence.

`pnpm install` (and `pnpm self-update`) resolve the declared version specifier and write
the resulting integrity into the `pnpm` entry of `pnpm-config-lock.yaml`. The lockfile
records which binary source was actually resolved, not which rule selected it:

pnpm binary sources, selected automatically at resolution time:

| pnpm version | Source package |
|---|---|
| ≥ 11.0.0-alpha | `pnpm` (npm package, JS, needs Node.js) |
| < 11.0.0-alpha | `@pnpm/exe` (standalone executable per platform) |

### 4. Startup sequence

```
process start
  │
  ├── read pnpm-config-lock.yaml (fast — small file, no full YAML parsing needed)
  │     ├── find pnpm entry (written from devEngines.packageManager)
  │     ├── resolve current platform key (e.g. darwin-arm64)
  │     └── get { version, integrity, tarball }
  │
  ├── check $PNPM_HOME/.tools/pnpm/
  │     ├── if version matches and hash matches → use it
  │     └── if not → install from store (fast) or fetch from tarball (slow, first time)
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

- **Transitive dependencies**: config deps can declare their own `dependencies`. They are
  resolved into a small, separate install rooted at
  `node_modules/.pnpm-config/{packageName}/`. Their transitive deps go into the global
  store; their integrities appear in `pnpm-config-lock.yaml`.

- **Platform-specific config deps**: any config dep that ships OS/arch-specific binaries
  (e.g. a custom fetcher that wraps a native binary) can be expressed with per-platform
  resolution entries in `pnpm-config-lock.yaml`, exactly like pnpm itself.

- **Richer plugin discovery**: lifting the `pnpm-plugin-*` naming restriction; any package
  listed in `configDependencies` may export a pnpmfile via its `main` entry point.

---

## File layout summary

```
project-root/
  pnpm-workspace.yaml          # declarations (specifiers only, no hashes)
  pnpm-config-lock.yaml             # committed lockfile for config deps + pnpm itself
  pnpm-lock.yaml               # unchanged — project dependency lockfile
  node_modules/
    .pnpm-config/              # installed config deps (unchanged location)
      @my-org/
        pnpm-config/
          ...

$PNPM_HOME/
  .tools/
    pnpm/                      # single active pnpm installation
      bin/pnpm                 # → store entry via node_modules symlink
      node_modules/
        pnpm/ or @pnpm/exe/
  store/                       # content-addressed store (unchanged)
```

---

## What stays the same

- `pnpm-lock.yaml` is unchanged. Project dependencies are unaffected.
- `node_modules/.pnpm-config/` installation location is unchanged.
- `configDependencies` can still reference private registries and custom tarballs.
- `pnpm add --config <pkg>` is the user-facing command to add a config dep.

---

## Migration path

1. `pnpm install` on a project with the old inline-hash format automatically migrates:
   - strips hashes from `pnpm-workspace.yaml` values
   - writes `pnpm-config-lock.yaml` with the extracted integrity data

2. Projects without `pnpm-config-lock.yaml` and without `configDependencies` are unaffected.

3. The `packageManager` field is still read for backwards compatibility but triggers a
   deprecation notice suggesting the user migrate to `configDependencies.pnpm`.

