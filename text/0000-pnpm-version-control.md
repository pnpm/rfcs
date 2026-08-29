# pnpm as a version control system

## Summary

This RFC proposes an experimental, distributed version-control system exposed through pnpm and backed by a new repository-level storage and transport layer in [Bit](https://github.com/teambit/bit). It versions an arbitrary working tree, but additionally records the pnpm workspace package graph and can project packages into Bit components. The universal layer consists of immutable blobs, trees, commits, and mutable refs; pnpm's package-aware and Bit's component-aware models sit above it. The first release is not a reimplementation of the Git wire protocol or a promise to replace every Git tool. It is a native local and remote VCS with a Git bridge, designed so an existing repository can adopt it without first becoming a Bit workspace.

## Motivation

pnpm already maintains much of the information that JavaScript repositories ask a version-control client to rediscover after every checkout: workspace boundaries, dependency edges, package-manager and runtime pins, content hashes, generated dependency trees, and the distinction between source inputs and reproducible installation outputs. Bit maintains the complementary half: immutable file objects, component revisions with parent links, branch-like lanes, component-level diff and merge, local and remote scopes, and dependency-aware history.

Git sees a pnpm workspace as one undifferentiated tree. A package rename is a collection of path changes. An affected package set is inferred by attributing changed paths to package roots and then traversing a separately reconstructed dependency graph. A release tag names the entire repository even when the released unit is one package. Sparse materialization is file-oriented even when the useful unit is a package and its transitive dependencies. None of this makes Git incorrect; it means the package semantics pnpm knows are always an index rebuilt above the source-control system rather than data the source-control system can preserve.

Bit has the richer unit, but its current persistence model is component-first. A Bit `Version` contains component files, dependencies, parents, build data, and aspect metadata; a lane is a set of component heads. That is a strong package VCS and a poor universal root-tree VCS. Files outside components, changes to component boundaries, repository-wide configuration, and atomic history across root files and many components need a repository object above the component graph. Generalizing the existing component version into that object would make the universal layer inherit assumptions about main files, scopes, environments, and build pipelines that arbitrary repositories do not have.

The opportunity is therefore not "make pnpm invoke Bit commands." It is to extract and extend the common storage model so the layering becomes:

```text
                  repository VCS kernel
          blobs · trees · commits · refs · transport
                        /             \
          Bit component model      pnpm VCS frontend
          snaps · lanes · CI       packages · lockfile
                        \             /
                    Bit scope backend
```

This gives pnpm a version-control model whose native semantic unit can be a package without preventing it from versioning arbitrary files. It gives Bit an atomic repository history beneath component history. It also removes a dependency cycle that a direct integration would create: Bit embeds pnpm's installation engine today, so pnpm must not load the Bit CLI or Harmony runtime to perform basic VCS operations.

The intended outcomes are:

- a repository can initialize and use the VCS without a `.bitmap`, Bit aspects, a `package.json`, or a pnpm workspace;
- a pnpm workspace gets exact package identities and dependency-aware change information without reconstructing them from path diffs;
- one repository revision atomically names root files, package files, workspace metadata, and component projections;
- local work remains fully distributed and offline;
- a Bit scope can host repositories as well as components, using its existing authentication and deployment model;
- Git repositories have an incremental migration path rather than an all-at-once conversion;
- pnpm's TypeScript and Rust implementations share one object format and one behavioral implementation.

This is an engine-sized project, not a feature-sized command addition. The RFC establishes the boundaries and invariants before a prototype accidentally turns its first storage layout into a permanent public format.

## Detailed Explanation

### Product boundary

The proposal has three separable pieces:

1. **A generic VCS kernel.** It owns object identity, local storage, the working-copy index, refs, checkout, history, merge, reachability, garbage collection, and the remote protocol. It has no dependency on pnpm package manifests or Bit aspects.
2. **A pnpm frontend.** It exposes commands, derives workspace metadata, and integrates package identities and dependency graphs into revisions. An arbitrary non-JavaScript repository uses the same frontend with the workspace metadata absent.
3. **A Bit backend and projection layer.** A scope stores and transfers generic repository objects and refs. For Bit-aware repositories, a committed workspace snapshot also maps components to the component revisions produced by the same transaction.

The generic kernel is the compatibility boundary. Neither pnpm CLI stack imports Bit's current runtime. Bit and pnpm both consume the kernel.

The [pnpm CI RFC](https://github.com/pnpm/rfcs/pull/25) describes version control as a read-only source provider from the CI engine's point of view and explicitly says that VCS mutation is a different product. This RFC is that different product. If both are adopted, the native provider implements the CI source-provider interface directly, while `pnpm ci` remains read-only and never commits or pushes as a side effect of running a pipeline.

### Command surface

During incubation, commands live under a namespace so pnpm does not reserve a large collection of top-level command names before their semantics are stable:

```shell
pnpm vcs init
pnpm vcs status
pnpm vcs add <path...>
pnpm vcs commit -m "describe the revision"
pnpm vcs log
pnpm vcs branch <name>
pnpm vcs switch <name>
pnpm vcs merge <name>
pnpm vcs fetch [remote]
pnpm vcs pull [remote]
pnpm vcs push [remote] [refspec]
```

These names explain the examples; ratification does not require settling the final namespace. `commit` is used for a repository revision and `component revision` for the corresponding Bit-level object. The implementation may offer a snapshot-all workflow in addition to an explicit staging index, but the storage and merge model must be capable of partial commits.

Initialization creates only local VCS metadata. It does not initialize a Bit workspace, choose a component scope, alter manifests, or contact a server. A directory without `package.json` is valid. When `pnpm-workspace.yaml` exists, the next commit may include a package-aware workspace index.

The only permanently implicit ignored path is the VCS's own metadata directory. `pnpm vcs init` may offer to add common generated paths such as `node_modules` to an ignore file, but status must not silently hide arbitrary pnpm-related paths. For migration, the first implementation reads `.gitignore`; whether it introduces a differently named native ignore file is unresolved.

All porcelain commands provide a stable structured-output form. Human output is not a protocol between the TypeScript CLI, the Rust CLI, Bit, CI, and editor integrations.

### Universal object model

Four object types are sufficient for the universal history graph:

```text
Blob
  bytes

Tree
  sorted entries:
    name
    kind: blob | tree | symlink
    executable: boolean
    object: ObjectId

RepositorySnapshot
  root: TreeId
  workspaceIndex?: WorkspaceIndexId
  componentSet?: ComponentSetId
  additionalMetadata?: typed object references

Commit
  snapshot: RepositorySnapshotId
  parents: CommitId[]
  author
  committer
  message
  typed headers
```

Every object begins with a domain-separated type and schema version and is serialized canonically. Its `ObjectId` is the digest of those canonical bytes, represented with an algorithm identifier rather than assuming SHA-1 forever. The initial algorithm is expected to be SHA-256, but the on-disk and network representations are hash-agile. Random UUID-derived revision IDs, useful for existing Bit snaps, are not used for native repository commits.

Blob contents are opaque bytes. The universal layer does not normalize line endings, parse JavaScript, run clean filters, or infer generated files. A tree records file kind and the executable bit. Symlink contents are their targets and checkout never follows a repository symlink while writing through it. Directory entries are sorted by a byte-level canonical ordering. Platform-specific filename representation and checkout restrictions are called out as an unresolved format decision rather than being accidentally defined by JavaScript strings.

`RepositorySnapshot` separates the exact working tree from its interpretation. A repository with no pnpm workspace has only a root tree. A pnpm-aware commit may additionally reference an immutable `WorkspaceIndex`:

```text
WorkspaceIndex
  schema
  producer
  workspace configuration identity
  packages:
    stable package identity
    manifest path
    package root or tree reference
    manifest identity
  dependency graph
```

The index is committed data, not a mutable database entry whose meaning changes when a newer pnpm parses an old lockfile. Its schema and producer are recorded. Historical operations can either consume the stored schema or derive a newer temporary view without rewriting history.

`ComponentSet` maps Bit component identities to component revision references and to the repository paths or tree objects from which those revisions were projected. The repository tree remains canonical. This direction matters: constructing the root tree from component versions cannot faithfully represent unowned root files, overlapping semantic units, or a change that moves a file across component boundaries.

The same immutable blobs can be reached from both the root tree and component revisions, so the additional graph does not duplicate file contents. Existing Bit scopes can retain their object types; a compatibility layer can project native objects into current Bit `Source`, `Version`, and `Lane` objects while the new model is introduced.

### Refs and transactions

Objects are immutable; refs are mutable names. A branch is a ref to a commit. A tag is an immutable or policy-protected ref with optional annotation. Local refs have reflogs so an accidental reset or branch deletion is recoverable.

The primitive mutation is a compare-and-swap transaction:

```text
updateRefs([
  { name, expectedOld, new },
  ...
])
```

Either every update succeeds or none does. A push cannot advance one package or component head and fail halfway through advancing the repository branch. The server verifies that the expected old values still match, every new object is available and valid, and repository policy authorizes each update. Force push is an explicit policy decision, never an omitted expected value by accident.

Mutable local indexes and refs require a transactional store. Immutable object files can remain simple files and later be packed; refs, reflogs, worktree registration, and the working-copy index should use a crash-safe database or journal. Correctness must survive two pnpm processes, an editor integration, and an interrupted checkout operating concurrently.

### Working copy and index

The working-copy index serves three purposes:

- a stat cache so clean status does not hash every file;
- the proposed contents of the next partial commit;
- multiple merge stages for conflicted paths.

It is not the source of committed history. It can be deleted and rebuilt from `HEAD` plus the working tree, except for intentionally staged content and unresolved conflict stages.

Status walks tracked and candidate paths without crossing the repository root. It uses `lstat`, never follows a symlink while discovering files, and validates that every path produced by ignore matching and filesystem traversal remains contained. File bytes are hashed only when the stat cache cannot prove that the indexed identity is still valid. A filesystem monitor may later invalidate index entries but is not required for correctness.

Checkout is a planned transaction:

1. calculate all removals, writes, directory changes, and conflicts;
2. refuse before mutation if an untracked or modified path would be lost unless the user explicitly chooses a destructive mode;
3. materialize into temporary paths;
4. validate object IDs and path containment;
5. replace destinations and update the index;
6. advance the worktree ref only after the filesystem operation succeeds.

Perfect atomicity across an arbitrary filesystem tree is not available on every platform. The journal must make an interrupted checkout detectable and recoverable rather than presenting a partially updated tree as clean.

Repository discovery, nested repositories, linked worktrees, sparse checkout, submodules, and large-file storage are distinct features. The object model must not prevent them, but only repository discovery and one working tree are required for the first implementation.

### Diff and merge

Diff operates between trees, between the index and a tree, or between the working copy and the index. Package and component summaries are derived views over the same path-level changes. A user can always inspect the underlying file changes; package-level output must not conceal them.

Merge is a three-way tree operation over a merge base, the current commit, and the other commit. The first complete implementation needs:

- multiple-parent merge commits;
- add/add, modify/modify, delete/modify, file/directory, executable-bit, and symlink conflicts;
- a line-oriented text merge with a binary fallback;
- conflict stages in the index;
- `continue` and `abort` with a journaled pre-merge state;
- deterministic results independent of directory traversal order.

Rename detection is initially an interpretation of delete/add pairs and is not part of object identity. Exact-content renames are cheap; heuristic similarity and rename-aware merge can follow after correctness for the basic conflict matrix. Package moves can be represented more strongly when a stable package or component identity exists, but arbitrary files retain path-based semantics.

Rebase, cherry-pick, revert, bisect, blame, stash, custom merge drivers, attributes, and hooks are not prerequisites for proving the storage and collaboration model. The commit graph and index should support adding them without a format migration.

### Package- and component-aware history

A pnpm workspace commit records a complete repository snapshot and, when available, a workspace index. This makes several operations native:

- changed packages are the packages whose referenced trees, manifests, or relevant dependency edges changed;
- dependents are obtained from the historical dependency graph stored with the compared revisions;
- a task's package input identity can start with a package tree ID rather than hashing a glob from scratch;
- package history can follow a stable identity through directory moves;
- a package and its dependency closure can be materialized without interpreting the current branch's lockfile;
- release tooling can associate a package version with the exact repository and component revisions that produced it.

The dependency graph is metadata, not an excuse to omit source. A checkout of a repository commit always reconstructs the universal root tree. Package-selective materialization is an explicit sparse operation whose incomplete state is recorded.

In a Bit-aware workspace, committing can produce component revisions and the repository commit in one local transaction. The `ComponentSet` referenced by the commit's snapshot records the corresponding component heads. Component revisions may record their source root-tree identity and projection paths, but cannot point back to the commit or snapshot that contains the `ComponentSet`: two content-addressed objects pointing at one another would create an unhashable cycle. A disposable reverse index can answer which repository commits contain a component revision. A Bit lane can become a component-oriented view of a repository branch. Component-only operations may continue to exist, but exporting them back into a repository must create an explicit repository commit rather than silently changing the working branch's meaning.

This relationship requires careful compatibility rules and is intentionally not defined as "a commit is a Bit snap batch." Existing snap `batchId`s group an operation but are not a canonical repository object and do not include the root tree.

### Local storage and garbage collection

The local object database begins with loose immutable objects addressed by digest. It later adds pack files and indexes without changing object IDs. Compression is outside the hashed representation. Reading an object always decompresses into bounded storage, verifies the digest, validates the typed schema, and only then exposes it to checkout or graph traversal.

Garbage collection is mark-and-sweep from refs, reflogs, worktree heads, staged state, in-progress operations, and explicitly retained promises for partial clones. A concurrent writer publishes objects before refs, so unreachable new objects are safe to collect only after a grace period or generation barrier. A reader pins the pack generation it is traversing.

The pnpm content-addressable package store and the VCS object store have related mechanics but different trust and retention rules. They may share low-level hashing, compression, indexing, and read-only-store code. They must not initially share one namespace or let pruning one store remove objects promised by the other.

### Remote protocol and Bit scopes

A Bit scope gains a generic repository service alongside its component service. Repository identity and authorization are explicit; a repository is not disguised as a synthetic Bit component. The protocol is versioned and capability-negotiated from its first public deployment.

The minimum protocol has three surfaces:

1. **Discovery:** repository identity, refs, object-format versions, hash algorithms, compression and protocol capabilities.
2. **Fetch:** the client sends wanted commits or refs and the commits it already has; the server returns a bounded stream of necessary objects plus promised-object metadata where partial fetch is supported.
3. **Push:** the client uploads a quarantined object stream and a ref transaction. The server validates every object, checks reachability and policy, promotes objects, and commits the ref transaction atomically.

The current Bit component fetch/put routes exchange component-aware object lists and run component validation. The repository protocol should be new rather than adding flags until those endpoints accept arbitrary trees. The implementations may share framing and storage underneath.

The server never trusts a client-provided object ID, uncompressed length, path, ref name, or reachability claim. Limits cover individual objects, aggregate pack expansion, graph depth, ref update count, and negotiation work. Uploaded packs remain in quarantine until validation completes. Authorization distinguishes reading objects, creating refs, fast-forward updates, destructive updates, and administrative retention.

Initial fetch may send every object reachable from a wanted commit and absent from an explicit client inventory. Efficient have/want negotiation, bitmaps, delta selection, shallow history, filters, and lazy blobs can evolve behind capabilities. Correct and resumable transfer comes before optimal packing.

Bit Cloud support is not required for the local proof of concept. A self-hosted scope is enough to validate the protocol. Hosting policy, quotas, repository discovery, and code-review UI are separate product decisions.

### Git bridge

Adoption cannot require abandoning existing forges, IDE integrations, and history on day one. The proposal therefore treats Git interoperability as a bridge with increasing levels:

1. import Git commits, trees, authors, timestamps, branches, and tags into native objects while retaining a bidirectional Git-OID mapping;
2. export native history into a Git repository with stable mappings for already exported commits;
3. provide a `git-remote-bit` helper so ordinary Git clients can fetch from and push to a Bit-hosted repository;
4. optionally retain original Git object bytes for exact round trips;
5. consider Git wire compatibility only if real integrations require it.

The remote helper is an early milestone, not a final dependency on Git. It tests the hardest backend properties—object transfer, concurrent ref updates, authentication, force-push policy, and large histories—using a mature working-copy frontend before pnpm's own frontend is complete.

Native commit IDs need not equal Git OIDs. The mapping must be immutable and collision-checked. Exact Git round-trip requires retaining details that are not part of a simplified semantic import, including raw header ordering, encoding headers, annotated-tag objects, and potentially SHA-1 object bytes. Whether exact or semantic round-trip is the first compatibility target remains unresolved and must be stated in the eventual CLI documentation.

Repositories may operate with both metadata directories during migration. Neither client mutates the other's refs implicitly. An explicit synchronization command imports or exports commits and reports divergence. Silently mirroring every native commit into `.git` would make two transaction systems appear atomic when they are not.

### One implementation across pnpm and Bit

pnpm currently has TypeScript and Rust CLI implementations whose user-visible behavior and on-disk formats are expected to remain aligned. Implementing object parsing, checkout safety, merge semantics, and pack validation twice would create both parity risk and a security boundary with two answers.

The proposed kernel should therefore have one systems implementation, expected to be Rust:

- the Rust pnpm implementation links it directly;
- the TypeScript pnpm implementation consumes a narrow N-API binding;
- Bit consumes the same N-API binding for local repositories and scope object validation;
- the object-format conformance suite is language-neutral and can test any future implementation.

The public API deals in typed requests, structured events, byte streams, and opaque object IDs. It does not expose Rust filesystem handles or Bit Harmony objects. Long operations support cancellation and progress events. The Node binding must not buffer whole packs or large blobs in JavaScript memory.

The kernel's repository is an ownership question, not an architectural one. It may begin in Bit while its object model is extracted, in pnpm while the frontend is prototyped, or in a neutral repository. Its format governance must include both projects once either publishes it.

### Security and trust

A source-control client writes attacker-controlled names and bytes into a developer's working directory and parses attacker-produced history before any project code runs. It is a security boundary at least as sensitive as package extraction.

The implementation requires, from the first prototype that reads untrusted data:

- containment checks against absolute paths, parent traversal, platform path prefixes, alternate separators, case folding, Unicode normalization, and reserved names;
- symlink-safe traversal and checkout;
- bounded decompression, allocation, nesting, graph traversal, delta depth, and negotiation work;
- digest verification after decompression and before parsing;
- canonical serialization tests and cross-platform golden vectors;
- atomic ref compare-and-swap and quarantine before push promotion;
- refusal to execute hooks, filters, merge drivers, workspace configuration, or package scripts while merely inspecting or checking out a revision;
- fuzzing of object, pack, tree, ref, ignore, index, and protocol parsers;
- recovery tests that terminate processes at every journal step;
- threat-model review before a remote accepts untrusted pushes.

Signed commits and tags can be added above immutable commit identities. Transport authentication does not imply author identity, and commit signatures do not authorize ref updates. These remain separate policy layers.

## Rationale and Alternatives

### Continue using Git and add only package metadata

pnpm could store a package graph in commits, notes, sidecar files, or a remote database while leaving Git responsible for every VCS operation. This is the lowest-cost answer and remains a valid stopping point. It improves affected selection and package history but does not give Bit scopes a repository protocol, package-granular materialization, one cross-product storage kernel, or atomic correspondence between repository and component history. Metadata stored outside the commit also develops consistency and authorization rules of its own.

This alternative should be the control in benchmarks and product evaluation. The native VCS is justified only where its package/component semantics or Bit backend produce meaningful improvements over a well-built Git integration.

### Use a Git library as the permanent kernel

Using an existing Git implementation would provide object compatibility, mature merge behavior, and immediate forge interoperability. pnpm could build package semantics above Git and Bit could host Git repositories alongside scopes. This is a credible implementation shortcut and may be used for the bridge or early frontend.

It also makes Git's repository object model and protocol the permanent universal layer. Component sets, workspace indexes, hash migration, package-selective fetch, and Bit-native scope transactions become extensions negotiated around Git rather than first-class typed objects. If exact Git compatibility is ultimately more valuable than those capabilities, this alternative should win; the RFC does not assume novelty is itself a benefit.

### Generalize the current Bit component model

A repository could be represented as one giant component, or root files could be placed in a synthetic workspace component while every package remains a normal component. This reuses the most code initially. It fails at the layer boundary: a component version requires component concepts, a lane names component heads rather than one canonical root tree, a batch groups snaps without becoming an atomic repository commit, and file ownership rules make arbitrary overlapping or changing boundaries awkward.

The proposal instead reuses Bit's immutable storage and distributed collaboration experience while placing a generic repository object below components.

### Build the VCS entirely inside pnpm

This avoids changing Bit and optimizes the feature for pnpm. It duplicates storage, remotes, authentication, component projections, and hosting, and leaves Bit consuming a second object model later. The premise of this RFC is that both projects can coordinate the kernel and that the Bit scope is strategically useful as the backend. If that coordination ceases to be practical, a pnpm-only experiment can still implement the universal object format, but remote hosting would need a separate proposal.

### Adopt another modern VCS frontend

Tools such as Jujutsu and Sapling demonstrate that Git's user interface and Git's storage/hosting ecosystem can be separated. pnpm could invoke one as its source provider rather than own a VCS frontend. This provides modern workflows sooner, but package and component semantics remain an external index and pnpm still does not own the stable provider available in every installation. Their design experience should inform working-copy and history semantics even if their product boundary is not adopted.

## Implementation

Implementation proceeds in format-first phases. No phase requires claiming that pnpm has replaced Git.

### Phase 0: conformance model

- Specify canonical object bytes, object IDs, tree ordering, refs, and transactions.
- Publish cross-language golden vectors and malformed-object fixtures.
- Define path and checkout invariants on Linux, macOS, and Windows.
- Build a repository corpus containing symlinks, executable changes, Unicode and case collisions, deep trees, merge conflicts, interrupted operations, and imported Git history.
- Prototype both content-addressed loose storage and the transactional ref/index database.

The exit condition is that Rust and an independent test decoder agree on every valid and invalid fixture. No public repository created before this point is promised long-term compatibility.

### Phase 1: local engine and experimental frontend

- Implement init, status, add, commit, log, branch, and safe checkout.
- Implement reflogs, fsck, reachable-object enumeration, and conservative GC.
- Expose the engine to both pnpm stacks through the shared Rust core.
- Keep the feature behind an explicit experimental opt-in and format version.
- Collect comparative performance data against Git for clean status, first commit, incremental commit, branch switch, repository size, and memory.

The local engine works in a repository without pnpm or Bit configuration.

### Phase 2: Git import/export and Bit remote

- Implement Git history import with stable OID mappings.
- Add the versioned repository service to self-hosted Bit scopes.
- Implement authenticated clone, fetch, push, and atomic ref transactions.
- Add quarantine, validation limits, resumable streams, and server-side fsck.
- Implement `git-remote-bit` to exercise the backend from existing Git clients.

The exit condition is two clients racing a ref update safely, recovery from interrupted transfers, and round-trip tests over the repository corpus.

### Phase 3: merge and collaboration

- Implement the full initial three-way conflict matrix, continue, and abort.
- Add pull policy, remote-tracking refs, protected refs, annotated tags, and signed-object plumbing.
- Add pack files, negotiation indexes, and scalable GC based on measured repositories.
- Integrate editor status/diff APIs using the structured protocol rather than parsing CLI output.

### Phase 4: pnpm and Bit semantics

- Define and persist `WorkspaceIndex` and `ComponentSet` schemas.
- Use package tree identities for changed-project selection and task inputs.
- Create repository commits and Bit component revisions in one transaction.
- Project repository branches into lanes and preserve repository provenance on component export.
- Add explicit package/component sparse materialization.
- Integrate the native provider with the CI engine's read-only source-provider interface.

Some package-aware metadata can be prototyped during earlier phases; this ordering means it does not get to redefine the universal object model before that model is sound.

### Affected repositories and components

- **pnpm/pnpm:** CLI command registration, configuration, structured reporting, the Rust workspace, N-API bindings, package graph integration, filters, and CI source-provider integration.
- **teambit/bit:** extracted object-store primitives, generic repository objects, local repository host integration, scope routes, ref transactions, authentication hooks, component projections, lane mapping, and migration tooling.
- **pnpm/rfcs:** follow-up RFCs for final command UX, the published remote protocol, and hosting policy if those decisions outgrow this document.

Published formats use independent schema versions. A pnpm release and a Bit release negotiate capabilities; neither assumes lockstep deployment.

## Prior Art

**[Git](https://git-scm.com/book/en/v2/Git-Internals-Git-Objects)** establishes the blob/tree/commit/ref model, content-addressed local operation, pack transfer, and an ecosystem whose compatibility is the baseline this proposal must measure against. Its [remote-helper protocol](https://git-scm.com/docs/gitremote-helpers) is the intended first bridge to a Bit backend.

**[Mercurial](https://www.mercurial-scm.org/)** provides a second mature distributed model and useful prior art for explicit repository requirements, revlogs, phases, and long-lived compatibility. The implementation should avoid accidentally treating Git behavior as the only possible answer where users depend only on the higher-level invariant.

**[Jujutsu](https://jj-vcs.github.io/jj/latest/)** demonstrates a modern VCS frontend over a Git-compatible backend, first-class working-copy commits, and a clean separation between operation history and repository history. Its coexistence model is relevant to running native and Git metadata side by side without pretending they share one transaction.

**[Sapling](https://sapling-scm.com/)** provides prior art for a source-control client designed around large repositories and for retaining Git interoperability while changing client behavior.

**[Bit](https://bit.dev/reference/components/snaps)** already implements independent component histories, dependency-aware revisions, lanes, checkout and merge, scopes, and remote object transfer. This proposal changes the layering so those features become a projection above an atomic repository tree rather than attempting to discard and recreate them in pnpm.

**Package-manager content-addressable stores**—including pnpm's own—provide implementation experience in immutable content, integrity verification, concurrent imports, side-effects isolation, and garbage collection. A source-control store has a different reachability graph and threat model, so reuse should be at the primitive and conformance level rather than by conflating the stores.

## Unresolved Questions and Bikeshedding

- **Product and command name.** Is `pnpm vcs` the permanent namespace, an incubation name, or should the generic client ultimately be branded independently of pnpm and Bit?
- **Metadata directory.** `.pnpm-vcs`, `.bit`, and a neutral name each communicate a different owner and create different coexistence behavior.
- **Staging UX.** The engine needs an index for conflicts and partial commits, but the default user workflow could snapshot every tracked change, stage packages/components, or follow Git's explicit staging model.
- **First hash and serialization.** SHA-256 is the expected baseline, but canonical CBOR, a custom length-prefixed format, and another specified encoding have different implementation and inspection tradeoffs.
- **Filename representation.** Tree names must round-trip Unix byte names while producing predictable errors for names unavailable on Windows and macOS.
- **Git round-trip target.** Is semantic import/export enough initially, or must an imported Git repository reproduce every original Git OID after export?
- **Git coexistence.** Which client owns the working copy when both metadata directories exist, and how is an explicit synchronization transaction recovered if its second side fails?
- **Repository identity in scopes.** Are repository names a new namespace under an organization, a capability of an existing scope, or a mapping to one distinguished component scope?
- **Canonical component relationship.** How do component-only snaps and lane operations relate to a repository branch when no working repository was involved?
- **Workspace-index evolution.** Which metadata is consensus-critical and committed, and which is a disposable derived acceleration index?
- **Package identity.** Package names can change and can be absent or duplicated in private workspaces. A stable identity cannot always be the current manifest name.
- **Large files and partial clones.** When should chunking, LFS-like promised objects, and lazy materialization become part of the public protocol?
- **Subrepositories and submodules.** The object model can represent a typed reference later, but cross-repository checkout and authorization semantics need their own design.
- **History editing.** Rebase, amend, and change evolution can follow Git, Jujutsu, or a Bit-lane-oriented model. The immutable commit graph supports each, while their collaboration UX differs considerably.
- **Remote review and forge features.** Hosting refs and objects does not by itself provide pull requests, review, issue tracking, or a merge queue. Whether Bit supplies those, existing forges remain canonical, or both interoperate is a separate product decision.
- **Signing and identity.** Commit signatures, organization identities, SSH keys, Sigstore-style identities, and protected-branch policy should not be collapsed into transport authentication.
- **Format governance.** Once pnpm and Bit both publish the kernel, object and protocol changes require a shared compatibility and security process.
- **Success criteria.** Before making the feature non-experimental, the project needs explicit adoption, performance, repository-size, recovery, security-audit, Git-bridge, and cross-platform thresholds.

This section should be reduced to genuinely open product decisions before ratification; format and safety questions must be resolved by the conformance phase rather than delegated indefinitely to implementations.
