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

Lockfiles stay deployment-portable by reusing pnpm's existing tarball-URL
reconstruction rather than persisting pnpr URLs. The one genuinely new lockfile
requirement is recording **registry identity** in package identity, so the same
`name@version` from two different mounts cannot collide — a gap that exists in
pnpm today.

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
- the root path is a configurable default target, never the internal primitive.

## Detailed Explanation

### Registry mounts

A registry mount is an addressable npm registry surface. It has a stable mount
ID and is available at:

```text
https://<pnpr>/~<mount>/
```

Mount IDs are pnpr route names, not npm package scopes. They must be path-safe,
operator-controlled identifiers. The leading `~` keeps mount routes out of the
normal npm package-name space and lets `@scope/pkg` keep its existing meaning
under every mount.

The root path is available as a configurable default surface, but internally it
always resolves to one named mount target (see
[Default target and the root facade](#default-target-and-the-root-facade)).

Illustrative config:

```yaml
mounts:
  acme:
    hostedOrg:
      org: acme
      access: team:acme

  npmjs:
    upstream:
      url: https://registry.npmjs.org/
      public: true

  corp:
    upstream:
      url: https://npm.corp.example/
      auth:
        tokenEnv: CORP_NPM_TOKEN
      access: team:acme

  # One URL that routes each package to exactly one concrete mount by the first
  # matching explicit name pattern. No existence-based fallback.
  main:
    router:
      routes:
        - patterns: ['@acme/*']
          source: acme
        - patterns: ['@corp/*']
          source: corp
        - patterns: ['**']
          source: npmjs

defaultTarget: main
```

The syntax is illustrative. The required model is two concrete mount kinds and
one composite:

```rust
enum RegistryMount {
    HostedOrg {
        org: OrgId,
        access: AccessPolicy,
    },
    /// Exactly one external origin. Not a chain and not a set of endpoints:
    /// one URL, one credential generation, one cache namespace.
    Upstream {
        url: Url,
        auth: Option<UpstreamAuth>,
        access: AccessPolicy,
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
    source: MountId, // a HostedOrg or Upstream mount; never another Router member by existence
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

A hosted organization mount stores packages on pnpr and serves only that
organization's hosted registry:

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

Publishing, dist-tag updates, unpublish, token policy, audit logs, quotas, and
billing all naturally attach to the hosted organization mount. This is the
standalone pnpr product surface, and it is the only mount kind that accepts
writes.

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
If the mount is private, the caller must be authorized to use the mount, but the
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
pattern-set coverage is decidable for the router's glob language:

- **No shadowed route.** A route whose patterns are fully covered by the union of
  earlier routes can never match and is rejected. The catch-all-before-a-narrower
  -route case above is the most important instance; the minimum viable check — a
  `**` catch-all that is not the last route — already catches the dangerous
  common mistake, and full earlier-shadows-later coverage detection is the
  complete form.
- **No duplicate or contradictory patterns** within one router.
- **Every `source` resolves** to a defined mount, and a router does not list
  itself as a source (nor form a cycle, if nesting is ever allowed).

Validation makes a misordered router a startup error an operator sees
immediately, holding routers to the same "misconfiguration is caught, not
silent" standard as the rest of the model. An operator who genuinely wants a
redundant route removes it; pnpr never silently serves the wrong origin because
a route was placed in the wrong order.

### Default target and the root facade

The npm registry protocol expects a registry root:

```text
GET /foo
GET /foo/-/foo-1.0.0.tgz
```

pnpr keeps this root surface for compatibility, configured as a **default
target** that resolves to exactly one named mount — a concrete mount or a
router:

```yaml
defaultTarget: main        # or: npmjs, acme, ...
```

Rules:

- **The default is an alias to one named mount.** `/foo` *is* `~main/foo`. If
  the default is a concrete mount, the root serves that one origin; if it is a
  router, the root routes by the router's declared patterns. Either way there is
  no ad-hoc blending and no existence-based fallback.
- **There is no implicit hosted uplink.** pnpr does not ship a magic `~hosted`
  default. Hosted orgs are explicit `~<org>` mounts. A deployment that wants the
  root to be its hosted org sets `defaultTarget: acme`; `pnpr init` for a
  single-org deployment may scaffold that line into generated config, but it is
  visible config, not built-in behavior. This keeps the product model
  (organizations, not "the hosted implementation") and the multi-tenant case
  (no single default org) coherent.
- **Publish-to-root is allowed only when the resolved target writes to a hosted
  org.** A hosted-org default accepts writes; a router default accepts a write
  only if the published name's route targets a hosted source. An upstream
  default, or a router route that targets an upstream, rejects the publish with
  a clear "name a hosted mount" error, so a publish can never silently land in
  the wrong place.

This makes the root path a product choice instead of the internal architecture.
Small deployments can keep one root URL (typically a router); standalone
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

So as long as each mount serves tarballs at the **canonical path for its own
registry base** (`https://<pnpr>/~npmjs/foo/-/foo-1.0.0.tgz` is canonical for
base `https://<pnpr>/~npmjs/`), pnpm already:

- keeps the pnpr host out of the lockfile;
- reconstructs the correct per-mount tarball URL from the client's mount config;
- lets the same lockfile move between pnpr deployments by changing only the
  registry/mount base in config.

This is why an earlier draft's machinery — pnpr rewriting `dist.tarball` to a
concrete mount and the client synthesizing a `namedRegistries` alias table into
`pnpm-workspace.yaml` — is unnecessary and is dropped.

**The one new lockfile requirement: registry identity in package identity.**
What pnpm's reconstruction cannot express is *which* concrete mount a package
came from when scope alone is ambiguous — the same `name@version` from two
mounts (and split-within-a-scope, e.g. `@acme/foo` hosted but `@acme/bar` from
npm). For those, the lockfile must record the registry identity of the
**concrete resolved source** in the package key so the two cannot collide, and
so reconstruction targets the right base. When a package is reached through a
router, the lockfile records the concrete source it resolved to, not the router,
so a later router edit cannot silently change a locked package's origin. This is
the gap described in the
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
maintainers (see [Unresolved Questions](#unresolved-questions-and-bikeshedding)),
but the invariant is fixed: registry identity is part of package identity, and
no tarball URL is persisted for canonical registry resolutions.

### Serving tarballs inside pnpr

pnpr currently has two tarball-serving handlers — the normal path and the
`~<uplink>` path. The mount model unifies them: both construct a concrete
**mount origin** and call one shared, origin-aware serving routine. Cache
namespace, credential generation, integrity verification, and advisory/OSV
screening are all keyed by the resolved mount origin.

A router serves tarballs at its own canonical base and **internally routes** to
the pattern-matched concrete source — deterministically the same source that
served the packument, so there is no risk of a tarball coming from a different
origin than the metadata that selected it. As a bandwidth optimization, pnpr
*may* HTTP-redirect a tarball request to a public source's own CDN; the redirect
`Location` never enters the lockfile (the persisted URL stays canonical for the
mount), and redirects are only ever issued to public origins, since a client has
no credentials for a private upstream.

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
tarball cache namespaces, and the lockfile's recorded registry identity, so
there is one origin concept across caching and serving.

### Authorization, cache, and policy

Authorization is checked at the mount boundary. The caller's pnpr identity may
authorize access to a hosted organization mount or to a private upstream mount,
but it is not forwarded to third-party registries.

Cache keys include the concrete mount identity:

- hosted organization packages are stored and cached under the organization
  mount namespace;
- upstream metadata and tarballs are cached under the upstream mount namespace,
  including credential generation where credentials are configured;
- routers cache nothing of their own; metadata and tarballs belong to the
  resolved concrete source.

This cache is also pnpr's outage resilience: once a source's packument or tarball
is cached, an outage of that source does not block installs of cached content.
Resilience is the cache's job, never a fall-through to a different origin.

Policy checks also receive the concrete mount identity. This avoids pretending
that `foo@1.0.0` from two registries is the same package for every policy
decision. Advisory policy, tarball integrity, package access, and audit logs can
all refer to the mount that actually supplied the package.

## Rationale and Alternatives

### Keep Verdaccio-style packages and uplink fallback as the core model

This preserves compatibility with Verdaccio configuration, but it keeps the
wrong primitive at the center. Package-name rules plus existence-based fallback
chains force pnpr to infer origin after the fact, and that inference is the
dependency-confusion vector. pnpr can still compile Verdaccio config into mounts
(a `packages:` pattern with one uplink becomes a router route; local-first
publish becomes a hosted route ahead of an upstream route), but the matched
route is authoritative — pnpr does not merge metadata across uplinks or fall
through to a public uplink when a private one misses or is down.

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

Implementation should be staged so compatibility remains intact while the
server internals move to the mount model.

1. Add a `RegistryMount` config model (`HostedOrg`, single-URL `Upstream`,
   `Router`) and compile existing `uplinks`/`packages` config into it: each
   `packages:` pattern with one uplink becomes a router route; multi-uplink
   merge and existence fallback are not carried over.
2. Introduce an internal `MountOrigin` value that route handlers construct for
   every packument and tarball request.
3. Unify the normal tarball path and the `~<uplink>` tarball path into one
   shared origin-aware serving routine that takes a `MountOrigin`.
4. Key metadata/tarball cache namespaces and integrity/OSV screening on the
   resolved mount origin.
5. Serve every mount's tarballs at the canonical path for its own registry base,
   so pnpm's existing canonical-URL reconstruction keeps the host out of the
   lockfile. Do not rewrite or persist concrete-mount tarball URLs.
6. Implement routers: routes are evaluated in declared order; the first matching
   route selects one source; the source is authoritative on not-found and on
   unavailable (return an error, never a `404`); routers own no cache; writes
   route by pattern and are accepted only when the matched source is hosted.
   Validate routers at config load — reject shadowed/unreachable routes
   (especially a non-last `**`), duplicate patterns, and unknown/self sources —
   and fail closed on a config reload that would introduce one.
7. Teach the resolver allowlist and route classifier to map `registry` and
   `namedRegistries` URLs to mount identities and reject unknown ones.
8. Add registry identity to pnpm/pacquet lockfile package identity so the same
   `name@version` from two mounts cannot collide, recording the concrete resolved
   source's identity (not a URL, and not the router) and reconstructing the
   tarball URL from config. Keep pacquet's reader, writer, and installer in parity
   with the format.
9. Make install, frozen-lockfile verification, and tarball fetching resolve a
   recorded registry identity through config, failing closed when it is absent.
10. Add hosted organization mount storage namespaces and route publish/unpublish
    to concrete hosted organization mounts (directly or via a router route that
    targets a hosted source).
11. Wire the configurable default target / root facade, with publish-to-root
    allowed only when the resolved target writes to a hosted org.

Tests should cover:

- the same `name@version` existing with different bytes in two mounts, kept
  distinct in `packages`/`snapshots` and not colliding;
- routing producing portable lockfiles with no pnpr host persisted,
  reconstructing the right per-mount tarball URL, and recording the concrete
  source (not the router) so a later route edit cannot relocate a locked package;
- moving a lockfile between pnpr deployments with the same mount IDs without
  package-entry churn;
- a private route returning not-found NOT falling through to a public source;
- a private route listed before a catch-all winning by route order, with the
  catch-all never consulted for that matched package name;
- config validation rejecting a router with a shadowed/unreachable route (a
  non-last `**`, or a later route fully covered by earlier ones) and failing a
  reload that would introduce one, so a private scope can never be misordered
  onto a public source;
- a private/matched source being **down** returning an error, never a `404`;
- hosted organization mounts not falling through to upstreams;
- an upstream being exactly one URL with no secondary/mirror endpoint behavior;
- private upstream mounts keeping credentials server-side;
- a recorded registry identity missing from config failing closed with a clear
  error instead of falling back to the root registry;
- cache namespace isolation by mount and credential generation;
- router writes accepted only when the matched route targets a hosted source;
- publish-to-root rejected when the resolved target does not write to a hosted org.

## Prior Art

vlt (the originators of named registries) make registry identity a first-class
component of the dependency identifier. A vlt **DepID** for a registry package
is a typed tuple `[type, registry, name@version]`, e.g. `registry··x@1.2.3` for
the default registry and `registry·npmjs·x@1.2.3` for a named one. Registry is
part of the lockfile key, so the same `name@version` from two registries cannot
collide — exactly the gap pnpm has. pnpm adopted vlt's named-registry *syntax*
(`gh:`/`alias:`) but not its *identity model*, which is why pnpm's `name@version`
key drops the registry. This RFC adopts the identity model (registry as a key
component), not vlt's serialization (which vlt itself is reconsidering).

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

## Unresolved Questions and Bikeshedding

- Is `~<mount>` the right path syntax for every mount, or should hosted
  organization mounts use a different namespace?
- Should the config terms be `mounts`/`router`/`routes`, or something else
  (`registries`, `registryMounts`, ...)?
- Router pattern language: glob (`@acme/*`, `**`) vs. prefix vs. full
  parser-combinator patterns. Precedence is not an open question: routes are
  evaluated in declared order and the first matching route wins.
- What exact lockfile encoding carries registry identity in package identity —
  a registry-qualified package key, a package-to-registry table, or another
  compact form — and how does it interoperate with the existing `name@version`
  depPath grammar (which today special-cases only prefixes like `runtime:`)?
- How is the registry name/identity mapped to a URL at install time — through
  `namedRegistries`, through a single `pnprServer` base, or both — so a
  deployment move updates one setting?
- Should a router be allowed to nest (a route whose source is another router),
  and if so how is the authoritative/no-fall-through rule preserved across nesting?
- How much Verdaccio config can pnpr auto-compile into routers, and what should
  happen to configs that rely on multi-uplink merge or existence fallback (warn,
  refuse, or compile to a single authoritative route)?
