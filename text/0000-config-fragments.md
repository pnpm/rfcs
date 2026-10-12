# Workspace config fragments

## Summary

pnpm reads its project settings from a single file, `pnpm-workspace.yaml`. Everything that can touch that file can change every setting in it: the workspace layout and the catalogs, and also `minimumReleaseAge`, `trustPolicy`, `overrides` and `allowBuilds`.

This RFC lets a workspace split its settings across files in a fixed directory next to `pnpm-workspace.yaml`:

```text
pnpm-workspace.yaml        # packages, catalogs, everyday settings
.pnpm/
  config/
    security.yaml          # minimumReleaseAge, trustPolicy, allowBuilds, overrides, ...
    deploy.yaml            # nodeLinker, injectWorkspacePackages, ...
```

Every `*.yaml` file in `.pnpm/config/` is loaded. Nothing has to list it. The rules:

- **A top-level setting is defined in exactly one file.** Defining it in two files is an error, not an override.
- **Writes go to the file that owns the setting.** `pnpm approve-builds` updates `allowBuilds` in `security.yaml` when that file defines it.

Together, these make "who may change this setting" a question about file paths. Agent harnesses, OS sandboxes and `CODEOWNERS` already answer questions about file paths.

A config dependency can also ship fragments in a `pnpm-config/` directory, so an organisation can publish one policy for all its repositories that no repository can override. See [Fragments from config dependencies](#fragments-from-config-dependencies).

Proposed in [pnpm/pnpm#16128](https://github.com/orgs/pnpm/discussions/16128).

## Motivation

### Path rules are the only rules most tools understand

Coding agents run under rules written as path patterns: "may edit `pnpm-workspace.yaml`", "may not write `.github/**`". Claude Code, Codex, Cursor and similar harnesses have no way to say "may edit `pnpm-workspace.yaml`, except the `minimumReleaseAge` key". The same is true of `CODEOWNERS`. It can require security review for `pnpm-workspace.yaml`, but then every catalog bump needs that review too.

So today a team has two options. It can protect the whole file, in which case agents and contributors cannot do routine work (add a catalog entry, add a workspace package) without approval. Or it can leave the file open, in which case anyone who can do that routine work can also do this:

```diff
-minimumReleaseAge: 1440
+minimumReleaseAge: 0
```

or approve every pending build with `pnpm approve-builds --all`.

### Agents change settings to make errors go away

The scenario in the discussion is concrete. An install fails because a fresh version is younger than `minimumReleaseAge`, or because a publisher's trust evidence got weaker. The agent's job is to make the install pass, and the shortest path is to relax the setting that blocked it. The agent then checks that everything works locally and reports success. The change is only noticed in review, if it is noticed at all.

A related failure: an agent changes a setting that only matters in an environment it cannot see (`nodeLinker`, `injectWorkspacePackages`, `supportedArchitectures`). Everything works locally and breaks in deployment.

In both cases the fix is to put these settings somewhere that a path rule can protect, while routine settings stay where they are today.

### Secondary uses

- **`CODEOWNERS`:** `.pnpm/config/security.yaml @org/security` gates the supply-chain policy without gating catalog bumps.
- **Machine-local settings:** a gitignored `.pnpm/config/local.yaml` can hold settings a developer does not want to commit, as long as no committed file defines them (see [Local fragments](#local-fragments)).
- **Large catalogs** ([pnpm/pnpm#12606](https://github.com/pnpm/pnpm/issues/12606)): named catalogs can live in their own files.

## Threat model

It is worth being exact about what this does and does not protect, because a feature motivated by security that only looks secure is worse than no feature.

**What this RFC provides:** the settings a team cares about can be placed in files that are never needed for routine work. A file can only be protected if it is never needed for routine work. If it is, people grant write access to it and the protection is gone.

**What enforces the boundary:** something outside pnpm that restricts writes by path:

1. **Review:** `CODEOWNERS` with required reviews. This is a real boundary for what lands on the default branch and runs in CI, whoever wrote the change.
2. **Write restrictions on the agent's processes:** an OS-level sandbox (for example a Claude Code sandbox with `.pnpm/config/**` in the write deny list), a read-only mount, or file permissions.

**What is not a boundary:** a harness rule that only applies to the agent's file-editing tool. An agent that can run shell commands can write the file through the shell. It can also run `pnpm approve-builds`, and pnpm itself will then write the protected file. pnpm cannot tell an agent from a human, and does not try to. Any "are you sure" mechanism inside pnpm would be a flag the agent can pass.

**Out of scope:** command-line flags and `pnpm_config_*` environment variables. An agent that runs `pnpm install --minimum-release-age=0` gets an install with that setting. That changes nothing in the repository, and CI does not use those flags. This RFC is about committed configuration.

These are the design constraints that follow:

- **The editable file must not control the protected files.** If `pnpm-workspace.yaml` listed its fragments in an `include:` field, removing one line would drop the whole security policy. Fragments are found by location, not by reference.
- **The editable file must not be able to override a protected setting.** If later files won, an agent could redefine the key in the file it is allowed to edit. If earlier files won, the agent would only need to know the order. Both lose; a duplicate definition is an error.
- **The editable file must not weaken a protected setting through a neighbouring key.** `minimumReleaseAge: 1440` in `security.yaml` means little if `pnpm-workspace.yaml` can add `minimumReleaseAgeExclude: ['*']`. See [Claiming a setting](#claiming-a-setting).

## Detailed Explanation

### Discovery

pnpm locates the workspace root as it does today, by finding `pnpm-workspace.yaml`. It then reads every regular file matching `*.yaml` directly inside `<root>/.pnpm/config/`, in lexicographic order of file name:

- Subdirectories are not read.
- Files whose names start with `.` are ignored.
- A `*.yml` file is an error that suggests renaming it, so a typo does not silently disable a policy.

A fragment uses the same schema as `pnpm-workspace.yaml`. Any setting allowed in `pnpm-workspace.yaml` is allowed in a fragment, including `packages`, `catalogs` and `configDependencies`. A fragment may be empty.

Paths inside a fragment are resolved relative to the workspace root, not to the fragment's directory. So `patchedDependencies`, `pnpmfile`, `storeDir` and `packages` mean the same thing wherever they are written, and a setting can move between files without being rewritten.

`.pnpm/config/` also counts as a workspace-root marker. If a directory contains `.pnpm/config/` but not `pnpm-workspace.yaml`, pnpm reports an error instead of searching parent directories. Otherwise, deleting `pnpm-workspace.yaml` would make pnpm skip the fragments entirely.

Fragments are read only for the project workspace. The global package directory and the global `config.yaml` are unchanged.

### Merging

The effective project settings are the union of the top-level keys of `pnpm-workspace.yaml` and all fragments.

**A top-level key may be defined in only one file.** If it is defined in two, pnpm fails before doing any work:

```text
ERR_PNPM_DUPLICATE_SETTING  "minimumReleaseAge" is defined in more than one config file:
  .pnpm/config/security.yaml
  pnpm-workspace.yaml
Each setting can be defined in one file only.
```

Aliases count as the same key. For example, `catalog` and `catalogs.default` are already treated as the same thing, and a legacy spelling counts the same as its current one.

**There is one exception: `catalogs`.** Named catalogs merge across files, and the duplicate rule applies one level down: a catalog name may be defined in only one file. This is the use case behind #12606:

```yaml
# pnpm-workspace.yaml
catalog:
  react: ^19.0.0

# .pnpm/config/catalogs-legacy.yaml
catalogs:
  react18:
    react: ^18.3.1
```

No other key merges. In particular, `overrides`, `allowBuilds`, `packageExtensions` and `patchedDependencies` do not merge. Allowing `pnpm-workspace.yaml` to add entries to `overrides` while a protected file defines other entries would let the editable file override a protected package. These maps belong wholly to one file.

Because no key can be defined twice, the result does not depend on file order. Order only determines which file is named first in error messages.

### Claiming a setting

A file claims a setting by defining it. This is how a protected file stops the editable file from defining a related key.

To claim a setting without choosing its value, the file sets it to `null`. **A top-level `null` means "defined in this file, with pnpm's built-in default"**:

```yaml
# .pnpm/config/security.yaml
minimumReleaseAge: ~          # owned here, follows pnpm's default
minimumReleaseAgeExclude: ~
trustPolicy: no-downgrade     # owned here, with an explicit value
trustPolicyExclude: ~
allowBuilds:
  esbuild: true
overrides: ~
configDependencies: ~
pnpmfile: []                  # not ~: the default auto-loads .pnpmfile.mjs
```

With this file in place, adding `minimumReleaseAgeExclude` or `overrides` to `pnpm-workspace.yaml` fails the install. It does not quietly weaken the policy.

Why `null` rather than writing the default out:

- **Writing the default out pins it.** `minimumReleaseAge: 1440` keeps a workspace at today's default. If a later pnpm raises the default, the workspaces that protected their policy would be the ones left behind.
- **The built-in default, not the next layer.** `null` skips the global `config.yaml`. That file lives in the user's home directory, outside the repository, and an agent can often write to it. If a claimed setting fell through to it, an agent could undo the policy without touching the repository.
- **No string can serve as the marker.** Settings such as `storeDir`, `cacheDir` and `pnpmfile` accept any string, so `storeDir: default` already means a directory named `default`. A `null` cannot collide with a real value.
- **A forgotten value is safe.** `minimumReleaseAge:` with nothing after it is also `null`. It means owned here, with the default.

This defines what `null` means for every top-level setting in `pnpm-workspace.yaml` as well, not only in fragments. It is a behavior change for the TypeScript CLI. Today it copies a top-level `null` straight onto the config, so `minimumReleaseAge: ~` appears to turn the release-age check off instead of keeping the default. The Rust CLI already reads `null` as unset. The new rule makes both CLIs keep the default.

When pnpm writes to a setting claimed with `null` (for example, `pnpm approve-builds` with `allowBuilds: ~`), it replaces the `null` in the same file.

The docs will include a recommended `security.yaml` that claims the full set of supply-chain settings.

### Code that runs while config loads

Files are merged first, and then pnpmfile `updateConfig` hooks run and can rewrite any setting. A policy in a fragment therefore holds only if no untrusted code gets a chance to run after the merge. Two settings decide which code that is, and the protected file must claim both:

- **`configDependencies`:** a config dependency named `pnpm-plugin-*` (or `@pnpm/plugin-*`, `@scope/pnpm-plugin-*`) has its `pnpmfile` loaded automatically. If the protected file claims `configDependencies`, the editable file cannot add a plugin. Its resolutions live in `pnpm-lock.yaml`, which the agent can edit, but pnpm already checks config-dependency lockfile entries against their declarations and the registry (`createConfigDepsVerifier`). An edited lockfile cannot swap the package behind a protected declaration.
- **`pnpmfile`:** when this setting is unset, pnpm loads `.pnpmfile.mjs` (or `.pnpmfile.cjs`) from the root if the file exists. Claiming it with `null` keeps that default, so an agent that creates `.pnpmfile.mjs` gets its hooks loaded. A workspace without a pnpmfile claims it with `pnpmfile: []`, which turns off the automatic loading. A workspace with a pnpmfile names it explicitly and protects its path along with the fragment.

pnpm already treats the automatically loaded root pnpmfile as untrusted in one place: `self-update` skips it, because its `updateConfig` hook could rewrite the release-age policy. Claiming `pnpmfile` applies the same reasoning to every command.

### Fragments from config dependencies

An organisation usually wants one supply-chain policy across all its repositories, not a copy of `security.yaml` in each. Today the only way a package can contribute settings is a plugin's `updateConfig` hook. That is code, and it can change anything. This RFC lets a config dependency ship fragments instead. A fragment is data, and it follows the same one-file-per-setting rule as local fragments.

```yaml
# .pnpm/config/security.yaml
configDependencies:
  '@acme/pnpm-policy': 3.1.0
pnpmfile: []
```

```text
@acme/pnpm-policy/
  package.json
  pnpm-config/
    security.yaml      # minimumReleaseAge, trustPolicy, overrides, ...
    catalogs.yaml      # catalogs shared across the org
```

**Discovery.** Every `*.yaml` file directly inside a config dependency's `pnpm-config/` directory is a fragment. The `.pnpm/config/` discovery rules apply. A config dependency without that directory contributes nothing, so existing plugins are unaffected. Package fragments from different config dependencies are read in the lexicographic order of the dependency names.

**Merging.** Package fragments join the merge alongside `pnpm-workspace.yaml` and the local fragments, under the same duplicate rule. If `@acme/pnpm-policy` defines `minimumReleaseAge`, defining it in any repository file is an error, and so is defining it in another config dependency. Named catalogs merge as usual, so an org-wide catalog and a repository-local catalog can sit side by side under different names. A `null` claim means the built-in default, as it does locally.

**What a package fragment may not define:**

- `configDependencies` and `pnpmfile`, which decide what code runs during config loading. Config dependencies are installed once, from the repository's own declaration. They do not pull in further config dependencies.
- `bootConfig`, which governs installing the package itself (see below).
- `packages` and settings whose values are paths (`patchedDependencies`, `storeDir`, `virtualStoreDir`, and the like). These describe a particular repository, and a path inside a package has no meaning relative to the workspace root.

A package fragment that defines any of these fails config loading, naming the package and the setting.

**Timing.** Config dependencies are installed after the repository's files are read, because the list of config dependencies comes from those files. Package fragments are merged as a second step: after config dependencies are installed and before any pnpmfile is loaded, so `updateConfig` hooks see the merged result. The duplicate check runs again over the full set of files.

**Failing closed.** A policy that silently fails to load is worse than none. Commands that install or resolve dependencies already fail when a config dependency cannot be installed. Only `pnpm config` commands tolerate that today, so that auth can be set up before the first install. They keep tolerating it, and `pnpm config list` and `pnpm config get` mark their output as incomplete when a config dependency was not installed.

Writes need a stricter rule, because pnpm cannot tell which settings an uninstalled config dependency owns. Consider `pnpm config set --location project minimumReleaseAge 0` run while `@acme/pnpm-policy` is not installed. pnpm would see the setting as undefined and add it to `pnpm-workspace.yaml`, and every later config load would then fail with a duplicate error. So while any config dependency is not installed, `pnpm config set --location project` and `pnpm config delete --location project` refuse to run. The error names the config dependencies that are not installed and says to run `pnpm install` first. Writes to the global `config.yaml` (`--location global`) are unaffected, so that auth can still be configured. Every other command that writes project settings already requires config dependencies to be installed, so the same rule holds for them without extra work. The source-aware writers still assert that ownership is complete, so a future command that tolerates missing config dependencies cannot reintroduce this problem.

**Writes.** pnpm never writes into a config dependency. A command that needs to change a setting owned by a package fragment fails with a pointer to the owner:

```text
ERR_PNPM_SETTING_OWNED_BY_CONFIG_DEPENDENCY  "allowBuilds" is defined by @acme/pnpm-policy@3.1.0 (pnpm-config/security.yaml).
Change it in that package and update the config dependency.
```

So a package fragment should own only the settings meant to be changed centrally. A setting that each repository should manage, such as `allowBuilds` in most organisations, belongs in the repository's own files.

**Settings for the boot phase: `bootConfig`.** Config dependencies are resolved and verified before their fragments are loaded, so a package fragment's settings cannot apply to installing that package. Today this phase uses `minimumReleaseAge` and `minimumReleaseAgeExclude` (`createConfigDepResolvers`). This stops working once a policy package owns `minimumReleaseAge`. The value the package ships is not loaded yet when the package is resolved, and the repository cannot define the key without a duplicate error. Config dependencies would be stuck on the built-in default, and an org could not get a new policy version into a repository on the day it is published.

The same question comes up for the other thing pnpm installs before running a command: the pnpm version requested by `packageManager`. Both are recorded in the first document of `pnpm-lock.yaml`, under `configDependencies` and `packageManagerDependencies`. The docs currently call it the env document. Both are resolved under the same release-age settings (`createConfigDepResolvers`, `self-updater/resolvePnpmVersion.ts`). This RFC calls that phase **boot**, renames the env document to the **boot document** to match, and gives the phase its own block of settings:

```yaml
# .pnpm/config/security.yaml
configDependencies:
  '@acme/pnpm-policy': 3.1.1
bootConfig:
  minimumReleaseAge: 0
```

- **Scope.** `bootConfig` applies to resolving and verifying everything in the boot document: config dependencies and the package-manager dependency. Regular dependencies never read it.
- **Allowed keys.** Only settings that govern how these packages are resolved: `minimumReleaseAge`, `minimumReleaseAgeExclude`, `minimumReleaseAgeStrict` and `minimumReleaseAgeIgnoreMissingTime`, and `trustPolicy` and its related settings if they are ever applied to this phase. Any other key is an error. In particular, registries are not accepted. The package-manager bootstrap takes registries only from non-project `.npmrc` files, and the error says so.
- **Fallback, per key.** pnpm uses the value in `bootConfig` if it is set. Otherwise it uses the top-level setting of the same name, if the repository's own files define it, and otherwise the built-in default. This keeps today's behavior for repositories without policy packages: a repository with `minimumReleaseAge: 10080` keeps that cutoff for its config dependencies and its pnpm version. A top-level value that comes from a package fragment never applies to this phase. `bootConfig: ~` claims the block, with every key falling back.
- **Repository files only.** A package fragment may not define `bootConfig`, since that would let a package set the policy for its own installation.
- **One file owns the whole block.** Its keys cannot be split across files. It belongs in the protected file next to `configDependencies`. Each entry in this phase is a reviewed, exact entry in that file, so a shorter delay here costs less than a shorter delay for the whole dependency graph.
- **No new trust.** `bootConfig` sets only what the top-level release-age settings already set for this phase today. It lets a repository set these values separately; it does not let project config reach anything that it cannot reach today.

**Changing the policy.** A policy change is a new version of the package. It reaches a repository as an edit to `configDependencies` in the protected local fragment, and that edit goes through the same review as any other policy change. The resolution recorded in `pnpm-lock.yaml` is checked against that declaration and the registry, as described above.

`pnpm config get minimumReleaseAge --verbose` shows the package source:

```text
1440  (@acme/pnpm-policy@3.1.0: pnpm-config/security.yaml)
```

### Writes

pnpm commands that write project settings send each change to the file that currently defines the setting:

| Command | Setting written |
| --- | --- |
| `pnpm approve-builds` | `allowBuilds` |
| `pnpm add` / `update` with catalogs | `catalog`, `catalogs.<name>` |
| `pnpm patch-commit` | `patchedDependencies` |
| `pnpm audit --fix`, `pnpm audit --ignore` | `overrides`, the audit ignore list |
| install with `minimumReleaseAgeExclude` / `trustPolicyExclude` prune | the exclude lists |
| `pnpm config set --location project` | any |

- If the setting is defined in a fragment, the fragment is edited and `pnpm-workspace.yaml` is left alone.
- If it is not defined anywhere, it is added to `pnpm-workspace.yaml`, as today.
- A new named catalog is added to `pnpm-workspace.yaml`. A new entry in an existing catalog goes to the file that defines that catalog.
- Removing the last entry of a map keeps the key with an empty value, so the claim stays where it is. (Today, an emptied list or map is removed from the file.)

pnpm never moves a setting between files and never deletes a fragment. A write that fails because of permissions reports the file that could not be written and stops. It does not fall back to `pnpm-workspace.yaml`.

### Inspecting the result

`pnpm config list` and `pnpm config get <key>` show the source file of each project setting:

```text
$ pnpm config get minimumReleaseAge --verbose
1440  (.pnpm/config/security.yaml)
```

### Local fragments

A gitignored fragment, for example `.pnpm/config/local.yaml`, is loaded like any other fragment and follows the same duplicate rule. It can set keys that no committed file defines. It cannot override a committed value; per-machine overrides of committed settings already belong in the global `config.yaml`, which keeps its current precedence.

### Interaction with the rest of pnpm

- **Lockfile:** the settings recorded in `pnpm-lock.yaml` (overrides, `packageExtensionsChecksum`, catalogs, and so on) are computed from the merged result. Moving a setting between files without changing it does not change the lockfile.
- **Cache keys and up-to-date checks:** anything that keys on the contents or modification time of `pnpm-workspace.yaml` (workspace state, `optimisticRepeatInstall`, the boot document check for config dependencies) must also key on the fragments. Adding or removing a fragment counts as a change.
- **`pnpm-workspace.yaml` stays required.** It is still the root marker that every tool in the ecosystem looks for. A workspace with everything in fragments has an empty or near-empty `pnpm-workspace.yaml`, which is allowed.

## Rationale and Alternatives

### An `include:` list in `pnpm-workspace.yaml`

This is closest to Cargo's `include` and to Git's `[include]`. It is explicit, and it supports any path. But it fails the first constraint: the file an agent is allowed to edit decides whether the protected file is read at all. An `include:` list is good for organising files, but it cannot be a security boundary on its own. It would remain safe only if `pnpm-workspace.yaml` were itself protected, and that is the situation this RFC exists to avoid.

An include list could be added later for organisation. If it is, files loaded through it must follow the same duplicate rule, and the fixed directory must still be loaded unconditionally.

### Precedence instead of an error ("fragments override `pnpm-workspace.yaml`")

The discussion's first suggestion. It is the familiar model (`conf.d`, layered `.npmrc`). It also lets a protected file pin a value without claiming neighbouring keys.

It fails silently in both directions. A value that a contributor writes in `pnpm-workspace.yaml` has no effect, and nothing reports it. When precedence is used to protect a setting, the editable file can still define a neighbouring key the protected file forgot about. With the error, the forgotten key is still a gap, but every attempt to define a key twice is reported, and both files are named in the message.

### The full `.pnpm/` project directory

The discussion also sketches `.pnpm/` as a home for all of pnpm's project files: `workspace.yaml`, the lockfile, the pnpmfile and patches. That layout is compatible with this RFC: fragments in `.pnpm/config/` work whether or not the other files move there. But moving the lockfile involves ecosystem tooling, CI cache keys and package-manager detection. This RFC solves the settings problem without waiting for that, and chooses a directory name that the larger layout can adopt unchanged.

### Generate `pnpm-workspace.yaml` from parts

The discussion considers a hook that assembles `pnpm-workspace.yaml` from several files. The generated file would need to be protected (otherwise editing it directly bypasses the parts), pnpm's own writes would go to the generated file, and the parts and the output could drift apart. This is the same feature with more ways to go wrong.

### Other ways to claim a setting without a value

- **A string marker** (`minimumReleaseAge: default` or `inherit`): this conflicts with settings that accept any string. `storeDir: default` is a valid directory today.
- **A YAML tag** (`minimumReleaseAge: !default`): this cannot collide with a real value, but generic YAML readers reject unknown tags (js-yaml throws). Tools that read pnpm config, such as Renovate reading catalogs, would fail on any file that uses one. Both CLIs' YAML writers would also need to keep the tag intact when they rewrite the file.
- **A reserved key** (`claim: [minimumReleaseAge, trustPolicy]`): this works with any YAML reader. But it separates the claim from the setting, which makes the protected file harder to review, and it reserves a name that no future setting can use.

`null` has none of these costs. Its one cost is the behavior change described in [Claiming a setting](#claiming-a-setting).

### Per-key permissions inside pnpm

pnpm could have a setting that marks keys as read-only, or require a flag to change them. That setting would live in a file the agent can edit, and the flag can be passed by the agent. Enforcement has to happen outside the process being restricted. pnpm's job is to make the policy something that can be restricted from outside.

## Implementation

### TypeScript CLI (`pnpm11/`)

- **`workspace/workspace-manifest-reader`:** add a reader that returns the merged manifest together with a key → source file map. It enforces the duplicate rule and the `catalogs` exception, and validates each fragment with `validateWorkspaceManifest`.
- **`workspace/root-finder`:** treat `.pnpm/config/` without `pnpm-workspace.yaml` as an error at that directory.
- **`config/reader`** (`getConfig/localSettings.ts`, `getConfig/workspaceManifestSettings.ts`): consume the merged manifest; keep the source map on the config for `pnpm config` output and for writers. In `addSettingsFromWorkspaceManifestToConfig`, a top-level `null` resets the setting to its value from `defaultOptions`. It is not assigned as-is, and it is not skipped either, since skipping would leave a value from the global `config.yaml` in place.
- **`workspace/workspace-manifest-writer`:** `updateWorkspaceManifest` currently writes to one file. It becomes source-aware: it groups the updated fields by owning file, patches each file with the existing `patchDocument` path, and keeps emptied maps. Its callers (`approveBuilds`, `updatePatchedDependencies`, the audit fix/ignore commands, `policyHandlers`, `handleIgnoredBuilds`, `configSet`, the catalog writers in `add`/`update`) pass changes through unchanged and need no per-command logic.
- **`config/writer` and `config/commands`:** route `--location project` writes the same way; add the source to `config get`/`list`.
- Add the fragment paths to the inputs of the workspace state and optimistic-repeat-install checks.
- **`pnpm/src/getConfig.ts`** (`installConfigDepsAndLoadHooks`): after `installConfigDeps` and before `loadPnpmfileHooks`, read `pnpm-config/*.yaml` from each installed config dependency in `node_modules/.pnpm-config`, reject the settings package fragments may not define, re-run the merge with the duplicate rule, and apply the result. Writers refuse to change a setting owned by a package fragment. They also refuse any project write while a config dependency is not installed (`config/commands` for `config set`/`delete`, plus an assertion in the source-aware writer).
- **`installing/env-installer`** (`createConfigDepResolvers`, `createConfigDepsVerifier`): read the release-age settings from `bootConfig`, falling back per key as described above. The resolver and the verifier must keep using the same cutoff.
- **`engine/pm/commands`** (`self-updater/resolvePnpmVersion.ts`) and the `packageManager` resolution in `pnpm/src`: the same `bootConfig` settings.

### Rust CLI (`pnpm/crates/`)

Parity is required: if one CLI ignores fragments, a protected setting is not applied when that CLI is used.

- **`config`** (`loading.rs`, `layout.rs`): the same discovery, merge and duplicate rule. A claim with `null` must use pnpm's built-in default, not a value from the global `config.yaml`.
- **`workspace-manifest-writer`:** the same source-aware writes.
- **`workspace-state`:** the same cache inputs.
- **`env-installer` / config loading:** the same second merge pass over package fragments after config dependencies are installed, and `bootConfig` for config dependencies and the package-manager dependency.

Shared fixtures in both test suites cover: a duplicate across files; the `catalogs` merge and a duplicate catalog name; a write routed to a fragment; an emptied map that stays in its file; a `*.yml` error; `.pnpm/config/` present without `pnpm-workspace.yaml`; a package fragment that conflicts with a local fragment; a package fragment that defines a forbidden setting; and a write refused because a package fragment owns the setting.

### Docs

- A page on splitting settings, including the recommended `security.yaml` and the threat model above stated plainly.
- Example rules for `CODEOWNERS` and for at least one agent sandbox.
- Rename the env lockfile / env document to the **boot document** in `lockfile.md`, `config-dependencies.md` and anywhere else it appears. Keep the old anchors, so existing links to "env document" still resolve. The lockfile format does not change, since the first document has no `env` key.

The code still uses the old name (`EnvLockfile`, `readEnvLockfile` and `env_lockfile` in both CLIs, and the published `@pnpm/installing.env-installer` package). Renaming it is a mechanical follow-up and is not required to ratify this RFC. New user-facing messages use "boot document" from the start.

## Prior Art

- **Cargo** `include` in `.cargo/config.toml`: explicit list, later files override earlier ones. Organisational, not a boundary.
- **Git** `[include]` / `[includeIf]`: ordered imports, last value wins.
- **Bazel** `.bazelrc` `import` / `try-import`: ordered imports; `try-import` is commonly used for a gitignored `user.bazelrc`, which is the local-fragment use case.
- **systemd, sudo, apt** `*.d/` directories: fixed-location discovery in lexical order, last wins. `sudoers.d` is the closest precedent for a security policy split by path so that separate parts can have separate owners.
- **`CODEOWNERS`**: the reason path granularity matters; the enforcement mechanism this RFC is designed around.

None of these treats a duplicate definition as an error. They are configuration systems where overriding is the purpose. Here overriding is the attack.

## Unresolved Questions and Bikeshedding

- **Directory name.** `.pnpm/config/` matches the larger layout sketched in the discussion. Alternatives: `pnpm-workspace.d/` (the `conf.d` convention, next to the file it extends) or `pnpm-workspace.*.yaml` siblings (no new directory; a glob rule that harnesses already support). Siblings are the easiest to find, but they mix fragments into the repo root.
- **Is `null` already meaningful for any setting?** Before ratification, audit the settings in both CLIs for any where `null` is documented or relied on, for example as a way to turn something off. Each one either moves to an explicit value or is listed as an exception.
- **How a config dependency declares its fragments.** A `pnpm-config/` directory is proposed. Alternatives are a `package.json` field listing the files, or a name convention like the one for plugins (`pnpm-config-*`). A directory needs no extra metadata. A field makes the contribution visible in the package's manifest.
- **Name of `bootConfig`.** "Boot" means loading what is needed before the main run, and pnpm's code already calls the pnpm download the package-manager bootstrap. Alternatives considered:
  - `preflightConfig`: has no existing meaning in pnpm, but suggests checks more than installs.
  - `envConfig`: matches the "env document" term already in the docs and needs no rename. It was rejected because in many tools "env config" means settings for a deployment environment (development, staging, production). `envConfig: { minimumReleaseAge: 0 }` reads as the value for some environment, and that misreading is likely here, since this RFC is partly about settings that differ between local and deployed environments. "Env" is also read as environment variables.
  - `bootstrapConfig`: inside pnpm, "bootstrap" already means the trusted package-manager download specifically, and `lerna bootstrap` means installing a monorepo's dependencies.
  - A suffixed setting such as `minimumReleaseAgeForConfigDeps`: needs one more top-level setting for each setting this phase overrides.

  Whatever the name, it must be a separate top-level key. Nesting it under `minimumReleaseAge` would put a policy package's value and the repository's boot value under one key, which the duplicate rule forbids.
- **Path-valued settings from packages.** Sharing `patchedDependencies` across an org would be useful, but it needs a rule for resolving paths inside the package. Forbidden for now.
- **Should any map other than `catalogs` merge?** Candidates are `packageExtensions` and `publishConfig`-style maps that carry no policy. Merging only `catalogs` is the conservative start.
- **Should pnpm warn about unclaimed neighbours?** For example, a fragment defines `minimumReleaseAge` but not `minimumReleaseAgeExclude`. It would be helpful, but it means maintaining a list of related settings.
- **Local overrides.** Is "a local fragment cannot override a committed value" too strict for settings such as `storeDir`, or is the global `config.yaml` enough?
- **`pnpm config set` for a setting that is not defined anywhere.** Should there be a `--file` option to write it to a chosen fragment, so it does not land in `pnpm-workspace.yaml` first and need moving?
