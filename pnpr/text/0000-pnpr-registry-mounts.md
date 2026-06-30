# Registry mounts for pnpr

## Summary

pnpr should model every addressable registry origin as a **registry mount**. A
mount owns one clear origin policy: a pnpr-hosted organization registry, an
upstream registry, or a **mirror group** of byte-equivalent members. pnpr can
also expose automatically derived **blended compatibility endpoints** that
overlay a private mount on a public fallback. Named mounts are exposed at
`https://<pnpr>/~<mount>/`, so every origin has an explicit identity in the URL,
in pnpr's internal routing, and in client resolution.

The default model is **strict**: a package is served from exactly one concrete
mount, addressed explicitly. Cross-origin composition is opt-in and split into
two clearly different modes so the dangerous one (same name from two origins
with different bytes) is never the default and is always obvious in config.
Future-proof private package declarations should use a named registry that
points at the strict `~<mount>` URL. Blended endpoints exist for backward
compatibility with single-registry clients that expect private-first,
public-fallback behavior.

Lockfiles stay deployment-portable by reusing pnpm's existing tarball-URL
reconstruction rather than persisting pnpr URLs. The one genuinely new lockfile
requirement is recording **registry identity** in package identity, so the same
`name@version` from two different mounts cannot collide — a gap that exists in
pnpm today.

## Motivation

pnpr currently inherits a Verdaccio-shaped mental model: the server is one
registry facade, package rules map names to uplinks, and an uplink list can be
used as a fallback chain. That model made sense when clients generally had one
registry URL and the proxy had to decide where every package came from by
looking only at the package name.

pnpm now has named registries and pnpr already has origin-qualified
`/~<uplink>/` endpoints for private proxied tarballs. That changes the design
space. The client, the resolver, and the server can all carry a registry
identity explicitly instead of rediscovering it by walking a fallback chain.

Fallback chains are a poor *default* abstraction across different security and
provenance domains:

- the same `name@version` can exist in two registries with different bytes,
  metadata, policies, or vulnerability posture;
- a tarball request that re-runs fallback can pick a different origin from the
  packument request that selected the version and `dist.integrity`;
- shared proxy caches need extra safeguards to avoid mixing private and public
  content for the same package name;
- server-owned upstream credentials need a stable route identity so they never
  leak into client lockfiles or another mount's cache.

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
- a mirror group can serve byte-equivalent mirrors of one origin;
- private packages are declared through named registries that point at strict
  private mounts;
- a deployment can expose derived `/~~<mount>` blended endpoints for
  Verdaccio/bit.cloud-style backward compatibility, without making blended
  fallback the recommended model;
- the root path is a configurable default target, never the internal primitive.

## Detailed Explanation

### Registry mounts

A registry mount is an addressable npm registry surface. It has a stable mount
ID and is available at its strict endpoint:

```text
https://<pnpr>/~<mount>/
```

When a public fallback is configured, pnpr may also expose a derived blended
compatibility endpoint for private mounts:

```text
https://<pnpr>/~~<mount>/
```

The double-tilde endpoint is not a second storage/cache namespace. It is a
facade over the strict private mount plus the configured public fallback.

Mount IDs are pnpr route names, not npm package scopes. They must be path-safe,
operator-controlled identifiers. The leading `~` keeps mount routes out of the
normal npm package-name space and lets `@scope/pkg` keep its existing meaning
under every mount.

The root path is available as a configurable default surface, but internally it
always resolves to one named mount target (see
[Default mount and the root facade](#default-mount-and-the-root-facade)).

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

  # Byte-equivalent mirrors of the same origin. Any member may serve a
  # tarball, because every member is declared to hold identical bytes.
  npm-mirrors:
    mirrorGroup:
      members:
        - npmjs
        - npmjs-backup

defaultTarget: npmjs
```

The syntax is illustrative. The required model keeps the two grouping modes as
distinct types so the dangerous one is never reachable by accident:

```rust
enum RegistryMount {
    HostedOrg {
        org: OrgId,
        access: AccessPolicy,
    },
    Upstream {
        url: Url,
        auth: Option<UpstreamAuth>,
        access: AccessPolicy,
        cache: CachePolicy,
    },
    /// Members are declared byte-equivalent for every `name@version`.
    /// Selecting any member is safe; tarballs may be served from any of them.
    MirrorGroup {
        members: Vec<MountId>,
    },
    /// Derived Verdaccio-style facade: serve a private mount first, then fall
    /// back to a public mount. Members are NOT byte-equivalent, so the first
    /// match shadows later ones. Integrity pinning is the provenance backstop.
    BlendedEndpoint {
        private: MountId,
        fallback: MountId,
    },
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

An upstream mount represents one external registry origin. It may be public or
private, and it may carry pnpr-managed credentials:

```text
GET /~npmjs/react
GET /~corp/@acme/private
```

An upstream mount is not a fallback chain. It has one upstream base URL, one
credential generation, and one cache namespace. If the mount is private, the
caller must be authorized to use the mount, but the upstream credential stays
server-side.

This is the current `~<uplink>` idea generalized into the primary origin model.
The name "uplink" can remain as a compatibility term, but architecturally it is
a registry mount.

### Mirror groups

A mirror group composes members that are **declared byte-equivalent** for every
`name@version` — the same origin behind several hosts, or a primary plus
backups:

```yaml
mounts:
  npm-mirrors:
    mirrorGroup:
      members:
        - npmjs
        - npmjs-backup
```

Because the operator declares the members equivalent, there is no registry
identity distinction inside the group: a packument may be assembled from
whichever member answers first, and a tarball may be served from any member,
since every member is asserted to return identical bytes that satisfy the same
`dist.integrity`. pnpr cannot guarantee that third-party mirrors actually remain
equivalent; it only guarantees that mirror groups operate according to that
declaration and still verify integrity when integrity is available. A mirror
group is its own type so this operator responsibility is explicit and never
confused with a general fallback.

A mirror group is read-only. It does not own bytes; it forwards to its members.
Its tarball URL base is the group's own `~<mount>` path, and the bytes are
reconstructed from the group registry (so the canonical-URL reconstruction in
[Client routing and lockfiles](#client-routing-and-lockfiles) still applies).

### Blended compatibility endpoints

Many proxy registries (Verdaccio, bit.cloud, GitHub Packages) let one URL host
private packages *and* fall back to public npm. This is a genuinely useful UX
and pnpr supports it — but as a derived compatibility endpoint whose risks are
named, not as the recommended future-proof package declaration model.

The recommended model for new private packages is strict routing: publish to a
private mount such as `/~acme/`, configure a named registry for that mount, and
declare private dependencies with that named registry in `package.json`. Blended
endpoints exist for legacy clients that still expect one registry URL to serve
private packages first and public npm second.

When a public fallback mount is configured, pnpr may derive a blended endpoint
for each private mount:

```yaml
mounts:
  acme:
    hostedOrg:
      org: acme

  npmjs:
    upstream:
      url: https://registry.npmjs.org/
      public: true

publicFallback: npmjs

# Derived by pnpr:
# /~acme/  => strict hosted organization registry
# /~~acme/ => acme first, then npmjs
```

Semantics:

- **First match shadows.** For `GET /~~acme/foo`, pnpr serves `~acme/foo` if it
  exists, otherwise `~npmjs/foo`. The members are not byte-equivalent, so the
  unselected origin is unreachable through this endpoint for that name. This is
  the intended "private overrides public" behavior, and it is also a
  dependency-confusion surface: whoever can publish to the private mount can
  shadow a public name. That is gated by publish authorization and must be
  documented, not silent.
- **Integrity is the provenance backstop.** The packument pins the selected
  origin's `dist.integrity`. If membership changes between the packument fetch
  and the tarball fetch and pnpr routes to a different origin, the client's
  integrity check fails closed — a loud, recoverable error, never silently-wrong
  bytes. This is the same safety level npm and Verdaccio already operate at.
- **The fallback member must be public.** See [Serving tarballs in blended
  mode](#serving-tarballs-in-blended-mode): blended mode relies on serving or
  redirecting tarballs at a canonical path, and a client cannot follow a
  redirect into a private upstream because the upstream credential is
  server-side. The private member is served by pnpr directly when it needs
  server-side credentials; the fallback member must be a public mount.
- **No separate state.** `/~~acme/` does not own package bytes, metadata cache, or
  tarball cache. It routes to the selected concrete member and uses that member's
  namespaces and policy context.
- **Writes follow the private member.** Publishing through a blended endpoint is
  allowed only if the private member is a hosted organization mount. If the
  private member is a private upstream, publish is rejected with the same clear
  "name a hosted mount" error as any other non-hosted target.

Blended mode is the only place cross-origin (non-equivalent) selection happens.
It is opt-in through the configured public fallback, derived mechanically from a
strict private mount, and confined to one private-over-public overlay rather than
an arbitrary ordered chain.

### Default mount and the root facade

The npm registry protocol expects a registry root:

```text
GET /foo
GET /foo/-/foo-1.0.0.tgz
```

pnpr keeps this root surface for compatibility, configured as a **default
target** that resolves to exactly one strict mount or derived compatibility
endpoint:

```yaml
defaultTarget: npmjs        # or: acme, npm-mirrors, ~~acme
```

Rules:

- **The default is an alias to one named target, never an ad-hoc blend.** Aliasing
  the root to a strict mount adds an address, not ambiguity: `/foo` *is*
  `~npmjs/foo`. The only way the root serves more than one origin is when the
  default target is a derived blended endpoint such as `~~acme`, which is opt-in
  through the configured public fallback and named in config.
- **There is no implicit hosted uplink.** pnpr does not ship a magic `~hosted`
  default. Hosted orgs are explicit `~<org>` mounts. A deployment that wants the
  root to be its hosted org sets `defaultTarget: acme`; `pnpr init` for a
  single-org deployment may scaffold that line into generated config, but it is
  visible config, not built-in behavior. This keeps the product model
  (organizations, not "the hosted implementation") and the multi-tenant case
  (no single default org) coherent.
- **Publish-to-root is allowed only when the default target writes to a hosted
  org.** If the default is a hosted org, or a blended endpoint whose private
  member is hosted, writes route there. If the default is an upstream, private
  upstream, mirror group, or non-hosted blended endpoint, unqualified publishes
  are rejected with a clear "name a hosted mount" error, so a publish can never
  silently land in the wrong place.

This makes the root path a product choice instead of the internal architecture.
Small deployments can keep one root URL; standalone deployments can point users
at `~<org>` registries directly.

### Client routing and lockfiles

The resolver treats registry identity as mount identity. A client maps scopes
and named registries to mount URLs, and pnpr rejects any registry URL that does
not map to an allowlisted mount before any server-side fetch:

```ini
registry=https://registry.example.com/~npmjs/
@acme:registry=https://registry.example.com/~acme/
@corp:registry=https://registry.example.com/~corp/
```

With explicit per-scope routing, composition happens in the client's registry
config, deterministically and visibly, rather than by server-side guessing.
Different scopes resolve against different mounts, each with its own auth, cache,
and tarball URL base.

For private package declarations, the recommended future-proof form is a
user-authored named registry that points at the strict private mount:

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

The compatibility form points legacy clients at the blended endpoint:

```ini
registry=https://registry.example.com/~~acme/
```

That keeps old single-registry workflows working, but new manifests should make
private registry selection visible in `package.json` through named-registry
specifiers.

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
`pnpm-workspace.yaml` — is unnecessary for the strict model and is dropped.

**The one new lockfile requirement: registry identity in package identity.**
What pnpm's reconstruction cannot express is *which* mount a package came from
when scope alone is ambiguous — the same `name@version` from two mounts (and
split-within-a-scope, e.g. `@acme/foo` hosted but `@acme/bar` from npm). For
those, the lockfile must record the registry identity in the package key so the
two cannot collide, and so reconstruction targets the right base. This is the
gap described in the [Motivation](#pnpm-cannot-represent-the-same-nameversion-from-two-registries-today),
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

### Serving tarballs in blended mode

Blended mode keeps the lockfile just as portable as the strict model, because
only the *persisted* `dist.tarball` must be canonical — the bytes can be served
from anywhere at fetch time:

- The packument served at `/~~acme/foo` selects the concrete origin and pins
  its `dist.integrity`.
- `dist.tarball` is written canonical for the `~~acme` base, so pnpm drops it
  and later reconstructs `https://<pnpr>/~~acme/foo/-/foo-1.0.0.tgz`.
- At fetch time pnpr **routes** that canonical request to the real bytes:
  - **serve internally** for the private member and for any same-host mount (no
    extra round-trip);
  - **HTTP-redirect to the public CDN** for the public fallback member, when a
    deployment wants to offload bandwidth. The redirect `Location` never enters
    the lockfile, so it may be non-canonical. Redirects are only legal to public
    targets, because the client has no credentials for a private upstream.

The redirect is therefore an optional bandwidth optimization for the public
passthrough, not the mechanism; the mechanism is "canonical persisted URL +
fetch-time origin routing, with integrity as the backstop."

### Serving tarballs inside pnpr

pnpr currently has two tarball-serving handlers — the normal path and the
`~<uplink>` path. The mount model unifies them: both should construct a concrete
**mount origin** and call one shared, origin-aware serving routine. Each mount
serves tarballs at the canonical path for its own base; cache namespace,
credential generation, integrity verification, and advisory/OSV screening are
all keyed by the resolved mount origin. Cross-origin selection happens only in
blended mode and only as described above.

### Relationship to auth-aware resolution caching

This RFC does not replace the authorization-aware resolution cache design. It
provides the cleaner origin model that design can use.

The existing auth-aware cache proposal distinguishes public routes,
pnpr-hosted private routes, and private proxied routes. Registry mounts make
those route identities explicit:

- public routes are public upstream mounts or the public member of a blended
  compatibility endpoint;
- pnpr-hosted private routes are hosted organization mounts;
- private proxied routes are private upstream mounts with pnpr-managed
  credentials;
- private access descriptors are derived from the selected concrete mount and
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
- mirror groups and blended compatibility endpoints cache nothing of their own;
  metadata and tarballs belong to the selected concrete member.

Policy checks also receive the concrete mount identity. This avoids pretending
that `foo@1.0.0` from two registries is the same package for every policy
decision. Advisory policy, tarball integrity, package access, and audit logs can
all refer to the mount that actually supplied the package.

## Rationale and Alternatives

### Keep Verdaccio-style packages and uplink fallback as the core model

This preserves compatibility with Verdaccio configuration, but it keeps the
wrong primitive at the center. Package-name rules and fallback chains force pnpr
to infer origin after the fact. That makes private tarball routing, cache
namespaces, named registries, and standalone hosted organization registries
harder to explain and easier to get wrong. pnpr can still support Verdaccio
config as a compatibility layer that compiles into strict mounts plus derived
`/~~<mount>` blended endpoints for the publish-private, fall-back-public shape,
but the core primitive is the mount.

### Use one blended root registry as the default model

pnpr could make the root smart enough to serve hosted, public, private, and
fallback content. This is convenient but hides provenance, and the more the root
does, the more hidden state a tarball request needs to answer safely. The mount
model keeps that identity explicit and confines blending to derived, named
compatibility endpoints.

### Rewrite `dist.tarball` to the concrete mount and persist it

An earlier draft had pnpr rewrite tarball URLs to the selected concrete mount
and persist them. This is rejected: a persisted concrete-mount URL is
non-canonical for the client's registry base, so pnpm cannot drop it, which
bakes the deployment host into every lockfile entry and churns the lockfile on
any host change. It regresses the portability pnpm already guarantees. Serving
at the canonical path plus fetch-time routing achieves the same provenance with
a portable lockfile.

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

1. Add a `RegistryMount` config model (`HostedOrg`, `Upstream`, `MirrorGroup`)
   plus derived `BlendedEndpoint` facades, and compile existing
   `uplinks`/`packages` config into it; a publish-private/fall-back-public config
   compiles into a strict private mount plus `/~~<mount>` compatibility endpoint.
2. Introduce an internal `MountOrigin` value that route handlers construct for
   every packument and tarball request.
3. Unify the normal tarball path and the `~<uplink>` tarball path into one
   shared origin-aware serving routine that takes a `MountOrigin`.
4. Key metadata/tarball cache namespaces and integrity/OSV screening on the
   resolved mount origin.
5. Serve every mount's tarballs at the canonical path for its own registry base,
   so pnpm's existing canonical-URL reconstruction keeps the host out of the
   lockfile. Do not rewrite or persist concrete-mount tarball URLs.
6. Implement mirror groups (any member may serve) and derived blended endpoints
   (private-over-public, integrity-backstopped, public-only redirect for the
   fallback member).
7. Teach the resolver allowlist and route classifier to map `registry` and
   `namedRegistries` URLs to mount identities and reject unknown ones.
8. Add registry identity to pnpm/pacquet lockfile package identity so the same
   `name@version` from two mounts cannot collide, recording a registry
   name/identity (not a URL) and reconstructing the tarball URL from config.
   Keep pacquet's reader, writer, and installer in parity with the format.
9. Make install, frozen-lockfile verification, and tarball fetching resolve a
   recorded registry identity through config, failing closed when it is absent.
10. Add hosted organization mount storage namespaces and route
    publish/unpublish to concrete hosted organization mounts, including through
    `/~~<mount>` only when the private member is hosted.
11. Wire the configurable default target / root facade, with publish-to-root
    allowed only for hosted targets.

Tests should cover:

- the same `name@version` existing with different bytes in two mounts, kept
  distinct in `packages`/`snapshots` and not colliding;
- strict per-scope routing producing portable lockfiles with no pnpr host
  persisted, reconstructing the right per-mount tarball URL;
- moving a lockfile between pnpr deployments with the same mount IDs without
  package-entry churn;
- mirror groups serving a tarball from any member and verifying one integrity;
- strict private mounts used through named-registry package.json declarations;
- `/~~<mount>` blended compatibility endpoints: private shadowing public;
  integrity mismatch failing closed when the selected origin changes between
  packument and tarball fetch; public-member redirect not persisted and never
  targeting a private upstream;
- a recorded registry identity missing from config failing closed with a clear
  error instead of falling back to the root registry;
- hosted organization mounts not falling through to upstreams;
- private upstream mounts keeping credentials server-side;
- cache namespace isolation by mount and credential generation;
- publish-to-root rejected when the default target is an upstream, mirror group,
  or blended endpoint whose private member is not hosted.

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

Verdaccio uses package rules and uplinks to let one registry facade proxy other
registries. That is useful for compatibility, and its publish-private,
fall-back-public shape is exactly what derived `/~~<mount>` endpoints model —
but it predates pnpm's named-registry use cases and pnpr's resolver/gateway
model.

Many hosted registries expose organization, project, or tenant boundaries in the
URL. `~<org>` follows that product shape while staying compatible with npm
package names and scopes beneath the mount.

## Unresolved Questions and Bikeshedding

- Is `~<mount>` the right path syntax for every mount, or should hosted
  organization mounts use a different namespace?
- Should the config term be `mounts`, `registries`, or `registryMounts`, and
  what are the final names for mirror groups and derived blended endpoints?
- What exact lockfile encoding carries registry identity in package identity —
  a registry-qualified package key, a package-to-registry table, or another
  compact form — and how does it interoperate with the existing `name@version`
  depPath grammar (which today special-cases only prefixes like `runtime:`)?
- How is the registry name/identity mapped to a URL at install time — through
  `namedRegistries`, through a single `pnprServer` base, or both — so a
  deployment move updates one setting?
- Should mirror-group membership be verifiable (pnpr asserting equivalence) or
  purely an operator declaration?
- Is `/~~<mount>` the right syntax for derived blended compatibility endpoints,
  and should pnpr expose them automatically for every private mount whenever a
  public fallback is configured?
- How much Verdaccio config compatibility should pnpr preserve, and for how
  long?
