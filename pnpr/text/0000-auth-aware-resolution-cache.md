# Authorization-aware resolution cache for pnpr

## Summary

pnpr disables its resolution cache for any request that forwards upstream
credentials, which makes authenticated installs ~2.8× slower than anonymous
ones even when every dependency is public. This RFC proposes making the cache
*authorization-aware* instead of all-or-nothing: a resolution is classified by
the set of non-public registries it touched (its **private footprint**), public
resolutions are shared globally, and private resolutions are keyed by the
credential that produced them so an invalid or absent credential can never
unlock them.

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

### Part 1 — Classify by registry, fail-safe to private

Privacy is a property of the **registry a package is served from**, not of the
package name and not of whether the name is scoped. This matters in both
directions:

- **Scoped names can be public** — `@babel/core` from the public registry is
  public.
- **Unscoped names can be private** — if a deployment's *default* `registry` is
  a self-hosted/corporate one (Verdaccio, Artifactory, GitHub Packages, a
  private proxy), then unscoped packages served from it can be fully private.

Therefore classification is done **per registry against a known-public
allowlist**, and the default is private:

- A registry is treated as **public only if it is known-public** — a built-in
  entry for `registry.npmjs.org`, plus any registries an operator explicitly
  declares public in pnpr config.
- **Every other registry is treated as private** — scoped or unscoped, default
  or named — unless proven public.

A resolution's **private footprint** is the set of non-public `(registry,
scope)` pairs whose metadata was actually fetched during that resolve. This is
derived from information pnpr already has during the single resolve it already
performs — **no per-package probing and no extra requests.** During resolution
pnpr knows, for every metadata fetch, which registry/scope it targeted; the
footprint is the subset of those not on the public allowlist.

A footprint that is **empty** means the resolution is fully public.

> Note on over-counting: because pnpm forwards the npm token even to the public
> registry, "an `Authorization` header was attached" is *not* a privacy signal —
> it would mark ordinary public installs as private. Classification is by
> registry identity against the allowlist, never by the presence of a forwarded
> header. The public registry is public because it is on the allowlist, not
> because it is the request's default.

### Part 2 — Key public entries globally, private entries by credential

The cache key behavior depends on the footprint:

- **Empty footprint (public):** the key excludes auth, exactly as today. The
  entry is shared globally across all callers. *This is what recovers the
  ~2.8×.*
- **Non-empty footprint (private):** the key additionally incorporates an HMAC
  (keyed with a server secret, so the key is not correlatable offline) of the
  credentials for the private `(registry, scope)` pairs in the footprint.

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
  clients.
- CI / teams using the same token: shared — repeated installs hit the cache.
- Two users with *different* valid tokens to the same private registry: do **not**
  share each other's private entries (different HMACs). This is a small,
  deliberate loss of sharing in exchange for the invariant.
- Self-hosted deployments whose *public* default registry is not on the
  allowlist: treated as private until the operator declares it public — a
  one-line config change recovers global sharing.

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
metadata requests, defeating the performance goal, and privacy is a per-registry
property so per-package probing is the wrong granularity anyway.

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

All changes are in `pnpr/` (the registry server). No pnpm/pacquet CLI changes
are required; the wire protocol is unchanged.

- **Record the per-fetch registry/scope during resolution.** The resolve path
  (`pnpr/crates/pnpr/src/resolver/resolve.rs` and the `pacquet-package-manager`
  resolution observer) must surface, for each metadata fetch, the `(registry,
  scope)` it targeted, so the footprint can be assembled. This is the only piece
  that reaches into the shared pacquet resolver; it can likely ride on the
  existing `ResolutionObserver` rather than changing resolution logic.
- **Known-public allowlist config.** Add a registry classification to pnpr's
  config (`pnpr/crates/pnpr/src/config.rs`): a built-in `registry.npmjs.org`
  entry plus an operator list of registries declared public. Default unknown →
  private.
- **Footprint + keying in the cache layer**
  (`pnpr/crates/pnpr/src/resolver.rs`):
  - Replace the `auth_headers.is_empty()` gate so a cache key is always
    computed.
  - After a resolve, compute the footprint from the recorded registries; if
    empty, store/lookup under the current public key; if non-empty, fold an
    HMAC of the relevant credentials into the key.
  - The HMAC server secret is generated/configured at startup; reuse existing
    config plumbing.
  - `CachedResolution` may need to carry its footprint so lookups can apply the
    revocation-validation path when enabled.
- **Optional opt-in validation** for zero revocation window, and **optional
  lazy per-registry probing** for auto-detecting public custom registries — both
  behind config, both addable after the base lands.

Risk areas: the footprint must be complete (a missed private fetch would
mis-classify a resolution as public), so the recording must cover every metadata
fetch path including uplink fallback ordering. Tests should cover: public-only
authenticated install hits the shared cache; private install is isolated by
credential; invalid credential misses; same-token reuse hits; custom private
default registry is not treated as public.

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
- **Config surface for declaring public registries:** a list of URLs, a per-
  uplink `public: true` flag, or both. Should `registry.npmjs.org` be the only
  built-in, or should other well-known public mirrors be included?
- **Default for revocation validation:** rely on the short TTL (proposed) vs.
  validate-on-private-hit by default. The latter is safer but adds a round trip
  to every private hit.
- **Metrics:** should pnpr expose cache hit/miss split by public vs. private
  footprint so operators can see the recovered hit rate?

---
Written by an agent (Claude Code, claude-opus-4-8).
