# pnpm as a version control system

## Summary

This RFC proposes that a pnpm workspace can use [Bit](https://github.com/teambit/bit) as its only version-control system, with no Git repository or Git executable. Every pnpm workspace project is treated as a Bit component. Files outside those projects belong to an implicit root component. A Bit snap operation is the workspace commit: all changed components receive versions carrying one shared batch ID and, on a lane, that batch also identifies the lane-history entry. During snap, Bit derives portable component state from workspace-wide authoring inputs: the pnpm lockfile produces each component's resolved dependency graph, while root configuration, workspace policy, and envs produce its effective configuration graph. A Bit lane is a branch and a Bit scope is the remote. As in Bit today, the lane stores component overrides on a main/fork baseline; the batch does not duplicate every unchanged component into a frozen Git-style tree.

pnpm supplies an optional VCS-oriented command surface and automatic project discovery, while Bit remains the version-control implementation. The proposal does not introduce a second blob/tree/commit database beneath Bit, translate every snap into a Git-shaped root-tree commit, or require Git as an interoperability layer. Existing Bit component versions, snap batches, lanes, lane histories, import/export, checkout, and merge are the foundation. The new work makes those facilities complete for a Git-free workspace by covering root files, removing Git from workspace-state persistence and restoration, exposing snap batches coherently in pnpm, and defining a stable integration boundary.

## Motivation

Bit already versions a workspace as components. A normal `bit snap` discovers the changed components, calculates their dependency graphs from the workspace lockfile, creates new component versions, updates dependencies between them, assigns the versions one shared batch ID, and persists the resulting objects. When the workspace is on a lane, it also updates that lane and writes a lane-history entry keyed by the batch ID. It does not need a repository-wide file tree to make that operation coherent: workspace state is composed from component heads, and every component version names its own files, parents, resolved dependencies, env, and aspect configuration.

pnpm already defines the component boundaries needed for this model. Each project matched by `pnpm-workspace.yaml` has a root, a manifest, a package identity where one is declared, and dependency edges to other projects. Treating those projects as Bit components preserves more useful structure than flattening the workspace into an undifferentiated Git tree. Package history is component history. A package-selective checkout is a component checkout. A lane overlays changed project heads on its main/fork baseline. A change affecting several projects is one snap batch.

Bit normally coexists with Git, however, and that division leaves several repository concerns to Git:

- root files such as `pnpm-workspace.yaml`, `pnpm-lock.yaml`, shared authoring configuration, CI configuration, documentation, and policy files may not belong to a component;
- `.bitmap` and workspace configuration are commonly restored by checking out Git before Bit loads the component heads they refer to;
- root configuration is convenient in a monorepo, but its effective component-level meaning is not yet captured and regenerated as systematically as the dependency graph derived from the lockfile;
- batches on main are present on their component versions but are not yet exposed as one pnpm-facing workspace log;
- cloning a Git repository supplies the initial workspace structure before `bit import`, while a Bit-only clone must be able to reconstruct the entire directory from scope objects;
- pnpm does not currently offer a VCS-oriented frontend or automatically register all workspace projects as components.

Those are integration and workspace-coverage gaps, not evidence that Bit lacks a workspace commit. The proposal fills them without introducing a competing source of truth.

Native repository mode must not reduce a scope to a monorepo that can only be cloned in full. A developer can also initialize an unrelated empty workspace and import any set of components they want to change. Those components bring their exact envs, effective portable configuration, aspect configuration, and lockfile-derived dependency graphs; the destination workspace derives its own root configuration and lockfile for that selection. This compositional workflow is a conformance requirement alongside reproducing the canonical monorepo.

The intended result is a workflow such as:

```shell
pnpm vcs init --scope my-org.my-repository
pnpm vcs status
pnpm vcs commit -m "update the parser"
pnpm vcs switch --create feature
pnpm vcs push --set-upstream origin feature
```

Internally these operations are Bit operations. `commit` snaps components, `switch` switches lanes, and `push` exports the lane and its component objects. A user may use the corresponding `bit` commands directly. Git is optional migration tooling, not part of native operation.

The same model can cover a repository that is not a pnpm workspace. With no discovered projects, the implicit root component owns the complete tracked tree. Splitting the tree into package components is therefore a semantic enhancement rather than a prerequisite for version control.

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

One invocation of Bit's version maker creates one batch ID and passes it into every component version made by that operation. On a lane, the same batch ID is used as the lane-history key. The history entry records that lane's component heads and deletions after the operation. Objects are persisted after component versions and lane history have been prepared. This is the workspace operation boundary the pnpm frontend consumes.

A lane is intentionally an overlay of component heads, not a Git branch containing a frozen copy of every component in the workspace. Components not present on the lane resolve through main or the lane it was forked from according to existing Bit semantics. Lane history therefore records lane-local state, not a second repository-wide tree. The pnpm frontend preserves this model rather than quietly changing lanes into Git branches.

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

In native pnpm mode, the workspace manifest is the default component-boundary declaration. pnpm resolves the workspace projects using the same rules as installation and sends Bit a structured project inventory:

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

Bit registers or updates these components before status or snap. Users do not have to run `bit add` for every package. Manual Bit components may coexist, but a path may have only one owner.

Package names are useful defaults for component identity but are not sufficient by themselves: packages may be unnamed, two private projects may temporarily declare the same name, and a package may be renamed. A version-free identity map persists the selected Bit component ID for every project root. It is workspace source configuration and can itself be tracked by the root component. Component versions never appear in that map, avoiding a self-reference when a snap updates component heads.

The exact identity file and naming transformation are unresolved. The invariant is that moving or renaming a package is an explicit component-identity operation, not an accidental deletion and creation caused only by its current path.

Components used for VCS tracking are allowed to be source-only. A valid workspace project need not have a Bit main file, build environment, publishable package name, or runnable entry point. Bit currently uses those concepts for development and packaging; the VCS layer must not invent them merely to store files. Existing Bit components retain their normal environment and build behavior.

### Workspace inputs and portable component state

A pnpm monorepo and a composable Bit workspace optimize configuration at different boundaries. A monorepo conventionally keeps one TypeScript, lint, test, and package-manager configuration at its root. A Bit component records its env and aspect configuration so that it can be imported into a newly created workspace without importing its original repository. Native VCS mode retains both properties by distinguishing authoring inputs from snap outputs and checkout projections:

| Concern | Workspace authoring input | Portable component state | Destination-workspace projection |
| --- | --- | --- | --- |
| Dependencies | `pnpm-lock.yaml`, manifests, catalogs, overrides | Resolved component dependency graph | A lockfile for the selected component set |
| Tooling and build behavior | Shared root configuration, workspace variants and policy | Exact env/aspect references and normalized effective component configuration | Hoisted root configuration and per-component extending files |
| Workspace layout | Project roots, globs, project references | Layout-independent component identity and relationships | Workspace globs, paths, and project references for the destination layout |

The component model, not the original root files, is the portability boundary. During status and snap, Bit calculates the effective configuration of every component from its previously stored model configuration, exact env, workspace variants and policy, shared root authoring configuration, and versionable component overrides. The result is a configuration graph containing:

- the exact env and aspect component references needed to interpret and execute the configuration;
- normalized semantic settings required to build, test, lint, format, or otherwise develop the component;
- references to shared, content-addressed configuration payloads when several components have the same effective settings;
- component-specific deltas from those shared settings.

Absolute paths, workspace-relative project references, component directories, and other checkout-layout details are not portable semantic settings. They are regenerated in the destination workspace. A configuration graph may be stored inline in existing aspect data or through immutable referenced objects; either representation is part of the component object graph and does not introduce a repository commit DAG.

Envs define the boundary for tool-specific configuration. An env may expose a resolver that converts its workspace authoring inputs into normalized portable component configuration, and a materializer that converts a set of component configurations into efficient workspace files. When the env component itself already contains all shared behavior, the portable state can consist primarily of the exact env reference and component-specific parameters. Conventional root configuration remains supported when an env knows how to capture its semantic fields. Generic pnpm or Bit code must not attempt to interpret every tool's configuration format.

The fingerprint of the resulting dependency and configuration graphs participates in component status. A lockfile, root configuration, workspace variant, or env change that changes a component's calculated model marks that component as modified even if none of its source files changed. Snapping records new versions for all such affected components in the same batch as the changed root or env component. A filtered snap must either include the affected components automatically or refuse to create a state whose root authoring inputs disagree with the newly snapped component models.

This is the same derivation pattern Bit already applies to dependencies. The root lockfile remains versioned so that the canonical workspace can be reproduced exactly, while each snapped component carries the dependency graph calculated from it. Likewise, root authoring configuration remains versioned for the canonical workspace, while each component carries the effective portable configuration calculated from it.

Importing selected components into a different workspace follows only their component object graphs. Bit imports the referenced envs, aspects, configuration payloads, and dependencies, then generates a destination lockfile and workspace configuration for precisely that selected set. Components with the same effective configuration share one hoisted root projection where the tool permits it. Components using different envs or configurations receive separate generated bases and thin extending files. Incompatibilities are reported instead of silently discarding a component's snapped configuration.

### Exclusive file ownership and the root component

Every tracked path belongs to exactly one component version. Ownership is calculated as follows:

1. Bit/VCS metadata and generated dependency directories are excluded.
2. A path below a pnpm project root belongs to that project.
3. Where project roots are nested, the deepest matching project owns the path; a parent project excludes the nested project's subtree.
4. Existing explicitly declared Bit ownership takes precedence only when it does not conflict with the pnpm inventory.
5. Every remaining tracked path belongs to the implicit root component.

The root component typically contains:

```text
pnpm-workspace.yaml
pnpm-lock.yaml
package.json                 # when the root is not itself a workspace project
workspace.jsonc
shared authoring configuration
.github/**
docs and repository policy
the version-free component identity map
other non-ignored, unclaimed files
```

If the workspace root is itself a pnpm project, it is also the root component; it owns root-level files but excludes nested project roots. Otherwise Bit creates a source-only internal component with a stable ID derived from the repository/scope identity.

This component removes the main reason Git is currently needed for completeness. A CI-only change may change only the root component. A lockfile or shared-configuration change also changes every project component whose calculated dependency or configuration graph changed. A checkout or clone materializes the root component together with the package components and therefore reconstructs the whole tracked workspace.

Generated paths such as `node_modules`, the pnpm store, capsules, local VCS metadata, and env-materialized configuration files are excluded. A generated root facade may be replaced by a small tracked authoring file only when the env defines which fields are authoritative inputs and which are derived layout. Existing ignore files can be honored without requiring Git to be installed. The final native ignore filename and compatibility with `.gitignore` are product decisions, but ignore evaluation must be identical in pnpm's project inventory and Bit's file tracker.

### Workspace metadata without Git

`.bitmap` currently combines durable component mapping with checkout state such as component versions. Tracking that exact file inside a component is unsuitable: snapping changes the versions written to `.bitmap`, which would make the root component modified immediately after its own snap.

Native VCS mode separates two kinds of state:

- **Durable, versioned workspace authoring configuration:** project root to stable component ID, default scope, variants/aspects, shared tool inputs, ignore rules, and other information needed to reproduce the canonical workspace. This contains no current component versions and belongs to the root component.
- **Derived local checkout state:** current lane, checked-out component heads, generated links, caches, and compatibility data needed by existing Bit internals. This is reconstructed from the selected lane/main heads and the durable identity map.

`.bitmap` may remain the local compatibility representation initially, but it is regenerated after clone, switch, checkout, and reset and is not the authority that a remote clone must preserve. In the longer term Bit may split its version-free mapping from checkout state explicitly.

`workspace.jsonc`, `pnpm-workspace.yaml`, `pnpm-lock.yaml`, and authored root configuration remain real tracked files in the root component. They are restored before the canonical workspace is fully loaded. Bootstrapping a clone therefore has two stages: fetch enough root metadata to discover identities and authoring configuration, then resolve and materialize the effective lane/main component state and its derived files. A newly composed workspace that imports only selected components instead creates or uses its own root component and materializes from the portable state in those component versions; it does not require the source workspace's root component.

### Snap batches as workspace commits

`pnpm vcs commit` delegates to a normal non-soft Bit snap. Before selecting the final component set, Bit calculates each component's resolved dependency graph from the lockfile and its effective configuration graph from env, aspect, workspace, root-authoring, and component inputs. A change in either calculated graph is a component change. Without an explicit filter, the operation includes all source-, dependency-, and configuration-changed components, matching the desired workspace-commit behavior. Bit may additionally auto-snap affected dependents according to its existing dependency rules. Every resulting version carries the operation's batch ID.

The workspace log displays one entry per batch rather than one row per changed component:

```text
4f8c2e1  update the parser
  parser       2a9e… → 91bc…
  parser-tests d81a… → c3f0…
  workspace    70ed… → 88a1…
```

The batch entry resolves to the component versions created by that operation. Components not in the batch are unchanged and continue to resolve through the current lane/main model. This permits a grouped workspace log, batch diff, and batch reset without inventing a second repository snapshot.

An explicitly filtered snap is still a valid workspace operation: only selected, affected, and auto-snapped components receive new versions. It cannot snap a new root lockfile or configuration input while deliberately omitting a component whose newly calculated portable model depends on that input. There is no file-level staging area in the first proposal. Bit's atomic unit is the component. File- or hunk-level partial snapshots could be considered later but are not required for workflows that intentionally commit at project/component granularity.

### Main and lanes

Non-default lanes already record lane-history entries after snaps. On main, the versions still carry their batch IDs, so a workspace log can group component histories by batch without first adding a default-lane object. An index may make that lookup efficient, but it does not become another history authority. Component version parents remain the authority for divergence and merge.

A lane remains an overlay of component heads with its existing fork and component ancestry semantics. Switching lanes writes the lane's components, resolves the remaining workspace components through Bit's existing fallback rules, and regenerates derived checkout metadata. Merging lanes continues to merge components independently, including the root component when it changed. The workspace log groups any component versions created by the merge operation and includes the existing lane-history event.

Lanes can contain different project sets. A component absent from the lane continues to resolve through the lane's main/fork baseline; a component explicitly deleted on the lane is removed. Creating or deleting a pnpm project records the corresponding component addition or deletion, and switching applies it with the same safety checks Bit checkout uses for modified or untracked files.

### Scope as the remote repository

The first native mode assigns all components owned by one pnpm workspace—including the root component—to one repository scope. External Bit components may remain dependencies from other scopes. Keeping owned heads and lane history in one scope gives clone, fetch, push, permissions, and divergence checking one collaboration boundary.

The existing Bit object protocol transfers component versions, file objects, lanes, and lane histories. Native pnpm VCS mode builds on that protocol rather than adding a Git object server. The required high-level operations are:

- **clone:** initialize local Bit metadata, fetch root metadata, the selected lane heads, and the main/fork heads needed to complete the effective workspace, then regenerate checkout metadata;
- **compose/import:** create or retain a destination root component, import an arbitrary selected component set and its reachable env/configuration/dependency objects, then generate the destination workspace's lockfile and shared configuration;
- **fetch:** import new component versions, heads, and histories without modifying the working tree;
- **pull:** fetch and then fast-forward or invoke the existing lane/component merge policy;
- **push:** export every object required by the selected batch/lane and advance the remote lane only if its expected head state has not changed;
- **status:** compare local component heads and files with the selected local and remote lane states.

Bit's existing divergence and export logic supplies most of this behavior. The VCS contract should tighten the externally visible transaction boundary: a rejected or interrupted push must not publish a lane state that references only part of a batch, and a concurrent push must fail or merge rather than silently overwrite heads.

Repository discovery and authorization can use scope identity initially. A separate forge, Git host, or Git credential is not required. Code review, issues, and a pull-request UI are distinct hosting features; they are not prerequisites for source transfer.

### pnpm and Bit integration

pnpm is the frontend, not a second implementation of Bit version control. During the initial implementation `pnpm vcs` communicates with a compatible Bit installation through a versioned structured subprocess or daemon protocol. This avoids a package dependency cycle: Bit embeds pnpm's installation engine, so the pnpm CLI must not load Bit's full Harmony runtime in-process.

The integration protocol includes:

- workspace project inventory;
- root authoring inputs and generated-path classification;
- command request and capability versions;
- structured status, including source, dependency-graph, and configuration-graph changes;
- structured history, diff, conflict, materialization, and progress events;
- cancellation and exit classification;
- the resulting batch, component, lane, and scope IDs.

Human-readable Bit output is not parsed. pnpm can report how to install a compatible Bit version when the capability is unavailable. A future extracted headless Bit client library may replace the subprocess without changing command semantics.

Direct Bit usage remains fully supported:

```shell
bit status
bit snap -m "update the parser"
bit lane create feature
bit switch feature
bit export
```

The pnpm commands add automatic pnpm project discovery and familiar VCS terminology; they do not create history invisible to Bit.

The [draft pnpm CI RFC](https://github.com/pnpm/rfcs/pull/25) treats version control as a read-only source provider. This native Bit mode implements that provider directly: changed projects come from component status/history, project input identity comes from component versions, and agents materialize a lane/batch through Bit. The CI engine remains read-only and never snaps or exports merely by running a pipeline.

### Operation without Git

Git-free operation is a conformance requirement, not a later optimization. Tests run with no `git` executable on `PATH` and no `.git` directory. A clean machine must be able to:

1. clone a main or lane state from a Bit scope;
2. restore root configuration and every project;
3. create a different workspace by importing any selected components without their source workspace's root component;
4. fetch their envs, aspects, portable configuration, and dependency graphs;
5. generate a destination lockfile and deduplicated root/per-component configuration;
6. install the workspace with pnpm;
7. detect source, root, dependency-graph, and configuration-graph changes;
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

It also duplicates information Bit already stores: file contents would be reached through both repository trees and component versions, component parent DAGs would coexist with a second repository DAG, lane state would be projected from another branch representation, and every operation would need rules for which graph is authoritative. For pnpm workspaces, exclusive component ownership plus an implicit root component already reconstructs the entire tree. The snap batch is already the multi-component operation boundary. This RFC therefore extends the existing model rather than placing another VCS underneath it.

The generic model should be reconsidered only if exclusive file ownership or component-granular history proves fundamentally incompatible with required workflows.

### Keep Git as the repository VCS and use Bit only for components

This is the current, mature workflow and remains supported. It has maximal ecosystem interoperability and requires less new bootstrap and root-file behavior. It also means a workspace cannot use Bit alone: Git remains the source of root configuration and mapping, CI needs a Git checkout before Bit import, and repository-level collaboration is split across two systems. It does not meet the goal of this RFC.

### Treat the entire workspace as one component

One component rooted at the workspace would make Bit a general file-tree VCS with almost no mapping work. It discards Bit's primary advantage: independent package histories, dependency-aware snaps, package-selective import, and component-level lanes and merge. The implicit root component is only the fallback owner; discovered projects remain separate components.

### Implement a separate VCS inside pnpm

pnpm could implement commits, branches, storage, and remotes independently and optionally publish package projections to Bit. This creates two implementations and makes Bit an export target rather than the version-control system. It also duplicates scope hosting, component history, checkout, merge, and dependency-aware behavior. The proposal instead keeps one authority and adds a pnpm frontend.

### Adopt another Git-compatible frontend

Tools such as Jujutsu provide improved workflows while retaining Git storage and hosting. They are useful prior art for history editing and coexistence but do not make pnpm projects or Bit components the native revision units, and Git remains part of the stack. This may be the better choice for users who prioritize universal Git tooling over Bit-native package history; it is not an implementation of Git-free Bit operation.

## Implementation

### Phase 1: make a Bit workspace complete without Git

- Introduce or formalize source-only components without required main files or build environments.
- Automatically construct components from a pnpm project inventory.
- Create the implicit root component and enforce exclusive ownership across nested projects.
- Split durable, version-free workspace identity from derived `.bitmap` checkout state.
- Classify authored root inputs separately from env-materialized configuration paths.
- Bootstrap root metadata before loading and materializing the rest of a cloned workspace.
- Add Git-free integration tests with root-file-only and multi-project changes.

### Phase 2: derive portable component state

- Formalize the lockfile-derived dependency graph as a component change input during status and snap.
- Define an env API for resolving workspace/root inputs into portable component configuration and materializing that state in another workspace.
- Store exact env/aspect references, normalized component configuration, and any shared content-addressed configuration payloads in the reachable component object graph.
- Include dependency-graph and configuration-graph fingerprints in component change detection.
- Auto-include affected components or reject a filtered snap when root inputs would otherwise disagree with newly snapped component models.
- Compose arbitrary imported component sets into a generated destination lockfile and deduplicated root/per-component configuration.
- Test one-env monorepos, mixed-env workspaces, root configuration changes, env upgrades, and incompatible imported configurations.

### Phase 3: expose the snap batch as the workspace operation

- Index batch IDs and expose them consistently in structured snap output.
- Build the main workspace log by grouping component histories by batch ID.
- Support workspace log, diff, and reset by batch ID.
- Ensure deletion, rename, merge, tag, reset, import, and export preserve batch grouping.
- Document and test that lane history contains lane-local heads while other components resolve through the lane baseline.
- Verify that persistence and recovery never expose a partial batch as a completed revision.

### Phase 4: native clone and collaboration

- Add clone/compose/fetch/pull/push operations expressed entirely through scopes, lanes, histories, and component objects.
- Regenerate derived checkout metadata after clone, switch, checkout, and reset.
- Fetch reachable env, aspect, configuration, and dependency objects when composing a workspace from selected components.
- Materialize the destination workspace without fetching the selected components' source root component.
- Require expected remote state for lane updates and test concurrent pushes.
- Restore the root component before full workspace initialization.
- Validate operation without Git installed on Linux, macOS, and Windows.

### Phase 5: pnpm frontend

- Add the experimental `pnpm vcs` command namespace to both pnpm CLI implementations.
- Define the versioned pnpm-to-Bit integration protocol and capability handshake.
- Use pnpm's canonical workspace discovery to supply project inventory.
- Expose structured source/dependency/configuration status, batch history, diff, materialization, lane, and remote results.
- Implement the native Bit source provider for pnpm CI.

The Rust and TypeScript pnpm CLIs implement the same user-visible contract. They may share the process-protocol client, fixtures, and output golden tests even though Bit performs the VCS operation.

### Phase 6: optional Git bridge

- Define explicit coexistence status when `.git` is present.
- Prototype Git commit to Bit batch import while preserving authors, messages, and parent relationships where meaningful.
- Record immutable Git commit to Bit batch mappings.
- Evaluate whether a Git remote helper adds value once native scope collaboration is available.

This phase is not required to call the native workflow complete.

### Affected repositories

- **teambit/bit:** source-only component support, pnpm inventory ingestion, root component ownership, metadata bootstrapping, portable configuration resolution/materialization, dependency/configuration change detection, batch indexing and operations, Git-free clone/composition/collaboration, structured integration protocol, and recovery/security tests.
- **pnpm/pnpm:** VCS command surface in both CLI stacks, canonical project inventory, root authoring inputs, Bit capability discovery, structured reporting, materialization coordination, configuration, and CI source-provider integration.
- **pnpm/rfcs:** follow-up RFCs if final command UX, identity naming, or multi-scope collaboration need independent ratification.

Bit and pnpm releases negotiate an explicit protocol version. pnpm does not assume that whichever `bit` executable is on `PATH` supports native VCS mode.

## Prior Art

**[Bit snaps](https://bit.dev/reference/components/snaps)** are the direct foundation: component versions with parent history, lockfile-derived dependency graphs, env/aspect configuration, dependency-aware multi-component snapping, and immutable file objects. This RFC promotes the shared batch to the pnpm-facing workspace operation and generalizes the workspace-input-to-portable-model derivation to shared configuration.

**[Bit lanes](https://bit.dev/reference/lanes/merge-lanes)** provide switching, independent component heads, main/fork fallback, merge, and remote collaboration. Native pnpm mode keeps this component-overlay model instead of translating it into frozen Git branch trees.

**[Git](https://git-scm.com/)** remains the compatibility baseline for user expectations around status, safe checkout, history, branches, concurrent push rejection, and recovery. Its single tree/commit representation is not copied where Bit's component graph already supplies the invariant.

**[Jujutsu](https://jj-vcs.github.io/jj/latest/)** demonstrates that a VCS frontend and its underlying storage model need not share Git's command semantics, and that coexistence must make operation ownership explicit.

**Monorepo tools** derive affected projects and project histories from Git paths plus a workspace graph. The Bit-native model stores project history directly and uses pnpm's declared boundaries, avoiding path attribution as the primary semantic model.

## Unresolved Questions and Bikeshedding

- **Command name.** Is `pnpm vcs` an incubation namespace, the permanent frontend, or should pnpm expose Bit terminology such as `snap` and `lane` directly?
- **Bit distribution.** Is a separately installed Bit executable required, can pnpm provision a compatible version, or should a headless Bit client eventually ship independently?
- **Root component identity.** How is its component ID derived, displayed, exported, and protected from collision with a user component?
- **Project identity.** How are scoped npm names mapped to Bit IDs, and how are unnamed, duplicate, renamed, or moved projects handled?
- **Identity-map format.** Which version-free file replaces the durable part of `.bitmap`, and how does existing Bit tooling migrate to and from it?
- **Main workspace log.** Is grouping component histories by batch ID sufficient, or is a derived persistent index needed for large scopes?
- **Batch ancestry.** Component version parents carry merge ancestry, while a batch groups an operation. Does workspace-level UI need explicit parent batch IDs, or would that incorrectly imply a second repository DAG?
- **Batch ID format.** Is the existing UUID sufficient as a public workspace revision ID, should it receive a short display form, or should future batches derive an ID from their resulting heads and metadata?
- **Owned scopes.** Must all workspace projects use one repository scope in the first release, and how should existing multi-scope Bit workspaces behave?
- **Lane overlay semantics.** How prominently must the pnpm frontend explain that a lane contains changed component heads and resolves other components through its baseline rather than freezing every workspace project as Git does?
- **Configuration authority.** Which shared settings should be authored in an env component, which conventional root configuration formats may an env capture, and which settings are workspace-only policy?
- **Portable configuration representation.** Is normalized configuration stored in existing aspect data, immutable shared objects referenced by component versions, or both? Which representation makes diffs and provenance understandable?
- **Env protocol.** What resolver and materializer contract lets an env capture semantic settings while excluding source-workspace paths and later generate efficient configuration for a different layout?
- **Change invalidation.** How does Bit efficiently determine which components' dependency or configuration graph changed after editing a lockfile, catalog, workspace variant, root tool configuration, or env?
- **Mixed-env composition.** When imported components require incompatible root-level tool configurations or package-manager constraints, which cases can be isolated and which must prevent composition?
- **Generated-file ownership.** How are authored root inputs distinguished from generated facades so status neither loses user changes nor snaps materialized output as a second source of truth?
- **Partial commits.** Is component-level selection sufficient, or must the pnpm frontend eventually support file- or hunk-level staging inside one component?
- **Nested and overlapping projects.** Is deepest-root ownership always correct, and how are pnpm projects that intentionally consume sources outside their roots represented?
- **Ignored files.** Should native mode keep `.gitignore` as a compatible convention without Git, introduce `.bitignore`, or support both with a defined precedence?
- **Clone address.** What user-facing URL identifies a repository scope and lane distinctly from importing an ordinary Bit component?
- **Remote atomicity.** Which existing export paths need strengthening so one pushed batch cannot leave a visible partial lane state?
- **Component-only operations.** How should a snap/export performed directly on a component outside a complete workspace appear in workspace batch history after import?
- **History editing.** Squash, rebase, amend, and cherry-pick can be component-oriented or batch-oriented; their workspace semantics require a separate design.
- **Git migration.** How much original Git history and parent structure must be retained when one Git commit maps to several component versions?
- **Success criteria.** Before leaving experimental status, the feature needs cross-platform Git-free clone/compose/commit/branch/merge/push tests, arbitrary component import with env/configuration portability, crash recovery, concurrent remote updates, large-workspace performance targets, and a security review.

These questions refine the existing Bit model; none requires a second canonical repository tree unless the component-ownership premise itself is rejected.
