# Bit version control for pnpm workspaces

## Summary

This RFC proposes that a pnpm workspace can use [Bit](https://github.com/teambit/bit) as its only version-control system, with no Git repository or Git executable. Every pnpm workspace project is treated as a Bit component. Files outside those projects belong to a root component. A Bit snap operation is the workspace commit: all changed components receive versions carrying one shared batch ID and, on a lane, that batch also identifies the lane-history entry. During snap, Bit derives each component's resolved dependency graph from the pnpm lockfile. Users choose their development tools and configure the workspace for the components they import. Bit versions that configuration as files without interpreting it as a compatibility contract. A Bit lane is a branch and a Bit scope is the remote. As in Bit today, the lane stores component overrides on a main/fork baseline; the batch does not duplicate every unchanged component into a frozen Git-style tree.

Bit supplies a pnpm-workspace adapter, while pnpm remains the package manager rather than becoming a VCS frontend. Portable component-to-component dependencies use `catalog:` in package manifests; each destination workspace binds them to `workspace:*` when the dependency component is present and to the exact snapped version when it is absent. The proposal does not introduce a second blob/tree/commit database beneath Bit, translate every snap into a Git-shaped root-tree commit, or require Git as an interoperability layer. Existing Bit component versions, snap batches, lanes, lane histories, import/export, checkout, and merge are the foundation. The new work makes those facilities complete for a Git-free workspace by covering root files, removing Git from workspace-state persistence and restoration, and teaching ordinary Bit operations how to maintain pnpm workspace structure.

## Motivation

Bit already versions a workspace as components. A normal `bit snap` discovers the changed components, calculates their dependency graphs from the workspace lockfile, creates new component versions, updates dependencies between them, assigns the versions one shared batch ID, and persists the resulting objects. When the workspace is on a lane, it also updates that lane and writes a lane-history entry keyed by the batch ID. It does not need a repository-wide file tree to make that operation coherent: workspace state is composed from component heads, and every component version names its own files, parents, resolved dependencies, env, and aspect configuration.

pnpm already defines the component boundaries needed for this model. Each project matched by `pnpm-workspace.yaml` has a root, a manifest, a package identity where one is declared, and dependency edges to other projects. Treating those projects as Bit components preserves more useful structure than flattening the workspace into an undifferentiated Git tree. Package history is component history. A package-selective checkout is a component checkout. A lane overlays changed project heads on its main/fork baseline. A change affecting several projects is one snap batch.

Bit normally coexists with Git, however, and that division leaves several repository concerns to Git:

- root files such as `pnpm-workspace.yaml`, `pnpm-lock.yaml`, shared authoring configuration, CI configuration, documentation, and policy files may not belong to a component;
- `.bitmap` and workspace configuration are commonly restored by checking out Git before Bit loads the component heads they refer to;
- batches on main are present on their component versions but are not yet exposed as one workspace log;
- cloning a Git repository supplies the initial workspace structure before `bit import`, while a Bit-only clone must be able to reconstruct the entire directory from scope objects;
- Bit does not currently adopt all projects in an otherwise ordinary pnpm workspace automatically.

Those are integration and workspace-coverage gaps, not evidence that Bit lacks a workspace commit. The proposal fills them without introducing a competing source of truth.

Native repository mode must not reduce a scope to a monorepo that can only be cloned in full. A developer can initialize an unrelated workspace and import a selected component set. The destination keeps its own root configuration, and the user is responsible for making it work with those components. Successful import means that the source and dependency metadata have been materialized; it does not guarantee that the components build, lint, test, or run together. This selective workflow is a conformance requirement alongside restoring the canonical workspace's tracked files.

The intended result is a workflow such as:

```shell
bit init --standalone --external-package-manager --default-scope my-org.my-repository
bit pnpm sync
pnpm install
bit import my-org.my-repository/parser
bit status
bit snap -m "update the parser"
bit lane create feature
bit export
```

`bit pnpm sync` is the adoption boundary: it discovers pnpm projects, assigns durable component identities, creates the root component, and normalizes portable workspace dependencies. After that, status, snap, lane, import, export, checkout, and merge are the ordinary Bit commands. pnpm installs dependencies and implements generic catalog semantics; it does not shell out to Bit or mirror Bit commands. Git is optional migration tooling, not part of native operation.

The same model can cover a repository that is not a pnpm workspace. With no discovered projects, the root component owns the complete tracked tree. Splitting the tree into package components is therefore a semantic enhancement rather than a prerequisite for version control.

## Detailed Explanation

### One model, not a backend translation

The central mapping is:

| pnpm/VCS concept | Bit representation |
| --- | --- |
| Workspace project | Component |
| Project revision | Component version/snap |
| Workspace commit | Snap batch of changed components |
| Commit ID | Snap batch ID |
| Branch | Lane |
| Main branch | Main component heads |
| History | Component histories grouped by batch; lane history on lanes |
| Remote repository | Scope |
| Clone/fetch | Lane and component import |
| Push | Lane and component export |
| Checkout | Component checkout/lane switch |
| Merge | Lane/component merge |

One invocation of Bit's version maker creates one batch ID and passes it into every component version made by that operation. On a lane, the same batch ID is used as the lane-history key. The history entry records that lane's component heads and deletions after the operation. Objects are persisted after component versions and lane history have been prepared. This is the workspace operation boundary exposed by the Bit CLI.

A lane is intentionally an overlay of component heads, not a Git branch containing a frozen copy of every component in the workspace. Components not present on the lane resolve through main or the lane it was forked from according to existing Bit semantics. Lane history therefore records lane-local state, not a second repository-wide tree. Native pnpm-workspace support preserves this model rather than quietly changing lanes into Git branches.

A separate immutable object containing another copy of all component heads is not required. The lane records its current heads, lane history records its operations, and component versions record their individual parent DAGs. A batch ID is an operation identity rather than a content digest. Source objects remain content-addressed, while component version refs retain Bit's existing immutable identities.

The proposal does make the batch a supported public concept rather than an implementation detail:

- every completed non-soft snap returns its batch ID in structured output;
- status, log, diff, reset, import, and export can accept or report a batch ID where a workspace-level operation is meaningful;
- a batch can be resolved efficiently without scanning every version object;
- a workspace log groups the component versions created by the same batch;
- a failed operation never exposes a history entry as a completed workspace revision;
- reset and garbage collection retain or remove all versions belonging to the batch consistently.

This can be implemented by indexing existing version and history data. It does not require a new root-tree commit graph.

### Project discovery and component identity

In native pnpm mode, the workspace manifest is the default component-boundary declaration. Bit's pnpm adapter reads it and resolves projects with pnpm-compatible package-pattern and exclusion semantics:

```text
ProjectInventory
  workspace root
  workspace configuration identity
  projects:
    project root
    manifest path
    package name, if declared
    persistent component identity
  ignored paths
```

`bit pnpm sync` registers or updates these components. It is required when project roots are added, moved, or deleted; ordinary status and snap then operate on the resulting component map. Users do not have to run `bit add` for every package. Manual Bit components may coexist, but a path may have only one owner. A future on-load hook may make synchronization automatic, but the explicit command keeps adoption and identity changes reviewable.

Package names are useful defaults for component identity but are not sufficient by themselves: packages may be unnamed, two private projects may temporarily declare the same name, and a package may be renamed. A version-free identity map persists the selected Bit component ID for every project root. It is workspace source configuration and can itself be tracked by the root component. Component versions never appear in that map, avoiding a self-reference when a snap updates component heads.

The initial format is a `vcs` block in `pnpm-workspace.yaml`:

```yaml
vcs:
  provider: bit
  schemaVersion: 1
  rootComponent: acme.workspace/root
  components:
    packages/app:
      componentId: acme.workspace/app
      manifestFile: package.json
```

IDs in this block are scoped and version-free. Bit writes the block after assigning identities, reads it when `.bitmap` is absent, and rejects unsupported providers or schema versions. pnpm treats the block as opaque workspace metadata. The root component stores the same topology in normalized aspect data so Bit can recognize and safely materialize the root before `pnpm-workspace.yaml` exists locally. The source YAML is authoritative; the model copy is a bootstrap projection of the same version-free data, not another checkout-state database.

For a new workspace, package names are normalized into the initial component-name candidates; unnamed packages fall back to their project paths and collisions receive path-derived suffixes. Once written, the durable ID wins over later package-name changes. Moving a project requires moving its identity-map entry as part of the same change, making the move an explicit component-identity operation rather than an accidental deletion and creation.

Components used for VCS tracking are allowed to be source-only. A valid workspace project need not have a Bit main file, build environment, publishable package name, or runnable entry point. Bit currently uses those concepts for development and packaging; the VCS layer must not invent them merely to store files. Existing Bit components retain their normal environment and build behavior.

### Workspace configuration is user-owned

Bit's VCS role is to store, transfer, and restore source and configuration. As with Git, choosing development tools and making the checked-out source work is the user's responsibility. Adopting a pnpm workspace must not require translating its build into Bit env APIs or declaring a new toolchain contract.

Root files such as `package.json`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, `tsconfig.json`, and CI configuration are ordinary versioned inputs owned by the root component. Cloning the canonical workspace restores them. Importing selected components into another workspace preserves the destination's configuration; the user supplies or adapts any shared configuration those components need. Bit does not automatically import their source root components, merge build configuration, or migrate configuration schemas. The adapter still updates project mappings and catalog bindings as described below; those structural edits do not configure the imported components' build or test setup.

This RFC introduces no workspace profiles, capability slots, component tool requirements, compatibility admission checks, or applied-profile fingerprints. A difference in runtime, compiler, bundler, or test runner does not itself prevent a VCS import. Existing package metadata such as `engines` remains ordinary package metadata, interpreted by the tools that already consume it.

Users may keep conventional `package.json` scripts, use existing Bit envs, or choose another development setup. A script adapter or an isolated workspace build may be useful separately, but neither is required for version control. Existing Bit env/aspect behavior remains available without becoming a prerequisite for tracking source-only components.

Configuration edits appear in the status and history of the component that owns the files. Changing a root build configuration does not by itself mark every project modified or prove compatibility. Dependency graphs and catalog bindings remain structured inputs to component change detection because they identify versioned dependencies, not because they describe the development environment.

### Portable workspace dependencies through catalogs

An ordinary pnpm workspace commonly expresses an internal edge as `"@acme/math": "workspace:*"`. That is correct while both projects are present, but it is not a portable component declaration: importing only the consumer into another workspace leaves no local package for the `workspace:` protocol to select.

Native VCS mode uses pnpm catalogs as the indirection boundary between the component's portable manifest and the destination workspace's current composition. The component keeps a catalog reference:

```json
{
  "name": "@acme/app",
  "dependencies": {
    "@acme/math": "catalog:"
  }
}
```

while the canonical monorepo binds that entry locally:

```yaml
catalog:
  '@acme/math': workspace:*
```

Catalog entries therefore allow the `workspace:` protocol. Resolution first dereferences `catalog:` and then applies normal workspace-protocol resolution. Publication performs the same two transformations in that order, so the exported package manifest receives a registry-installable version rather than either protocol.

`bit pnpm sync` migrates existing intra-workspace `workspace:` declarations in `dependencies`, `devDependencies`, `optionalDependencies`, and `peerDependencies` to the default `catalog:` and copies their original specifiers into `pnpm-workspace.yaml`. One catalog key has one workspace-wide meaning. If projects use inconsistent `workspace:` specifiers for the same package, synchronization rejects the migration and asks the user to choose one binding instead of silently changing dependency semantics.

During snap, Bit already derives the resolved dependency graph from `pnpm-lock.yaml` and stores the exact component dependency in the component model. The source `package.json` can consequently retain `catalog:` without losing the dependency component ID or version. This model data is the authority used when composing another workspace; the source workspace's catalog is not copied wholesale.

Selective import is maintained by Bit itself:

1. `bit import` materializes the requested components and reads each `catalog:` dependency from the component's resolved dependency model. Legacy imported `workspace:` declarations are normalized to `catalog:` at this boundary.
2. Bit adds the imported roots and durable component identities to `pnpm-workspace.yaml`.
3. A catalog dependency whose project is present is bound to `workspace:*`; one whose project is absent is bound to the exact version recorded in the imported component model. A snap hash is represented by its publish-compatible `0.0.0-<hash>` version.
4. If the dependency component is imported later, Bit changes its existing exact catalog binding to `workspace:*`, so the same consumer manifest now resolves to the local project. The user then runs ordinary `pnpm install`, or an integration may request installation explicitly.

For example, importing only `@acme/app` produces:

```yaml
packages:
  - bit-components/acme/app
catalog:
  '@acme/math': 0.0.0-3f10c2a7
```

Importing `@acme/math` later adds its component root and changes only the catalog entry:

```yaml
catalog:
  '@acme/math': workspace:*
```

Bit calculates these edits from component objects already fetched by the normal import operation. Existing Bit scopes and bit.cloud need no server-side protocol or storage change: the dependency graph required for reconciliation is already part of each snapped component model. pnpm only needs generic support for resolving and publishing `workspace:` values reached through catalogs.

### Exclusive file ownership and the root component

The proposed [Bit root-component capability (teambit/bit#10698)](https://github.com/teambit/bit/pull/10698) is the foundation for root-file tracking. It allows a component with `rootDir: "."` to own workspace files while excluding nested component roots and Bit/Git metadata. The root is rescanned so newly added files can be tracked without maintaining a frozen file list. The PR is open at the time of this revision; this RFC depends on that capability becoming available, with clone and workspace restoration handled by the integration described below.

Every tracked path belongs to exactly one component version. Ownership is calculated as follows:

1. Bit/VCS metadata and generated dependency directories are excluded.
2. A path below a non-root pnpm project belongs to that project.
3. Only the root component may contain other component roots. Nested non-root projects are rejected with an actionable ownership error; they must be reorganized or tracked within one component.
4. Existing explicitly declared Bit ownership takes precedence only when it does not conflict with the pnpm inventory.
5. Every remaining tracked path belongs to the root component.

The root component typically contains:

```text
pnpm-workspace.yaml
pnpm-lock.yaml
package.json                 # also when the root is a workspace project
workspace.jsonc
shared authoring configuration
.github/**
docs and repository policy
the version-free component identity map in `pnpm-workspace.yaml`
other non-ignored, unclaimed files
```

If the workspace root is itself a pnpm project, it is also the root component; it owns root-level files but excludes nested project roots. Otherwise Bit creates a source-only root component and persists its stable ID in the identity map. In both cases it uses the same root-component capability, not a separate root-file storage mechanism.

This component removes the main reason Git is currently needed for completeness. A CI-only change may change only the root component. A lockfile change also changes every project component whose calculated dependency graph changed. A checkout or clone materializes the root component together with the package components and therefore reconstructs the whole tracked workspace.

Generated paths such as `node_modules`, the pnpm store, capsules, local VCS metadata, and ignored build output are excluded. Existing ignore files can be honored without requiring Git to be installed. The final native ignore filename and compatibility with `.gitignore` are product decisions, but the Bit adapter's project discovery and file tracker must apply one consistent ownership and exclusion model.

### Workspace metadata without Git

`.bitmap` currently combines durable component mapping with checkout state such as component versions. Tracking that exact file inside a component is unsuitable: snapping changes the versions written to `.bitmap`, which would make the root component modified immediately after its own snap.

Native VCS mode separates two kinds of state:

- **Durable, versioned workspace authoring configuration:** project root to stable component ID, default scope, variants/aspects, shared tool inputs, ignore rules, and other information needed to reproduce the canonical workspace. This contains no current component versions and belongs to the root component.
- **Derived local checkout state:** current lane, checked-out component heads, generated links, caches, and compatibility data needed by existing Bit internals. This is reconstructed from the selected lane/main heads and the durable identity map.

`.bitmap` may remain the local compatibility representation initially, but it is regenerated after clone, switch, checkout, and reset and is not the authority that a remote clone must preserve. In the longer term Bit may split its version-free mapping from checkout state explicitly.

`workspace.jsonc`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, and authored root configuration remain real tracked files in the root component. They are restored before the canonical workspace is fully loaded. The root component's model carries a normalized copy of the version-free topology. This small projection lets a clean client validate that it fetched the requested root, materialize it at `.`, read the authoritative YAML, and only then import every mapped component at its declared project root. Bit generates `.bitmap` entries with the fetched heads during those imports.

A Bit-native clone command will implement this bootstrap in a temporary sibling directory. It initializes standalone Bit metadata, imports the validated root component at `.`, imports the mapped main heads, runs or requests `pnpm install`, and renames the completed staging directory into place. A lane form instead resolves the root against the remote lane first, activates that lane from the validated one-component bootstrap state, and then imports every mapped component through normal lane resolution. Components carried by the lane use its heads and the rest fall back to main. The proposed bootstrap permits initial cross-lane root materialization only in a restricted root-bootstrap state: exactly one component is written at `.`, its model must contain a valid root topology, and any staged-component switch exception is limited to that validated root. Normal tracking then uses root-directory rescanning with nested component roots excluded. An existing destination is never overwritten and unsafe component roots are rejected. Recovery of an interrupted non-staged materialization remains follow-up work. A newly composed workspace uses its own root component and user-selected configuration; it does not automatically import and merge the source workspaces' root configurations.

### Snap batches as workspace commits

`bit snap` is the workspace commit. Before selecting the final component set, Bit calculates each component's resolved dependency graph from the lockfile. A changed dependency graph or used catalog binding is a component change. Without an explicit filter, the operation includes all components with source or dependency changes, matching the desired workspace-commit behavior. Bit may additionally auto-snap affected dependents according to its existing dependency rules. Every resulting version carries the operation's batch ID.

The workspace log displays one entry per batch rather than one row per changed component:

```text
4f8c2e1  update the parser
  parser       2a9e… → 91bc…
  parser-tests d81a… → c3f0…
  workspace    70ed… → 88a1…
```

The batch entry resolves to the component versions created by that operation. Components not in the batch are unchanged and continue to resolve through the current lane/main model. This permits a grouped workspace log, batch diff, and batch reset without inventing a second repository snapshot.

An explicitly filtered snap is still a valid workspace operation: only selected, affected, and auto-snapped components receive new versions. It cannot snap a new root lockfile or catalog binding while deliberately omitting a component whose newly calculated portable model depends on that input. There is no file-level staging area in the first proposal. Bit's atomic unit is the component. File- or hunk-level partial snapshots could be considered later but are not required for workflows that intentionally commit at project/component granularity.

### Main and lanes

Non-default lanes already record lane-history entries after snaps. On main, the versions still carry their batch IDs, so a workspace log can group component histories by batch without first adding a default-lane object. An index may make that lookup efficient, but it does not become another history authority. Component version parents remain the authority for divergence and merge.

A lane remains an overlay of component heads with its existing fork and component ancestry semantics. Switching lanes writes the lane's components, resolves the remaining workspace components through Bit's existing fallback rules, and regenerates derived checkout metadata. Merging lanes continues to merge components independently, including the root component when it changed. The workspace log groups any component versions created by the merge operation and includes the existing lane-history event.

Lanes can contain different project sets. A component absent from the lane continues to resolve through the lane's main/fork baseline; a component explicitly deleted on the lane is removed. Creating or deleting a pnpm project records the corresponding component addition or deletion, and switching applies it with the same safety checks Bit checkout uses for modified or untracked files.

### Scope as the remote repository

The first native mode assigns all components owned by one pnpm workspace—including the root component—to one repository scope. External Bit components may remain dependencies from other scopes. Keeping owned heads and lane history in one scope gives clone, fetch, push, permissions, and divergence checking one collaboration boundary.

The existing Bit object protocol transfers component versions, file objects, lanes, and lane histories. Native pnpm VCS mode builds on that protocol rather than adding a Git object server. The required high-level operations are:

- **clone:** initialize local Bit metadata, fetch and validate the root component, read its durable topology, materialize the selected lane/main component heads, then regenerate checkout metadata;
- **compose/import:** import selected components and their reachable dependency and existing env/aspect objects, reconcile project identities and catalog bindings, then let pnpm generate the destination lockfile; users configure the workspace to develop those components;
- **fetch:** import new component versions, heads, and histories without modifying the working tree;
- **pull:** fetch and then fast-forward or invoke the existing lane/component merge policy;
- **push:** export every object required by the selected batch/lane and advance the remote lane only if its expected head state has not changed;
- **status:** compare local component heads and files with the selected local and remote lane states.

Bit's existing divergence and export logic supplies most of this behavior. The VCS contract should tighten the externally visible transaction boundary: a rejected or interrupted push must not publish a lane state that references only part of a batch, and a concurrent push must fail or merge rather than silently overwrite heads.

Repository discovery and authorization can use scope identity initially. A separate forge, Git host, or Git credential is not required. Code review, issues, and a pull-request UI are distinct hosting features; they are not prerequisites for source transfer.

### Bit-native pnpm workspace integration

Bit owns the version-control command surface and the pnpm-workspace adapter. There is no `pnpm vcs` subprocess protocol and no duplicate status, commit, lane, clone, import, or export command implemented by pnpm. This avoids a fragile wrapper layer and keeps operation semantics, diagnostics, ownership checks, and recovery in the system that owns the component model.

The adapter has two responsibilities:

- `bit pnpm sync` adopts a raw pnpm workspace, discovers project boundaries, assigns durable identities, creates or refreshes the root component, and migrates portable dependency declarations;
- ordinary Bit lifecycle hooks keep dependency graphs, used catalog bindings, import reconciliation, topology, and derived checkout state current.

The public workflow uses existing Bit terminology:

```shell
bit status
bit snap -m "update the parser"
bit lane create feature
bit switch feature
bit export
```

pnpm remains responsible for installation, lockfile generation, and catalog/workspace-protocol resolution. Its changes are generic and independently useful: catalogs may contain `workspace:` values, exportable manifests dereference catalogs before converting workspace ranges, and opaque workspace-owner metadata is tolerated. pnpm neither knows how to snap a component nor locates or invokes a Bit executable.

The [draft pnpm CI RFC](https://github.com/pnpm/rfcs/pull/25) treats version control as a read-only source provider. This native Bit mode implements that provider directly: changed projects come from component status/history, project input identity comes from component versions, and agents materialize a lane/batch through Bit. The CI engine remains read-only and never snaps or exports merely by running a pipeline.

### Operation without Git

Git-free operation is a conformance requirement, not a later optimization. Tests run with no `git` executable on `PATH` and no `.git` directory. A clean machine must be able to:

1. clone a main or lane state from a Bit scope;
2. restore root configuration and every project;
3. create a different workspace and import selected components without a development-tool compatibility gate;
4. fetch their dependency graphs and any existing env/aspect objects;
5. synthesize exact or `workspace:*` catalog bindings for the selected component set, generate a destination lockfile, and preserve user-owned build configuration;
6. install the workspace with pnpm;
7. detect source, root-file, dependency-graph, and used catalog-binding changes;
8. commit them as one snap batch;
9. create, switch, and merge lanes;
10. inspect component/batch history and reset a batch;
11. push and pull through the Bit scope.

No native command may silently shell out to Git for ignore evaluation, diff, merge, author identity, editor invocation, credential storage, or shallow-history repair. Git interoperability, when implemented, is an explicit adapter.

### Git coexistence and migration

Repositories may continue to use Git and Bit together exactly as they do today. Native mode does not require deleting `.git`. It merely stops treating Git as required durable storage for the Bit workspace.

Importing existing Git commit history into snap batches is optional follow-up work. A bridge could replay each Git commit, snap the components changed by it, and retain a Git commit to Bit batch mapping. That is useful for migration but does not define the native object model.

When both systems are present, neither advances the other's branch implicitly. Explicit synchronization reports which Git commits and Bit batches correspond. Pretending two independent ref-update systems are one atomic transaction would create silent divergence.

### Security and recovery

Removing Git makes Bit responsible for every tracked path in the working tree, including root files. Existing component checkout and import protections must therefore be applied to the union of all components:

- component paths are relative, normalized, and contained by the workspace root;
- no two component heads may own the same path;
- symlinks are not followed while materializing another path;
- checkout refuses to overwrite modified or untracked files without explicit destructive authorization;
- a component ID, root, or manifest supplied by workspace source cannot escape the workspace;
- fetched objects are integrity-checked and validated before checkout;
- interrupted clone/switch/merge operations leave recoverable state and are never reported as clean;
- lane updates use expected remote state to prevent lost concurrent pushes;
- merely inspecting or checking out source never executes package scripts or Bit aspects from that source.

The root component receives the same validation as every other component. It is not allowed to claim generated VCS metadata or nested project files.

## Rationale and Alternatives

### Add a generic blob/tree/commit kernel beneath Bit

An earlier version of this RFC proposed a Git-like repository layer whose root tree was canonical and whose component set was a projection. That model naturally represents arbitrary files and gives every workspace operation one content-addressed commit object.

It also duplicates information Bit already stores: file contents would be reached through both repository trees and component versions, component parent DAGs would coexist with a second repository DAG, lane state would be projected from another branch representation, and every operation would need rules for which graph is authoritative. For pnpm workspaces, exclusive component ownership plus a root component already reconstructs the entire tree. The snap batch is already the multi-component operation boundary. This RFC therefore extends the existing model rather than placing another VCS underneath it.

The generic model should be reconsidered only if exclusive file ownership or component-granular history proves fundamentally incompatible with required workflows.

### Keep Git as the repository VCS and use Bit only for components

This is the current, mature workflow and remains supported. It has maximal ecosystem interoperability and requires less new bootstrap and root-file behavior. It also means a workspace cannot use Bit alone: Git remains the source of root configuration and mapping, CI needs a Git checkout before Bit import, and repository-level collaboration is split across two systems. It does not meet the goal of this RFC.

### Treat the entire workspace as one component

One component rooted at the workspace would make Bit a general file-tree VCS with almost no mapping work. It discards Bit's primary advantage: independent package histories, dependency-aware snaps, package-selective import, and component-level lanes and merge. The root component is only the fallback owner; discovered projects remain separate components.

### Implement a separate VCS inside pnpm

pnpm could implement commits, branches, storage, and remotes independently and optionally publish package projections to Bit. This creates two implementations and makes Bit an export target rather than the version-control system. It also duplicates scope hosting, component history, checkout, merge, and dependency-aware behavior. The proposal instead keeps one authority and adds a Bit-owned pnpm-workspace adapter.

### Adopt another Git-compatible frontend

Tools such as Jujutsu provide improved workflows while retaining Git storage and hosting. They are useful prior art for history editing and coexistence but do not make pnpm projects or Bit components the native revision units, and Git remains part of the stack. This may be the better choice for users who prioritize universal Git tooling over Bit-native package history; it is not an implementation of Git-free Bit operation.

## Implementation

### Phase 1: make a Bit workspace complete without Git

- Introduce or formalize source-only components without required main files or build environments.
- Automatically construct components from a pnpm project inventory.
- Build on teambit/bit#10698 for a root component at `.` with dynamic file discovery; exclude nested components and reject nesting between non-root components.
- Split durable, version-free workspace identity from derived `.bitmap` checkout state.
- Bootstrap root metadata before loading and materializing the rest of a cloned workspace.
- Add Git-free integration tests with root-file-only and multi-project changes.
- Preserve existing build configuration as tracked files and allow source tracking and import without configuring a Bit build environment.

### Phase 2: make component dependencies portable

- Allow `workspace:` values in pnpm catalogs and resolve them after catalog dereferencing.
- Convert catalog references before workspace references when producing a publishable package manifest.
- Migrate consistent intra-workspace `workspace:` declarations to `catalog:` during `bit pnpm sync`.
- Derive import reconciliation from each imported component's manifest and lockfile-derived dependency model inside Bit.
- Have `bit import` add component roots and durable identities to the destination pnpm workspace and bind catalog entries to exact versions or `workspace:*` according to which projects are present.
- Rebind an exact entry to `workspace:*` when its component is imported later.
- Keep dependency installation in pnpm while root workspace-file ownership and composition remain in Bit.
- Test consumer-only import, subsequent dependency import, named catalogs, conflicting bindings, snap versions, and ordinary semver versions against an unchanged Bit remote.

### Phase 3: track dependency changes consistently

- Formalize the lockfile-derived dependency graph and used catalog bindings as component change inputs during status and snap.
- Auto-include affected components or reject a filtered snap when a dependency change would otherwise leave inconsistent component models.
- Test lockfile-only and catalog-only changes, including filtered snaps.
- Test selective import into a workspace with different build configuration, preserving that configuration and allowing the user to adapt it after import.
- Verify that root build-configuration edits change the root component without automatically refreshing unrelated component models.

### Phase 4: expose the snap batch as the workspace operation

- Index batch IDs and expose them consistently in structured snap output.
- Build the main workspace log by grouping component histories by batch ID.
- Support workspace log, diff, and reset by batch ID.
- Ensure deletion, rename, merge, tag, reset, import, and export preserve batch grouping.
- Document and test that lane history contains lane-local heads while other components resolve through the lane baseline.
- Verify that persistence and recovery never expose a partial batch as a completed revision.

### Phase 5: native clone and collaboration

- Add clone/compose/fetch/pull/push operations expressed entirely through scopes, lanes, histories, and component objects.
- Regenerate derived checkout metadata after clone, switch, checkout, and reset.
- Fetch reachable dependency and existing env/aspect objects when composing a workspace from selected components.
- Preserve the destination's build configuration when importing selected components; leave any required adjustments to the user.
- Require expected remote state for lane updates and test concurrent pushes.
- Restore the root component before full workspace initialization.
- Validate operation without Git installed on Linux, macOS, and Windows.

### Phase 6: Bit-native product surface

- Stabilize `bit pnpm sync` as the explicit raw-workspace adoption and topology-refresh command.
- Make `bit import` reconcile pnpm package patterns, durable identities, and catalogs automatically when the destination has a Bit-owned root component.
- Add Bit-native repository clone/materialization UX over the validated root-bootstrap path.
- Expose structured source/dependency status, batch history, diff, materialization, lane, and remote results from ordinary Bit commands.
- Keep pnpm changes limited to generic catalog resolution/publication and opaque workspace metadata support.
- Implement the native Bit source provider for pnpm CI without routing mutations through pnpm.

### Phase 7: optional Git bridge

- Define explicit coexistence status when `.git` is present.
- Prototype Git commit to Bit batch import while preserving authors, messages, and parent relationships where meaningful.
- Record immutable Git commit to Bit batch mappings.
- Evaluate whether a Git remote helper adds value once native scope collaboration is available.

This phase is not required to call the native workflow complete.

### Prototype validation

The current proof of concept covers raw-workspace adoption and the first end-to-end composition and repository-bootstrap slices in Bit. `bit pnpm sync` converts a regular pnpm workspace's local `workspace:` edges to catalogs, assigns every project a component identity, creates a root component, and permits ordinary `bit status` and `bit snap` to operate without Git. `bit import` imports only a consumer into a new empty pnpm workspace with an exact catalog binding and later rebinds that entry to `workspace:*` when the dependency component is imported, without editing the consumer's dependency declaration by hand.

The prototype also writes the version-free path-to-component map into `pnpm-workspace.yaml`, stores its normalized projection on the root component, and recovers project identities without `.bitmap`. Build-adapter experiments also exercised conventional scripts and isolated workspace reconstruction. Those experiments are separate from the VCS requirements in this revision; users remain responsible for their build setup. The root bootstrap must adopt the proposed dynamic root-component capability instead of relying on a fixed list of root files.

A previously prototyped Rust pnpm clone wrapper proved the root-first and lane-aware bootstrap algorithm against bit.cloud, including root and project overrides on a lane plus a project inherited from main. That wrapper has deliberately been removed: the validated algorithm belongs behind Bit-native clone UX. The path uses ordinary component and lane objects and therefore requires a compatible Bit client but no bit.cloud server change.

This prototype does not imply that all phases above are complete. In particular, adoption of the root-component capability, native pull/push UX, batch-level history operations, cross-platform conformance, and recovery work remain part of the RFC.

### Affected repositories

- **teambit/bit:** raw pnpm workspace discovery, source-only component support, root component ownership and topology projection, durable identity, validated root bootstrapping, dependency/catalog change detection, selective-import reconciliation, batch indexing and operations, Git-free clone/composition/collaboration, and recovery/security tests. Existing Bit scope servers do not require a catalog-bridge or root-bootstrap change.
- **pnpm/pnpm:** generic `workspace:` catalog values, catalog-before-workspace publication conversion, and tolerance for opaque durable workspace-owner metadata. pnpm contains no Bit client and no VCS command surface.
- **pnpm/rfcs:** follow-up RFCs if final command UX, identity naming, or multi-scope collaboration need independent ratification.

The durable manifest and Bit component metadata are schema-versioned. There is no pnpm-to-Bit executable protocol to negotiate.

## Prior Art

**[Bit snaps](https://bit.dev/reference/components/snaps)** are the direct foundation: component versions with parent history, lockfile-derived dependency graphs, env/aspect configuration, dependency-aware multi-component snapping, and immutable file objects. This RFC promotes the shared batch to a workspace operation while retaining the existing dependency graph.

**[Bit lanes](https://bit.dev/reference/lanes/merge-lanes)** provide switching, independent component heads, main/fork fallback, merge, and remote collaboration. Native pnpm mode keeps this component-overlay model instead of translating it into frozen Git branch trees.

**[Git](https://git-scm.com/)** remains the compatibility baseline for user expectations around status, safe checkout, history, branches, concurrent push rejection, and recovery. Its single tree/commit representation is not copied where Bit's component graph already supplies the invariant.

**[Jujutsu](https://jj-vcs.github.io/jj/latest/)** demonstrates that a VCS frontend and its underlying storage model need not share Git's command semantics, and that coexistence must make operation ownership explicit.

**[pnpm catalogs](https://pnpm.io/catalogs)** already separate the dependency reference stored in project manifests from the version policy selected by the workspace. This RFC extends catalog values to `workspace:` so the same indirection can select either a local project or an exact remotely installable component version.

**Monorepo tools** derive affected projects and project histories from Git paths plus a workspace graph. The Bit-native model stores project history directly and uses pnpm's declared boundaries, avoiding path attribution as the primary semantic model.

## Unresolved Questions and Bikeshedding

- **Adapter command.** Should `bit pnpm sync` remain an explicit topology-changing operation, run automatically before selected Bit commands, or graduate into generic workspace discovery with pnpm as one boundary provider?
- **Root component identity.** The prototype derives an initial `<root-package-name>-workspace` name and then persists the assigned ID. Is that naming policy suitable for the permanent UX, and how should the root be displayed and protected from collision with a user component?
- **Project identity UX.** The durable map settles identity after initialization, but which explicit command should move, rename, or reassign an entry and preview its effect on history?
- **Main workspace log.** Is grouping component histories by batch ID sufficient, or is a derived persistent index needed for large scopes?
- **Batch ancestry.** Component version parents carry merge ancestry, while a batch groups an operation. Does workspace-level UI need explicit parent batch IDs, or would that incorrectly imply a second repository DAG?
- **Batch ID format.** Is the existing UUID sufficient as a public workspace revision ID, should it receive a short display form, or should future batches derive an ID from their resulting heads and metadata?
- **Owned scopes.** Must all workspace projects use one repository scope in the first release, and how should existing multi-scope Bit workspaces behave?
- **Lane overlay semantics.** How prominently must Bit explain that a lane contains changed component heads and resolves other components through its baseline rather than freezing every workspace project as Git does?
- **Change invalidation.** How does Bit efficiently determine which components' dependency graphs or used catalog bindings changed after editing a lockfile or catalog?
- **Catalog conflict UX.** Native mode deliberately requires one binding per package in a catalog. Should pnpm offer a guided normalization command when an existing workspace uses different `workspace:` ranges for the same internal dependency, or is an actionable initialization error sufficient?
- **Generated-file ownership.** How are authored root inputs distinguished from generated facades so status neither loses user changes nor snaps materialized output as a second source of truth?
- **Partial commits.** Is component-level selection sufficient, or must Bit eventually support file- or hunk-level staging inside one component?
- **Nested and overlapping projects.** The initial model allows nesting only beneath the root component. Should support for nested non-root projects be added later, and how are projects that consume sources outside their roots represented?
- **Ignored files.** Should native mode keep `.gitignore` as a compatible convention without Git, introduce `.bitignore`, or support both with a defined precedence?
- **Clone address.** What user-facing URL identifies a repository scope and lane distinctly from importing an ordinary Bit component?
- **Remote atomicity.** Which existing export paths need strengthening so one pushed batch cannot leave a visible partial lane state?
- **Component-only operations.** How should a snap/export performed directly on a component outside a complete workspace appear in workspace batch history after import?
- **History editing.** Squash, rebase, amend, and cherry-pick can be component-oriented or batch-oriented; their workspace semantics require a separate design.
- **Git migration.** How much original Git history and parent structure must be retained when one Git commit maps to several component versions?
- **Success criteria.** Before leaving experimental status, the feature needs cross-platform Git-free clone/compose/commit/branch/merge/push tests, consumer-only import followed by local dependency rebinding, named-catalog and catalog-conflict coverage, selective import across different development setups without overwriting destination build configuration, dynamic root-file discovery and ownership tests, crash recovery, concurrent remote updates, large-workspace performance targets, and a security review.

These questions refine the existing Bit model; none requires a second canonical repository tree unless the component-ownership premise itself is rejected.
