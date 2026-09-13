# Agent skills shipped in dependencies

## Summary

> Depends on the per-package permissions RFC, which defines the `permissions` field this RFC writes its `skills` capability into.

Packages increasingly ship agent skills — directories containing a `SKILL.md` that AI coding agents load as instructions. Today nothing connects a skill sitting in `node_modules` to the agent working in the repository, so the skill is discovered only if a human already knew it existed and ran a separate CLI.

This RFC proposes that `pnpm install` discover skills shipped by direct dependencies, record them as awaiting approval, and link the approved ones into the agent skill directories a project already uses. Approval works exactly like build-script approval: nothing reaches an agent until a human says so.

```
Packages awaiting approval:
  drizzle-kit  build, skills
  esbuild      build

Run "pnpm approve" to review them.
```

Skills appear alongside build scripts in that one section rather than in a section of their own, per the permissions RFC.

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

It holds for the standalone collections too, which matters because they are installed straight from their repositories rather than repackaged. Across `anthropics/skills`, `vercel-labs/agent-skills`, `vercel-labs/skills`, `matthewp/tideway` and `supabase/agent-skills`, 35 of 36 `SKILL.md` files sit at `skills/<name>/SKILL.md`.

The one exception argues for this glob rather than against it: `anthropics/skills` keeps a `template/SKILL.md` for authoring a new skill. A recursive `**/SKILL.md` search would install that scaffold as a real skill. Matching one level under `skills/` skips it, and skips the `references/` and `rules/` subdirectories these skills carry, which are supporting files rather than skills of their own.

Transitive dependencies are never considered. A package that wants its skills seen must be depended on directly.

### Packages that are nothing but skills

Nothing in the above requires a package to ship code. A package whose entire content is `skills/*/SKILL.md` is discovered, approved and linked like any other, and this is arguably the more important case rather than a degenerate one: it gives the standalone skill collections that exist today — `vercel-labs/agent-skills`, `supabase/agent-skills` and their kind — a distribution channel with a lockfile pin, a version, an approval gate and an uninstall, none of which a `git clone` into a config directory provides.

Such a package is normally a `devDependency`, which is the correct scope: a `--prod` install has no agent to serve and links nothing.

A **git repository** of skills works with no manifest at all. `pnpm add github:owner/repo` installs a repository that has no `package.json`, synthesising one named after the repository at version `0.0.0`, so the skill repositories that exist today are installable as they are, unchanged.

The repository is keyed in `permissions` by pkgId rather than by name, so moving the pinned commit asks for approval again — which for a dependency whose whole payload is agent instructions is the behaviour worth having, and happens only on a deliberate `pnpm update`.

### Version selection

Within a workspace a package may resolve to more than one version. Comparable here has the same meaning it already has for build permissions: every resolution parses as `name@version` with a valid semver version, which is exactly the test that makes `allow_build_key_from_ignored_build` key the package by its bare name. When that holds, the **highest resolved version decides**, including when that version ships no skills at all — in which case nothing is linked and nothing is reported. A `skills/` directory removed in a later release is a deliberate act by the author, and falling back to an older version's copy would resurrect content that was withdrawn, from a version nobody is running.

Divergence is not reported during install. The rule is documented, the symlink resolves to a versioned path in the virtual store, and an install line that fires on every install in any workspace holding two versions of a skill-shipping package would be noise on the one surface that can least afford it. `pnpm permissions list` shows which version a linked skill came from, which is where someone asking the question is already looking.

### Approval

Packages shipping skills that have not been approved are recorded in `node_modules/.modules.yaml`, the same way ignored build scripts already are, and reported as a single line naming the packages.

Skills are granted through `pnpm permissions approve`, the single approval command defined by the per-package permissions RFC. This RFC deliberately does not add a per-capability command: `pnpm approve-builds` exists for backward compatibility, not because one command per capability is a good shape, and one install should produce one prompt covering everything a package is asking for.

The flow is the existing one:

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

`pnpm install` links everything already approved, so a fresh clone or a CI run materialises skills without any prompt. `pnpm permissions approve` handles only the newly approved delta, linking for a `skills` grant the way it schedules a rebuild for a `build` grant. This split matches what `install` and `approve-builds` already do between them.

**A pending skill never fails the install.** `strictDepBuilds` defaults to `true`, so an ignored build script fails the install with `ERR_PNPM_IGNORED_BUILDS` today. Skills do not join that: a skipped build script can leave a package unusable, whereas an unapproved skill only means an agent does not receive extra instructions. Nothing is broken, so nothing should fail. The practical consequence matters as much as the principle — sharing a pending section with build scripts would otherwise mean that adding any skill-shipping dependency reddens CI until somebody approves it.

**An approved skill is always materialised.** If pnpm cannot determine a target directory, cannot create one, or cannot create the link, the command fails and names `skills.dirs` as the fix. An approval that silently links nothing is worse than a failure: the grant is recorded, the file says the agent has the skill, and nothing tells anyone otherwise.

These two rules meet rather than conflict. A capability nobody granted is not pnpm's problem to enforce, so it warns. A capability somebody granted and pnpm cannot honour is a broken promise, so it fails.

This is why the nested `.gitignore` matters beyond keeping links out of commits. It is a real committed file, so the directory holding it survives a fresh clone, and CI detects the same target the developer approved against rather than failing on a directory that git could not carry.

The package is identified in `permissions` exactly as it is for build scripts, through `allow_build_key_from_ignored_build`: a bare name when the dep path is `name@version` with a valid semver version, and the full pkgId otherwise. A git-hosted or tarball dependency is therefore keyed by its source identity, and one package means the same thing across every capability rather than each one inventing its own key shape.

An approval covers a package, not an individual skill. As with build scripts, this is trust in a publisher rather than review of a specific text: a later release may change a skill or add one.

It persists across upgrades for a package keyed by name. A package keyed by pkgId is asked about again when its source identity changes, because a new commit, ref or tarball URL produces a different key. That is a consequence of the shared key shape rather than a separate rule, and it is the behaviour worth having: a different commit of a git dependency is different code, and re-approving it is the point of approving it at all.

### Materialisation

Approved skills are symlinked into agent skill directories. By default pnpm writes to those **that already exist in the project** — `.claude/skills/`, `.cursor/skills/`, and so on. It does not maintain a list of agents and does not detect whether an agent is running; it checks which directories are present, so a new agent works on the day it ships with no pnpm release.

A project that has no agent directory yet would never get one from that scan alone. Two things fill the gap.

**An agent that identifies itself gets its own directory.** When a known agent environment variable is set — `CLAUDECODE` and its equivalents — the directory for that agent is added to the set and created if missing. The agent running the install has named itself, which is a firmer basis for picking a directory than any inference.

This table is deliberately not load-bearing. A variable pnpm does not recognise costs nothing: the scan and the setting below still apply, so an out-of-date table degrades to the behaviour pnpm would have had anyway. Detection also never decides *whether* to link — only where. Skills are linked in CI and in a plain terminal exactly as they are under an agent, or a fresh clone would silently differ from the machine the install was first run on.

The set of target directories is therefore the union of those detected on disk and the one named by the environment, and the first run under a new agent creates the directory that every later run then finds by scanning.

**The directories can also be named explicitly:**

```yaml
skills:
  dirs:
    - .claude/skills
```

Settings for this feature live under a `skills` section rather than as flat top-level keys, following `update:` and `audit:`, which pnpm already groups this way — `updateConfig` was superseded by `update:` for exactly this reason. The unresolved questions below imply siblings already: whether global and `dlx` installs participate, and whether pnpm manages the nested `.gitignore` or leaves that to the project.

When set, the setting is authoritative: it replaces both the scan and environment detection, so it can also be used to keep pnpm out of a directory that does exist.

An empty list disables the capability rather than merely linking. Skills stop being offered for approval at all, so no grant can exist that pnpm is then unable to honour, and anything already materialised is pruned on the next install. This is what keeps the always-materialise rule from contradicting itself: that rule fires when pnpm cannot find a target, never when the project has said there is not one. Grants already recorded stay in the file and do nothing.

The scan avoids creating directories because the absence of one carries no instruction. An explicitly named directory and a self-identifying agent both do, so either causes pnpm to create it.

Paths are relative to the workspace root.

Skill discovery in these directories is one level deep, so entries are flat and named:

```
pnpm-<package>-<skill>
```

The package segment is escaped the way pnpm already escapes one for the virtual store, so `@supabase/supabase-js` becomes `@supabase+supabase-js` — `dep_path_to_filename` is the existing implementation. Without it, `@supabase/supabase-js` with a skill `auth` and a package `supabase` with a skill `supabase-js-auth` would flatten to the same name, and which link survived would depend on installation order.

Escaping removes the scoped case but not every one: two unscoped packages can still collide across the package-skill boundary. pnpm therefore also detects a collision before writing and fails, rather than letting one approved skill silently replace another.

The package segment comes from whichever part of the source's identity is actually unique.

- **A registry package** uses its package name. npm guarantees these are unique, so nothing further is needed.
- **A git-hosted package** uses its package name too, which is what its manifest declares, or `@owner/repo` when it has no manifest. Both are unique, so nothing further is needed here either. This relies on the git naming change described below.
- **Anything else** — a tarball URL, a `file:` dependency — uses the dependency key the project declared, since these have no short unique identity to derive one from.

The pkgId is never used: `pnpm-foo@https+++codeload.github.com+user+repo+abc123-migrations` is a directory name nobody can read, and the link is a label rather than an identity. The collision check above remains as a backstop for whatever this scheme still fails to separate.

Two properties are worth the extra rule. The names are **stable** — adding a second vendor never renames the first vendor's links, which a disambiguate-only-on-collision scheme could not promise, and an agent's skill directory changing underneath it is worse than a long name. And they are **traceable**: `pnpm-vercel-labs-agent-skills-composition-patterns` says where the skill came from, which a bare `agent-skills` does not.

Deriving from the declared key everywhere was considered and rejected. A key is unique within one `package.json`, so it separates two vendors inside a single project, but it says nothing across a workspace: two projects may each declare `agent-skills` for different repositories, and the collision would return. Keying on names that merely *look* like skill collections — matching `skills` as a substring — was also rejected: it would catch `my-skills-helper` and miss `agent-kit`, and it makes the naming rule depend on what a repository happens to be called.

That leaves one case the version rule cannot arbitrate. Two sources can supply the same package name — a fork pinned by git in one project and the registry copy in another — and their versions are not comparable, since two git refs can each declare `1.0.0`. "Highest version wins" has no meaning across them, so neither is preferred and both materialise.

This needs no rule of its own: it resolves through the collision check above, which compares generated link names rather than package names. Two sources shipping different skills produce different names and both link. Two shipping a skill of the same name produce the same link name and fail, which is the same failure any other collision produces.

The prefix is pnpm's own, not the `npm-` that skills-npm writes. Sharing that prefix would have made the two tools produce the same name for the same skill, which is a collision rather than coexistence, and would have left pnpm unable to tell its own entries from another tool's when pruning.

The prefix marks intent, but it cannot establish ownership on its own, because an entry is not always a symlink: the Windows fallback produces a junction or a copy, and a copy carries no target to inspect. pnpm therefore records the entries it materialises in `.modules.yaml`, and prunes exactly those. Anything it did not record is left alone whatever its name or type, so a hand-authored skill, or one written by another tool, is never replaced or deleted.

Links point at the skill **directory**, never at `SKILL.md`, because a skill's supporting files are referenced relatively.

Every install reconciles the recorded entries against the directories currently being targeted, so an entry is pruned when the dependency is removed, the approval is revoked, the resolved version no longer ships that skill, or the directory holding it is no longer a target — whether because `skills.dirs` now names a different directory or because it is empty. Disabling removes what was materialised rather than stranding it: a revoked or disabled skill that stays on disk keeps being loaded, which is the failure the approval gate exists to prevent.

### Git

A symlink into `node_modules` dangles on a fresh clone, so these entries cannot be committed. Because hand-authored skills live in the same directory and generally *are* committed, the exclusion has to be per entry rather than per directory. pnpm writes a nested ignore file, merging rather than overwriting:

```
# .claude/skills/.gitignore
pnpm-*
```

This keeps pnpm out of the root `.gitignore`, and the `pnpm-` prefix serves three purposes at once: staying clear of other tools' entries, marking ownership for pruning, and giving the ignore rule one stable pattern.

### What pnpm deliberately does not do

- **Does not read a skill.** pnpm globs for `SKILL.md`, links the directory, and stops. It does not open the file, including for the approval prompt, which identifies a package and the names of the skill directories it ships. Those names are what a human needs in order to decide whether to trust the publisher, and stopping at them keeps the rule absolute rather than qualified: no content from inside a skill reaches any pnpm output, ever.
- **Does not print package-authored prose.** The install line names packages, which are already in `package.json`, the lockfile and `node_modules`. It introduces no text that was not already in view.
- **Does not create agent directories it was not told about**, and does not use the environment to decide whether a project gets skills at all — only which directory an agent that identified itself should receive them in.

### Two git-dependency changes this depends on

Neither is specific to skills, but skill collections make both certain to be hit, because they are near-uniformly named `agent-skills` or `skills`.

**A repository without a manifest is named `@owner/repo`.** pnpm synthesises such a name from the repository name alone today, so `anthropics/skills` and `vercel-labs/skills` both become `skills`, and the two cannot coexist in one `package.json`. Scoping by owner makes the name unique and says where the code came from, and it is the shape a repository author picks unprompted: `vercel-labs/agent-skills` declares `@vercel-labs/agent-skills` by hand. The synthesised name was never a published contract, so nothing breaks that was promised, but every existing manifest-less git dependency is renamed and its lockfile entry rewritten. Hosts that do not expose an owner segment keep the bare repository name, and the result is lowercased and sanitised to be a valid package name.

**`pnpm add` does not silently replace a dependency that resolves to a different source.** Today the second of these discards the first with no warning, reported as though it were an ordinary version change:

```
$ pnpm add github:anthropics/skills
$ pnpm add github:vercel-labs/skills
  - skills https://codeload.github.com/anthropics/skills/...
  + skills 1.5.26
```

Changing the version of a dependency stays silent, as it should. Replacing one whose specifier points at a different source does not: that is a removal the user did not ask for, and the fix — aliasing one of them — is something pnpm can name at the point of failure.

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
- **No new commands.** The `pnpm permissions` commands come from the permissions RFC; this RFC adds `skills` as a capability they already handle. The post-approval action dispatches per capability: linking here, a rebuild for `build`.
- **Linking** must use the existing symlink helpers rather than a direct `symlink_dir`, so that unprivileged Windows falls back to junctions or copies as it does elsewhere in the store.
- **Pruning** runs with the rest of `node_modules` reconciliation; dangling links must never be left behind, since a broken symlink in a repository can fail unrelated tooling.

Affected areas: install (discovery, linking, pruning), `.modules.yaml`, the workspace manifest writer, the reporter, and two new commands.

A changeset targets `pacquet`.

## Prior Art

- **skills-npm** (antfu) — globs `node_modules/**/skills/*/SKILL.md` and symlinks into agent directories as `npm-<package>-<skill>`. Deliberately defines no manifest field. Requires a `prepare` script the user opts into. This RFC adopts its directory convention, but not its link naming: a shared prefix would collide on the same skill and would make the two tools' entries indistinguishable when pruning.
- **agentskills/agentskills#81** and **npm-agentskills** (onmax) — proposed an `agentskills` field with an exporter to `.claude/skills/` and `.github/skills/`. Closed as not planned.
- **node-agent-skill-coordinator** (netresearch) — writes discovered skills into `AGENTS.md` from a `postinstall` hook, which pnpm users must allowlist.
- **skills** (vercel-labs) — installs skills from git repositories into agent directories. Solves installation rather than discovery, and is version-decoupled from the package a skill documents.
- **Claude Code plugin hints** — a vendor channel by which a CLI can ask a harness to offer a plugin. One vendor, one artifact type, and a push channel rather than an install-time one.
- **pnpm's own `allowBuilds`** — the existing precedent for gating untrusted package-supplied behaviour behind explicit approval, with the interactive flow this RFC reuses. It becomes the `build` capability under the per-package permissions RFC.

## Unresolved Questions and Bikeshedding


- **Per-package or per-skill approval.** Per-package matches build scripts and keeps the prompt short; per-skill is finer but means re-prompting whenever a package adds one.
- **One level deep is verified for one agent.** Claude Code's discovery is documented as non-recursive. Whether every target directory behaves the same way has not been confirmed, and a nested layout would be tidier if they do.
- **Path scope of `skills.dirs`.** Whether absolute paths are accepted, so that a home directory such as `~/.claude/skills` can be targeted, or whether the global-install question below should settle that instead.
- **Whether disabling deserves its own key.** `skills.dirs: []` disables linking today, which works but reads obliquely next to an explicit `skills.enabled: false`.
- **How many agents to recognise.** The environment table maps a variable to a directory, which is narrower than knowing whether some agent is running, but it still has to be maintained. How many entries are worth carrying before the explicit setting is the better answer is an open question.
- **Global and `dlx` installs.** Whether globally installed packages should link into `~/.claude/skills/`, and whether `pnpm dlx` should participate at all.
- **Reporting wording** is settled in the permissions RFC, which replaces `Ignored build scripts:` with one section covering every pending capability.
- **Interaction with `npx skills`.** Distinct prefixes keep the two tools from overwriting or pruning each other, at the cost of a skill installed by both routes appearing twice under different names.
