# Workspace task orchestration

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

### Cycles

Package-level cycles are already possible and `graphSequencer` already reports them through `safe`, which the run path currently ignores. Task-level `dependsOn` adds a second way to build one — `a:build → b:build → a:test → a:build` — without any package cycle existing.

A cycle in the task graph is an error naming the participating tasks, not a warning. The current silent behaviour, where a cyclic graph is sequenced into some order and runs, is worse than it appears: it produces a run that succeeds or fails depending on which arbitrary order the sequencer chose. Since this RFC has to detect cycles anyway to schedule, reporting package cycles at the run layer comes along for free and should be taken.

**Detection is scoped to the invocation's own graph, not to the workspace.** It runs after project selection and after `^` expansion and missing-script pass-through, immediately before scheduling — not when `pnpm-workspace.yaml` is parsed. A cycle among tasks a `--filter` did not select cannot fail the run, because those tasks are not in the graph being scheduled and nothing about the run depends on them being acyclic. Validating the whole workspace at config load would make an unrelated corner of a large monorepo everyone's problem, which is a good way to have the feature disabled.

### Flags defined in terms of chunks

Three things are currently specified against the chunk list and need redefinition. All three are behaviour changes and belong in the RFC rather than in the implementation's judgement:

- **`--resume-from`** slices the chunk array (`getResumedPackageChunks`, `pnpm11/exec/commands/src/exec.ts:104`), meaning "from that project's chunk onward". Without chunks it should mean "that task and everything that transitively depends on it", which is closer to what users want and is what the name suggests.
- **`--reverse`** reverses the chunk array. Against a graph it means running the reverse graph — dependents before dependencies.
- **Output mode.** `stdio` is `'inherit'` when concurrency is 1 or the chunk shape guarantees a single script, and `'pipe'` otherwise. The equivalent condition on a graph is that at most one task can ever be in flight; where that does not hold, output is piped and prefixed as it is today.

**Failure handling** is not chunk-defined but needs a stated meaning, because "runnable when every dependency succeeded" leaves a dependent of a failed task with nowhere to go. Four states, and they are the same states in both modes: `passed`, `failed`, `skipped` (no such script, or a dependency did not pass), and `not run` (never dispatched because the run stopped).

- **`--bail`** (the default): on the first failure, no further tasks are dispatched. Tasks already running are allowed to finish and are reported with whatever they returned. Everything not dispatched is `not run`.
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

**Configuration.** The `tasks` section in `pnpm-workspace.yaml` — the key is currently unused — with `dependsOn` parsing and validation at load, and `^` resolution, missing-script pass-through, and cycle detection per invocation against the selected graph.

**Flags.** `--resume-from`, `--reverse`, the output-mode condition, and the four task states with their `--bail`/`--no-bail` behaviour and exit code, per the section above.

**`--dry-run`,** with `--json`.

**Both stacks, per the repository's parity rule.** The Rust side has the executor and the workspace graph crates; the scheduler is the shared piece and is where a behavioural divergence between the two CLIs would be least visible to whoever introduced it.

Ordering: scheduler first, since it is independently valuable, changes no configuration, and lets the graph-consuming code exist before anything depends on it. Then `dependsOn`, then the flag redefinitions, then `--dry-run` — though `--dry-run` is the thing that makes `dependsOn` reviewable in practice, and pulling it earlier is defensible.

## Prior Art

**Turborepo** supplies `dependsOn` and the `^` convention, adopted here. Its `--dry=json` is the model for the dry-run output. Its scheduler is per-task, which is what makes the comparison against pnpm's chunking unflattering.

**Nx** demonstrates the same scheduling model with a much larger surface around it — generators, executors, plugins — which is the outcome this RFC's exclusions exist to avoid. Nx also shows that once a task graph exists, users immediately want to see it, which is the argument for `--dry-run` shipping close behind.

**Rush** phases are a more expressive version of the same idea, letting a project's build be split into ordered phases with their own dependencies. That is where `dependsOn` leads if it succeeds, and the syntax here does not foreclose it.

**pnpm's own filter language** is the reason this is a smaller change than it looks. Selection — which projects, and their dependencies or dependents, and what changed since a git ref — is the part most tools get wrong or omit, and it already exists and already resolves through the full workspace graph.

## Unresolved Questions and Bikeshedding

- **The `tasks` key name.** `tasks` is unused in `pnpm-workspace.yaml` today and reads well, but `scripts` is what package.json calls these and `pipeline` is what Turborepo called it before renaming to `tasks`. Also unresolved: whether per-project overrides live in `package.json` or as globs in the workspace file.

  The `^` sigil itself is settled rather than open, and deliberately: it is what Turborepo, Nx, Lerna (via Nx), and Lage all use, which is the overwhelming majority of the installed base and what anyone migrating will paste. Rush spells the same distinction as explicit `upstream`/`self` keys and moon uses `^:`/`~:`; both are more readable on first encounter and neither is a convention there is any compatibility argument for adopting.
- **`--resume-from` semantics, and what it names.** "That task and its transitive dependents" is proposed above, but the current meaning is "that chunk onward", and some users may be relying on the accidental inclusion of unrelated projects that happened to share a chunk. Sharper still: the flag takes a *project* name today, while a task is a `(project, script)` pair, so `pnpm -r run test --resume-from foo` has to resolve to a task — and it is not obvious what it should resolve to when `foo` has no `test` script and pass-through applies. `pnpm -r exec` has no script name at all, so it may need a different rule or none.
- **Per-task concurrency limits.** `--workspace-concurrency` is global. A workspace with a memory-hungry task may want to limit that task specifically, which is a natural extension and an easy one to over-design.
- **Interaction with `pnpm -r exec`.** `exec` runs a command rather than a named script, so it has no name to hang `dependsOn` on. It should get the scheduler; whether it can participate in a declared graph at all is open.
- **Cycles that exist today.** Making a currently silent cycle an error is a breaking change for any workspace that has one and has not noticed. Whether that needs a deprecation window or is simply a bug fix wants a survey of how common they are.
