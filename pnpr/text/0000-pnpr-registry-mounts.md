# pnpr registries

## Summary

pnpr should model every origin it serves as a named **registry**. There are
exactly two concrete registry kinds — a pnpr-**hosted** organization registry
and an **upstream** registry proxying a single external origin — plus one
composite, a **router** that presents an ordered list of concrete registries
behind one URL. Every concrete registry declares the **packages** it serves —
a map from name patterns to per-package access rules — and a router
routes each package to the first listed source that claims it. Each registry
is exposed at `https://<pnpr>/~<name>/`, so every origin has an explicit
identity in the URL, in pnpr's internal routing, and in client resolution.

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

The namespace and its authorization live on the origin itself: a registry can
neither serve nor accept a publish of a name outside its declared packages, on
any path to it, and every access decision is made by the rules of the concrete
registry that serves the request. There is no global, name-keyed configuration
anywhere — routing and authorization are both derived from declared ownership.
A router orders competing claims; it never assigns a name to a registry that
does not claim it.

The pnpr-server implementation keeps lockfiles deployment-portable by serving
canonical tarball URLs for the registry base the client addressed, so pnpm can
reuse its existing tarball-URL reconstruction rather than persisting pnpr URLs.
A later pnpm/pacquet lockfile follow-up should record **registry identity** in
package identity, so the same `name@version` from two different registries
cannot collide — a gap that exists in pnpm today but is outside PR 12747.

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
  leak into client lockfiles or another registry's cache;
- most importantly, **an outage or a missing name in a private origin must never
  resolve to a public origin**. Treating "unavailable" or "not found" as "look
  elsewhere" is the dependency-confusion mechanism: an attacker who registers a
  private package's name publicly is served the moment the private origin 404s
  or is briefly down. This model forbids that fall-through entirely.

### Routing must be derived from the origin's declared namespace

An earlier iteration of this design declared name patterns on router routes
while the concrete registries themselves accepted any name. That decoupling
has two costs:

- **The hosted namespace is open.** A hosted registry accepts a publish of any
  name (gated only by an identity-flavored ACL whose default is permissive). A
  name no router routes to the registry becomes dormant stored state:
  unreachable today, authoritative the moment an operator edits routes or a
  client addresses `/~<name>/` directly. The static router validation built to
  catch "a config mistake silently sends a name to the wrong origin" cannot
  see stored state, so route edits are not as safe as the validation implies.
  A typo'd scope published to the registry's own URL also succeeds silently
  and then 404s through the router. And a private upstream registry will fetch
  arbitrary public names through its server-owned credential for any caller
  its `access:` admits.
- **Patterns are duplicated.** The names a registry is *for* and the names
  routed *to* it are the same fact, written in different places — on every
  router that includes the registry, and implicitly in what gets published to
  it — and the copies can drift.

Declaring the namespace on the registry itself closes both: the origin's
namespace is one declaration, enforced at publish and serve time on every path
to the registry, and routers derive their routing from it instead of restating
it.

### Authorization must be registry-scoped for the same reason

Earlier iterations also kept Verdaccio's top-level `packages:` block as a
**global** ACL layer: `access`/`publish`/`unpublish` rules keyed by package
name alone, applied identically whichever registry served the name. But this
design's central claim is that a package name is *not* a global identifier —
the same name from two registries is two different packages. A name-keyed
global ACL contradicts that, and it splits authorization across two places
(the registry's own `access:` plus the global table), against the
"authorize at the concrete source" rule the rest of the model follows.

Scoping the ACL to the registry reveals that it is the *same declaration* as
the namespace: one map whose keys say what the registry serves and
whose values say who may read and write it. The namespace field and the ACL
block merge into a per-registry `packages:` map, and no global, name-keyed
configuration remains.

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
because named registries are niche, but this model surfaces it routinely, so
the lockfile must carry registry identity.

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
- every registry declares the packages it serves and the access rules for
  them, and pnpr enforces both on the registry itself — an undeclared name can
  be neither published to nor served from it;
- a router can present several of these behind one URL, routing each package to
  exactly one of them by the sources' declared packages — never by guessing;
- private packages are declared (by scope/pattern or named registry), so a
  private name can never silently resolve to a public origin;
- the path-less base URL is an optional, configurable default, never a
  privileged registry — every named registry is already a registry root in its
  own right.

## Detailed Explanation

### Named registries

A pnpr registry is an addressable npm registry root. It has a stable name and
is available at:

```text
https://<pnpr>/~<name>/
```

Registry names are pnpr route names, not npm package scopes. They must be
path-safe, operator-controlled identifiers, and pnpr enforces that at config
load: a registry name must be a single URL-safe path segment (no separators,
traversal, `%`, `?`, `#`, whitespace, control characters, or Windows drive
prefixes), because it is addressed as one path segment and embedded verbatim
in rewritten `dist.tarball` URLs. A name that cannot survive that round trip
is a startup error, not an unreachable registry. The leading `~` keeps
registry routes out of the normal npm package-name space and lets `@scope/pkg`
keep its existing meaning under every registry.

The path-less base URL (no registry in the path) resolves to one named
registry via a configurable default (see
[The default registry and the path-less base](#the-default-registry-and-the-path-less-base)).

Illustrative config:

```yaml
registries:
  acme:
    type: hosted
    org: acme
    access: team:acme          # default for everything this registry serves
    packages:
      '@acme/secret':
        access: acme-admins    # the most specific matching key wins
        publish: acme-admins
      '@acme/*': {}            # served with the registry defaults

  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true
    # No packages map: serves every name — the catch-all in any router.

  corp:
    type: upstream
    url: https://npm.corp.example/
    auth:
      tokenEnv: CORP_NPM_TOKEN
    access: team:acme
    packages:
      '@corp/*': {}

  # One URL that routes each package to the first listed registry that
  # claims it. No existence-based fallback.
  main:
    type: router
    sources: [acme, corp, npmjs]

defaultRegistry: main
```

The YAML syntax is intentionally tagged by `type:` (`hosted`, `upstream`,
`router`) so a registry cannot accidentally declare more than one kind. The
required model is two concrete registry kinds and one composite:

```rust
enum Registry {
    Hosted {
        org: OrgId,
        /// Default rules for everything this registry serves.
        access: AccessPolicy,
        /// Namespace + per-package rules. Empty = every name, default rules.
        packages: PackageRules,
    },
    /// Exactly one external origin. Not a chain and not a set of endpoints:
    /// one URL, one credential generation, one cache namespace.
    Upstream {
        url: Url,
        auth: Option<UpstreamAuth>,
        access: Option<AccessPolicy>,
        cache: CachePolicy,
        /// Namespace + per-package rules. Empty = every name, default rules.
        packages: PackageRules,
    },
    /// An ordered list of concrete registries. A package resolves to the
    /// first source that claims it — authoritatively. A source's "not found"
    /// or "unavailable" is final; the router never tries another.
    Router {
        sources: Vec<RegistryName>, // Hosted or Upstream; never another Router
    },
}

/// Rules keyed by name pattern (access / publish / unpublish). The key set
/// is the registry's namespace; the most specific matching key selects the
/// rules, so entry order carries no meaning.
struct PackageRules(Vec<(PackagePattern, PackagePolicy)>);
```

A registry owns:

- its declared packages — the namespace it serves and the rules for it;
- its authorization policy for the pnpr caller;
- its upstream credential, if it has one;
- its metadata and tarball cache namespace;
- its storage namespace, if it hosts packages;
- its tarball URL base;
- its package policy and OSV/advisory policy context.

The important invariant is that a package request resolves to one concrete
origin descriptor before metadata, tarballs, cache, or advisory decisions are
made.

### The `packages:` map — namespace and rules in one declaration

Every concrete registry may declare the packages it serves as a map from name
pattern to per-package rules:

- **The key set is the namespace.** Keys are deliberately restricted to four
  statically decidable shapes: `**` for all packages, `@*/*` for all
  well-formed scoped packages, `@scope/*` for one concrete scope, and an exact
  package name such as `foo` or `@scope/foo`. Any other `*` form is a config
  error, not a literal that silently never matches. Duplicate keys within one
  registry are rejected.
- **Omitted `packages:` means every name, default rules.** A registry with no
  map serves any package under the registry-level defaults. This keeps the
  single-purpose cases free of ceremony: a pure hosted registry, or a public
  npmjs upstream acting as a router's catch-all.
- **The namespace is enforced on the registry itself, on every path to it.** A
  publish of a name outside the registry's packages is rejected; a read of one
  is a definitive `404`, answered before storage or the upstream is consulted
  — whether the request came through a router or addressed `/~<name>/`
  directly.
- **Values are the per-package rules.** Each entry may set `access`,
  `publish`, and `unpublish` permission lists (users, groups, and the built-in
  `$all`/`$authenticated`/`$anonymous`). An omitted field falls back to the
  registry-level default (`access:` on the registry), then to the safe
  defaults: `access: $all`, `publish: $authenticated` (hosted only), and
  `unpublish` denied. `publish`/`unpublish` entries on an upstream registry
  are a config error — a write can never land there.
- **The most specific entry wins — key order carries no meaning.** For any
  name, the keys that can match it form a strict specificity chain — its
  exact name, then its `@scope/*`, then `@*/*`, then `**` — and two distinct
  keys of the same tier can never match the same name, so every name has
  exactly one winning entry regardless of where it appears in the map. This
  is deliberate: YAML mappings are formally unordered, and a formatter, a
  `yq` round-trip, or a JSON conversion must not be able to change which
  access rule applies. It also means no entry can be dead — an exact key
  carves its name out of a scope key, which still serves the rest — so there
  is no shadowed-entry validation to run inside a registry; a duplicate key
  remains the only error. (Router `sources:` are different: a YAML *list* is
  syntactically ordered, so source order stays meaningful there.)

One declaration therefore does triple duty: it is the registry's **routing
claim** (what routers use to select a source), its **request filter**, and its
**access policy**. For a hosted registry the filter closes name squatting: a
name outside the declared namespace cannot be published at all, so no dormant
stored state can become authoritative when routes are later edited. For a
private upstream registry the filter stops an authorized caller from pulling
arbitrary public names through the registry's server-owned credential,
spending its quota and filling its private cache namespace with content that
belongs on a public route.

There is deliberately **no global, name-keyed ACL** beside this map — the
top-level `packages:` block of the Verdaccio model is removed, not relocated
(see [Rationale](#keep-a-global-packages-acl-beside-the-registry-namespaces)).
Because silently ignoring a previously enforced ACL would be a security
regression, a config that still contains a top-level `packages:` block is a
**startup error** naming the per-registry replacement, not an ignored key.

### Hosted organization registries

A hosted organization registry serves only that organization's packages — its
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

Hosted organization registries have no upstream fallback. A request to
`/~acme/foo` returns `foo` only if the registry's packages claim `foo`, the
`acme` organization hosts it, and the matching entry's `access` rule admits
the caller. A publish is likewise accepted only for a claimed name whose
`publish` rule admits the caller, so the stored namespace can never grow
beyond the declared one.

The implemented hosted registry is pnpr-native storage with a per-org
namespace. The YAML `org` field selects that namespace; an omitted or empty
`org` uses the flat storage root, which keeps the bundled registry-mock
fixtures working without moving seeded packages. Local filesystem storage and
S3/R2-compatible storage both apply the same namespace, so two hosted
registries can publish or serve the same `name@version` without colliding. The
`org` value is validated as a single path-safe segment before startup (no
separators, traversal, leading dot, or Windows drive prefix) because it
becomes a filesystem path or object-key component, and two hosted registries
may not declare the same `org` — they would alias one storage namespace and
break the declared-provenance isolation, so the collision is a config error.

With the pnpr-native hosted store, the hosted registry is the only kind that
accepts writes. Publishing, dist-tag updates, unpublish, token policy, audit
logs, quotas, and billing all attach naturally to the hosted organization
registry. Writes route into the resolved org namespace, and the publish
journal records that org so crash recovery promotes staged packages into the
same namespace.

The hosted-registry abstraction can still grow a read-only external projection
in a future implementation, but PR 12747 implements the pnpr-owned hosted
store and org namespace described above; it does not introduce a generic
`HostedBackend` plugin interface.

### Upstream registries

An upstream registry proxies **exactly one** external origin. It may be public
or private, and it may carry pnpr-managed credentials:

```text
GET /~npmjs/react
GET /~corp/@acme/private
```

One upstream is one URL, one credential generation, one cache namespace. It is
not a fallback chain and it has no secondary or "mirror" endpoints. If you want
to use a particular mirror of a registry, point an upstream at that mirror's
URL — that is the whole feature; the mirror operator owns the mirror's
availability, and pnpr's cache (below) absorbs transient outages of the origin.

`public: true` describes the **upstream fetch**: the origin is anonymous and
world-readable, so the registry cannot declare `auth`, a registry-level
`access:` default, or **any** custom request header — a credential can ride in
`X-Api-Key` or a cookie as easily as in `Authorization`, and a public origin
is fetched anonymously and sends none. Per-package `access` rules in the
`packages:` map remain permitted on a public upstream: they gate who may read
*through pnpr*, not how pnpr fetches from the origin. A non-public upstream
must declare a registry-level `access` list naming which pnpr callers may use
that registry; any upstream credential stays server-side.

An upstream registry's `packages:` keys bound which names may be requested
through it. This matters most for a private upstream: without a namespace
bound, any caller the registry's `access:` admits could fetch arbitrary public
package names through the registry's server-owned credential. With
`packages: {'@corp/*': {}}`, `/~corp/lodash` is a `404`, and `lodash` reaches
its declared public origin instead.

This is the earlier `~<uplink>` idea generalized into the primary origin
model. The name "uplink" can remain as a historical term, but architecturally
it is an upstream registry.

### Routers

A router presents several concrete registries behind one URL and selects
exactly one of them per package, by the sources' own declared packages:

```yaml
registries:
  main:
    type: router
    sources: [acme, corp, npmjs]
```

This is the one-URL convenience of a Verdaccio facade, made safe by being
**declarative and authoritative** rather than existence-based:

- **A router restates nothing.** It is an ordered list of concrete source
  registries; the namespaces live on the sources. A router can order competing
  claims, but it cannot assign a name to a registry that does not claim it —
  routing is derived from declared ownership, so a registry's namespace and
  its routing cannot drift apart.
- **One package, one source.** A request resolves the package name by
  evaluating the sources in declared order. The first source that claims the
  name is authoritative; later sources are not consulted even if they would
  also claim it. A source with no `packages:` map claims every name, so it
  must be listed last — anything after it is unreachable and rejected by
  validation. A name no source claims is a definitive not-found.
- **The matched source is authoritative.** If `@acme/foo` is claimed by `acme`
  and `acme` returns *not found*, the router returns not found. It does
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
- **Writes route to a hosted source.** Publishing `@acme/foo` through the
  router selects `acme` (the first source claiming the name) and writes there
  because `acme` is hosted and its matching `publish` rule admits the caller.
  Publishing a name whose selected source is an upstream is rejected with a
  clear "name a hosted registry" error — a write can never land on an
  upstream.

Routing is deterministic from config, so the same name always resolves to the
same concrete source until an operator edits the registries. That is the only
thing a router does: pick one declared origin. It cannot merge metadata across
sources, and it cannot fall through between them.

### Router validation

Because routing is first-source-in-order, source order is load-bearing, and a
misordered router is the one way a *configuration mistake* can reintroduce the
cross-origin hazard the model otherwise forbids — silently sending a private
scope to a public origin:

```yaml
sources: [npmjs, acme]   # npmjs claims every name; acme is unreachable
```

pnpr must therefore validate every router at config load and **refuse to start
(and fail a config reload) on an unreachable source**, rather than accept it and
serve a private name from a public source. The checks are static, because
pattern-set coverage is decidable for the restricted pattern language:

- **No unreachable source.** A source all of whose namespace keys are covered
  by the union of earlier sources' keys can never be selected and is rejected.
  A map-less source that is not last is the most important instance — it
  claims every name, so everything after it is dead.
- **No shadowed claim.** The same check applies per key, not only per source:
  a single namespace key of a later source strictly covered by an earlier
  source's key is dead in this router even when the rest of the source stays
  reachable, and is rejected by name. Otherwise a registry claiming
  `['@secret/foo', 'plainpkg']` listed after a registry claiming `@*/*` would
  validate while silently sending `@secret/foo` to the earlier source. Two
  registries whose namespaces overlap in both directions cannot be ordered at
  all — that is genuinely ambiguous provenance, and the operator must adjust
  the declared namespaces rather than pick an order that silently reassigns
  part of one.
- **No duplicate source** within one router, and no duplicate key within one
  registry's `packages:` map.
- **Every source resolves** to a defined concrete registry. A source cannot be
  unknown, the router itself, or another router.
- **No empty router.** A router with no sources can never serve a package.

Validation makes a misordered router a startup error an operator sees
immediately, holding routers to the same "misconfiguration is caught, not
silent" standard as the rest of the model. An operator who genuinely wants a
redundant source removes it; pnpr never silently serves the wrong origin
because a source was placed in the wrong order.

### The default registry and the path-less base

Every `~<name>/` is already a complete npm registry root. The npm protocol
treats whatever base URL a client is configured with as the registry, so

```text
GET  <base>/foo
GET  <base>/foo/-/foo-1.0.0.tgz
```

work identically whether `<base>` is `https://<pnpr>/~main/`,
`https://<pnpr>/~acme/`, or anything else. npm does not require the registry
to live at the domain root — a path prefix is a perfectly valid registry. So
pnpr's registries are not "sub-registries" under a privileged real root; each
one *is* a root.

The only base that does not name a registry is the **path-less** one — a
client configured with `registry=https://<pnpr>/` and no registry in the path.
That base needs something to answer it, so pnpr lets an operator alias it to
one named registry:

```yaml
defaultRegistry: main        # or: npmjs, acme, ...
```

This is purely a convenience for clients (and tools like a bare `npm publish`)
configured with the host and no registry path; it adds an address, not a
privileged registry.

Rules:

- **The path-less base is optional.** A deployment may omit `defaultRegistry`
  entirely and expose only `~<name>/` URLs; the bare host then has no registry
  and clients must address one by name. The default exists only to give the
  path-less base a meaning when a deployment wants one.
- **The default is an alias to one named registry.** With
  `defaultRegistry: main`, `<base>/foo` *is* `~main/foo`. If the default is a
  concrete registry, the path-less base serves that one origin; if it is a
  router, it routes by its sources' declared packages. Either way there is no
  ad-hoc blending and no existence-based fallback.
- **There is no implicit hosted registry.** pnpr does not ship a magic
  `~hosted` default. Hosted orgs are explicit `~<org>` registries. A
  deployment that wants the path-less base to be its hosted org sets
  `defaultRegistry: acme`; `pnpr init` for a single-org deployment may
  scaffold that line into generated config, but it is visible config, not
  built-in behavior. This keeps the product model (organizations, not "the
  hosted implementation") and the multi-tenant case (no single default org)
  coherent.
- **An unqualified publish (to the path-less base) is allowed only when the
  resolved target writes to a hosted org.** A hosted-org default accepts
  writes; a router default accepts a write only if the published name is
  claimed by a hosted source. An upstream default, or a router selection of an
  upstream, rejects the publish with a clear "name a hosted registry" error,
  so a publish can never silently land in the wrong place.

This makes the path-less base a product choice instead of the internal
architecture. Small deployments can point clients at one base URL (typically a
router); standalone deployments can point users at `~<org>` registries
directly.

### Client routing and lockfiles

Composition can happen on either side, and both are declarative — never
existence-based:

- **Server-side:** a router behind one URL (above). The client configures a
  single registry; pnpr routes by declared claim to one concrete source.
- **Client-side:** scoped and named registries pointing at concrete pnpr
  registries. The client makes the routing explicit in its own config:

  ```ini
  registry=https://registry.example.com/~npmjs/
  @acme:registry=https://registry.example.com/~acme/
  @corp:registry=https://registry.example.com/~corp/
  ```

pnpr rejects any registry URL that does not map to one of its declared
registries before any server-side fetch. For private package declarations, the
most explicit form is a named registry pointing at the concrete private one:

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
- reconstructs the correct per-registry tarball URL from the client's registry
  config;
- lets the same lockfile move between pnpr deployments by changing only the
  registry base in config.

This is also how routers are implemented: a packument requested from `/~main/`
gets `dist.tarball` URLs under `/~main/`, even if the router resolved the
package to `acme` or `npmjs` internally. A packument requested from the
path-less base gets path-less tarball URLs. The tarball request routes through
the same registry graph again by package name, so the URL remains canonical
for the client's configured registry instead of baking the resolved concrete
source into the lockfile.

This is why an earlier draft's machinery — pnpr rewriting `dist.tarball` to a
concrete source registry and the client synthesizing a `namedRegistries` alias
table into `pnpm-workspace.yaml` — is unnecessary and is dropped.

**Future lockfile requirement: registry identity in package identity.**
What pnpm's reconstruction cannot express is *which* concrete registry a
package came from when scope alone is ambiguous — the same `name@version` from
two registries (and split-within-a-scope, e.g. `@acme/foo` hosted but
`@acme/bar` from npm). For those, a pnpm/pacquet follow-up should record the
registry identity of the **concrete resolved source** in the package key so
the two cannot collide. When a package is reached through a router, that
future identity should be the concrete source it resolved to, not the router,
so a later router edit cannot silently change a locked package's origin. This
is the gap described in the
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
(`namedRegistries` / pnpr registry config), and a missing entry fails closed
with a configuration error rather than silently recomputing provenance.

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
`~<uplink>` path. The registry model unifies them: both construct a concrete
**origin** and call one shared, origin-aware serving routine. Cache namespace,
credential generation, integrity verification, and advisory/OSV screening are
all keyed by the resolved origin.

A router serves tarballs at the same canonical base that served the packument
and **internally routes** to the claimed concrete source — deterministically
the same source that served the metadata, so there is no risk of a tarball
coming from a different origin than the metadata that selected it. pnpr
fetches upstream tarballs through the selected registry, verifies them against
that source's packument integrity, and caches them in the selected registry's
namespace when caching is enabled. The client never needs credentials for a
private upstream, and the lockfile never records a redirect or concrete
upstream URL for canonical registry tarballs.

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
local-storage-only: it scans the hosted store, filters results through each
registry's package rules, and never queries upstream registries or merges
search results across sources. This keeps search aligned with the no-merge
provenance model. Extending search across hosted org namespaces is an
implementation detail; upstream search fan-out is intentionally not part of
the router.

### Relationship to auth-aware resolution caching

This RFC does not replace the authorization-aware resolution cache design. It
provides the cleaner origin model that design can use.

The existing auth-aware cache proposal distinguishes public routes,
pnpr-hosted private routes, and private proxied routes. Named registries make
those route identities explicit:

- public routes are public upstream registries (or a router source that is
  one);
- pnpr-hosted private routes are hosted organization registries;
- private proxied routes are private upstream registries with pnpr-managed
  credentials;
- private access descriptors are derived from the resolved concrete registry
  and its package rules or credential generation.

The same registry identity should feed resolver cache keys, metadata cache
keys, tarball cache namespaces, and the future lockfile registry identity, so
there is one origin concept across caching, serving, and client package
identity.

### Authorization, cache, and policy

Authorization lives entirely on the resolved concrete registry: its
registry-level defaults and the per-package rules in its `packages:` map. The
caller's pnpr identity may authorize access to a hosted organization registry
or to a private upstream registry, but it is not forwarded to third-party
origins.

**Authorize at the concrete source.** Every registry is independently
addressable at its own `~<name>/` URL, so a private package's access rules
must live on the concrete source registry that holds it — that is the boundary
a request cannot bypass. A router or the default registry is *not* where
private packages are protected: the source is reachable directly at its own
URL regardless of any router in front of it. Routers have no rules of their
own; a request is authorized by the resolved source's rules alone. To make a
whole deployment internal, every reachable registry needs its rules set, not
just the router URL users are expected to use.

Worked example — private packages on a public registry (e.g. npm, which serves
a private `@myorg` scope and all public packages from one origin). Two
upstream registries point at the same `registry.npmjs.org`: an authenticated
`npm-private` (`auth.tokenEnv: NPM_TOKEN`, `access: team:myorg`,
`packages: {'@myorg/*': {}}`) and an anonymous, map-less `npm-public`
(`public: true`), with a router `sources: [npm-private, npm-public]`. The
`team:myorg` rule lives on `npm-private`, so both `/~main/@myorg/secret`
(via the router) and `/~npm-private/@myorg/secret` (direct) are gated, while
`lodash` stays open by either path — and `npm-private`'s namespace bound means
`/~npm-private/lodash` is a `404` rather than a public fetch spent through the
private credential. Putting `team:myorg` on the router instead would not
protect `@myorg/*` (already protected at the source) and would not close
`/~npm-public/`. An unauthorized caller for a private hosted package receives
`404`, not `403`, so private-package existence is not revealed.

Any response that can vary by caller — one resolved through a private source,
or one whose package rule denies anonymous access even through a public
source — carries `Cache-Control: private, no-store` and `Vary: Authorization`
on every URL surface (`/~<name>/` unconditionally; the path-less base when the
resolution or the package rule is caller-gated). A shared HTTP cache in front
of pnpr therefore can never replay an authenticated response to an anonymous
caller, while truly public resolutions on the path-less hot path stay
cacheable.

The Verdaccio-shaped top-level `packages:` block is **gone entirely** — not
relocated, removed. Its two jobs are absorbed by the registries: routing lives
in the namespace keys, and `access`/`publish`/`unpublish` rules live in the
values, scoped to the one registry that serves the name. A `proxy:` entry has
no equivalent at all; package routing lives only in `registries:` and
`defaultRegistry:`.

Cache keys include the concrete registry identity **and its declared origin**:

- hosted organization packuments and tarballs are cached under the organization
  registry's namespace (and stored there by the default storage backend);
- public upstream metadata and tarballs use a stable, secret-free namespace
  keyed by the registry name and its origin URL, so cache hits survive process
  restarts;
- private upstream metadata and tarballs use a secret-keyed namespace derived
  from the registry, its origin URL, and its effective upstream headers, so
  credential/header rotation naturally moves future fetches to a new namespace;
- because the URL is part of both keys, **repointing an upstream registry's
  `url:` abandons the previous origin's cache** — the cache is a mirror of one
  declared origin, and bytes fetched from the old origin can never answer for
  the new one;
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

Policy checks also receive the concrete registry identity. This avoids
pretending that `foo@1.0.0` from two registries is the same package for every
policy decision. Advisory policy, tarball integrity, package access, and audit
logs can all refer to the registry that actually supplied the package.

## Rationale and Alternatives

### Keep Verdaccio-style packages and uplink fallback as the core model

This preserves compatibility with Verdaccio configuration, but it keeps the
wrong primitive at the center. Package-name rules plus existence-based fallback
chains force pnpr to infer origin after the fact, and that inference is the
dependency-confusion vector. This model makes the declared registries the only
package-routing surface. A migration tool can help operators rewrite a
Verdaccio config into explicit registries, but pnpr should not keep a live
compatibility mode that interprets `packages: proxy:` as a fallback chain. The
matched source is authoritative — pnpr does not merge metadata across uplinks
or fall through to a public uplink when a private one misses or is down.

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
secondary endpoint is a cross-origin substitution wearing a single-registry
label — the same forbidden fall-through. It is also redundant: an upstream you
would point at is already highly available behind its own URL, and pnpr's
cache absorbs transient outages. If you want a specific mirror, make it the
upstream's single URL.

### Use one blended root registry that does everything

pnpr could make the root serve hosted, public, and private content with implicit
fallback. This hides provenance and needs hidden state to answer a tarball
request safely. A router gives the same one-URL ergonomics with explicit,
declared routing and no fall-through.

### Declare routing patterns on router routes instead of on the registries

The first iteration of this design (implemented by pnpm/pnpm#12747) attached
patterns to router routes (`routes: [{patterns, source}, ...]`) and left the
concrete registries namespace-less. Rejected on two grounds.

First, duplication: the names a registry is for must be restated by every
router that includes it, and the restatements can drift.

Second, and more important, the registry's own surface was unconstrained. A
hosted registry accepted a publish of any name, so names never routed to it
accumulated as dormant stored state that a later route edit — or a client
addressing `/~<name>/` directly — would surface as authoritative; a typo'd
scope published to the registry URL succeeded silently and then 404'd through
the router; and route-level patterns could not stop an authorized caller from
pulling arbitrary public names through a private upstream's server-owned
credential.

Moving the namespace onto the registry makes it one enforced declaration and
reduces the router to the one fact that is genuinely per-router: precedence
order. The expressiveness lost is per-router narrowing — two routers can no
longer expose different slices of the same source. The identity dimension of
that use case is covered by per-package `access` rules; where distinct slices
of one origin are truly needed, two upstream registries over the same URL
express it. Per-source overrides could be added later if a concrete need
appears.

### Keep a global `packages:` ACL beside the registry namespaces

The second iteration declared the namespace on each registry (as a separate
`patterns:` list) but kept Verdaccio's top-level `packages:` block as a
global ACL layer. Rejected: a name-keyed global ACL contradicts the model's
central claim that package names are not global identifiers, and it splits
authorization across two places when the rest of the design insists on
authorizing at the concrete source. Worse, the two surfaces used different
pattern languages — full globs in the ACL, the restricted decidable language
in the namespaces — so the same-looking rule meant different things in each.

Scoping the ACL to the registry showed the two declarations were one: the
namespace is the key set of the rules map. Merging them into a per-registry
`packages:` map removed a top-level concept, a second pattern language, and
the interim `patterns:` field name in one move. The costs are accepted
deliberately: per-package rule keys are now limited to the restricted pattern
language (a within-name prefix shape such as `@acme/util-*` could be added
later — prefix coverage stays statically decidable), and a blanket
cross-registry rule ("deny `left-pad` everywhere") must be repeated on each
registry that could serve the name — a job that belongs to the advisory/OSV
policy layer anyway.

### Keep the `mounts:` name from PR 12747

PR 12747 shipped the config block as `mounts:` (with `defaultTarget:`), naming
the unit for its URL attachment at `/~<name>/`. Renamed to `registries:` /
`defaultRegistry:` here. Every explanation of a mount began "a mount is a full
npm registry at…" — when every definition of X starts with "X is really a Y",
the name should be Y. The rename also aligns the server with the client's
`namedRegistries` (names mapping to registry definitions on one side and
registry URLs on the other), and `defaultRegistry` is instantly legible
because it is npm's own concept. The earlier blocker — a top-level `registry:`
surface toggle as a sibling key — was removed separately (pnpm/pnpm#12767),
so the name is free. What "mount" bought was greppability and the filesystem
metaphor; both serve pnpr's developers rather than its operators. The
vocabulary this RFC uses instead is: **registry** — a named surface pnpr
serves; **origin** — the external URL an upstream registry fetches from.

### Run separate pnpr instances for each registry

Separate instances give strong isolation but remove useful composition, and
require duplicated routing, config, caches, and publish/auth surfaces for
registries that logically belong to one product. Named registries in one
server keep the isolation boundary explicit without a process per registry.

## Implementation

PR 12747 implemented this model as a replacement of the legacy
Verdaccio-shaped routing model, in its first-iteration shape: `mounts:` with
route-level patterns on routers, namespace-less concrete mounts, and the
top-level `packages:` ACL kept as a global layer. This revision replaces that
shape outright — pnpr is pre-1.0, no compatibility mode — with:

- the per-registry `packages:` map (namespace keys + per-package rules),
  absorbing both the route-level patterns and the global ACL;
- routers as ordered `sources:` lists;
- the rename `mounts:` → `registries:`, `defaultTarget:` → `defaultRegistry:`
  (see Rationale).

1. Add a tagged `registries:` config model (`type: hosted`, `type: upstream`,
   `type: router`) plus optional `defaultRegistry:`. This is the only package
   routing surface. Hosted and upstream registries take an optional
   `packages:` map from name pattern to `access`/`publish`/`unpublish` rules
   (omitted map = every name, default rules; the most specific matching key
   selects the rules; omitted rule fields fall back to the registry-level
   defaults, then the safe defaults). A router is an ordered `sources:` list
   of concrete registry names.
2. Remove the top-level `packages:` block. A config that still contains one is
   a startup error naming the per-registry replacement — silently ignoring a
   previously enforced ACL would be a security regression, so the key must not
   be dropped like an unknown Verdaccio field.
3. Implement the restricted `PackagePattern` language for the map keys: `**`,
   `@*/*`, `@scope/*`, and exact package names. Reject unsupported wildcard
   forms and duplicate keys within one registry. Reject `publish`/`unpublish`
   rules on upstream registries.
4. Build a validated registry graph at config load. Hosted registries populate
   a hosted table (`org`, defaults, package rules), upstream registries
   populate the runtime upstream table with theirs, and routers hold ordered
   source lists.
5. Validate routers statically: reject empty routers, duplicate sources,
   unreachable sources (all namespace keys covered by earlier sources' keys,
   including a non-last map-less source), individually shadowed keys of
   otherwise-reachable sources, unknown sources, self-references, and sources
   that are another router. Validate every registry name as a single URL-safe
   path segment. (Within one registry no order validation exists: rule
   selection is by specificity, so no entry can be dead.)
6. Route every read through the registry graph, and enforce the resolved
   registry's `packages:` at the registry itself: an unclaimed name is a
   definitive `404` on reads and a rejection on writes, before storage or the
   upstream is consulted, on the direct `/~<name>/...` address and through any
   router alike; a claimed name is then gated by the matching entry's rules.
   The path-less base routes through `defaultRegistry` when configured and
   returns `404` when it is not.
7. Serve `dist.tarball` URLs canonical for the registry base the client
   addressed, not the resolved concrete source. Tarball requests re-enter the
   same registry graph by package name, then fetch from the selected hosted or
   upstream source.
8. Route writes through the same graph. A write is accepted only when the
   resolved source is hosted, its packages claim the name, and the matching
   `publish`/`unpublish` rule admits the caller; a selection of an upstream is
   rejected with a clear "name a hosted registry" error. Publish, dist-tag,
   unpublish, and batch publish all write into the resolved hosted org
   namespace.
9. Namespace hosted storage by `org` for both local filesystem and S3/R2-backed
   storage. An empty `org` keeps using the flat storage root for registry-mock
   fixtures. The publish journal records the org so recovery promotes staged
   packages into the correct namespace.
10. Enforce the per-registry rules on served reads and writes. Private hosted
    registries hide unauthorized package existence with `404`; private
    upstream credentials stay server-side.
11. Key upstream cache namespaces by registry identity and origin URL: public
    upstreams use a stable secret-free namespace over the registry name and
    URL, while private upstreams use a secret-keyed digest of the registry,
    its URL, and its effective upstream headers — so repointing a registry
    abandons the previous origin's cache. Remove shared mirror
    failover/conditional-validator behavior from the registry serving path.
12. Keep search local-storage-only. It scans the flat hosted store and filters
    results through the serving registry's package rules; it does not query or
    merge upstream searches.
13. Leave pnpm/pacquet lockfile registry identity out of this PR. The server
    side keeps lockfiles portable by returning canonical tarball URLs for the
    addressed base; the package-identity format change is a separate follow-up.

Tests should cover:

- tagged `registries:` config parsing (including per-registry `packages:` maps
  and router `sources:`), unknown `type:` rejection, and undefined
  `defaultRegistry` rejection;
- a present top-level `packages:` block failing startup with an error naming
  the per-registry replacement;
- pattern parsing for `**`, `@*/*`, `@scope/*`, and exact names, plus rejection
  of unsupported wildcard forms and duplicate keys within one registry;
- `publish`/`unpublish` rules on an upstream registry rejected at config load;
- a registry's namespace enforced at the registry itself: an off-namespace
  publish rejected and an off-namespace read a `404` before storage or the
  upstream is consulted, both through a router and at the registry's own
  `/~<name>/` URL;
- per-package rule selection: the most specific matching key wins (exact over
  `@scope/*` over `@*/*` over `**`) regardless of key order — including after
  a key-reordering YAML round-trip; omitted rule fields falling back to
  registry-level defaults, then the safe defaults (`access: $all`,
  `publish: $authenticated`, `unpublish` denied);
- a per-package `access` rule on a **public** upstream gating anonymous reads
  through pnpr while the upstream fetch itself stays anonymous;
- a private upstream's namespace preventing a public name from being fetched
  through its server-owned credential;
- a private source returning not-found NOT falling through to a public source;
- a private source listed before a map-less catch-all winning by source order,
  with the catch-all never consulted for that claimed package name;
- config validation rejecting a router with an unreachable source (a non-last
  map-less source, or a source whose namespace keys are fully covered by
  earlier sources) and failing a reload that would introduce one, plus a
  single shadowed key of an otherwise-reachable source, duplicate sources,
  empty routers, unknown sources, self-references, and router-as-source;
- config validation rejecting URL-unsafe registry names and duplicate hosted
  `org` namespaces;
- a private/matched source being **down** returning an error, never a `404`;
- hosted organization registries not falling through to upstreams;
- hosted storage namespaced by `org`, including an empty `org` mapping to the
  flat storage root and path-traversal-like org names rejected at config load;
- publish, batch publish, dist-tag, and unpublish routing to the resolved hosted
  org, gated by the matching entry's `publish`/`unpublish` rules, and rejecting
  upstream write targets;
- an upstream being exactly one URL with no secondary/mirror endpoint behavior;
- public upstream registries rejecting credentials, a registry-level `access:`
  default, and any custom request header, private upstream registries
  requiring one, and private upstream credentials staying server-side;
- a private source's rules enforced both through a router and via the source's
  own `~<name>/` URL, so an open router never exposes a gated source;
- an unauthorized caller for a private hosted package receiving `404`, not
  `403`;
- tarball URLs rewritten to the addressed registry base (path-less or
  `/~<name>/`) and tarball requests routing back through the same registry
  graph;
- cache namespace isolation by public registry name and private effective
  upstream headers, and a repointed upstream `url:` abandoning the previous
  origin's cached content;
- caller-gated path-less responses (private source, or a package rule denying
  anonymous access through a public source) carrying private-cache headers
  while public resolutions stay shared-cacheable;
- stale cache serving on transient upstream failure only — an authoritative
  `4xx` surfacing rather than being masked, and a definitive `404` purging the
  cached packument;
- search remaining flat-hosted-storage-only and filtered by the serving
  registry's package rules;
- router writes accepted only when the selected source is hosted, claims the
  name, and its matching rule admits the caller;
- an unqualified publish to the path-less base rejected when the resolved target
  does not write to a hosted org, and the path-less base disabled entirely when
  no `defaultRegistry` is set.

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

Most-specific-match selection (the `packages:` map rule) is the established
choice wherever a restricted pattern language makes it decidable. npm's own
client-side registry selection is the nearest precedent: `@scope:registry`
beats the default registry, order-free — pnpr's map selection mirrors the
other half of the same system. Go 1.22's `net/http` ServeMux uses "the most
specific pattern wins" and makes registering two patterns where neither is
more specific a registration-time error — the same pair of rules as this map
(specificity plus duplicate-claim errors). IP longest-prefix routing, DNS
wildcard precedence (RFC 4592), and the Servlet URL-mapping rules are the
same shape. nginx marks the boundary condition: its prefix locations match
longest-first, order-free, while its regex locations fall back to declaration
order because regex overlap is undecidable — a restricted language is what
makes order-free selection possible. Systems that genuinely need declaration
order (`.gitignore`, ESLint overrides, iptables, `match` arms) need it
because they support negation or incomparable overlap, both of which this
pattern language deliberately omits.

Verdaccio uses package rules and uplinks to let one registry facade proxy and
merge other registries with existence-based fallback. The router covers its
one-URL ergonomics, but deliberately omits the two unsafe behaviors —
cross-uplink metadata merge and existence-based fall-through — in favor of
authoritative, declared routing. Verdaccio's `packages:` block survives here
only in spirit: its rule shape (`access`/`publish`/`unpublish` lists) moves
into each registry's `packages:` map, scoped to the one origin that serves the
name instead of keyed globally.

Many hosted registries expose organization, project, or tenant boundaries in the
URL. `~<org>` follows that product shape while staying compatible with npm
package names and scopes beneath the registry path.

## Follow-up Questions and Bikeshedding

- PR 12747 uses `/~<name>/` for every registry, including hosted org
  registries; future path changes would be migrations rather than unresolved
  design.
- This revision uses `registries:`, `type: router`, `sources`, per-registry
  `packages:` maps, and `defaultRegistry:` (replacing PR 12747's `mounts:` /
  `routes:` / `defaultTarget:` and the top-level `packages:` ACL); future
  renaming would be a migration, not an unresolved implementation choice.
- The pattern language is fixed: `**`, `@*/*`, `@scope/*`, and exact names.
  Future extensions — a within-name prefix shape such as `@acme/util-*` is the
  most likely candidate — must keep static coverage decidable so shadow
  validation (across router sources and within one registry's rules) remains
  possible. Precedence is not an open question: sources are evaluated in
  declared order and the first source that claims the name wins.
- Within one registry, the `packages:` map already selects by specificity — a
  YAML mapping has no spec-guaranteed order to depend on. Should **router
  source** selection eventually be specificity-based too (an exact claim
  beats `@scope/*` beats `@*/*` beats a catch-all), making `sources:` an
  unordered set? The same totality argument applies across the sources of one
  router — identical claims are a validation error either way — but
  `sources:` is a syntactically ordered *list*, so declared order is
  available and is kept in this revision, with unreachable-source and
  shadowed-claim validation catching misordering.
- What exact lockfile encoding carries registry identity in package identity —
  a registry-qualified package key, a package-to-registry table, or another
  compact form — and how does it interoperate with the existing `name@version`
  depPath grammar (which today special-cases only prefixes like `runtime:`)?
- How is the registry name/identity mapped to a URL at install time — through
  `namedRegistries`, through a single `pnprServer` base, or both — so a
  deployment move updates one setting?
- Router nesting is out of scope; a router source must be a concrete hosted or
  upstream registry. Any future nesting proposal must preserve static
  validation and resolve to one concrete source before serving.
- Should there be a separate migration tool for Verdaccio configs that rewrites
  `packages: proxy:` into explicit registries, and should it warn or refuse
  configs that rely on multi-uplink merge or existence fallback?
