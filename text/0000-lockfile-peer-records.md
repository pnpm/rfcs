# Record resolution inputs and peer bindings in the lockfile

## Summary

The next lockfile format records what the resolver decided and what it read,
so that code reading the lockfile doesn't have to work either out again.

- A snapshot records its resolved peer dependencies in its own
  `peerDependencies` map, separate from the package's own `dependencies` and
  `optionalDependencies`. A name the package declares as both a dependency and
  a peer is recorded in the map that matches how it was bound.
- `transitivePeerDependencies` becomes a map from each peer name to the
  provider it resolved to. Together, the two maps hold every peer that the
  snapshot's key suffix stands for, so the key can be rebuilt from the
  snapshot, even when the suffix is hashed.
- Every `packages` entry of a directory package records `specifiers`: the
  dependency specifiers the resolver read from the package's manifest.

## Motivation

These problems came up while writing a specification of the lockfile format
for the pnpm website (`pnpm/docs/lockfile-format.md` in pnpm/pnpm).

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

This shows up as noise in lockfile diffs. In pnpm/pnpm#17773, a minor release
of one package added a dependency on `bare-buffer`. pnpm then resolved
`bare-buffer` as the optional peer of two packages whose versions did not
change. Both gained an `optionalDependencies` block for a dependency neither
of them declares.

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
pnpm/tasks#221 tracks this, and proposes recording the decision in the
snapshot.

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

### transitivePeerDependencies mixes two cases

`transitivePeerDependencies` lists the peers that packages below a snapshot
need and that the subtree does not provide. A name in the list is in one of
two states, and the list does not say which:

- Resolved from outside the package. The provider is part of the key suffix.
- Not resolved at all. It is not part of the key.

```yaml
  container@3.2.2(buffer-impl@3.7.1):
    dependencies:
      fs-impl: 4.8.4(buffer-impl@3.7.1)
    transitivePeerDependencies:
      - abort-impl    # not installed
      - buffer-impl   # resolved, in the suffix
```

### A hashed suffix cannot be read back from its snapshot

When the peer suffix is longer than `peersSuffixMaxLength`, pnpm replaces it
with a hash. The key then names no peers. To find out which providers a
hashed key stands for, pnpm walks the snapshot's descendants. It looks for the
packages that declare each name in `transitivePeerDependencies` and reads
which version they link. A tool that reads the lockfile has to do the same
walk. When the walk is not enough, pnpm searches the subsets of the declared
peers for one whose hash matches the key. That search is bounded, so it can
fail.

## Detailed Explanation

### Snapshots

A snapshot gets these maps:

| Key | Content |
| --- | --- |
| `dependencies` | The manifest's `dependencies`, and nothing else. |
| `optionalDependencies` | The manifest's `optionalDependencies`, and nothing else. |
| `peerDependencies` | The package's own peer dependencies that are resolved, as `name: reference`. These are linked next to the package. A name the package also declares as a dependency is here when a parent provided it. |
| `transitivePeerDependencies` | The peers of packages below this one that are resolved from outside this package, as `name: reference`. These are not linked next to the package. |

References use the existing forms: a version with suffixes, `<name>@<version>`
for an alias, or `link:<path>`.

The example above becomes:

```yaml
snapshots:
  container@3.2.2(buffer-impl@3.7.1):
    dependencies:
      fs-impl: 4.8.4(buffer-impl@3.7.1)
    transitivePeerDependencies:
      buffer-impl: 3.7.1

  fs-impl@4.8.4(buffer-impl@3.7.1):
    dependencies:
      path-impl: 3.1.2
    peerDependencies:
      buffer-impl: 3.7.1
```

Rules:

- The key suffix is built from the union of `peerDependencies` and
  `transitivePeerDependencies`, with the same grammar and the same hashing as
  today. A reader rebuilds an unhashed suffix from the snapshot alone, and
  checks a hashed one the same way.
- Whether a peer edge is optional is read from the `packages` entry's
  `peerDependenciesMeta`. The installer treats a failure on an optional peer
  edge as it does today.
- A name is in at most one of `dependencies`, `optionalDependencies`, and
  `peerDependencies`. A name declared as both a dependency and a peer is in
  `peerDependencies` if a parent provided it. It is in its dependency map if
  the package's own dependency was installed.
- An unresolved peer is not recorded in the snapshot. The `packages` entry
  already lists every peer the package declares.

In the example from the motivation, the snapshot body says which binding was
made:

```yaml
# A parent provided widget.
peerDependencies:
  widget: 2.0.0
```

```yaml
# No parent provided widget, so shared's own dependency was installed.
dependencies:
  widget: 2.0.0
```

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

## Rationale and Alternatives

1. **Record resolved peers under `dependencies` only.** This removes the
   optional-dependency confusion with no new key. The installer would read
   `peerDependenciesMeta` to decide whether an edge is optional. A reader still
   cannot tell a dependency from a peer edge without checking the `packages`
   entry, and a hashed suffix still cannot be read back.
2. **Add a list of the dependencies that a peer replaced.** pnpm/tasks#221
   proposes this, for example `peerShadowedDependencies: [widget]`. The field
   is additive. Older pnpm versions ignore it, and new readers fall back to
   suffix recovery when it is absent, so no version change is needed. It
   fixes the dependency-and-peer ambiguity only. Optional peers stay under
   `optionalDependencies`, and hashed suffixes still need recovery. It is
   also open whether older versions keep an unknown snapshot field when they
   rewrite the lockfile.
3. **Keep `specifiers` optional in lockfile 9.0** (pnpm/pnpm#17477). This
   needs no format change. The derived walk stays as a fallback until every
   lockfile has been rewritten, and nothing tells pnpm when that is.
4. **Keep the format and document it.** The specification now describes the
   current behavior. This costs nothing, but the diff noise and the hashed
   suffix walk remain.
5. **Separate `peerDependencies` and a `transitivePeerDependencies` map,
   and required `specifiers`**
   (this proposal). Each map holds one kind of edge, the snapshot holds all
   the data its key is derived from, and a directory entry holds the inputs
   its children were resolved from. The cost is a format change that rewrites
   every snapshot that has a resolved peer.

## Implementation

pnpm v12 only. pnpm v11 keeps the current format, including the optional
`specifiers` field that pnpm/pnpm#17477 adds to both CLIs.

- `pnpm/crates/lockfile`: `SnapshotEntry` gets `peer_dependencies`, and
  `transitive_peer_dependencies` becomes a map. Reading a current-format
  lockfile converts it: children that match the `packages` entry's
  `peerDependencies` move to `peer_dependencies`, and each transitive peer name
  is resolved to its provider by the walk used today.
- `pnpm/crates/package-manager/src/dependencies_graph_to_lockfile/packages.rs`:
  write peer edges to the new map. The resolver already knows which children
  are peers.
- `pnpm/crates/resolving-deps-resolver/src/resolve_importer/locked_peers/hashed_suffix.rs`:
  read the providers of a hashed suffix from the two maps, without walking the
  descendants.
- `pnpm/crates/cli/src/cli_args/deploy`: read peer edges from
  `peerDependencies` instead of deciding them from the peer suffix.
- `pnpm/crates/package-manager/src/install/lockfile_freshness/directory_deps`:
  compare `specifiers` with the manifest and remove the derived walk. See the
  question about older lockfiles below.
- `pnpm/crates/deps-restorer`: link `peerDependencies` like other children,
  and take optionality from `peerDependenciesMeta`.
- Docs: update `pnpm/docs/lockfile-format.md`.

## Unresolved Questions and Bikeshedding

- **The version number.** pnpm/rfcs#68 records the decision to avoid lockfile
  version bumps. This change alters what an older reader sees in
  `dependencies`, so it needs a new `lockfileVersion`. pnpm v12 already accepts
  major `12`.
- **Unresolved transitive peers.** Today they are listed. Does any code depend
  on knowing that a peer below a snapshot was missing, for example to
  re-resolve when a provider is added? If so, they need their own list.
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
    with `@`, or it has an `@` before any `(` or `:`. An explicit form would
    remove the guess.
  - A `packages` entry carries `version` only when its key's version is a URL,
    a `file:` path, or a `runtime:` version, except for directories.
