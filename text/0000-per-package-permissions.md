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

The general test, now that pnpm has two grouping shapes, is which question the setting answers:

- **Package-major**, like this field: facts *about a package*, where the set of things you can say is open-ended. "What may esbuild do" is a question about esbuild.
- **Feature-major**, like the `update:` and `audit:` sections: parameters *of a feature* that happen to name packages. "Why is `@babel/*` not being delayed" is a question about the release-age check, not about babel.

That test also explains why the key shapes differ. A permission is granted to a specific package identity somebody reviewed, so its keys are exact. A policy exemption covers a class of packages whose publisher is trusted wholesale, so its keys are globs.

### Capabilities

- **`build`** — run lifecycle scripts. Replaces `allowBuilds`.
- **`skills`** — contribute agent skills. Defined by the agent skills RFC, which depends on this one.

New capabilities are added as keys. An unknown capability key is reported through the existing unknown-settings path rather than ignored, so a typo is visible rather than silently denying.

`dangerouslyAllowAllBuilds` stays a separate global setting. It is not a statement about a package, so it does not belong in a package-keyed map.

### Migration

`allowBuilds` continues to be read, and is folded into the `build` capability. When pnpm next writes a decision, it writes `permissions` and clears the legacy key, which is what `set_allow_builds_clearing_legacy` already does one generation back. A project that never runs an approval keeps working without ever being rewritten.

Precedence, when both are present for the same package: `permissions` wins, and the duplicate is reported.

### Reporting

Capabilities awaiting approval are reported in one section rather than one per capability, since a single install produces a single pending set and the package is the unit of the decision:

```
Packages awaiting approval:
  drizzle-kit  build, skills
  esbuild      build

Run "pnpm approve" to review them.
```

This replaces `Ignored build scripts: …`, whose wording does not extend. An ignored build script is one that did not run; an unapproved skill was never going to do anything on its own, so "ignored" describes the wrong thing.

Being listed in that section does not imply that the install fails. Strictness is per capability, and `build` is the only capability that carries it: `strictDepBuilds` defaults to `true`, so an ignored build script fails the install, while a pending `skills` grant only warns. A skipped build script can leave a package unusable; an ungranted capability that withholds instructions from an agent breaks nothing. A capability that failed the install merely for being unapproved would make adding a dependency that requests it redden CI.

Two things survive the rename. The text appears twice today — as a notice in `default-reporter`, and as an install failure in `package-manager` when `strictDepBuilds` is set — and both move together. `ERR_PNPM_IGNORED_BUILDS` keeps its code even though its message changes, because the code is the stable identity that CI matches on.

### Approval

One prompt per install instead of one per capability. A package appears once with everything it is requesting:

```
  drizzle-kit  build, skills
  esbuild      build
```

The commands live under a `permissions` namespace, named for the field:

- **`pnpm permissions approve`**, aliased to **`pnpm approve`** — the interactive prompt above.
- **`pnpm permissions list`**, or a bare `pnpm permissions` — the current state: granted, denied, and awaiting approval. This is the per-package question the field is organised around, and a better home for per-capability auditing than the file layout.

The namespace exists because listing needs a home and because pnpm qualifies approval by its object elsewhere: `pnpm stage approve` approves a staged publish. The alias exists because this is the most-travelled path in the whole feature, and `pnpm permissions approve` is longer to type than the `pnpm approve-builds` it replaces. pnpm aliases freely — `i`, `up`, `ls`, `rm`, `rb` — so the short form is the idiomatic answer rather than a compromise. The install hint prints `pnpm approve`, since that is the line people copy.

The verb stays `approve`: it is already the verb in `approve-builds`, in `stage approve`, and in the prompt's own wording.

Approving dispatches per capability afterwards: a `build` grant schedules a rebuild, a `skills` grant links the skill. That is why one command is workable at all — the prompt is shared, the consequence is not.

The grant is written before its action runs, and a failing action fails the command without rolling the grant back. The user did approve; it is the environment that is wrong. Because `pnpm install` performs the same action for every already-granted capability, the failure recurs on each install until it is fixed, rather than being a one-time error that leaves a grant nothing enforces.

**New capabilities do not get their own commands.** `pnpm approve-builds` and `pnpm ignored-builds` remain as flat aliases for the `build`-filtered views, so existing muscle memory, documentation and CI scripts keep working, but they are compatibility surface rather than a pattern to extend. A user who wants to review one capability in isolation filters the unified command rather than learning a new verb per capability.

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
- `approve_builds.rs`: generalise the pending/prompt/write flow over a capability set rather than assuming build scripts, move it under the `permissions` namespace, and keep `approve-builds` as a filtered alias.
- `.modules.yaml`: `ignoredBuilds` gains a sibling for other capabilities, or generalises, so pending state is capability-aware.
- `default-reporter/src/state/notices.rs` and `package-manager/src/install/errors.rs`: the two places carrying the `Ignored build scripts:` wording, which change together.
- Unknown-capability reporting joins the existing unknown-settings path.

v12 only, per the version policy. A changeset targets `pacquet`.

## Prior Art

- **pnpm's own `onlyBuiltDependencies` → `allowBuilds`** — the precedent for migrating a setting in this area, including the legacy-clearing write this RFC extends.
- **Deno's permission flags** — capability grants named per resource, granted explicitly, denied by default.
- **VS Code workspace trust** — a single trust decision per workspace that gates several distinct capabilities, rather than one prompt per capability.
- **Browser permissions** — grouped per origin, with an explicit denied state that suppresses re-prompting, which is the behaviour `false` preserves here.

## Unresolved Questions and Bikeshedding

- **The policy exemptions do not join, but they have their own consolidation.** `minimumReleaseAgeExclude` and `trustPolicyExclude` are per-package trust decisions, so merging them here is tempting. They fail the test above: both are parameters of a check rather than facts about a package, both are globs where these keys are exact, and `minimumReleaseAge: false` reads backwards as a permission. The grouping they want is feature-major — `minimumReleaseAge: { minutes, exclude, excludePrune }` — which also puts `minimumReleaseAgeExcludePrune` next to what it prunes instead of leaving two flat siblings that only make sense together. That is a separate, smaller cleanup, with one wrinkle: `minimumReleaseAge` is a scalar today, so a section form has to keep accepting `minimumReleaseAge: 1440` as shorthand.
- **Command naming.** `permissions` is plural to match the settings field, where most pnpm namespaces are singular (`config`, `access`, `stage`). Whether the capability filter is a flag (`--capability`) or positional. Whether revoking a granted permission is a third verb or a pass through `approve`.
- **Long pending lists.** Whether the section truncates, and at what point, given that today's single line simply wraps.
- **Whether `dangerouslyAllowAllBuilds` generalises** to a per-capability escape hatch, or stays specific to builds.
- **Capability naming.** `build` and `skills` are nouns describing the artifact; `runScripts` and `provideSkills` would be verbs describing the act.
