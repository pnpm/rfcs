# Workspace task cache

> **Status: proposed.** Nothing specified here is implemented. Two things it stands on do exist: the task graph and scheduler of [RFC 0008](./0008-workspace-task-orchestration.md), shipped in both stacks, and the kind-discriminated artifact `subject` this document asked for, shipped in [pnpm/pnpm#14267](https://github.com/pnpm/pnpm/pull/14267). A `pnpm pipeline` proof of concept ([pnpm/pnpm#14233](https://github.com/pnpm/pnpm/pull/14233), draft, Rust only) exercised the local tier end to end — key production, input hashing, output capture, and restoration into a working tree — and what it established is folded into the text below rather than left as speculation. It is not a shipped feature, and this document does not propose that command.

## Summary

pnpm knows which workspace projects exist, how they depend on each other, and what each one's dependency graph resolves to, and it already runs their scripts in topological order. What it does not do is notice that a script's inputs have not changed since the last time it ran. This RFC adds a content-addressed cache for the output of a workspace project's own scripts — files, logs, and exit code — keyed on the task's declared inputs. The local tier needs no new trust machinery. The remote tier is [RFC 0007](./0007-shared-side-effects-cache.md) with a different subject and nothing else new.

It builds on the [workspace task orchestration](./0008-workspace-task-orchestration.md) RFC, which supplies the task graph. That dependency is load-bearing rather than cosmetic: a cache key has to name the upstream task whose output this task consumes, and package topology alone cannot.

## Motivation

A monorepo's second `pnpm -r run build` rebuilds every package, including the twelve nobody touched. The standard answer today is to put Turborepo or Nx in front of pnpm, which most large pnpm workspaces do. That works, and this RFC does not claim those tools are wrong. It claims that the part of their job that is *caching* — as distinct from task orchestration, terminal UI, and cloud services — is mostly a function of information pnpm already holds and infrastructure pnpm already runs, and that duplicating it above pnpm costs the ecosystem a second dependency graph, a second hashing pass over the same files, and a second store of the same bytes.

Two things specifically are pnpm's to know and nobody else's:

**The resolved dependency graph, per project.** A task cache has to invalidate when a project's dependencies change. Turborepo approximates this by hashing the lockfile — so any lockfile edit anywhere busts every project's cache, and a workspace where one team bumps a dependency daily gets much less cache reuse than it should. pnpm computes the actual resolved graph for each importer and already hashes it: `calcDepStateInputKey` (`pnpm11/deps/graph-hasher/src/index.ts`) exists and shipped with RFC 0007. Keying a task on the graph that project actually resolves is strictly better invalidation, and it is not available to a tool sitting on top.

**A content-addressed store at file granularity.** pnpm's store already deduplicates identical files across every package on the machine. Turborepo stores each task's output as its own archive, so a file present in ten outputs is stored ten times. RFC 0007 notes this as a byproduct; here it is a direct consequence of putting task outputs in the store pnpm already has.

The transport, the signing, the trust policy, and the artifact format are already built and shipped for dependency builds. What is missing is a key, a way to say what a task reads and writes, and a safe way to put files back into a directory the user owns.

### What already exists

- **A task graph and a scheduler**, from the orchestration RFC: `dependsOn`, per-task dispatch, cycle detection, and `--dry-run`. This RFC consumes all four and adds none of them. Shipped in both stacks — `pnpm11/workspace/task-scheduler/` and `pnpm/crates/workspace-task-scheduler/` — with the one change noted under Implementation.
- **A per-project dependency-graph hash.** `calcDepStateInputKey`, shipped in RFC 0007, minus the parts specific to a dependency.
- **A content-addressed store and an importer.** `storeController.addFileToStore` and `importIndexedDir` (`pnpm11/fs/indexed-pkg-importer/src/importIndexedDir.ts`), plus `packageImportMethod` (`pnpm11/config/reader/src/Config.ts:209`) with `clone`, `clone-or-copy`, and `copy` alongside `hardlink`.
- **The whole artifact protocol.** The signed envelope, owner scope, compatibility constraints, the `{added, deleted}` manifest, manifest path validation, and pnpr's `/-/pnpr/v0/artifacts{,/resolve,/blob}` endpoints behind `resolver.artifacts` — including, since [pnpm/pnpm#14267](https://github.com/pnpm/pnpm/pull/14267), the kind-discriminated `subject` a workspace task needs to be nameable at all.

## Detailed Explanation

### Why this needs the task graph

A cache key must account for everything a task reads. A task reads its own project's files, its resolved dependencies, its environment — and the built output of other tasks, which is the part package topology cannot describe.

Consider project A whose `test` script consumes `node_modules/B/dist`, produced by B's **`build`**. Keyed on package topology alone, the only upstream task available is B's *same-named* script, `test` — which is wrong in both directions: it includes B's tests, which A does not consume, and omits B's build, which A does. The workspace dependency supplies nothing to make up the difference, because link targets enter the dependency-graph hash by identity only (`pnpm11/deps/graph-hasher/src/index.ts:481-486` gives them `fullPkgId: linkTargetNode` and `children: {}`, with no content).

The failure is silent and it is not exotic. If **B has `build` but no `test` script**, B contributes no upstream task key at all, and A's `test` key contains nothing whatsoever about B. Change B's source, rebuild, and A's `test` takes a cache hit against a `dist` it has never seen.

With a declared graph the question is answered exactly: `test: dependsOn: ['build']` names the producing task, and the orchestration RFC's pass-through rule — a missing script resolves the edge through to that project's own upstream tasks — means a project without the named script cannot sever the chain. The key is then correct by construction rather than, as an earlier draft of this RFC was, correct by accident whenever the generous default input set happened to propagate the change transitively.

### The task key

Keys are opaque to the server and domain-separated by kind, per RFC 0007's forward-compatibility constraint. A task key is `workspace-task:v1:` followed by a hash over, in canonical order:

- **The task's identity** — the workspace-relative project directory and the script name. Both, deliberately, even though a task whose script text, inputs, environment, and dependencies all matched another's would produce the same output. Including identity costs nothing, because the store deduplicates the *files* regardless of how many manifests point at them, and it removes a class of false-sharing bug that would be extremely unpleasant to diagnose.
- **The script text actually executed** — including `pre`/`post` scripts when `enablePrePostScripts` is on, since those run as part of the task and change what it produces.
- **The declared input files**, as canonically ordered `(relative path, mode, content digest)` triples.
- **The declared environment**, as a name-to-value map. Names alone are insufficient, for exactly the reason RFC 0007 gives for builder profiles: `NODE_ENV=production` and `NODE_ENV=development` satisfy the same allowlist and produce different output.
- **The project's resolved dependency-graph hash**, from `calcDepStateInputKey`'s graph component. This covers the task's *external* dependencies — what npm packages it resolves — and is the component a tool sitting above pnpm has to approximate by hashing the lockfile.
- **The keys of the tasks this task declares a dependency on**, resolved through the orchestration RFC's graph including its pass-through rule. This covers the task's *internal* dependencies — what other tasks in the workspace produced the bytes it reads. **The graph those keys resolve through must be dependency-closed, whatever narrowed the run.** A filter or an affected-since-base selection decides which tasks are *requested*; it must not decide which projects the graph *contains*. If it does, a `^` edge truncates at the selection boundary and the key becomes a function of how the run was invoked rather than of what it computes. The mild symptom is a spurious miss whenever the selection widens. The severe one is the silent stale hit this section exists to prevent, arriving by a second door: a workspace dependency outside the selection contributes nothing to the key at all, so a rebuild of it goes unnoticed. Projects the closure pulls in are built or restored like any other upstream task and gain no requested tasks of their own.
- **The execution environment that changes output** — the resolved Node or runtime version, `executionEnv`, and the shell settings (`scriptShell`, `shellEmulator`) that decide how the script text is interpreted. The runtime version must be resolved *from the workspace*, not from the invoking process. With a context-aware toolchain — pnpm's own Node shim included — `node --version` answers per directory, so a `--dir` invocation that fingerprints its own process splits the cache against a workspace whose scripts would have run under a different runtime.

Deliberately *not* in the key: the absolute path of the workspace, the concurrency, the reporter, and anything else describing how the run was invoked rather than what it computes. Absolute paths are the one uncomfortable omission — a build that embeds one in a source map or a binary is not relocatable and a cache hit on another machine restores something subtly wrong. This is the same problem RFC 0007 leaves unresolved for dependency builds, and it gets the same answer for the default: it is eligibility's job, not the key's, because fragmenting the key by workspace path would disable cross-machine reuse entirely, which is the point of the feature.

A task that knows it is not relocatable says so, and `relocatable: false` puts the absolute workspace path into its key. That is deliberately a blunt instrument: it does not merely decline remote reuse, it declines reuse across two checkouts on the same machine, because a source map naming `/home/ci/build-7/src` is as wrong in a sibling checkout as it is on another host. Detection is not attempted — pnpm cannot know which bytes in a build output are a path — and normalization in Gradle's manner is a larger feature that this flag does not foreclose.

**Environment values are hashed, never recorded.** RFC 0007 denies undeclared variables from the builder profile because provenance is served to clients and a name allowlist would leak secrets. Here the rule differs and should not be copied by analogy: a declared variable's *value* goes into the key as a digest input, which is safe, and must not appear in the artifact's provenance block, which would not be. A task legitimately keyed on `NPM_TOKEN` is keyed on a digest of it; nothing published carries it.

### A task that declares no edges

The orchestration RFC defaults an undeclared `dependsOn` to `['^<own name>']`. That default is right for `build`, whose dependents do consume their dependencies' builds, and wrong for `test`, which consumes its own project's `build` and its dependencies' `build` rather than their tests. A cache cannot adopt a rule that is right by luck.

**With no `dependsOn`, the internal component of the key is the digest of what each workspace dependency exposes — its packlist — rather than any task's key.** The packlist is the file set that dependency would publish, which is exactly the surface a consumer can read, and pnpm already computes it (`@pnpm/fs.packlist`). It captures the upstream's output regardless of which task produced it, cannot drift from a declaration because there is no declaration to drift from, and over-invalidates only when the dependency's exposed files actually change.

This is the alternative this RFC declines as the *primary* mechanism, adopted exactly where the argument against it does not apply. As a primary mechanism it hashes a materialized tree where a declared edge costs a key lookup; as the undeclared fallback there is no declared edge to be cheaper than, and the choice is against a rule known to be wrong.

The consequence is the incentive this feature wants: **declaring `dependsOn` is a performance optimization, not a correctness requirement.** An undeclared task is cached correctly and more slowly; declaring its edges replaces a tree digest with a key lookup and narrows what invalidates it. A user who declares nothing gets a slower cache, never a wrong one — which is the opposite of the failure mode every task runner is complained about for.

### Declaring inputs and outputs

Configuration lives in `pnpm-workspace.yaml`, per project glob, in the shape the workspace already uses:

```yaml
tasks:
  build:
    dependsOn: ['^build']
    outputs: ['dist/**']
    env: ['NODE_ENV']
    inputs: ['+generated/schema.d.ts']
  lint:
    outputs: []
  test:
    dependsOn: ['build']
    outputs: ['coverage/**']
    inputs: ['src/**', 'test/**']
  bench:
    dependsOn: ['build']
    outputs: ['results/**']
    cache: false
```

`dependsOn` is the orchestration RFC's field; this RFC adds `outputs`, `inputs`, `env`, and `cache` to the same section rather than introducing a second one.

**Outputs must be declared, and absence is not permission.** A task with no `outputs` key is not cacheable and runs normally — it is not treated as a task that produces nothing. An explicit `outputs: []` is the positive assertion that the task produces no files, and such a task is cached for its logs and exit code alone, which is the whole point of caching `lint`. This is RFC 0007's rule for compatibility constraints applied to a different field, and for the same reason: a mistyped or forgotten key must not silently mean "cache everything about this and restore nothing".

**Two tasks may not claim the same output root.** If `build` and `bundle` both declare `dist/**`, each one's restore deletes the other's files as stale, and the failure appears as a cache that corrupts output rather than as the configuration error it is. Configuration load rejects it: within one project, two tasks whose output globs share a literal prefix — comparing the leading path segments before the first wildcard — are an error naming both tasks. `dist/js/**` and `dist/css/**` are fine and common; `dist/**` twice is not. Precedence was the alternative and is worse: it is a rule users must learn to predict which task owns a file, in exchange for permitting a configuration nobody writes on purpose.

**`cache: false` opts out without withdrawing the declaration.** A task whose `outputs` exist for the orchestration RFC's benefit, or whose output is real but not worth storing — a benchmark writing timings that differ every run — sets `cache: false` and runs every time. This is deliberately not the same as omitting `outputs`: the declaration stays, and so does everything else that reads it. Its invocation-level counterpart is `--no-cache`, which ignores existing entries for a whole run. It skips the *lookup* and not the store, which makes it the repair path as well as the bypass: a run that distrusts an entry re-runs the task and overwrites it, because a successful task is always stored. A second flag for repair would have to be named against `--force`, which the CLI already uses for something else, and would differ from this one only in a way nobody debugging a bad entry wants.

**Inputs default to the project's source files.** Every file under the project directory that git either tracks or would let you add — `git ls-files --cached --others --exclude-standard` — minus declared outputs and `node_modules`, and always the project manifest. Untracked-but-unignored files belong in the set: a source file created and not yet `git add`ed is one somebody is actively editing, and a default that skipped it would serve a stale hit for exactly the change most likely to be in flight. A workspace not in a git repository falls back to a filesystem walk honoring `.gitignore`, which is slower but not different in meaning. Explicit `inputs` entries narrow that set to the listed globs; a `+`-prefixed entry adds to the default rather than replacing it, so the common case of "the default plus this one ignored file" does not require re-listing the default.

The default is deliberately generous. An over-broad input set costs cache hits; an under-broad one produces wrong output, and a user who narrows the set has done so on purpose. A tool cannot tell the difference after the fact, so the safe direction is the default.

### Restoring into a working tree

This is the part with no precedent in RFC 0007, and it is where the work is. A dependency artifact lands in the virtual store, which pnpm owns entirely. A task artifact lands in `packages/foo/dist`, which the user owns, may have edited, and may have files in that pnpm did not put there. Three rules follow, and all three are correctness requirements rather than polish:

**Restoration must not hardlink.** The store's blobs are shared by every project on the machine. A hardlink from a CAFS blob into `dist/` means the first tool that opens that file for writing corrupts it for everything else that shares it. Task output must be restored with reflink (`clone`) where the filesystem supports it and a plain copy otherwise — `packageImportMethod`'s existing `clone-or-copy` behaviour — and `hardlink` must be refused for this path regardless of what the user configured for packages.

**A restore must remove what the previous run left.** Cache restoration that only adds files leaves a deleted-since-then output in place, and the next tool to read the output directory sees a mixture of two builds. The artifact's `deleted` list is not enough on its own, because it is a diff against the artifact's own base rather than against whatever this working tree currently holds. What is needed is a per-task record of what the last run or restore produced — and it must record **content hashes, not a path set**. A set cannot distinguish "the previous run produced it" from "the previous run produced it and the user has edited it since", so a path-keyed record overwrites and deletes exactly the user edits the next rule exists to protect (the pnpm-ci proof of concept clobbered a user's edit this way before its record grew hashes). A file is overwritten or deleted only when its current content matches what the record says was left there; anything else falls through to the next rule. The deletion of the difference stays confined, like everything else, to the declared output roots.

**A restore must not clobber a file it does not account for.** A file inside a declared output root that neither the previous run nor this artifact produced is the user's, or another tool's. Overwriting it silently is the failure mode that makes people distrust caches permanently. The restore stops with a diagnostic naming the file, and the task runs normally.

RFC 0007's manifest path validation — absolute paths, `..` segments, separators, control characters, case-insensitive collisions, unmodelled symlinks and special files, mode and size limits — applies unchanged and matters more here, since the destination is a working tree rather than the store. Additionally, and this is the constraint RFC 0007 anticipated, every restored path must resolve inside a declared output root: individually well-formed relative paths are not sufficient when the base directory is one the user cares about.

### A hit has to look like a run

A cache hit that prints nothing is indistinguishable from a task that silently did not happen, and users reasonably conclude the cache is broken. The artifact therefore carries the task's captured stdout and stderr as an ordered stream, replayed on a hit, and its exit code.

**Only successful tasks are cached.** A failing task's output directory is in an unknown state and its logs are the thing the user most needs to see produced fresh. Caching failures is a Turborepo feature and is deliberately not adopted here; it can be added later as an opt-in without changing anything.

This also disposes of watch-mode and other long-running tasks without a rule of their own. A `dev` task never completes, so it is never stored and never restored — the cache simply does not reach it. Nothing needs to detect that a task is long-running, and a `dev` script that someone has given `outputs` for another reason opts out with `cache: false`.

**Injected dependencies still need syncing.** `syncInjectedDepsAfterScripts` runs after a named script completes (`pnpm11/exec/commands/src/run.ts:469`) and copies the project's files into the `node_modules` of every project that injected it. A cache hit skips the script but must not skip the sync, or consumers of an injected dependency see the previous build. This is easy to miss and produces a bug that looks like a cache correctness failure while actually being a lifecycle one.

### The local tier needs no trust machinery

A local task cache is blobs in the CAFS plus an index from task key to manifest, log stream, and exit code. There is no owner, no signature, no compatibility tag, and no network. It is worth stating plainly that this tier is where most of the wall-clock win is — the repeated build on a developer's own machine — and that it can ship and be useful before any of the rest exists.

The index lives with the store rather than beside the workspace, and that is a decision worth stating because it is easy to get wrong: task keys are path-independent by construction, so two checkouts of the same repository at the same commit share hits, which a per-workspace cache directory would silently give up. The proof of concept keys its entry directory by workspace path and does give it up.

Artifacts are regenerable derived data and belong to the store's prunable set: `pnpm store prune` collects task artifacts whose blobs nothing else references.

That alone does not bound the index, since a monorepo's task keys turn over on every commit and every one of them stays referenced by its own entry. The bound is **age since last use**: an entry not read for a configurable period, thirty days by default, is dropped by the same prune. Age rather than a size ceiling, because a size ceiling has to choose a victim and every choice needs a policy the user cannot predict, while "you have not used this in a month" explains itself and degrades into exactly the LRU behaviour a ceiling would approximate. An entry's last-use timestamp is written on a hit, which is the only bookkeeping this requires.

### The remote tier is RFC 0007 with a different subject

Everything RFC 0007 specifies for transport and trust carries over without modification: the `{added, deleted}` manifest over CAS blobs, the signed envelope and its digest, batched lookup over `POST /-/pnpr/v0/artifacts/resolve`, per-blob reads with digest recomputation, trust policy re-evaluated at every reuse, owner-namespaced storage in pnpr's disposable `cache` root, and the split that keeps signing material out of `pnpm-workspace.yaml`.

Three of its axes simply collapse:

- **Owner is always `organization`.** A workspace task has no publisher, and the `publisher` arm of owner scope goes unused rather than gaining a new meaning.
- **Compatibility is usually `universal`.** A `tsc` or `eslint` task produces platform-independent output, and it asserts that positively. Tasks that genuinely are platform-specific use the same tag vocabulary as dependency artifacts — now a concrete one, `pnpm:v1:{linux-…-glibc,darwin-…-macos,win32-…-windows}`, specified in RFC 0007 and shipped in [pnpm/pnpm#14263](https://github.com/pnpm/pnpm/pull/14263) — which is why the vocabulary is worth having even though most tasks will never use it. Note that those tags encode a Node major, which the task key already covers through its runtime fingerprint; for a task the tag's job is the OS floor, and the duplication is harmless but should not be mistaken for the key's account of the runtime.
- **Eligibility is not a question.** RFC 0007 needs a package eligibility contract because the build being cached belongs to a third party. A workspace task is first-party by construction, so the "named owner who accepts the residual risk" is the same organization running the install, and the argument that section makes is satisfied trivially.

RFC 0007's two then-unfinished items have both since shipped, and they land here unevenly.

**Persistent quarantine** ([pnpm/pnpm#14259](https://github.com/pnpm/pnpm/pull/14259)) is inherited whole and matters more here than it does for dependencies: task artifacts are fetched on every run, so a poisoned blob re-fetched every time is a permanent cost rather than an occasional one. Its per-channel retention of the newest sixty-four envelope digests needs no change for a task subject, and the re-verification of a persisted artifact before every reuse is what makes a local hit on a remotely-sourced artifact safe.

That reuse is currently scoped to the channel it came from: a persisted entry is a miss once the configured `pnprServer` changes. RFC 0007 records this as a conservative narrowing pending multi-channel support, and it should stay the narrowing here too rather than being relaxed for tasks. The cost is different in kind, though, and worth stating: a dependency artifact discarded on a channel change is rebuilt once, while a workspace's whole task cache going cold on a server rename is the sort of thing that gets blamed on the cache being unreliable. Whatever multi-channel support decides, tasks want the answer at the same time.

**Lockfile pinning** ([pnpm/pnpm#14243](https://github.com/pnpm/pnpm/pull/14243)) does not carry over, and **task artifacts are unpinnable**. A pin is recorded *on a package snapshot* as `artifactPins[inputKey][owner][platformFingerprint] = envelopeDigest`, and a workspace task has no snapshot; that is the mechanical obstacle, but it is not the reason.

The reason is what a pin defends. It closes the window in which a compromised signing key serves different bytes for an input that cannot change — a dependency version published years ago, which the consumer cannot rebuild and can only accept or refuse. A workspace task inverts every term. Its input is first-party and mutable, the organization running the install is the same organization that signed the artifact, and the fallback is not "trust it or fail" but "build it, which is what you would have done anyway". Pinning would record a digest defending the workspace against itself, expire on the next source edit, and need a repopulating command with nothing to repopulate between edits.

What would reopen this is a task artifact signed by someone the consumer is not: a publisher-owned task cache, which does not exist and which the owner-scope collapse above rules out by construction. If it ever does, this section is where the argument has to be redone rather than extended.

### The subject, in the shipped protocol

RFC 0007 requires that "the protocol must not embed package identity in the shape of a request or response key — a request assuming `(package, version)` cannot carry `(project, task, hash)`". The v0 implementation honoured that for the key and the response but not for the request or the signed payload: `ArtifactCandidate` and `ArtifactPayload` both carried `package` and `sourceIntegrity` as required top-level fields next to the opaque key.

**This has shipped**, ahead of everything else here and while it was still free, in [pnpm/pnpm#14267](https://github.com/pnpm/pnpm/pull/14267). Both types now carry a kind-discriminated `subject` — `{ kind: 'dependency-side-effects', package, sourceIntegrity }` against `{ kind: 'workspace-task', project, task }` — in `shared-artifact-protocol` and `@pnpm/pnpr.client` alike. pnpr was unaffected, as expected: `resolve_candidate` looks up by key and owner.

Two rules the implementation settled, which this RFC now inherits rather than proposes:

- **The artifact kind, the key prefix, and the subject must agree.** A payload declaring `workspace-task:v1` carries a `workspace-task` subject and an input key prefixed `workspace-task:v1:`; a mismatch is a malformed envelope, not a lookup miss.
- **A `workspace-task` subject with a `publisher` owner is rejected outright.** The "owner is always `organization`" observation above is enforced at validation rather than left to convention.

## Rationale and Alternatives

**Do nothing; let Turborepo and Nx do this.** The honest baseline, and it is not obviously wrong — both tools work, both are widely deployed on pnpm workspaces, and neither is asking for help. The case against it is that they solve the problem with less information than pnpm has, most visibly in lockfile-hash invalidation, and that the caching half of their job is now a small delta over machinery pnpm shipped anyway. The case *for* it is real and should not be waved away: this is the first thing in pnpm's scope that competes with a tool many of its users already run and like, and shipping a worse version of it would be worse than not shipping one.

**Be a remote-cache backend instead of a cache.** Turborepo's remote cache API is a keyed blob `PUT`/`GET` with optional HMAC signing; pnpr could implement it and be the self-hosted cache those users already want, without pnpm gaining a task cache at all. This is cheap — genuinely a small amount of work on top of what pnpr already stores — and it is not exclusive with this RFC. It is worth doing on its own merits, and it is the fallback if the answer to the previous alternative is "do nothing".

**Cache without a task graph, using package topology.** This was this RFC's original scope, and it is wrong rather than merely limited: as shown above, keying on the same-named upstream script produces silent stale hits whenever an upstream project lacks that script. The near-miss is instructive — with the generous default input set, the change usually propagates transitively anyway, so the bug would have surfaced late, in workspaces that had narrowed their `inputs`, which are exactly the workspaces that tuned their cache and trust it most.

**Hash the materialized content of workspace dependencies instead of naming upstream tasks.** Include whatever is in `node_modules/<workspace dep>` in the task's inputs, and the graph becomes unnecessary for correctness — it captures what the upstream produced regardless of which task produced it, and cannot drift from a declaration. The ordering objection does not apply, since the upstream must have been built before this task runs anyway. It is declined as the primary mechanism because it hashes an entire materialized tree on every miss where a declared edge costs a key lookup, and because it cannot distinguish the part of an upstream package a task actually consumes. It is **adopted as the fallback for a task that declares no `dependsOn`**, in the narrowed form specified above — the dependency's packlist rather than its whole materialized tree — where it is strictly better than assuming the same-named script, and it is worth keeping as an opt-in check that declared edges are complete.

**Reuse the Bazel Remote Execution API.** RFC 0007 declines REAPI because its lookup is exact-digest and dependency artifacts need floor-based compatibility selection. That objection is materially weaker here: a task is almost always `universal`, so exact-digest lookup is exactly the right model, and REAPI's `Action`/`Command` shape fits a task far better than it fits a dependency build. The reason to decline it anyway is not technical merit but coherence — running two protocols for two artifact kinds that share a store, a trust model, and a client would cost more than the standard buys. If pnpm ever supports REAPI, it should be for both kinds or neither.

**Cache into a project-local directory rather than the store.** Simpler, and it is what most task runners do. It gives up file-level deduplication across projects and tasks, which is the main structural advantage pnpm has here, and it adds a second thing to garbage-collect. Declined.

## Implementation

**Protocol.** Shipped in [pnpm/pnpm#14267](https://github.com/pnpm/pnpm/pull/14267): the discriminated `subject` on `ArtifactCandidate` and `ArtifactPayload` in both `shared-artifact-protocol` and `@pnpm/pnpr.client`, with the kind, key-prefix, and owner rules enforced on both sides.

**Input hashing.** A git-backed enumerator and hasher for a project's tracked and untracked-unignored files, with a `.gitignore`-honouring filesystem fallback. `git-utils` and `crypto-hash` exist on the Rust side, `@pnpm/crypto.hash` and `@pnpm/fs.packlist` on the TypeScript side. This is the component whose performance decides whether the feature is worth using: it runs over every selected project on every invocation, including the misses.

**Configuration.** The `tasks` section in `pnpm-workspace.yaml`, per project glob, with `outputs` mandatory for cacheability and `inputs`/`env`/`cache` optional. Both stacks validate task fields against a known-key set so a typo is an error rather than a silent no-op — `TASK_SETTING_FIELDS` in `pnpm11/config/reader/src/getOptionsFromRootManifest.ts:110` and the flatten-unknown check in `pnpm/crates/config/src/workspace_yaml.rs` — and both must learn the new fields in the same change, or valid configuration is rejected as a mistake.

**Key production.** `workspace-task:v1:` over the components listed above, reusing `calcDepStateInputKey`'s graph hash for the dependency component and each workspace dependency's packlist digest for a task that declares no edges. Byte-identical across both stacks, like the platform fingerprint, since a divergence means the two CLIs disagree about what is cached.

**The scheduler's graph scope.** The shipped scheduler builds tasks for the projects the run selected. It needs the requested-versus-contained split the key demands: the project map it is given is the dependency closure, and a separate list names the projects whose tasks the invocation actually requests, with the rest participating through `dependsOn` edges alone.

**The runner integration.** A lookup before each task, a capture after a successful one, and log replay on a hit, against the scheduler the orchestration RFC introduced. A task's key is not computable until the keys of the tasks it depends on are, so key production is itself a traversal of the task graph in the same order the scheduler walks it — which is an argument for computing keys inside the scheduler rather than in a pass before it.

**Working-tree restoration.** The new importer path: reflink-or-copy, output-root confinement, the previous-output record and stale-file deletion, and the refusal to overwrite unaccounted files.

**Remote tier.** The existing client, with the task subject and `universal` compatibility. Almost nothing new.

**Both stacks, per the repository's parity rule**, with the key producer as the piece where divergence would be least visible and most damaging.

Ordering: the orchestration RFC's scheduler and the protocol's `subject` have both shipped, which was the right order and leaves this RFC's own work to begin at the graph's closure and the key. Then input hashing and configuration, then the local tier end to end, then the remote tier. The local tier is independently shippable and independently valuable, and it exercises the restoration path — which is where the correctness risk lives — before any bytes cross a machine boundary.

## Prior Art

**Turborepo** is the direct model and the direct competitor. `inputs`, `outputs`, `env`, `dependsOn`, log replay on a hit, and cache-only-on-success are all its design, and this RFC adopts most of them because they are right. Two of its decisions are not adopted: caching failed tasks, and hashing the lockfile as a proxy for dependency changes. The second is the one pnpm can straightforwardly beat.

**Nx** demonstrates that a task cache and a task graph are sold together for a reason — the cache is only as good as the graph's account of what feeds what — which is why this RFC gave up on being independent of one and took a dependency on the orchestration RFC instead.

**Gradle's build cache** is the most instructive prior art on the part everyone underestimates: input *normalization*. Gradle has explicit runtime-classpath normalization, ignored-file declarations, and relocatability checks, because naive hashing of every input over-invalidates until the cache stops paying for itself, and because a build that embeds its own path is not shareable. Both problems are in this RFC's unresolved list, and Gradle's experience says they arrive sooner than expected.

**ccache and sccache** are the long-running evidence that a content-addressed compilation cache is worth its complexity, and also that the hard part is deciding what counts as an input — `ccache`'s direct-versus-preprocessor mode distinction is precisely this trade-off, made twice because neither answer is right for every project.

**RFC 0007** is the immediate prior art and supplies the transport, the signing, the trust model, and the artifact format. Its forward-compatibility section anticipated this document; the constraints it recorded there are the reason this RFC is mostly a key producer and a restorer rather than a protocol.

## Unresolved Questions and Bikeshedding

- **Input hashing cost.** The only question here still waiting on evidence rather than a decision. The hasher runs over every selected project on every invocation, so a cache miss must not be slower than the build it was trying to skip. The design is settled — the fast path git itself uses, where `git ls-files -s` returns index blob hashes for free and only stat-dirty files are hashed, the same trick that keeps `git status` fast. What wants measuring before the default input set is final is the residue that path does not cover: untracked-but-unignored files, which have no index entry and are hashed every run, and how often real checkouts carry stale stat information. The pnpm-ci proof of concept implements the naive form and is where the cost is visible.
- **Interaction with `pnpm deploy`.** Deploy consumes a project's built output and could restore it rather than build it, but it does not participate in the task graph today, so there is nothing to key it on. Deliberately out of scope here: it is a question about giving `deploy` a task identity, not about the cache.
- **Verifying declared edges.** A task that reads an upstream output it never declared is a silent stale-hit generator. This is now a smaller question than it was, because an undeclared task is keyed on its dependencies' packlists and is correct without any declaration at all — the exposure is narrowed to a task whose declared edges are *wrong*, rather than any task whose edges are missing. Whether pnpm can cheaply detect that, by comparing declared edges against the packlist digests the undeclared path already computes, is worth building as an opt-in audit and is not a prerequisite.
