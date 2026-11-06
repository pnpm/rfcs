# Record peer bindings explicitly in lockfile snapshots

## Summary

In the next lockfile format, a snapshot records its resolved peer dependencies
in its own `peerDependencies` map, separate from the package's own
`dependencies` and `optionalDependencies`. `transitivePeerDependencies` becomes
a map from each peer name to the provider it resolved to. Together, the two
maps hold every peer that the snapshot's key suffix stands for, so the key can
be rebuilt from the snapshot, even when the suffix is hashed.

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
walk.

## Detailed Explanation

A snapshot gets these maps:

| Key | Content |
| --- | --- |
| `dependencies` | The manifest's `dependencies`, and nothing else. |
| `optionalDependencies` | The manifest's `optionalDependencies`, and nothing else. |
| `peerDependencies` | The package's own peer dependencies that are resolved, as `name: reference`. These are linked next to the package. |
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
- An unresolved peer is not recorded in the snapshot. The `packages` entry
  already lists every peer the package declares.

## Rationale and Alternatives

1. **Record resolved peers under `dependencies` only.** This removes the
   optional-dependency confusion with no new key. The installer would read
   `peerDependenciesMeta` to decide whether an edge is optional. A reader still
   cannot tell a dependency from a peer edge without checking the `packages`
   entry, and a hashed suffix still cannot be read back.
2. **Keep the format and document it.** The specification now describes the
   current behavior. This costs nothing, but the diff noise and the hashed
   suffix walk remain.
3. **Separate `peerDependencies` and a `transitivePeerDependencies` map**
   (this proposal). Each map holds one kind of edge, and the snapshot holds all
   the data its key is derived from. The cost is a format change that rewrites
   every snapshot that has a resolved peer.

## Implementation

pnpm v12 only. pnpm v11 keeps the current format.

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
