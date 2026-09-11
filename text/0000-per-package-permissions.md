# Per-package permissions

## Summary

`allowBuilds` records which packages may run lifecycle scripts. It is the first of a family: a package can now also ship agent skills, and MCP servers are the obvious next case. Each new capability currently means another top-level setting with the same shape, the same lifecycle, and its own approval command.

This RFC replaces that family with a single package-keyed field:

```yaml
permissions:
  esbuild:
    build: true
  drizzle-kit:
    build: true
    skills: true
```

`allowBuilds` migrates into it as the `build` capability, the way `onlyBuiltDependencies` already migrated into `allowBuilds`.

## Motivation

### The family is about to grow

pnpm denies by default and asks. That pattern now has more than one subject:

- **`build`** — may run lifecycle scripts. Shipped, as `allowBuilds`.
- **`skills`** — may contribute instructions an agent loads on every session, including executable scripts it can run. Proposed in the agent skills RFC.
- Plausibly next: MCP servers declared by a package, and whatever else packages start shipping for agents.

Adding each as its own top-level setting gives N settings with an identical shape (`package → bool`), an identical lifecycle (recorded as pending during install, resolved by a prompt, written to `pnpm-workspace.yaml`), and N separate approval commands that each fire after the same install.

### The question users ask is per package

"What has this package been allowed to do?" is the question behind every one of these settings, and with parallel maps it is answered by reading N places and assembling the result. Grouping by package answers it by reading one block.

It also matches how the decision arises. A package is installed, it requests things, a human grants or denies them. That is one moment about one package, not N moments about N capabilities.

### Doing it later costs more than doing it now

pnpm already carries a legacy path for one rename in this area: `set_allow_builds_clearing_legacy` handles `onlyBuiltDependencies` → `allowBuilds`. If the skills capability ships as `allowSkills` first, a later consolidation migrates two settings instead of one, and users who adopted `allowSkills` in between are moved twice.

## Detailed Explanation

### Shape

```yaml
permissions:
  <package>:
    <capability>: true | false
```

Keys are package names, or `name@version`, exactly as `allowBuilds` keys are today. The value is a map of capability to boolean.

`false` is meaningful and must be recorded: it is how a denial persists so that the next install does not prompt again.

Denials are not the rare case. The interactive prompt records `false` for every pending package the user does not tick, so approving two of five pending packages writes two `true` and three `false`. A typical file holds more denials than grants.

Entries are written in block style, one capability per line, rather than as a flow mapping. `pnpm-workspace.yaml` is committed and reviewed, and block style makes a new grant a single added line instead of a rewrite of the package's entire entry.

### Scope rule

**`permissions` holds capability grants, not content modifications.**

A capability grant answers "what may this package do to my machine". `overrides`, `packageExtensions`, `patchedDependencies`, `configDependencies` and `ignoredOptionalDependencies` answer "what is this package, and is it installed at all" — a different question, and one for which a permission reads as nonsense. They stay where they are.

### Capabilities

- **`build`** — run lifecycle scripts. Replaces `allowBuilds`.
- **`skills`** — contribute agent skills. Defined by the agent skills RFC, which depends on this one.

New capabilities are added as keys. An unknown capability key is reported through the existing unknown-settings path rather than ignored, so a typo is visible rather than silently denying.

`dangerouslyAllowAllBuilds` stays a separate global setting. It is not a statement about a package, so it does not belong in a package-keyed map.

### Migration

`allowBuilds` continues to be read, and is folded into the `build` capability. When pnpm next writes a decision, it writes `permissions` and clears the legacy key, which is what `set_allow_builds_clearing_legacy` already does one generation back. A project that never runs an approval keeps working without ever being rewritten.

Precedence, when both are present for the same package: `permissions` wins, and the duplicate is reported.

### Approval

One prompt per install instead of one per capability. A package appears once with everything it is requesting:

```
  drizzle-kit  build, skills
  esbuild      build
```

`pnpm permissions` lists the current state — granted, denied, and awaiting approval — which is the per-package question the field is organised around, and the better home for per-capability auditing than the file layout.

Approving dispatches per capability afterwards: a `build` grant schedules a rebuild, a `skills` grant links the skill. That is why one command is workable at all — the prompt is shared, the consequence is not.

**New capabilities do not get their own commands.** `pnpm approve-builds` and `pnpm ignored-builds` remain as views filtered to the `build` capability, so existing muscle memory, documentation and CI scripts keep working, but they are compatibility surface rather than a pattern to extend. A user who wants to review one capability in isolation filters the unified command rather than learning a new verb per capability.

## Rationale and Alternatives

### Keep one flat setting per capability

`allowBuilds`, `allowSkills`, and so on. Least disruption today, and the most compact form for the single-capability case — one line per package instead of three.

It multiplies settings, prompts and commands with every capability, and it answers the per-package question worst. The compactness is also recoverable with a shorthand if it proves to matter (see below).

### Group by capability instead of by package

```yaml
permissions:
  build:
    esbuild: true
  skills:
    drizzle-kit: true
```

Better for auditing one capability across all packages, and a mechanical migration from `allowBuilds`. But approving a single package writes into N places, which is the wrong shape for the moment the decision is actually made, and it is exactly as verbose as separate settings while adding a level of nesting. Per-capability auditing is better served by a command than by the file layout.

### A list of granted capabilities

```yaml
permissions:
  drizzle-kit: [build, skills]
```

Compact, and the most attractive of the alternatives at first glance. It fails on denials, which are the majority of what gets written: a list of grants can only express "denied" by omission, which is indistinguishable from "not yet decided", so every install would re-prompt for things the user already refused. A fully denied package would be written `drizzle-kit: []`, which reads as no opinion rather than as a decision.

Adding a negation marker rescues the semantics but not the ergonomics. `!` is the natural choice, since `pnpm approve-builds` already accepts `!<pkg>` on the command line, but `!` introduces a **tag** in YAML — `[build, !skills]` does not parse as intended and has to be quoted as `"!skills"`. That puts an escape character on the majority of entries in the file.

### Defer until a third capability exists

Reasonable in isolation, but the second capability is the moment the cost of waiting starts compounding, because the alternative is shipping `allowSkills` and migrating it later.

## Implementation

- `pnpm-workspace-manifest-writer`: block-style emission for these entries, so an approval adds lines rather than rewriting them. The existing `flow.rs` single-line splicing is deliberately not used here.
- `pnpm_config`: a `permissions` setting; `allow_builds` becomes a legacy input folded into it rather than a separate consumer-facing map. `AllowBuildPolicy::from_config` reads the `build` capability.
- `pnpm_workspace_manifest_writer`: extend the existing legacy-clearing write so it targets `permissions` and clears `allowBuilds`, alongside the `onlyBuiltDependencies` handling already there.
- `approve_builds.rs`: generalise the pending/prompt/write flow over a capability set rather than assuming build scripts, and keep `approve-builds` as a filtered entry point.
- `.modules.yaml`: `ignoredBuilds` gains a sibling for other capabilities, or generalises, so pending state is capability-aware.
- Unknown-capability reporting joins the existing unknown-settings path.

v12 only, per the version policy. A changeset targets `pacquet`.

## Prior Art

- **pnpm's own `onlyBuiltDependencies` → `allowBuilds`** — the precedent for migrating a setting in this area, including the legacy-clearing write this RFC extends.
- **Deno's permission flags** — capability grants named per resource, granted explicitly, denied by default.
- **VS Code workspace trust** — a single trust decision per workspace that gates several distinct capabilities, rather than one prompt per capability.
- **Browser permissions** — grouped per origin, with an explicit denied state that suppresses re-prompting, which is the behaviour `false` preserves here.

## Unresolved Questions and Bikeshedding

- **Do the policy exemptions join?** `minimumReleaseAgeExclude` and `trustPolicyExclude` are per-package trust decisions and belong here by intent. Two things block a clean merge: they are **glob** patterns (`@babel/*`) where `allowBuilds` keys are exact, so one map would have to settle whether `foo@1.0.0` is a key or a pattern; and they are exemptions rather than grants, so `minimumReleaseAge: false` reads backwards. A grant-shaped name (`installBeforeMinimumAge: true`) reads correctly but is clumsy. Both also have pruning machinery tied to their list shape. This RFC proposes leaving them out initially and revisiting once the grant vocabulary is settled.
- **Command naming.** `pnpm approve` is unqualified, and `pnpm stage approve` already uses the verb in another namespace. Whether the filter is a flag (`--capability`) or positional. Whether `pnpm permissions` should also be the command that revokes one, or whether revocation stays inside `approve`.
- **Whether `dangerouslyAllowAllBuilds` generalises** to a per-capability escape hatch, or stays specific to builds.
- **Capability naming.** `build` and `skills` are nouns describing the artifact; `runScripts` and `provideSkills` would be verbs describing the act.
