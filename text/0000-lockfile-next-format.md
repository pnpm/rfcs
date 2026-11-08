# Key snapshots by package and record peer bindings as properties

## Summary

In the next lockfile format, a snapshot is keyed by `name@version` only. The
peer suffix leaves the key, and leaves every reference to the key. The peer
bindings become properties of the snapshot: a `peerDependencies` map for the
package's own peers, and a `transitivePeerDependencies` map for the peers
bound below it. A package that pnpm installs in several peer variants has a
list of snapshots under its key. A reference names a version, and the parent's
own bindings determine which variant it links.

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

### Snapshots

`snapshots` maps `name@version` to a list of snapshots. Each snapshot is one
peer variant of the package:

```yaml
packages:
  buffer-impl@3.7.1: ...
  container@3.2.2: ...
  fs-impl@4.8.4:
    resolution: {integrity: sha512-...}
    peerDependencies:
      buffer-impl: '*'
    peerDependenciesMeta:
      buffer-impl:
        optional: true
  path-impl@3.1.2: ...

snapshots:
  buffer-impl@3.7.1:
    - {}
  path-impl@3.1.2:
    - {}
  fs-impl@4.8.4:
    - dependencies:
        path-impl: 3.1.2
      peerDependencies:
        buffer-impl: 3.7.1
    - dependencies:
        path-impl: 3.1.2
      peerDependencies:
        buffer-impl: 3.8.0
  container@3.2.2:
    - dependencies:
        fs-impl: 4.8.4
      transitivePeerDependencies:
        buffer-impl: 3.7.1
```

A snapshot has these maps:

| Key | Content |
| --- | --- |
| `dependencies` | The manifest's `dependencies`, and nothing else. |
| `optionalDependencies` | The manifest's `optionalDependencies`, and nothing else. |
| `peerDependencies` | The package's own peers that are bound, as `name: reference`. These are linked next to the package. |
| `transitivePeerDependencies` | Every peer bound below this package whose binding is not already an entry of the three maps above, as `name: reference`. These are not linked next to the package. |

Rules:

- The key is `name@version` for every snapshot, with the same `<version>`
  forms as today's keys and no suffix. The `packages` key is the same string.
- A reference is a version, `<name>@<version>` for an alias, or `link:<path>`,
  as today, with no suffix.
- A name is in at most one of `dependencies`, `optionalDependencies`, and
  `peerDependencies`. A name declared as both a dependency and a peer is in
  `peerDependencies` if a parent provided it. It is in its dependency map if
  the package's own dependency was installed.
- Whether a peer edge is optional is read from the `packages` entry's
  `peerDependenciesMeta`. The installer treats a failure on an optional peer
  edge as it does today.
- An unbound peer is not recorded. The `packages` entry lists every peer the
  package declares.
- The list is sorted by the snapshots' binding maps, so the file is
  reproducible.
- An importer has the same `transitivePeerDependencies` map, for the peers
  bound below it that its own dependencies don't hold.

### Following a reference

The *context* of a snapshot or importer is the union of its `dependencies`,
`optionalDependencies`, `peerDependencies`, and `transitivePeerDependencies`.
Each name maps to one reference.

To follow the reference `fs-impl: 4.8.4` from a parent, pick the snapshot
under `fs-impl@4.8.4` whose `peerDependencies` and
`transitivePeerDependencies` agree with the parent's context on every name
they share. At most one snapshot matches, because the context holds one
reference per name, and two variants of a package differ in at least one
binding.

This holds because every binding below a parent is in the parent's context by
construction. A peer is bound to the parent's own dependency, to the parent's
own peer, or to something outside the parent's subtree. The first two are in
the parent's dependency and peer maps. The third is in the parent's
`transitivePeerDependencies`. The optional-peer rule that binds from anywhere
in the graph (pnpm/pnpm#7830) and `dedupePeerDependents` only change which
reference is written, not where.

A lockfile in which a parent's context matches no snapshot, or more than one,
is malformed. pnpm reports it as it reports a missing snapshot today, and
doesn't guess.

### Diffs

A bump of `buffer-impl` to 3.7.2 changes the binding lines and nothing else:

```diff
   fs-impl@4.8.4:
     - dependencies:
         path-impl: 3.1.2
       peerDependencies:
-        buffer-impl: 3.7.1
+        buffer-impl: 3.7.2
   container@3.2.2:
     - dependencies:
         fs-impl: 4.8.4
       transitivePeerDependencies:
-        buffer-impl: 3.7.1
+        buffer-impl: 3.7.2
```

A package that didn't change and binds a new optional peer, the
pnpm/pnpm#17773 case, gains one line per snapshot that binds it:

```diff
   fs-impl@4.8.4:
     - dependencies:
         path-impl: 3.1.2
+      peerDependencies:
+        bare-buffer: 2.6.1
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
- The field is in `packages`, not `snapshots`, because it is the same for
  every peer variant of the package.
- The field is required on every directory entry. The frozen-lockfile check
  compares it with the package's current manifest, and doesn't derive the
  children from the manifest.

pnpm/pnpm#17477 already writes the field in lockfile 9.0. The next format
makes it required and removes the fallback for entries without it.

### Performance

The reference parser does less: a key is split at one `@`, and a reference is
a version. Following an edge is a probe into the package's list and a
comparison of two small maps, in place of parsing a suffix. The comparison is
bounded by the number of peer names the child binds. The file is smaller by the share the
suffixes take. Lockfile reuse in the resolver matches variants the same way,
so the hashed-suffix recovery and its search are removed. A provider bump no
longer renames variant directories.

These are expectations. The claim to verify on a prototype is that
`isolated-fresh-restore` and `repeat-install` do not regress, and that a
"bump a widely bound peer and reinstall" scenario, which the integrated
benchmark lacks, improves.

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
8. **This proposal.** Alternative 7 with the key reduced to `name@version`,
   which is possible because the parent's context already identifies the
   variant. No grammar, no collision rule, no hash, and the diff is the set
   of bindings that changed. The cost is that an edge is a match instead of
   a lookup, and every consumer has to implement the match.

## Implementation

pnpm v12 only. pnpm v11 keeps lockfile 9.0, including the optional
`specifiers` field that pnpm/pnpm#17477 adds to both CLIs.

- `pnpm/crates/lockfile`: `snapshots` becomes a map from a package key to a
  list of `SnapshotEntry`. `SnapshotEntry` gets `peer_dependencies`, and
  `transitive_peer_dependencies` becomes a map. `ProjectSnapshot` gets
  `transitive_peer_dependencies`. A function follows a reference from a
  context, with the rule above, and is the one place that rule lives. Reading
  a 9.0 lockfile converts it. The suffix becomes the maps, and children that
  match the `packages` entry's `peerDependencies` move to
  `peer_dependencies`. A hashed suffix is recovered once by today's walk.
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
  binding map. A changed binding is a changed field on the same snapshot, not
  a removal and an addition.
- `pnpm/crates/package-manager/src/install/lockfile_freshness/directory_deps`:
  compare `specifiers` with the manifest and remove the derived walk.
- Published readers: the `@pnpm/lockfile.*` packages and the specification
  carry the reference-following rule, so Bit, Renovate, Dependabot, and
  scanners implement it once.
- Docs: `pnpm/docs/lockfile-format.md`, and the settings page for
  `peersSuffixMaxLength`.

## Unresolved Questions and Bikeshedding

- **The version number.** pnpm/rfcs#68 records the decision to avoid lockfile
  version bumps. This change alters the shape of `snapshots`, so an older
  reader cannot read it, and it needs a new `lockfileVersion`. pnpm v12
  already accepts major `12`.
- **The context when a parent's dependency and its subtree disagree.** The
  table defines `transitivePeerDependencies` so that a binding below the
  parent that differs from the parent's own dependency of that name is
  recorded. Check in the resolver whether this can happen. If it cannot, the
  definition can be the simpler "peers bound from outside the package".
- **Unbound transitive peers.** Today they are listed. Does any code depend
  on knowing that a peer below a snapshot was missing, for example to
  re-resolve when a provider is added? If so, they need their own list.
- **One snapshot or a list.** Nearly every package has one variant. The list
  could be replaced by a single map when there is one variant, at the cost of
  two shapes for every consumer.
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
