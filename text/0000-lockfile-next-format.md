# One lockfile entry per package, with peer bindings as properties

## Summary

In the next lockfile format, `packages` has one entry per package, keyed by
`name@version`. The entry holds the package's metadata, its resolved
dependencies, and its peer bindings as properties: a `peers` map for the
package's own peers, and a `transitivePeers` map for the peers bound below
it. There is no `snapshots` section. A package that pnpm installs in several
peer variants writes what the variants share once and lists what differs
under `variants`. A reference names a version, and the referrer's own
bindings determine which variant it links. Where they don't, the reference
keeps its peer groups.

A prototype behind an opt-in setting reads and writes this format exactly for
170 of the 173 real lockfiles it was tried on, and writes a 5,623-snapshot
lockfile in 61% of today's bytes. See [Prototype results](#prototype-results).

The format also requires every directory package's `packages` entry to record
`specifiers`, the dependency specifiers the resolver read from its manifest.

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

An importer records each dependency's `specifier` beside its `version`, so the
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

### Packages

`packages` maps `name@version` to one entry per package. The entry carries
what today's `packages` and `snapshots` entries carry, together:

```yaml
packages:
  buffer-impl@3.7.1:
    resolution: {integrity: sha512-...}
  path-impl@3.1.2:
    resolution: {integrity: sha512-...}
  fs-impl@4.8.4:
    resolution: {integrity: sha512-...}
    peerDependencies:
      buffer-impl: '*'
    peerDependenciesMeta:
      buffer-impl:
        optional: true
    dependencies:
      path-impl: 3.1.2
    variants:
      - peers:
          buffer-impl: 3.7.1
      - peers:
          buffer-impl: 3.8.0
  container@3.2.2:
    resolution: {integrity: sha512-...}
    dependencies:
      fs-impl: 4.8.4
    transitivePeers:
      buffer-impl: 3.7.1
```

Besides today's metadata fields, an entry has these:

| Key | Content |
| --- | --- |
| `dependencies` | The manifest's `dependencies`, and nothing else. |
| `optionalDependencies` | The manifest's `optionalDependencies`, and nothing else. |
| `peers` | The package's own peers that are bound, as `name: reference`. These are linked next to the package. |
| `transitivePeers` | The peers bound below this package from the scope above it, as `name: reference`. These are not linked next to the package. |
| `missingPeers` | The other peers needed below this package, as a list of names: those nothing provides, and those bound from anywhere in the graph (pnpm/pnpm#7830). Neither kind is part of the package's identity. |
| `variants` | For a package installed in several peer variants, a list with one item per variant, holding the same keys as the entry. |

`peerDependencies` keeps today's meaning, the ranges the package declares,
because it mirrors `package.json`. The resolved maps have their own names.

Rules:

- The key is `name@version`, with the same `<version>` forms as today's keys
  and no peer groups. A patched package keeps its `(patch_hash=…)` group,
  which is the same for every variant.
- A reference is a version, `<name>@<version>` for an alias, or `link:<path>`,
  as today, without peer groups, except where
  [following it](#following-a-reference) needs them.
- Peer variants of a package resolve the same dependency versions. Only the
  peer bindings differ. So an entry of any map that every variant has with the
  same value is written once, in the entry, and each variant lists the rest.
  A variant is the entry's maps with the variant's entries added. A package
  with one variant has no `variants` list.
- A name is in at most one of `dependencies`, `optionalDependencies`, and
  `peers` of a variant. A name declared as both a dependency and a peer is in
  `peers` if a parent provided it. It is in its dependency map if the
  package's own dependency was installed. Such a name differs between the
  variants, so it is never hoisted.
- Whether a peer edge is optional is read from `peerDependenciesMeta`. The
  installer treats a failure on an optional peer edge as it does today.
- An unbound own peer is not recorded. `peerDependencies` lists every peer the
  package declares.
- `variants` is sorted by today's keys of the variants, so the file is
  reproducible. An entry's keys are written in today's `packages` order, then
  `dependencies`, `optionalDependencies`, `peers`, `transitivePeers`,
  `missingPeers`, `variants`.
- A package no snapshot uses is not written. Nothing can reach it, and an
  entry with metadata only would read back as a package with an empty
  snapshot.

### Following a reference

The *scope* of a variant or importer is the union of its `dependencies`,
`optionalDependencies`, `peers`, and `transitivePeers`. A variant's scope also
holds the variant itself under its own name, because a package can be the
peer of its own dependency, as `@babel/core` is of
`@babel/helper-module-transforms`. Each
name maps to one reference.

To follow the reference `fs-impl: 4.8.4` from a referrer:

1. If `fs-impl@4.8.4` has one variant, that is the one.
2. If the reference carries peer groups, pick the variant whose providers are
   the packages those groups name.
3. Otherwise pick the variant whose `peers` and `transitivePeers` all agree
   with the referrer's scope, binding the most names.

The scope selects the variant whenever every binding below the referrer is in
it. A peer bound to the referrer's own dependency or own peer is. A peer bound
from the scope above the referrer is in its `transitivePeers`.

Two resolver features break that, and the prototype found both in real
lockfiles:

- `dedupePeerDependents` can link a variant whose bindings the referrer's
  scope doesn't hold. In the lockfile of a large front-end monorepo, a package
  with no peers and no transitive peers links one of three variants of its
  child.
- An optional peer bound from anywhere in the graph (pnpm/pnpm#7830) is not in
  the referrer's scope. If the provider has several variants, nothing in the
  scope says which one.

So the writer reads its output back, and wherever step 3 picks another
variant, it writes that reference with its peer groups and reads back again.
Two rounds settled every lockfile the prototype was tried on. In the monorepo
above, 138 lines keep peer groups, where today's format has 14,820.

### Providers of a binding

Without `dedupePeers`, a snapshot's key today nests the key of each provider,
with the provider's own peers: `helper@1.0.0(@babel/core@7.29.7(supports-color@10.2.2))`.
Rebuilding today's keys, which pnpm keeps using internally and for the
`node_modules/.pnpm` directories, needs the provider's variant, and a binding
names only its version.

The provider of a binding is the variant linked under the peer's name in the
scope above the snapshot. pnpm looks under the referrer that first reaches the
snapshot in a walk from the importers, then under that referrer's referrer,
and so on.
The walk takes importers and children in name order, so every reader reaches
the same snapshot first. Peers in a cycle are written as `name@version`, as
the resolver writes them. Where the walk does not find the provider, the
binding keeps its peer groups, like a reference in step 2.

### Diffs

A bump of `buffer-impl` to 3.7.2 changes the binding lines and nothing else:

```diff
   fs-impl@4.8.4:
     resolution: {integrity: sha512-...}
     peerDependencies:
       buffer-impl: '*'
     dependencies:
       path-impl: 3.1.2
     peers:
-      buffer-impl: 3.7.1
+      buffer-impl: 3.7.2
   container@3.2.2:
     resolution: {integrity: sha512-...}
     dependencies:
       fs-impl: 4.8.4
     transitivePeers:
-      buffer-impl: 3.7.1
+      buffer-impl: 3.7.2
```

A package that didn't change and binds a new optional peer, the
pnpm/pnpm#17773 case, gains one line per variant that binds it:

```diff
   fs-impl@4.8.4:
     resolution: {integrity: sha512-...}
     dependencies:
       path-impl: 3.1.2
+    peers:
+      bare-buffer: 2.6.1
```

In the branch example from the motivation, branch B writes `fs-impl: 4.8.4`.
After the merge, the binding is 3.7.2, and the reference is still right.

### The virtual store

The lockfile no longer names a snapshot's directory under `node_modules/.pnpm`.
The installer derives it: `name@version`, then the names of the bound peers in
parentheses. A version is added to a name only when two variants of the
package differ on that name. `fs-impl@4.8.4(buffer-impl)` for a package with
one variant, `fs-impl@4.8.4(buffer-impl@3.7.1)` and
`fs-impl@4.8.4(buffer-impl@3.8.0)` when there are two. The patch hash and
`virtualStoreDirMaxLength` apply to the directory name as today.

Under a provider bump, a variant keeps its directory, and the installer
repoints one symlink inside it. Today it removes the directory and creates
the renamed one, with every symlink and hard link in it.

### Settings

`peersSuffixMaxLength` is removed. No suffix is written, so no suffix is
hashed.

### Directory package specifiers

A `packages` entry whose resolution is `type: directory` has a `specifiers`
field:

```yaml
packages:
  peer@file:../peer:
    resolution: {directory: ../peer, type: directory}
    peerDependencies:
      host: 1.0.0
    specifiers:
      dependencies:
        host: workspace:../host
```

Rules:

- `specifiers` holds the children the resolver walked, as the manifest
  declares them after `readPackage` hooks and `packageExtensions` are applied.
  It is grouped under `dependencies` and `optionalDependencies`, the groups
  the snapshot records the children under.
- Specifiers are recorded before catalog resolution and before peer
  shadowing. A name declared as both a dependency and a peer is in
  `specifiers`. Whether it was bound as a peer is in the snapshot's
  `peerDependencies` (see [Snapshots](#snapshots)).
- The field is the same for every peer variant of the package, so it is
  never in `variants`.
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

A prototype in pnpm v12 reads and writes the format behind a
`lockfile.format: next` setting. It converts at the boundary: the in-memory
lockfile keeps today's keys, so no install logic changes.

**Correctness.** 170 of 173 real lockfiles round-trip exactly through the
written text, from small fixtures up to a 4.4 MB monorepo lockfile with 5,623
snapshots. An install from the next format produces the same
`node_modules/.pnpm` directories and the same current lockfile as an install
from today's format. The three that can't be written:

- Two fixtures written by an older pnpm, which recorded an optional peer under
  `dependencies`.
- One lockfile with two hashed-suffix variants whose bindings are equal and
  whose nested providers differ. Without the suffix, nothing tells them apart.

The writer refuses these with an error rather than writing a lockfile that
reads back differently.

**Diffs.** Updating `react` from 18.3.1 to 18.2.0 in a project with nine
React libraries changes 142 lines today and 58 in this format. That lockfile
is 29 KB instead of 35 KB.

**Speed of the lockfile alone**, minimum of 15 runs:

| Lockfile | Bytes, today | Bytes, next | Read, today | Read, next | Write, today | Write, next |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| 255 snapshots | 85 KB | 71 KB | 2.0 ms | 3.0 ms | 0.8 ms | 2.8 ms |
| 1,664 snapshots | 860 KB | 783 KB | 3.3 ms | 7.2 ms | 4.4 ms | 11 ms |
| 5,623 snapshots | 4.40 MB | 2.69 MB | 15 ms | 34 ms | 13 ms | 93 ms |

Both formats parse `importers`, `packages`, and `snapshots` on separate
threads, and cut a large section into one chunk per thread at entry
boundaries. That took today's format from 49 ms to 15 ms on the
5,623-snapshot lockfile. Of the next format's 34 ms, 14 ms is the prototype
converting back to today's keys, which a native implementation would not do.
The remaining difference is YAML parsing of the one struct that carries both
metadata and graph fields. Writing is slower: it converts, hoists what the
variants share, then reads its output back and compares, in one round or
two.

**Installs**, the same binary on a workspace of 237 projects and 1,664
snapshots, median of 20 runs:

| Scenario | Today | Next | 95% interval of the difference |
| --- | ---: | ---: | --- |
| Frozen restore, warm store | 571 ms | 566 ms | −33 to +31 ms |
| Restore without `--frozen-lockfile` | 559 ms | 586 ms | −30 to +33 ms |
| Repeat install | 152 ms | 155 ms | +1 to +9 ms |
| Resolve after a manifest change | 584 ms | 604 ms | +4 to +29 ms |
| `pnpm add` | 564 ms | 572 ms | +4 to +23 ms |
| Regenerate a deleted `pnpm-lock.yaml` | 79 ms | 89 ms | +6 to +12 ms |

Today's format parses in parallel chunks in these runs too. What the next
format pays on top is the prototype's conversion on every read and its
read-back on every write. The regeneration scenario writes without resolving,
so it shows the write cost in full.

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
   Stable under a bump, but a new variant can renumber the ones after it, and
   a key means nothing outside the one file it is in.
6. **Group snapshots by suffix.** The binding set is written once as a group
   key, with the packages that share it below. A parent's binding set is
   usually a superset of its children's, so parent and child land in
   different groups and references keep the suffix. A package that gains a
   binding moves between groups, which is a worse diff than today's rename.
7. **Key by peer names, not versions.** `fs-impl@4.8.4(buffer-impl)`, with
   versions as properties and a version in the key only where variants
   differ on a name. Reproducible and stable under a bump. It keeps a key
   grammar and a collision rule, and a package that gains a binding is still
   renamed with its references.
8. **Alternative 7 with the key reduced to `name@version`**, which is
   possible because the referrer's scope identifies the variant in nearly
   every case. No collision rule and no hash, and the diff is the set of
   bindings that changed. An edge is a match instead of a lookup, and every
   consumer has to implement the match. The writer has to read its output
   back to find the references that need peer groups.
9. **This proposal.** Alternative 8 with `packages` and `snapshots` merged,
   since they share their keys, and with what a package's variants share
   written once. The second prototype measured the first: the shared
   dependency maps were 19% of the `snapshots` section's lines, and the
   duplicated keys another 8%. The cost is that the resolved peer maps need
   names other than `peerDependencies`, and that a `packages` entry no
   snapshot uses cannot be written.

## Implementation

pnpm v12 only. pnpm v11 keeps lockfile 9.0, including the optional
`specifiers` field that pnpm/pnpm#17477 adds to both CLIs.

- `pnpm/crates/lockfile`: one `Package` type holds the metadata, the shared
  graph fields, and the variants. A function follows a reference from a
  scope, with the rule above, and is the one place that rule lives. Reading
  a 9.0 lockfile converts it. The suffix becomes the maps, and children that
  match the declared `peerDependencies` move to `peers`. A hashed suffix is
  recovered once by today's walk.
- `pnpm/crates/package-manager/src/dependencies_graph_to_lockfile`: write the
  bindings to the maps and sort the variants. The resolver already knows which
  children are peers.
- `pnpm/crates/deps-restorer`: follow references with the shared function,
  derive directory names from the snapshot, and take optionality from
  `peerDependenciesMeta`.
- `pnpm/crates/resolving-deps-resolver`: lockfile reuse follows references
  with the shared function. `locked_peers/hashed_suffix.rs` is removed.
- `pnpm/crates/cli/src/cli_args/deploy`: read peer edges from
  `peerDependencies` instead of deciding them from the peer suffix.
- `pnpm/crates/lockfile/src/merge_lockfile_changes.rs`: merge variants by
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

- **The version number.** pnpm/rfcs#68 records the decision to avoid lockfile
  version bumps. This change removes `snapshots` and reshapes `packages`, so
  an older reader cannot read it, and it needs a new `lockfileVersion`. pnpm
  v12 already accepts major `12`.
- **References that keep peer groups.** They exist because
  `dedupePeerDependents` and optional peers bound from anywhere choose variants
  the referrer's scope does not predict. Changing either rule so the scope
  always predicts the variant would make every reference a bare version, and
  would remove the writer's read-back. That is a resolution change, with its
  own trade-offs.
- **`missingPeers`.** The prototype needs the list to rebuild today's
  `transitivePeerDependencies` exactly. If nothing reads the missing
  names, the list can go once pnpm no longer keeps today's keys in memory.
- **Lockfiles that cannot be written.** A writer that fails is safe but
  unfriendly. pnpm could fall back to today's format for them, which keeps the
  install working and changes the format without being asked.
- **Where the remaining parse cost goes.** YAML parsing is paid per node, not
  per byte. Each transitive binding is `name: version`, two nodes where
  today's name list has one, and that is what lets a referrer's scope select
  its children without a walk. Dropping `missingPeers` would remove about
  1,500 more nodes from the 5,623-snapshot lockfile.
- **Unused `packages` entries.** Today's lockfiles can hold a `packages`
  entry no snapshot uses, and this format drops it. If that entry carries
  something a command reads, it needs a place of its own.
- **Older lockfiles without `specifiers`.** Without the derived walk, a frozen
  install can't check a directory entry that has no `specifiers`. It could
  fail and ask for a non-frozen install, or keep the walk only for lockfiles
  of the older version.
- **The shape of `specifiers`.** An importer writes each `specifier` beside
  its `version`. A directory entry writes its specifiers apart from its
  children, because the children are in `snapshots`. The next format could
  write both the same way, for example by giving importers a `specifiers` map
  too.
- **Other inconsistencies found while writing the specification**, which a
  format change could fix at the same time:
  - An importer's `link:` path is relative to the project directory. A
    snapshot's `link:` path and every `file:` path are relative to the
    lockfile's directory.
  - Whether a reference is an alias is decided by a character rule: it starts
    with `@`, or it has an `@` before any `:`. An explicit form would remove
    the guess.
  - A `packages` entry carries `version` only when its key's version is a URL,
    a `file:` path, or a `runtime:` version, except for directories.
