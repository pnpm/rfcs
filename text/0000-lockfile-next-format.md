# One lockfile entry per package, with peer bindings as properties

## Summary

In the next lockfile format, `packages` has one entry per package, keyed by
`name@version`. The entry holds the package's metadata, its resolved
dependencies, and its peer bindings as properties: `peerDependencies` holds
each declared peer's range and the version it was bound to, and, marked
`below: true`, the bindings a dependency below the package needed. There is no `snapshots` section. A package that pnpm installs with
several sets of peers has one snapshot per set. The entry writes what the
snapshots share once and lists what differs under `snapshots`. A reference
is a version. The referrer's own bindings determine which snapshot it links;
where they don't, the reference names the snapshot by its peer bindings. No
key or reference carries a peer or patch group.

pnpm v12 reads and writes this format behind the opt-in `lockfile.format: v13`
setting (pnpm/pnpm#17886, merged). That implementation round-trips 142 of the
146 real lockfiles it was tried on exactly, and writes a 5,623-snapshot
lockfile in 55% of today's bytes. See [Prototype results](#prototype-results).

The format also requires every directory package's `packages` entry to record
`specifiers`, the dependency specifiers the resolver read from its manifest,
folds `peerDependenciesMeta` into `peerDependencies`, and writes a project's
dependency as its specifier alone when the version follows from it. The file
is written as JSON with a trailing comma after every entry, which is valid
YAML, so `pnpm-lock.yaml` keeps its name and its readers.

## Motivation

These problems came up while writing a specification of the lockfile format
for the pnpm website (`pnpm/docs/lockfile-format.md` in pnpm/pnpm), and while
reading the lockfile diff of pnpm/pnpm#17773.

### The peer suffix is repeated at every reference

A snapshot key embeds the versions of the peers the snapshot binds, and every
reference to the snapshot repeats that key. When a provider's version changes,
every key that embeds it is renamed, and every reference to each of those keys
is rewritten.

In pnpm/pnpm#17773, a minor release of one package added a dependency on
`bare-buffer`. The lockfile diff is 309 lines. 10 of them are snapshot keys
and 129 are references to those keys. 13 mention `bare-buffer` at all.

The repetition also takes space. In the lockfile of a large front-end
monorepo, suffixes are 26% of the file (4.4 MB, 3.3 MB without them), on
19% of the lines.

### A clean merge can produce a broken lockfile

Branch A bumps `buffer-impl` from 3.7.1 to 3.7.2. Branch B adds a package
that depends on `fs-impl`, and writes `fs-impl: 4.8.4(buffer-impl@3.7.1)` in a
part of the file branch A never touched. Git merges the two without a
conflict. The merged file references a snapshot that branch A renamed. No
conflict marker says so, and the next install has to find out and resolve
again.

The reference carries a fact that another branch can change. A reference that
carries only the version cannot go stale this way.

### Optional peers are recorded as optional dependencies

A snapshot lists a resolved peer as a child, because the provider has to be
linked next to the package. A required peer goes under `dependencies`. An
optional peer goes under `optionalDependencies`:

```yaml
packages:
  fs-impl@4.8.4:
    resolution: {integrity: sha512-...}
    peerDependencies:
      buffer-impl: '*'
    peerDependenciesMeta:
      buffer-impl:
        optional: true

snapshots:
  fs-impl@4.8.4(buffer-impl@3.7.1):
    dependencies:
      path-impl: 3.1.2
    optionalDependencies:
      buffer-impl: 3.7.1
```

`fs-impl` does not declare `buffer-impl` as an optional dependency. The
snapshot says it does. An optional dependency and an optional peer dependency
are different relationships. The first is installed if it can be. The second
is linked only if something else installs it. A reader can tell them apart
only by checking each child against the `packages` entry's `peerDependencies`.

### A dependency that is also a peer is ambiguous

A package can declare the same name as a dependency and as a peer. If a parent
provides the name, pnpm binds it as a peer. Otherwise the package's own
dependency is installed as a child. Both outcomes are written the same way:

```yaml
snapshots:
  shared@file:shared(widget@2.0.0):
    dependencies:
      widget: 2.0.0
```

The snapshot doesn't say whether `widget` is the parent's version or the
package's own. Code that reads only the lockfile, such as `pnpm deploy` and
lockfile reuse, has to reconstruct that decision from the key's peer suffix.
pnpm/tasks#221 tracks this.

### A hashed suffix cannot be read back

When the peer suffix is longer than `peersSuffixMaxLength`, pnpm replaces it
with a hash. The key then names no peers. To find out which providers a
hashed key stands for, pnpm walks the snapshot's descendants. When the walk is
not enough, it searches the subsets of the declared peers for one whose hash
matches the key. That search is bounded, so it can fail. pnpm/pnpm#17320 was
its cost. A tool that reads the lockfile has to do the same.

### transitivePeerDependencies mixes two cases

`transitivePeerDependencies` lists the peers that packages below a snapshot
need and that the subtree does not provide. A name in the list is in one of
two states, and the list does not say which:

- Resolved from outside the package. The provider is part of the key suffix.
- Not resolved at all. It is not part of the key.

### A directory package's inputs are not recorded

A project records each dependency's `specifier` beside its `version` in
`importers`, so the
frozen-lockfile check compares strings. A directory package, such as an
injected workspace package or a `file:` directory, records only its resolved
children in `snapshots`. To find out whether its manifest changed, the check
reads the manifest, runs the hooks, and derives the children again, with
catalogs and peer shadowing. That is a second resolver, and it has needed a
fix each time the resolver gained an input. pnpm/tasks#228 lists 21 such
fixes.

pnpm/pnpm#17477 adds the `specifiers` field to lockfile 9.0, as an optional
field. A lockfile written before it has no `specifiers`, so the check keeps the
derived walk as a fallback.

## Detailed Explanation

### Syntax

The file is JSON, with a trailing comma after every entry of a collection
that spans lines:

```json
{
  "lockfileVersion": "13.0",

  "projects": {
    ".": {
      "dependencies": {
        "left-pad": "1.3.0",
      },
    },
  },

  "packages": {
    "left-pad@1.3.0": {
      "integrity": "sha512-...",
    },
  },
}
```

That is YAML 1.2 flow syntax with every scalar double-quoted, so a YAML
reader parses the file as the same document: `pnpm-lock.yaml` keeps its
name, and a tool that reads it as YAML today keeps working once it knows
the new shape. pnpm reads it with a JSON parser that allows the trailing
comma, which parses the text in a quarter of the time a YAML parser takes,
and writes it with one quoting rule. Since every key and string is quoted,
no reader applies YAML's implicit typing: `yes` and `1.10` are the strings
they look like.

The trailing comma is what lets git treat the last entry of a map like any
other: adding or removing it changes one line. A tab indents each level:
inside a flow collection YAML allows a tab as separation, every reader of
`pnpm-lock.yaml` that was tried accepts it, and it makes the file a tenth
smaller than two spaces would. A blank line separates the entries of the
top-level map and of `projects` and `packages`. Every map other than fixed
package metadata (`resolution` unless it has `variations` or `binary`,
`engines`, `cpu`, `os`, and `libc`) is written across lines, one entry per
line, closing with its own `},` line: a dependency's `specifier` and
`version`, a peer's `specifier`, `version`, `optional`, and `below`, a
catalog entry's `specifier` and `version`, and `with`. That puts each field
on its own line between the entry's `"<name>": {` and `},` lines, so two
branches that change adjacent map entries (such as the `version` of two
adjacent ranged dependencies or catalog entries) or fields separated by an
unchanged line merge without a conflict. The examples below are fragments
of such a file, with their indentation reduced.

A lockfile with a boot document, the first document of a two-document
`pnpm-lock.yaml` that records `configDependencies` and
`packageManagerDependencies`, writes that document in the format of the
pnpm major that owns the project: the pinned one, or the running one when
nothing is pinned. The boot document is what a pnpm that is not the pinned
version reads to find the pinned version and switch to it, and that pnpm
may be older than this format, so the boot document changes format one
major after the main document: a project that pins pnpm 13 or earlier
keeps lockfile 9.0's boot document whatever format the main document is
in, and a project that pins pnpm 14 or later has a boot document in this
format. Keying the format on the pinned major rather than on the writer
matters because either pnpm may write the document: the pinned pnpm when
it records its own pin or installs config dependencies, and the pnpm
switching into the project on the first switch, when it resolves the
pinned version's packages before installing them. Both write what the
pinned pnpm would. The two documents stay separated by `---`, the main
one in the format `lockfileVersion` names.

pnpm 11 and pnpm 12 read a boot document in this format ahead of any pnpm
writing one, and pnpm 12 already writes it for a project that pins pnpm 14
or later. pnpm 14 rewrites a boot document it finds in another format than
the project's, which repairs one written by a pnpm 11 that made the first
switch into the project.

A pnpm 11 or 12 released before that reader takes a boot document in this
format for a missing one. It then resolves the pinned version from the
registry instead of from the lockfile, as it does for a project with no
boot document, and a frozen install fails with
`ERR_PNPM_FROZEN_LOCKFILE_WITH_OUTDATED_LOCKFILE`. A project that pins pnpm 14 or later
therefore needs the pnpm making the switch to be at least the release that
reads this format. Since the format reaches boot documents only with pnpm
14, those releases precede the first such document by a major.

### Packages

`packages` maps `name@version` to one entry per package. The entry carries
what today's `packages` and `snapshots` entries carry, together:

```json
"packages": {
  "buffer-impl@3.7.1": {
    "integrity": "sha512-...",
  },

  "path-impl@3.1.2": {
    "integrity": "sha512-...",
  },

  "fs-impl@4.8.4": {
    "integrity": "sha512-...",
    "peerDependencies": {
      "buffer-impl": {
        "specifier": "*",
        "optional": true,
      },
    },
    "dependencies": {
      "path-impl": "3.1.2",
    },
    "snapshots": [
      {
        "peerDependencies": {
          "buffer-impl": {
            "version": "3.7.1",
          },
        },
      },
      {
        "peerDependencies": {
          "buffer-impl": {
            "version": "3.8.0",
          },
        },
      },
    ],
  },

  "container@3.2.2": {
    "integrity": "sha512-...",
    "peerDependencies": {
      "buffer-impl": {
        "version": "3.7.1",
        "below": true,
      },
    },
    "dependencies": {
      "fs-impl": "4.8.4",
    },
  },
},
```

Besides today's metadata fields, an entry has these:

| Key | Content |
| --- | --- |
| `integrity` | The integrity of the package's tarball, for every resolution kind that has one. `resolution` holds the rest of the resolution. A registry package without a revision has none, nor does a directory package, whose directory is the `<path>` of its `name@file:<path>` key. |
| `peerDependencies` | The peers the package declares, as `name: range`. A peer that `peerDependenciesMeta` in `package.json` describes is `name: {specifier: range, optional: true}` instead. There is no `peerDependenciesMeta` field. |
| `dependencies` | The manifest's `dependencies`, and nothing else. |
| `optionalDependencies` | The manifest's `optionalDependencies`, and nothing else. |
| `peerDependencies` | Every peer the package declares, and every peer bound below it. A declared peer is its range, `name: ^1.0.0`, or a map with `specifier`, `version` when it is bound, and `optional: true` from `peerDependenciesMeta`. A bound declared peer is linked next to the package. A peer bound below the package from the scope above it, which the package does not declare, is `{version, below: true}`, and is not linked next to it. |
| `snapshots` | For a package installed with several sets of peers, a list with one snapshot per set, holding the same keys as the entry. |

`peerDependencies` keeps today's meaning, the ranges the package declares,
and takes `peerDependenciesMeta` in: a peer's entry is its range, or a map of
`specifier`, `version`, and `optional` when there is something to say besides
the range. The map is written one field per line, so binding or updating a
peer changes only the `"version"` line while `"specifier"` and `"optional"`
stay untouched, and two branches that bump adjacent peers merge without a
conflict.

Rules:

- The key is `name@version`, with the same `<version>` forms as today's keys
  and no groups. A patched package has the key an unpatched one would;
  `patchedDependencies` says it is patched, and with what hash (see
  [Patches](#patches)).
- A reference is a version, `<name>@<version>` for an alias, or `link:<path>`,
  as today, without groups. Where [following it](#following-a-reference)
  needs to name the snapshot, the reference is a map, `{version, with}`,
  whose `with` holds the bindings that tell the package's snapshots apart.
  It is written one binding per line:

  ```json
      "react-dom": {
        "version": "18.3.1",
        "with": {
          "react": "18.3.1",
        },
      },
  ```

  This is the snapshot of `react-dom@18.3.1` resolved with `react@18.3.1`,
  as today's `react-dom@18.3.1(react@18.3.1)` is. The layout is for git:
  the version and the bindings change for different reasons, and on
  separate lines two branches that change one each merge without a
  conflict. The `"with": {` line between them is what git needs to keep
  the two changes apart.
- The snapshots of a package resolve the same dependency versions. Only the
  peer bindings differ. So an entry of any map that every snapshot has with the
  same value is written once, in the entry, and each snapshot lists the rest.
  A snapshot is the entry's maps with the snapshot's entries added. A package
  with one snapshot has no `snapshots` list.
- A name is bound in at most one of `dependencies`, `optionalDependencies`,
  and `peerDependencies` of a snapshot. A name declared as both a dependency
  and a peer has a `version` under `peerDependencies` if a parent provided
  it. It is in its dependency map if the package's own dependency was
  installed. Such a name differs between the snapshots, so it is never
  hoisted.
- Whether a peer edge is optional is read from `optional` in
  `peerDependencies`. The installer treats a failure on an optional peer edge
  as it does today.
- An unbound own peer is its range alone. `peerDependencies` lists every
  peer the package declares.
- Hoisting works per field. The range and `optional` of a declared peer are
  the package's, so they are in the entry; a `version` every snapshot shares
  is in the entry, and one that differs is in each snapshot, as
  `{version: …}` beside the entry's range.
- `snapshots` is sorted by today's keys of the snapshots, so the file is
  reproducible. An entry's keys are written metadata first, `resolution`,
  `integrity`, then today's `packages` order, then `dependencies`,
  `optionalDependencies`, `snapshots`. Nested maps are alphabetical.
- A peer that nothing provides is not recorded. Today's
  `transitivePeerDependencies` lists, beside the bound peers, the peers the
  packages below declare and nothing binds; a reader that needs that list
  derives those names from the `peerDependencies` of the packages below,
  which is where the resolver takes them from. A package's own unbound peers
  are listed by its dependents, not by the package.
- `deprecated` is not recorded. A deprecation is a fact about the registry's
  current metadata, not about the resolution, and pnpm reads it from the
  registry when it resolves. The lockfile's copy went stale the moment a
  package was deprecated or undeprecated after the install that wrote it.
- `hasBin` stays, with the other fields the installer reads before it has
  the package: `engines`, `cpu`, `os`, `libc`, `bundledDependencies`. The
  bin linker takes the set of packages with bins from the lockfile and reads
  a manifest only for those. Without the flag it would read one manifest per
  dependency it links bins under, which is more file reads on every install.
- Every entry is a package the graph reaches. Today a `packages` entry
  without a snapshot can only come from a hand-written or broken lockfile.
  This format has no way to write one: an entry with metadata only would read
  back as a package with an empty snapshot.

### Projects

The section is `projects`, the word pnpm uses for these entries everywhere
else; today's format calls it `importers`. A project's dependency is its
specifier alone when the version follows from the specifier, and a
`{specifier, version}` map otherwise:

```json
"projects": {
  "packages/app": {
    "dependencies": {
      "@babel/core": "catalog:",
      "debug": {
        "specifier": "^4.0.0",
        "version": "4.4.0",
      },
      "injected-utils": "workspace:*",
      "left-pad": "1.3.0",
      "shared-utils": "workspace:*",
    },
  },

  "packages/shared-utils": {
    "name": "shared-utils",
  },
},
```

The specifier is written alone only when the version pnpm resolved is the
one the specifier implies; a dependency whose version differs writes both.
The version follows from:

- an exact version, which is the version. An exact version that
  `linkWorkspacePackages` resolved to a workspace link differs, and writes
  both;
- `npm:name@1.2.3`, which is `name@1.2.3`;
- a `link:` or `file:` path, which is the specifier itself. A project's
  `file:` version is relative to the lockfile's directory where the
  specifier is relative to the project, so the two read the same only for
  the root project, and a `file:` dependency of another project writes
  both;
- `catalog:` or `catalog:<name>`, which is the version `settings.catalogs`
  section records for the entry, as `name@version` when the catalog entry is
  an `npm:` alias;
- `workspace:` for an injected workspace package, which is `file:<path>`,
  the `<path>` of the package's one `name@file:<path>` entry in `packages`,
  as `name@file:<path>` when the specifier is `workspace:name@range`. The
  name is the dependency's, or the one the specifier gives;
- `workspace:` for a linked workspace project, which is `link:<path>`, the
  path from the project to the project that records the dependency's name
  as its `name`. A project that a `workspace:` dependency links to records
  its `name`: projects are keyed by path, and the name is the one fact a
  link needs that the lockfile would not otherwise hold.

A range writes both, because the version is the lock. A `workspace:`
dependency writes both when the lockfile cannot tell where it points: a
link to a path that is no project, a name two projects claim, or a
package that is both injected and linked. A dependency whose
version does not select its snapshot adds `with`, one binding per line
like a [reference](#packages) that names its snapshot. It leaves the
version out when the version follows from the specifier, as a dependency
without `with` does:

```json
      "react-dom": {
        "specifier": "^18.0.0",
        "version": "18.3.1",
        "with": {
          "react": "18.3.1",
        },
      },
      "ui-kit": {
        "specifier": "2.4.0",
        "with": {
          "react": "18.3.1",
        },
      },
```

A `catalogs` entry under `settings` follows the same rule: `"react":
"19.2.7"` for an exact specifier, and a `{"specifier": "^7.29.7", "version":
"7.29.7"}` map for a range. A catalog entry records the bare version
of an `npm:` alias, so `npm:name@1.2.3` implies `1.2.3` there.

A reader that rejects a lockfile whose specifier implies a version it cannot
find, such as a `catalog:` entry `settings.catalogs` lacks or a catalog
entry `npm:name` without a version, fails with
`ERR_PNPM_NEXT_LOCKFILE_UNDERIVABLE_VERSION`.

### Patches

No key or reference carries a `(patch_hash=…)` group. `patchedDependencies`,
under `settings`, records every patch once, keyed as today by `name@version`, a range, or a
name, with the patch file's hash. A reader matches a package to that section
the way the resolver does: the exact version first, then the one range it
satisfies, then the bare name.

Today the hash in the key is what makes a changed patch file re-link the
package. With the hash in one place, the installer treats a changed hash in
`patchedDependencies` as a change to every package it matches. The
`node_modules/.pnpm` directory name keeps the hash, as
[the virtual store](#the-virtual-store) section says.

A `patched` flag on the entry would say nothing `patchedDependencies` does
not, so there is none. Across the lockfiles the prototype was tried on, no
version was present both patched and unpatched, which is what lets the key
drop the hash.

### Following a reference

The *scope* of a snapshot or project is the union of its `dependencies`,
`optionalDependencies`, and the bound entries of its `peerDependencies`,
`below` or not. A snapshot's scope also
holds the snapshot itself under its own name, because a package can be the
peer of its own dependency, as `@babel/core` is of
`@babel/helper-module-transforms`. Each
name maps to one reference.

To follow the reference `fs-impl: 4.8.4` from a referrer:

1. If `fs-impl@4.8.4` has one snapshot, that is the one.
2. If the reference has `with`, pick the snapshot that binds each peer it
   names to the reference given.
3. Otherwise pick the snapshot none of whose peer bindings disagrees with
   the referrer's scope, agreeing on the most names, then binding the
   fewest names the scope lacks, and among equals the one listed first
   under `snapshots`. A name the scope lacks is neutral, because an
   optional peer bound from elsewhere in the graph is in no referrer's
   scope. The list order is part of the file, so every reader breaks a
   tie the same way, and the writer's read-back below adds `with` wherever
   that choice is not the snapshot it meant.

The scope selects the snapshot whenever every binding below the referrer is in
it. A peer bound to the referrer's own dependency or own peer is. A peer bound
from the scope above the referrer is among its `below` bindings.

Two resolver features break that, and the prototype found both in real
lockfiles:

- `dedupePeerDependents` can link a snapshot whose bindings the referrer's
  scope doesn't hold. In the lockfile of a large front-end monorepo, a package
  with no peers and no transitive peers links one of three snapshots of its
  child.
- An optional peer bound from anywhere in the graph (pnpm/pnpm#7830) is not in
  the referrer's scope. If the provider has several snapshots, nothing in the
  scope says which one.

So the writer reads its output back, and wherever step 3 picks another
snapshot, it writes that reference with the `with` map that names its snapshot,
the bindings the package's snapshots differ in, and reads back again. Two
rounds settled every lockfile the prototype was tried on. In the monorepo
above, 270 references name their snapshot, 174 of them in `projects`, where
the project's scope has no say between the snapshots. Today's format has
18,961 lines with peer groups there. In pnpm's own lockfile, no reference
does.

### Providers of a binding

Without `dedupePeers`, a snapshot's key today nests the key of each provider,
with the provider's own peers: `helper@1.0.0(@babel/core@7.29.7(supports-color@10.2.2))`.
Rebuilding today's keys, which pnpm keeps using internally and for the
`node_modules/.pnpm` directories, needs the provider's snapshot, and a binding
names only its version.

The provider of a binding is the snapshot linked under the peer's name in the
scope above the snapshot. pnpm looks under the referrer that first reaches the
snapshot in a walk from the projects, then under that referrer's referrer,
and so on.
The walk takes projects and children in name order, so every reader reaches
the same snapshot first. Peers in a cycle are written as `name@version`, as
the resolver writes them. Where the walk does not find the provider, the
binding names its provider's snapshot, like a reference in step 2. That is
also how two snapshots of a package whose bindings agree on every version
but whose providers' snapshots differ are told apart: the read-back finds
that the version alone does not select the provider, and the binding
names the provider's snapshot. The one such case the prototype refuses is
a lockfile whose hashed suffixes hide which provider snapshot each of two
snapshots bound, so there is nothing to write; see [Prototype
results](#prototype-results).

### Diffs

A bump of `buffer-impl` to 3.7.2 changes the binding lines and nothing else:

```diff
   "fs-impl@4.8.4": {
     "integrity": "sha512-...",
     "peerDependencies": {
       "buffer-impl": {
         "specifier": "*",
-        "version": "3.7.1",
+        "version": "3.7.2",
       },
     },
     "dependencies": {
       "path-impl": "3.1.2",
     },
   },

   "container@3.2.2": {
     "integrity": "sha512-...",
     "peerDependencies": {
       "buffer-impl": {
-        "version": "3.7.1",
+        "version": "3.7.2",
         "below": true,
       },
     },
     "dependencies": {
       "fs-impl": "4.8.4",
     },
   },
```

A package that didn't change and binds a new optional peer, the
pnpm/pnpm#17773 case, gains one line per snapshot that binds it:

```diff
   "fs-impl@4.8.4": {
     "integrity": "sha512-...",
     "peerDependencies": {
       "bare-buffer": {
         "specifier": "*",
+        "version": "2.6.1",
         "optional": true,
       },
     },
     "dependencies": {
       "path-impl": "3.1.2",
     },
   },
```

In the branch example from the motivation, branch B writes `fs-impl: 4.8.4`.
After the merge, the binding is 3.7.2, and the reference is still right.

### The virtual store

The lockfile no longer names a snapshot's directory under `node_modules/.pnpm`.
The installer derives it: `name@version`, then the names of the bound peers in
parentheses. A version is added to a name only when two snapshots of the
package differ on that name. `fs-impl@4.8.4(buffer-impl)` for a package with
one snapshot, `fs-impl@4.8.4(buffer-impl@3.7.1)` and
`fs-impl@4.8.4(buffer-impl@3.8.0)` when there are two. Two snapshots that
bind the same version of a name to different snapshots of the provider nest
the provider's discriminator, as today's keys do:
`fs-impl@4.8.4(host@1.0.0(react@17.0.0))` and
`fs-impl@4.8.4(host@1.0.0(react@18.0.0))`, so their directories and the
peer links inside them stay apart. The patch hash and
`virtualStoreDirMaxLength` apply to the directory name as today.

Under a provider bump, a snapshot keeps its directory, and the installer
repoints one symlink inside it. Today it removes the directory and creates
the renamed one, with every symlink and hard link in it.

### Settings

`settings` records what `pnpm-workspace.yaml` said when the lockfile was
written. Beside today's settings it holds the sections that are top-level
keys today and top-level keys of `pnpm-workspace.yaml`: `catalogs`,
`overrides`, `patchedDependencies`, `ignoredOptionalDependencies`, and the
checksums `packageExtensionsChecksum` and `pnpmfileChecksum` that stand in
for `packageExtensions` and the pnpmfile. The top level of the lockfile is
then `lockfileVersion`, `settings`, `projects`, `packages`, and `time`: the
version, the inputs, the two outputs, and the dates.

`lockfileVersion` is `'13.0'`. pnpm/rfcs#68 records the decision to avoid
lockfile version bumps, and this change cannot avoid one: it removes
`snapshots` and reshapes `packages`, so an older reader cannot read it. The
number follows pnpm's convention that a new lockfile version takes the
number of the pnpm major that first writes it by default, as `'9.0'` did
with pnpm 9, so a reader that meets the file knows which pnpm it needs.
pnpm v12 writes it behind a setting before then; a lockfile written that
way may need regenerating when pnpm 13 ships, since the format may still
change.

```json
"settings": {
  "autoInstallPeers": true,
  "excludeLinksFromLockfile": false,
  "catalogs": {
    "default": {
      "@babel/core": {
        "specifier": "^7.29.7",
        "version": "7.29.7",
      },
      "react": "19.2.7",
    },
  },
  "overrides": {
    "react@19": "19.2.7",
  },
  "patchedDependencies": {
    "foo@2.1.0": "abc123",
  },
},
```

Today's settings come first, alphabetical, then the recorded sections in the
order above. `catalogs` is the one entry resolution touches: its entries are
configuration, and each carries the version it resolved to, the way an
project's dependency does, specifier alone when the version follows from
it. The up-to-date check compares the one subtree against the
current configuration.

`peersSuffixMaxLength` is removed. No suffix is written, so no suffix is
hashed.

### Directory package specifiers

A `packages` entry of a directory package, keyed `name@file:<path>`, has a
`specifiers` field:

```json
"packages": {
  "peer@file:../peer": {
    "peerDependencies": {
      "host": "1.0.0",
    },
    "specifiers": {
      "dependencies": {
        "host": "workspace:../host",
      },
    },
  },
},
```

Rules:

- `specifiers` holds the children the resolver walked, as the manifest
  declares them after `readPackage` hooks and `packageExtensions` are applied.
  It is grouped under `dependencies` and `optionalDependencies`, the groups
  the snapshot records the children under.
- Specifiers are recorded before catalog resolution and before peer
  shadowing. A name declared as both a dependency and a peer is in
  `specifiers`. Whether it was bound as a peer is in the entry's
  `peerDependencies` (see [Packages](#packages)).
- The field is the same for every snapshot of the package, so it is
  never in `snapshots`.
- The field is required on every directory entry. The frozen-lockfile check
  compares it with the package's current manifest, and doesn't derive the
  children from the manifest.

pnpm/pnpm#17477 already writes the field in lockfile 9.0. The next format
makes it required and removes the fallback for entries without it.

### The current lockfile

`node_modules/.pnpm/lock.yaml` keeps today's peer-suffixed shape, because
pnpm reads it on every install and that shape needs no conversion. It records
the same `lockfileVersion` as `pnpm-lock.yaml`, because the check that skips
an up-to-date install compares the two lockfiles whole. pnpm reads it as
suffixed whatever its version says.

When `pnpm-lock.yaml` is missing, pnpm regenerates it from the current
lockfile, and the check that skips the install uses the current lockfile's
digest in its place. That works today because the two files are byte for byte
the same. In this format the digest is taken from the text the regeneration
writes.

### Prototype results

pnpm v12 reads and writes the format behind a `lockfile.format: v13`
setting, merged in pnpm/pnpm#17886. The implementation is a prototype in
one sense: it converts at the boundary, and the in-memory lockfile keeps
today's keys, so no install logic changes.

**Correctness.** 142 of 146 real lockfiles round-trip exactly through the
written text, from small fixtures up to a 4.4 MB monorepo lockfile with 5,623
snapshots. An install from the next format produces the same
`node_modules/.pnpm` directories and the same current lockfile as an install
from today's format. The four that can't be written:

- Two fixtures written by an older pnpm, which recorded an optional peer under
  `dependencies`.
- One lockfile with two hashed-suffix snapshots whose bindings are equal and
  whose nested providers differ. Without the suffix, nothing tells them apart.
- One fixture whose keys keep the `/` prefix of lockfile 6, so its references
  point at snapshots it does not have. Today's suffix was the only thing that
  let them read back.
- One old audit fixture whose `transitivePeerDependencies` lists do not
  follow the current resolver's rule, under which the reader derives them.

The writer refuses these with an error rather than writing a lockfile that
reads back differently.

**Syntax.** The generated lockfiles, JSON with trailing commas, were read
as the same documents by libyaml, PyYAML, js-yaml and pnpm's own Rust YAML
parser.

**Diffs.** Updating `react` from 18.3.1 to 18.2.0 in a project with nine
React libraries changes 142 lines today and 52 in this format. That lockfile
is 31 KB instead of 35 KB.

The 5,623-snapshot lockfile has 99,955 lines today and 69,624 in this
format, 2.43 MB instead of 4.40 MB, and 466 KB instead of 576 KB gzipped.
Its `projects` section, a third of the file before, writes 5,823 of its
8,204 dependencies as the specifier alone.

**Speed of the lockfile alone**, minimum of 15 runs:

| Lockfile | Bytes, today | Bytes, next | Read, today | Read, next | Write, today | Write, next |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 255 snapshots | 85 KB | 80 KB | 1.9 ms | 1.4 ms | 0.7 ms | 1.9 ms |
| 1,664 snapshots | 860 KB | 702 KB | 4.1 ms | 6.7 ms | 4.6 ms | 11 ms |
| 5,623 snapshots | 4.40 MB | 2.43 MB | 13 ms | 30 ms | 17 ms | 61 ms |

Both formats parse their large sections on separate threads, and cut a
section into one chunk per thread at entry boundaries. That took today's
format from 49 ms to 15 ms on the 5,623-snapshot lockfile. Serially, the
next format's JSON parses into its entries in 15 ms, where the same
document as YAML takes 69 ms and today's format 68 ms.

Where the time goes, on the 5,623-snapshot lockfile, minimum of 15 runs. The
first column is what pnpm would pay with the format's entries in memory, which
is how pnpm v13 is expected to implement it. The second is what the prototype
pays to keep today's peer-suffixed keys in memory:

| Stage | Inherent to the format | Prototype only |
| --- | ---: | ---: |
| Read: parse the document into entries | 6.1 ms | |
| Read: match each reference to a snapshot and each binding to its provider | 3.0 ms | |
| Read: split each entry into metadata and snapshots | | 4.7 ms |
| Read: rebuild the peer suffixes | | 0.4 ms |
| Read: build today's sections from them | | 6.7 ms |
| Write: serialize the document | 9.5 ms | |
| Write: group today's snapshots by package and find their providers | | 5.2 ms |
| Write: convert, read back, and compare, over the rounds | | 35 ms |
| Write: hoist what the snapshots share | | 6.6 ms |

Today's format reads this lockfile in 13 ms and writes it in 17 ms. The
inherent stages of the next format add up to 9.1 ms and 9.5 ms. The
matching stage has to stay, because the lockfile no longer names which
snapshot a reference links. The peer suffixes go once the virtual store
derives its directory names from the bindings, as [proposed
above](#the-virtual-store). The native numbers are a sum of measured stages,
not a measurement of an implementation.

**Installs**, the same binary on a workspace of 237 projects and 1,664
snapshots, median of 20 runs, measured with the YAML syntax the prototype
wrote before the JSON one:

| Scenario | Today | Next | 95% interval of the difference |
| --- | ---: | ---: | --- |
| Frozen restore, warm store | 564 ms | 557 ms | −41 to +15 ms |
| Restore without `--frozen-lockfile` | 573 ms | 565 ms | −36 to +20 ms |
| Repeat install | 153 ms | 158 ms | +2 to +10 ms |
| Resolve after a manifest change | 587 ms | 605 ms | +1 to +25 ms |
| `pnpm add` | 563 ms | 582 ms | +8 to +26 ms |
| Regenerate a deleted `pnpm-lock.yaml` | 82 ms | 91 ms | +5 to +11 ms |

Today's format parses in parallel chunks in these runs too. The restores,
which read the lockfile and link, are even. What the next format pays in the
other scenarios is the prototype's conversion: about 3 ms on each read and 7 ms
on each write of this lockfile. The regeneration scenario writes without
resolving, so it shows the write cost in full.

## Rationale and Alternatives

1. **Keep the format and document it.** The specification now describes the
   current behavior. This costs nothing, and every problem above stays.
2. **Add a list of the dependencies that a peer replaced.** pnpm/tasks#221
   proposes this, for example `peerShadowedDependencies: [widget]`. The field
   is additive and needs no version change. It fixes the dependency-and-peer
   ambiguity only.
3. **Keep `specifiers` optional in lockfile 9.0** (pnpm/pnpm#17477). This
   needs no format change. The derived walk stays as a fallback until every
   lockfile has been rewritten, and nothing tells pnpm when that is.
4. **Keep the suffix keys, add the binding maps.** The first draft of this
   RFC. It fixes the optional-dependency confusion, the ambiguity, and the
   hashed suffix, because the bindings are stored. Keys and references still
   embed versions, so the diff churn and the stale-reference merges stay.
5. **Short ordinal ids.** `fs-impl@4.8.4#1`, numbered by a deterministic walk.
   Stable under a bump, but a new snapshot can renumber the ones after it, and
   a key means nothing outside the one file it is in.
6. **Group snapshots by suffix.** The binding set is written once as a group
   key, with the packages that share it below. A parent's binding set is
   usually a superset of its children's, so parent and child land in
   different groups and references keep the suffix. A package that gains a
   binding moves between groups, which is a worse diff than today's rename.
7. **Key by peer names, not versions.** `fs-impl@4.8.4(buffer-impl)`, with
   versions as properties and a version in the key only where snapshots
   differ on a name. Reproducible and stable under a bump. It keeps a key
   grammar and a collision rule, and a package that gains a binding is still
   renamed with its references.
8. **Alternative 7 with the key reduced to `name@version`**, which is
   possible because the referrer's scope identifies the snapshot in nearly
   every case. No collision rule and no hash, and the diff is the set of
   bindings that changed. An edge is a match instead of a lookup, and every
   consumer has to implement the match. The writer has to read its output
   back to find the references that need to name their snapshot.
9. **This proposal.** Alternative 8 with `packages` and `snapshots` merged,
   since they share their keys, and with what a package's snapshots share
   written once. The second prototype measured the first: the shared
   dependency maps were 19% of the `snapshots` section's lines, and the
   duplicated keys another 8%. The cost is that the resolved peer maps need
   names other than `peerDependencies`.

**Syntax.** Three were weighed. Today's YAML block style needs quoting
rules, `'@scope/pkg'`, `'catalog:'` and `'*'` among them, exposes YAML 1.1
readers to implicit typing, and parses in four times the time of JSON.
Plain JSON parses as fast but makes adding or removing the last entry of a
map a two-line change, and a separate `pnpm-lock.json` would break every
tool that detects pnpm by the file's name. JSON with trailing commas keeps
the name and the one-line diffs, at 16% more bytes raw and 2% more gzipped
on the 5,623-snapshot lockfile, and a tool that drops its YAML parser for a
JSON one needs one that allows the comma, as Bun's lockfile needs.

## Implementation

pnpm v12, except for one reader in pnpm v11. pnpm v11 keeps lockfile 9.0,
including the optional `specifiers` field that pnpm/pnpm#17477 adds to
both CLIs, and reads a boot document in this format, so that it can
switch to the pnpm version a project pins. pnpm/pnpm#17886 carries that
reader (`@pnpm/lockfile.fs`), and the v12 writer of the boot document in
this format for projects that pin pnpm 14 or later.

The prototype converts at the boundary and keeps today's peer-suffixed keys in
memory. A native implementation, which keeps the format's entries in memory
and pays none of the conversion, is planned for pnpm v13. Until it exists, its
speed is the estimate in [Prototype results](#prototype-results), not a
measurement.

- `pnpm/crates/lockfile`: the file is written as JSON with trailing commas
  and read with a JSON parser that allows them, with `projects` and
  `packages` split at their entries across threads; a 9.0 lockfile is read
  as YAML. One `Package` type holds the metadata, the shared
  graph fields, and the snapshots. A function follows a reference from a
  scope, with the rule above, and is the one place that rule lives. Reading
  a 9.0 lockfile converts it. The suffix becomes the bindings, and children
  that match the declared `peerDependencies` become their `version`. A
  hashed suffix is
  recovered once by today's walk.
- `pnpm/crates/package-manager/src/dependencies_graph_to_lockfile`: write the
  bindings to the maps and sort the snapshots. The resolver already knows which
  children are peers.
- `pnpm/crates/deps-restorer`: follow references with the shared function,
  derive directory names from the snapshot, and take optionality from
  `peerDependencies`.
- `pnpm/crates/resolving-deps-resolver`: lockfile reuse follows references
  with the shared function. `locked_peers/hashed_suffix.rs` is removed.
- `pnpm/crates/cli/src/cli_args/deploy`: read peer edges from
  `peerDependencies` instead of deciding them from the peer suffix.
- `pnpm/crates/lockfile/src/merge_lockfile_changes.rs`: merge snapshots by
  binding map. A changed binding is a changed field on the same entry, not a
  removal and an addition.
- `pnpm/crates/package-manager/src/install/lockfile_freshness/directory_deps`:
  compare `specifiers` with the manifest and remove the derived walk.
- Published readers: the `@pnpm/lockfile.*` packages and the specification
  carry the reference-following rule, so Bit, Renovate, Dependabot, and
  scanners implement it once.
- Docs: `pnpm/docs/lockfile-format.md`, and the settings page for
  `peersSuffixMaxLength`.

## Unresolved Questions and Bikeshedding

- **References that name their snapshot.** They exist because
  `dedupePeerDependents` and optional peers bound from anywhere choose snapshots
  the referrer's scope does not predict. Changing either rule so the scope
  always predicts the snapshot would make every reference a bare version, and
  would remove the writer's read-back. That is a resolution change, with its
  own trade-offs.
- **Lockfiles that cannot be written.** A writer that fails is safe but
  unfriendly. pnpm could fall back to today's format for them, which keeps the
  install working and changes the format without being asked.
- **Older lockfiles without `specifiers`.** Without the derived walk, a frozen
  install can't check a directory entry that has no `specifiers`. It could
  fail and ask for a non-frozen install, or keep the walk only for lockfiles
  of the older version.
- **The shape of `specifiers`.** A project writes each `specifier` beside
  its `version`. A directory entry writes its specifiers apart from its
  children, because the children are in `snapshots`. The next format could
  write both the same way, for example by giving projects a `specifiers` map
  too.
- **Other inconsistencies found while writing the specification**, which a
  format change could fix at the same time:
  - A project's `link:` path is relative to the project directory. A
    snapshot's `link:` path and every `file:` path are relative to the
    lockfile's directory.
  - Whether a reference is an alias is decided by a character rule: it starts
    with `@`, or it has an `@` before any `:`. An explicit form would remove
    the guess.
  - A `packages` entry carries `version` only when its key's version is a URL,
    a `file:` path, or a `runtime:` version, except for directories.
