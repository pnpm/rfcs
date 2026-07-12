# Native monorepo versioning

## Summary

pnpm gains built-in release management for workspaces: recording change intents as files, consuming them to bump versions across the workspace (with dependent propagation, fixed/linked groups, and per-package prerelease lines), and writing changelogs. The intent-file format is compatible with [changesets](https://github.com/changesets/changesets), so existing repositories can adopt it by deleting a devDependency. Configuration lives in `pnpm-workspace.yaml`.

## Motivation

pnpm already owns both ends of the release pipeline. It installs and links the workspace, understands the `workspace:` protocol, builds the project dependency graph, rewrites manifests at pack time (`publishConfig`, `workspace:` ranges, catalogs), and publishes recursively in topological order with provenance and trusted publishing. The one step in the middle — deciding *what version everything becomes next* — is delegated to external tools, most commonly changesets.

That delegation has real costs:

1. **A second tool with a second config.** Every workspace that releases packages carries `@changesets/cli` as a devDependency, a `.changeset/config.json` that duplicates knowledge pnpm already has (which packages exist, which are private, how they depend on each other), and a second mental model for contributors.

2. **The external engine cannot use what pnpm knows.** Changesets re-derives the workspace through `@manypkg`, has its own partial understanding of the `workspace:` protocol, and knows nothing about catalogs — a catalog bump changes a package's dependencies without touching its directory, defeating the directory-diff change detection external tools rely on (raised in [pnpm/pnpm#2225](https://github.com/pnpm/pnpm/issues/2225); catalog support in changesets is still an open issue, [changesets/changesets#1707](https://github.com/changesets/changesets/issues/1707)). Dependent propagation ("bump the packages whose dependency ranges no longer match") is a graph question pnpm answers natively for `--filter` and topological builds.

3. **Structural limitations show up in real repositories.** pnpm's own monorepo hit three while unifying its release process (pnpm/pnpm#12949):
   - **Prerelease mode is global to the workspace.** Changesets' pre mode is a single `pre.json` for everything, so a repo cannot release stable versions of some packages while others sit on a prerelease line (e.g. `12.0.0-alpha.N`). pnpm's repo works around this with a script that re-derives prerelease versions after every `changeset version` run.
   - **Everything is keyed by package name.** The changeset file format references packages by bare name, so the engine silently misbehaves when names collide, and there is no way to aggregate prerelease changelogs per package.
   - **Only `X.Y.Z-tag.N` prerelease lines are expressible.** Any other scheme has to be abandoned or maintained by hand.

4. **The ecosystem around versioning is consolidating away from neutral tools.** Lerna is maintained by Nx and steers toward Nx adoption; changesets is community-maintained with a slow release cadence. A package manager is the natural neutral home for this capability, the same way `pnpm patch`, catalogs, and workspace publishing absorbed adjacent territory.

A native release command has been an open request since 2019 ([pnpm/pnpm#2225](https://github.com/pnpm/pnpm/issues/2225), following [pnpm/pnpm#1098](https://github.com/pnpm/pnpm/issues/1098) which produced `pnpm publish -r`). The expected outcome: a pnpm workspace can go from "PR merged" to "published with correct versions and changelogs" using pnpm alone, and repositories with per-package release trains (the situation pnpm's own monorepo is in) are supported natively instead of via wrapper scripts.

## Detailed Explanation

### Recording change intents

A new command records an intent file describing which packages a change affects, the bump type for each, and a human-written summary that becomes the changelog entry:

```
pnpm change [--bump <type>] [--summary <text>] [<pkg>...]
```

Run without arguments it is interactive (select packages, select bump types, write the summary), mirroring `changeset add`. With flags it is scriptable.

The on-disk format is the changesets format, unchanged — a Markdown file with YAML frontmatter in `.changeset/`:

```markdown
---
"@example/ui": minor
"@example/core": patch
---

Added a `variant` prop to `Button`.
```

Format compatibility is a deliberate, load-bearing choice (see Rationale). Existing `.changeset/*.md` files written by humans, bots, or the changesets CLI are consumed as-is.

`pnpm change status` reports pending intents and the release plan they would produce (the equivalent of `changeset status`).

### Enforcing coverage: `pnpm change check`

Adopted from Rush's `rush change --verify` and Yarn's `yarn version check`: a CI-runnable command that computes which workspace packages the branch's diff touches (against configurable base refs) and fails unless every touched package is either named in a pending intent file or explicitly declined. Declines are recorded in the intent frontmatter as a `none` strategy — an additive extension to the changesets format — so "this refactor needs no release" becomes a reviewable, per-package statement in the PR instead of a bot-comment convention or an opaque empty changeset. The touched-package computation is not new machinery: it is the changed-since detection already behind `--filter="[<ref>]"` (`@pnpm/workspace.projects-filter`), and the existing `changed-files-ignore-pattern` and `test-pattern` settings already express "these files never warrant a release" — no new configuration key is introduced. The interactive `pnpm change` preselects the same diff-derived package set.

The diff attribution is workspace-aware in a way a directory diff cannot be: a change to a catalog entry in `pnpm-workspace.yaml` marks every package referencing that entry as touched, even though no file in their directories changed. This is the change-detection gap catalogs opened for external tools, closed by owning both halves — and because it lands in the shared filtering machinery, `--filter="[<ref>]"` builds and tests gain the same catalog awareness.

Because the check is a plain command over git and the workspace graph, enforcement works on any CI and any forge — no webhook bot required.

### Applying versions

```
pnpm version -r [--filter <pattern>] [--dry-run] [--snapshot [<tag>]]
```

(Command naming is bikeshed; see Unresolved Questions.) The flags follow pnpm's existing conventions — there is no new `--workspace` concept. `pnpm version` keeps its current npm-style meaning when given an explicit bump argument (`pnpm version patch`, recursively with `-r`); the **bare** recursive form is what consumes pending intents. `--filter` narrows the release to the selected packages' portion of the pending plan, with fixed-group companions and range-invalidated dependents of the selection pulled in — which is how one product line releases from a repository that hosts several.

The bare recursive form:

1. **Assembles a release plan.** Direct bumps from intent files; propagation to dependents through the project graph pnpm already builds (a dependent whose dependency range no longer matches the new version — or that uses `workspace:*`/`workspace:^`/`workspace:~` — receives at least a patch bump); fixed and linked group constraints applied.
2. **Writes manifests** with pnpm's format-preserving manifest writer, updating both `version` fields and internal dependency ranges (with native understanding of the `workspace:` protocol and catalogs).
3. **Writes changelogs** — a `CHANGELOG.md` section per released package, composed from the consumed intent summaries.
4. **Deletes consumed intent files** (with two exceptions below: packages on a prerelease line, and registry changelog storage, which defers deletion by one release).

`pnpm publish -r` then completes the flow unchanged.

### Configuration

Configuration lives in `pnpm-workspace.yaml` under a single key, replacing `.changeset/config.json`:

```yaml
versioning:
  fixed:
    - ["@example/cli", "@example/cli-bindings"]
  linked:
    - ["@example/plugin-*"]
  ignore:
    - "@example/internal-fixtures"
  changelog:
    format: github      # or a changelog-writer package resolved like a config dependency
    storage: repository # opts out of the default "registry" mode: see "Changelog storage" below
```

Private packages without a `version` field are ignored automatically. The `fixed`/`linked`/`ignore` semantics match changesets so migration is mechanical.

One constraint key is adopted from Rush's `lockedMajor` version policy: `versioning.maxBump` caps the bump a release from the current checkout may apply (`maxBump: patch` rejects intents declaring `minor` or `major`). Because `pnpm-workspace.yaml` is committed per branch, a maintenance branch such as `release/11.1` sets the cap once, and a cherry-picked intent carrying a larger bump fails `pnpm version -r` loudly — naming the offending intent file — instead of silently shipping a feature release from a patch line.

### Changelog storage: the registry as the changelog archive

Committed changelogs are derived data and a chronic source of friction: they dominate release-PR diffs, conflict on every cherry-pick and merge-back, and record nothing the repository doesn't already hold — every intent file's prose stays in git history after deletion, and the published packages carry the rendered result. `versioning.changelog.storage` selects where the accumulated history lives:

- **`registry`** (default): no changelog file is committed at all. The intent files remain the only copy of the prose: `pnpm version -r` bumps manifests and, instead of deleting the consumed intent files, records their ids in a small committed ledger (mapping `package@version` → intent ids), so the tagged release commit is self-contained. At publish time, pnpm composes the release's changelog section from the ledgered intent files present at the tag, fetches the previously published version's tarball, prepends the new section to its `CHANGELOG.md`, and packs the concatenation into the new tarball. Consumed intents are garbage-collected by a later `pnpm version -r` run, and the timing is a safety property, not a convenience: until its release is published, the intent file in the repository is the only copy of the prose anywhere, so a file is deleted only when it is (a) ledgered, (b) recorded against a version the registry confirms is published, and (c) not still needed by a package on a prerelease line awaiting graduation. Deletion cannot happen at publish time because publish must not push commits; gating it on registry confirmation makes the lifecycle self-healing — a release that was versioned and tagged but never published keeps its intents (and their prose) intact for a rerun.
- **`repository`** (opt-out, changesets-compatible): `CHANGELOG.md` is appended in place and committed, and intent files are deleted as they are consumed — changesets' behavior, unchanged.

There is deliberately no committed rendering of the changelog in `registry` mode. The prose is reviewed once, in the PR that adds the intent file; the release PR's reviewable surface is the version bumps and the ledger entry, and `pnpm change status` / `--dry-run` renders the exact section that will be packed. Two alternative lifecycles were considered and rejected: committing the newest section into `CHANGELOG.md` duplicates the intent text for no review benefit (churn), and reconstructing deleted intent files from the release commit's diff at publish time breaks under shallow CI checkouts and rewritten release branches. Deleting intents at publish time instead of the next version run would require the publish step to push commits, which it must not.

The **previous version** is the highest published version semver-lower than the one being published — never a dist-tag lookup. This makes concurrent release lines chain correctly on their own ancestry: `12.0.0-alpha.9` extends `12.0.0-alpha.8`'s changelog, a backport `11.1.5` extends the `11.1.x` line's history, and `11.13.0` extends `11.12.x` even when higher prereleases exist. If the previous version is unpublished, the next-highest is used; a first publish starts the history from the new section alone. The fetched tarball is integrity-verified against the registry metadata, exactly as an install would be.

The default rests on preconditions that hold for most workspaces but not all; `repository` mode is the opt-out for the ones where they don't:

- **pnpm must be the publisher.** The composed section comes into existence only inside pnpm's pack/publish step; a version published by any other tool ships without it. To keep that mistake from destroying prose, the GC check verifies not just that the ledgered version exists on the registry but that its tarball's `CHANGELOG.md` actually contains the composed section — a foreign-published version therefore blocks intent deletion instead of losing the entries. Pipelines that pack or publish with other tools should opt out.
- **Ecosystem tooling reads changelogs from the repository, not from tarballs.** Renovate and Dependabot surface release notes from committed changelog files and GitHub releases; a repo in `registry` mode needs GitHub releases (or equivalent) for its changes to stay visible in downstream update PRs.
- **The published artifact embeds registry-fetched content**, so it is no longer byte-reproducible from the tagged sources alone — repositories that attest builds and consider that property load-bearing should opt out.
- There is no single rendered history file in the working tree. Nothing is lost — the prose remains in git history and the rendered history in every published tarball — but casual repo browsing needs GitHub releases (or the tarball) instead of a `CHANGELOG.md`.

For migrating changesets repositories, the codemod sets `changelog.storage: repository` when it finds committed `CHANGELOG.md` files, so adoption preserves their existing behavior; the default applies to setups initialized by pnpm.

### Per-package prerelease lines

The headline capability changesets structurally cannot offer. A package (or fixed group) can be placed on a named prerelease line:

```
pnpm version pre enter alpha --filter @example/cli...
pnpm version pre exit --filter @example/cli...
```

Membership is recorded in `pnpm-workspace.yaml` (`versioning.prereleases`), so it is reviewable state, and it is **per package**, not per workspace:

```yaml
versioning:
  prereleases:
    "@example/cli": alpha
```

While `@example/cli` is on the `alpha` line, `pnpm version -r` computes the stable target version from the intents as usual (say `2.1.0`), then emits `2.1.0-alpha.N`, incrementing `N` on each run. Packages not listed release stable versions from the same run. Exiting the line releases the accumulated stable version.

**Half-consumed intents are handled explicitly**, which is the design corner that makes this impossible to retrofit into changesets: an intent file may name both a stable package (its portion is consumed and the changelog written now) and a prerelease-line package (its portion must survive until graduation so the stable changelog aggregates every change since the line began). Instead of keeping or deleting whole files, consumption is tracked per package: when an intent is applied to a prerelease-line package, its summary is recorded into a per-package pending-changelog state (under `.changeset/`, exact shape TBD), and the file is deleted once every named package has consumed it. Graduation flushes the accumulated entries into the stable version's changelog section.

### Snapshot releases

`pnpm version -r --snapshot [tag]` produces one-off `0.0.0-<tag>-<timestamp>` versions without consuming intent files or touching changelogs, matching `changeset version --snapshot` for CI preview publishing.

## Rationale and Alternatives

1. **Status quo: keep recommending changesets.** Zero implementation cost. But the pains in Motivation are structural, not bugs to wait out: global pre mode and name-keyed data cannot be fixed in changesets without changing its file format, which its ecosystem cannot absorb. pnpm's own monorepo already maintains a wrapper script re-implementing prerelease versioning after every changesets run — evidence that sophisticated workspaces outgrow the tool.

2. **Patch or fork changesets.** Analyzed for pnpm's own repo: the per-package pre-mode feature is implementable in a fork of `@changesets/assemble-release-plan` and friends, but the half-consumed-intent semantics have no clean answer within changesets' file lifecycle, and a fork of a release engine is a permanent maintenance liability for a narrow win. A `pnpm patch` of the engine pins exact versions and breaks on every upgrade.

3. **A new, incompatible intent format.** Designing from scratch would allow richer intents (per-package summaries in one file, machine-readable metadata). But the changesets format has years of ecosystem inertia — bots that comment on PRs missing changesets, the changesets GitHub Action, contributor muscle memory. Compatibility means every existing changesets repo is a potential adopter at the cost of deleting a devDependency, and the format can be extended additively later. The richer-format experiments can live behind the same reader.

4. **Adopt/bless an existing alternative (Nx release, release-please, semantic-release).** These couple versioning to a task runner, a forge, or commit-message conventions respectively. pnpm should not require any of the three, and none solves per-package prerelease lines either.

Native, format-compatible implementation is the only option that removes the second tool, exploits pnpm's own workspace knowledge, and fixes the structural limitations — while keeping the migration cost near zero.

## Implementation

All user-visible behavior lands in both CLI stacks of pnpm/pnpm (the TypeScript CLI and the Rust port), per that repository's parity rule; the two implementations share the file formats and can share test fixtures.

New code concentrates in a new `releasing/` workspace package (TypeScript) and a matching crate (Rust):

- **Intent reader**: parse `.changeset/*.md` frontmatter; validate package names against the workspace.
- **Release-plan assembler**: direct bumps, dependent propagation via the existing project graph (`@pnpm/workspace.pkgs-graph`), fixed/linked constraints, prerelease-line versioning. For calibration, changesets' equivalent (`@changesets/assemble-release-plan`) is ~660 lines; this is the algorithmic core.
- **Applier**: manifest updates through the existing format-preserving manifest reader/writer; internal range updates with native `workspace:`/catalog awareness; changelog composition; intent-file lifecycle including per-package consumption state.
- **CLI commands**: `pnpm change`, `pnpm change status`, `pnpm version -r` (or final names), `pnpm version pre enter/exit`, wired through the existing command infrastructure in `releasing/commands`, with config plumbed through `@pnpm/config`.

Existing machinery that needs no change: workspace discovery, the dependency graph, `pnpm publish -r` (topological order, provenance), git utilities, `exportable-manifest`.

**Correctness oracle**: because the input format matches changesets, differential tests can run both engines against the same fixture workspaces and diff the resulting trees (manifests and changelogs) for every feature changesets supports; only the native extensions (per-package prerelease lines) need standalone specification.

Affected repositories: `pnpm/pnpm` (both stacks, docs for new commands and `pnpm-workspace.yaml` keys), `pnpm/pnpm.io` (documentation), and potentially a migration codemod (`.changeset/config.json` → `versioning:` key).

Risks: versioning tools accrete edge cases (peer-dependency bump semantics, ignored-package dependents, `workspace:` range edge cases at publish time). Scoping v1 to changesets-parity-plus-prerelease-lines bounds this; the differential oracle keeps parity honest.

## Prior Art

- **changesets** — the intent-file model this RFC adopts; its global pre mode and name-keyed format are the structural limits addressed here.
- **Lerna** (`lerna version`) — fixed/independent modes; couples versioning to its own workspace model; now maintained by Nx. The maintained **lerna-lite** fork is the external tool that has chased pnpm-native semantics hardest — it added `workspace:` and `catalog:` protocol support and OIDC trusted publishing — and is conventional-commits-driven; that it had to reimplement pnpm's protocols to stay usable is itself an argument for the native approach.
- **Rush** (`rush change` / version policies) — the intent-file idea with JSON ceremony, and the original diff-derived CI enforcement (`rush change --verify`, adopted here). Its version policies prefigure several pieces of this proposal: `lockStepVersion` is fixed groups with a policy-declared current version and `nextBump` (Rush's way of running prerelease lines like `1.0.0-dev.6`), and `individualVersion`'s `lockedMajor` constraint is the ancestor of the `maxBump` cap adopted here for maintenance branches.
- **Yarn Berry** (`yarn version check` / deferred versions) — the closest prior art for a package manager owning versioning natively. Its diff-derived coverage check and explicit per-package declines are adopted here (`pnpm change check`); its `changesetIgnorePatterns` maps onto pnpm's existing `changed-files-ignore-pattern` setting. Its deferred files record bump strategies only — no prose — so Yarn has no changelog story; and it requires dependents of a released workspace to decide explicitly (bump or decline), where this proposal propagates mechanically. Instructively, Yarn's first implementation stored the pending bump in each manifest (`nextVersion`) and had to be rewritten into per-branch files ([yarnpkg/berry#682](https://github.com/yarnpkg/berry/pull/682)) because manifest-resident intents caused constant merge conflicts — the same lesson as Bit's `.bitmap`.
- **Bit** (`bit tag --soft` / snaps) — a third independent convergence on recorded-intent versioning: a soft tag records the pending bump and changelog message for CI to apply later with `bit tag --persist`. Two lessons: it stores intents in the single mutable `.bitmap` state file, so concurrent PRs conflict where per-change files don't — validating the file-per-intent format; and it auto-tags dependents by default (opt-out via `--skip-auto-tag`), a data point for this proposal's mechanical propagation over Yarn's explicit dependent decisions. Its content-addressed snaps are the maximalist form of snapshot releases; its per-component prose is captured at tag time rather than change time, which forfeits the review-once property.
- **Nx release** — release groups are prior art for fixed/linked semantics; requires Nx.
- **cargo-release / release-plz** — single-ecosystem release managers demonstrating changelog-from-intent and workspace propagation in Rust.

## Unresolved Questions and Bikeshedding

- **Command naming.** `pnpm change` vs `pnpm changeset` (familiar, but implies the third-party tool) for recording; `pnpm version -r` vs a new verb (`pnpm bump` reads well but any new top-level command shadows same-named `package.json` scripts — a real compat hazard, since e.g. pnpm/pnpm itself has a repo script named `bump`).
- **Config key.** `versioning:` vs `release:` vs `changes:` in `pnpm-workspace.yaml`.
- **Changelog pluggability.** changesets supports custom changelog generators (`@changesets/changelog-github`); should pnpm resolve a changelog-writer package via config dependencies, or start with built-in `plain`/`github` formats only?
- **Should `.changeset/config.json` be read for migration** (warn-and-translate) or is a codemod enough?
- **Git integration scope.** Should `pnpm version -r` create the release commit/tags itself (and what tag scheme for multi-package releases — `pkg@1.2.3` per package, one tag per run, or configurable), or stay filesystem-only and leave git to CI, as changesets does?
- **GitHub Action / bot story.** Does pnpm ship a first-party action equivalent to `changesets/action` (open a release PR, publish on merge), or document recipes only?
- **Name for the intent directory.** Keep `.changeset/` for compatibility, or also read a pnpm-native location (`.pnpm-changes/`) with `.changeset/` as a fallback?
- **Interactive UX.** A checkbox TUI (changesets-style), or the `git rebase -i`-style editor buffer listing the touched packages with proposed next versions, as originally sketched in [pnpm/pnpm#2225](https://github.com/pnpm/pnpm/issues/2225)?
- **Conventional-commits bridge.** Requested repeatedly in [pnpm/pnpm#2225](https://github.com/pnpm/pnpm/issues/2225): should `pnpm change --from-commits` draft intent files from conventional commits since the last release — letting commit-message-driven teams generate intents while keeping the reviewable files as the source of truth?
- **Atomic multi-package publish.** Should the publish step adopt the two-phase strategy discussed in [pnpm/pnpm#1098](https://github.com/pnpm/pnpm/issues/1098) (Lerna's approach): publish every package under a temporary dist-tag and flip the real dist-tag only once all packages have landed, so a partially failed release is never visible under `latest`?
- **Coverage-check scope.** Yarn's check extends transitively: dependents of a released workspace must also declare a bump or a decline, surfacing semantic questions ("core changed behavior — does the CLI need a minor too?") that mechanical range-based propagation answers silently. This proposal checks only directly-touched packages and auto-propagates to dependents; is the Yarn-style explicit-dependent-decision mode worth offering behind a flag?
- **Registry changelog mode: graduation base.** When a package graduates off a prerelease line (`2.1.0` after `2.1.0-alpha.N`), should its changelog chain from the last prerelease (keeping the alpha sections in the history) or from the last stable version (replacing them with the aggregated stable section)?
