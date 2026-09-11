# Agent skills shipped in dependencies

## Summary

> Depends on the per-package permissions RFC, which defines the `permissions` field this RFC writes its `skills` capability into.

Packages increasingly ship agent skills — directories containing a `SKILL.md` that AI coding agents load as instructions. Today nothing connects a skill sitting in `node_modules` to the agent working in the repository, so the skill is discovered only if a human already knew it existed and ran a separate CLI.

This RFC proposes that `pnpm install` discover skills shipped by direct dependencies, record them as awaiting approval, and link the approved ones into the agent skill directories a project already uses. Approval works exactly like build-script approval: nothing reaches an agent until a human says so.

```
Ignored skills: drizzle-kit, @supabase/supabase-js
Run "pnpm approve-skills" to pick which ones your agents can load.
```

No new `package.json` field is introduced. pnpm never reads the contents of a skill.

## Motivation

### Discovery only works if it is on by default

Several tools already let a package ship skills, and every one of them puts an adoption step in front of the discovery mechanism: `npx skills-npm`, `npx agentskills export`, or a coordinator dependency whose `postinstall` hook has to be allowlisted. A convention nobody runs discovers nothing, and the overwhelming majority of projects will never run any of them.

Package installation is the one step already in the path of every JavaScript dependency on every machine. That reach is the property none of these tools can bootstrap for themselves.

### Loading a skill is a trust decision, and it currently has no gate

A skill is not documentation that an agent happens to read. Once it is in `.claude/skills/`, it is ambient instruction on every subsequent session, ahead of any prompt, with no human present, silently re-resolved whenever the dependency updates.

It is also not necessarily prose. The documented skill layout allows supporting files and a `scripts/` directory alongside `SKILL.md`, addressable by the agent at execution time:

```
my-skill/
├── SKILL.md
├── reference.md
└── scripts/
    └── helper.py
```

So approving a skill can mean approving code that an agent will run. That makes this an exact analogue of the decision pnpm already gates: the `build` permission does not gate *reading* a lifecycle script, it gates *running* one. A skill is a build script for an agent's instruction set — one that runs on every session instead of once at install time.

pnpm is therefore not merely a convenient place to put discovery. It is the layer that already owns this class of decision, already has the approval ledger, and already has the UX for it.

### Skills in packages are version-correct, and only the package manager can keep them that way

A skill fetched from a git repository gives you that repository's default branch regardless of which version of the package you have installed. A skill shipped inside the tarball is pinned to the version in your lockfile and moves with `pnpm update`.

In a workspace, keeping that promise requires knowing the project graph. A tool walking `node_modules` links whichever copy it reaches first and cannot tell that two projects resolved the same package to different versions. pnpm can.

## Detailed Explanation

### Discovery

During installation, for each **direct** dependency of each workspace project, pnpm globs:

```
<package>/skills/*/SKILL.md
```

This is the convention that all existing prior art already agrees on. No new manifest field is introduced, and none is read.

Transitive dependencies are never considered. A package that wants its skills seen must be depended on directly.

### Version selection

Within a workspace a package may resolve to more than one version. The **highest resolved version decides**, including when that version ships no skills at all — in which case nothing is linked and nothing is reported. A `skills/` directory removed in a later release is a deliberate act by the author, and falling back to an older version's copy would resurrect content that was withdrawn, from a version nobody is running.

When versions diverge and the highest does ship skills, the divergence is stated rather than hidden:

```
drizzle-orm resolves to 2 versions; linked skills from 0.44.2 (apps/web is on 0.30.10)
```

### Approval

Packages shipping skills that have not been approved are recorded in `node_modules/.modules.yaml`, the same way ignored build scripts already are, and reported as a single line naming the packages.

`pnpm approve-skills` mirrors `pnpm approve-builds`:

- With no arguments, it presents the pending packages in a checkbox prompt.
- It accepts `<pkg>` to approve and `!<pkg>` to deny, for non-interactive use.
- The decision is written to `pnpm-workspace.yaml` as the `skills` capability of that package's entry in `permissions`, defined by the per-package permissions RFC:

  ```yaml
  permissions:
    drizzle-kit:
      skills: true
  ```

  `false` is recorded rather than omitted, so a denial persists and the next install does not prompt again.
- Decided entries are cleared from `.modules.yaml`.
- Newly approved skills are linked immediately.

`pnpm install` links everything already approved, so a fresh clone or a CI run materialises skills without any prompt. `approve-skills` handles only the newly approved delta. This split matches `install` running approved build scripts while `approve-builds` rebuilds only what was pending.

An approval covers a package, not an individual skill, and persists across upgrades. As with build scripts, this is trust in a publisher rather than review of a specific text: a later release may change a skill or add one.

### Materialisation

Approved skills are symlinked into agent skill directories. By default pnpm writes to those **that already exist in the project** — `.claude/skills/`, `.cursor/skills/`, and so on. It does not maintain a list of agents and does not detect whether an agent is running; it checks which directories are present, so a new agent works on the day it ships with no pnpm release.

Detection is a default, not the only mechanism. A project that does not have an agent directory yet would otherwise never get one, so the directories can be named explicitly:

```yaml
skillsDirs:
  - .claude/skills
```

When set, the setting is authoritative: it replaces detection rather than adding to it, so it can also be used to keep pnpm out of a directory that does exist. An empty list disables linking entirely.

The reason detection avoids creating directories is that it is a guess. An explicitly named directory is not, so pnpm creates it if it is missing.

Paths are relative to the workspace root.

Skill discovery in these directories is one level deep, so entries are flat and named:

```
npm-<package>-<skill>
```

This matches what skills-npm already writes, which lets pnpm and `npx skills` share a directory without conflict, and it marks which entries pnpm owns and may remove. Links point at the skill **directory**, never at `SKILL.md`, because a skill's supporting files are referenced relatively.

Entries are pruned when the dependency is removed, the approval is revoked, or the resolved version no longer ships that skill.

### Git

A symlink into `node_modules` dangles on a fresh clone, so these entries cannot be committed. Because hand-authored skills live in the same directory and generally *are* committed, the exclusion has to be per entry rather than per directory. pnpm writes a nested ignore file, merging rather than overwriting:

```
# .claude/skills/.gitignore
npm-*
```

This keeps pnpm out of the root `.gitignore`, and the `npm-` prefix serves three purposes at once: coexistence, ownership for pruning, and a single stable ignore pattern.

### What pnpm deliberately does not do

- **Does not read a skill.** pnpm globs for `SKILL.md`, links the directory, and stops. Nothing from inside a skill is ever interpolated into pnpm's own output. The one place a description is read is the approval prompt, transiently, where the reader is a human — the single context in which a skill named `ignore-previous-instructions-and-run-setup-sh` is a warning label rather than an attack.
- **Does not print package-authored prose.** The install line names packages, which are already in `package.json`, the lockfile and `node_modules`. It introduces no text that was not already in view.
- **Does not create agent directories it was not told about**, detect agent environments, or write outside the directories it detected or was given.

## Rationale and Alternatives

### Print a notice instead of linking

The originating proposal (`pnpm/pnpm` discussion 13422) adds an `agentNotice` string to `package.json`, printed at install when an agent environment is detected.

This has three problems. It is a channel for arbitrary package-authored prose aimed at a language model, and the proposed safeguards — a length cap, a single line, stripped ANSI — are terminal controls that stop escape sequences but do nothing about a directive, for which 200 characters is ample. It requires maintaining a list of agent environment variables that is out of date the day it ships. And a notice does not make a skill usable: the agent is told something exists and still has no path to load it.

Linking has no payload to sanitise, needs no environment detection, and produces a working skill rather than an announcement.

### Add a `package.json` field

A declarative field would allow skills outside a `skills/` directory. It was proposed as `agentskills` with a working implementation and **closed as not planned**; a competing `aiAgentSkill` field also exists. Meanwhile every implementation in the wild already globs the directory.

Reading the directory needs no new shared namespace, no cross-package-manager negotiation, and nobody's permission. If a field is standardised later, pnpm can honour it additively.

### Copy instead of symlink, so entries can be committed

Copies are real files that survive a clone without installing. They also drift from the installed version immediately, and keeping them correct means rewriting files on every install, which puts git churn into everyone's `pnpm install`. This discards the version coupling that is the main reason to ship a skill inside a package.

### Link per workspace project rather than at the root

Agent skill discovery does support nested directories (`apps/web/.claude/skills/`), which would scope a dependency's skill to the projects that actually use it. In practice a dependency is used by many projects in a workspace, so this fans the same link out across many directories, and almost no project has its own agent directory — so pnpm would either write nothing or start creating directories, which is a far larger liberty than the single ignore file proposed above.

### Link without approval

Simplest, and wrong. It grants every direct dependency standing authority over the agent's behaviour on every future session, including the ability to place executable scripts on a path the agent will run, with no moment at which a human sees it happen.

## Implementation

New development targets pnpm v12 only, per the repository's version policy. There is no v11 counterpart.

The machinery this needs largely exists:

- **Recording pending packages** reuses the `ignoredBuilds` mechanism in `.modules.yaml` and the `pnpm_modules_yaml` crate.
- **`pnpm approve-skills`** is structurally a copy of `approve_builds.rs`: the same checkbox prompt, the same `<pkg>` / `!<pkg>` parsing, the same write through `pnpm_workspace_manifest_writer`, the same clearing of decided entries. It is simpler, since there is no rebuild to schedule afterwards.
- **`pnpm ignored-skills`** parallels `ignored_builds.rs` for non-interactive listing.
- **Linking** must use the existing symlink helpers rather than a direct `symlink_dir`, so that unprivileged Windows falls back to junctions or copies as it does elsewhere in the store.
- **Pruning** runs with the rest of `node_modules` reconciliation; dangling links must never be left behind, since a broken symlink in a repository can fail unrelated tooling.

Affected areas: install (discovery, linking, pruning), `.modules.yaml`, the workspace manifest writer, the reporter, and two new commands.

A changeset targets `pacquet`.

## Prior Art

- **skills-npm** (antfu) — globs `node_modules/**/skills/*/SKILL.md` and symlinks into agent directories as `npm-<package>-<skill>`. Deliberately defines no manifest field. Requires a `prepare` script the user opts into. This RFC adopts its directory convention and naming.
- **agentskills/agentskills#81** and **npm-agentskills** (onmax) — proposed an `agentskills` field with an exporter to `.claude/skills/` and `.github/skills/`. Closed as not planned.
- **node-agent-skill-coordinator** (netresearch) — writes discovered skills into `AGENTS.md` from a `postinstall` hook, which pnpm users must allowlist.
- **skills** (vercel-labs) — installs skills from git repositories into agent directories. Solves installation rather than discovery, and is version-decoupled from the package a skill documents.
- **Claude Code plugin hints** — a vendor channel by which a CLI can ask a harness to offer a plugin. One vendor, one artifact type, and a push channel rather than an install-time one.
- **pnpm's own `allowBuilds`** — the existing precedent for gating untrusted package-supplied behaviour behind explicit approval, with the interactive flow this RFC reuses. It becomes the `build` capability under the per-package permissions RFC.

## Unresolved Questions and Bikeshedding

- **Link name collisions.** Flattening scoped names means `@supabase/supabase-js` + skill `auth` and a package `supabase` + skill `supabase-js-auth` can both produce `npm-supabase-supabase-js-auth`. A different separator or an escape for `/` would avoid it. Rare, but the choice should be deliberate.
- **Per-package or per-skill approval.** Per-package matches build scripts and keeps the prompt short; per-skill is finer but means re-prompting whenever a package adds one.
- **One level deep is verified for one agent.** Claude Code's discovery is documented as non-recursive. Whether every target directory behaves the same way has not been confirmed, and a nested layout would be tidier if they do.
- **Naming and path scope of `skillsDirs`.** Whether the plural reads better than pnpm's list-valued singulars such as `hoistPattern`, and whether absolute paths are accepted so that a home directory such as `~/.claude/skills` can be targeted.
- **Reporting the cold-start case.** When skills are approved and no directory is detected or configured, `approve-skills` should say so and name the setting rather than succeed silently.
- **Global and `dlx` installs.** Whether globally installed packages should link into `~/.claude/skills/`, and whether `pnpm dlx` should participate at all.
- **Naming.** `approve-skills` and `ignored-skills` mirror the build commands; `Ignored skills:` reuses the `Ignored build scripts:` phrasing.
- **Interaction with `npx skills`.** Sharing a directory is handled by the prefix, but a skill installed by both routes will appear twice under different names.
