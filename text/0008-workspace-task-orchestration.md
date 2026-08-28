# Workspace task orchestration

> **Status: implemented.** Shipped in both the TypeScript CLI and pacquet: the per-task scheduler and `dependsOn` in [pnpm/pnpm#14209](https://github.com/pnpm/pnpm/pull/14209), extracted to a shared module in [pnpm/pnpm#14215](https://github.com/pnpm/pnpm/pull/14215), the chunk loops retired in [pnpm/pnpm#14221](https://github.com/pnpm/pnpm/pull/14221), per-task concurrency in [pnpm/pnpm#14229](https://github.com/pnpm/pnpm/pull/14229), bail cancellation in [pnpm/pnpm#14251](https://github.com/pnpm/pnpm/pull/14251), and `--resume-from`'s run-state journal in [pnpm/pnpm#14258](https://github.com/pnpm/pnpm/pull/14258). This document describes what shipped, with one exception marked where it occurs: `persistent` is specified and not yet implemented. Both questions it originally left open — per-task concurrency limits and persisted run state — were answered by the implementation and are specified in the body rather than deferred; the one rule that did not survive contact with real scripts is called out where it occurs.

## Summary

`pnpm -r run` computes a real dependency graph and then throws most of it away, flattening it into topological chunks separated by barriers. This RFC replaces the chunk barrier with per-task scheduling, and adds a `dependsOn` declaration so a task can depend on something other than the same-named script in its workspace dependencies. Both are changes to *which tasks may start when*. Neither adds a cache; the workspace task cache RFC ([pnpm/rfcs#22](https://github.com/pnpm/rfcs/pull/22)) builds on this one.

## Motivation

pnpm already has the parts of a task runner that are hard to retrofit: a filter language with dependency and dependent traversal and git-diff selectors, topological ordering computed through the full workspace graph, `--workspace-concurrency`, `--bail`, `--reverse`, `--resume-from`, output aggregation, run summaries, pre/post lifecycle scripts, a shell emulator, and per-project `executionEnv`. What it lacks are two well-understood things that every dedicated task runner has, and one of them costs users wall-clock time today with no configuration involved.

**The scheduler is the naive one.** `runRecursive` (`pnpm11/exec/commands/src/runRecursive.ts:109`) is, in outline:

```js
for (const chunk of packageChunks) {
  await Promise.all(chunk.map(runOne))
}
```

Every project in chunk *N* must finish before any project in chunk *N+1* starts. A project whose own dependencies completed in the first second waits for the slowest, least related project in its chunk. On a wide workspace with uneven build times this is the difference between a critical path and a sum of per-chunk maxima, and it is paid by every user of `pnpm -r run` and `pnpm -r exec` right now.

**There is no way to say a task depends on anything but its own name.** `pnpm -r run test` runs `test` in topological project order, but nothing expresses that a project's `test` needs that project's `build` first. The ecosystem's workaround is `pnpm -r run build && pnpm -r run test`, which inserts a workspace-wide barrier that is strictly worse than the chunk barrier it is layered on: no project's tests may start until every project's build has finished.

### What already exists

The graph is not missing. It is computed and then discarded.

- `sequenceGraph` (`pnpm11/workspace/projects-sorter/src/index.ts`) builds a `Map` of project to dependencies and hands it to `graphSequencer` from `@pnpm/deps.graph-sequencer`, which returns `{ chunks, safe }`.
- `sortProjects` and `sortFilteredProjects` return `.chunks` and drop `safe`, so cycle information does not reach the run path at all.
- `sortFilteredProjects` already resolves edges through the *full* workspace graph, so a relationship between two selected projects via an unselected one is honoured. That subtlety is worth preserving verbatim; it is easy to lose in a rewrite.

The change this RFC proposes is mostly *stop flattening*: the scheduler consumes the graph the sequencer already built.

## Detailed Explanation

### Per-task scheduling

A task is a `(project, script)` pair. A task becomes runnable when every task it depends on has completed successfully; runnable tasks are dispatched under the existing `--workspace-concurrency` limit. The chunk loop disappears.

**The scheduler preserves dependency order and nothing else, and that is a behaviour change worth stating plainly.** Under chunking, every task in a chunk finishes before any task in the next one starts, which incidentally orders many tasks that have no dependency between them. Per-task scheduling drops those incidental barriers: if `a:build` finishes early, `b:build` may start while an unrelated `c:build` is still running, which chunking would have prevented. Start order, completion order, and therefore the interleaving of side effects all change for independent tasks.

For most workspaces this is invisible and is the entire win — a workspace with one slow project sees every unrelated project stop waiting for it. It is *not* invisible for tasks that were relying on the accidental barrier: two tasks writing the same file outside their own project, or contending for a fixed port, or mutating a shared directory, were previously separated by chunk boundaries often enough to seem safe. Such tasks were already racing whenever they landed in the same chunk; this makes the race reachable more often rather than introducing it. Declaring an edge between them is the fix, and is now expressible.

What does not change: no task runs that did not run before, no task is skipped that was not skipped before, and no task starts before something it depends on has finished.

`--sequential` continues to mean concurrency 1, and with a real graph it now has an unambiguous meaning it did not quite have before: one task at a time, in a topological order.

### Per-task concurrency

`--workspace-concurrency` is global, so a workspace with one greedy task — an integration suite that starts a database, a bundler that wants every core — can only slow that task down by slowing everything down. A task may therefore declare its own limit:

```yaml
tasks:
  test:
    dependsOn: ['build']
    concurrency: 2
```

The limit is per task *name*, counted across projects, and it is a ceiling rather than a reservation: at most two projects' `test` tasks run at once, within whatever `--workspace-concurrency` allows overall. A positive integer is the only meaningful value.

**The permit is taken before dispatch, not inside the worker.** A task waiting on its group's permit must not occupy a global concurrency slot, or the workspace idles behind its own narrowest task while unrelated work is ready to run. It also means a waiter is, until dispatched, indistinguishable from any other queued task — including when a `--bail` stops the run, where it is simply never dispatched.

### Tasks that never exit

*Not implemented.* A `dev` server, a `--watch` build, a file-syncing daemon: tasks that run until interrupted, which the graph has no way to distinguish from a task that is merely slow.

```yaml
tasks:
  dev:
    persistent: true
```

**A task may not depend on a persistent one, and declaring that is an error.** `dependsOn: ['dev']` asks the scheduler to wait for something that will never finish, and today it does exactly that — the run hangs, with no output explaining why, and the user's first theory is that pnpm deadlocked. Rejecting the configuration turns an unbounded wait into a sentence naming both tasks.

Every comparable tool has this declaration, which is the strongest argument for it: Turborepo's `persistent: true`, moon's `persistent` with a `server` preset inferring it for `dev`, `start`, and `serve`, and Nx's `continuous: true`. The name here is `persistent`, being what two of the three use.

**A persistent task still occupies a concurrency slot, and a run that cannot fit them all is an error rather than a hang.** Ten projects with a `dev` task and `--workspace-concurrency=4` is six servers that never start and four that never finish, which presents as a run that came up short and stayed there. The check is arithmetic — a run whose persistent tasks outnumber its concurrency limit fails before dispatch, naming the limit it would need — and it is Turborepo's answer to the same problem. The alternative, exempting persistent tasks from the limit as moon does by running them last and in parallel, silently overrides a number the user chose; a workspace that wants every server running can say so once.

What this deliberately does *not* do is let a dependent proceed against a running persistent task. Nx's `continuous` does, which is how an `e2e` target waits for a `dev` server to be *ready* rather than *finished* — a genuinely better answer that needs a readiness signal pnpm has no way to observe. If that signal ever exists, this is the section to revisit; erroring is the honest behaviour until then.

### Declaring task dependencies

A `tasks` section in `pnpm-workspace.yaml`, with Turborepo's `^` convention because it is the one the ecosystem already reads:

```yaml
tasks:
  build:
    dependsOn: ['^build']
  test:
    dependsOn: ['build']
  lint: {}
```

`^name` means the named task in each of this project's workspace dependencies. A bare `name` means the named task in this same project.

**The default is today's behaviour.** A task with no entry behaves as `dependsOn: ['^<its own name>']`, which is exactly what the current chunking implies. A workspace that adds no configuration gets the scheduler improvement and nothing else, and no existing command changes meaning.

**A missing script is a satisfied edge that passes through.** If a project has no script by that name, its task is skipped, and dependents resolve the edge through to *that project's* upstream tasks rather than stopping there. A pure pass-through package — one that re-exports its dependencies and builds nothing — must not sever a chain, which it would if a missing script simply terminated the traversal. This rule matters more than it looks: the task cache RFC depends on it to key a task on the upstream tasks that actually produced the bytes it consumes.

`lint: {}` above is a task with an explicitly empty dependency list — it depends on nothing and may start immediately. That is a different statement from omitting the entry, and the difference is deliberate.

**What the graph contains and what the invocation requests are two inputs, not one.** The scheduler is given a project map and, separately, the set of projects whose tasks are actually asked for; everything else in the map participates only by being on the other end of an edge. A caller that narrows only the requested set, which is what an affected-since-a-git-ref selection wants to do, gets edges that resolve identically however the run was narrowed. Keeping the two separable costs nothing here and is load-bearing for the cache RFC, whose task keys are wrong if a `^` edge can be truncated by a selection.

**A `--filter` narrows both, and an edge that leaves the selection is dropped rather than deferred.** Tasks exist only for selected projects, so `dependsOn` never runs anything the filter excluded. Concretely: `a` depends on `b`, `test: dependsOn: ['^test']`, and the invocation is `pnpm --filter a run test`. `b#test` is not created, `a#test`'s edge to it does not exist, and `a#test` becomes runnable immediately. It is not a pass-through — that rule is about a project that was selected and lacks the *script* — and it is not a deferred prerequisite. The same graph is what the scheduler runs, what `--dry-run --json` prints, and what `--resume-from` slices, so none of the three can disagree about it.

This is today's behaviour preserved rather than a new rule: `pnpm --filter a run test` has never built `b`, and a user who wants the dependencies wants `--filter a...` and can say so. It is worth stating explicitly because the alternative reading — that a declared edge conscripts an unselected project — is the one a reader arrives with from Turborepo, where filtering does pull dependencies in.

### `pnpm -r exec` takes the scheduler, not the declarations

`exec` runs a command rather than a named script, so there is nothing for a `tasks` entry to be keyed by: a declaration names a script, and a command typed at the prompt has no name in the workspace file. Its graph is one task per selected project over the project dependency edges. It gains the per-task scheduling and loses the chunk barrier along with `run`; `dependsOn` does not reach it.

This is a definition rather than a gap to be closed later. `exec` could only join a declared graph by first acquiring a name, and minting one — a reserved `exec` entry, or a flag naming some other task's declarations to borrow — would give a workspace two ways to spell its task relationships, which then have to be kept from disagreeing. If naming a command is ever worth doing, the coherent shape is a workspace entry that *is* a task rather than a field bolted onto this one, and it deserves its own argument.

### Cycles

Package-level cycles are already possible and `graphSequencer` already reports them through `safe`, which the run path currently ignores. Task-level `dependsOn` adds a second way to build one — `a:build → b:build → a:test → a:build` — without any package cycle existing.

A cycle in the task graph is an error naming the participating tasks, not a warning: `ERR_PNPM_TASK_CYCLE`, spelling the cycle out as `a#build → b#build → a#build`. The current silent behaviour, where a cyclic graph is sequenced into some order and runs, is worse than it appears: it produces a run that succeeds or fails depending on which arbitrary order the sequencer chose. Since this RFC has to detect cycles anyway to schedule, reporting package cycles at the run layer comes along for free and should be taken.

**`ignoreWorkspaceCycles` remains the escape hatch, and gains no new spelling.** A workspace that has already declared its cycles deliberate keeps that declaration here: the error is downgraded to a warning, the cycle members lose their mutual edges, and they run in the sequencer's deterministic order rather than an arbitrary one. Introducing a second, task-specific setting for the same statement would leave a workspace with two places to say it and one place to forget.

**A cycle that exists today becomes an error without a deprecation window.** This does turn a silently-succeeding run into a failing one for any workspace that has a cycle and has not noticed, which normally argues for a window. It does not here, for two reasons that are specific rather than general. The escape hatch already exists and predates this RFC — `ignoreWorkspaceCycles` is a setting such a workspace may already have, and the error's own hint names it — so the remedy is one line in a file the user already has, discovered from the message rather than from release notes. And the run being "broken" is the one that was already broken: a cyclic graph sequenced into an arbitrary order succeeds or fails by which order the sequencer happened to choose, so what a window would preserve is the ability to keep getting an answer that means nothing.

**Detection is scoped to the invocation's own graph, not to the workspace.** It runs after project selection and after `^` expansion and missing-script pass-through, immediately before scheduling — not when `pnpm-workspace.yaml` is parsed. A cycle among tasks a `--filter` did not select cannot fail the run, because those tasks are not in the graph being scheduled and nothing about the run depends on them being acyclic. Validating the whole workspace at config load would make an unrelated corner of a large monorepo everyone's problem, which is a good way to have the feature disabled.

### Flags defined in terms of chunks

Three things are currently specified against the chunk list and need redefinition. All three are behaviour changes and belong in the RFC rather than in the implementation's judgement:

- **`--resume-from`** slices the chunk array (`getResumedPackageChunks`, `pnpm11/exec/commands/src/exec.ts:104`), meaning "from that project's chunk onward". Three things need saying without chunks: which task the flag names, what the resume set is, and what `exec` does.

  **It still names a project, and resolves to that project's task for this invocation** — `(project, <script being run>)` for `pnpm -r run`, and the project's single command task for `pnpm -r exec`, which has one task per project and therefore no ambiguity at all. A project that is not selected is `RESUME_FROM_NOT_FOUND`, as today. A project that *is* selected but has no such script resolves to its pass-through node: today a scriptless project is a valid resume anchor because the flag is positional, and there is no reason for a graph to be pickier than the chunk list was.

  **The resume set is every selected task except the anchor's transitive dependencies.** This is the graph-native reading of "from that chunk onward": what a chunk slice approximates positionally is "everything not ordered strictly before the anchor", and under a graph "before" means exactly "is a transitive dependency of". It is *more* precise than the chunk slice, which also re-runs everything sharing the anchor's chunk.

  An earlier draft of this RFC proposed "the anchor and its transitive dependents", which is wrong. After a `--bail` run stops, work unrelated to the failure has not run either, and a dependents-only closure would silently skip it — leaving the user to discover later that resuming did not resume everything. Excluding only the anchor's dependencies skips exactly what is known to have finished and nothing else.

  **Completion is recorded, not only inferred.** The rule above *infers* that the anchor's dependencies finished. That holds when resuming after a failure and does not hold when resuming a run that was interrupted, or a deliberately partial one, where a dependency may never have started. pnpm therefore keeps an append-only journal of the tasks that passed, written as they pass, under `node_modules/.pnpm-task-run-state-v1/`, and when a compatible journal exists it *replaces* the graph rule rather than refining it: the tasks skipped are exactly those the journal records as passed, wherever they sit in the graph. Replacement rather than intersection, and deliberately — a journal frequently skips *fewer* tasks than graph position would, and that case is the interrupted run whose dependencies never finished, which is the one the journal exists for. It is evidence; the graph rule is an assumption, and an assumption does not get to narrow evidence.

  Compatibility carries the whole safety argument, so a journal is keyed by an invocation identity covering the selected task graph — each task's own scripts and edges included — the command and its arguments, and the execution settings that change what a script does. An edited script, a different filter, a changed shell or pre/post setting therefore yields a different identity and no reuse at all. That is the safe direction to fail in: the cost of a mismatch is re-running work that had already passed, never skipping work that never ran.

  Each invocation owns its own journal behind an atomically published latest-run pointer, so overlapping runs cannot consume or delete one another's state; a state directory that cannot be written disables persistence rather than failing the run.

  **With no compatible journal, `--resume-from` is back to inferring, and that is a known limitation rather than an oversight.** A run interrupted before anything was persisted, or one whose state directory was unwritable, leaves the flag with nothing but graph position to go on — the very inference the journal exists to replace — so it can exclude a dependency that never actually passed. Failing the resume instead would be the conservative answer and is worse in practice: it turns the flag off precisely for the interrupted runs people reach for it after, having already lost the work. The honest resolution is to make the absence visible, so a resume that could not consult a journal says so rather than silently returning to the weaker rule.
- **`--reverse`** reverses the chunk array. Against a graph it means running the reverse graph — dependents before dependencies.
- **`--no-sort`** (and `--parallel`, which implies it) means disregarding ordering entirely, and that has to extend to the declarations: every task becomes independent, `dependsOn` is not applied, and no task is pulled in by an edge. A workspace that has declared `tasks` and then asks for no ordering is given a warning rather than a silent reinterpretation, because the two requests genuinely conflict. `--reverse` and `--resume-from` have nothing left to order and become no-ops.
- **Output mode.** `stdio` is `'inherit'` when concurrency is 1 or the chunk shape guarantees a single script, and `'pipe'` otherwise. The equivalent condition on a graph is that at most one task can ever be in flight; where that does not hold, output is piped and prefixed as it is today.

**Failure handling** is not chunk-defined but needs a stated meaning, because "runnable when every dependency succeeded" leaves a dependent of a failed task with nowhere to go. Four states, and they are the same states in both modes: `passed`, `failed`, `skipped` (no such script, or a dependency did not pass), and `not run` (never dispatched because the run stopped). A fifth exists only under `--bail`: a task interrupted mid-flight, which is reported as still `running`, because that is what it was when the run ended.

- **`--bail`** (the default): on the first failure, no further tasks are dispatched **and tasks already running are cancelled**, their process trees along with them. Everything not dispatched is `not run`. Letting in-flight work finish is the gentler rule and it does not survive contact with real scripts: a watch-style task never exits, so a bailing run would hang indefinitely instead of reporting the failure the user is waiting to see. The failure that triggered the bail remains the reported one — an interrupted task is not a second failure, and does not add to the exit code's count.
- **`--no-bail`**: the run continues. Every task whose dependencies all passed still runs, so an unrelated subtree completes normally. Tasks that transitively depended on a failed task are `skipped`, never `failed` — reporting them as failures, which a naive implementation does, turns a one-line error in a leaf package into what reads as a workspace-wide outage.

**The exit code is non-zero if and only if at least one task is `failed`.** A `skipped` dependent does not add to the count and does not need to: the failure that blocked it is already counted, and counting both would say two things went wrong when one did.

### Dry runs

`pnpm -r run --dry-run <script>` prints the task graph that would execute, without running anything. `--dry-run` does not exist on `run` today in either form; this is a new recursive-run feature, and it is recursive-only because root-level tasks are out of scope.

**What it prints is a partial order, not an order.** Independent tasks have no required sequence, and output that implies one would be read as a promise the scheduler does not make. The `--json` form therefore emits *nodes and edges*: each task as a stable identifier — the workspace-relative project directory and the script name, the same pair the scheduler and the cache key use — with its resolved dependency edges after `^` expansion and missing-script pass-through, and a flag for tasks skipped because the script is absent. Consumers reconstruct any ordering they need from the edges.

The human-readable form does print a sequence, because a list is what a person reads, and it is *one* valid linearization rather than the one the scheduler will follow. Ties among simultaneously runnable tasks are broken by project directory, so repeated runs print the same thing and a diff of two dry runs is meaningful.

This is not a nicety. The first question anyone asks of a task graph is "why did that run" or "why didn't it", and the second, once the cache RFC lands, is "why was that a miss". Without a way to see the resolved graph, both are answered by guesswork. Turborepo's `--dry=json` is load-bearing in practice for exactly this reason, and adding it after the fact means every early adopter debugs blind.

### What this does not add

Watch mode, a task-graph visualizer, a new terminal UI, tasks that are not npm scripts, and root-level tasks are all out of scope. So is caching, which is a separate RFC that consumes this one.

The exclusions are not permanent judgements — watch mode in particular is a reasonable thing to want — but each is a self-contained feature that does not change the scheduler or the graph, and bundling them would make this RFC a referendum on how much of a task runner pnpm should be rather than on whether the scheduler should stop discarding the graph.

## Rationale and Alternatives

**Fix the scheduler, skip `dependsOn`.** The scheduler change is a pure win with no configuration surface, and it is tempting to ship it alone. The reason to specify both together is that the scheduler's value is capped without the declaration: the largest barrier in a typical workspace is not the chunk boundary but the `&&` between `pnpm -r run build` and `pnpm -r run test`, which only `dependsOn` removes. They are separable in implementation and should ship in that order; they are not separable in argument.

**Adopt Turborepo's configuration wholesale, including its file.** Reading `turbo.json` would give instant compatibility for workspaces already using it. Declined: it would make pnpm's behaviour depend on another tool's schema evolution, and the subset pnpm implements would silently differ from the file's meaning under Turborepo. Using the same `^` convention inside `pnpm-workspace.yaml` gets the familiarity without the coupling.

**Infer task dependencies from what tasks read and write.** Rather than declaring `test: ['build']`, notice that `test` reads `dist/` and `build` writes it. Attractive, and wrong in the same way that inferring build inputs is wrong — it requires observing the task, which means running it, which is what the dependency was supposed to order. Worth revisiting only as a *check* on declared edges, not as a substitute.

**Do nothing; workspaces that need this run Turborepo or Nx.** The honest baseline. It is a weaker argument here than for caching, because the chunk barrier is a cost pnpm imposes on its own users today whether or not they want a task graph, and because the graph pnpm would need is one it already computes.

## Implementation

**The scheduler.** Replace the chunk loop in `runRecursive` and the equivalent in `pnpm -r exec` with a runnable-set scheduler over the graph `sequenceGraph` already builds, preserving `sortFilteredProjects`'s per-project graph selection so prod-only selected projects keep tunnelling through the prod-pruned graph. The recursive summary is currently constructed from the chunk list (`createEmptyRecursiveSummary(chunks)`) and needs a task-keyed equivalent.

**Configuration.** The `tasks` section in `pnpm-workspace.yaml` — an unused key before this RFC — with `dependsOn` and `concurrency` parsing and validation at load, and `^` resolution, missing-script pass-through, and cycle detection per invocation against the selected graph. Task fields are validated against a known-key set in both stacks, so an unrecognised one is an error rather than a silent no-op; the cache RFC adds more fields to the same section and depends on that gate existing.

**Flags.** `--resume-from` with its run-state journal, `--reverse`, `--no-sort`, the output-mode condition, and the task states with their `--bail`/`--no-bail` behaviour and exit code, per the section above. Cancelling in-flight work on a bail is the part with the most ways to go wrong: a cancelled task must still settle, or the scheduler waits forever for a task nobody is going to report.

**`--dry-run`,** with `--json`.

**Both stacks, per the repository's parity rule.** The Rust side has the executor and the workspace graph crates; the scheduler is the shared piece and is where a behavioural divergence between the two CLIs would be least visible to whoever introduced it.

Ordering: scheduler first, since it is independently valuable, changes no configuration, and lets the graph-consuming code exist before anything depends on it. Then `dependsOn`, then the flag redefinitions, then `--dry-run` — though `--dry-run` is the thing that makes `dependsOn` reviewable in practice, and pulling it earlier is defensible.

## Prior Art

**Turborepo** supplies `dependsOn` and the `^` convention, adopted here. Its `--dry=json` is the model for the dry-run output. Its scheduler is per-task, which is what makes the comparison against pnpm's chunking unflattering.

**Nx** demonstrates the same scheduling model with a much larger surface around it — generators, executors, plugins — which is the outcome this RFC's exclusions exist to avoid. Nx also shows that once a task graph exists, users immediately want to see it, which is the argument for `--dry-run` shipping close behind.

**Rush** phases are a more expressive version of the same idea, letting a project's build be split into ordered phases with their own dependencies. That is where `dependsOn` leads if it succeeds, and the syntax here does not foreclose it.

**pnpm's own filter language** is the reason this is a smaller change than it looks. Selection — which projects, and their dependencies or dependents, and what changed since a git ref — is the part most tools get wrong or omit, and it already exists and already resolves through the full workspace graph.

## Unresolved Questions and Bikeshedding

- **Whether per-project overrides are needed at all.** They are not implemented: `tasks` is one workspace-wide map, and a project that needs different `dependsOn` from its siblings has no way to say so. Whether real workspaces need that is the open half.

  Where such overrides would live is not open. They belong in `pnpm-workspace.yaml`, keyed by project glob, and never in `package.json` — the same argument that keeps `exec` out of the declarations applies here, that one workspace should not be able to spell its task relationships in two files that then have to be kept from disagreeing. The workspace task cache RFC already assumes this shape for its own `tasks` fields.

  The `^` sigil was settled from the start, and deliberately: it is what Turborepo, Nx, Lerna (via Nx), and Lage all use, which is the overwhelming majority of the installed base and what anyone migrating will paste. Rush spells the same distinction as explicit `upstream`/`self` keys and moon uses `^:`/`~:`; both are more readable on first encounter and neither is a convention there is any compatibility argument for adopting.

- **A `--continue` flag**, whose meaning is settled and whose implementation is not. It runs every task the most recent compatible run-state journal does not record as passed, which is `--resume-from` with the anchor removed — and the anchor is exactly what the journal made unnecessary, since it exists to say what finished rather than to have it inferred from a project name. A task that completed because it had no script to run is recorded like any other completed task, so a pass-through neither comes back as work nor severs a chain on the second attempt. With no compatible journal it is an error rather than a silent full run: a user asking to continue is asserting there is something to continue from, and quietly running everything is the answer least likely to be what they meant. What is open is whether it earns its place next to `--resume-from`, which is a question about the CLI's surface rather than about behaviour.
