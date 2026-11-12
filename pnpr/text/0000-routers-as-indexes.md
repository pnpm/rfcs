# Routers as indexes: source identity for routed packages

> Status: draft. Amends [pnpr registries](./0000-pnpr-registry-mounts.md).

## Summary

A router should only decide which concrete registry owns a package name. It
should not serve that package's tarball under its own URL.

Every version pnpr serves names its concrete **source** registry in its
metadata. pnpm records that source in the lockfile package key
(`lodash@npmjs:4.17.21`) and rebuilds the tarball URL from the source, not from
the router the project is configured with. No tarball URL is written to the
lockfile.

Because the source decides where its tarballs live, a public upstream can also
declare that its tarballs are fetched from the origin directly, without passing
through pnpr.

This is the lockfile follow-up that the registries RFC left open under
[Future lockfile requirement][future-requirement]. It replaces that RFC's
rule that a router serves tarballs on the base the client addressed.

[future-requirement]: ./0000-pnpr-registry-mounts.md#client-routing-and-lockfiles

## Motivation

### One tarball has many URLs

Today a router rewrites `dist.tarball` onto the base the client addressed. With
this configuration:

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
added to pnpm to close exactly this substitution gap for named registries. A
router reopens it one level down.

The registries RFC already states the fix: "When a package is reached through a
router, that future identity should be the concrete source it resolved to, not
the router, so a later router edit cannot silently change a locked package's
origin."

### Public tarballs do not need to pass through pnpr

pnpr's resolve endpoint already leaves a public package on its upstream URL, so
the client downloads it from the origin or its CDN. The registry path cannot do
this: it always rewrites `dist.tarball` onto pnpr. An operator who wants pnpr
for routing, policy, and private packages still pays storage and egress for
every public tarball.

## Detailed Explanation

### Source identity in metadata

Every version in a packument or version manifest that pnpr serves carries the
name of its concrete source:

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

- A router sets `source` to the source that claims the name.
- A concrete registry addressed at its own `/~<name>/` base sets `source` to
  its own name.
- The `/npm/` base sets what the registry named by `defaultRegistry` would set.
- `dist.tarball` points at the source's tarball location (see below), whichever
  base the client addressed. Clients that do not read `dist.pnpr` follow it.

A packument for one package is therefore the same document on every base that
reaches the same source.

### Where a source's tarballs live

| Source | Tarball location |
| --- | --- |
| hosted | `<pnpr>/npm/~<name>/<pkg>@<version>` |
| upstream (default) | `<pnpr>/npm/~<name>/<pkg>@<version>`, fetched, verified, and cached by pnpr |
| upstream with `fetch: direct` | the origin's own tarball URL |

`fetch: direct` is accepted only on a `public: true` upstream that serves
`.tgz`. pnpr still serves and filters that upstream's metadata.

A router no longer serves tarball bytes. Its tarball routes answer with a
redirect to the source's location, so lockfiles and clients from before this
change keep working.

### Lockfile

pnpm keys a package by its source:

```yaml
packages:
  lodash@npmjs:4.17.21:
    resolution: {integrity: sha512-...}
  '@acme/ui@a-private:2.0.0':
    resolution: {integrity: sha512-...}
```

- The key is the same whether the project addressed a router, the `/npm/`
  base, or the source's own base.
- A direct dependency and a transitive one resolve to the same key, so one
  copy is installed.
- The rule that the default registry's packages keep plain keys no longer
  applies to packages whose metadata names a source.
- pnpm reads `dist.pnpr.source` only from a registry declared with
  `serverType: pnpr`. From any other registry it is ignored.

### Mapping a source name to a URL

A frozen install has no metadata, so the URL must come from the key and the
project's configuration. pnpm resolves a source name in this order:

1. A registry that declares that name as its `prefix` in `registries`.
2. A sibling of the pnpr registry the package name routes to. For a routed
   base of `https://pnpr.example.com/npm/~team-a/` or
   `https://pnpr.example.com/npm/`, the source `npmjs` is
   `https://pnpr.example.com/npm/~npmjs/`.

A project that only configures a pnpr base needs no extra declaration.

At resolution time pnpm compares the URL it would rebuild with `dist.tarball`.
When they differ, it writes the tarball URL into the lockfile entry, as it does
today for any URL it cannot rebuild. The lockfile is always installable, and it
is URL-free whenever the project's configuration agrees with the server.

For a `fetch: direct` upstream, the project declares the origin under the
source's name:

```yaml
registries:
  https://registry.npmjs.org/:
    prefix: npmjs
```

Pointing the same prefix at `https://pnpr.example.com/npm/~npmjs/` instead
sends the downloads through pnpr. The lockfile key is the same in both cases.

### What direct fetch gives up

- A frozen install never asks pnpr about a direct package. A version that pnpr
  starts hiding after it was locked still installs.
- pnpr does not mirror the bytes, so an origin outage fails the install.
- Clients need network access to the origin.
- pnpr sees no download of the package.

### Server-side resolution

`POST /-/pnpr/v0/resolve` returns the same keys and the same tarball locations
as a client-side resolution through the registry path.

## Rationale and Alternatives

### Redirect only

A router answers each tarball request with a redirect to
`/npm/~<source>/<pkg>@<version>`. This fixes the duplicate cache entries with
no client or lockfile change, and a Worker in front of pnpr can follow the
redirect so the client sees one response. It does not record the source in the
lockfile and does not allow direct fetch. This RFC keeps the redirect as the
compatibility path.

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
  the base to rewrite onto. A router request passes the source's location.
- The `dist.pnpr` object is already written by `server/tar_registry.rs`. It
  gains `source`.
- Router tarball routes in `server/routing.rs` redirect.
- `resolver/wire/tarball_router.rs` emits source-qualified keys and the same
  locations.
- `fetch` is added to the upstream registry config and validated against
  `public` and `tarballs`.

### pnpm

- The key prefix is chosen before metadata is fetched, from configuration
  alone (`resolving-npm-resolver/src/npm_resolver/registry_dependency.rs`). It
  moves after the pick, in `build_resolve_result`.
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
  plain one, so adoption re-resolves the lockfile once.

This is a pnpm v12 change. A client without it follows `dist.tarball` and
writes the URL of every routed package into its lockfile.

## Prior Art

vlt makes the registry a component of package identity in its lockfile keys.
pnpm's registry-qualified keys for named registries follow the same idea and
are the encoding this RFC reuses.

## Unresolved Questions and Bikeshedding

- The name and values of the upstream setting: `fetch: direct`, or another
  spelling.
- Whether a built-in prefix such as `npmjs` should win over the sibling lookup,
  so that a direct npm upstream needs no declaration in the project.
- Whether a router keeps its own `tarballs: tar` setting. A router that serves
  no bytes has no format, so a tar and a `.tgz` view of one origin become two
  upstream registries with two caches.
- Whether hosted and private sources should be qualified too when the project
  addresses them through a router, or only public upstreams. This RFC
  qualifies all of them.
- Renaming a registry in `pnpr.yaml` re-keys every lockfile that used it.
  Should a registry be able to declare a stable identity apart from its name?
- How long the router redirect stays.
