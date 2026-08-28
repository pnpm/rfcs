# pnpm as the CI engine

## Summary

A CI pipeline for a JavaScript workspace is, almost entirely, a re-statement of things pnpm already knows: which projects exist, what order their tasks run in, what toolchain they need, and what has changed. This RFC adds a `pnpm ci` command that makes pnpm the engine a CI run delegates to — one invocation that enforces a fully pinned environment, selects work from a git base, schedules the task graph, reuses cached results, and emits a machine-readable account of what happened. The CI *host* — GitHub Actions, GitLab, Jenkins, a cron job — keeps the jobs this RFC does not take: reacting to events, provisioning machines, holding secrets, and drawing the UI. A companion RFC, [the pnpm CI server](./0000-pnpm-ci-server.md), proposes moving all of those except the machines onto pnpr; this document is complete and independently ratifiable without it.

It is the third RFC in a sequence and proposes little machinery of its own. [Workspace task orchestration](https://github.com/pnpm/rfcs/pull/23) supplies the graph and the scheduler; the [workspace task cache](https://github.com/pnpm/rfcs/pull/22) supplies skipping and replay; [RFC 0007](./0007-shared-side-effects-cache.md) supplies the remote transport and trust model. What this RFC adds is the entry point, the contract, and the policy: what "run CI" means precisely enough that every workspace stops re-deriving it in YAML.

## Motivation

Look at what a real CI workflow file for a pnpm monorepo contains. A checkout step with a hand-tuned fetch depth, because someone learned that affected detection needs history. A Node version, restated from wherever the repository actually pins it, drifting independently. An install step, hopefully frozen. A cache configuration for a bolted-on task runner, with its own signing secret. Then the actual pipeline: an ordered list of `run` invocations that encodes, by hand and by barrier, a dependency structure the workspace's own graph already expresses — `build`, *wait for everything*, `test`, *wait for everything*, `lint`. Every one of those lines is a second copy of information pnpm holds natively, and every one of them rots separately.

The two RFCs this one builds on remove the worst of it: with a task graph and a cache, the pipeline body collapses to an install and a `pnpm run`. The remaining gap is not mechanism but *definition*. What base does an affected run diff against, and who fetches the history to compute it? Is the toolchain the one the lockfile pins or the one the runner image happened to ship? May this run write to the shared cache, or only read it? What does the host's status check get, other than one exit code for a thousand tasks? Today each workspace answers these in YAML, differently, and the answers are where CI configuration actually breaks. A CI engine is precisely the thing that owns those answers.

There is also a straightforwardly strategic motivation, stated openly. pnpm already ships a task scheduler, a content-addressed store, a remote artifact protocol with signing and trust policy, a registry server to host it, and — unique among tools in this space — a lockfile that pins not only the dependency graph but the package managers and runtimes that operate on it. Turborepo and Nx built businesses on the observation that CI for JS monorepos is mostly graph-plus-cache. pnpm holds more of the necessary information than either, first-hand rather than inferred. Declining to compose the pieces it already has is a choice, and it should at least be a deliberate one.

### What already exists

- **Selection.** The filter language already answers "what changed": `--filter='...[origin/main]'` selects projects affected since a ref, with `--changed-files-ignore-pattern` to exclude noise, and traversal through the full workspace graph so an unselected project in the middle of a chain does not break it.
- **Scheduling.** The orchestration RFC's per-task scheduler, `dependsOn`, cycle detection, the four task states, and the `--dry-run --json` graph with stable task identities.
- **Skipping and replay.** The task cache RFC's keys, restoration rules, and log replay; its remote tier over RFC 0007's signed artifacts and pnpr's `/-/pnpr/v0/artifacts` endpoints.
- **Environment pinning.** `frozen-lockfile` already defaults to true when a CI environment is detected. The lockfile pins the dependency graph; pnpm 12 additionally records the package managers and runtimes a workspace depends on, and `executionEnv` pins a project's Node version. The materials for "this run's environment is exactly what the repository states" all exist.
- **Hosting.** pnpr already stores organization-scoped artifacts with per-owner quotas, policy, and access control, and RFC 0007 already separates the right to read artifacts from the separately granted right to publish them — which is exactly the read/write split a CI cache policy needs.

## Detailed Explanation

### A CI system, taken apart

"CI engine" is a bundle, and the proposal only makes sense unbundled. A CI system (1) reacts to events, (2) provisions machines, (3) holds secrets, (4) reproduces an environment, (5) decides what to run, (6) runs it in order, (7) skips what hasn't changed, (8) reports what happened, and (9) distributes work across machines.

This RFC assigns (4) through (8) to pnpm, because each is a function of information pnpm holds first-hand, and leaves (1) through (3) with the host. The companion CI-server RFC proposes taking (1), (3), and the presentation half of (8) onto pnpr, together with the coordination half of (9), whose forward-compatibility constraints are recorded below. (2) stays with the operator in every version of this plan: the machines are always the organization's own, and pnpm's ambition on them stops at running an agent. What no RFC in this sequence proposes is pnpm provisioning compute.

The resulting division is legible in the workflow file. The host's job shrinks to: get a machine, check out the commit, expose secrets, run `pnpm ci`, and render what it reported. That skeleton is identical across hosts, which is the point — the pipeline's *meaning* lives in the repository, in one file, executed identically on a laptop and on any runner.

### The command and the pipeline

Configuration lives in `pnpm-workspace.yaml`, next to the `tasks` section it builds on:

```yaml
tasks:
  build:
    dependsOn: ['^build']
    outputs: ['dist/**']
  test:
    dependsOn: ['build']
    outputs: ['coverage/**']
  lint:
    outputs: []

ci:
  base: origin/main
  pipelines:
    check: [lint, build, test]
    release: [build, test, publish-check]
```

A pipeline is a named set of task requests. `pnpm ci check` requests `lint`, `build`, and `test` in every selected project; the task graph — `dependsOn`, `^` expansion, missing-script pass-through — does everything else, exactly as if each had been requested by `pnpm -r run`. A pipeline deliberately has no ordering of its own: `[lint, build, test]` is a set, and any ordering among its members is `dependsOn`'s job. Giving pipelines sequence would reintroduce, one level up, the hand-written barriers the orchestration RFC exists to remove — the current `&&` chains between `-r run` invocations are the workspace-wide barrier this RFC inherits the duty to kill, and a pipeline is exactly that chain with the barrier deleted.

The command's fixed semantics, each the answer to a question every workspace currently answers in YAML:

- **The install is frozen, always.** `pnpm ci` performs an install with `frozen-lockfile` unconditionally — not defaulted on, not overridable within the command. A CI run whose lockfile does not match its manifests is reporting on a tree nobody has, and the correct response is the existing frozen-lockfile failure, immediately.
- **The toolchain is the pinned one, always.** Tasks run under the runtime the repository records — the lockfile's package-manager dependencies and per-project `executionEnv` — never under whatever the runner image ships. A workspace that pins nothing gets the ambient runtime and a warning naming what was unpinned, because an engine that silently varies with the runner image is reproducing the bug this section exists to fix. This is also load-bearing for the cache: the task key includes the resolved runtime version, so an unpinned runtime does not corrupt anything — it just misses, runner image by runner image, and the warning tells the user why.
- **The run continues past failure.** `pnpm ci` behaves as `--no-bail`: every task whose dependencies passed still runs, failures accumulate, dependents of failures are `skipped`, and the exit code is non-zero iff at least one task `failed` — the orchestration RFC's states and exit contract, unchanged. Bail is the right default for a developer's inner loop, where the goal is the fastest route back to editing. A CI run's goal is a complete account: a contributor who gets one failure per push, fixes it, and discovers the next is being charged a round trip per bug.

`pnpm ci` with no argument runs the pipeline named `default` if one exists, and fails with the available names otherwise. And the command is a strict composition: everything it does is expressible as filter arguments plus recursive run under the two prior RFCs. That is a feature. `pnpm ci` is a *contract*, not a capability — a workspace can always drop back to the underlying flags, and the command's value is that nobody has to.

### Selection is an optimization, not the correctness boundary

The instinctive model of an affected-only CI run — pnpm's `--filter='...[base]'`, Nx's `affected` — treats selection as the correctness mechanism: run what changed, trust that the rest didn't matter. That model is fragile exactly where it is relied on, because selection sees project topology, not task inputs — a root config edit selects everything or, worse, nothing.

With the task cache underneath, the burden moves. The correct mental model for `pnpm ci` is: **the pipeline always covers every project; the cache decides what actually executes.** A task whose key is unchanged is a hit whether or not any selection would have picked it, and a task whose key changed runs even if a coarse diff heuristic would have skipped it. Selection survives as a *pre-pass* — computing keys for ten thousand tasks costs real time, and pruning the graph to projects the diff can reach makes the common PR cheap — but a selection mistake now costs seconds of key hashing, not a stale green check.

For selection-as-pre-pass to be sound, the pruned set must be a superset of every task whose key could have changed. Project-graph reachability gives that for everything except inputs outside any project — the root lockfile, the workspace file, a shared tsconfig pulled in by relative path. A diff touching files not attributable to selection's model of the world therefore disables pruning for that run and falls through to the full graph, where the cache still does its work. Slow and right beats fast and green.

Soundness constrains the *shape* of the pruned graph too, in a way the proof of concept had to learn by measuring: **the task graph must span the dependency closure of the affected set, with only the affected projects' tasks requested.** Selecting changed-plus-dependents alone and building the graph over that subset fails twice. A task's cache key includes its upstream task keys, and if `^` edges resolve only within the selection, the same task keys differently in a narrowed run than in a full one — at best spurious misses, at worst an edge to an unselected dependency vanishing from the key entirely, and with it the only representation of that workspace dependency's content the key has. And on a fresh machine, a selected project's build needs its unselected upstream's output to exist at all. Both resolve the same way: the affected set's transitive dependencies are included in the graph so `dependsOn` can pull their tasks in as non-requested nodes (an upstream `build` without its `lint` or `test`), which in the steady state are cache hits. This is also Turborepo's shape, arrived at independently and presumably for the same reasons.

Two git realities become pnpm's job the moment it owns selection, because every workspace currently rediscovers them:

- **The base is the merge base.** Diffing a PR against the tip of `origin/main` reports changes that landed on main after the branch point as if the PR made them. `pnpm ci` resolves `ci.base` to `merge-base(HEAD, base)` — three-dot, not two-dot — and says so in the run report.
- **Shallow clones are the default, and history is the engine's dependency.** CI checkouts default to depth 1, under which the merge base does not exist. `pnpm ci` deepens the fetch until the merge base is reachable rather than failing or silently diffing against the wrong commit. This is the single most common way affected-CI setups are quietly broken today, and an engine that documents the pitfall instead of absorbing it has declined its job.

### Cache policy is part of the engine

A shared task cache in CI has an asymmetry that the configuration must be able to express, because getting it wrong is how a cache becomes an attack surface: **everyone may read; only trusted runs may write.**

The concrete threat is the fork pull request. A PR's tasks execute the PR's code — that is what testing means — so a cache-writing PR run lets any contributor manufacture a green, replayable entry for a task key that the post-merge run on `main` will then hit. Poisoning the artifact, poisoning the logs, or simply caching a success that never happened are all the same move. The unedifying conclusion every cache-in-CI deployment reaches independently: untrusted runs read, trusted runs (the main branch, a merge queue, release tags) write.

This RFC adds no mechanism for it, because RFC 0007 already built the split: publishing artifacts is a separate grant from reading them, held as signing material that is refused in the committed workspace file and lives only in the trusted machine's own configuration. Fork PRs are read-only by construction — the runner simply holds no signing key — and no `pnpm ci` flag can override an authorization that operates by possession. What this RFC adds is the statement that this is the intended deployment shape, and one honest caveat inherited from RFC 0007: reading is not risk-free either, since a cache hit restores files and replays logs produced elsewhere. The trust boundary is the artifact signature and the organization's own key discipline, per RFC 0007's trust section, and a workspace that wants PRs fully hermetic sets the remote cache read-only off for untrusted refs at the host level.

One free lunch is worth naming: **CI log storage falls out.** The task cache RFC stores each task's ordered output stream and exit code in the artifact. A CI run that publishes its results has therefore already durably stored its logs, per task, content-addressed and deduplicated, on infrastructure the organization already operates. "Where do build logs live" is a product pillar for CI vendors; here it is a byproduct.

### Reporting

One exit code is the interface `pnpm ci` has with the host's gate, and it follows the orchestration RFC's rule. Everything richer is data, not prose: `pnpm ci` emits, alongside human-readable output, a structured event stream — one JSON object per line — and a final summary document.

The vocabulary already exists. The orchestration RFC's `--dry-run --json` defines the task identity (workspace-relative project directory plus script name) and the resolved edges; the events reference the same identities. A task's lifecycle events carry its state (`passed`, `failed`, `skipped`, `not run`), its cache disposition (hit or miss, local or remote, and the key), timing, and exit code. The summary aggregates the same, plus the resolved base, the selection that was applied, and the environment pins in effect.

What this buys is per-task integration for the price of a twenty-line adapter: a GitHub check run per task rather than one opaque "CI" check, an annotation on the failing project instead of a log-dive, a comment that says "142 tasks, 139 cache hits, 2 passed, 1 failed" with the failure's replayed log attached. The adapters themselves — the GitHub Action, the GitLab template — are deliberately out of tree. They version with the hosts, not with pnpm, and the stable surface between them is the event stream, which is versioned like any other machine-readable pnpm output. Publishing an official thin action is worthwhile and is not part of this RFC.

Cache dispositions in the events answer the question that actually gets asked — "why did this run?" — with the key components that differed, in the terms the task cache RFC defines. The first debugging experience with any cache determines whether a team keeps it, and an engine that cannot explain a miss is indistinguishable from one that missed for no reason.

### What this does not add

No triggers, no runners, no secret store, no web UI in this RFC — those are the companion CI-server RFC's subject, and they are a *split*, not an exclusion: this document stands alone, and the server tier consumes its contract. One thing is excluded from the whole sequence: a YAML dialect for expressing arbitrary jobs. A step that is not a workspace task — building a Docker image, deploying, posting to Slack — belongs to the host, or in a script a task runs. The moment `pnpm ci` grows an escape hatch for "arbitrary command with arbitrary dependencies on non-task things", it has begun re-implementing the host's job description, and the host is better at it. This boundary is what keeps the proposal a composition of two shipped RFCs rather than a product.

Distributed execution — one logical run fanned out across machines — likewise moves to the companion RFC rather than being declined. The economics only exist because of the cache (an agent's completed task is an artifact any other agent can restore), and the coordination problem is real work: leases, work stealing, partial failure, a scheduler that spans machines. It should not gate this document, and this document must not foreclose it.

Four constraints are recorded now to guarantee that, in the manner of RFC 0007's forward-compatibility section:

- **Run identity exists from day one.** The summary document names the commit, the pipeline, the resolved base, and every task key. A distributed run is this document with tasks executed by more than one machine; nothing in the format may assume a single executor.
- **Task keys are computable without executing.** The task cache RFC computes keys as a traversal of the graph in scheduler order; a coordinator partitioning work needs exactly that computation and no more. Key production must stay separable from execution.
- **"Affected since last green" needs a results store, and pnpr is where it would live.** Selecting against the last commit whose pipeline passed — rather than against the branch tip — requires someone to remember run outcomes by commit and pipeline. That is a small, append-only record, it is organization-scoped, and pnpr is the natural host; RFC 9's pnpm agent already establishes the pattern of pnpr doing per-organization work adjacent to the artifact store. The companion RFC proposes exactly that store; the summary document is deliberately sufficient as what it stores.
- **The server never becomes the root of trust.** Everything pnpr does today keeps its compromise bounded by verification the client performs — a compromised registry cannot substitute lockfile-pinned bytes, cannot sign artifacts, cannot unmask denied packages. Any control-plane tier built on this RFC's contract must preserve that pattern: secrets it holds are ciphertext sealed to agent identities or references into stores it cannot read, and nothing an agent restores or executes is accepted on the server's word alone. This is the condition under which the host tier is acceptable at all, and it is recorded here so the companion RFC answers to it rather than to convenience.

## Rationale and Alternatives

**Do nothing beyond RFCs 22 and 23.** The honest baseline, and it is strong: once those land, `pnpm install --frozen-lockfile && pnpm -r run test` in any host is already a cached, graph-scheduled CI run, and this RFC adds no capability that composition lacks. What it adds is the removal of a hundred small decisions — merge-base semantics, fetch depth, toolchain enforcement, bail behavior, read/write cache split, result format — each of which is currently made per-workspace, in YAML, wrongly often enough to be the actual failure mode of monorepo CI. Conventions are cheap to skip and expensive to lack; the counterargument is that a convention can also be a documentation page, and the response is that a documentation page cannot deepen a shallow clone or refuse an unfrozen install.

**Ship host adapters instead of a command.** An official GitHub Action encoding the same decisions would serve the largest host population without touching the CLI. Declined as the primary shape because it inverts the portability: the meaning of the pipeline would live in host-specific packaging, N times, and a laptop could not run what CI runs. The adapters are wanted — as thin translations of the event stream, on top of the command.

**Own more: triggers, runners, a hosted product.** The Nx Cloud / Vercel trajectory, and where the commercial gravity in this space points. Split rather than declined: the control plane — triggers, secrets delivery, results store and UI, agent coordination — is proposed in the companion CI-server RFC, on the pnpr instance organizations already operate; owning fleets of machines, or a hosted multi-tenant service, remains outside every RFC in the sequence. The split keeps this document ratifiable on its own — nothing here depends on the server tier, and the server tier consumes this one's run summary and event stream as its stored record.

**Sit on an existing engine — Dagger, Earthly, Bazel's ecosystem.** Each brings container-grade hermeticity that this proposal lacks and a price this proposal exists to avoid: a second build-definition language, a second graph, and a model where pnpm is an opaque step inside someone else's cache boundary. The entire advantage argued here is that pnpm's own graph, store, and lockfile are the finest-grained truth available; delegating to an engine that cannot see them discards exactly that.

## Implementation

Both stacks, per the repository's parity rule, and almost entirely composition:

- **Configuration.** The `ci` section — `base`, `pipelines` — parsed and validated with the same machinery as `tasks`, with unknown-key handling per the existing settings strictness rules.
- **The command.** `pnpm ci <pipeline>`: frozen install, base resolution (merge-base computation and fetch-deepening in `git-utils` / the TS equivalent), selection pre-pass with the outside-any-project fallthrough, then multi-task requests dispatched into the orchestration scheduler with `--no-bail` semantics. The scheduler and cache are consumed exactly as shipped; the one new demand on them is requesting several named tasks in one invocation, which the task graph already represents and `run`'s CLI surface merely never asked for.
- **The event stream and summary.** A versioned serialization of state the scheduler and cache already hold. The dry-run JSON's identity vocabulary is reused verbatim.
- **Toolchain enforcement.** Wiring the run's execution environment to the lockfile's package-manager pins and `executionEnv`, plus the unpinned-runtime warning.
- **pnpr.** Nothing. The read/write split, quotas, and policy shipped with RFC 0007; this RFC only documents the deployment shape.

Ordering: the command with selection and reporting first — it is useful with only a local cache, and it exercises the contract end to end. Cache-policy documentation and the reference host adapter alongside. The results store and distribution, if ever, as their own RFCs.

**A proof of concept exists**, Rust-only, on the `pnpm-ci-poc` branch: `pnpm pipeline` with the `pipelines`/`pipelineBase` settings, frozen install, merge-base selection with the root-change fallthrough, the multi-task graph under `--no-bail` semantics, a local-tier task cache with log replay and guarded restoration, and the event stream and summary. The same branch carries the companion RFC's tier 1 — pnpr run-record endpoints and a viewer, fed by `pnpm pipeline --report` — whose lessons are recorded there; the one that reaches back into this document is ordering: the run is submitted before the failure exit is raised, so a red run is recorded too. Besides the naming and dependency-closure findings folded in above, it surfaced one correction that belongs to the task cache RFC (pnpm/rfcs#22): the per-task record of what the last run produced must carry **content hashes, not a file set** — a set cannot distinguish "the previous run produced it" from "the previous run produced it and the user has edited it since", and the PoC's restore overwrote a user's edit before the record grew hashes. The same rule guards stale-file deletion: a stale output the user has modified is theirs now, and the restore refuses rather than deleting it.

## Prior Art

**Turborepo** demonstrated the collapsed pipeline this RFC wants (`turbo run` as the only CI step) and its remote-cache CI deployment converged on the same read/write asymmetry argued above. It also demonstrates the limit this RFC inherits: an engine below the container boundary cannot offer hermetic execution, only honest keys.

**Nx and Nx Cloud** are the full trajectory: `affected` selection, then caching, then distributed task execution and a hosted results UI. Nx's own migration of `affected` from correctness mechanism to optimization — precisely the "cache decides, selection prunes" model above — is the strongest external validation that the burden belongs on keys. Nx Cloud's agents are the reference for what the deferred distribution RFC would build.

**Bazel with BuildBuddy or Buildbarn** is the mature end state: remote cache, remote execution, and a result store (the Build Event Protocol) feeding a UI. The Build Event Protocol in particular is prior art for this RFC's event stream — the lesson being that the *stream* is the stable product surface and every UI is a consumer of it.

**Gradle Enterprise build scans** made the case that a per-run, durable, shareable record of what executed and why is worth more to debugging than live logs. The summary document is that record, kept deliberately small.

**GitHub's merge queue** is the trusted-ref pattern the cache-write policy assumes: a serialized, post-approval lane whose runs are exactly the ones that should hold signing material.

**`npm ci`** is prior art of an awkward kind: it established, ecosystem-wide, that `ci` after a package manager's name means "clean frozen install", not "run my pipeline". The collision is real and is bikeshedded below rather than waved off.

## Unresolved Questions and Bikeshedding

- **The name.** Settled by the proof of concept, and not in this document's favor: `pnpm ci` is not merely muscle-memory-adjacent to `npm ci` — it *is* the clean-install command in both stacks today (aliases `clean-install`, `ic`, `install-clean`), and `ci` is additionally a taken *setting* name (the CI-detection boolean), so the `ci:` configuration section proposed above collides too. The PoC ships as `pnpm pipeline <name>` with flat `pipelines:` and `pipelineBase:` settings, which read well in practice; this document keeps `pnpm ci` in its prose as the concept's name, but the ratified command and section names must be the non-colliding ones, with `pipeline`/`pipelines` as the working candidates.
- **Matrix runs.** "Test on three OSes" is standard CI and sits uneasily on the cache: a `test` task whose artifact is `universal` would let a Linux pass satisfy the macOS leg of the same matrix, which is exactly wrong for the platform-sensitive tests the matrix exists to catch. The task cache RFC's compatibility vocabulary can express per-platform keys; whether matrix legs must declare non-`universal` outputs, or `pnpm ci` should key matrix pipelines by platform fingerprint implicitly, needs deciding before anyone's matrix silently collapses to one leg.
- **Flaky tests and retries.** CI engines retry; the cache stores first successes. A flaky test that passes on retry is cached as a clean pass and replayed until its key changes, which converts "flaky" into "invisible". Whether `pnpm ci` offers per-task retry at all, and whether a passed-on-retry result should be cacheable, is open — the conservative answer (no retries; flakiness is the workspace's bug) is defensible and probably unpopular.
- **Secret-bearing env in keys.** The task cache hashes declared env values into the key, so a routine token rotation busts every key that declared the token. CI is where tokens rotate. A declaration that a variable must be *present* but not *keyed* (needed to run, asserted not to affect output) would fix it and is one more way for a user to lie to the cache; this belongs with the task cache RFC's normalization questions but CI is where it will be hit first.
- **Live logs.** Replayed-on-hit logs and an event stream serve the summary well; a developer watching a 40-minute run wants streaming output *now*, and piped-and-prefixed output is already the orchestration RFC's answer. Whether the event stream should carry incremental log chunks (making richer live host UIs possible) or only terminal-per-task results is open.
- **`ci.base` per pipeline.** A `release` pipeline plausibly wants a different base (previous tag) than `check` (main). Whether `base` belongs at the `ci` level, the pipeline level, or both-with-override is pure bikeshed.
- **Root-level work.** The orchestration RFC excludes root-project tasks; pipelines inherit the exclusion, and real pipelines have workspace-level steps (a repo-wide typecheck, docs build). This RFC takes no position and notes that whichever RFC lifts the exclusion, pipelines get it for free.
