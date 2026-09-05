# Cargo ecosystem: pnpm for Rust crates, pnpr as a crate registry

> **Status: proposed.**

## Summary

pnpm learns a second ecosystem. A workspace that opts in has its Rust crate dependencies resolved, fetched, verified and materialized by pnpm, from the same content-addressable store and with the same registry, auth, audit, licensing, filtering and task machinery the npm side already has. Cargo keeps its job: it compiles. It never touches the network in a pnpm-managed workspace, because pnpm points every registry and git source at a per-workspace directory of links into crates unpacked once, machine-wide, in the store — the global virtual store, applied to crates — and `Cargo.lock` stays exactly the file cargo would have written. pnpr grows a `cargo` registry protocol alongside npm: the sparse index, the download endpoint, the publish/yank/owners/search web API, and an upstream mode that proxies crates.io — so the same server, the same `registries:` model, and the same access rules serve both ecosystems.

The immediate reason is that pnpm no longer dogfoods itself. The CLI is Rust now; its more than seven hundred crates.io dependencies are fetched by cargo, cached by a GitHub Action, audited by cargo-deny, and never pass through pnpm or pnpr. This RFC puts them back on the path we ship.

## Motivation

**We stopped eating our own food the day the rewrite landed.** Every `pnpm install` in this repository installs the TypeScript workspaces under `pnpm11/` — the 11.x line — and the tooling around them. The product we actually release is built by `cargo build --locked --release --bin pnpm`, and nothing about that build exercises pnpm: not the resolver, not the store, not the fetcher, not the registry client, not the lockfile, not pnpr. The Rust CI caches `~/.cargo/registry` with `Swatinem/rust-cache`; the release workflow builds every target with cargo alone; `deny.toml` carries the advisory and license policy that `pnpm audit` and `pnpm licenses` exist to enforce. The largest, most actively developed Rust workspace we own — 467 commits under `pnpm/` since August against 237 under `pnpm11/` — is the one workspace whose dependency management we have no stake in.

Dogfooding is not a vanity metric here. The bugs pnpm finds fastest are the ones its own developers hit at their own desks: a resolver that hangs on a pathological graph, a store that corrupts under a crash, a registry that serves a stale index. On the npm side those bugs surface in this repository before they surface in issues. On the Rust side they cannot, because we are cargo's users, not pnpm's.

**Rust projects have the problems pnpm was built to solve.** A `~/.cargo/registry/src` with several toolchains' worth of unpacked crates is the same disk-space story as `node_modules` before the content-addressable store; a monorepo with a Cargo workspace and an npm workspace side by side — every napi-rs package, every Tauri app, this repository — runs two package managers with two caches, two lockfiles, two audit tools, two ways of saying "these projects depend on those", and two CI cache actions. Cargo has no equivalent of `pnpm --filter ...[origin/main]`, of `dependsOn` task scheduling across a mixed workspace, of catalogs beyond `[workspace.dependencies]`, of named registries in the lockfile, of a minimum release age, of `pnpm audit --fix` re-picking versions, or of a registry that speaks two ecosystems' protocols under one access policy. Every one of these is code pnpm already has and cargo will not grow.

**pnpr already wants to be the registry a company runs, and companies do not run one language.** The registry-mounts model — declared namespaces, no cross-origin fall-through, hosted and upstream registries composed by routers — is protocol-agnostic in its rules and npm-shaped in its URLs. The teams that would deploy pnpr for npm are the teams that today deploy Artifactory or Cloudsmith precisely because one server fronts npm *and* crates.io *and* PyPI. Cargo's registry protocol is small, fully specified, and stable; it is the cheapest second protocol pnpr could possibly add, and it is the one our own CI needs.

### What the outcome looks like

In this repository, after this RFC:

```console
$ pnpm install --frozen-lockfile
Crates: 714 resolved from Cargo.lock, 714 linked into .pnpm/crates from the store
Packages: already up to date
$ cargo build --locked --release --bin pnpm      # no network, same Cargo.lock
```

`pnpm audit` covers `Cargo.lock` through OSV's `crates.io` ecosystem. `pnpm licenses list` covers both manifests. `pnpm --filter ...[origin/main] exec cargo nextest run` selects the crates a branch touched. Dependabot's `chore(cargo): bump` PRs become `pnpm update`. The CI cache is the pnpm store. And the index those crates come from is a pnpr registry proxying crates.io, so every install exercises pnpr's upstream path, its cache, and its circuit breaker.

## Detailed Explanation

### Terminology

- **Cargo ecosystem** — the set of behaviours this RFC adds. A workspace is *cargo-managed* when it has opted in (below).
- **crate registry** — a registry speaking Cargo's sparse index protocol: a `config.json`, index files, a download endpoint and, optionally, the web API. crates.io is one; a pnpr registry declared with `protocol: cargo` is another.
- **crate project** — a workspace project whose manifest is a `Cargo.toml` with a `[package]` table. A directory holding both `package.json` and `Cargo.toml` is one project with two manifests.
- **crate slot** — one crate, unpacked once per machine under the store, with its `.cargo-checksum.json`. The crate-side counterpart of a global virtual store slot.
- **crate source directory** — the per-workspace directory pnpm maintains for one replaced cargo source, in the layout cargo's `directory` source reads: one entry per locked crate, each a link to that crate's slot.
- **source key** — cargo's identity for where a package comes from, as written in `Cargo.lock`: `registry+https://github.com/rust-lang/crates.io-index`, `sparse+https://…`, or `git+https://…#…`.

> **pnpm owns resolution, fetching, verification and materialization; cargo owns compilation. Cargo never contacts a registry in a cargo-managed workspace, and `Cargo.lock` is byte-for-byte the file cargo would write for the same graph.**

Everything below follows from that sentence. If a design choice would have cargo fetch anything, or would give `Cargo.lock` a shape cargo would rewrite, it is the wrong choice.

### Division of labour

Cargo already has the seam this needs: **source replacement**. A `[source]` table in `.cargo/config.toml` may redirect a registry or git source to a `directory` source — a folder of unpacked crates each carrying a `.cargo-checksum.json` — and cargo then builds without the network. `cargo vendor` uses this; so do Nix and Bazel. Two properties of the seam matter:

- Cargo's stated core assumption is that "the source code is exactly the same from both sources" and that "a replacement source is not allowed to have crates which are not present in the original source". A crate source directory that contains exactly the packages `Cargo.lock` records for that source satisfies this by construction.
- `Cargo.lock` records the *original* source key, not the replacement. Vendoring crates.io does not put a local path in the lockfile. The lockfile therefore stays portable and identical to the one an unmanaged cargo would write, which is what lets the two worlds coexist in one repository.

pnpm's side of the seam is machinery it already has. The store keeps unpacked files by sha512 in a content-addressable layout; `import_indexed_dir` in `deps-restorer` materializes any directory from any in-package-path → CAS-path map and is already used outside `node_modules` for `.pnpm-config/`; the global virtual store keeps one materialized copy per machine under `<store>/links` with a projects registry for pruning. A crate slot is the first primitive filling a directory under the store, and a crate source directory is the second primitive's symlink farm pointed at those slots.

**What pnpm does not do: build.** No feature resolution for a particular target, no `cfg()` evaluation, no rustc invocation, no `target/`. `cargo build` is a task like `tsc` is a task; it runs under `pnpm run`/`exec` or directly, and it reads what pnpm materialized.

### Opting in

A workspace opts in through `pnpm-workspace.yaml`:

```yaml
cargo:
  enabled: true
  # Where crates.io's index is read from. Cargo.lock keeps recording
  # crates.io as the source, so this is deployment configuration, not identity.
  cratesIoIndex: https://pnpr.example.com/~crates-io/   # default: https://index.crates.io/
```

Opt-in is deliberate rather than inferred from the presence of a `Cargo.toml`. Every napi-rs package and every Tauri app has a `Cargo.toml` next to a `package.json`; their authors did not ask pnpm to start downloading crates, rewriting `.cargo/config.toml`, and failing installs on a resolver difference. The presence of a manifest is a fact about the repository; management is a decision, and the file where pnpm records decisions is the workspace manifest.

**Cargo's own configuration is honoured, not duplicated.** Alternative registries are declared where cargo already declares them — `[registries.<name>] index = "sparse+https://…"` in the `.cargo/config.toml` hierarchy — and credentials come from where cargo already keeps them: `$CARGO_HOME/credentials.toml` and `CARGO_REGISTRIES_<NAME>_TOKEN`. pnpm reads both. A workspace that publishes to a private registry should not have to say so twice, and `cargo publish` needs the cargo copy anyway. The single pnpm-side URL setting exists only because crates.io's identity is a fixed string cargo cannot re-point without source replacement, and source replacement is the mechanism this RFC spends on the directory.

### Projects, workspaces and identity

A directory containing a `Cargo.toml` with a `[package]` table is a project. `PROJECT_MANIFEST_BASENAMES` in `workspace/src/projects.rs` grows from `["package.json", "package.yaml"]` to include `Cargo.toml`; `Project` gains an optional crate manifest beside its optional npm manifest, and a project may have either or both. Discovery is the union of the `packages:` globs and the root `Cargo.toml`'s `[workspace].members` minus `exclude`, so an existing Cargo workspace needs no glob edits.

**Identity is the directory, and names are aliases.** This is the model the monorepo-versioning RFC settled and it is the right one for a mixed workspace: `pnpm/npm/pnpm` and `pnpm/crates/cli` are two projects that publish under two ecosystems, and a `--filter` by name resolves against the union of npm names and crate names. An alias matching more than one project is a validation error listing the candidates, as that RFC already specifies; nothing here changes the rule, it only widens the alias space. Filter selectors, `...[ref]` git-diff selection, `-r`, topological ordering, and the task scheduler all operate on project directories and gain crate projects without modification. Project edges for crate projects come from path dependencies and from `workspace = true` inheritance, resolved against sibling crates' `[package].name`/`version` the way npm edges are resolved against `package.json`.

A crate project has no `scripts`. `pnpm run build` in one is "no such script", as today for a `package.json` without it, and the pass-through rule from the task orchestration RFC applies. `pnpm exec` and `pnpm -r exec` work unchanged, which is how `cargo nextest run` gets a filter. Whether crate projects should carry implicit tasks (`build`, `test`, `clippy`) is a follow-up; see the open questions.

### Resolution

The npm resolver cannot be reused for this, and it is worth being precise about why, because the difference is the largest single piece of new code.

npm resolution is per-dependency: given one wanted dependency, pick one version, and let two dependents that disagree each get their own copy nested underneath them. Cargo forbids that. Within one semver-compatible range — one major for `>= 1.0`, one minor for `0.x` — the whole graph gets exactly one version, and a conflict is resolved by backtracking to an older version that satisfies both dependents, not by duplicating. Add feature unification (a feature enabled by one dependent turns on optional dependencies for all of them), weak dependency features (`serde?/derive`), `links` uniqueness, MSRV-aware selection under `resolver = "3"`, and yanked versions that are illegal to pick fresh but legal to keep from a lockfile, and what is needed is a whole-graph backtracking solver with cargo's exact rules. That is a new crate, `pnpm-cargo-resolver`, and it does not implement the `Resolver` trait from `resolving-resolver-base`, because that trait's contract — one wanted dependency in, one resolution out, `None` to defer — is the npm model.

**The solver is PubGrub over the sparse index.** The `pubgrub` crate is a maintained Rust implementation of the algorithm, the cargo team has been testing a PubGrub-based resolver against cargo's for compatibility, and its model — versions as a totally ordered set per package, incompatibilities derived from requirements — is what cargo's rules reduce to once features are folded into the package set. Version ordering and requirement matching use the `semver` crate, which is cargo's own implementation of cargo's flavour of semver, so `^`, `~`, `=`, `*`, comparators and pre-release matching cannot drift.

**What `Cargo.lock` locks is the maximal graph.** Cargo generates its lockfile with every workspace member's every feature enabled, every target's dependencies included, and dev-dependencies included. The lockfile is therefore platform- and feature-independent, and pnpm resolves the same superset: it does not need to know which target the user will build for, only which optional dependencies *any* feature activation reaches. The resolver-version setting (`resolver = "1"/"2"/"3"`) changes how cargo unifies features at build time and, for `3`, which versions are eligible; it is read for eligibility and otherwise left to cargo.

The index is the input. pnpm fetches `config.json` once per registry and then one file per crate name from the sparse index — `1/a`, `2/ab`, `3/a/abc`, `ab/cd/abcd` — with `ETag`/`If-None-Match` conditional requests against a mirror under `<cache_dir>/v11/cargo-index/<registry>/`, the same cache discipline `fetch_full_metadata_cached` uses for packuments. Index lines are parsed with `cargo-util-schemas`' index types, which are cargo's own published schema crate, so `features2`, `v`, `rust_version`, `links`, `package` renames and dependency `kind`s are read the way cargo reads them. The `pubtime` field crates.io added to its index lines is what `minimumReleaseAge` and time-based resolution consume; a registry that omits it is treated as it is on the npm side when a packument has no `time`.

Existing lockfile versions are preferred, as `preferred_versions` already does for npm: a resolve with an existing `Cargo.lock` reproduces it unless a manifest requirement it no longer satisfies forces a change, and `pnpm update` widens the preference. `[patch]` sections are honoured as cargo honours them — a patched source replaces the index's candidates for that name. `[replace]` is deprecated in cargo and is an error here.

**Correctness is tested against cargo, not argued.** Cargo's resolver is the oracle. The test corpus is, first, this repository's own workspace — 811 lockfile entries, proc-macros, `links` crates, target-specific and optional dependencies, features with `dep:` syntax — where pnpm's resolve against the checked-in `Cargo.lock` must reproduce it exactly; then a set of public workspaces chosen for resolver edge cases; then a differential fuzz that generates manifests and compares `pnpm`'s lock to `cargo generate-lockfile`'s. A difference is a bug in pnpm, by definition, until cargo's own documentation says otherwise.

### The lockfile is `Cargo.lock`

There is one lockfile for crates, and cargo already defines it. pnpm reads and writes `Cargo.lock` (format version 4) and records nothing about crates in `pnpm-lock.yaml`.

**The lock's content must be what cargo would accept under `--locked`, and its bytes should be what cargo would write.** The two are different bars. Cargo with `--locked` fails only when the *resolve* recorded in the file needs to change — a missing checksum, a version that no longer satisfies a requirement — and silently rewrites formatting differences; a lock with two packages swapped or trailing blank lines passes `--locked` and is quietly normalized on disk. pnpm targets the stronger bar anyway: a `cargo` invocation that rewrites a committed file in CI is a dirty checkout, and the format is small and deterministic — packages sorted by name then version then source, the `# This file is automatically @generated by Cargo.` header, `dependencies` entries qualified with a version only when the name is ambiguous. The `cargo-lock` crate (RustSec's parser and serializer) is the starting point; a golden test round-trips this repository's lockfile.

`--frozen-lockfile` means for crates what it means for npm and what `--locked` means for cargo: resolve with the lock as preference, and fail with `ERR_PNPM_OUTDATED_LOCKFILE` naming the requirement that would change it rather than write.

Because the lock records source keys rather than URLs, mapping a locked package back to an index is configuration: `registry+https://github.com/rust-lang/crates.io-index` → `cargo.cratesIoIndex`; `sparse+<url>` → that URL, with credentials looked up by matching it against `[registries]`; `git+<url>#<rev>` → pnpm's git fetcher. A source key with no way to reach it fails closed, the way a named registry without a URL mapping does on the npm side.

### Fetching and the store

A locked package from a registry is fetched from the registry's `dl` template — `{crate}/{version}/download` appended when the template has no markers, otherwise the `{crate}`, `{version}`, `{prefix}`, `{lowerprefix}` and `{sha256-checksum}` substitutions — and verified against the `cksum` from the index, which is also the `checksum` in `Cargo.lock`. A `.crate` is a gzipped tarball with a `<name>-<version>/` prefix; pnpm's tarball fetcher handles it as it handles a `.tgz`, stripping one leading component.

**Unpacked files go into the same store, keyed by sha512 as today.** Nothing about the CAS layout or the store version changes; a crate's files are files. The immutable file blobs are shared across ecosystems, but the store index row is projection-qualified: a raw Cargo archive uses a separate namespace from an npm package projection, while ordinary npm packages retain their existing key byte-for-byte. The crate's sha256 integrity is still the verified archive identity — expressed as an `sha256-…` SRI, which `ssri` accepts and the row's per-row `algo` field records — but integrity alone does not identify the projected file set.

Cargo's `directory` source requires each crate directory to carry a `.cargo-checksum.json`: a map of every file path to its sha256, plus the sha256 of the `.crate` under `package`. The store records sha512 per file, not sha256, and widening the index row would touch a format shared byte-for-byte with the TypeScript CLI's store. So the checksum file is not derived at link time — it is **synthesized once at unpack and stored as an ordinary file of the package**. It is computed while the bytes are streaming through the hasher anyway, it is content-addressed like everything else, and it links out with the rest of the files. Git-sourced crates get `"package": null`, as `cargo vendor` writes for them.

Git dependencies are fetched by the existing git fetcher and resolver at the locked revision, and materialized the same way. Path dependencies and workspace members are cargo's own business and are not materialized at all.

### Materialization and `.cargo/config.toml`

Crates are unpacked once per machine, and a workspace holds links:

```
<store>/v11/crates/                          # machine-wide crate slots
  serde-1.0.229-4148590afebada386688f18773da617792bf2ef03ffc1e4cbd2b1d45b023e0ba/
    .cargo-checksum.json
    Cargo.toml
    src/…
  serde_derive-1.0.229-e7a5d71263a5a7d47b41f6b3f06ba276f10cc18b0931f1799f710578e2309348/

<workspace>/.pnpm/crates/                    # per workspace: one link per locked crate
  crates-io/                                 # one directory per replaced source key
    serde-1.0.229        -> <store>/v11/crates/serde-1.0.229-4148590afebada386688f18773da617792bf2ef03ffc1e4cbd2b1d45b023e0ba
    serde_derive-1.0.229 -> <store>/v11/crates/serde_derive-1.0.229-e7a5d71263a5a7d47b41f6b3f06ba276f10cc18b0931f1799f710578e2309348
  registry-pnpr.example.com-9f1c2a/          # sparse+https://pnpr.example.com/~internal/
    acme-telemetry-0.4.1 -> …
  git-bar-3f2a8c1/                           # git+https://github.com/foo/bar#3f2a8c1…
    bar-0.9.0            -> …
  .state.json                                # lock hash, layout version
```

**This is the global virtual store, not a per-workspace tree.** A crate slot is filled once, under the store, by hardlinking its files out of the CAS through `import_indexed_dir`; every workspace that locks that crate gets a symlink — a junction on Windows, as `node_modules` already does — and nothing else. An install of 714 crates is 714 symlinks, not tens of thousands of file links, and two workspaces on one machine share every byte and every path. The slot is keyed by name, version and the full crate sha256, so two registries serving different bytes under one `name@version` get two slots without making collision resistance depend on an arbitrary truncation length.

Cargo is indifferent to the indirection. Its directory source lists the source directory's entries, skips dot-prefixed ones (so `.state.json` is invisible to it), keeps every entry under which `Cargo.toml` *exists* — a check that follows symlinks — and reads the package through the link. It records the package's checksum from `.cargo-checksum.json` and verifies each listed file's sha256 when it compiles the crate, with an error that says directory sources are not meant to be edited. Two scratch workspaces sharing one central tree through symlink farms both build offline on cargo 1.97. Slot files retain the store's file modes: marking a hardlinked slot file read-only would chmod the same CAS inode and every npm import linked to it. Instead pnpm verifies a warm slot against its CAS map before reuse and `import_indexed_dir` repairs any mismatch; direct edits remain unsupported and Cargo's checksum is the second line of defence.

One source directory per replaced source, not one for all of them, because cargo's replacement rule is per source and a directory that mixed two registries' crates would let a name present in both be served for either. Directory names are for readability only — a source directory is named by the registry host or git repository plus a short hash of the full source key, a link by `<name>-<version>` — because cargo does not care what an entry is called.

Pruning happens at two levels, as it does for the npm global virtual store. The workspace's links are pruned to the locked set on every install; a stale link that cargo would otherwise still see is the one way to violate the "exactly the same as the original source" rule. Slots are pruned by `pnpm store prune`, which walks the store's projects registry — every cargo-managed workspace registers itself there, as global-virtual-store projects do — and removes slots no registered workspace's `Cargo.lock` references.

Identical absolute source paths across every workspace on a machine are also the precondition for anything that would later share compiled artifacts between workspaces: rustc embeds source paths in debug info and fingerprints, and a per-workspace tree would make every workspace's build unique by construction.

**pnpm owns a marked region of the workspace's `.cargo/config.toml`, and that file is committed.**

```toml
[alias]
codecov = "…"                              # the user's own settings are untouched

# >>> pnpm-managed source replacement — do not edit by hand >>>
[source.crates-io]
replace-with = "pnpm-crates-io"

[source.pnpm-crates-io]
directory = ".pnpm/crates/crates-io"

# sparse+https://pnpr.example.com/~internal/
[source.pnpm-upstream-9f1c2a]
registry = "sparse+https://pnpr.example.com/~internal/"
replace-with = "pnpm-registry-9f1c2a"

[source.pnpm-registry-9f1c2a]
directory = ".pnpm/crates/registry-pnpr.example.com-9f1c2a"

# git+https://github.com/foo/bar#3f2a8c1…
[source.pnpm-upstream-3f2a8c1]
git = "https://github.com/foo/bar"
rev = "3f2a8c1…"
replace-with = "pnpm-git-3f2a8c1"

[source.pnpm-git-3f2a8c1]
directory = ".pnpm/crates/git-bar-3f2a8c1"
# <<< pnpm-managed <<<
```

The region is regenerated on every install from the set of source keys in `Cargo.lock`, and only the region: the file may hold aliases, build settings, `[registries]`, anything. Committing it is the point. It is the crate-side equivalent of `packageManager` — the statement that this workspace is built through pnpm — and it makes the failure mode of skipping `pnpm install` a clear cargo error naming the missing directory rather than a silent fetch from a registry the workspace decided not to trust directly.

The alternative of injecting `--config` when pnpm spawns cargo was considered and rejected below; it works for `pnpm run` and for nothing else, and rust-analyzer alone is reason enough.

### Commands

The verbs are pnpm's verbs, and the crate ecosystem is selected by the `crate:` specifier protocol, alongside `npm:`, `jsr:`, `catalog:`, `workspace:` and `runtime:`. `crate` joins `RESERVED_VERSION_PREFIXES` so no named registry can shadow it.

```console
$ pnpm add crate:serde@^1 --features derive
$ pnpm add crate:tokio --features rt,macros --no-default-features
$ pnpm add -D crate:insta                     # [dev-dependencies]
$ pnpm add --save-build crate:cc               # [build-dependencies]
$ pnpm add crate:pnpr-auth@workspace:          # sibling crate, written as a path dependency
$ pnpm remove crate:serde
$ pnpm update crate:tokio                      # re-resolve one crate within its requirement
$ pnpm update --latest crate:tokio             # bump the requirement too
```

**`pnpm install`** resolves and materializes both ecosystems concurrently in one run. Their graphs are independent, so serializing them would add their wall times without establishing a useful ordering; a crate project with a `build.rs` that needs `node_modules` is not a case worth ordering installation around. The installers share one configured HTTP client and its throttles, so concurrency does not multiply `networkConcurrency` or `maxSockets`. The coordinator is ecosystem-neutral: a later PyPI installer joins the same set of concurrent tasks and receives the same network client rather than introducing another serial phase or an independent connection budget. `--frozen-lockfile`, `--prefer-offline`, `--offline`, `--lockfile-only`, `--ignore-scripts` mean what they mean; scripts do not exist on the crate side, so the last is a no-op there.

**Manifest edits are format-preserving** — `toml_edit`, as cargo itself uses, so `pnpm add` on a hand-formatted `Cargo.toml` changes one line. In a workspace whose root declares the crate under `[workspace.dependencies]`, `pnpm add crate:serde` in a member writes `serde.workspace = true` and leaves the requirement where it lives; `[workspace.dependencies]` **is** the crate catalog, and `catalog:` for a `crate:` specifier is spelled `workspace = true`. Adding to the catalog itself is `pnpm add --catalog crate:serde@^1` at the root. Named catalogs have no cargo equivalent and are not invented.

**Which ecosystem a bare name means** is decided by the project the command runs in: a project with only a `Cargo.toml` takes bare names as crates, one with only a `package.json` takes them as npm packages, and one with both requires the prefix and says so. The `-O`/`--save-optional` flag is npm's "may fail to install" and is rejected for `crate:` specifiers; cargo's optional — a dependency gated behind a feature — is `--optional`, and the two must not be confused by sharing a spelling.

**`pnpm outdated`**, **`pnpm list`**, **`pnpm why`** read `Cargo.lock` through the same `deps-inspection` graph library the npm commands use, with crate nodes tagged so the renderer can show `serde 1.0.229 (crate)`.

**`pnpm audit`** queries OSV's batch API with `{"package": {"name", "ecosystem": "crates.io"}, "version"}` for every locked registry crate; RustSec advisories are published to OSV, so this is cargo-audit's data. `--fix` uses the same `PackageVersionGuard` seam the npm side uses to veto vulnerable picks during a re-resolve.

**`pnpm licenses`** reads `license`/`license-file` from each materialized crate's `Cargo.toml` — the field is SPDX by convention — and reports both ecosystems in one table. `deny.toml`'s allow-list has a home in `pnpm-workspace.yaml` once this exists; cargo-deny's `bans` (duplicate versions, wildcard requirements) do not, and are a follow-up.

**`pnpm publish`** and **`pnpm pack`** for a crate project delegate the archive to `cargo package --no-verify --offline`. Producing a `.crate` means writing the normalized `Cargo.toml` cargo generates from the original plus `Cargo.toml.orig` and `.cargo_vcs_info.json`, and that normalization is cargo's, changes across cargo versions, and is not specified anywhere pnpm could conform to. pnpm then performs the upload itself — the `PUT /api/v1/crates/new` framing, the registry choice, the token — so pnpm's auth, pnpr's policies, `--dry-run`, and workspace-aware ordering apply. `pnpm unpublish` maps to yank; there is no crate unpublish.

**Not in this RFC:** `pnpm dlx crate:…` (it would mean compiling, which is a build cache question), toolchain management from `rust-toolchain.toml` (the `runtime:` machinery is the obvious home and it is a follow-up), and a `cargo build` task cache (the workspace task cache RFC's problem, once `cargo build` is a task).

### Architecture exposed by a second ecosystem

Cargo is the first case that forces a distinction which was mostly invisible while pnpm managed only npm packages: dependency management has an ecosystem-neutral lifecycle, but its semantics are not ecosystem-neutral. pnpm should share the lifecycle and the resources it consumes. It should not make Cargo, npm and a future Python implementation conform to one resolver, one package-name grammar or one lockfile model.

The shared install kernel owns orchestration. It supplies an install context containing the configured HTTP client and aggregate concurrency budget, request-auth routing, store access, reporting, cancellation, filesystem capabilities and install-wide flags. It schedules independent ecosystem plans concurrently, coordinates failure, and commits their staged workspace changes. Workspace discovery and verified artifact ingestion also belong here once more than one adapter needs them: the repository should be walked once, and download, integrity verification, extraction and CAS insertion should have one implementation.

An ecosystem adapter owns the meaning of its inputs and outputs. It detects its manifests within the shared workspace inventory, parses its own package specifiers, resolves according to its native rules, reads and writes its native lockfile, and produces its ecosystem-specific projection from verified store content. Cargo therefore keeps feature unification, source keys, Cargo semver and `Cargo.lock`; npm keeps peers, npm package identities and `pnpm-lock.yaml`; a Python adapter will own extras, environment markers, wheels and whichever native lockfile contract is selected. Package identities crossing the shared boundary are ecosystem-qualified values rather than strings that assume npm scopes or Cargo's flat namespace.

The adapter boundary is initially internal and plan-based, not a public plugin API. A command partitions package specifiers by protocol and asks each selected adapter to prepare a plan. A plan progresses through discovery, resolution and validation, verified artifact ingestion, staged workspace writes, commit, and projection materialization. Store insertions may survive a failed install because immutable unreferenced content is harmless and prunable. User-owned manifests and native lockfiles must not be left in a mixed old/new state when another ecosystem's plan fails. Generated projections use completion markers or atomic replacement so an interrupted install is distinguishable from a complete one.

Verified artifact ingestion ends at an explicit projection boundary. Integrity identifies immutable archive bytes and CAS blobs, which ecosystems may share. It does not identify the file map produced after extraction: npm may add a completion marker or synthesize a runtime `package.json`, Cargo must preserve the raw archive before adding `.cargo-checksum.json`, and a Python wheel will have its own projection rules. Every cache tier that stores a projected file map — in-memory work sharing and persistent store-index rows alike — therefore includes every input that can change that map. Projection namespaces and content hashes prevent cross-ecosystem poisoning while leaving the ordinary npm key and hot path unchanged.

Changing a projection key is a cache migration even when the underlying bytes are immutable. A new raw projection intentionally re-ingests an old npm-shaped Cargo row once. A synthesized-manifest projection may fall back to a legacy row only after its new key misses and only when the cached manifest bytes exactly match the requested manifest. Compatibility belongs at this boundary and is tested for warm, offline and both-completion-order cases; adapters must not silently accept a row produced under different projection semantics.

Credential discovery is separate from request authentication. `.npmrc`, pnpm's global configuration, Cargo credentials and future Python credential sources may all feed a normalized URL-based routing layer. Ecosystem concepts such as npm scopes remain inputs to their adapter rather than fields every credential source must emulate.

The workspace inventory slice in [pnpm/pnpm#14581](https://github.com/pnpm/pnpm/pull/14581) collects requested manifest basenames in one traversal and initializes the result lazily from the concurrently polled ecosystem work. Cargo owns the interpretation of its discovered manifests and reduction to workspace roots. The inventory is not constructed on npm-only installs. A later Python reader must distinguish installable projects from tool-only `pyproject.toml` files and exclude managed environments from discovery.

The next Python experiment is test-only. It resolves a fixture graph of exact-pinned, pure-Python wheels and exercises the existing shared inventory, URL-routed authentication, verified ZIP ingestion, raw CAS projection, offline store replay, and mixed-writer settlement and rollback. Python owns wheel identity checks, RECORD verification, dependency interpretation, and installed-path collision checks. The fixture produces no workspace writes until resolution and validation succeed. Its declared file footprint lets the existing metadata transaction restore all participating lockfiles and remove newly projected Python files after another participant fails; immutable CAS insertions may remain. This does not yet establish crash-safe publication or safe replacement of an existing virtual environment.

For now, Python uses the standard [PEP 751 `pylock.toml` format](https://packaging.python.org/en/latest/specifications/pylock-toml/), separate from `pnpm-lock.yaml`. A Python section in the YAML file could preserve separate resolver semantics, but would also require coordinated ownership of its writes, freshness, and pruning. That is not the direction chosen for this experiment. `uv.lock` remains uv-owned: uv documents capabilities beyond pylock, but also [discourages tools from depending directly on its lockfile format](https://docs.astral.sh/uv/reference/internals/metadata/). uv already supports [exporting and installing pylock files](https://docs.astral.sh/uv/concepts/projects/layout/#relationship-to-pylocktoml). Revisit the standard format only when a concrete pnpm requirement demonstrates a limitation.

This boundary is deliberately demand-driven. The Python fixture is not a general Python resolver or environment installer and does not justify a stable adapter trait. General requirements, extras, markers, target environments, native wheels, source builds, and interpreter-dependent installation still need validation. The current authenticated GET helper also needs per-request content negotiation before a real Simple API client can request JSON through the same network and redirect policy; the fixture endpoint always serves JSON. Resolver algorithms, manifest grammars, feature or marker evaluation, lockfile formats and projection layouts remain explicit non-goals for unification.

#### npm install performance is an invariant

The architecture must retain the current performance of an npm-only CLI install. When no non-Node.js ecosystem is enabled, dispatch stays on the existing npm path: no adapter collection is built, no additional manifest discovery runs, and no ecosystem-neutral workspace inventory, lockfile parsing, filesystem traversal or synchronization primitive is introduced. Adapter dispatch is once per participating ecosystem, never once per dependency or inside resolver and linker hot loops.

A mixed install shares the existing install-wide HTTP client and its limits. Adding an adapter must not multiply `networkConcurrency` or create an independent socket budget, and scheduling another ecosystem must not serialize the npm graph behind it. The repository's integrated benchmark suite is the acceptance gate for every extraction touching install. A repeatable npm-only regression means the abstraction must change; the regression is not an accepted cost of the multi-ecosystem design.

### pnpr as a crate registry

pnpr's registry model is preserved and gains a discriminator. A registry declares its **protocol**, and everything else about it — its name under `~<name>/`, its kind, its namespace, its access rules, the no-fall-through invariant — is the same model:

```yaml
registries:
  crates-io:
    type: upstream
    protocol: cargo
    url: https://index.crates.io/                 # sparse index root; dl/api come from its config.json
    public: true
  internal:
    type: hosted
    protocol: cargo
    org: acme
    access: team:acme
    packages:
      'acme-*': { publish: team:acme-rust }
  crates:
    type: router
    sources: [internal, crates-io]

  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true
```

`protocol` defaults to `npm` so no existing configuration changes meaning. A router's sources must share a protocol, statically validated at startup like every other router rule — a request has one URL grammar, and a router that mixed two could answer neither.

**The namespace language gains one crate-shaped pattern.** Crate names have no scopes, so `@scope/*` and `@*/*` are rejected on a `cargo` registry, and `<prefix>-*` is accepted: exactly one trailing `*` after a literal ending in `-` or `_`, matching what crates.io's own naming conventions produce. `**` and exact names carry over. Shadowing detection extends naturally — `acme-*` covers `acme-telemetry`, and a router listing them in the wrong order is refused. Crate names are matched with `-` and `_` treated as equal, as crates.io does, so a namespace cannot be escaped by respelling.

A `cargo` registry at `https://<pnpr>/~<name>/` serves:

| Path | Role |
|---|---|
| `config.json` | `{"dl": "<base>/api/v1/crates", "api": "<base>", "auth-required": <not public>}` |
| `1/<n>`, `2/<n>`, `3/<c>/<n>`, `<ab>/<cd>/<n>` | index files, one JSON line per version, `ETag`/`Last-Modified` with `304` |
| `api/v1/crates/{name}/{version}/download` | the `.crate` |
| `PUT api/v1/crates/new` | publish: `u32le` json length, json, `u32le` crate length, crate |
| `DELETE …/{name}/{version}/yank`, `PUT …/unyank` | the only mutation of an index line |
| `GET/PUT/DELETE api/v1/crates/{name}/owners` | owners, mapped onto pnpr's teams |
| `GET api/v1/crates?q=&per_page=` | search over hosted crates |
| `me` | where `cargo login` sends the user to obtain a token |

The `api` base has no trailing slash and `dl` has no markers, so cargo appends `/{crate}/{version}/download` — the default and the shape crates.io's own `dl` uses. Both URLs are relative to the base the client addressed, which is the rule the npm side already follows for `dist.tarball`: a registry served under `/~crates/` advertises `/~crates/` and no pnpr host is ever persisted anywhere, because `Cargo.lock` persists source keys.

**Upstream mode proxies crates.io as a mirror, which is the only thing cargo's replacement rule permits it to be.** Index files are fetched conditionally from `https://index.crates.io/<path>`, cached under the upstream's namespace, and served byte-identical — pnpr does not rewrite index lines, because a dependency line's `registry: null` means "the index this line came from" and a proxied index remains that index. Crates are fetched from the `dl` in the upstream's own `config.json` (`https://static.crates.io/crates/{crate}/{crate}-{version}.crate`), verified against the index `cksum`, and cached. crates.io's crawler policy requires an identifying `User-Agent`; pnpr sends `pnpr/<version> (+https://github.com/pnpm/pnpm)` rather than the `pnpm` string it inherits from the shared client today. The circuit breaker, throttled client and `maxage`/`timeout` knobs are the existing ones.

**Hosted mode stores `.crate` files by name and version beside the index file, and the index file is the packument.** A publish parses the framing, checks the metadata against the `Cargo.toml` inside the archive, applies crates.io's name rules — ASCII alphanumerics with `-` and `_`, first character alphabetic, at most 64, no Windows reserved names, case- and `-`/`_`-insensitive collision against existing names — checks the namespace, rejects a version that already exists including one differing only in build metadata, never compiles anything, and appends one line to the index file under the same compare-and-swap the packument write already uses (`write_packument_if_current`), so two concurrent publishes of the same crate serialize rather than clobber. The line's `cksum` is computed by pnpr from the received bytes, not trusted from the client. Yank flips `yanked` on the line and nothing else; `enforce_published_version_immutability` and `TarballFinalize::Conflict` apply to `.crate` bytes exactly as to `.tgz` bytes. The crash-atomic journal covers the pair of writes.

**Authentication accepts cargo's header.** Cargo sends the token bare — `Authorization: <token>` — where npm clients send `Bearer <token>`; `identify()` accepts the bare form on `cargo` registries only. The token itself is the same opaque pnpr token: a user who ran `pnpm login` against a pnpr host pastes the same value into `cargo login --registry <name>`, and `pnpm login --cargo-registry <name>` can write it into `credentials.toml` for them. A `401` on a non-public registry carries `WWW-Authenticate: Cargo login_url="<base>/me"` so cargo prints the right hint. Owners are a view over registry teams: `GET owners` lists the team members allowed to publish the name, `PUT`/`DELETE` are refused with the same `reject_team_mutation` answer the npm team API gives, because pnpr's teams are declared in configuration, not edited through a client.

**OSV screening keys on the registry's protocol.** `pnpr-osv` filters the local dump on `ecosystem == "npm"` today; it loads `crates.io` for cargo registries, and the resolver-side guard is unchanged in shape. Search is the same one-shot scan over hosted metadata, rendered in the web API's `{"crates": [...], "meta": {"total": n}}` shape.

**What pnpr does not need for pnpm to work, and vice versa.** pnpm speaks the sparse protocol, so it works against crates.io directly, against kellnr, against Artifactory. pnpr speaks the sparse protocol, so plain `cargo` works against it with a `[registries]` entry and no pnpm at all. The pair is better than either alone — pnpr's access model and pnpm's lockfile discipline — but neither is load-bearing for the other, and the dogfooding plan below depends on that.

### Parity with the TypeScript CLI

The Cargo ecosystem lands only in the Rust pnpm v12 CLI. This follows the repository's version policy in `AGENTS.md`: new features target v12, while the TypeScript v11 CLI is maintained for bug fixes. Shared bug fixes still require implementation and tests in every affected version. Ecosystem features and the supporting architecture do not require a TypeScript implementation merely because they extend shared concepts such as workspace discovery or artifact storage.

### Dogfooding this repository

The migration is the acceptance test, and it is incremental. Each step is useful without the next.

1. **pnpr proxies crates.io.** A `cargo` upstream registry is deployed; this repository's `.cargo/config.toml` replaces `crates-io` with a `registry` source pointing at it. Plain cargo, no pnpm involvement, and every CI build exercises pnpr's cargo upstream, cache, and breaker. This is the first shippable milestone and the one that finds pnpr's bugs earliest.
2. **pnpm resolves and reproduces `Cargo.lock`.** `pnpm install --lockfile-only` against the repository must leave `Cargo.lock` unchanged. This gate runs in CI before any install path is trusted.
3. **pnpm materializes; cargo builds offline.** `cargo.enabled: true`, the managed region replaces the step-1 mirror entry, `pnpm install --frozen-lockfile` precedes every `cargo` invocation in `pacquet-ci.yml`, `build-pnpr.yml`, `release.yml`, and the benchmark workflows, and `Swatinem/rust-cache`'s registry caching is dropped in favour of the store cache `pnpm/action-setup` already restores. `cargo build --locked --offline` is the invocation from then on.
4. **pnpm replaces the satellites.** `pnpm audit` and `pnpm licenses` take over `deny.toml`'s advisories and licenses sections in `audit.yml`; Dependabot's cargo ecosystem is replaced by `pnpm update`; the `chore(cargo): bump` commit convention goes away.

Bootstrapping needs an explicit transition. Until a released pnpm understands `cargo.enabled`, the repository cannot commit that setting or the source-replacement block: CI installs the released binary before it builds the branch, and that binary would either reject the setting or point Cargo at a directory it cannot populate. During milestone 2, a CI job first builds the candidate pnpm with ordinary Cargo and then uses that binary for the lockfile-reproduction gate. The repository commits the opt-in and generated source replacement only after the feature ships in a released pnpm; from then on the released pnpm builds the next pnpm, as the released TypeScript pnpm always installed the TypeScript workspace.

## Rationale and Alternatives

### Stop at a crates.io mirror in pnpr

Deploy step 1 of the dogfooding plan and declare victory: cargo keeps managing dependencies, pnpr serves them. It is cheap, it is a real milestone, and it is kept as one. As the whole answer it is rejected: it dogfoods the registry and nothing else, and the resolver, store, lockfile and fetcher — the parts of pnpm with the most surface and the most history of subtle bugs — stay untouched by our own build.

### Let cargo resolve and have pnpm only fetch

Read a cargo-written `Cargo.lock` and materialize it. This halves the work and avoids the resolver. Rejected, because it cannot be made coherent: once crates.io is replaced by a directory source, cargo cannot consult an index, so `cargo add` and `cargo update` stop working and the lockfile can only be changed by temporarily un-replacing the source. Two resolvers writing one lockfile, each blind while the other holds the pen, is the worst of the options. If pnpm is going to own the directory it has to own the resolve.

### Embed cargo's resolver as a library

The `cargo` crate is on crates.io and its resolver is the oracle this RFC tests against. Rejected as a dependency: its API is explicitly unstable, it pulls in the whole of cargo (git2, curl, the build system), and it couples pnpm's release cadence to a toolchain's. It remains the oracle in the differential tests, where a version coupling is a feature.

### A `local-registry` source instead of a `directory` source

Cargo's other vendoring shape keeps `.crate` archives and an index. Rejected: pnpm's store holds unpacked files, not archives, and a local registry would mean keeping both or re-packing on link. The directory source is a slot filled straight out of the store and shared by every workspace on the machine, which is the property pnpm exists for. The cost is the synthesized `.cargo-checksum.json`, which is one small file per crate.

### Record crates in `pnpm-lock.yaml` and derive `Cargo.lock`

One lockfile for the workspace is attractive and is rejected. Cargo will not run without `Cargo.lock`, so the derived file exists regardless; two files describing one graph is a consistency obligation with no consumer on the pnpm side that needs the second copy. Cargo's lockfile is well specified, versioned, and already what every Rust tool reads. The registry-identity question the npm lockfile is still working through — how two registries resolving the same `name@version` avoid colliding — cargo answered by writing the source key next to every package, and that answer is adopted rather than re-derived.

### Inject `--config` when spawning cargo instead of editing `.cargo/config.toml`

Keeps the user's file pristine and works for `pnpm run build`. Rejected because the user's editor does not run cargo through pnpm: rust-analyzer, `cargo clippy` from a shell, CI steps that call cargo directly would all fetch from crates.io and silently bypass the workspace's decision. A committed, generated region is explicit about what it owns and fails loudly when skipped.

### A per-workspace tree of hardlinked files instead of the global virtual store

The first draft of this RFC materialized every crate's files into the workspace, hardlinked from the store, the way `node_modules` is built without the global virtual store. The objections to a central layout were that cargo scans a directory source wholesale, that a shared tree cannot be pruned per workspace, and that a committed `.cargo/config.toml` needs a relative path. All three are objections to a *single shared directory source*, and none survives a per-workspace symlink farm: cargo scans only the workspace's links, the links are pruned per workspace and the slots per machine, and the config path stays relative. Rejected in favour of the layout above; a self-contained tree remains a possible opt-in for checkouts that must build without the store present, which is an open question.

### Infer management from the presence of `Cargo.toml`

Zero configuration for pure-Rust projects. Rejected for the napi-rs case: a workspace that has a `Cargo.toml` because one package binds to native code should not have its `pnpm install` start resolving crates, rewriting cargo configuration, and failing on a resolver disagreement, without anyone asking for that.

### A separate `pnpm-cargo` plugin binary

Isolates the new code and the new dependencies. Rejected because most of the value is in the integration — one project graph, one filter language, one task scheduler, one audit, one registry client with one auth story — and a plugin would re-implement or bypass each of them.

## Implementation

### pnpm

The initial proof of concept implements a deliberately smaller vertical slice: when `cargo.enabled` is set, `pnpm install` invokes `cargo metadata --no-deps` only to normalize workspace manifests, fetches crates.io sparse-index entries itself, resolves the fresh maximal graph with `pubgrub` and Cargo's `semver` and index schemas, writes a format-v4 `Cargo.lock` through `cargo-lock`, and materializes that lock through pnpm's store. This work runs concurrently with the npm install through an ecosystem coordinator whose task collection can grow beyond two languages; both paths share the install-wide throttled HTTP client. A fixture with an optional workspace dependency and serde's derive graph is byte-identical to `cargo generate-lockfile` and builds with `cargo build --locked --offline` from the generated directory source. It activates every workspace feature, unifies requested registry features, includes target-specific and workspace dev-dependencies while excluding registry dev-dependencies, understands renamed dependencies, and excludes freshly yanked versions.

The proof of concept also accepts explicit `crate:` selectors in `pnpm add`. Selector partitioning happens before npm parsing, and non-Node ecosystems have a separate manifest-preparation boundary, so a later Python protocol does not have to enter npm-specific code. A crate without a requirement selects the newest stable, non-yanked release. The command updates the selected Cargo dependency table without reformatting unrelated manifest content, regenerates `Cargo.lock`, and runs a mixed npm/Cargo add through the same concurrent installer coordinator. A crate-only add creates no Node manifest or lockfile. Adding from a Cargo workspace member changes that member's manifest and keeps `Cargo.lock` at the workspace root.

The first shared registry-auth slice establishes a URL-routed request-auth representation rather than a universal credential format. npm configuration continues to contribute already-formatted npm headers, while the Cargo adapter reads the crates.io token from Cargo's environment and credential files and contributes Cargo's bare `Authorization` value. Discovery happens once at the command boundary, only after Cargo work is selected, and is skipped by offline operation; npm-only installs therefore do not inspect Cargo environment variables, home directories, or files. Adding an overlay preserves unrelated and more-specific routes, and redirects look up authorization again for the destination URL so credentials are retained on same-origin redirects but not forwarded across origins. Credential-file parse errors identify the file without including parser output that could disclose its contents.

That slice deliberately scopes a crates.io token to `https://index.crates.io/`, not to the separate `static.crates.io` download host. Explicit pnpm URL-matched authentication can authorize either route. Named Cargo registries still require the planned source-key-to-index mapping and Cargo configuration discovery; once that exists, their credentials will feed the same request-auth representation rather than adding registry concepts to the shared network layer. A future Python adapter should follow the same split: interpret ecosystem configuration at its boundary, format the scheme it requires, and supply URL routes to the common client.

That result validates the end-to-end seam, not Cargo compatibility. The proof of concept is crates.io-only. Plain install does not yet validate or refresh a stale lock, while add regenerates the graph without preferring existing locked versions. Alternate registries, git and non-member path dependencies, `[patch]`, `links` uniqueness, resolver-3 MSRV filtering, conditional index requests, recursive and filtered crate adds, feature-selection flags, optional crate declarations, and backtracking to older compatibility lines and candidates whose dependency names differ from the newest candidate are not implemented. Those remain acceptance gates before enabling dogfooding in this repository.

New crates under `pnpm/crates/`, following the domain-prefix convention:

1. **`cargo-manifest`** — read `Cargo.toml` with `cargo-util-schemas` (`TomlManifest`), apply workspace inheritance (`workspace = true` for dependencies, `package.*`, `lints`), and write edits with `toml_edit`. Extends `workspace-manifest-writer`'s format-preserving approach.
2. **`cargo-index-client`** — sparse index fetch over `pnpm-network` with the conditional-request mirror at `<cache_dir>/v11/cargo-index/`, `config.json` discovery, cargo config discovery (`[registries]`, `credentials.toml`, `CARGO_REGISTRIES_*_TOKEN`) and the `cratesIoIndex` override. Index lines typed by `cargo-util-schemas::index`.
3. **`cargo-resolver`** — PubGrub solver with cargo's semantics: feature unification into the maximal graph, optional and weak dependencies, `dep:` features, `links` uniqueness, yank rules, MSRV eligibility under `resolver = "3"`, `[patch]`, lock preference and `update` widening. `PackageVersionGuard` is honoured for `audit --fix`.
4. **`cargo-lockfile`** — `Cargo.lock` v4 read/write, starting from the `cargo-lock` crate; golden round-trip test on this repository's lockfile; canonical ordering and dependency qualification rules.
5. **`cargo-source-linker`** — fill and integrity-check crate slots under `<store>/crates/` through `deps-restorer::import_indexed_dir`; maintain the per-workspace symlink farms (junctions on Windows), prune them to the lock, write `.pnpm/crates/.state.json`, register the workspace in the store's projects registry, and regenerate the managed region of `.cargo/config.toml` idempotently.
6. **`cargo-registry-api`** — publish framing, yank/unyank, owners, search, against a registry's `api`.

Changes to existing crates:

7. **`workspace`** — `Cargo.toml` as a manifest basename; `Project` gets an optional crate manifest; discovery unions `[workspace].members`. **`workspace-projects-graph`** — edges from path deps and `workspace = true`. **`workspace-projects-filter`** — name aliases over the union.
8. **`config`** — `cargo.enabled` and `cargo.cratesIoIndex` in `WorkspaceSettings`/`Config`/`known_settings.rs`.
9. **`tarball`** / **`store-dir`** — `.crate` unpack with the synthesized `.cargo-checksum.json`; sha256-keyed index rows (`algo` per row already exists); `StoreDir::crates()` beside `links()`; `store prune` extended to crate slots via the projects registry. No store version bump.
10. **`deps-path`** — `crate` in `RESERVED_VERSION_PREFIXES`.
11. **`package-manager`** / **`cli`** — independent ecosystem phases of `install`, coordinated concurrently through the shared install context; `--frozen-lockfile`/`--lockfile-only`/`--offline` plumbing and explicit settlement before mixed-write rollback.
12. **`cli`** — `crate:` handling in `add`/`remove`/`update`/`outdated`/`list`/`why`/`audit`/`licenses`/`publish`/`pack`; the `--features`, `--no-default-features`, `--save-build`, `--optional` flags; ecosystem defaulting by project shape; `login --cargo-registry`.
13. **`deps-inspection`** — a crate node kind and `Cargo.lock` as a graph source.
14. **`AGENTS.md`** (root and `pnpm/`) — apply the existing v12-only feature policy and affected-version coverage for shared bug fixes.

New third-party dependencies, each needing the usual approval and `deny.toml` review: `pubgrub`, `cargo-util-schemas`, `cargo-lock`, `toml_edit`, `cargo-platform` (to parse `cfg()` targets for display and validation, not evaluation). `semver` is already present.

### pnpr

15. **`pnpr-registry`** — `protocol: npm | cargo` on `RegistryFile`/`Registry`; router protocol homogeneity in `Registries::validate`; `PackagePattern::Prefix` for cargo namespaces with `-`/`_` folding; `@`-patterns refused on cargo registries.
16. **`pnpr-cargo`** (new crate) — handlers for `config.json`, index paths, download, publish, yank/unyank, owners, search, `me`; framing parser; crates.io name validation; index-line construction with server-computed `cksum`.
17. **`pnpr`** routing — the generic segment-count dispatch gains the cargo path shapes under a cargo-protocol `~<name>/`, and the path-less base when `defaultRegistry` names a cargo registry.
18. **`pnpr-storage`** — `<crate>/index` and `<crate>/<crate>-<version>.crate` beside the existing layout; CAS append for the index file over `write_packument_if_current`; journal coverage.
19. **`pnpr-upstream`** — `fetch_index_file` (conditional) and `fetch_crate` (by the upstream's `dl`), a cargo-specific `User-Agent`, and `config.json` discovery for the upstream.
20. **`pnpr-auth`** — bare-token `Authorization` on cargo registries; `WWW-Authenticate: Cargo login_url=…`.
21. **`pnpr-osv`** — ecosystem by protocol. **`pnpr-search`** — the crates response shape.
22. **`pnpr-fixtures`** — a small crate fixture set (a `links` crate, a proc-macro, an optional dependency, a yanked version) and index files for it.

### Order of delivery

Steps 15–21 first (dogfooding milestone 1, pnpr proxying crates.io for plain cargo); then 1–4 with the reproduce-`Cargo.lock` gate (milestone 2); then 5, 7–11 (milestone 3, offline builds); then 6, 12–13 and the satellite replacements (milestone 4).

### Tests should cover

- **Lockfile reproduction**: this repository's `Cargo.lock` is reproduced byte-for-byte from its manifests with the lock as preference; `cargo metadata --locked --offline` accepts every lockfile pnpm writes without rewriting it.
- **Differential resolution** against `cargo generate-lockfile` on a fixture corpus: semver-compatible unification, backtracking to an older version on conflict, `0.x` minor-as-major, pre-release matching, `=`/`~`/`*` requirements, optional deps activated by features from different dependents, `dep:` and weak `?/` features, `links` conflicts (must error, matching cargo's message class), yanked versions kept from a lock and refused fresh, `rust_version` eligibility under `resolver = "3"` with fallback when nothing eligible exists, `[patch]` overriding an index candidate, renamed dependencies (`package =`), target-specific dependencies included regardless of host, dev-dependencies of members included and of non-members excluded.
- **Fetch and store**: `.crate` checksum mismatch fails before any file reaches the store; the synthesized `.cargo-checksum.json` matches what `cargo vendor` writes for the same crate; sha256-keyed rows do not collide with sha512 rows; a store shared between the TypeScript CLI and pacquet round-trips crate rows.
- **Materialization**: a slot is filled once and reused by a second workspace; an edited file fails Cargo's checksum check and the next pnpm install repairs the slot from verified CAS content; two registries serving different bytes for one `name@version` get two slots; pruning of a removed crate's link leaves the slot until `store prune` finds no registered workspace referencing it; two versions of one crate side by side; one directory per source key; junctions on Windows; `cargo build --locked --offline` succeeds on a fresh clone after `pnpm install`; the managed region is idempotent, preserves the user's own tables and comments, and is regenerated when a git source is added or removed.
- **Commands**: `add` writes `workspace = true` when the root catalog has the crate and a requirement otherwise; `-O` on a `crate:` spec is refused; bare names default by project shape; `remove` cleans the managed region; `update`/`--latest`; `audit` against a recorded OSV response; `licenses` over both ecosystems; `publish --dry-run` framing round-trip; `unpublish` yanks.
- **Workspace**: a directory with both manifests is one project; `--filter` by crate name; `...[ref]` over crate paths; a name shared by an npm package and a crate in different directories is an error listing both.
- **pnpr protocol**: `config.json` per base, index path scheme for 1/2/3/4+ character names, `304` on `If-None-Match`, `404` for unknown names, publish framing incl. truncated and oversized bodies, `cksum` computed server-side, duplicate version including build-metadata variants refused, yank/unyank as the only mutation, owners as a team view, search shape, bare-token auth accepted on cargo and refused on npm registries, `WWW-Authenticate` on `401`, `-`/`_` namespace folding, router protocol mismatch refused at startup, upstream served byte-identical to `index.crates.io` with the crawler `User-Agent`, breaker and cache behaviour under upstream failure, journal recovery across a crashed publish.
- **End to end**: plain `cargo` against a pnpr cargo registry with only a `[registries]` entry; pnpm against `index.crates.io` with no pnpr; pnpm against pnpr; the repository's own CI on milestone 3.

## Prior Art

- **`cargo vendor`** and **`cargo-local-registry`** are the two cargo-blessed shapes for building without the network and define the directory-source contract this RFC targets, including `.cargo-checksum.json` and the source-replacement config snippet. Nix's `cargo vendor`-based builders and Bazel's `rules_rust` crate_universe both vendor into a directory source, which is evidence the seam is stable under real load.
- **JSR in pnpm** is the existing precedent for a second registry namespace, and the contrast is instructive: JSR was folded into the npm resolver as a name mapping because JSR speaks npm's protocol. Cargo does not, which is why this is a resolver and not a mapping. The **runtime resolvers** (`node@runtime:`, Deno, Bun, Yarn) are the precedent for a separate artifact protocol with its own lockfile resolution shape.
- **pixi** manages conda and PyPI dependencies in one lockfile with one CLI; **uv** resolves Python packages with a PubGrub resolver of its own rather than pip's. Both demonstrate that a second ecosystem behind one CLI is a product, not a hack, and that "own the resolver, test against the incumbent" is the working pattern.
- **Artifactory**, **Cloudsmith**, **kellnr**, **ktra** and **Alexandrie** serve cargo's sparse protocol; the first two serve it beside npm under one access model, which is the deployment pnpr is aiming at. **panamax** mirrors crates.io wholesale and is the reference for the mirror's obligations (identical bytes, identifying user agent).
- **cargo-deny**, **cargo-audit** and **cargo-license** are the satellites `pnpm audit` and `pnpm licenses` replace for this repository; RustSec's advisory database is what OSV's `crates.io` ecosystem republishes.
- The **registry-mounts** RFC's `protocol`-agnostic rules — declared namespaces, ordered routers, no fall-through, unavailable is not not-found — are adopted unchanged; the **integrity-addressed tarballs** and **auth-aware cache** RFCs are npm-shaped and not extended here, though nothing prevents a `.crate` from being served by digest later.

## Unresolved Questions and Bikeshedding

- **Specifier prefix**: `crate:` reads naturally in `pnpm add crate:serde` and is what this RFC uses; `cargo:` names the tool rather than the thing. Lean `crate:`.
- **Opt-in key**: `cargo.enabled` under a `cargo:` block that can grow (`cratesIoIndex`, a future `sourceDir`, a future toolchain setting), versus a flat `manageCargo: true`. Lean the block.
- **Directory name**: `.pnpm/crates/` at the workspace root is new territory — `node_modules/.pnpm` is inside a directory cargo has no reason to read, and `target/` is cargo's to `clean`. A single `.pnpm/` root beside `node_modules/` also gives a future home to other non-npm materializations. Lean `.pnpm/crates/`.
- **A self-contained layout for checkouts without a store**: a Docker build that copies the workspace, or a `cargo vendor`-style handoff, needs the crate files inside the workspace rather than links into a store that is not there. `pnpm deploy` is the npm answer; a `--cargo-source-layout=local` that fills the source directories with hardlinked or copied files is the obvious crate answer, and is not proposed here. Lean a follow-up once the global layout has shipped.
- **`--optional`** for cargo's feature-gated dependencies collides in spirit with `-O`. `--optional` is what `cargo add` calls it and this RFC keeps it, refusing `-O` on crate specs. Alternatively `--feature-gated`. Lean `--optional`.
- **Committing the managed region** versus generating `.cargo/config.toml` into an ignored file. The committed region is the explicit contract and the dogfooding requirement; it also means every contributor to a cargo-managed workspace runs pnpm. That is the intent for this repository and a stronger ask for a general Rust project. Lean committed, with the failure mode documented.
- **How faithfully the `cargo-lock` crate serializes**: it is a parser with a serializer, not cargo's serializer. If it cannot be made byte-identical for v4, `cargo-lockfile` writes its own, which is a small amount of code. Verify before milestone 2.
- **Resolver version and the lock's package set**: the claim that `Cargo.lock` is independent of `resolver = "1"/"2"` beyond MSRV eligibility should be confirmed against `cargo::ops::resolve_ws` before it is relied on; the differential tests will settle it either way.
- **`registry:` on hosted index lines**: cargo sends a dependency's registry URL on publish when it differs from the target registry, and `null` otherwise. A hosted pnpr registry that is only ever addressed through a router base serves lines whose `null` means "this base", which is right; a client that addresses the hosted registry directly and whose dependencies live on crates.io needs those lines to name crates.io. Storing what cargo sent is correct in both cases and is what this RFC does; whether pnpr should rewrite `null` to an explicit URL when serving a hosted registry outside a router is open.
- **Writing `credentials.toml` from `pnpm login`**: convenient, and it puts pnpm's hands on a cargo-owned file. `--cargo-registry <name>` as an explicit opt-in per invocation is the lean; never by default.
- **Namespace patterns**: `<prefix>-*` is one pattern kind; crates.io's own convention makes a prefix the natural unit, but a registry that wants a list of exact names has that already. Whether `*` should be allowed mid-name is not proposed. Lean prefix-only.
- **Implicit tasks for crate projects** (`build` → `cargo build -p <name>`, `test` → `cargo nextest run -p <name>`) so `pnpm -r run build` and the task cache work over a mixed workspace without a `package.json` per crate. Likely wanted and out of scope here; the task cache RFC is the place, since a `cargo build` task's inputs and outputs are the hard part.
- **Toolchain management**: `rust-toolchain.toml` maps cleanly onto `devEngines.runtime` and the `runtime:` resolver family, with `static.rust-lang.org` manifests signed the way Node's shasums are. Follow-up RFC; rustup stays in the meantime.
- **Where the pnpr half lives**: the integrity-addressed tarball work was split into a `text/` document for pnpm and a `pnpr/text/` companion under pnpr's licence. This RFC is one document because the protocol split is the design; if the repository prefers the pair, the "pnpr as a crate registry" section and steps 15–22 move as-is.
