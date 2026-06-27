# Authorization-aware resolution cache for pnpr

## Summary

pnpr currently disables its resolution cache for any request that carries
client-provided upstream credentials, which makes authenticated installs ~2.8×
slower than anonymous ones even when every dependency is public. This RFC
proposes making the cache *authorization-aware* instead of all-or-nothing: each
fetch route is classified once, before the request is sent. Known-public routes
(including the default unscoped npmjs public route) are fetched anonymously and
shared globally. Routes that match configured private/auth rules are fetched
with the selected credential or hosted-package policy and added to the
resolution's **private footprint**; private resolutions and private metadata are
then keyed by the private access descriptor that produced them. For proxied
upstream packages, that descriptor is always a pnpr-managed upstream credential alias, not a
client-forwarded token. For pnpr-hosted packages, the descriptor is the hosted
route/access-policy identity and hits re-run package authorization for the
caller. Client auth identifies the caller to pnpr and may authorize packages
hosted by pnpr itself, but it is not forwarded to third-party registries and is
not digested as an upstream cache credential. The same route decision governs
tarball URLs returned by resolution: public routes may keep direct upstream
tarball URLs, while private or unknown routes are rewritten to pnpr read-only
install gateway URLs so resolver responses and cached lockfiles never expose
upstream credentials or unusable private upstream URLs.

## Motivation

pnpr's resolution cache stores the result of a dependency resolution keyed by
the resolution inputs (registry, named registries, overrides, project
manifests, lockfile, trust policy, …). On a cache hit it returns the resolved
lockfile without re-running resolution, which is the single biggest speedup pnpr
offers over resolving locally.

The cache is **disabled the moment the client sends any upstream auth entry** in
the resolve request (`pnpr/crates/pnpr/src/resolver.rs`):

```rust
let resolution_cache_key = if request.auth_headers.is_empty() && request.lockfile.is_none() {
    resolution_cache_key(config, &request)
} else {
    None
};
```

This RFC removes both broad bypasses from the resolution-cache path. Caller
identity, selected pnpr-managed upstream aliases, and lockfile presence are
inputs to auth-aware cache lookup, not reasons to skip it. Lockfile-seeded
update resolves must still participate in the cache: the base resolution-input
key includes the input lockfile and the lockfile mode flags (`frozenLockfile`,
`preferFrozenLockfile`, `trustLockfile`, manifest checks, and trust policy), so
a hit is scoped to the exact lockfile-seeded request shape. A proven-fresh
frozen lockfile may keep its existing short-circuit before cache lookup because
it does not run resolution at all; any request that needs a real resolve should
be cacheable.

The reason is correct: a resolution computed with an upstream credential that
can see a private package *contains* that package's versions and tarball URLs,
and the current cache key deliberately omits auth, so caching it globally could
serve one user's private resolution to another. Skipping the cache is the only
safe behavior given the current key.

But it is far too blunt. The measured impact (from
[pnpm/pnpm#12604](https://github.com/pnpm/pnpm/issues/12604)):

| Scenario (hot cache/store) | Time | Speedup vs. resolving directly |
| --- | --- | --- |
| Anonymous resolve | 0.523 s | ~2.3× faster |
| Authenticated resolve | 1.484 s | ~1.0× (no speedup) |

The crucial observation is that **most authenticated installs are 100% public.**
Today a pnpm client may send npmrc-derived upstream auth entries even when every
dependency comes from the public registry, so the request "looks authenticated"
and loses the cache despite touching nothing private. In the proposed model,
client auth only identifies the caller to pnpr; third-party upstream credentials
are pnpr-managed aliases selected by route policy. Either way, caller
authentication alone is not a privacy signal. We should not pay the full cost of
the privacy guarantee on requests that have no private data at all.

The goal: keep the guarantee — one user's private resolution must never be
served to another — while restoring the cache for the dominant public path and
sharing private resolutions safely among callers who legitimately share access.

## Detailed Explanation

The design has two independent parts: deciding **what is private** in a
resolution (classification), and deciding **who may read** a cached entry
(keying/gating).

### Part 1 — Classify by fetch route, no double requests

Privacy is a property of the **fetch route that serves a package**, not of
whether the install request happened to contain any credential. The route is the
registry URL plus the package name/scope and the configured auth/private rules.
This matters in both directions:

- **Scoped names can be public** — `@babel/core` from the public registry is
  public.
- **Unscoped names can be private** — if a deployment's *default* `registry` is
  a self-hosted/corporate one (Verdaccio, Artifactory, GitHub Packages, a
  private proxy), then unscoped packages served from it can be fully private.

The base design must not probe every package twice. Classification therefore
chooses **one** request shape for each metadata or resolve-time tarball fetch:

- **Built-in public npm route:** by default, unscoped packages served from the
  exact official npm registry (`registry.npmjs.org`) are treated as public. The
  [npm scope docs](https://docs.npmjs.com/about-scopes/) describe unscoped
  packages as public and private packages as user- or organization-scoped, so
  pnpr should deliberately select the anonymous upstream fetch shape for these
  metadata requests. The caller's pnpr identity is irrelevant to this public
  upstream route; it is used later only for pnpr-hosted package access and for
  deciding whether the caller may use a pnpr-managed upstream alias. Operators
  may disable or override this built-in route if npm's policy changes or if they
  want a conservative deployment.
- **Configured public routes:** an operator may explicitly declare additional
  registries, scopes, or package patterns public. These are fetched without
  upstream auth and can participate in the global public cache.
- **Private proxied routes:** if a proxied route matches a configured private
  rule, pnpr may send only a pnpr-managed upstream credential alias selected for
  that route and authorized for the caller. There is no anonymous preflight and
  no fallback to a client-forwarded upstream credential.
- **Private pnpr-hosted routes:** if the package is hosted by pnpr itself, the
  client auth identifies the caller to pnpr and is checked against pnpr's package
  access policy. It is not forwarded upstream and is not used as the cache
  credential digest.
- **Inline URL credentials:** dependency URLs or resolved tarball URLs that
  contain `user:pass@host` credentials are rejected by the resolver. They must
  not be converted into Basic auth, used as an upstream credential identity, or
  stored in any shared cache. Operators that need authenticated direct tarball
  URLs should model that host with a pnpr-managed upstream credential alias
  instead.
- **Unknown routes:** default to private unless explicitly declared public. If
  no pnpr-managed upstream credential alias or pnpr-hosted authorization is
  available for an unknown/private route, pnpr may resolve it anonymously only as
  a non-shareable private miss when the route permits anonymous access; it must
  not write the result to the global public cache.

A route is **pnpr-hosted** when the normalized request registry points at this
pnpr service, or at an operator-declared hosted registry alias, and the package
matches pnpr's hosted package policy. Hosted classification wins before proxied
uplink selection. This avoids treating pnpr itself as a third-party upstream,
forwarding client tokens back to pnpr as upstream auth, or double-applying the
registry proxy's uplink policy when a client's default or named registry is the
pnpr server.

A resolution's **private footprint** is the set of private route identities
whose metadata or resolve-time tarball data was actually fetched during that
resolve, including direct HTTP tarball dependencies fetched during resolution to
read their manifest and integrity. Each route is paired with the exact private
access descriptor selected for it:

- **Proxied upstream descriptor:** the selected pnpr-managed upstream alias plus
  generation.
- **Hosted package descriptor:** the pnpr-hosted route/access-policy identity,
  not the caller's bearer token.

This is derived from information pnpr already has during the single resolve it
already performs — **no per-package probing and no extra requests.** During
resolution pnpr knows, for every fetch, which route it targeted and which route
policy and private access descriptor were selected.

A footprint that is **empty** means the resolution is fully public.

> Note on over-counting: a request can have a pnpr caller identity even when the
> install is fully public. That caller identity is *not* a privacy signal and
> must not mark ordinary public installs as private. Classification is by route
> policy and the private access descriptor actually selected for that route. In
> particular, unscoped packages on the official npm registry are fetched without
> upstream auth, so pnpr caller identity does not pessimize the common unscoped
> public path.

### Part 2 — Key public entries globally, private entries by access descriptor

The cache key behavior depends on the footprint:

- **Empty footprint (public):** the key excludes auth, exactly as today. The
  entry is shared globally across all callers. *This is what recovers the
  ~2.8×.*
- **Non-empty footprint (private):** the key additionally incorporates an HMAC
  (keyed with a server secret, so the key is not correlatable offline) of the
  exact private access descriptors selected for the private routes in the
  footprint.

Private access descriptors have two shapes:

- **Proxied:** key contribution = HMAC of `upstream-alias + generation`; gate =
  the caller can still select that alias for the route.
- **Hosted:** key contribution = HMAC of the hosted route/access-policy
  identity; gate = pnpr re-runs package access policy for the caller. Hosted
  entries are not keyed per caller, and they are not keyed by a bearer token.

Lookup still happens before resolution. pnpr first computes the normal
resolution-input key, then reads the candidate list stored under that base key.
Each candidate is matched against the private access descriptors the current
request can select or satisfy for that candidate's private footprint. Public
candidates match every caller. Private candidates match only when the current
request can reproduce the same proxied alias descriptors and satisfy the same
pnpr-hosted route policies.
The footprint discovered during resolution is used only when writing a new
candidate after a miss, not for finding the first candidate set.

Why key and gate by private access descriptor rather than "does the caller have
some auth for this scope?" — because the latter is an authorization bypass. A
bogus client token for `@acme` must never unlock a private proxied resolution.
Proxied packages therefore do not use client-forwarded upstream auth at all:

- If the caller is authorized for a **pnpr-managed upstream credential alias**,
  the request selects that alias identity and can match entries produced by the
  same alias.
- If the caller is not authorized for the alias, the request cannot select that
  alias identity, so it cannot match the alias's private entries.
- If the package is **hosted by pnpr itself**, the request is matched by
  re-running pnpr package access policy for the caller against the hosted package
  routes in the footprint. The client's bearer token authenticates the caller; it
  is not the upstream cache credential.

The cache key requires the current request to reproduce the exact private access
descriptor that produced the entry. For proxied entries that means selecting the
same authorized alias generation. For pnpr-hosted entries that means satisfying
the same package route policy at lookup time. An invalid, absent, or
unauthorized credential cannot match.

**Safety invariant:** a private-footprint entry is keyed by the same private
access descriptor that produced it. pnpr-managed alias entries are gated by
alias access policy, and pnpr-hosted entries are gated by pnpr package access
policy. Private data can never leak beyond the alias authorization or package
access policy that produced it. The design does not rely on hiding whether a
base resolution key has private candidates; unauthorized callers always observe
a miss/fail-closed resolve, candidate counts and footprint details must not be
returned to clients, and private hit/miss metrics should be operator-only. This
leaves only a low-bandwidth existence/timing oracle over non-secret base inputs;
deployments that treat that as sensitive can normalize miss/hit timing or
disable private candidate lookup for unauthorized callers at a performance cost.

### Team-owned upstream credentials

For third-party/proxied packages, pnpr should not forward arbitrary client auth
to the upstream registry. The resolver should use the same proxy shape as the
registry server: clients authenticate to pnpr, and pnpr uses operator-managed
upstream credentials when the caller is allowed to use them. pnpr already has
server-owned uplink auth for the registry proxy and package access policies for
callers; this RFC extends that model to resolver-managed third-party
credentials.

An operator can configure **upstream credential aliases** for third-party
registries:

- each alias has a registry/scope/package route policy, the upstream credential
  material (for example from an environment variable or secret store), and an
  access policy saying which pnpr users may use it;
- callers authenticate to pnpr as they already can with pnpr bearer tokens;
- pnpr, not the client, selects the alias when the caller is authorized for that
  route, sends the alias's upstream credential, and records the alias identity
  in the private footprint;
- cache keys and metadata mirrors use a stable proxied descriptor such as
  `credential-alias + generation`, HMACed with the server secret. The raw token
  is never written to cache keys or exposed to clients.

This gives teams the fast path without allowing client-supplied third-party
tokens to affect proxied resolution. Everyone authorized for the same alias
shares private resolution entries and private metadata entries. A caller who is
not authorized for the alias cannot select that alias and therefore cannot
match its cache entries.

Route selection precedence:

1. Public route policy wins and suppresses auth.
2. A pnpr-hosted private route uses the caller's pnpr identity and package
   access policy; no upstream credential is involved.
3. A proxied private route selects an authorized pnpr-managed upstream credential
   alias.
4. If the route is private/unknown and no hosted-package authorization or
   upstream alias is available, resolve anonymously only as a non-shareable
   private miss when the route permits anonymous access; otherwise fail closed.
   Never forward client auth to the proxied upstream registry and never write a
   global public cache entry for this path.

When an upstream credential is rotated, the alias generation changes, so new
resolves populate a new private namespace. Old entries age out by TTL. If a
user's team access is revoked, pnpr re-evaluates alias authorization before
serving private hits, so the caller stops matching those private entries even
while the shared upstream token itself remains valid for other team members.

### Tarball URL routing

Resolution cache entries include tarball URLs, so the URL routing decision must
be part of the same route policy. A private cached resolution is safe to replay
only if replaying it does not hand a client an upstream URL that requires
server-owned credentials or reveal those credentials.

For each selected package, pnpr applies the same route classification when it
emits streamed package frames and the returned lockfile:

- **Public/anonymous route:** pnpr may emit the upstream `dist.tarball` URL.
  Known-public and configured-public routes do not need an anonymous tarball
  probe on the hot path; the route policy is the proof.
- **Private proxied route:** pnpr emits a pnpr read-only install gateway URL.
  The gateway fetches from the upstream registry with the selected
  pnpr-managed upstream alias after re-checking caller access to that alias.
- **Private pnpr-hosted route:** pnpr emits a pnpr-hosted tarball URL and
  re-checks the caller against pnpr package access policy before serving it.
- **Unknown route:** pnpr emits a pnpr gateway URL unless the operator has
  explicitly classified the route as public or opted into a successful
  public-route detection path.

Resolver responses, streamed package frames, and returned lockfiles must never
include upstream `Authorization` values, upstream tokens, or a private upstream
URL solely because pnpr fetched it successfully with server-owned credentials.
Cached resolutions store already-routed tarball URLs, and clients must consume
those URLs instead of reconstructing private upstream URLs from local registry
config.

Frozen and lockfile-seeded paths must preserve this routing decision. If an
existing lockfile contains registry-shaped entries or tarball URLs that do not
match the current route policy, pnpr should return a freshly routed lockfile or
force the client through the pnpr gateway rather than letting the client fetch a
private or unknown upstream URL directly.
Lockfile-seeded update resolves still use the resolution cache; their cache key
includes the input lockfile and lockfile mode flags, and the cached value is the
routed output lockfile for that exact input.

### Private metadata cache

The resolution cache cannot be safe if the lower packument mirror is still
auth-blind. Private metadata must follow the same route policy:

- **Public/anonymous route:** use the existing global metadata mirror unchanged.
  This keeps the dominant public path as fast as today.
- **Private route with a private access descriptor:** use a descriptor-scoped
  metadata namespace keyed by the same HMACed private access descriptor, for
  example `metadata-private/<descriptor-id>/<registry>/<package>.jsonl`. The
  existing disk fast paths, in-memory cache, request coalescing, and conditional
  GETs can remain; only the namespace changes.
- **Private/unknown route without a private access descriptor:** bypass persistent
  sharing or store only in a request-local cache. Never write it to the global
  public mirror.

The metadata cache scope is **per metadata fetch**, not per resolve. A single
workspace resolve may read public npmjs metadata from the global mirror, private
metadata for alias A from alias A's descriptor namespace, private metadata for
alias B from alias B's namespace, and pnpr-hosted metadata from a hosted-policy
namespace. Implementations must not approximate this by swapping the whole
`cache_dir` for a request. The chosen scope has to flow into every packument
fast path: persistent mirror path, in-memory metadata key, fetch-lock key,
conditional request headers, prefer-offline reads, maturity/published-by
shortcuts, and disk fallback after a failed fetch.

On `401`/`403` from the upstream registry, pnpr must fail closed: do not fall
back to a stale global mirror and do not use a descriptor-scoped mirror from a
different private access descriptor. For transport failures (`5xx`, timeout,
network failure), pnpr may fall back only to metadata in the same public or
descriptor-scoped namespace and only within the normal freshness policy.
Registries that hide private packages as `404` must be treated the same way for
private routes: do not satisfy the miss from a broader or auth-blind mirror.

### Consequences for sharing

- Public installs: shared globally — full cache benefit, even for authenticated
  clients, as long as the packages route through public/anonymous fetch rules.
  Public tarballs may still download directly from upstream URLs.
- CI / teams using a pnpr-managed upstream credential alias: shared across every
  pnpr user authorized for that alias, without exposing the raw upstream token to
  clients. Private tarballs are served through pnpr gateway URLs, not by sending
  upstream tokens or private upstream URLs to clients.
- pnpr-hosted private packages: shared across callers who currently satisfy the
  package access policy recorded in the footprint. Client tokens authenticate
  callers to pnpr; they are not cache keys.
- Scoped public packages on npmjs with an authenticated pnpr request: still
  public if the route policy says public. Client auth to pnpr is not forwarded
  upstream and does not make the entry private.
- Self-hosted deployments whose *public* default registry is not covered by a
  public route rule: treated as private until the operator declares it public —
  a one-line config change recovers global sharing.

### Revocation window

For pnpr-managed upstream credential aliases, rotating the alias generation
immediately moves future hits and writes to a new namespace; the old namespace
ages out. pnpr-hosted package access is re-evaluated before serving a
pnpr-hosted private hit. For deployments that want zero upstream revocation
window, an **opt-in** lightweight validation — a single authenticated request
(one round trip, not a full re-resolve) — can run before serving an upstream
private hit. The default relies on the short TTL plus per-hit alias/package
authorization checks.

### Optional: auto-detecting public custom registries

Operators who do not want to declare a public self-hosted registry can opt into
lazy classification **at the registry/scope granularity** (never per package):
attempt the first fetch to an unknown registry anonymously. If the route later
needs auth, use only a configured pnpr-managed upstream alias, never a
client-forwarded upstream credential. Cache the `(registry, scope) →
requires-auth` verdict with a TTL. Cost: at most one extra round trip per new
private registry-scope per TTL window, and zero for the public path. This is a
later refinement; the base design is config-driven with no probing.

## Rationale and Alternatives

**Alternative A — Per-package anonymous probing.** Classify each package by
fetching it both anonymously and with the selected upstream credential.
Rejected: it roughly doubles metadata requests, defeating the performance goal.
This RFC instead uses a single route-policy decision per fetch.

**Alternative B — Always vary by auth or credential.** Simple and safe, but it
gives *every* authenticated caller or selected upstream credential its own cache
namespace, so the dominant public case never shares across users — it only helps
repeat installs by the same caller or alias. It also hashes the wrong thing for
this design: client auth is a caller identity for pnpr-hosted packages, not an
upstream credential for proxied packages. This RFC's footprint split keeps
public resolutions globally shared while applying private-descriptor keying only
where it is actually needed.

**Alternative C — "Caller has a token for the scope" gate.** Share a private
entry with any authenticated caller or any caller that presents some credential
for the registry. Rejected: it is an authorization bypass. Client auth to pnpr
authorizes only pnpr-hosted package access; proxied package access requires the
caller to be authorized for the selected upstream alias. Keying by credential
alias identity plus access-policy checks closes this.

**Alternative D — Server-side encrypted credential vault.** Store upstream
credentials in pnpr (clients authenticate only to pnpr) and encrypt them at
rest. This is a worthwhile, separate feature for credential *management* and
*transmission*, but it does **not** fix caching on its own: the cache still has
to be gated by the selected upstream alias identity and caller access.
Encrypting the cached resolutions themselves would actively *defeat* sharing.
The server-managed alias feature in this RFC is the minimal useful slice of that
vault idea for performance: it supplies a stable proxied descriptor and an
access policy, while full credential lifecycle management can remain a later
extension.

**Alternative E — Proxy all tarballs through pnpr.** This is the simplest
tarball-routing rule: clients never receive upstream credentials or private
upstream URLs. Rejected as the default because public packages dominate many
installs, and proxying all public tarball bytes through pnpr increases
bandwidth, CPU, storage, and latency on the path this RFC is trying to keep
fast. Private and unknown tarballs still use this path.

**Alternative F — Return upstream auth tokens to clients.** This keeps pnpr out
of the tarball byte path, but it turns pnpr into a credential broker. Generic
npm registry credentials are reusable bearer or Basic tokens and may grant
broader access than the resolved install. Rejected: pnpr must keep upstream
credentials server-side. A future optimization could support non-reusable,
audience-bound, package-scoped download URLs where an uplink provides them, but
that cannot be the baseline for npm-compatible registries.

**Alternative G — Do nothing.** Authenticated installs stay ~2.8× slower. Given
that clients commonly authenticate even when resolving public packages, this
penalizes the common case for no privacy benefit.

This proposal is the minimal change that recovers the common-case performance
without weakening the privacy guarantee, and it stands alone regardless of
whether a credential vault is later added.

## Implementation

No pnpm wire-protocol change is required: route selection fits in the existing
streamed package frames and returned lockfile URLs. The implementation does need
changes in pnpr's server-side resolver integration, in the pacquet fetch path
that chooses metadata/tarball auth, and in client lockfile materialization so
clients consume routed tarball URLs instead of reconstructing private upstream
URLs from registry config.

- **Thread pnpr caller identity into resolution.** The HTTP request's resolved
  pnpr identity (`AuthedCaller` / `Identity`) must be passed into
  `serve_resolve`, `handle_resolve`, cache lookup, cache write, route selection,
  and any pnpr-hosted package access checks. This is a server-internal API
  change, not a pnpm client wire-protocol change: the client already
  authenticates to pnpr with the request `Authorization` header.
- **Route policy + auth selection.** Add route classification to pnpr config
  (`pnpr/crates/pnpr/src/config.rs`): built-in anonymous-public handling for
  unscoped packages on `registry.npmjs.org`, plus operator rules for public and
  private registries/scopes/package patterns, plus optional server-managed
  upstream credential aliases with per-alias access policies. The fetch path
  must choose the route policy and selected upstream auth together. Public routes
  send no upstream auth; private proxied routes send only the selected
  pnpr-managed upstream alias; pnpr-hosted routes use client auth only to
  authorize the caller against pnpr package policy. Hosted-route detection must
  normalize request registry URLs against the pnpr service's own public base URL
  and any configured hosted aliases before considering proxied uplinks.
- **Authorize pnpr-managed upstream credentials.** Reuse pnpr's existing caller
  identity (bearer-token-backed users), uplink auth configuration concepts, and
  registry package permission policy shape where possible to decide whether a
  request may use a configured upstream credential alias. Add an optional
  named-group/team layer if operators need group reuse. If no pnpr identity is
  available, pnpr cannot select a team credential; it fails closed for private
  proxied routes unless anonymous access is explicitly permitted as a
  non-shareable private miss.
- **Define private access descriptor inputs.** For proxied packages, the private
  access descriptor is the configured upstream alias plus generation, HMACed
  with the server secret. It is not derived from
  `AuthHeaders::for_url_with_package`, request `Authorization` headers,
  registry/scope config entries, or inline `user:pass@host` URL auth. URL and
  package normalization still matter for route-policy and alias selection, but
  once a proxied alias is selected, the cache descriptor is the alias identity
  plus generation. For pnpr-hosted packages, the footprint stores the hosted
  route/access-policy identity and cache hits re-run pnpr authorization for the
  caller.
- **Reject inline URL auth before fetch.** The resolver must reject dependency
  URLs and resolved tarball URLs whose parsed URL contains username/password
  credentials. This avoids accidentally following pacquet's default
  `AuthHeaders::for_url*` inline-Basic behavior on a path that has no
  pnpr-managed private access descriptor. Rejection should happen before the
  HEAD/GET request and before any resolution or metadata cache write.
- **Record the per-fetch route during resolution.** The resolve path
  (`pnpr/crates/pnpr/src/resolver/resolve.rs`) and the pacquet npm/tarball
  fetchers must surface, for each metadata fetch and resolve-time tarball fetch,
  including direct HTTP tarball dependencies fetched for manifest/integrity, the
  route identity, whether it was public or private, and the selected private
  access descriptor digest. The existing `ResolutionObserver` reports resolved
  tarball packages after a resolver result, so it is not sufficient by itself;
  the hook needs to sit at the actual fetch/auth-selection layer. If a fast path
  serves metadata from memory or disk without making an HTTP request, it must
  still use and record the route/cache scope that would have been used for the
  fetch.
- **Route emitted tarball URLs.** When emitting streamed package frames and the
  returned lockfile, use the same route decision: public/anonymous routes may
  keep upstream `dist.tarball` URLs; private proxied and unknown routes must use
  pnpr read-only install gateway URLs; pnpr-hosted packages use pnpr-hosted
  tarball URLs. The resolver must not return upstream auth headers, upstream
  tokens, or private upstream tarball URLs that are fetchable only with
  server-owned credentials.
- **Make lower metadata caches descriptor-scoped.** Pacquet's shared metadata
  mirror is currently keyed by registry/package, not by auth. Add a metadata
  cache scope to the npm resolver fetch context:
  - `Public` uses the current global mirror path unchanged.
  - `Private { access_descriptor }` uses a descriptor-scoped mirror path and
    matching in-memory key/fetch-lock key, preserving the current fast disk and
    request-coalescing paths without cross-descriptor sharing.
  - `Bypass` uses only request-local caching.
  Private/auth metadata must not be loaded from or written to a global auth-blind
  mirror in a way that lets a later invalid credential reuse it. Upstream
  `401`/`403` must fail closed; only transport failures may fall back to stale
  metadata, and only within the same cache scope. This scope must be computed per
  `(registry, package)` fetch and threaded through `pick_package`,
  `fetch_full_metadata_cached`, `get_pkg_mirror_path`, in-memory cache keys,
  fetch-lock keys, verifier paths, and every disk fallback path.
- **Apply metadata cache scope to verification fetches too.** Lockfile
  verification and trust checks do not add packages to the resolution footprint,
  but their packument fetches still read and populate metadata mirrors. They must
  use the same `Public` / `Private { access_descriptor }` / `Bypass` metadata
  scope as resolution fetches. Otherwise an unauthorized caller could use
  verifier verdicts as a private-metadata oracle, or a verifier could
  populate/read an auth-blind mirror that later affects resolution.
- **Footprint + keying in the cache layer**
  (`pnpr/crates/pnpr/src/resolver.rs`):
  - Replace the current `auth_headers.is_empty() && request.lockfile.is_none()`
    gate. A cache candidate base key is computed for requests with or without an
    input lockfile. Fresh frozen-lockfile reuse may still short-circuit before
    cache lookup, but any request that proceeds to resolution should be eligible
    for cache lookup and write.
  - Keep the current resolution-input key as the base cache key, but store one
    or more candidate entries under it. Each candidate carries its private
    footprint and the HMAC digest of the private access descriptors that
    produced it. A public candidate has an empty footprint and matches every
    caller; a private candidate matches only when the current request can select
    the same proxied alias descriptors and the caller is still authorized for
    any pnpr-managed aliases or pnpr-hosted package policies in the footprint.
    This avoids knowing the footprint before the first resolve and avoids a
    second resolve or anonymous probe.
  - After a resolve, compute the footprint from the recorded routes; store an
    empty-footprint candidate for public resolutions, or a descriptor-keyed
    candidate for private resolutions.
  - For lockfile-seeded requests, include a stable digest of the canonical input
    lockfile plus the lockfile behavior flags in the base key. A different input
    lockfile, update mode, freshness policy, or trust policy must not match a
    prior candidate.
  - Store already-routed tarball URLs in cached resolutions. A private cache hit
    must replay pnpr gateway URLs for private/unknown tarballs, not raw upstream
    URLs that require the selected server-owned credential.
  - Bound the number of candidates stored under one base key and evict expired
    or least-recently-used private candidates first. Alias generation rotation,
    route-policy changes, and unusual workspaces must not make lookup cost grow
    without bound.
  - The HMAC server secret is generated/configured at startup; reuse existing
    config plumbing.
  - `CachedResolution` may need to carry its footprint so lookups can apply the
    revocation-validation path when enabled.
- **Optional opt-in validation** for zero revocation window, and **optional
  lazy per-registry probing** for auto-detecting public custom registries — both
  behind config, both addable after the base lands.
- **Lockfile/frozen path routing.** Fresh frozen-lockfile reuse and verifier-only
  paths can avoid a full resolve, but pnpr and its clients must still honor
  tarball route policy. A lockfile that contains private/unknown upstream URLs
  should be rerouted through pnpr or rejected for a fresh server-routed
  resolution instead of being materialized directly by the client.

Risk areas: the footprint must be complete (a missed private fetch would
mis-classify a resolution as public), so the recording must cover every metadata
fetch path, resolve-time tarball fetches (including direct HTTP tarball
dependencies fetched for manifest/integrity), auth-blind metadata mirror fast
paths, and uplink fallback ordering. Tests should cover: unscoped npmjs packages are
fetched without upstream auth and hit the shared cache even when the request has
a pnpr caller identity; private pnpr-hosted packages require caller package
access, including when the client registry points at pnpr itself; private
proxied packages require an authorized upstream alias and never use
client-forwarded upstream auth; mixed public/private resolves use the correct
metadata namespace per package fetch; invalid or unauthorized access misses and
does not reuse private metadata mirrors; `401`/`403` and private-route `404`
responses do not fall back to broader mirrors; inline URL credentials are
rejected before fetch; verifier metadata fetches cannot read or populate the
wrong cache scope; different pnpr users authorized for the same upstream
credential alias share resolution and metadata cache entries; revoked
alias/package access stops matching private hits; public tarballs can keep
upstream URLs while private and unknown tarballs are emitted as pnpr gateway
URLs; cached private resolutions do not replay private upstream tarball URLs;
lockfile/frozen paths do not bypass tarball route policy; custom private default
registry is not treated as public; per-base-key candidate lists stay bounded.

## Prior Art

- **Verdaccio / Artifactory / corporate proxies** terminate upstream
  credentials at the proxy and apply per-package access control, but they cache
  package *metadata and tarballs* per upstream rather than caching whole
  *resolutions*; pnpr's resolution cache is a higher-level artifact that needs
  its own authorization model.
- **HTTP caching with `Vary`** expresses "this cached response depends on these
  request attributes." The footprint key is the same idea specialized to
  access descriptors: public responses do not vary by auth; private responses
  vary by the descriptor and policy that can read them.
- **npm scopes** are often assumed to equal privacy; this RFC deliberately does
  not rely on that, because a private default registry makes unscoped packages
  private and a public registry makes scoped packages public.

## Unresolved Questions and Bikeshedding

- **Granularity of the private key:** HMAC over the union of all private access
  descriptors in the footprint (simple; an entry is shared only by callers that
  select or satisfy the identical private descriptors) vs.
  per-`(registry, scope)` sub-keys
  (more sharing across callers who overlap on some but not all private scopes,
  at higher complexity). Start with the union.
- **Config surface for declaring route policy:** a list of public/private URLs,
  per-scope/package patterns, a per-uplink `public: true` flag, or a combination.
  Should `registry.npmjs.org` unscoped packages be the only built-in public
  route, or should other well-known public mirrors be included?
- **Config surface for team credentials:** upstream credential aliases need
  route policy, secret source, generation/rotation metadata, and caller access
  policy. Should access be inline user lists, named groups/teams, reused package
  permissions, or all of the above?
- **Default for revocation validation:** rely on the short TTL (proposed) vs.
  validate-on-private-hit by default. The latter is safer but adds a round trip
  to every private hit.
- **Read-only install gateway shape:** exact pnpr tarball URL shape, cache
  headers, and feature flags for enabling the resolver's read-only gateway
  without enabling the full npm registry API.
- **Metrics:** should pnpr expose cache hit/miss split by public vs. private
  footprint and tarball routing decisions so operators can see the recovered hit
  rate and gateway load?
