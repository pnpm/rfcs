# The pnpm CI server

## Summary

The [`pnpm ci` RFC](./0000-pnpm-ci.md) leaves four jobs with the CI host: reacting to events, holding secrets, drawing the UI, and coordinating machines. This RFC moves all four onto pnpr, in the shape Buildkite proved: a control plane on a server the organization operates, executing on agents the organization runs on its own machines. pnpr gains a run-results store and web UI, a task-leasing coordinator serving a `pnpm ci agent` fleet, forge and registry-native triggers, and sealed secrets delivery. The forge keeps git hosting, code review, and the merge queue; the operator keeps the hardware; pnpm never provisions compute.

One invariant, recorded in the `pnpm ci` RFC's forward-compatibility section, governs every feature here: **the server never becomes the root of trust.** pnpr's compromise today is bounded by verification clients perform — it cannot substitute lockfile-pinned bytes, cannot sign artifacts, cannot unmask denied packages — and nothing in this RFC is allowed to break that bound. The sections below are largely the working-out of that sentence.

## Motivation

After `pnpm ci` lands, look at what a workflow file still contains: a trigger stanza, a runner label, a block of secret plumbing, and an implicit subscription to the host's UI. The pipeline's *meaning* moved into the repository; what remains with the host is event delivery, machine brokerage, and presentation — and the host is priced as if it still owned the pipeline. Meanwhile the organization running pnpr already operates a server that holds the run's logs (in the artifact store, per task, content-addressed), the cache the run hits, the auth and policy layer, and per-organization storage with quotas. The CI vendor's product pillars — log storage, cache, build history — are already on the org's own server. What is missing is coordination and presentation, which are the cheap layers, sitting on storage and trust, which are the expensive ones and are built.

Three capabilities become available to a registry-resident control plane that no external CI can match:

- **Registry-native triggers.** pnpr sees every publish, and pnpm knows every workspace's dependency graph. "When `@ourco/foo` publishes, run `check` in every workspace that depends on it" becomes a first-class subscription rather than a Renovate polling loop — the trigger a forge structurally cannot offer, because the forge cannot see the graph.
- **Self-provisioning agents.** A CI agent normally needs a maintained image: toolchains baked in, drifting from what repositories actually pin. A pnpm agent needs git and pnpm, full stop — the lockfile pins the runtime and package managers, the store materializes them, and a warm agent's checkout-and-install is mostly reflinks. The `pnpm ci` RFC's toolchain enforcement is what makes agent images trivial; this RFC is where that pays off.
- **The cache is the data plane.** Distributed execution is economical only because a completed task is an artifact any other agent can restore, and the artifact store, its signing, and its quotas shipped with RFC 0007. A coordinator adds bookkeeping, not a transfer protocol.

### What already exists

- **The stored record.** The `pnpm ci` RFC's run summary and event stream, with stable task identities, cache dispositions, and the resolved base — designed there to assume nothing about a single executor, for this RFC's benefit.
- **Logs.** Already durable, per task, in the owner-scoped artifact store, as a byproduct of the task cache RFC.
- **Auth, policy, quotas, masking.** pnpr's token layer, per-package policy, per-owner storage accounting, and denied-as-not-found masking all carry over to run records and leases.
- **The trust pattern.** RFC 0007's constraint that signing material lives on machines and never in committed files, and that clients verify against keys configured out of band. Agent identity and secrets sealing below are that pattern, reused.
- **The hardened write path.** RFC 0007's v0 notes record what accepting untrusted uploads into pnpr cost — locking, accounting, size and concurrency bounds, crash residue. Run-record submission and lease endpoints inherit that machinery and that caution rather than discovering the cost again.
- **Precedent for pnpr-side compute.** [RFC 9](https://github.com/pnpm/rfcs/pull/9)'s pnpm agent already proposes pnpr doing per-organization work adjacent to the store.

## Detailed Explanation

The four tiers below are severable and ordered by risk. Each is independently valuable, each is a valid stopping point, and later tiers consume earlier ones without amending them.

### Tier 1: run records and the UI

`pnpm ci` gains the ability to submit its summary and event stream to pnpr at the end of a run, into an organization-scoped, append-only `runs` namespace keyed by workspace identity, commit, and pipeline. Two properties are structural rather than optional:

- **Records are signed by the submitting identity** — the same signing material that authorizes cache writes. Trusted runs produce records; an untrusted run (a fork PR holding no key) can execute against the cache read-only exactly as before, and simply has no standing to assert history. This is not a limitation to work around; a results store whose writers are unauthenticated is a false-green generator with an API.
- **Records live in pnpr's authoritative `storage` root, not the disposable `cache` root.** An artifact is regenerable from a rebuild; a run record is testimony about something that happened once. Wiping the cache must not erase history, so run records are the first CI data with the durability contract of packages rather than of artifacts.

The UI is a reader over this store and the artifact store: runs by branch and commit, the task graph with the four states, cache dispositions with the key components that differed on a miss, and per-task logs streamed out of the artifacts that already hold them. This is the BuildBuddy / Gradle-Enterprise shape — the result store and UI are worth deploying while execution still happens on the forge's runners, which is exactly how this tier should be adopted first.

One trust subtlety must be stated because it is the single place server data influences what runs. The `pnpm ci` RFC anticipated "affected since last green" — selecting against the last commit whose pipeline passed, which requires this store. A server that lies about the last green commit causes over-pruning, and over-pruning is a false green: the selection pre-pass's safety argument ("a mistake costs key hashing, not correctness") holds for a *wrong base*, not a *withheld task*. The bound is auditability plus consent: a run selecting against stored history records, in its own summary, the run it trusted — id, commit, and record digest — so a fabricated green is visible in the evidence chain of every run that relied on it; and last-green selection is opt-in per pipeline, with branch-tip base remaining the default. Organizations that do not accept server-influenced selection simply never enable it and lose nothing else in this RFC.

### Tier 2: agents and leasing

`pnpm ci agent` is a long-lived daemon on the organization's machines. It holds an identity keypair, is enrolled by an operator against the org's pnpr, advertises its platform by the same canonical tag vocabulary and fingerprint RFC 0007 defined for artifacts — which is what routes a matrix leg to a matching machine for free — and long-polls for work. The machines are the operator's: bare metal, VMs, containers, spot instances, pnpr does not know or care. pnpm's ambition stops at the agent binary; provisioning compute is the host-industry's business and stays there.

A distributed run works as follows. A run is created (by a trigger, tier 3, or by anything that can call the endpoint — a thin forge workflow being the expected on-ramp). **Planning is itself a leased task**: the first agent to take the run checks out the commit, resolves the pipeline exactly as single-machine `pnpm ci` does — base, selection, graph, and task keys, which the `pnpm ci` RFC required to be computable without executing — and posts the plan. The coordinator then leases tasks whose dependencies have completed, in the orchestration RFC's semantics lifted server-side: same states, same `--no-bail` accounting, same exit contract assembled into the same summary document, now with more than one executor named in it. A completed task's outputs are published as cache artifacts before the lease is marked done, so any downstream agent restores rather than rebuilds; the store is the only transfer channel, and no new one is added. Leases expire and re-lease on agent death; a task is idempotent by its key, so a re-run of a completed lease is a cache hit and double execution is wasteful rather than wrong.

Because each task runs on an agent that materialized only its declared world, this tier is also where hermeticity — the task cache's weakest assumption — gets its upgrade path: an agent can execute tasks in a rootless container with the store mounted read-only and the checkout confined to declared inputs, turning "undeclared input" from silent key corruption into a visible failure. That mode is optional and its details are the task cache RFC's concern, but the agent is the natural place it lands.

Single-machine `pnpm ci` is unchanged, and remains the degenerate case: a run whose planner and only executor are the same process.

### Tier 3: triggers

With run records and agents in place, starting runs server-side is a small step, taken three ways:

- **Forge events.** pnpr accepts webhooks (push, pull request, tag) from GitHub, GitLab, and Gitea, mapped to pipelines by a `ci.triggers` section next to `ci.pipelines`. For deployments that cannot accept inbound internet traffic — the registry's natural habitat is behind the perimeter — the same mapping drives a polling mode against the forge API instead; webhooks are an optimization, not the mechanism.
- **Schedules.** Cron pipelines are server-side state, trivially.
- **Registry events.** The flagship: a subscription mapping publishes of matching packages to pipeline runs of dependent workspaces. What the pipeline does with the event — update the dependency, test, open a PR — is the workspace's business, expressed as ordinary tasks.

The trust split follows the pattern established twice already. *What* runs — the trigger mapping, the pipelines — is repository policy and lives in the committed workspace file. *How much a ref is trusted* — which refs' runs may hold signing material, write records, receive secrets — is never committed, because on a pull request the workspace file is attacker-controlled input; ref-trust classification is server-side organization configuration, the exact reasoning by which RFC 0007 refuses signing configuration in `pnpm-workspace.yaml`. An untrusted ref's run executes that ref's own pipeline definition safely because the lease it runs under carries nothing: no signing key, no record-write standing, no secrets.

### Tier 4: secrets

The tier that historically turns CI servers into breach headlines, and the one the invariant was recorded for. The design constraint, stated as the feature: **pnpr relays secrets it cannot read.** Two mechanisms satisfy it, and both should exist:

- **Sealed secrets.** A secret is encrypted (HPKE-style) to the identity keys of the agents authorized to use it, at the time an operator stores it. pnpr holds ciphertext, admission policy decides which leases carry which ciphertexts, and decryption happens only on the agent, inside the lease. A compromised pnpr yields ciphertext and metadata, not credentials.
- **References.** A secret entry may instead name an external store — Vault, a cloud KMS — which the agent resolves with its own identity. pnpr stores a pointer and never touches the material at all.

Delivery policy is a function of (pipeline, ref trust): untrusted refs receive no secrets *by construction*, because the lease never includes them — the same possession-shaped rule as cache writes, not a flag an option can override. Access is logged append-only next to the run records it belongs to. Enrollment of a new agent re-seals the secrets its roles grant it, which is an operator action, deliberately — a server that could re-seal on its own initiative could seal to a key it invented, which is exactly the capability the invariant denies it.

The honest residue, so the RFC does not overclaim: a compromised *server* is bounded, but CI's job is to hand credentials to machines that execute code, so a compromised *agent*, or a malicious task on a trusted ref, still reaches whatever secrets its leases carry. That is every CI system's floor. What the invariant buys is that the control plane — the internet-adjacent, everyone-connects-to-it component — is not the jackpot, and that the blast radius of any compromise is a policy-enumerable set of leases rather than the organization's vault.

### What this does not add

Git hosting, code review, and the merge queue stay with the forge permanently; the boundary this RFC moves is from "everything after the YAML" to "everything after the push". Machine provisioning stays with the operator permanently, per the companion RFC's decomposition. No hosted multi-tenant service is proposed: pnpr remains software an organization runs, and whether anyone operates it commercially is not an RFC question. And the companion RFC's exclusion of arbitrary non-task steps is inherited unchanged — the server schedules workspace tasks, not a job DSL.

## Rationale and Alternatives

**Stop at `pnpm ci` on existing hosts.** The honest baseline, again, and for many organizations the permanent answer: the forge's bundled CI is adequate machine brokerage, and everything valuable about the engine already works there. The counter is the tiering itself — tier 1 is worth deploying *on top of* forge CI (the result store and miss-explanation UI have no host equivalent), tier 2 is wanted anyway for fan-out, and an organization can stop after any tier. The RFC does not require believing in tier 4 to ratify tier 1.

**Distribution without hosting.** Ship the coordinator and agents, let the forge keep triggering — the run's first lease created by a workflow job. This is not an alternative but a stopping point: it is exactly tiers 1–2, and the phasing makes it first-class. Recording it here is the answer to "why is this one RFC and not two": because the trigger and secret tiers are increments on the coordinator, not siblings of it, and splitting them would re-derive the same agent and record machinery twice.

**Implement an existing runner protocol** — register pnpr-side agents as GitHub self-hosted runners or GitLab runners. Cheap compatibility, and it surrenders the entire premise: those protocols lease *jobs* (opaque shell scripts on one machine), not tasks, so the graph, the cache-as-data-plane, and per-task placement by platform fingerprint are all invisible to them. The forge-runner protocols are what the workflow-file skeleton already uses to reach `pnpm ci`; there is nothing further there.

**Adopt Buildkite itself as the control plane.** The closest existing shape, and the strongest external product in it. Declined for the same reason as every "sit on an engine" alternative in this sequence: the value argued here is task-level integration with pnpm's own keys, store, and registry, and a control plane that cannot see them reduces pnpm to a build step again — at which point the organization should just use Buildkite and stop at the companion RFC, which works there too.

**Store secrets server-side in plaintext-at-rest-encrypted form, like the incumbents.** Every mainstream CI does it, it is operationally simpler, and it is the design this RFC exists to refuse: it makes the control plane the root of trust, converts every pnpr CVE into a potential credential breach, and breaks the one property that makes a package-manager project shipping a CI server defensible. If the sealed design proves too rigid in practice, the fallback is references-only, never server-readable storage.

## Implementation

pnpr side (Rust, in-repo with the existing crates): a `runs` store in the authoritative storage root with signed-append semantics; lease coordination endpoints with the write-path hardening budget RFC 0007's v0 already paid; webhook/polling trigger adapters per forge; the scheduler lifted from the orchestration RFC's semantics; the UI as static assets served by pnpr; sealed-secret storage and lease assembly. Registry-event triggers hook the existing publish path.

Client side, in both stacks per the repository's parity rule: run-record submission from `pnpm ci`, and the opt-in last-green base resolution with its recorded-evidence rule. **The agent is proposed as pnpr-shipped tooling rather than a CLI subcommand** — it is server infrastructure in the way pnpr itself is, pairing with one coordinator protocol version, and holding it to CLI parity would either freeze it to the slower stack or double the cost of its fastest-moving component. It reuses the Rust stack's executor, store, and toolchain crates, which is most of its body. This is a deliberate exception to the parity rule and is called out for review rather than assumed.

Ordering is the tiering: records and UI; then agents and leasing; then triggers; then secrets. Tier 1 has no new trust surface beyond signed appends and should ship first both because it is cheap and because its store is what every later tier coordinates through.

## Prior Art

**Buildkite** is the architecture: control plane and UI centrally, agents and secrets on the customer's machines. This RFC moves even the control plane onto the customer's own server, and adopts Buildkite's core insight unchanged — the vendor who refuses to hold your compute is also refusing your largest risks.

**GitHub self-hosted runners and GitLab Runner** are the agent half without the graph: job-granular, image-provisioned, and blind to what the job does. The contrast — task-granular leases, lockfile-provisioned toolchains, cache-as-transfer — is the substance of tier 2.

**Drone and Woodpecker** show a self-hosted forge-adjacent CI is a sustainable small-surface project, and their YAML-jobs model shows the surface this RFC's task-only rule avoids.

**BuildBuddy and Buildbarn** are tier 1's model: a result store and UI over a remote-execution cache, valuable before and without owning execution. The Build Event Protocol's lesson — the stream is the product surface, every UI is a consumer — shaped the companion RFC's event stream and is consumed here.

**Nx Cloud's distributed task execution** is the direct model for tier 2's leasing, including planning-as-a-task.

**Jenkins** is the cautionary tale this RFC is designed against: an internet-adjacent coordinator with an unbounded plugin surface and server-held credentials, and the exploitation history that follows. The TeamCity and CircleCI incidents are the same lesson priced in production: the control plane gets attacked, so the control plane must not be the jackpot. The invariant is that lesson, stated as a design rule.

**Concourse** pioneered "the pipeline is code in the repo, the server is dumb", which is the division of meaning this sequence follows; its resource model is also a warning about how quickly a minimal server grows a DSL.

## Unresolved Questions and Bikeshedding

- **Workspace identity.** Run records key on "workspace identity" — repository URL, a declared name, or both. URLs are ambiguous across mirrors and renames; declared names collide and need claiming under org auth. Needs one answer before tier 1's schema.
- **Source delivery to agents.** Agents fetch the commit from the forge, and for a private repository that read credential is itself a secret — one that untrusted PR runs legitimately need, which makes it the sole exception to "untrusted leases carry nothing". Options: a read-only credential class sealed like any secret but granted to untrusted leases; forge-minted ephemeral tokens via the trigger integration; or operator-managed mirrors. The first is simplest and its blast radius (read access attackers already have via the PR) may be acceptable; not settled.
- **Lease granularity and checkout amortization.** A lease per task is clean and makes every agent pay a checkout; leasing subgraphs, or letting agents keep persistent per-repo workspaces keyed by remote, trades scheduler freedom for wall-clock. Probably measurement-driven, like the input-hashing question in the task cache RFC.
- **Agent enrollment.** Operator-approved key registration is the floor; whether to support attestation (workload identity on cloud agents) so enrollment scales past hand-approval, and how re-sealing on enrollment behaves for large fleets, is open.
- **Registry-trigger loops.** Publish → run → publish is a cycle the subscription model makes expressible. Depth limits, provenance tags on trigger-initiated publishes, or refusing subscriptions that match a pipeline's own publish targets — something must be the guard, and "operators will be careful" is not it.
- **UI scope discipline.** Result stores grow dashboards, dashboards grow features. Whether the UI is versioned and shipped inside pnpr or as a separate artifact consuming only public endpoints decides how much pressure the server's release cadence absorbs; the separate artifact keeps the invariant-bearing component small and is the current lean.
- **The parity exception.** The agent shipping Rust-only is asserted in Implementation and needs explicit sign-off, since it is the first user-facing executable in the sequence exempted from the two-stack rule.
