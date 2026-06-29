# Registry mounts for pnpr

## Summary

pnpr should model every addressable registry surface as a **registry mount**.
A mount owns one clear origin policy: a pnpr-hosted organization registry, an
upstream registry, or a fallback/group that delegates to other mounts. Named
mounts are exposed at `https://<pnpr>/~<mount>/`, so hosted organizations,
server-owned upstream credentials, and composed fallback registries all share
one architecture. Fallback becomes composition over concrete registry mounts,
not a special "try these uplinks" behavior inside the default registry facade.

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

Fallback chains are a poor default abstraction across different security and
provenance domains:

- the same `name@version` can exist in two registries with different bytes,
  metadata, policies, or vulnerability posture;
- a tarball request that re-runs fallback can pick a different origin from the
  packument request that selected the version and `dist.integrity`;
- shared proxy caches need extra safeguards to avoid mixing private and public
  content for the same package name;
- server-owned upstream credentials need a stable route identity so they never
  leak into client lockfiles or another mount's cache.

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
- `~public` can be a fallback/group mount over other mounts;
- the root path can remain a compatibility/default facade, without being the
  internal primitive.

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

The root path remains available as a compatibility/default surface, but
internally it should resolve to a mount or a facade over mounts.

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

  public:
    fallback:
      members:
        - acme
        - npmjs

defaultMount: public
```

The syntax is illustrative. The required model is:

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
    Fallback {
        members: Vec<MountId>,
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

Hosted organization mounts have no upstream fallback by default. A request to
`/~acme/foo` should return `foo` only if the `acme` organization hosts `foo` and
the caller is authorized to read it.

Publishing, dist-tag updates, unpublish, token policy, audit logs, quotas, and
billing all naturally attach to the hosted organization mount. This is the
standalone pnpr product surface.

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

### Fallback mounts

A fallback mount delegates to an ordered list of other registry mounts:

```yaml
mounts:
  public:
    fallback:
      members:
        - acme
        - npmjs
```

Fallback is therefore composition over registry surfaces, not a separate
mechanism that knows how to fetch tarballs itself. A fallback mount does not own
package bytes. Its job is to select a concrete child mount.

Child mount authorization is still enforced by the child. A caller who can use
`~public` does not automatically gain access to `~corp` just because `corp` is
listed in `public`'s fallback members. The fallback walk skips children the
caller cannot use or fails according to the configured policy.

For a packument request:

```text
GET /~public/foo
```

pnpr tries each child mount according to the fallback policy. If `~acme/foo`
does not exist and `~npmjs/foo` exists, the selected concrete origin is
`~npmjs`.

For generic npm registry clients, the response can rewrite `dist.tarball` URLs
to the concrete selected mount:

```text
https://<pnpr>/~npmjs/foo/-/foo-1.0.0.tgz
```

not back to:

```text
https://<pnpr>/~public/foo/-/foo-1.0.0.tgz
```

This keeps tarball provenance stable. The tarball request is now explicitly for
the mount whose packument selected the version and declared `dist.integrity`.
It does not re-run fallback and cannot silently switch registries.

pnpm/pacquet lockfiles should not have to persist that concrete URL. The
concrete mount URL is a wire-level npm registry detail. Resolver output intended
for pnpm/pacquet should preserve the same selected mount provenance through a
symbolic registry identity, described below.

Fallback mounts may still be useful for compatibility and mirrors:

- "host organization packages first, then npmjs";
- "try the local public mirror, then npmjs";
- "try these equivalent upstream mirrors in order".

When a fallback is configured as a true mirror group whose members are declared
equivalent for package identity and bytes, pnpr may choose to keep group URLs
for tarballs. That should be an explicit mirror-group mode, not the default
fallback behavior.

Fallback mounts are read-only by default. Writes should target a concrete hosted
organization mount unless pnpr later adds an explicit write-routing policy.

### Root registry facade

The npm registry protocol expects a registry root such as:

```text
GET /foo
GET /foo/-/foo-1.0.0.tgz
```

pnpr can keep this root surface for compatibility. It should be configured as a
default mount or facade over mounts:

```yaml
defaultMount: public
```

If the default mount is concrete, root tarball URLs can remain root-relative.
If the default mount is a fallback/group, pnpr should prefer concrete mount
tarball URLs in packuments for generic npm clients so origin identity remains
explicit after the packument is served.

This makes the root path a product choice instead of the internal architecture.
Small deployments can keep the Verdaccio-like root behavior, while standalone
pnpr deployments can point users at `~<org>` registries directly.

### Named registries and resolver installs

The resolver should treat registry identity as a mount identity.

When a client sends a default registry or named registry, pnpr maps that URL to
one configured mount. A registry URL that does not map to an allowlisted mount
is rejected before any server-side fetch.

Examples:

```ini
registry=https://registry.example.com/~public/
@acme:registry=https://registry.example.com/~acme/
@corp:registry=https://registry.example.com/~corp/
```

With this model, pnpr does not need to fetch everything through the same
registry facade. Different scopes and named registries can naturally resolve
against different mounts, each with its own auth, cache, and tarball URL base.

Resolver output follows the same rule as registry packuments:

- public direct routes may keep direct upstream tarball URLs when the
  auth-aware routing policy allows it;
- private or pnpr-hosted routes use pnpr mount identities;
- fallback/group routes record the selected concrete mount identity.

### Generated named registries and lockfiles

pnpm/pacquet resolver installs should avoid hardcoding pnpr deployment URLs in
lockfile package entries. A lockfile that was resolved through:

```text
https://registry.example.com/~npmjs/foo/-/foo-1.0.0.tgz
```

should not have to store that full tarball URL forever. The selected mount is
the important fact, not the current public URL of the pnpr deployment.

pnpr should reuse pnpm's named-registry mechanism as the URL indirection layer.
When a package document is served from one registry surface but the selected
tarball route belongs to another concrete mount, pnpr should report the required
mount registry alias to the client. The client can then add or update a
generated `namedRegistries` entry in `pnpm-workspace.yaml` before writing the
lockfile:

```yaml
namedRegistries:
  pnpr-mount-npmjs: https://registry.example.com/~npmjs/
```

The lockfile package entry should then stay a registry resolution that records
which named registry alias to use:

```yaml
packages:
  foo@1.0.0:
    resolution:
      integrity: sha512-...
      registry: pnpr-mount-npmjs
```

The exact lockfile shape is a design detail. It could be an optional
`registry`/`registryAlias` field on registry resolutions, a lockfile-level
package-to-registry-alias map, or another compact representation. The
requirement is that the package entry stays a **registry resolution** rather
than a `TarballResolution` with a hardcoded URL.

At install time, the client resolves the package through the generated named
registry:

```text
pnpr-mount-npmjs -> https://<current-pnpr>/~npmjs/
```

If the same lockfile is used against another pnpr deployment with the same mount
IDs, only the generated `namedRegistries` URLs need to change. Package entries
do not churn. If a lockfile refers to a generated named registry that is absent
from the workspace config, the install should fail with a clear configuration
error instead of falling back to the root registry or recomputing provenance.

Generated mount aliases differ from user-authored named registries such as
`gh:`. They are generated by pnpr from mount IDs, use a reserved prefix so user
config cannot shadow them, and are scoped to pnpr-routed registry resolutions.
User-authored named registries still work as today for manifest specifiers and
ordinary registry selection.

The trigger is not "any arbitrary tarball URL host differs from the package
document host." Many registries use CDN tarball hosts that are not registry
origins and cannot serve packuments. The safe trigger is: pnpr's mount selection
knows the package document came through one mount but the tarball must be served
through another concrete pnpr mount. In that case the generated named registry
points at the concrete pnpr mount (`/~npmjs/`, `/~corp/`, ...), not at a raw
third-party tarball host.

The client should not rewrite `package.json` dependency specifiers to generated
mount aliases. The generated `namedRegistries` entries are a URL table for the
lockfile and install pipeline. They are not a change to the user's authored
dependency declarations.

Generic npm clients that only consume packuments cannot use this symbolic
lockfile mechanism, so registry responses may still contain concrete
`dist.tarball` URLs. The no-hardcoded-URL requirement applies to pnpm/pacquet
resolver output and lockfile persistence.

### Relationship to auth-aware resolution caching

This RFC does not replace the authorization-aware resolution cache design. It
provides the cleaner origin model that design can use.

The existing auth-aware cache proposal distinguishes public routes,
pnpr-hosted private routes, and private proxied routes. Registry mounts make
those route identities explicit:

- public routes are public upstream mounts or concrete children of fallback
  mounts;
- pnpr-hosted private routes are hosted organization mounts;
- private proxied routes are private upstream mounts with pnpr-managed
  credentials;
- private access descriptors are derived from the selected concrete mount and
  its access policy or credential generation.

The same mount identity should feed resolver cache keys, metadata cache keys,
tarball cache namespaces, and pnpr-routed symbolic registry identities. This
avoids having one concept for resolver caching and a different concept for
registry serving.

### Authorization, cache, and policy

Authorization is checked at the mount boundary. The caller's pnpr identity may
authorize access to a hosted organization mount or to a private upstream mount,
but it is not forwarded to third-party registries.

Cache keys include the concrete mount identity:

- hosted organization packages are stored and cached under the organization
  mount namespace;
- upstream metadata and tarballs are cached under the upstream mount namespace,
  including credential generation where credentials are configured;
- fallback mounts may cache delegation decisions, but package metadata and
  tarballs belong to the selected child mount.

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
harder to explain and easier to get wrong.

pnpr can still support Verdaccio-shaped config as a compatibility layer, but it
should compile that config into mounts internally.

### Use one blended root registry

pnpr could keep one root registry and make it smart enough to serve hosted
packages, public npm packages, private upstream packages, and fallback chains.
This is convenient for clients, but it hides provenance. The more the root path
does, the more internal state is needed to answer a tarball request safely:
which upstream supplied the packument, which credential was used, whether the
tarball cache is shared, and which policy context applies.

Registry mounts keep that identity in the URL and in the server's internal
origin descriptor.

### Add a generic `~hosted` registry

A `~hosted` mount would separate hosted packages from upstream packages, but it
does not match the product model. Operators and users think in terms of
organizations, teams, projects, or tenants, not "the hosted implementation".

`~<org>` gives hosted packages a natural ownership boundary and leaves package
scopes free to mean normal npm scopes.

### Run separate pnpr instances for each registry

Separate instances give strong physical isolation, but they remove useful
composition. Operators would need extra routing, duplicated config, duplicated
caches, and separate publish/auth surfaces for registries that logically belong
to the same product.

Mounts keep the isolation boundary explicit without requiring a process per
registry.

## Implementation

Implementation should be staged so compatibility remains intact while the
server internals move to the mount model.

1. Add a `RegistryMount` config model and compile existing `uplinks` and
   `packages` config into it.
2. Introduce an internal `ResolvedOrigin`/`MountOrigin` value that route
   handlers use for packument and tarball requests.
3. Refactor the current normal tarball path and `~<uplink>` tarball path so
   they construct a mount origin and call shared origin-aware serving code.
4. Move metadata and tarball cache namespace decisions behind the selected
   mount origin.
5. Make fallback/group mounts select a concrete child mount for packuments and
   rewrite generic npm `dist.tarball` URLs to that child mount by default.
6. Teach the resolver allowlist and route classifier to map `registry` and
   `namedRegistries` URLs to mount identities.
7. Generate reserved `namedRegistries` aliases for concrete mounts selected by
   pnpr and persist/update them in `pnpm-workspace.yaml` before writing the
   lockfile.
8. Add a pnpm/pacquet lockfile representation for mount-routed registry
   resolutions so resolver output records the generated named-registry alias
   instead of a concrete tarball URL.
9. Teach install, frozen-lockfile verification, and tarball fetching to resolve
   generated mount aliases through `namedRegistries`.
10. Add hosted organization mount storage namespaces and make publish/unpublish
   write to concrete hosted organization mounts.
11. Keep the root registry facade as a compatibility/default mount, but avoid
   making root fallback behavior the internal tarball provenance mechanism.

Tests should cover:

- the same package name and version existing with different bytes in two
  mounts;
- fallback packuments emitting concrete child mount tarball URLs;
- pnpm/pacquet resolver output using generated named-registry aliases instead
  of concrete pnpr tarball URLs;
- client-side `pnpm-workspace.yaml` updates adding generated mount aliases
  without overwriting conflicting user-authored aliases;
- tarball requests not re-running fallback after a lockfile or packument selected
  an origin;
- hosted organization mounts not falling through to upstreams;
- private upstream mounts keeping credentials server-side;
- cache namespace isolation by mount and credential generation;
- resolver named registries mapping to mounts and rejecting unknown registry
  URLs;
- moving the same lockfile between pnpr deployments with the same mount IDs
  without package-entry churn;
- root compatibility behavior for existing Verdaccio-style configs.

## Prior Art

Verdaccio uses package rules and uplinks to let one registry facade proxy other
registries. That design is useful for compatibility, but it predates pnpm's
current named-registry use cases and pnpr's resolver/gateway model.

npm clients already support routing different package scopes to different
registries through configuration. Registry mounts make that registry identity a
first-class server concept instead of collapsing all requests into one facade.

Many hosted package registries expose organization, project, or tenant
boundaries in the URL. `~<org>` follows that product shape while staying
compatible with npm package names and scopes beneath the mount.

## Unresolved Questions and Bikeshedding

- Is `~<mount>` the right path syntax for every mount, or should hosted
  organization mounts use a different namespace?
- Should the config term be `mounts`, `registries`, or `registryMounts`?
- What exact lockfile shape should reference a generated named-registry alias:
  `resolution.registry`, a package-to-registry-alias table, or another compact
  form?
- What should the reserved generated alias prefix be, and how should pnpm
  handle an existing user-authored `namedRegistries` entry that collides with
  it?
- Should generated `namedRegistries` URLs be written as absolute pnpr URLs, or
  should pnpm support values derived from `pnprServer` so deployment changes
  update one setting instead of several aliases?
- Should pnpr ever support write-routing from a fallback mount to a designated
  hosted child?
- Should mirror groups be a separate mount type from ordinary fallback groups?
- What should the default root facade be for a new standalone pnpr deployment?
- How much Verdaccio config compatibility should pnpr preserve, and for how
  long?
