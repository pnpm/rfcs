# Authorization-aware resolution cache for pnpr

## Summary

pnpr disables its resolution cache for any request that forwards upstream
credentials, which makes authenticated installs ~2.8× slower than anonymous
ones even when every dependency is public. This RFC proposes making the cache
*authorization-aware* instead of all-or-nothing: each fetch route is classified
once, before the request is sent. Known-public routes (notably unscoped packages
on the public npm registry) are fetched anonymously and shared globally. Routes
that match configured private/auth rules are fetched with the selected
credential and added to the resolution's **private footprint**; private
resolutions are then keyed by the credential that produced them so an invalid or
absent credential can never unlock them.

## Motivation

pnpr's resolution cache stores the result of a dependency resolution keyed by
the resolution inputs (registry, named registries, overrides, project
manifests, lockfile, trust policy, …). On a cache hit it returns the resolved
lockfile without re-running resolution, which is the single biggest speedup pnpr
offers over resolving locally.

The cache is **disabled the moment any credential is forwarded** with the
request (`pnpr/crates/pnpr/src/resolver.rs`):

```rust
let resolution_cache_key = if request.auth_headers.is_empty() && request.lockfile.is_none() {
    resolution_cache_key(config, &request)
} else {
    None
};
```

The reason is correct: a resolution computed with a token that can see a private
package *contains* that package's versions and tarball URLs, and the cache key
deliberately omits auth, so caching it globally could serve one user's private
resolution to another. Skipping the cache is the only safe behavior given the
current key.

But it is far too blunt. The measured impact (from
[pnpm/pnpm#12604](https://github.com/pnpm/pnpm/issues/12604)):

| Scenario (hot cache/store) | Time | Speedup vs. resolving directly |
| --- | --- | --- |
| Anonymous resolve | 0.523 s | ~2.3× faster |
| Authenticated resolve | 1.484 s | ~1.0× (no speedup) |

The crucial observation is that **most authenticated installs are 100% public.**
A pnpm client forwards the user's npm token even when every dependency comes
from the public registry, so the request "looks authenticated" and loses the
cache despite touching nothing private. We are paying the full cost of the
privacy guarantee on requests that have no private data at all.

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

- **Built-in public npm route:** unscoped packages served from the official npm
  registry (`registry.npmjs.org`) are always public. The npm docs state that
  [unscoped packages are always public](https://docs.npmjs.com/about-scopes/)
  and that private packages are user- or organization-scoped, so pnpr should
  deliberately omit forwarded auth for these metadata requests even if the
  client sent a registry-wide npm token.
- **Configured public routes:** an operator may explicitly declare additional
  registries, scopes, or package patterns public. These are fetched without
  forwarded auth and can participate in the global public cache.
- **Private/auth routes:** if a route matches a configured private rule or the
  auth lookup selects a credential for its registry/scope/package, pnpr sends
  that credential and treats the route as private. There is no anonymous
  preflight; the credentialed request is the only request.
- **Unknown routes:** default to private unless explicitly declared public. If
  no credential is available for an unknown/private route, pnpr may resolve it
  anonymously, but the resulting resolution must not be stored in the global
  public cache.

A resolution's **private footprint** is the set of private route identities
whose metadata or resolve-time tarball data was actually fetched during that
resolve, paired with the exact credential selected for each route. This is
derived from information pnpr already has during the single resolve it already
performs — **no per-package probing and no extra requests.** During resolution
pnpr knows, for every fetch, which route it targeted and whether the route was
sent anonymously or with a credential.

A footprint that is **empty** means the resolution is fully public.

> Note on over-counting: because pnpm forwards the npm token even to the public
> registry, "an `Authorization` header was attached" is *not* a privacy signal —
> it would mark ordinary public installs as private. Classification is by route
> policy and the credential actually selected for that route. In particular,
> unscoped packages on the official npm registry are fetched without auth, so a
> broad npm token does not pessimize the common unscoped public path.

### Part 2 — Key public entries globally, private entries by credential

The cache key behavior depends on the footprint:

- **Empty footprint (public):** the key excludes auth, exactly as today. The
  entry is shared globally across all callers. *This is what recovers the
  ~2.8×.*
- **Non-empty footprint (private):** the key additionally incorporates an HMAC
  (keyed with a server secret, so the key is not correlatable offline) of the
  exact credentials selected for the private routes in the footprint.

Why HMAC-of-credential rather than "does the caller have a token for this
scope?" — because the latter is an authorization bypass. A caller could send a
**bogus** token for `@acme` and be served a private resolution it cannot
actually access. We do not trust the claim; we require the caller to *reproduce
the working credential*:

- Caller presents an **invalid** token → its HMAC differs from the HMAC of the
  token that produced the entry → **cache miss** → the request falls through to
  a real resolve using the invalid token, which the **upstream registry itself
  rejects** (401/403). No cached private data is served.
- Caller presents the **valid** token → it holds a credential the registry
  honors, which *is* access by definition → serving the cache is correct.

We never separately answer "does this caller really have access?" The registry
is the source of truth for access; the key simply requires the caller to present
the exact credential, and an invalid one cannot match.

**Safety invariant:** a private-footprint entry is keyed by the same credential
that produced it, so by construction it contains only what that credential could
fetch. It can never leak beyond that credential.

### Consequences for sharing

- Public installs: shared globally — full cache benefit, even for authenticated
  clients, as long as the packages route through public/anonymous fetch rules.
- CI / teams using the same token: shared — repeated installs hit the cache.
- Two users with *different* valid tokens to the same private registry: do **not**
  share each other's private entries (different HMACs). This is a small,
  deliberate loss of sharing in exchange for the invariant.
- Scoped public packages on npmjs with a broad registry token: if the route
  policy selects that token, pnpr treats the route as private and does not share
  globally. Operators can recover sharing for known-public scopes/package
  patterns by declaring them public, which also makes pnpr omit auth for those
  fetches.
- Self-hosted deployments whose *public* default registry is not covered by a
  public route rule: treated as private until the operator declares it public —
  a one-line config change recovers global sharing.

### Revocation window

A token revoked upstream still matches its HMAC until the entry's TTL expires
(already short — `packument_ttl`, ~5 min by default), during which a private hit
serves data the token could see moments earlier. For deployments that want zero
window on private hits, an **opt-in** lightweight validation — a single
authenticated request (one round trip, not a full re-resolve) — can run before
serving a private hit. The default relies on the short TTL.

### Optional: auto-detecting public custom registries

Operators who do not want to declare a public self-hosted registry can opt into
lazy classification **at the registry/scope granularity** (never per package):
attempt the first fetch to an unknown registry anonymously, attach auth only on
`401/403` (or `404` when auth is available, since some registries hide private
packages as 404), and cache the `(registry, scope) → requires-auth` verdict with
a TTL. Cost: at most one extra round trip per new private registry-scope per TTL
window, and zero for the public path. This is a later refinement; the base
design is config-driven with no probing.

## Rationale and Alternatives

**Alternative A — Per-package anonymous probing.** Classify each package by
fetching it both anonymously and authenticated. Rejected: it roughly doubles
metadata requests, defeating the performance goal. This RFC instead uses a
single route-policy decision per fetch.

**Alternative B — Always hash auth into the cache key.** Simple and safe, but it
gives *every* authenticated request its own cache namespace, so the dominant
public case never shares across users — it only helps repeat installs by the
same caller. This RFC's footprint split keeps public resolutions globally shared
while applying credential-keying only where it is actually needed.

**Alternative C — "Caller has a token for the scope" gate.** Share a private
entry with any caller that presents some credential for the registry. Rejected:
it is an authorization bypass (an invalid token unlocks private data). The
HMAC-of-credential key closes this.

**Alternative D — Server-side encrypted credential vault.** Store upstream
credentials in pnpr (clients authenticate only to pnpr) and encrypt them at
rest. This is a worthwhile, separate feature for credential *management* and
*transmission*, but it does **not** fix caching on its own: a vaulted token
produces exactly the same scope-specific resolution as a forwarded one, so the
cache would still have to be gated by access. Encrypting the cached resolutions
themselves would actively *defeat* sharing. The vault is complementary and out
of scope here; this design composes with it (a vault could supply the stable
credential identity used in the private key).

**Alternative E — Do nothing.** Authenticated installs stay ~2.8× slower. Given
that forwarding the npm token to a public registry is the pnpm default, this
penalizes the common case for no privacy benefit.

This proposal is the minimal change that recovers the common-case performance
without weakening the privacy guarantee, and it stands alone regardless of
whether a credential vault is later added.

## Implementation

No pnpm wire-protocol change is required. The implementation does need changes
in pnpr's resolver integration and in the pacquet fetch path that chooses
metadata/tarball auth.

- **Route policy + auth selection.** Add route classification to pnpr config
  (`pnpr/crates/pnpr/src/config.rs`): built-in anonymous-public handling for
  unscoped packages on `registry.npmjs.org`, plus operator rules for public and
  private registries/scopes/package patterns. The fetch path must choose the
  route policy and selected auth together, using the same package-aware lookup
  that will set the `Authorization` header. Public routes suppress forwarded
  auth; private/auth routes send the selected credential and record it for the
  footprint.
- **Record the per-fetch route during resolution.** The resolve path
  (`pnpr/crates/pnpr/src/resolver/resolve.rs`) and the pacquet npm/tarball
  fetchers must surface, for each metadata fetch and resolve-time tarball fetch,
  the route identity, whether it was public or private, and the selected
  credential digest. The existing `ResolutionObserver` reports resolved tarball
  packages after a resolver result, so it is not sufficient by itself; the hook
  needs to sit at the actual fetch/auth-selection layer.
- **Make lower metadata caches respect the same classification.** Pacquet's
  shared metadata mirror is currently keyed by registry/package, not by auth.
  Private/auth metadata must not be loaded from or written to a global
  auth-blind mirror in a way that lets a later invalid credential reuse it. The
  same route policy should either key private metadata by credential digest or
  bypass/fail closed for private metadata cache hits.
- **Footprint + keying in the cache layer**
  (`pnpr/crates/pnpr/src/resolver.rs`):
  - Replace the `auth_headers.is_empty()` gate so a cache key is always
    computed.
  - Keep the current resolution-input key as the base cache key, but store one
    or more candidate entries under it. Each candidate carries its private
    footprint and the HMAC digest of the credentials that produced it. A public
    candidate has an empty footprint and matches every caller; a private
    candidate matches only when the current request can reproduce the same
    HMACs for that stored footprint. This avoids knowing the footprint before
    the first resolve and avoids a second resolve or anonymous probe.
  - After a resolve, compute the footprint from the recorded routes; store an
    empty-footprint candidate for public resolutions, or a credential-keyed
    candidate for private resolutions.
  - The HMAC server secret is generated/configured at startup; reuse existing
    config plumbing.
  - `CachedResolution` may need to carry its footprint so lookups can apply the
    revocation-validation path when enabled.
- **Optional opt-in validation** for zero revocation window, and **optional
  lazy per-registry probing** for auto-detecting public custom registries — both
  behind config, both addable after the base lands.

Risk areas: the footprint must be complete (a missed private fetch would
mis-classify a resolution as public), so the recording must cover every metadata
fetch path, resolve-time tarball fetches, auth-blind metadata mirror fast paths,
and uplink fallback ordering. Tests should cover: unscoped npmjs packages are
fetched without auth and hit the shared cache even when a registry-wide npm token
is forwarded; private install is isolated by credential; invalid credential
misses and does not reuse private metadata mirrors; same-token reuse hits; custom
private default registry is not treated as public.

## Prior Art

- **Verdaccio / Artifactory / corporate proxies** terminate upstream
  credentials at the proxy and apply per-package access control, but they cache
  package *metadata and tarballs* per upstream rather than caching whole
  *resolutions*; pnpr's resolution cache is a higher-level artifact that needs
  its own authorization model.
- **HTTP caching with `Vary`** expresses "this cached response depends on these
  request attributes." The footprint key is the same idea specialized to
  credentials: public responses do not vary by auth; private responses vary by
  the credential that can read them.
- **npm scopes** are often assumed to equal privacy; this RFC deliberately does
  not rely on that, because a private default registry makes unscoped packages
  private and a public registry makes scoped packages public.

## Unresolved Questions and Bikeshedding

- **Granularity of the private key:** HMAC over the union of all private
  credentials in the footprint (simple; an entry is shared only by callers with
  the identical private credential set) vs. per-`(registry, scope)` sub-keys
  (more sharing across callers who overlap on some but not all private scopes,
  at higher complexity). Start with the union.
- **Config surface for declaring route policy:** a list of public/private URLs,
  per-scope/package patterns, a per-uplink `public: true` flag, or a combination.
  Should `registry.npmjs.org` unscoped packages be the only built-in public
  route, or should other well-known public mirrors be included?
- **Default for revocation validation:** rely on the short TTL (proposed) vs.
  validate-on-private-hit by default. The latter is safer but adds a round trip
  to every private hit.
- **Metrics:** should pnpr expose cache hit/miss split by public vs. private
  footprint so operators can see the recovered hit rate?

---
Written by an agent (Claude Code, claude-opus-4-8).
