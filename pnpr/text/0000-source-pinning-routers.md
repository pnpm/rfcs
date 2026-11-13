# Source-pinning routers and direct upstream fetch

> Status: draft. Extends [pnpr registries](./0000-pnpr-registry-mounts.md).

## Summary

Two opt-in settings, both off by default. With both off, pnpr behaves as it
does today.

1. **`pinSources` on a router.** The router names the concrete **source**
   registry of every version in its metadata and points `dist.tarball` at the
   source's own URL. pnpm records the source in the lockfile package key
   (`lodash@npmjs:4.17.21`) and rebuilds the tarball URL from the source, not
   from the router. No tarball URL is written to the lockfile.
2. **`fetch: direct` on a public upstream.** Its tarballs are fetched from the
   origin, without passing through pnpr. This builds on the first setting.

This is the lockfile follow-up that the registries RFC left open under
[Future lockfile requirement][future-requirement], offered as an option
instead of a replacement for the current behavior.

[future-requirement]: ./0000-pnpr-registry-mounts.md#client-routing-and-lockfiles

## Motivation

### One tarball has many URLs

A router rewrites `dist.tarball` onto the base the client addressed. With this
configuration:

```yaml
registries:
  npmjs: { type: upstream, url: https://registry.npmjs.org/, public: true }
  team-a: { type: router, sources: [a-private, npmjs] }
  team-b: { type: router, sources: [b-private, npmjs] }
defaultRegistry: team-a
```

the same bytes are served at four URLs:

```text
/npm/lodash@4.17.21
/npm/~team-a/lodash@4.17.21
/npm/~team-b/lodash@4.17.21
/npm/~npmjs/lodash@4.17.21
```

pnpr's own upstream cache is keyed by the upstream, so it stores one copy. A
CDN or any other shared HTTP cache is keyed by URL, so it stores four and takes
four cold misses per edge location. The number grows with every router that
lists the upstream.

### The lockfile does not say where a package came from

A package reached through a router gets a plain key. Reordering the router's
sources, or moving a name from one source to another, changes the origin of a
locked package and the lockfile shows no diff. Registry-qualified keys were
added to pnpm to close this substitution gap for named registries. A router
reopens it one level down.

The registries RFC states the fix: "When a package is reached through a
router, that future identity should be the concrete source it resolved to, not
the router, so a later router edit cannot silently change a locked package's
origin."

### Public tarballs do not need to pass through pnpr

pnpr's resolve endpoint already leaves a public package on its upstream URL, so
the client downloads it from the origin or its CDN. The registry path cannot do
this: it always rewrites `dist.tarball` onto pnpr. An operator who wants pnpr
for routing, policy, and private packages still pays storage and egress for
every public tarball.

### Why these are options

Each setting trades something away, and operators weigh the trade differently.

| | Default | `pinSources` | `pinSources` + `fetch: direct` |
| --- | --- | --- | --- |
| Tarball URLs per package | one per base | one | one, at the origin |
| Lockfile records the source | no | yes | yes |
| pnpr can block a version that is already locked | yes | yes | no |
| pnpr mirrors the bytes | yes | yes | no |
| Lockfile changes when a registry is renamed or a package moves between sources | no | yes | yes |
| Works with clients that do not know the setting | yes | yes, but they write tarball URLs into their lockfiles | same |

## Detailed Explanation

### `pinSources`

```yaml
registries:
  main:
    type: router
    sources: [private, npmjs]
    pinSources: true
```

Every version in a packument or version manifest served through a pinning
router carries the name of its source:

```json
{
  "name": "lodash",
  "version": "4.17.21",
  "dist": {
    "integrity": "sha512-...",
    "tarball": "https://pnpr.example.com/npm/~npmjs/lodash@4.17.21",
    "pnpr": { "source": "npmjs" }
  }
}
```

- `source` is the source that claims the name.
- `dist.tarball` points at the source's `/npm/~<source>/` base. Clients that do
  not read `dist.pnpr` follow it.
- The `/npm/` base behaves like the registry that `defaultRegistry` names.
- A concrete registry that a pinning router lists names itself the same way
  when it is addressed at its own base. A package then has one key, whichever
  base a project addresses.
- The router's own tarball routes keep serving, so lockfiles written before
  the setting was turned on still install.

A router without `pinSources` is unchanged: `dist.tarball` stays on the
addressed base and no `source` is emitted.

### Lockfile

pnpm keys a package by the source its metadata names:

```yaml
packages:
  lodash@npmjs:4.17.21:
    resolution: {integrity: sha512-...}
  '@acme/ui@private:2.0.0':
    resolution: {integrity: sha512-...}
```

- A direct dependency and a transitive one resolve to the same key, so one
  copy is installed.
- pnpm reads `dist.pnpr.source` only from a registry declared with
  `serverType: pnpr`. From any other registry it is ignored.
- A version without `dist.pnpr.source` is keyed as it is today.

### Mapping a source name to a URL

A frozen install has no metadata, so the URL must come from the key and the
project's configuration. pnpm resolves a source name in this order:

1. A registry that declares that name as its `prefix` in `registries`.
2. A sibling of the pnpr registry the package name routes to. For a routed
   base of `https://pnpr.example.com/npm/~main/` or
   `https://pnpr.example.com/npm/`, the source `npmjs` is
   `https://pnpr.example.com/npm/~npmjs/`.

A project that only configures a pnpr base needs no extra declaration.

At resolution time pnpm compares the URL it would rebuild with `dist.tarball`.
When they differ, it writes the tarball URL into the lockfile entry, as it does
today for any URL it cannot rebuild. The lockfile is always installable, and it
is URL-free whenever the project's configuration agrees with the server.

### `fetch: direct`

```yaml
registries:
  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true
    fetch: direct
```

- `dist.tarball` is the origin's own tarball URL. pnpr still serves and filters
  the upstream's metadata.
- The upstream names itself in `dist.pnpr.source` on every base that reaches
  it, so a router that lists it must set `pinSources`. pnpr refuses to start
  otherwise.
- It is accepted only on a `public: true` upstream that serves `.tgz`.

The project declares the origin under the source's name:

```yaml
registries:
  https://registry.npmjs.org/:
    prefix: npmjs
```

Pointing the same prefix at `https://pnpr.example.com/npm/~npmjs/` instead
sends the downloads through pnpr. The lockfile key is the same in both cases.

What direct fetch gives up:

- A frozen install never asks pnpr about a direct package. A version that pnpr
  starts hiding after it was locked still installs.
- pnpr does not mirror the bytes, so an origin outage fails the install.
- Clients need network access to the origin.
- pnpr sees no download of the package.

### Server-side resolution

`POST /-/pnpr/v0/resolve` returns the same keys and the same tarball locations
as a client-side resolution through the registry path.

### Compatibility

Both settings are additive, so they can ship after pnpr 1.0:

- `dist.pnpr.source` is a new field. pnpr already strips any `dist.pnpr` that
  an upstream sends.
- `pinSources` and `fetch` are new configuration keys that default to today's
  behavior.
- pnpm changes a key only when the field is present.

One thing has to hold before 1.0: a registry name must be usable as a lockfile
key prefix. pnpm's prefix grammar requires a leading letter, then letters,
digits, `.`, `_`, and `-`, and rejects reserved specifier prefixes such as
`npm` and `workspace`. pnpm/pnpm#18043 restricts pnpr registry names to that
grammar.

## Rationale and Alternatives

### Make source pinning the only behavior

Every router pins its sources and stops serving tarballs under its own URL.
This gives one model to explain, but it forces registry names into every
lockfile, re-keys lockfiles when an operator renames a registry, and makes
older clients write tarball URLs. An operator who only wants a registry facade
gets those costs with no benefit.

### Redirect only

A router answers each tarball request with a redirect to
`/npm/~<source>/<pkg>@<version>`. This fixes the duplicate cache entries with
no client or lockfile change, and a Worker in front of pnpr can follow the
redirect so the client sees one response. It does not record the source in the
lockfile and does not allow direct fetch. It remains a cheap fix for the cache
problem alone.

### Client-side routing with declared prefixes

The project declares every concrete registry and routes scopes to them itself.
The router leaves the path, so its dependency-confusion guarantee is lost, the
routing rules are copied into every project, and a router that claims unscoped
names cannot be expressed.

### Rewrite dependency specifiers in packuments

A router rewrites `"bar": "^1.0.0"` to `"bar": "npmjs:^1.0.0"`. Direct
dependencies are not covered, so one package gets a plain key and a qualified
key and is installed twice. Clients other than pnpm cannot read the
specifiers.

### Infer the source from the tarball URL

pnpm matches `dist.tarball` against the declared registries and takes the
prefix of the one it falls under. This needs no new metadata field, but every
source must be declared in every project, and a project cannot choose between
direct and mirrored downloads without changing what the server emits.

## Implementation

### pnpr

- `rewrite_upstream_tarball_urls` in `crates/upstream/src/packument.rs` takes
  the base to rewrite onto. A pinning router passes the source's base.
- The `dist.pnpr` object is already written by `server/tar_registry.rs`. It
  gains `source`.
- `resolver/wire/tarball_router.rs` emits source-qualified keys and the same
  locations.
- `pinSources` is added to the router config and `fetch` to the upstream
  config, with the validation described above.

### pnpm

- The key prefix is chosen before metadata is fetched, from configuration
  alone (`resolving-npm-resolver/src/npm_resolver/registry_dependency.rs`).
  For a `serverType: pnpr` registry it is chosen after the pick, in
  `build_resolve_result`.
- The store fast path (`npm_resolver/store_peek.rs`) creates a package id
  without metadata. It needs the source recorded in the store index, or it is
  skipped for `serverType: pnpr` registries.
- `registry_and_version` in
  `deps-restorer/src/install_package_by_snapshot/tarball_resolution.rs` and
  `metadata_registry` in
  `package-manager/src/dependencies_graph_to_lockfile/packages.rs` gain the
  sibling lookup.
- `rekeyed` in `lockfile/src/resolution/registry.rs` derives keys from
  configuration. It cannot produce a source-qualified key for an existing
  plain one, so turning `pinSources` on re-resolves the lockfile once.

This is a pnpm v12 change. A client without it follows `dist.tarball` and
writes the URL of every package from a pinning router into its lockfile.

## Prior Art

vlt makes the registry a component of package identity in its lockfile keys.
pnpm's registry-qualified keys for named registries follow the same idea and
are the encoding this RFC reuses.

## Unresolved Questions and Bikeshedding

- The names `pinSources` and `fetch: direct`.
- Whether a built-in prefix such as `npmjs` should win over the sibling lookup,
  so that a direct npm upstream needs no declaration in the project.
- A pinning router with `tarballs: tar` over a `.tgz` source serves different
  bytes than the source's own base. Should that combination be refused, or
  should the router keep its own tarball URL for it?
- Should a concrete registry name itself only when a pinning router lists it,
  as proposed, or whenever it is addressed?
- Renaming a registry in `pnpr.yaml` re-keys every lockfile that pinned it.
  Should a registry be able to declare a stable identity apart from its name?
