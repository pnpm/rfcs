# Workspace config fragments

## Summary

pnpm reads its project settings from a single file, `pnpm-workspace.yaml`. Everything that can touch that file can change every setting in it: the workspace layout and the catalogs, and also `minimumReleaseAge`, `trustPolicy`, `overrides` and `allowBuilds`.

This RFC lets a workspace split its settings across explicitly included files and config dependencies:

```yaml
# pnpm-workspace.yaml (protected)
configDependencies:
  '@acme/pnpm-policy': 3.1.0
pnpmfile: []
include:
  ./config/security.yaml: '*'
  ./config/catalog.yaml: [catalog]
  ./config/workspace.yaml: [packages]
  '@acme/pnpm-policy': [minimumReleaseAge, trustPolicy]
```

- **The root manifest controls delegation and must be protected.** Everyday edits go to the included files whose paths can have separate permissions or `CODEOWNERS`.
- **A local file can have a setting list or `'*'`.** A list permits and reserves exactly those settings, including settings the file omits. `'*'` lets a trusted file define any allowed setting without updating the root mapping every time it adds a key.
- **A config dependency always has an explicit setting list.** A package update cannot introduce another setting without a reviewed change to the root mapping.
- **Each setting has one owner.** Conflicting ownership or definitions are errors, not overrides. Writes go to the owning local file; package-owned settings are read-only.

There is no special fragment directory and no automatic discovery. A local path identifies a file; a config dependency name identifies a package whose manifest declares its config entry file. External tools can find explicitly delegated settings through the root index without scanning every fragment.

Proposed in [pnpm/pnpm#16128](https://github.com/orgs/pnpm/discussions/16128). Explicit includes and setting delegation build on [gluxon's review](https://github.com/pnpm/rfcs/pull/38#pullrequestreview-5528843769).

## Motivation

### Path rules are the only rules most tools understand

Coding agents run under rules written as path patterns: "may edit catalogs", "may not write security policy". Agent harnesses, OS sandboxes and `CODEOWNERS` can enforce access to a whole file, but cannot generally distinguish individual YAML settings within it.

Today, protecting `pnpm-workspace.yaml` also gates every catalog bump and workspace package change. Leaving it open lets anyone doing that routine work also change `minimumReleaseAge: 1440` to `minimumReleaseAge: 0` or approve every pending build.

Explicit includes let a team protect the root manifest as the ownership index and delegate everyday settings to other files. Catalog updates and changes to workspace package patterns then need no edit to the protected root. Adding a new delegation does need review.

### Agents change settings to make errors go away

An install fails because a fresh version is younger than `minimumReleaseAge`, or because a publisher's trust evidence got weaker. An agent whose job is to make the install pass can relax the setting, check locally and report success. A related failure is changing a setting that only matters in deployment, such as `nodeLinker` or `injectWorkspacePackages`.

A delegated file may define only its assigned settings. An agent allowed to update the catalog cannot add `minimumReleaseAgeExclude: ['*']` to that file, even when no policy file currently defines that neighbouring key.

### Reducing churn and limiting package authority

Requiring a list for every local file would repeat each new setting in the protected root. Trusted policy files can use `'*'` instead. Their existing definitions still own settings, and ordinary additions require only an edit to the policy file.

Config dependencies receive explicit lists because their contents come from packages. The root declares what a package may control before pnpm installs it. A package update that adds an undelegated setting fails validation instead of silently acquiring authority over that setting.

### Secondary uses

- **`CODEOWNERS`:** protect `pnpm-workspace.yaml` and `config/security.yaml` with the security or infrastructure team, while `config/catalog.yaml` has broader ownership.
- **Large catalogs** ([pnpm/pnpm#12606](https://github.com/pnpm/pnpm/issues/12606)): named catalogs can live in separately delegated files.
- **Organisation policy:** one config dependency can supply shared values for its delegated settings across repositories.
- **Tooling:** a tool interested in `catalog` can locate its explicitly assigned file through the root mapping. Unrestricted includes still require inspection when looking for settings not explicitly mapped.

## Threat model

**What this RFC provides:** a protected root declares which sources are loaded and which settings less-trusted sources may control. The settings a team cares about can remain in files that routine work never needs to edit.

**What enforces the file boundary:** something outside pnpm that restricts writes by path:

1. **Review:** `CODEOWNERS` with required reviews protects what lands on the default branch and runs in CI.
2. **Process write restrictions:** an OS-level sandbox, a read-only mount or file permissions restricts the agent's processes, including commands that ask pnpm to write configuration.

A harness rule that covers only an agent's file-editing tool is insufficient. A shell-capable agent can write through the shell or run `pnpm approve-builds`, which writes through pnpm. pnpm cannot distinguish an agent from a human.

These are the design constraints:

- **Protect `pnpm-workspace.yaml`.** An actor who can edit the index can remove a policy include, change its delegation or configure hooks. This design deliberately requires root protection; routine settings move into delegated files.
- **Use explicit lists for broadly editable files.** A file included with `'*'` is trusted to introduce settings, so it must be protected too. A setting list prevents neighbouring-key bypasses without needing to anticipate every security-sensitive key.
- **Reject conflicting ownership and undelegated settings.** Ordering must never let one source override another.
- **Protect referenced paths and their targets.** Included files, parent directories and any symlink targets need appropriate write restrictions. The choice of directory is not itself a security boundary.

Command-line flags, `pnpm_config_*` environment variables and global `config.yaml` are outside the committed repository boundary and retain their existing precedence. Delegation without a value does not pin an effective value: lower layers and built-in defaults may still apply. A team that needs a fixed policy supplies explicit values.

pnpmfile `updateConfig` hooks run after configuration is merged and can rewrite even explicit protected values. Hook files and the settings selecting them must be trusted. This RFC limits declarative configuration; it does not sandbox executable hooks or re-enforce values after hooks run.

## Detailed Explanation

### Includes

pnpm locates the workspace root as it does today, using `pnpm-workspace.yaml`. Only that root file may define `include`. It is a mapping from a source to either a list of settings or the string `'*'`:

```yaml
include:
  ./config/security.yaml: '*'
  ./config/catalog.yaml: [catalog]
  ./config/frontend.yaml: [catalogs.frontend]
  '@acme/pnpm-policy': [minimumReleaseAge, trustPolicy]
```

**Local sources** start with `./` or `../` and are resolved relative to the workspace root. They name individual YAML files, with either `.yaml` or `.yml` extensions. Globs, directories, URLs and absolute paths are not accepted. An include outside the workspace is subject to the same trust requirements as any other include. Paths used inside a local fragment, such as `packages`, `patchedDependencies` and `pnpmfile`, are also resolved relative to the workspace root, so moving a setting between local files does not change its meaning.

**Package sources** are exact names of declared `configDependencies`, including scoped names. The version and integrity remain governed by the existing config-dependency declaration and lockfile verification. An include cannot fetch an undeclared package, select a different version, use a package subpath or fall back to a regular dependency or Node resolution. Package sources require a setting list; `'*'` is rejected.

The `include` mapping itself is structural metadata, not a setting that can be delegated. Included files may not define `include`, so there are no nested includes. All local fragments are read before resolving package sources; a protected local fragment may own `configDependencies`.

Every referenced local file must exist, be readable and contain valid workspace settings; an empty YAML document is allowed. Missing, invalid or unreadable files are errors. Duplicate YAML keys, repeated sources after path normalization or symlink resolution, and includes of the root manifest itself are errors. Files not referenced by `include` are never loaded, regardless of directory or filename.

Workspaces without `include` keep today's behavior. The global package directory and global `config.yaml` are unchanged.

### Delegation and ownership

A setting list both **permits** and **reserves** its settings. Lists contain canonical setting names or supported aliases; unknown names and invalid selectors are errors. An empty list permits no settings. For example:

```yaml
# pnpm-workspace.yaml
include:
  ./config/catalog.yaml: [catalog]
  ./config/deploy.yaml: [nodeLinker, injectWorkspacePackages]
```

`catalog.yaml` may define only `catalog`. Defining `minimumReleaseAge` there is an error. `deploy.yaml` owns both listed settings even if the file is empty. Another file, including the root manifest, cannot define either setting. A later write to an omitted `nodeLinker` goes directly to `deploy.yaml`.

An unrestricted local include (`'*'`) owns the settings it actually defines. It does not reserve omitted settings and does not get precedence over another source. A trusted security file can therefore add a setting without changing the root mapping, but cannot define a setting already delegated to another source.

Settings defined directly in the root belong to the root. Settings with neither a delegation nor a definition have no project owner and retain today's inherited or built-in values. Aliases are canonicalized before checking permission and ownership; a legacy spelling cannot evade a delegation or create a separate owner.

Each setting has one owner. Overlapping lists, a definition outside its source's list, and a definition that conflicts with another source's reservation are errors even if one owner supplies no value. Repeating a selector within a list is also an error. Validation names the setting, the source and the conflicting definition or delegation:

```text
ERR_PNPM_DUPLICATE_SETTING  "minimumReleaseAge" is delegated to @acme/pnpm-policy
but is also defined in ./config/security.yaml.
Each setting can have one owner only.
```

The result does not depend on include order. Includes are processed in root mapping order only for deterministic diagnostics. No ordinary map merges across owners: `overrides`, `allowBuilds`, `packageExtensions` and `patchedDependencies` each belong wholly to one source.

### Catalogs

Catalogs retain one owner per catalog name. Their entries are never split across sources. The default catalog can be written as `catalog` or `catalogs.default`; both names identify the same ownership slot.

Setting lists support these selectors:

- `catalog` (or `catalogs.default`) reserves the default catalog.
- `catalogs.<name>` reserves one named catalog.
- `catalogs` reserves all catalogs, including the default and names added later. This lets a delegated catalog file grow without editing the root for every new catalog.

`catalogs.<name>` treats the entire suffix as the catalog name, not a general nested property path. Nested selectors for other settings are not supported. A whole-`catalogs` reservation conflicts with any reservation or definition of an individual catalog in another source. A file assigned only one named catalog cannot define another.

```yaml
# pnpm-workspace.yaml
include:
  ./config/catalog.yaml: [catalog]
  ./config/catalogs-legacy.yaml: [catalogs.react18]

# config/catalog.yaml
catalog:
  react: ^19.0.0

# config/catalogs-legacy.yaml
catalogs:
  react18:
    react: ^18.3.1
```

For unrestricted local files and the root, ownership follows the names actually present. Defining catalog `foo` in two sources is an error even when its entries do not overlap. When `pnpm add` saves an entry to `foo`, there is exactly one destination.

### Omitted values and null

An explicit delegation reserves ownership without requiring a placeholder in the included file. Omitting a delegated setting supplies no value at the project layer; normal inherited configuration and built-in defaults still apply. This does not force the built-in default or bypass global configuration.

For unrestricted files, key presence establishes ownership, including when the value is null. Existing null values keep their setting-specific semantics. Where null means unset, it contributes no override; where it already clears or disables a value, that behavior remains.

For example, pnpm v12 with global `minimumReleaseAge: 0` and project `minimumReleaseAge: null` keeps `0`. `hoistPattern: null` disables private hoisting, and `scriptShell: null` clears an inherited shell. Moving these values into fragments does not change their meaning. No new YAML marker or universal null-to-default conversion is introduced.

A null placeholder is no longer needed to reserve a setting with an explicit delegation. Switching between an omitted setting and a configured value edits only the included file; its root mapping stays the same.

### Code that runs while config loads

Files are merged first, then pnpmfile `updateConfig` hooks run. In a workspace using delegation, broadly editable files must not be assigned hook-selection settings. Those settings belong in the protected root or another trusted local file:

- **`configDependencies`:** existing plugin-named config dependencies can load hooks automatically. Their declaration must be protected. Config-dependency lockfile entries remain checked against declarations and the registry by the existing verifier.
- **`pnpmfile`:** without an explicit value, pnpm may automatically load `.pnpmfile.mjs` or `.pnpmfile.cjs` from the root. A workspace without a pnpmfile uses `pnpmfile: []` to disable workspace pnpmfile loading. Otherwise, it names a trusted file and protects its path. Omission or a null placeholder is not an explicit disable.
- **`globalPnpmfile`:** project configuration can select this hook too. Keep its selector in trusted configuration and protect any selected file. An omitted value may inherit a trusted global value.

`pnpmfile: []` does not disable a configured global pnpmfile. Global hooks can rewrite protected values after merging; the non-project configuration selecting them and the hook files themselves must be trusted separately. See [globalPnpmfile](https://pnpm.io/pnpmfile#globalpnpmfile).

A package's declarative setting list does not constrain executable hooks supplied by that package. Including a config entry does not itself execute code, but any hooks loaded under existing plugin rules still require full code trust.

### Fragments from config dependencies

An organisation can publish shared declarative policy as a config dependency:

```yaml
# pnpm-workspace.yaml (protected)
configDependencies:
  '@acme/pnpm-policy': 3.1.0
pnpmfile: []
include:
  ./config/catalog.yaml: [catalog]
  '@acme/pnpm-policy': [minimumReleaseAge, trustPolicy]
```

The package declares one YAML config entry in its `package.json`:

```json
{
  "name": "@acme/pnpm-policy",
  "version": "3.1.0",
  "pnpm": {
    "config": "./config.yaml"
  }
}
```

```yaml
# @acme/pnpm-policy/config.yaml
minimumReleaseAge: 1440
trustPolicy: no-downgrade
```

`pnpm.config` is a package-relative path to a regular `.yaml` or `.yml` file. It must stay inside the installed package after path and symlink resolution. It cannot name a directory, glob, URL or executable entry point. Missing or invalid metadata, an escaping path or an unreadable entry is an error when the package is included. A config dependency not named in `include` contributes no declarative configuration, regardless of whether it has this field; existing plugin behavior is unchanged.

**Authority.** The root setting list is authoritative. The package file may define only those settings; anything else fails loading. A package update cannot expand its own list. Omitted settings remain reserved for the package, even before it has been installed, and their values inherit normally when the package has loaded successfully. Another source cannot supply a value for a package-owned setting.

**What a package may not define or be delegated:**

- `include`, `configDependencies`, `pnpmfile` and `globalPnpmfile`, which control configuration loading or select code.
- `bootConfig`, which governs installing the package itself.
- `packages` and path-valued settings such as `patchedDependencies`, `storeDir` and `virtualStoreDir`. These describe a particular repository and need a separate path-resolution design before sharing them from packages.

These restrictions apply even if a prohibited setting appears in the root delegation but the package omits it. Delegation is validated before package installation.

**Timing.** Read the root and all local fragments, validate their ownership and package delegations, then resolve and verify config dependencies using the repository's boot policy. Load the explicitly included package entries before any pnpmfile hooks run, validate their allowed settings, and apply their values to the merged configuration. There is no package directory scan or recursive config dependency installation.

**Failing closed.** Installation and resolution commands must fail when an included package or its entry cannot be loaded or validated. They must not continue using inherited defaults in place of a missing policy. `pnpm config` commands retain their existing allowance for unavailable config dependencies so credentials can be configured before the first install. Their `get` and `list` output marks configuration as incomplete; it must not present a package-owned value as final when its source is unavailable. Invalid delegation, malformed entries and undelegated settings are configuration errors, not an excuse to ignore the package.

**Writes.** pnpm never writes into a config dependency. A command that needs to change a package-owned setting fails even if the package is not installed:

```text
ERR_PNPM_SETTING_OWNED_BY_CONFIG_DEPENDENCY  "minimumReleaseAge" is delegated to @acme/pnpm-policy.
Change it in that package and update the config dependency.
```

Ownership is known from the root mapping and local files before package installation. `pnpm config set --location project` and `delete` can therefore write unrelated local settings while a config dependency is unavailable, provided local configuration and delegations are valid. They cannot create a local definition of a package-owned setting. Global writes remain unaffected. Other commands keep their existing requirement to load config dependencies before proceeding.

A package should own only settings meant to change centrally. Repository-managed settings, such as `allowBuilds` in many organisations, belong in local files instead.

**Changing the policy.** A policy update is a new package version, applied through a reviewed edit to the protected `configDependencies` declaration. Assigning another setting also requires a change to `include`. Updating a package cannot silently acquire authority over an undelegated setting.

### Settings for the boot phase: `bootConfig`

Config dependencies must be resolved and verified before their config entries can be read. A package's policy therefore cannot govern installing that same package. The same question applies to the pnpm version requested by `packageManager`: both are resolved before the main command, using release-age settings, and both are recorded in the first document of `pnpm-lock.yaml`.

This RFC calls that phase **boot**, renames the env document to the **boot document**, and gives the phase its own settings block:

```yaml
# pnpm-workspace.yaml (protected)
configDependencies:
  '@acme/pnpm-policy': 3.1.1
bootConfig:
  minimumReleaseAge: 0
include:
  '@acme/pnpm-policy': [minimumReleaseAge, trustPolicy]
```

- **Scope:** `bootConfig` governs resolving and verifying config dependencies and the package-manager dependency. Regular dependencies do not read it.
- **Allowed keys:** only settings governing this phase: `minimumReleaseAge`, `minimumReleaseAgeExclude`, `minimumReleaseAgeStrict`, `minimumReleaseAgeIgnoreMissingTime`, and `trustPolicy` and related settings if they are applied to this phase. Other keys are errors. In particular, the package-manager bootstrap continues to take registries only from non-project `.npmrc` files.
- **Fallback, per key:** use a non-null `bootConfig` value when present; otherwise use the same setting from configuration loaded before package entries, with normal precedence through repository values, inherited global values and built-in defaults. A null placeholder does not stop fallback. A package-owned setting whose value is not loaded yet contributes no boot value.
- **Repository files only:** the whole block belongs to one trusted local source or the root. A package cannot define it or receive a delegation for it. Its entries cannot be split between sources.
- **No new trust:** the block separates values for the boot phase; it does not let project configuration reach settings unavailable to that phase today.

Repositories without policy packages retain today's fallback: a local `minimumReleaseAge: 10080` applies to config dependencies and the pnpm version unless `bootConfig` supplies a different value. The resolver and verifier must use the same boot policy.

### Writes

Commands route each project configuration change by ownership:

| Command | Setting written |
| --- | --- |
| `pnpm approve-builds` | `allowBuilds` |
| `pnpm add` / `update` with catalogs | `catalog`, `catalogs.<name>` |
| `pnpm patch-commit` | `patchedDependencies` |
| `pnpm audit --fix`, `pnpm audit --ignore` | `overrides`, the audit ignore list |
| install with `minimumReleaseAgeExclude` / `trustPolicyExclude` prune | the exclude lists |
| `pnpm config set --location project` / `delete` | any ordinary setting |

- An explicitly delegated setting goes to its assigned local file even when omitted there. A package-owned setting is read-only.
- A setting defined in an unrestricted local file stays there. A setting defined in the root stays in the root.
- A setting with no owner is added to the root, as today. No unrestricted fragment is implicitly selected as a destination. If the root is protected, the write fails and the user must delegate it or make an authorized root edit.
- A new catalog goes to its explicit named owner or the owner of all `catalogs`, if either exists. Otherwise it is added to the root. An entry in an existing catalog goes to that catalog's owner.
- Removing the last map or list entry preserves an explicit empty `{}` or `[]`, retaining its value semantics and any ownership established by key presence. An explicit `config delete` removes the value, but leaves any root delegation intact; ownership based solely on a definition ends when that definition is deleted.

pnpm never moves a setting between files, deletes an included file or edits the include mapping as a side effect of writing a setting. Changing delegation requires an explicit root-manifest edit; `include` cannot be mutated through the ordinary setting writer. A failed write reports its destination and stops without falling back to another file.

### Inspecting the result

`pnpm config list` and `pnpm config get <key>` report project ownership separately from the value's source when they differ:

```text
$ pnpm config get minimumReleaseAge --verbose
0  (global config.yaml; project owner: ./config/security.yaml)

$ pnpm config get trustPolicy --verbose
no-downgrade  (@acme/pnpm-policy@3.1.0: config.yaml)
```

The first example applies when the key is delegated but omitted, or uses a null that means unset. An unavailable package source is marked incomplete rather than reported as an intentional omission.

### Local fragments

A gitignored local fragment must be explicitly included. It follows the same ownership rules and cannot override a committed owner's value. Every local include is required: a committed include of a gitignored file requires developers and CI to create that file, even if empty. Optional includes are outside this proposal. Existing global `config.yaml` remains available for machine-specific settings with its existing precedence.

### Interaction with the rest of pnpm

- **Lockfile:** settings recorded in `pnpm-lock.yaml` are computed from the merged result. Moving a value between sources without changing it does not change those recorded settings.
- **Cache keys and up-to-date checks:** workspace state, optimistic repeat installs and boot-policy checks must account for the root mapping, every included local file, and included packages' verified resolutions, config-entry metadata and contents. A missing required input cannot be treated as an unchanged configuration.
- **Workspace root:** `pnpm-workspace.yaml` remains required and is the sole root marker. An unreferenced directory does not load policy or mark a workspace root. Protecting the root index also protects against removing its includes or deleting the manifest.

## Rationale and Alternatives

### Automatic discovery in a fixed directory

An earlier version loaded every `.pnpm/config/*.yaml` file so an editable root could not remove a protected policy include. That allowed `pnpm-workspace.yaml` to remain broadly editable, but required protected files to claim neighbouring settings individually, including absent values. Tooling had to scan all fragments, and package ownership could remain unknown until packages were installed.

This proposal accepts a protected root index instead. Explicit lists limit less-trusted files, preserve ownership without placeholders and make package authority known in advance. There is no longer a reason to require a particular fragment directory.

### Explicit lists for every local file

This would make ownership entirely visible in the root, but adding each new trusted setting would require an accompanying mapping edit. Unrestricted local includes avoid that churn. They are a trust choice: a broadly editable file needs an explicit list, while a protected security file can use `'*'`.

Config dependencies require explicit lists because package updates must not silently expand their authority. This protection applies to declarative settings, not executable plugin hooks.

### Precedence instead of ownership errors

Ordered overrides make a contributor's setting silently ineffective or let an editable source replace protected values. A delegation says which source may supply a setting; a conflicting source is an error. No ordering rule is needed.

### The full `.pnpm/` project directory

Moving the lockfile, pnpmfile and workspace manifest into `.pnpm/` involves ecosystem tooling, CI cache keys and package-manager detection. Explicit includes work with any chosen local layout without requiring those moves.

### Generate `pnpm-workspace.yaml` from parts

The generated file must be protected, pnpm's writes need routing back to the original parts, and output can drift from its sources. Native includes supply that routing without a generated intermediate manifest.

### Placeholders and separate claim lists

Null placeholders preserve existing YAML semantics and can still appear in unrestricted files, but an explicit delegation reserves a key without a value. No custom `!inherit` or `!default` tag is needed. There is also no need to move keys between a separate `claimedSettings` list and configured values: an include's delegation stays present whether its source omits or defines the setting.

### Per-key permissions inside pnpm

A prompt or bypassable flag is not a file-permission boundary. pnpm validates delegation and routes writes; external review or process restrictions protect the root index and trusted files.

## Implementation

This is a new feature targeting the Rust CLI (`pnpm/crates/`). The TypeScript CLI's existing single-manifest behavior is unchanged; tooling and documentation must state which pnpm versions support includes.

- **`config` (`loading.rs`, `layout.rs`):** parse the root include index, resolve local sources and build canonical ownership before applying values. Validate exact setting lists, unrestricted local definitions, catalog selectors, duplicate sources and ownership conflicts. Only the root accepts `include`.
- **Value handling:** preserve existing overlays and setting-specific null semantics. Capture ownership before optional-field deserialization can collapse omitted keys and nulls; reserve explicitly delegated keys independently of source values.
- **Config dependency loading:** validate package delegations before installation, resolve included names only through verified config dependencies, load `package.json`'s `pnpm.config` entry inside the package, and validate its entire contents before applying values or loading hooks. Reject `'*'`, forbidden settings and entry paths escaping the package.
- **`workspace-manifest-writer`:** route writes through ownership, including omitted delegated settings and named catalogs. Enforce package read-only ownership even when its contents are unavailable. Never edit the root index implicitly or fall back after a failed write.
- **Config commands:** distinguish unavailable package values from absent values; allow unrelated local writes when local ownership is valid, and keep package-owned writes blocked.
- **`workspace-state`:** include the ownership index, included files and package inputs in invalidation checks.
- **Boot resolution and verification:** use the same `bootConfig` fallback for config dependencies and the package-manager dependency. Load no package-provided boot policy.

Fixtures cover local paths and package names; `.yaml` and `.yml`; unreferenced files being ignored; missing required inputs; duplicate sources and aliases; nested-include rejection; undelegated settings; empty delegated files and writes to omitted keys; unrestricted local additions without root edits; overlapping reservations; named and whole-catalog delegation and writer routing; package metadata and path validation; package updates adding undelegated settings; forbidden package delegations; and package-owned writes rejected before installation while unrelated local config writes succeed.

Value fixtures preserve v12 behavior before and after moving a setting: global `minimumReleaseAge: 0` survives project omission or null where appropriate, `hoistPattern: null` disables private hoisting, `scriptShell: null` clears an inherited shell, and `bootConfig` nulls fall through. Tests also distinguish an unavailable package from a successfully loaded package that omits a delegated value.

### Docs

- A page on splitting settings, with a protected-root example, explicit delegation for everyday files and unrestricted includes only for trusted local files.
- Config dependency examples showing `pnpm.config`, setting lists, bootstrap policy and the distinction between declarative authority and executable hooks.
- Example rules protecting the root, policy files and hook paths for `CODEOWNERS` and at least one process sandbox.
- Rename the env lockfile / env document to the **boot document** in documentation, retaining existing anchors. The lockfile format does not change because the first document has no `env` key.

Existing internal names such as `EnvLockfile`, `readEnvLockfile` and `env_lockfile` can be renamed separately; that mechanical change is not required to ratify this RFC. New user-facing messages use "boot document".

## Prior Art

- **Git** `[include]` / `[includeIf]`: explicitly referenced configuration files with ordered overrides.
- **Bazel** `.bazelrc` `import` / `try-import`: explicit required and optional imports. This proposal supports required includes only.
- **systemd, sudo, apt** `*.d/` directories: automatic discovery with location-based conventions. This proposal uses a protected explicit index instead.
- **`CODEOWNERS`:** the path-based review mechanism motivating delegation into separate files.

Here the index also assigns setting authority, and conflicting ownership is an error rather than an override.

## Unresolved Questions and Bikeshedding

- **Package entry field name:** `pnpm.config` is proposed as a single YAML entry. Its package-relative meaning must be clear alongside existing repository metadata in `package.json`.
- **Existing null semantics:** inventory setting-specific behaviors before implementation and preserve them with fixtures. Delegation must not change value interpretation.
- **Name of `bootConfig`:** "boot" covers loading prerequisites for the main command. Alternatives include `preflightConfig` and `bootstrapConfig`; the block must remain separate from settings a policy package owns.
- **Path-valued settings from packages:** shared patches could be useful, but need an explicit path-resolution and trust design. They remain forbidden here.
- **Optional local includes:** these could ease gitignored machine-local configuration, but silently missing policy must not be allowed by default.
- **Selecting a destination for a new setting:** a future explicit `--file` option could help with unowned keys. Ordinary writes in this proposal use existing ownership or the root and never create delegation implicitly.
