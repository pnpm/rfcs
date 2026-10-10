# The `tool:` protocol

## Summary

pnpm provisions a growing set of executables that are not ordinary npm dependencies — the Node.js, Bun, and Deno runtimes, and Yarn 6 — and names all of them with the `runtime:` protocol (`node@runtime:24.6.0`, `yarn@runtime:6.0.0`). The protocol describes a role, and the role is wrong for Yarn and only half right for Bun. This RFC replaces it with a role-neutral `tool:` protocol: `<name>@tool:<version>` names a tool pnpm provisions through that tool's own **tool definition**, which maps the tool's versions to where they come from and how they are verified. The role a tool plays — runtime, package manager, build tool — comes from the edge that references it, as it does for every npm package. `runtime:` stays accepted in manifests and on the command line; lockfiles switch to `tool:` keys in the next major.

## Motivation

### `runtime:` names a role, and some tools play another

`runtime:` was introduced for runtimes: `"node": "runtime:22"` in `dependencies` means "this package runs on Node 22", and that is how `engines.runtime` and `devEngines.runtime` are recorded. When pnpm v12 learned to install Yarn 6, which ships as native binaries in `yarnpkg/zpm` GitHub releases rather than as an npm package, the resolver reused the same protocol, because it was the only way pnpm had to say "a platform archive rather than a package". Its own module documentation says what is wrong with that: *Yarn is a package manager, not a runtime.*

Nothing misfires today — the code that treats `runtime:` dependencies as runtimes checks the name against `node`, `bun`, and `deno` — but the specifier says something false to anyone reading it, and the mismatch is about to spread:

- **Bun is both.** One Bun binary is a runtime and a package manager. A protocol that encodes the role needs two names for one artifact; `pnpm add bun@runtime:1.3.0` already has to be spelled that way to disambiguate from `pnpm add bun`, which declares Bun as the project's package manager.
- **Committed lockfiles would carry the misnomer.** `yarn@runtime:` appears today only in the per-package-manager environment lockfiles under the pnpm home, which are machine-local and re-resolvable. The [locked build environments RFC](https://github.com/pnpm/rfcs/pull/21) records the package manager that prepares a git-hosted dependency in the project's lockfile, which would put `yarn@runtime:6.0.0` into committed files. Renaming after that is a lockfile migration; renaming before it is nearly free.

### The role belongs on the edge

npm package keys never say what a package is used for. `typescript@5.9.2` is the same key whether a project lists it in `dependencies` or `devDependencies`; the field holding the edge says what it is for. Provisioned tools should work the same way: `devEngines.runtime`, `engines.runtime`, `devEngines.packageManager`, and the build-tool edges of the locked build environments RFC each say what their tool is for, and the tool's key only has to say which artifact it is.

### Four resolvers, one shape

Each provisioned tool has its own resolver crate today:

| Tool | Version list | Artifacts | Integrity |
|---|---|---|---|
| Node.js | nodejs.org release index, with channels (`release`, `nightly`, `rc`, …), `lts`, and LTS codenames | one archive per `(os, cpu, libc)`, enumerated from `SHASUMS256.txt` | `SHASUMS256.txt`, signature-verified |
| Bun | the `bun` npm package's versions | one zip per `(os, cpu)`, optional `-musl` | the release's `SHASUMS256.txt` |
| Deno | the `deno` npm package's versions | one zip per `(os, cpu)` | per-asset `.sha256sum` files |
| Yarn 6 | the `yarnpkg/zpm` GitHub releases API | one zip per Rust target triple | digests reported by the releases API |

They differ in where each answer comes from, not in what the questions are: list the versions, select one, map each platform to an artifact, and find each artifact's integrity. For these four, which all ship as platform archives, the result is the same lockfile shape: a `variations` resolution with one integrity-pinned variant per platform. That is the shape tools like aqua and proto give a *tool definition*, and it is the shape this RFC names.

Meanwhile, Yarn's other lines are npm packages under different names — Yarn Classic is `yarn`, Yarn 2–5 is `@yarnpkg/cli-dist` — so "Yarn at version X" is today three different specifiers depending on X. A tool definition can own that mapping too.

## Detailed Explanation

### 1. The protocol

`<name>@tool:<spec>` asks for the tool `<name>`, provisioned by its tool definition, at `<spec>`. Lockfile keys use the resolved version:

```yaml
packages:

  node@tool:24.6.0:
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

  yarn@tool:6.0.0:
    resolution:
      type: variations
      variants: …

  yarn@tool:4.9.2:
    resolution:
      tarball: https://registry.npmjs.org/@yarnpkg/cli-dist/-/cli-dist-4.9.2.tgz
      integrity: sha512-…
```

- **The name is the tool's name**, not a package name. `yarn@tool:4.9.2` is Yarn 4.9.2, wherever Yarn 4.9.2 comes from.
- **The resolution records where it came from.** A tool distributed as platform archives gets a `variations` resolution; a tool line distributed as an npm package gets a tarball resolution carrying the tarball URL and integrity, since the URL can no longer be derived from the key. Both shapes exist in the lockfile today.
- **The key says nothing about the role.** Bun used as a runtime and Bun used as a package manager are one `bun@tool:1.3.0` entry, referenced from two kinds of edges.
- **`tool:` doesn't replace npm packages.** `pnpm add yarn@npm:yarn@1.22.22` still installs the npm package named `yarn` as an ordinary dependency, and npm packages that happen to share a tool's name are unaffected. `tool:` is the explicit request for the provisioned tool.

### 2. Tool definitions

Every name `tool:` accepts has a tool definition built into pnpm. A definition answers four questions, and may answer them differently for different version ranges:

1. **Versions.** Where the list of released versions comes from, which aliases and channels a spec may use (`lts`, `nightly`, LTS codenames for Node), and, for every version, its **publish time** and its **trust evidence** (below).
2. **Selection.** How a spec picks a version.
3. **Artifacts.** Which artifact each platform gets — including libc variants, archive formats, and fallbacks such as Yarn 6's static musl build serving glibc hosts.
4. **Integrity.** Where each artifact's integrity comes from. An archive is pinned by an integrity recorded in the lockfile; a tool line published to npm is pinned by its tarball integrity, like any npm package.

#### Release age and trust apply to every tool

Today the policies that guard npm resolution reach provisioned tools unevenly. Bun and Deno select versions through their npm packages, so `minimumReleaseAge` and `trustPolicy` apply to the version list. The Node.js and Yarn 6 resolvers report no publish time, so `minimumReleaseAge` doesn't apply to them at all, and no tool outside npm has trust evidence that `trustPolicy` could compare. Making every definition answer the same questions closes that gap:

- **Publish time is required.** Every definition reports when each version was published — the Node.js release index carries a date per release, and the GitHub releases API carries `published_at` — so `minimumReleaseAge` and `minimumReleaseAgeExclude` apply to every tool exactly as they do to npm packages.
- **Trust evidence is declared per version.** A definition reports which evidence a version has, ordered from weakest to strongest: none; a checksum file served from the same release as the archive; a checksum file signed by the project's release keys (Node.js `SHASUMS256.txt`); a verifiable build attestation, such as a GitHub artifact attestation. A version distributed through npm reports npm's own levels (provenance, trusted publisher).
- **`trustPolicy: no-downgrade` compares those levels** the way it compares npm trust levels: by publish date, ignoring prereleases for a stable install. A version with weaker evidence than an earlier release of the same tool fails resolution, unless `trustPolicyExclude` names it (`node@tool:24.6.0`, `yarn@tool`). Evidence that cannot be verified counts as missing.
- **A definition may require a minimum level.** Node.js release-channel versions require a valid signature on `SHASUMS256.txt`, as the resolver already does; a version without it fails resolution regardless of `trustPolicy`, since an unsigned release there is evidence of tampering rather than a policy question.

**Trust and integrity are different checks.** Trust evidence is evaluated once, at resolution, and says who published a version. Integrity is recorded in the lockfile and says that later downloads are the bytes resolution saw. A checksum served from the same place as the archive — Bun's and Deno's GitHub releases, the digests the GitHub releases API reports for Yarn 6 — establishes integrity for every later install, but says nothing about the publisher; that is why it ranks just above no evidence.

The initial set, all of which pnpm already resolves today:

| Tool | Ranges and sources |
|---|---|
| `node` | nodejs.org release index and mirrors |
| `bun` | `bun` npm versions; GitHub release archives |
| `deno` | `deno` npm versions; GitHub release archives |
| `yarn` | `<2`: npm package `yarn`; `2`–`5`: npm package `@yarnpkg/cli-dist`; `>=6`: `yarnpkg/zpm` release archives |

`yarn` is the case that motivates per-range mappings: one tool, three distribution channels, and today three spellings. With a definition, `yarn@tool:4` and `yarn@tool:6` are the same request at different versions.

**Definitions are code first, data later.** The first step moves the four existing resolvers behind one tool-definition interface without changing what they resolve; that interface is what the protocol needs. A declarative format, in the spirit of aqua's registry, is the natural second step for tools whose answers are URL templates and checksum files — Bun and Deno would fit it today — with the interface remaining available to tools that need logic, such as Node's channels and mirrors. That step changes no lockfile output and can land separately.

**Definitions are built in.** A definition decides which URL is downloaded and which checksum source is trusted at resolution time; after that, the lockfile pins integrity. Loading definitions from users or third parties would make that first resolution trust whoever wrote them, so this RFC ships built-in definitions only, reviewed like resolver code is today. Opening definitions up turns pnpm into a general toolchain manager alongside proto, aqua, and mise; that is a product decision for its own RFC, not a detail of this one.

### 3. Where the protocol appears

| Place | Today | With this RFC |
|---|---|---|
| Lockfile package keys | `node@runtime:24.6.0`, `yarn@runtime:6.0.0` | `node@tool:24.6.0`, `yarn@tool:6.0.0` |
| Lockfile importer specifier and version | `specifier: runtime:24.6.0` | `specifier: tool:24.6.0` |
| Environment lockfiles under the pnpm home | `yarn@runtime:<version>` | `yarn@tool:<version>` |
| Manifest dependency specifiers | `"node": "runtime:22"` | `"node": "tool:22"`; `runtime:` still accepted |
| CLI | `pnx node@runtime:22`, `pnpm add bun@runtime:1.3.0` | `pnx node@tool:22`, `pnpm add bun@tool:1.3.0`; `runtime:` still accepted |

What does **not** change:

- `engines.runtime`, `devEngines.runtime`, and `devEngines.packageManager`. They are role fields, and correctly named: they say what the tool is for. Only the protocol inside the recorded specifier changes.
- `pnpm runtime set`. It is a command about the runtime role, and keeps its name.
- The bare forms that already map a name to its tool: `pnx yarn@4`, `pnx node@22`, `pnpm add -g node@22`. They resolve through the same definitions.

### 4. Compatibility

- **`runtime:` is accepted indefinitely** wherever a user writes a specifier — manifests and the command line — as an alias for `tool:`. For `node`, `bun`, and `deno` it is the spelling existing projects and documentation use, and nothing is gained by breaking it.
- **Lockfiles switch to `tool:` keys in the next major, with a `lockfileVersion` major bump.** An older pnpm cannot resolve a `tool:` key, so it must refuse the lockfile rather than misread it. The new pnpm reads `@runtime:` keys and rewrites them as `@tool:` on the next install; under `--frozen-lockfile`, a lockfile with `@runtime:` keys is accepted as-is and rewritten by the next regular install, so upgrading does not break CI.
- **Environment lockfiles under the pnpm home** are rewritten on next use. They are machine-local, so nothing else depends on their keys.
- **Yarn's npm lines.** A project that pinned Yarn Berry through the bare npm alias in its manifest (`yarn@npm:@yarnpkg/cli-dist@4.9.2`) keeps an ordinary npm dependency. Only pnpm's own records of a provisioned Yarn move to `yarn@tool:`.

### 5. Relation to the locked build environments RFC

That RFC records the tools that prepare a git-hosted dependency in the project's lockfile. With this protocol, its record is uniform across tools and lines:

```yaml
buildDependencies:
  node: tool:24.6.0
  yarn: tool:4.9.2
```

This RFC is a prerequisite of that one: both are major changes, and landing them together moves lockfile keys once.

## Rationale and Alternatives

1. **Keep `runtime:` and redefine it as "a provisioned tool".** No migration at all, and the code already treats it that way. But the specifier keeps saying something false, and the cost of fixing it grows the moment build tools are recorded in committed lockfiles.

2. **A `pm:` protocol for package managers, `runtime:` for runtimes.** Accurate for Yarn, but it puts the role back into the key. Bun would need `bun@pm:1.3.0` and `bun@runtime:1.3.0` for the same binary — two lockfile entries, two store lookups — and every future dual-role tool repeats the problem.

3. **Protocols per source** (`github-release:`, `nodejs-dist:`, …). Accurate about where an artifact comes from, but that is already recorded in the resolution, and it makes a tool's source part of its identity: Yarn moving from npm to GitHub releases at version 6 would change its protocol. The user asks for a tool at a version; where it comes from is the definition's business.

4. **Open tool definitions now** (user-supplied, like aqua's registry or proto's plugins). More useful in the long run, but it widens the trust boundary from "pnpm's reviewed code" to "whoever wrote the definition" at exactly the step that decides what gets downloaded and executed. Built-in definitions solve the problem at hand without that decision. Script-based plugins in the asdf style, which run arbitrary code at resolution time, are rejected outright.

5. **Route tools through pnpmfile custom resolvers.** pnpm already lets a pnpmfile resolve specifiers. That mechanism is per-project and runs project code; provisioned tools need to resolve identically in every project, before any project code is trusted.

## Implementation

pnpm v12 only, per the version policy.

- A `tool-resolver` crate with the tool-definition interface; `engine-runtime-node-resolver`, `engine-runtime-bun-resolver`, `engine-runtime-deno-resolver`, and `engine-pm-yarn-resolver` become its definitions, and `yarn` gains the npm-line mappings now spread across `package-manager` and `cli/dlx`.
- `resolving-default-resolver` claims `tool:` and `runtime:` specifiers through that crate.
- The definition interface carries a publish time and a trust-evidence level per version, wired into the existing `minimumReleaseAge` and `trustPolicy` checks. The Node.js and Yarn 6 definitions start reporting publish times, from the release index's dates and the releases API's `published_at`.
- `lockfile` — `tool:` keys, `@runtime:` keys accepted on read, the `lockfileVersion` major bump.
- `deps-path` (`try_get_package_id`) — treats `tool:` as it treats `runtime:` today.
- `package-manifest` (`runtime.rs`) — `engines.runtime` / `devEngines.runtime` round-trip writes `tool:` and reads both.
- `cli` — `dlx` provisioning, global shims, `deploy`'s lockfile handling, and `pnpm add`'s runtime and package-manager paths accept both spellings and write `tool:`.
- `package-manager` (`catalog_mode.rs`) — the catalog exemption for `runtime:` specifiers covers `tool:`.

## Prior Art

- **aqua** keeps a registry of declarative package definitions — version source, per-platform asset templates, checksum files, and optional cosign or SLSA verification — and pins checksums in a lockfile. The model for the second step of this RFC.
- **proto** defines each tool as a plugin: TOML for tools whose downloads follow a template, WASM for tools that need logic. The same split between data and code this RFC proposes for definitions.
- **asdf** plugins are shell scripts, run with the user's privileges at install time. The flexibility is real, and so is the trust it requires; it is the model this RFC deliberately does not follow.
- **mise** resolves tools through several backends (core, aqua, ubi, npm, …), keeping the tool's name separate from where it comes from.
- **Volta** supports a fixed set of tools — Node, npm, Yarn, pnpm — with built-in knowledge of each, the closest to this RFC's built-in-only first step.

## Unresolved Questions and Bikeshedding

- **The name.** `tool:` is short and role-neutral. Alternatives: `bin:`, `exe:`, `dist:`.
- **pnpm itself.** pnpm's own pin is recorded under `packageManagerDependencies` as `@pnpm/exe` and its platform packages. Whether `pnpm@tool:<version>` should become the way it is named, and whether npm and Yarn Classic/Berry are ever recorded as plain npm packages rather than through their tool definitions.
- **Tools as installed dependencies.** `node@runtime:` can be a project dependency today, linking `node` into `node_modules/.bin`. Whether `yarn@tool:` and other non-runtime tools may be installed the same way, or remain provisioned-only.
- **The declarative format.** Its schema, and whether it is YAML, TOML, or Rust data, is left to the second step.
