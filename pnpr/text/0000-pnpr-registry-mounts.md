# Registry mounts for pnpr

## Summary

pnpr should model every addressable registry origin as a **registry mount**.
There are exactly two concrete mount kinds — a pnpr-**hosted** organization
registry and a single-origin **upstream** registry — plus one composite, a
**router** that maps package-name patterns to concrete mounts. Named mounts are
exposed at `https://<pnpr>/~<mount>/`, so every origin has an explicit identity
in the URL, in pnpr's internal routing, and in client resolution.

The design is governed by one invariant:

> **Provenance is declared, never inferred. Every package resolves to exactly
> one declared concrete origin, and no configuration can express a cross-origin
> fall-through — not on "not found", and not on "unavailable".**

There is no existence-based fallback (try private, else public), no mirror group,
and no multi-endpoint failover. Those are the constructs that let a transient
outage or a missing name silently switch a package to a different origin — the
dependency-confusion class — and the model omits them by construction rather
than mitigating them. Outage resilience comes from pnpr's own cache, never from
trying a different origin.

The pnpr-server implementation keeps lockfiles deployment-portable by serving
canonical tarball URLs for the registry base the client addressed, so pnpm can
reuse its existing tarball-URL reconstruction rather than persisting pnpr URLs.
A later pnpm/pacquet lockfile follow-up should record **registry identity** in
package identity, so the same `name@version` from two different mounts cannot
collide — a gap that exists in pnpm today but is outside PR 12747.

## Motivation

pnpr currently inherits a Verdaccio-shaped mental model: the server is one
registry facade, package rules map names to uplinks, and an uplink list is a
fallback chain. That made sense when clients generally had one registry URL and
the proxy had to decide where every package came from by looking only at the
package name.

pnpm now has named registries and pnpr already has origin-qualified
`/~<uplink>/` endpoints for private proxied tarballs. The client, the resolver,
and the server can all carry a registry identity explicitly instead of
rediscovering it by walking a fallback chain.

Existence-based fallback chains are unsafe across security and provenance
domains:

- the same `name@version` can exist in two registries with different bytes,
  metadata, policies, or vulnerability posture;
- a tarball request that re-runs fallback can pick a different origin from the
  packument request that selected the version and `dist.integrity`;
- shared proxy caches need extra safeguards to avoid mixing private and public
  content for the same package name;
- server-owned upstream credentials need a stable route identity so they never
  leak into client lockfiles or another mount's cache;
- most importantly, **an outage or a missing name in a private origin must never
  resolve to a public origin**. Treating "unavailable" or "not found" as "look
  elsewhere" is the dependency-confusion mechanism: an attacker who registers a
  private package's name publicly is served the moment the private origin 404s
  or is briefly down. The mount model forbids that fall-through entirely.

### pnpm cannot represent the same `name@version` from two registries today

This is not only a pnpr concern — it is a concrete, pre-existing gap in pnpm's
lockfile model, and it is the strongest motivation for first-class registry
identity.

pnpm already has a named-registries feature (`gh:`, `jsr:`, and user-configured
`namedRegistries`). When the resolver resolves such a dependency it knows the
registry it came from, but the package id it builds drops that fact:

```ts
// resolving/npm-resolver/src/index.ts (pickFromSimpleRegistry)
id: `${pickedPackage.name}@${pickedPackage.version}` as PkgResolutionId,
```

The `registryName` is returned in the resolver result but never reaches the
lockfile key — it is consumed only inside `npm-resolver`. The package key is
`name@version`, and the `packages`/`snapshots` maps hold exactly one entry per
key. So two registries serving the same `name@version` with different bytes
**collide on one key**, and whichever resolves first wins; pnpm does not error,
it silently treats them as the same package.

A single registry still round-trips correctly because the stored resolution is a
`TarballResolution` carrying the real per-registry tarball URL. The gap is
strictly: two registries, same `name@version`, in one graph. It is latent today
because named registries are niche, but the mount model surfaces it routinely,
so the lockfile must carry registry identity.

### Product framing

If pnpr is launched as a standalone product, the default product primitive is
not "hosted" as an implementation detail. It is an organization registry:
`/~acme/` is the registry for the `acme` organization, and it may host packages
directly on pnpr. A package scope such as `@scope/pkg` remains an npm package
name inside that organization registry; it is independent from the pnpr
organization in the URL.

The desired outcome is a design where:

- `~acme` can serve only packages hosted by the `acme` organization;
- `~npmjs` can serve the public npm registry;
- `~corp` can serve a private upstream with pnpr-managed credentials;
- a router can present several of these behind one URL, routing each package to
  exactly one of them by an explicit name pattern — never by guessing;
- private packages are declared (by scope/pattern or named registry), so a
  private name can never silently resolve to a public origin;
- the path-less base URL is an optional, configurable default target, never a
  privileged registry — every mount is already a registry root in its own right.

## Detailed Explanation

### Registry mounts

A registry mount is an addressable npm registry surface. It has a stable mount
ID and is available at:

```text
https://<pnpr>/~<mount>/
```

Mount IDs are pnpr route names, not npm package scopes. They must be path-safe,
operator-controlled identifiers, and pnpr enforces that at config load: a mount
name must be a single URL-safe path segment (no separators, traversal, `%`,
`?`, `#`, whitespace, control characters, or Windows drive prefixes), because
it is addressed as one path segment and embedded verbatim in rewritten
`dist.tarball` URLs. A name that cannot survive that round trip is a startup
error, not an unreachable mount. The leading `~` keeps mount routes out of the
normal npm package-name space and lets `@scope/pkg` keep its existing meaning
under every mount.

The path-less base URL (no mount in the path) resolves to one named mount via a
configurable default target (see
[Default target and the path-less base](#default-target-and-the-path-less-base)).

Illustrative config:

```yaml
mounts:
  acme:
    type: hosted
    org: acme
    access: team:acme

  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true

  corp:
    type: upstream
    url: https://npm.corp.example/
    auth:
      tokenEnv: CORP_NPM_TOKEN
    access: team:acme

  # One URL that routes each package to exactly one concrete mount by the first
  # matching explicit name pattern. No existence-based fallback.
  main:
    type: router
    routes:
      - patterns: ['@acme/*']
        source: acme
      - patterns: ['@corp/*']
        source: corp
      - patterns: ['**']
        source: npmjs

defaultTarget: main
```

The YAML syntax is intentionally tagged by `type:` (`hosted`, `upstream`,
`router`) so a mount cannot accidentally declare more than one kind. The
required model is two concrete mount kinds and one composite:

```rust
enum RegistryMount {
    Hosted {
        org: OrgId,
        access: AccessPolicy,
    },
    /// Exactly one external origin. Not a chain and not a set of endpoints:
    /// one URL, one credential generation, one cache namespace.
    Upstream {
        url: Url,
        auth: Option<UpstreamAuth>,
        access: Option<AccessPolicy>,
        cache: CachePolicy,
    },
    /// Maps package-name patterns to concrete mounts in declared order. The
    /// first matching route resolves to that route's source — authoritatively.
    /// A source's "not found" or "unavailable" is final; the router never
    /// tries another.
    Router {
        routes: Vec<Route>,
    },
}

struct Route {
    patterns: Vec<PackagePattern>,
    source: MountId, // a Hosted or Upstream mount; never another Router member by existence
}
```

A mount owns:

- its authorization policy for the pnpr caller;
- its upstream credential, if it has one;
- its metadata and tarball cache namespace;
- its storage namespace, if it hosts packages;
- its tarball URL base;
- its package policy and OSV/advisory policy context.

The important invariant is that a package request resolves to one concrete
origin descriptor before metadata, tarballs, cache, or advisory decisions are
made.

### Hosted organization mounts

A hosted organization mount serves only that organization's registry — its
packuments and tarballs (how those bytes are produced is a backend detail,
below):

```text
GET /~acme/foo
GET /~acme/foo/-/foo-1.0.0.tgz
GET /~acme/@scope/pkg
GET /~acme/@scope/pkg/-/pkg-1.0.0.tgz
```

`acme` is the pnpr organization. `@scope/pkg` is still just the npm package
name. This avoids a generic `~hosted` URL that describes implementation rather
than ownership.

Hosted organization mounts have no upstream fallback. A request to `/~acme/foo`
returns `foo` only if the `acme` organization hosts `foo` and the caller is
authorized to read it.

The implemented hosted mount is pnpr-native storage with a per-org namespace.
The YAML `org` field selects that namespace; an omitted or empty `org` uses the
flat storage root, which keeps the bundled registry-mock fixtures working
without moving seeded packages. Local filesystem storage and S3/R2-compatible
storage both apply the same namespace, so two hosted mounts can publish or serve
the same `name@version` without colliding. The `org` value is validated as a
single path-safe segment before startup (no separators, traversal, leading dot,
or Windows drive prefix) because it becomes a filesystem path or object-key
component, and two hosted mounts may not declare the same `org` — they would
alias one storage namespace and break the declared-provenance isolation, so the
collision is a config error.

With the pnpr-native hosted store, the hosted mount is the only mount kind that
accepts writes. Publishing, dist-tag updates, unpublish, token policy, audit
logs, quotas, and billing all attach naturally to the hosted organization mount.
Writes route into the resolved org namespace, and the publish journal records
that org so crash recovery promotes staged packages into the same namespace.

The hosted-mount abstraction can still grow a read-only external projection in a
future implementation, but PR 12747 implements the pnpr-owned hosted store and
org namespace described above; it does not introduce a generic `HostedBackend`
plugin interface.

### Upstream mounts

An upstream mount represents **exactly one** external registry origin. It may be
public or private, and it may carry pnpr-managed credentials:

```text
GET /~npmjs/react
GET /~corp/@acme/private
```

One upstream is one URL, one credential generation, one cache namespace. It is
not a fallback chain and it has no secondary or "mirror" endpoints. If you want
to use a particular mirror of a registry, point an upstream at that mirror's
URL — that is the whole feature; the mirror operator owns the mirror's
availability, and pnpr's cache (below) absorbs transient outages of the origin.

`public: true` means the upstream is anonymous and world-readable: it cannot
also declare `auth`, `access`, or **any** custom request header — a credential
can ride in `X-Api-Key` or a cookie as easily as in `Authorization`, and a
public origin is fetched anonymously and sends none. A non-public upstream must
declare an `access` list naming which pnpr callers may use that mount; any
upstream credential stays server-side.

This is the current `~<uplink>` idea generalized into the primary origin model.
The name "uplink" can remain as a compatibility term, but architecturally it is
a registry mount.

### Router mounts

A router presents several concrete mounts behind one URL and selects exactly one
of them per package, by an explicit name pattern:

```yaml
mounts:
  main:
    router:
      routes:
        - patterns: ['@acme/*']
          source: acme
        - patterns: ['@corp/*']
          source: corp
        - patterns: ['**']
          source: npmjs
```

This is the one-URL convenience of a Verdaccio facade, made safe by being
**declarative and authoritative** rather than existence-based:

- **Small pattern language.** Router patterns are deliberately restricted to
  four statically decidable shapes: `**` for all packages, `@*/*` for all
  well-formed scoped packages, `@scope/*` for one concrete scope, and an exact
  package name such as `foo` or `@scope/foo`. Any other `*` form is a config
  error, not a literal that silently never matches.
- **One package, one route, one source.** A request resolves the package name
  by evaluating the router's routes in declared order. The first route whose
  patterns match is authoritative; later routes are not consulted even if their
  patterns would also match. A `**` catch-all is an ordinary route and should be
  listed last when it is meant to handle only otherwise-unmatched names.
- **The matched source is authoritative.** If `@acme/foo` matches the `acme`
  route and `acme` returns *not found*, the router returns not found. It does
  **not** consult `npmjs`. A private name therefore can never resolve to a
  public origin, which is the dependency-confusion vector closed by construction.
- **Unavailable is not "not found".** If the matched source is down (`5xx`,
  timeout), the router returns an **error**, never a `404`. Reporting a down
  source as "not found" would let a client's own next-configured-registry, or a
  downstream proxy, fall through to a different origin — reintroducing the same
  vector one layer out. There is nowhere for the router itself to fall through
  to; its job is to surface the source's real answer.
- **No state of its own.** A router owns no bytes, metadata cache, or tarball
  cache. It resolves to a concrete member and uses that member's namespaces,
  credentials, and policy context.
- **Writes route by pattern to a hosted source.** Publishing `@acme/foo` through
  the router matches the `acme` route and writes there because `acme` is hosted.
  Publishing a name whose route targets an upstream is rejected with a clear
  "name a hosted mount" error — a write can never land on an upstream.

Routing is deterministic from config, so the same name always resolves to the
same concrete source until an operator edits the routes. That is the only thing
a router does: pick one declared origin. It cannot merge metadata across
sources, and it cannot fall through between them.

### Router validation

Because routing is first-match-in-order, route order is load-bearing, and a
misordered router is the one way a *configuration mistake* can reintroduce the
cross-origin hazard the model otherwise forbids — silently sending a private
scope to a public origin:

```yaml
routes:
  - patterns: ['**']        # catch-all first
    source: npmjs
  - patterns: ['@acme/*']   # unreachable: @acme/* already resolves to npmjs
    source: acme
```

pnpr must therefore validate every router at config load and **refuse to start
(and fail a config reload) on an unreachable route**, rather than accept it and
serve a private name from a public source. The checks are static, because
pattern-set coverage is decidable for the router's restricted pattern language:

- **No shadowed route.** A route whose patterns are fully covered by the union of
  earlier routes can never match and is rejected. The catch-all-before-a-narrower
  route case above is the most important instance; the minimum viable check — a
  `**` catch-all that is not the last route — already catches the dangerous
  common mistake, and full earlier-shadows-later coverage detection is the
  complete form.
- **No shadowed pattern.** The same check applies per pattern, not only per
  route: a single pattern strictly covered by an earlier route's pattern is
  dead even when the rest of its route stays reachable, and is rejected by
  name. Otherwise a route like `['@secret/foo', 'plainpkg']` declared after a
  broad `@*/*` route would validate while silently sending `@secret/foo` to the
  earlier source.
- **No empty route.** A route with no patterns is rejected because it can never
  select a source.
- **No duplicate or contradictory patterns** within one router.
- **Every `source` resolves** to a defined concrete mount. A route source cannot
  be unknown, the router itself, or another router.

Validation makes a misordered router a startup error an operator sees
immediately, holding routers to the same "misconfiguration is caught, not
silent" standard as the rest of the model. An operator who genuinely wants a
redundant route removes it; pnpr never silently serves the wrong origin because
a route was placed in the wrong order.

### Default target and the path-less base

Every `~<mount>/` is already a complete npm registry root. The npm protocol
treats whatever base URL a client is configured with as the registry, so

```text
GET  <base>/foo
GET  <base>/foo/-/foo-1.0.0.tgz
```

work identically whether `<base>` is `https://<pnpr>/~main/`,
`https://<pnpr>/~acme/`, or anything else. npm does not require the registry to
live at the domain root — a path prefix is a perfectly valid registry. So mounts
are not "sub-registries" under a privileged real root; each mount *is* a root.

The only base that does not name a mount is the **path-less** one — a client
configured with `registry=https://<pnpr>/` and no mount in the path. That base
needs something to answer it, so pnpr lets an operator alias it to one named
mount via a **default target**:

```yaml
defaultTarget: main        # or: npmjs, acme, ...
```

This is purely a convenience for clients (and tools like a bare `npm publish`)
configured with the host and no mount path; it adds an address, not a privileged
registry.

Rules:

- **The path-less base is optional.** A deployment may omit `defaultTarget`
  entirely and expose only `~<mount>/` URLs; the bare host then has no registry
  and clients must address a mount. The default target exists only to give the
  path-less base a meaning when a deployment wants one.
- **The default is an alias to one named mount.** With `defaultTarget: main`,
  `<base>/foo` *is* `~main/foo`. If the default is a concrete mount, the
  path-less base serves that one origin; if it is a router, it routes by the
  router's declared patterns. Either way there is no ad-hoc blending and no
  existence-based fallback.
- **There is no implicit hosted uplink.** pnpr does not ship a magic `~hosted`
  default. Hosted orgs are explicit `~<org>` mounts. A deployment that wants the
  path-less base to be its hosted org sets `defaultTarget: acme`; `pnpr init`
  for a single-org deployment may scaffold that line into generated config, but
  it is visible config, not built-in behavior. This keeps the product model
  (organizations, not "the hosted implementation") and the multi-tenant case
  (no single default org) coherent.
- **An unqualified publish (to the path-less base) is allowed only when the
  resolved target writes to a hosted org.** A hosted-org default accepts writes;
  a router default accepts a write only if the published name's route targets a
  hosted source. An upstream default, or a router route that targets an
  upstream, rejects the publish with
  a clear "name a hosted mount" error, so a publish can never silently land in
  the wrong place.

This makes the path-less base a product choice instead of the internal
architecture. Small deployments can point clients at one base URL (typically a
router mount); standalone
deployments can point users at `~<org>` registries directly.

### Client routing and lockfiles

Composition can happen on either side, and both are declarative — never
existence-based:

- **Server-side:** a router behind one URL (above). The client configures a
  single registry; pnpr routes by pattern to one concrete source.
- **Client-side:** scoped and named registries pointing at concrete mounts. The
  client makes the routing explicit in its own config:

  ```ini
  registry=https://registry.example.com/~npmjs/
  @acme:registry=https://registry.example.com/~acme/
  @corp:registry=https://registry.example.com/~corp/
  ```

pnpr rejects any registry URL that does not map to an allowlisted mount before
any server-side fetch. For private package declarations, the most explicit form
is a named registry pointing at the concrete private mount:

```yaml
namedRegistries:
  acme: https://registry.example.com/~acme/
```

```json
{
  "dependencies": {
    "foo": "acme:foo@1.0.0"
  }
}
```

**Lockfiles are already deployment-portable — reuse that, do not rewrite URLs.**
pnpm does not persist canonical registry tarball URLs. It stores integrity and
rebuilds the URL at install time from the registry config:

```ts
// lockfile/utils/src/pkgSnapshotToResolution.ts
let registry = (name[0] === '@') ? registries[name.split('/')[0]] : ''
if (!registry) registry = registries.default
tarball = getNpmTarballUrl(name, version, { registry })   // host comes from config
```

and the writer drops any tarball URL that `isCanonicalRegistryTarballUrl`
recognizes as the canonical shape for its registry
(`resolving/tarball-url/src/index.ts`). Its docstring even anticipates a proxy
serving tarballs on a non-canonical path: rewrite the resolved tarball to
`getNpmTarballUrl(name, version, { registry })` so nothing host-specific is
persisted.

So as long as pnpr serves tarballs at the **canonical path for the registry base
the client addressed** (`https://<pnpr>/~npmjs/foo/-/foo-1.0.0.tgz` is canonical
for base `https://<pnpr>/~npmjs/`, while
`https://<pnpr>/foo/-/foo-1.0.0.tgz` is canonical for the path-less default
base), pnpm already:

- keeps the pnpr host out of the lockfile;
- reconstructs the correct per-mount tarball URL from the client's mount config;
- lets the same lockfile move between pnpr deployments by changing only the
  registry/mount base in config.

This is also how routers are implemented: a packument requested from `/~main/`
gets `dist.tarball` URLs under `/~main/`, even if the router resolved the
package to `acme` or `npmjs` internally. A packument requested from the
path-less base gets path-less tarball URLs. The tarball request routes through
the same mount graph again by package name, so the URL remains canonical for the
client's configured registry instead of baking the resolved concrete mount into
the lockfile.

This is why an earlier draft's machinery — pnpr rewriting `dist.tarball` to a
concrete mount and the client synthesizing a `namedRegistries` alias table into
`pnpm-workspace.yaml` — is unnecessary and is dropped.

**Future lockfile requirement: registry identity in package identity.**
What pnpm's reconstruction cannot express is *which* concrete mount a package
came from when scope alone is ambiguous — the same `name@version` from two
mounts (and split-within-a-scope, e.g. `@acme/foo` hosted but `@acme/bar` from
npm). For those, a pnpm/pacquet follow-up should record the registry identity of
the **concrete resolved source** in the package key so the two cannot collide.
When a package is reached through a router, that future identity should be the
concrete source it resolved to, not the router, so a later router edit cannot
silently change a locked package's origin. This is the gap described in the
[Motivation](#pnpm-cannot-represent-the-same-nameversion-from-two-registries-today),
and the prior art is vlt's DepID, which makes registry a first-class component
of package identity:

```text
registry··x@1.2.3       # default registry (empty registry component)
registry·npmjs·x@1.2.3  # the npmjs registry — a distinct key from the default
```

The default registry uses an empty component, so ordinary dependencies carry no
extra noise; only a non-default registry qualifies the key. The lockfile stores
the registry *name/identity*, not a URL; the name→URL mapping lives in config
(`namedRegistries` / mount config), and a missing entry fails closed with a
configuration error rather than silently recomputing provenance.

The exact key encoding is a design detail to settle with the pnpm lockfile
maintainers (see
[Follow-up Questions](#follow-up-questions-and-bikeshedding)),
but the invariant is fixed: registry identity is part of package identity, and
no tarball URL is persisted for canonical registry resolutions.

PR 12747 implements the pnpr-server side only. It preserves lockfile portability
by serving canonical tarball URLs for the addressed base, but it does **not**
change pnpm's or pacquet's lockfile package identity. The registry-identity
lockfile work remains a separate pnpm/pacquet follow-up.

### Serving tarballs inside pnpr

pnpr historically had two tarball-serving handlers — the normal path and the
`~<uplink>` path. The mount model unifies them: both construct a concrete
**mount origin** and call one shared, origin-aware serving routine. Cache
namespace, credential generation, integrity verification, and advisory/OSV
screening are all keyed by the resolved mount origin.

A router serves tarballs at the same canonical base that served the packument
and **internally routes** to the pattern-matched concrete source —
deterministically the same source that served the metadata, so there is no risk
of a tarball coming from a different origin than the metadata that selected it.
pnpr fetches upstream tarballs through the selected mount, verifies them against
that source's packument integrity, and caches them in the selected mount's
namespace when caching is enabled. The client never needs credentials for a
private upstream, and the lockfile never records a redirect or concrete upstream
URL for canonical registry tarballs.

A warm cache hit is served without re-reading the packument: the entry was
bound to a declared version and integrity-verified when it was written, its
namespace pins it to one declared origin, and the client re-verifies what it
receives against its lockfile — re-parsing a multi-megabyte packument per
tarball request would dominate warm serving for no additional guarantee. The
packument bind runs when new bytes are fetched (that is what it protects), and
before the cache read when OSV screening is enabled, since OSV needs the
packument-resolved version.

### Search

The npm search endpoint is not a router aggregate. PR 12747 keeps search
local-storage-only: it scans the hosted store, filters results through the
per-package access policy, and never queries upstream mounts or merges search
results across sources. This keeps search aligned with the no-merge provenance
model. Extending search across hosted org namespaces is an implementation detail;
upstream search fan-out is intentionally not part of the mount router.

### Relationship to auth-aware resolution caching

This RFC does not replace the authorization-aware resolution cache design. It
provides the cleaner origin model that design can use.

The existing auth-aware cache proposal distinguishes public routes,
pnpr-hosted private routes, and private proxied routes. Registry mounts make
those route identities explicit:

- public routes are public upstream mounts (or a router route that targets one);
- pnpr-hosted private routes are hosted organization mounts;
- private proxied routes are private upstream mounts with pnpr-managed
  credentials;
- private access descriptors are derived from the resolved concrete mount and
  its access policy or credential generation.

The same mount identity should feed resolver cache keys, metadata cache keys,
tarball cache namespaces, and the future lockfile registry identity, so there is
one origin concept across caching, serving, and client package identity.

### Authorization, cache, and policy

Authorization is checked at the resolved source and at the per-package policy
layer. The caller's pnpr identity may authorize access to a hosted organization
mount or to a private upstream mount, but it is not forwarded to third-party
registries.

**Authorize at the concrete source.** Every mount is independently addressable at
its own `~<mount>/` URL, so a private package's access policy must live on the
concrete source mount that holds it — that is the boundary a request cannot
bypass. A router or default target is *not* where private packages are protected:
the source is reachable directly at its own URL regardless of any router in front
of it. In PR 12747 routers have no access list of their own; access composes from
the resolved source mount plus the per-package `packages:` ACL layer. To make a
whole deployment internal, every reachable mount needs an access policy, not just
the router URL users are expected to use.

Worked example — private packages on a public registry (e.g. npm, which serves a
private `@myorg` scope and all public packages from one origin). Two upstream
mounts point at the same `registry.npmjs.org`: an anonymous `npm-public`
(`public: true`) and an authenticated `npm-private` (`auth.tokenEnv: NPM_TOKEN`,
`access: team:myorg`), with a router sending `@myorg/*` to `npm-private` and
`**` to `npm-public`. The `team:myorg` policy lives on `npm-private`, so both
`/~main/@myorg/secret` (via the router) and `/~npm-private/@myorg/secret`
(direct) are gated, while `lodash` stays open by either path. Putting
`team:myorg` on the router instead would not protect `@myorg/*` (already
protected at the source) and would not close `/~npm-public/`. An unauthorized
caller for a private hosted package receives `404`, not `403`, so private-package
existence is not revealed.

Any response that can vary by caller — one resolved through a private source,
or one whose per-package ACL denies anonymous access even through a public
source — carries `Cache-Control: private, no-store` and `Vary: Authorization`
on every URL surface (`/~<mount>/` unconditionally; the path-less base when the
resolution or the package ACL is caller-gated). A shared HTTP cache in front of
pnpr therefore can never replay an authenticated response to an anonymous
caller, while truly public resolutions on the path-less hot path stay
cacheable.

The Verdaccio-shaped `packages:` block no longer routes packages. It is an ACL
layer only: `access`, `publish`, and `unpublish` rules are compiled into package
policies and enforced centrally on mounted reads and writes, whether the selected
source is hosted or upstream. A `proxy:` entry in `packages:` is not part of the
mount model; package routing lives only in `mounts:` and `defaultTarget:`.

Cache keys include the concrete mount identity **and its declared origin**:

- hosted organization packuments and tarballs are cached under the organization
  mount namespace (and stored there by the default storage backend);
- public upstream metadata and tarballs use a stable, secret-free namespace keyed
  by the mount name and its upstream URL, so cache hits survive process
  restarts;
- private upstream metadata and tarballs use a secret-keyed namespace derived
  from the mount, its upstream URL, and its effective upstream headers, so
  credential/header rotation naturally moves future fetches to a new namespace;
- because the URL is part of both keys, **repointing a mount's `url:` abandons
  the previous origin's cache** — the cache is a mirror of one declared origin,
  and bytes fetched from the old origin can never answer for the new one;
- routers cache nothing of their own; metadata and tarballs belong to the
  resolved concrete source.

This cache is also pnpr's outage resilience: once a source's packument or
tarball is cached, an outage of that source does not block installs of cached
content. The stale-serving is scoped to genuine unavailability — a transport
failure, a `5xx`, an open circuit. An authoritative `4xx` from the origin (auth
revoked, `410 Gone`, throttled) is surfaced, never masked by stale cache, and a
definitive upstream `404` purges the cached packument so an unpublished package
cannot be resurrected later. Resilience is the cache's job, never a
fall-through to a different origin.

Policy checks also receive the concrete mount identity. This avoids pretending
that `foo@1.0.0` from two registries is the same package for every policy
decision. Advisory policy, tarball integrity, package access, and audit logs can
all refer to the mount that actually supplied the package.

## Rationale and Alternatives

### Keep Verdaccio-style packages and uplink fallback as the core model

This preserves compatibility with Verdaccio configuration, but it keeps the
wrong primitive at the center. Package-name rules plus existence-based fallback
chains force pnpr to infer origin after the fact, and that inference is the
dependency-confusion vector. PR 12747 makes `mounts:`/`defaultTarget:` the only
package-routing surface and keeps `packages:` as access policy only. A migration
tool can help operators rewrite a Verdaccio config into explicit mounts and
routes, but pnpr should not keep a live compatibility mode that interprets
`packages: proxy:` as a fallback chain. The matched route is authoritative —
pnpr does not merge metadata across uplinks or fall through to a public uplink
when a private one misses or is down.

### Support existence-based fallback (private first, else public)

Rejected. "Serve the private package if it exists, otherwise the public one"
infers provenance from existence, so a missing private name — or a private
origin that is briefly down — resolves to a public origin of the same name. That
is exactly the dependency-confusion attack. Declarative pattern routing covers
every legitimate case (scope or name pattern → private source) without ever
inferring; the only thing it cannot express is "a private package with an
arbitrary, unknown, public-colliding name, auto-preferred", which is precisely
the unsafe case. Integrity does not rescue this: it guards a tarball's bytes
against the pinned metadata, but not the choice of which metadata (which origin)
to resolve in the first place.

### Mirror groups / multi-endpoint upstreams for failover

Rejected. "Members are byte-equivalent, serve from any" relies on an
operator declaration pnpr cannot enforce, and a failover-on-`down` to a
secondary endpoint is a cross-origin substitution wearing a single-mount label —
the same forbidden fall-through. It is also redundant: an upstream you would
point at is already highly available behind its own URL, and pnpr's cache
absorbs transient outages. If you want a specific mirror, make it the upstream's
single URL.

### Use one blended root registry that does everything

pnpr could make the root serve hosted, public, and private content with implicit
fallback. This hides provenance and needs hidden state to answer a tarball
request safely. A router gives the same one-URL ergonomics with explicit,
declared routing and no fall-through.

### Rewrite `dist.tarball` to the concrete mount and persist it

Rejected: a persisted concrete-mount URL is non-canonical for the client's
registry base, so pnpm cannot drop it, which bakes the deployment host into every
lockfile entry and churns the lockfile on any host change. Serving at the
canonical path plus internal routing achieves the same provenance with a portable
lockfile.

### Persist the full tarball URL in the lockfile

Equivalent regression for the same reason: pnpm deliberately drops canonical
registry tarball URLs. Full URLs are correct only for genuinely non-canonical
paths, and even there the recommended approach is to rewrite to canonical so the
URL can be dropped.

### Add a generic `~hosted` registry

A `~hosted` mount would separate hosted from upstream packages, but it does not
match the product model. Operators and users think in terms of organizations,
teams, projects, or tenants, not "the hosted implementation". `~<org>` gives
hosted packages a natural ownership boundary and leaves package scopes free to
mean normal npm scopes.

### Run separate pnpr instances for each registry

Separate instances give strong isolation but remove useful composition, and
require duplicated routing, config, caches, and publish/auth surfaces for
registries that logically belong to one product. Mounts keep the isolation
boundary explicit without a process per registry.

## Implementation

PR 12747 implements the pnpr-server side as a replacement of the legacy
Verdaccio-shaped routing model.

1. Add a tagged `mounts:` config model (`type: hosted`, `type: upstream`,
   `type: router`) plus optional `defaultTarget:`. This is the only package
   routing surface. `packages:` remains as access policy (`access`, `publish`,
   `unpublish`) and does not proxy or fall through.
2. Build a validated mount graph at config load. Hosted mounts populate a hosted
   table (`org`, `access`), upstream mounts populate the runtime upstream table,
   and routers hold ordered routes to concrete sources.
3. Implement the restricted `PackagePattern` language: `**`, `@*/*`, `@scope/*`,
   and exact package names. Reject unsupported wildcard forms.
4. Validate routers statically: reject empty routes, duplicate patterns,
   shadowed/unreachable routes (including a non-last `**`), individually
   shadowed patterns inside otherwise-reachable routes, unknown sources,
   self-references, and routes whose source is another router. Validate every
   mount name as a single URL-safe path segment.
5. Route every mounted read through the mount graph. `/~<mount>/...` addresses
   that mount directly; the path-less base routes through `defaultTarget` when
   configured and returns `404` when it is not.
6. Serve `dist.tarball` URLs canonical for the registry base the client
   addressed, not the resolved concrete source. Tarball requests re-enter the
   same mount graph by package name, then fetch from the selected hosted or
   upstream source.
7. Route writes through the same graph. A write is accepted only when the
   resolved source is hosted; a route to an upstream is rejected with a clear
   "name a hosted mount" error. Publish, dist-tag, unpublish, and batch publish
   all write into the resolved hosted org namespace.
8. Namespace hosted storage by `org` for both local filesystem and S3/R2-backed
   storage. An empty `org` keeps using the flat storage root for registry-mock
   fixtures. The publish journal records the org so recovery promotes staged
   packages into the correct namespace.
9. Enforce source and package access on mount-served reads and writes. Private
   hosted mounts hide unauthorized package existence with `404`; private upstream
   credentials stay server-side.
10. Key upstream cache namespaces by mount identity and origin URL: public
    upstreams use a stable secret-free namespace over the mount name and URL,
    while private upstreams use a secret-keyed digest of the mount, its URL,
    and its effective upstream headers — so repointing a mount abandons the
    previous origin's cache. Remove shared mirror failover/conditional-validator
    behavior from the registry serving path.
11. Keep search local-storage-only. It scans the flat hosted store and filters
    results through package access policy; it does not query or merge upstream
    searches.
12. Leave pnpm/pacquet lockfile registry identity out of this PR. The server
    side keeps lockfiles portable by returning canonical tarball URLs for the
    addressed base; the package-identity format change is a separate follow-up.

Tests should cover:

- tagged `mounts:` config parsing, unknown `type:` rejection, undefined
  `defaultTarget` rejection, and `packages:` deriving ACLs without routing;
- pattern parsing for `**`, `@*/*`, `@scope/*`, and exact names, plus rejection
  of unsupported wildcard forms;
- a private route returning not-found NOT falling through to a public source;
- a private route listed before a catch-all winning by route order, with the
  catch-all never consulted for that matched package name;
- config validation rejecting a router with a shadowed/unreachable route (a
  non-last `**`, or a later route fully covered by earlier ones) and failing a
  reload that would introduce one, plus a single shadowed pattern inside an
  otherwise-reachable route, duplicate patterns, empty routes, unknown sources,
  self-references, and router-as-source;
- config validation rejecting URL-unsafe mount names and duplicate hosted
  `org` namespaces;
- a private/matched source being **down** returning an error, never a `404`;
- hosted organization mounts not falling through to upstreams;
- hosted storage namespaced by `org`, including an empty `org` mapping to the
  flat storage root and path-traversal-like org names rejected at config load;
- publish, batch publish, dist-tag, and unpublish routing to the resolved hosted
  org and rejecting upstream write targets;
- an upstream being exactly one URL with no secondary/mirror endpoint behavior;
- public upstream mounts rejecting credentials, access gates, and any custom
  request header, private upstream mounts requiring access, and private
  upstream credentials staying server-side;
- a private source's access policy enforced both through a router and via the
  source's own `~<mount>/` URL, so an open router never exposes a gated source;
- an unauthorized caller for a private hosted package receiving `404`, not
  `403`;
- tarball URLs rewritten to the addressed registry base (path-less or
  `/~<mount>/`) and tarball requests routing back through the same mount graph;
- cache namespace isolation by public mount name and private effective upstream
  headers, and a repointed upstream `url:` abandoning the previous origin's
  cached content;
- caller-gated path-less responses (private source, or a package ACL denying
  anonymous access through a public source) carrying private-cache headers
  while public resolutions stay shared-cacheable;
- stale cache serving on transient upstream failure only — an authoritative
  `4xx` surfacing rather than being masked, and a definitive `404` purging the
  cached packument;
- search remaining flat-hosted-storage-only and filtered by package access
  policy;
- router writes accepted only when the matched route targets a hosted source;
- an unqualified publish to the path-less base rejected when the resolved target
  does not write to a hosted org, and the path-less base disabled entirely when
  no `defaultTarget` is set.

## Prior Art

vlt (the originators of named registries) make registry identity a first-class
component of the dependency identifier. A vlt **DepID** for a registry package
is a typed tuple `[type, registry, name@version]`, e.g. `registry··x@1.2.3` for
the default registry and `registry·npmjs·x@1.2.3` for a named one. Registry is
part of the lockfile key, so the same `name@version` from two registries cannot
collide — exactly the gap pnpm has. pnpm adopted vlt's named-registry *syntax*
(`gh:`/`alias:`) but not its *identity model*, which is why pnpm's `name@version`
key drops the registry. The pnpm/pacquet follow-up should adopt the identity
model (registry as a key component), not vlt's serialization (which vlt itself
is reconsidering).

Nexus's **repository group** and Artifactory's **virtual repository** are the
established names for "aggregate several repos behind one URL". They are
deliberately *not* reused here, because both **merge** npm metadata across
members (Artifactory even queries every member to surface the latest version).
Merging is the source of the same-`name@version`-from-two-origins ambiguity this
RFC exists to remove, so the router is **select-one**, not merge. The closest
prior art for select-one composition is conda's **strict channel priority**,
which conda added for the same reason: stop mixing builds of one package across
sources. A router is strict-priority routing keyed by name pattern.

Verdaccio uses package rules and uplinks to let one registry facade proxy and
merge other registries with existence-based fallback. The router covers its
one-URL ergonomics, but deliberately omits the two unsafe behaviors —
cross-uplink metadata merge and existence-based fall-through — in favor of
authoritative, declared routing.

Many hosted registries expose organization, project, or tenant boundaries in the
URL. `~<org>` follows that product shape while staying compatible with npm
package names and scopes beneath the mount.

## Follow-up Questions and Bikeshedding

- PR 12747 uses `/~<mount>/` for every mount, including hosted org mounts;
  future path changes would be migrations rather than unresolved design.
- PR 12747 uses `mounts`, `type: router`, and `routes`; future renaming would
  be a migration, not an unresolved implementation choice.
- Router pattern language is fixed for PR 12747: `**`, `@*/*`, `@scope/*`, and
  exact names. Future pattern extensions must keep static coverage decidable so
  router-shadow validation remains possible. Precedence is not an open question:
  routes are evaluated in declared order and the first matching route wins.
- What exact lockfile encoding carries registry identity in package identity —
  a registry-qualified package key, a package-to-registry table, or another
  compact form — and how does it interoperate with the existing `name@version`
  depPath grammar (which today special-cases only prefixes like `runtime:`)?
- How is the registry name/identity mapped to a URL at install time — through
  `namedRegistries`, through a single `pnprServer` base, or both — so a
  deployment move updates one setting?
- Router nesting is not part of PR 12747; a route source must be a concrete
  hosted or upstream mount. Any future nesting proposal must preserve static
  validation and resolve to one concrete source before serving.
- Should there be a separate migration tool for Verdaccio configs that rewrites
  `packages: proxy:` into explicit mounts/routes, and should it warn or refuse
  configs that rely on multi-uplink merge or existence fallback?
