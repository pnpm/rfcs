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
upstream packages, that descriptor is always a pnpr-managed uplink, not a
client-forwarded token. For pnpr-hosted packages, the descriptor is the hosted
route/access-policy identity and hits re-run package authorization for the
caller. Client auth identifies the caller to pnpr and may authorize packages
hosted by pnpr itself, but it is not forwarded to third-party registries and is
not digested as an upstream cache credential.

The registries pnpr will fetch from server-side are an **allowlist** — the
built-in npm route, operator-declared public routes, configured uplinks, and
pnpr itself — and a client `registry`/`namedRegistries` matching none of them is
rejected before any fetch. Being default-deny rather than denylisting dangerous
hosts closes the resolver's server-side request-forgery surface (a caller can no
longer point pnpr at cloud instance metadata or an internal service), and it
removes the "unknown route" entirely: every resolved route is either public
(globally shared) or carries a private access descriptor, so the cache has just
two states with no non-shareable tier.

The same route decision governs the tarball URLs in resolver output: public
routes keep their direct upstream (CDN) tarball URLs, while private proxied
routes are served through pnpr's **per-uplink registry endpoints**
(`https://<pnpr>/~<uplink>`) that the client points the corresponding scope at.
Because those URLs are canonical for the configured registry, lockfile entries
collapse to integrity-only registry resolutions whose host lives in client
config, not in the lockfile — so a project produces the **same lockfile whether
it is resolved server-side through `/resolve` or client-side directly against
the same registry configuration**. Server-side resolution becomes a pure
accelerator, never a fork in the lockfile, and upstream credentials and unusable
private upstream URLs never reach clients.

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
identity, selected pnpr-managed uplinks, and lockfile presence are inputs to
auth-aware cache lookup, not reasons to skip it. Lockfile-seeded update resolves
must still participate in the cache: the base resolution-input key includes the
input lockfile and the lockfile mode flags (`frozenLockfile`,
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
are pnpr-managed uplinks selected by route policy. Either way, caller
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

Every registry pnpr will fetch from server-side is an **allowlist** derived from
the operator's configuration: the built-in npm route, operator-declared public
routes, configured uplinks (and their `/~<uplink>/` endpoints), and pnpr-hosted
registries. A client `registry`/`namedRegistries` whose origin matches none of
these is **rejected at the request boundary, before any fetch**: pnpr is
default-deny, so a caller cannot direct it at an arbitrary host (see
[Fetch allowlist and SSRF](#fetch-allowlist-and-ssrf)). There is consequently no
"unknown route" to resolve anonymously and no non-shareable cache state — every
resolved route is either public or carries a private access descriptor.

The base design must not probe every package twice. For each allowed metadata or
resolve-time tarball fetch, classification chooses **one** request shape:

- **Built-in npm route:** the official npm registry (`registry.npmjs.org`) is a
  built-in public route — always allowlisted and fetched anonymously, with no
  configuration required and ahead of any uplink credential for the same origin
  (public wins). An anonymously-successful fetch is public and globally
  shareable — including scoped names. The
  [npm scope docs](https://docs.npmjs.com/about-scopes/) note that private
  packages are organization-scoped, but a *private* scoped npm package 404s on an
  anonymous fetch (pnpr forwards no client credential), so it is never resolved
  this way; a private npm dependency must be fronted by a uplink. The caller's
  pnpr identity is irrelevant to this public route.
- **Configured public routes:** an operator may declare additional registries,
  scopes, or package patterns public. These are fetched without upstream auth and
  participate in the global public cache.
- **Private proxied routes:** a route whose origin matches a configured uplink
  the caller is authorized for is fetched with that uplink's server-owned
  credential — never a client-forwarded one — and recorded under the uplink's
  private access descriptor. There is no anonymous preflight and no
  client-credential fallback.
- **Private pnpr-hosted routes:** if the package is hosted by pnpr itself, the
  client auth identifies the caller to pnpr and is checked against pnpr's package
  access policy. It is not forwarded upstream and is not used as the cache
  credential digest.
- **Inline URL credentials:** dependency URLs or resolved tarball URLs that
  contain `user:pass@host` credentials are rejected by the resolver. They must
  not be converted into Basic auth, used as an upstream credential identity, or
  stored in any shared cache. Operators that need authenticated direct tarball
  URLs should model that host with a pnpr-managed uplink instead.

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
read their manifest and integrity. Each route is paired with the exact
**private access descriptor** selected for it. A descriptor is the cache
namespace input plus the authorization predicate for that private route:

- proxied routes derive the descriptor from the selected pnpr-managed uplink
  plus a digest of its credential, and its gate is "the caller may still use this uplink";
- pnpr-hosted routes derive the descriptor from the hosted route/access-policy
  identity, and its gate is "the caller still satisfies this package policy".

The descriptor abstraction is shared; the difference is only how the descriptor
is derived and checked. Hosted descriptors are not keyed per caller, and they are
not keyed by a bearer token.

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

A private access descriptor is one abstraction with two derivation sources:

- **Proxied route:** descriptor key input = `uplink + credential-digest`; gate
  = the caller can still select that uplink for the route.
- **pnpr-hosted route:** descriptor key input = hosted route/access-policy
  identity; gate = pnpr re-runs package access policy for the caller.

Lookup still happens before resolution. pnpr first computes the normal
resolution-input key, then reads the candidate list stored under that base key.
Each candidate is matched against the private access descriptors the current
request can select or satisfy for that candidate's private footprint. Public
candidates match every caller. Private candidates match only when the current
request can reproduce the same descriptor key inputs and satisfy each
descriptor's authorization gate.
The footprint discovered during resolution is used only when writing a new
candidate after a miss, not for finding the first candidate set.

Why key and gate by private access descriptor rather than "does the caller have
some auth for this scope?" — because the latter is an authorization bypass. A
bogus client token for `@acme` must never unlock a private proxied resolution.
Proxied packages therefore do not use client-forwarded upstream auth at all:

- If the caller is authorized for a **pnpr-managed uplink**, the request selects
  that uplink identity and can match entries produced by the same uplink.
- If the caller is not authorized for the uplink, the request cannot select that
  uplink identity, so it cannot match the uplink's private entries.
- If the package is **hosted by pnpr itself**, the request is matched by
  re-running pnpr package access policy for the caller against the hosted package
  routes in the footprint. The client's bearer token authenticates the caller; it
  is not the upstream cache credential.

The cache key requires the current request to reproduce the exact private access
descriptor that produced the entry and pass that descriptor's authorization
gate. For proxied entries that means selecting the same authorized uplink
with an unchanged credential. For pnpr-hosted entries that means satisfying the same package route
policy at lookup time. An invalid, absent, or unauthorized credential cannot
match.

**Safety invariant:** a private-footprint entry is keyed by the same private
access descriptor that produced it. pnpr-managed uplink entries are gated by
uplink access policy, and pnpr-hosted entries are gated by pnpr package access
policy. Private data can never leak beyond the uplink authorization or package
access policy that produced it. The design does not rely on hiding whether a
base resolution key has private candidates; unauthorized callers always observe
a miss/fail-closed resolve, candidate counts and footprint details must not be
returned to clients, and private hit/miss metrics should be operator-only.

### Fetch allowlist and SSRF

`/-/pnpr/v0/resolve` and `/-/pnpr/v0/verify-lockfile` fetch package metadata
server-side from the registries the client supplies (`registry` and every
`namedRegistries` alias). If those URLs are honored verbatim, an authenticated
caller can point them at the link-local range that fronts cloud instance
metadata (`http://169.254.169.254/`), at an internal service, or at any other
host reachable from the server's network position — server-side request forgery,
including the classic IMDS credential-theft target.

The allowlist closes this at the source. Before any server-side fetch, pnpr
rejects a `registry`/`namedRegistries` whose origin matches no allowlisted route
— the built-in npm route, an operator-declared public route, a configured uplink
origin (or its `/~<uplink>/` endpoint), or pnpr itself. Because the set is
**default-deny**, a caller cannot make pnpr issue a request to a host the
operator did not approve: IMDS, internal services, and any future
metadata endpoint are all unreachable without a config change, rather than
patched one address range at a time.

Registries are not the only fetch trigger: a resolve also follows **direct-URL
dependencies** — an `http(s)` tarball spec or a `git`/`git+…` URL — through
pacquet's tarball and git resolvers. The same boundary therefore rejects any
`http(s)`/`git` URL appearing in a dependency spec, an `overrides` leaf, or an
input lockfile's `resolution.tarball` whose origin is off the allowlist (a `git+`
transport prefix is stripped before the origin check). A semver range or
`npm:`/`workspace:`/`file:`/`link:` alias never reaches the network and is
ignored. A direct URL dependency consequently requires the operator to allowlist
its origin as a public route, the same as any registry — the deliberate
default-deny posture, extended to every URL that can cause a server-side fetch.

This is strictly stronger than denylisting dangerous hosts (link-local ranges,
known metadata hostnames): a denylist is default-allow, so every new SSRF target
is a gap, and it must deliberately keep private/loopback ranges reachable
(internal registries are pnpr's core use case) — exactly the ranges an attacker
would target. The allowlist instead lets the operator reach an internal registry
by *allowlisting that specific registry*, while everything else stays blocked.

Two residuals remain, both closed by the same **connect-time guard** in the HTTP
client's connector — a separate hardening step:

- **DNS rebinding.** An allowlisted *hostname* that resolves to a link-local or
  internal address only at connect time is not caught by an origin-level
  allowlist.
- **Transitive dependencies.** The request-boundary check validates the
  *client's* `registry`/`namedRegistries` and direct-URL dependency specs. But a
  package resolved from an allowlisted registry can declare a *transitive*
  dependency on an off-allowlist `http(s)`/git URL, which the resolver fetches
  server-side during the tree walk — outside the request boundary. (This is a
  mostly-blind SSRF: pnpr fetches the URL but does not echo the response to the
  caller.)

A connector-level guard that filters the actual connect target IP (blocking
link-local/private/metadata ranges, including IP-literal URLs that skip DNS)
closes both for `http(s)` fetches at once; git remotes — resolved via the `git`
binary rather than the shared client — need their own gate. The origin-level
allowlist still removes the far larger surface of arbitrary caller-named hosts
and every direct-input vector.

### Team-owned upstream credentials (pnpr-managed uplinks)

For third-party/proxied packages, pnpr should not forward arbitrary client auth
to the upstream registry. The resolver should use the same proxy shape as the
registry server: clients authenticate to pnpr, and pnpr uses operator-managed
upstream credentials when the caller is allowed to use them. This RFC unifies
that with Verdaccio's existing **uplink** concept rather than introducing a
separate credential-alias config block: an uplink already names an upstream
registry and its credential; this RFC adds an access policy to it, namespaces its private cache by a digest
of that credential (so a rotation re-keys automatically), and exposes it as a
registry endpoint.

A **pnpr-managed uplink** has:

- a backing upstream registry URL and the upstream credential material (for
  example from an environment variable or secret store);
- an **access policy** saying which pnpr users/teams may use it;
- a **registry endpoint path** (`https://<pnpr>/~<uplink>`) clients can point a
  scope at.

Routing to an uplink is by the **registry origin** a package resolves to, taken
from the request's registry/named-registry configuration — not by a
per-credential package glob. The same scope can therefore resolve through
different uplinks for different clients depending on each client's registry
config, and a single uplink can serve whatever packages its upstream serves.

A credential is attached **only** to fetches whose origin matches the uplink's
own configured origin. A tarball host taken from a packument's `dist.tarball` is
untrusted input and never causes pnpr to send the packument-serving uplink's
credential to that host; see [Split-domain registries](#split-domain-registries-separate-tarball-host).

- callers authenticate to pnpr as they already can with pnpr bearer tokens;
- pnpr, not the client, supplies the uplink's upstream credential when the caller
  is authorized for that origin, and records the uplink identity in the private
  footprint;
- cache keys and metadata mirrors use a stable proxied descriptor `uplink +
  credential-digest`, HMACed with the server secret. The raw token is never
  written to cache keys, embedded in any client-visible URL, or exposed to
  clients.

This gives teams the fast path without allowing client-supplied third-party
tokens to affect proxied resolution. Everyone authorized for the same uplink
shares private resolution entries and private metadata entries. A caller who is
not authorized for the uplink cannot select it and therefore cannot match its
cache entries.

Route selection precedence:

1. Public route policy wins and suppresses auth.
2. A pnpr-hosted private route uses a shared hosted access descriptor, such as
   a team or package access-policy identity, and re-runs that policy for the
   caller at lookup time; no upstream credential is involved.
3. A proxied private route selects an authorized pnpr-managed uplink by registry
   origin.
4. A route matching no allowlisted registry is **rejected** at the request
   boundary (see [Fetch allowlist and SSRF](#fetch-allowlist-and-ssrf)); pnpr
   never forwards client auth to an upstream and never resolves an
   off-allowlist host anonymously. A caller authorized for an uplink origin but
   not for *this caller* still fails closed at that uplink's access gate.

When an upstream credential is rotated, the digest pnpr keys the uplink's
private cache by changes automatically, so new resolves populate a new private
namespace and the old one ages out by TTL — no manual epoch to bump. The digest
is a server-side cache-namespacing detail and is never embedded in a
client-visible URL. If a user's team access is revoked, pnpr re-evaluates uplink
authorization before serving private hits, so the caller stops matching those
private entries even while the shared upstream token itself remains valid for
other team members.

### Tarball URL routing and lockfile parity

Resolution cache entries include tarball URLs, so the URL routing decision is
part of the same route policy. But the routing must not make the lockfile depend
on *how* it was produced: a project must resolve to the same lockfile whether it
went through pnpr's `/resolve` accelerator or was resolved client-side directly
against the same registry configuration. Otherwise server-side resolution forks
the lockfile, and a `pnpm add` performed without the accelerator rewrites every
private entry.

The design achieves this by routing private packages through **per-uplink
registry endpoints** rather than opaque per-tarball gateway URLs:

- Each pnpr-managed uplink is exposed as a registry endpoint at a stable path,
  `https://<pnpr>/~<uplink>`. The `~` prefix keeps the endpoint out of the
  package-name path namespace (`/<name>` is a packument), since `~` cannot begin
  an npm package name. It behaves as a normal npm registry: it serves
  packuments (rewriting `dist.tarball` to canonical
  `https://<pnpr>/~<uplink>/<pkg>/-/<file>` URLs, as Verdaccio does) and proxies
  tarball bytes from the backing upstream using the server-owned credential,
  after re-checking caller access to the uplink.
- Clients reach a private route by pointing the corresponding scope's registry
  at that endpoint — `@acme:registry=https://<pnpr>/~<uplink>` — exactly the
  mechanism npm already provides for scoped private registries. This works
  **without** the resolver: ordinary client-side resolution against the endpoint
  yields correctly-routed, credential-free tarball URLs.
- Because the emitted tarball URL is canonical for the configured registry, the
  lockfile entry collapses to an **integrity-only registry resolution**. The
  registry host lives in client `.npmrc`, never in the lockfile. The same
  `{integrity}` entry reconstructs to the pnpr endpoint for a client configured
  to use pnpr, and to the upstream for a client with direct access — so the
  lockfile is identical across both, the same way a project already shares one
  lockfile across a public registry and an internal mirror.

For each selected package, pnpr applies the same route classification when it
emits streamed package frames and the returned lockfile:

- **Public/anonymous route:** emit the upstream `dist.tarball` URL (canonical for
  the public registry → integrity-only). Public tarballs download directly from
  the upstream CDN and never traverse pnpr; the route policy is the proof, with
  no anonymous tarball probe on the hot path.
- **Private proxied route:** emit a URL canonical for the uplink's registry
  endpoint, integrity-only relative to the scope's configured
  `https://<pnpr>/~<uplink>` registry. The endpoint fetches from the upstream with
  the server-owned uplink credential after re-checking caller access.
- **Private pnpr-hosted route:** emit a pnpr-hosted tarball URL (canonical for the
  pnpr registry) and re-check the caller against pnpr package access policy before
  serving it.
There is no "unknown route" tarball case: a package whose metadata could only
come from an off-allowlist registry is rejected during resolution before it
reaches the lockfile, so the emitted set is limited to the three routes above.
pnpr never mints a per-tarball opaque gateway URL.

Resolver responses, streamed package frames, and returned lockfiles must never
include upstream `Authorization` values, upstream tokens, or a raw private
upstream URL solely because pnpr fetched it successfully with server-owned
credentials.

Routing therefore depends on the caller having pointed each private scope at the
matching `/~<uplink>/` endpoint. When a fresh resolve hits a private route whose
scope is instead configured directly at the raw upstream, the resolver cannot
emit a usable credential-free URL for it: it must **fail closed with an
actionable error** naming the scope and the uplink endpoint it should point at,
rather than emitting an upstream URL the credential-less client cannot fetch or
silently forwarding the client's own upstream auth. How clients are provisioned
that scope→endpoint mapping (operator-pushed `.npmrc`, a discovery handshake, …)
is an open question below.

Frozen and lockfile-seeded paths must preserve this routing the same way: if an
existing lockfile contains raw private/unknown upstream tarball URLs that the
client cannot authenticate to, pnpr should return a freshly routed lockfile or
reject it rather than letting the client fetch a private or unknown upstream URL
directly.
Lockfile-seeded update resolves still use the resolution cache; their cache key
includes the input lockfile and lockfile mode flags, and the cached value is the
routed output lockfile for that exact input.

### Split-domain registries (separate tarball host)

Some private registries serve package documents on one origin and the
`dist.tarball` on another (GitHub Packages, AWS CodeArtifact, Artifactory/Azure
with a separate asset CDN). Because pnpr's uplink endpoint rewrites every
`dist.tarball` onto its own host before serving the packument, the
**client-facing** tarball URL stays canonical for the configured registry and
the lockfile entry stays integrity-only — the cross-domain split is invisible to
the client and parity is preserved. (A direct, non-pnpr resolve, by contrast,
records the cross-domain URL as an explicit tarball resolution, so this is one
more case the endpoint normalizes that a host-swap could not.)

The split constrains pnpr's **outbound** behavior, and the constraint is a
security boundary, not just a routing detail:

- **Credentials are sent only to explicitly configured origins.** A packument's
  `dist.tarball` domain is untrusted data: a compromised, misconfigured, or
  malicious upstream could point it at an attacker-controlled host. pnpr must
  **never** infer "this tarball belongs to the uplink that served the packument,
  so attach that uplink's credential." An uplink's credential is attached only to
  fetches whose origin matches that uplink's own configured origin.
- **The operator configures a differing tarball origin explicitly.** When a
  registry's tarball host differs from its packument host, the operator must
  declare that tarball origin — as a separate uplink, or as an additional origin
  on the same uplink — with whatever auth it needs (often none, for
  pre-signed/time-limited asset URLs). pnpr selects the credential for the tarball
  fetch by the tarball's **own** origin against that explicit config.
- **Unconfigured tarball origins get no credential and fail closed if they need
  one.** If the tarball origin is not covered by explicit configuration, pnpr
  fetches it without any uplink credential; if that origin then returns
  `401`/`403`, the fetch fails closed and the operator is expected to configure
  it. pnpr does not retry with the packument uplink's credential. pnpr should
  also refuse to proxy tarball origins that are neither the registry origin nor an
  explicitly configured/allowlisted origin, so a malicious packument cannot turn
  the endpoint into an open relay to arbitrary or internal hosts.
- **Footprint attribution follows the fetch's own origin.** A cross-domain
  tarball fetch contributes the configured uplink's descriptor when one matches,
  or no private descriptor when it is an anonymous asset fetch. An anonymous
  tarball fetch does not make the resolution public — the packument fetch already
  recorded the private route that makes the resolution private.

Time-limited/signed asset URLs (CodeArtifact, Artifactory) must not be pinned in
the mirror past their validity: serving a split-domain tarball honors metadata
freshness and re-derives the asset URL from a fresh packument when needed.

As with any non-canonical upstream URL, the only residual divergence is a
**mixed** deployment where some clients go through pnpr and some resolve directly
against the upstream: pnpr clients get integrity-only entries while direct
clients get the explicit cross-domain URL. A homogeneous "everyone through pnpr"
deployment is fully consistent.

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

Because off-allowlist routes are rejected before any fetch, there is no
"descriptor-less private" metadata to handle — every fetched route is public
(global mirror) or carries a descriptor (its own namespace). No request-local
bypass tier is needed.

The metadata cache scope is **per metadata fetch**, not per resolve. A single
workspace resolve may read public npmjs metadata from the global mirror, private
metadata for uplink A from uplink A's descriptor namespace, private metadata for
uplink B from uplink B's namespace, and pnpr-hosted metadata from a hosted-policy
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
  Public tarballs download directly from upstream CDN URLs and never traverse
  pnpr.
- CI / teams using a pnpr-managed uplink: shared across every pnpr user
  authorized for that uplink, without exposing the raw upstream token to clients.
  Private tarballs are served through the uplink's registry endpoint, not by
  sending upstream tokens or raw private upstream URLs to clients. Lockfile
  entries stay integrity-only, so they are identical with or without server-side
  resolution.
- pnpr-hosted private packages: shared across callers who currently satisfy the
  hosted access descriptor recorded in the footprint. The descriptor should be
  policy- or team-scoped where possible so a team shares cache entries; client
  tokens authenticate callers to pnpr, but they are not cache keys.
- Scoped public packages on npmjs with an authenticated pnpr request: still
  public — the npmjs host is allowlisted and the anonymous fetch succeeds. Client
  auth to pnpr is not forwarded upstream and does not make the entry private.
- Self-hosted deployments whose default registry is not yet allowlisted: every
  request to it is **rejected** until the operator declares it (a public route to
  share its public packages globally, or a uplink to serve it privately) — a
  one-line config change. This is the deliberate default-deny posture; nothing is
  fetched from an unapproved host.

### Revocation window

For pnpr-managed uplinks, rotating the upstream credential immediately moves future
hits and writes to a new namespace; the old namespace ages out. pnpr-hosted
package access is re-evaluated before serving a pnpr-hosted private hit. For
deployments that want zero upstream revocation window, an **opt-in** lightweight
validation — a single authenticated request (one round trip, not a full
re-resolve) — can run before serving an upstream private hit. The default relies
on the short TTL plus per-hit uplink/package authorization checks.

### No registry auto-detection

An earlier draft proposed lazily probing an unknown registry anonymously to
learn whether it is public. The allowlist removes the need: a registry pnpr will
fetch from is always operator-declared (a public route or a uplink), so there is
no unknown registry to probe and no `(registry, scope) → requires-auth` verdict
to cache. Adding a registry is a one-line config change, not a runtime guess —
and runtime probing of caller-named hosts is exactly the SSRF surface the
allowlist exists to remove.

## Rationale and Alternatives

**Alternative A — Per-package anonymous probing.** Classify each package by
fetching it both anonymously and with the selected upstream credential.
Rejected: it roughly doubles metadata requests, defeating the performance goal.
This RFC instead uses a single route-policy decision per fetch.

**Alternative B — Always vary by auth or credential.** Simple and safe, but it
gives *every* authenticated caller or selected upstream credential its own cache
namespace, so the dominant public case never shares across users — it only helps
repeat installs by the same caller or uplink. It also hashes the wrong thing for
this design: client auth is a caller identity for pnpr-hosted packages, not an
upstream credential for proxied packages. This RFC's footprint split keeps
public resolutions globally shared while applying private-descriptor keying only
where it is actually needed.

**Alternative C — "Caller has a token for the scope" gate.** Share a private
entry with any authenticated caller or any caller that presents some credential
for the registry. Rejected: it is an authorization bypass. Client auth to pnpr
authorizes only pnpr-hosted package access; proxied package access requires the
caller to be authorized for the selected uplink. Keying by uplink identity plus
access-policy checks closes this.

**Alternative D — Server-side encrypted credential vault.** Store upstream
credentials in pnpr (clients authenticate only to pnpr) and encrypt them at
rest. This is a worthwhile, separate feature for credential *management* and
*transmission*, but it does **not** fix caching on its own: the cache still has
to be gated by the selected uplink identity and caller access. Encrypting the
cached resolutions themselves would actively *defeat* sharing. The pnpr-managed
uplink feature in this RFC is the minimal useful slice of that vault idea for
performance: it supplies a stable proxied descriptor and an access policy, while
full credential lifecycle management can remain a later extension.

**Alternative E — Proxy all tarballs through pnpr.** Route *every* tarball,
public and private, through a pnpr endpoint. Rejected as the default because
public packages dominate many installs, and proxying all public tarball bytes
through pnpr increases bandwidth, CPU, storage, and latency on the path this RFC
is trying to keep fast — and forfeits direct upstream-CDN downloads. Only private
and unknown tarballs route through a pnpr uplink endpoint; public tarballs keep
their upstream CDN URLs.

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

**Alternative H — Opaque per-tarball gateway URLs in the lockfile.** Have
`/resolve` rewrite each private/unknown tarball into a bespoke pnpr gateway URL
(for example `/-/pnpr/v0/tarballs/<uplink>/<generation>/<pkg>/-/<file>`, or an
opaque server-keyed token for unknown routes). Rejected: those URLs are
*non-canonical*, so they are written verbatim into the lockfile as explicit
tarball resolutions. That forks the lockfile — a project resolved through
`/resolve` gets gateway URLs while the same project resolved client-side gets
upstream URLs — breaks `pnpm add`/update performed without the accelerator,
bakes the rotation generation into committed URLs, and requires a stateful
key→URL map on the server for the unknown case. The chosen design instead
exposes each uplink as a *canonical* registry endpoint and keeps lockfile entries
integrity-only, so the lockfile is identical across modes, rotation stays
server-side, and no per-tarball gateway state is needed.

**Alternative I — Denylist dangerous fetch hosts instead of an allowlist.** Keep
resolving against any registry the client sends, but block known-bad targets
(link-local ranges `169.254.0.0/16` / `fe80::/10`, well-known metadata
hostnames). Rejected as the model, though it remains useful as defense-in-depth.
A denylist is default-allow, so it leaks every host it forgot — and it must
deliberately keep private/loopback ranges reachable (internal registries are
pnpr's core use case), which are exactly the addresses an SSRF attacker targets.
It also leaves the unknown-route resolve path (and its non-shareable cache state)
in place. The allowlist is default-deny: pnpr fetches only operator-approved
registries, which both closes the SSRF surface at the source and removes the
unknown route. (The connect-time DNS-rebinding guard is orthogonal and complements
either model.)

This proposal is the minimal change that recovers the common-case performance
without weakening the privacy guarantee, and it stands alone regardless of
whether a credential vault is later added.

## Implementation

No pnpm wire-protocol change is required: route selection fits in the existing
streamed package frames and returned lockfile URLs, and private routing is driven
by the client's existing scoped-registry configuration pointing at pnpr uplink
endpoints. The implementation does need changes in pnpr's server-side resolver
integration, in the pacquet fetch path that chooses metadata/tarball auth, and in
how pnpr exposes uplinks as registry endpoints.

- **Expose uplinks as registry endpoints.** Each pnpr-managed uplink is served
  under a stable registry path (`https://<pnpr>/~<uplink>`): packument reads
  (with `dist.tarball` rewritten to canonical endpoint URLs) and tarball proxying
  with the server-owned credential, gated by the uplink access policy. This is
  the same registry surface that backs client-side resolution when a scope is
  pointed at the endpoint, so both `/resolve` and direct installs share one code
  path and produce identical, integrity-only lockfile entries.
- **Recognize own `/~<uplink>/` endpoints during resolution.** When a request's
  registry/named-registry config points a scope at `https://<self>/~<uplink>/`,
  the resolver must normalize that to "resolve through the `<uplink>` uplink"
  (its credential, access policy, and footprint descriptor) rather than treating
  the endpoint as an opaque third-party upstream or recursively issuing HTTP
  requests to its own endpoint. This is the uplink-endpoint analogue of the
  hosted-route normalization against pnpr's own base URL, and it is what lets
  `/resolve` emit URLs already canonical for the client's configured endpoint.
- **Merge upstream credentials into uplink config.** Rather than a separate
  credential-alias block, extend pnpr's existing `uplinks` config
  (`pnpr/crates/pnpr/src/config.rs`) with an access policy (which pnpr
  users/teams may use it) and its exposed endpoint path; its private cache is
  namespaced by a digest of the credential, so a rotation re-keys automatically.
  Match an uplink to a fetch by **registry origin**, not by a per-credential
  package glob. The package-pattern routing remains the operator's existing
  `packages`/route-policy concern.
- **Never send credentials to an inferred origin.** Credential selection is by
  the fetch's own origin against explicit uplink config. A tarball origin
  discovered from a packument's `dist.tarball` that does not match a configured
  origin is fetched without any uplink credential, and is refused outright unless
  it is the registry origin or an explicitly allowlisted/configured origin.
  Split-domain registries (separate tarball host) therefore require the operator
  to configure the tarball origin explicitly; pnpr must not attach the packument
  uplink's credential to a different host.
- **Thread pnpr caller identity into resolution.** The HTTP request's resolved
  pnpr identity (`AuthedCaller` / `Identity`) must be passed into
  `serve_resolve`, `handle_resolve`, cache lookup, cache write, route selection,
  and any pnpr-hosted package access checks. This is a server-internal API
  change, not a pnpm client wire-protocol change: the client already
  authenticates to pnpr with the request `Authorization` header.
- **Route policy + auth selection.** Add route classification to pnpr config:
  built-in anonymous-public handling for unscoped packages on
  `registry.npmjs.org`, plus operator rules for public and private
  registries/scopes/package patterns, plus the pnpr-managed uplinks with
  per-uplink access policies. The fetch path must choose the route policy and
  selected upstream auth together. Public routes send no upstream auth; private
  proxied routes send only the selected uplink credential; pnpr-hosted routes use
  client auth only to authorize the caller against pnpr package policy.
  Hosted-route detection must normalize request registry URLs against the pnpr
  service's own public base URL and any configured hosted aliases before
  considering proxied uplinks.
- **Authorize pnpr-managed uplinks.** Reuse pnpr's existing caller identity
  (bearer-token-backed users), uplink auth configuration, and registry package
  permission policy shape where possible to decide whether a request may use a
  configured uplink. Add an optional named-group/team layer if operators need
  group reuse. If no pnpr identity is available, pnpr cannot select a team
  credential; it fails closed for private proxied routes (an off-allowlist
  registry is already rejected at the request boundary, below).
- **Reject off-allowlist registries at the request boundary.** Before any
  server-side fetch, both `/resolve` and `/verify-lockfile` must reject a
  `registry`/`namedRegistries` whose origin matches no allowlisted route (the
  built-in npm route, an operator-declared public route, a configured uplink
  origin or its `/~<uplink>/` endpoint, or pnpr itself). This is the SSRF
  boundary and the reason there is no unknown-route resolve path; it runs in the
  same place as the inline-URL-credential rejection, before the request body is
  collected. An off-allowlist host returns an actionable `400`/`403` naming the
  registry, never a silent anonymous fetch.
- **Define private access descriptor inputs.** A descriptor has a key input and
  an authorization gate. Proxied routes use `uplink + credential-digest` as the key
  input and uplink authorization as the gate. pnpr-hosted routes use the hosted
  route/access-policy identity as the key input and package policy as the gate.
  Descriptor key inputs are HMACed with the server secret before they are used in
  cache keys or metadata namespaces. They are not derived from
  `AuthHeaders::for_url_with_package`, request `Authorization` headers,
  registry/scope config entries, or inline `user:pass@host` URL auth. URL and
  package normalization still matter for route-policy and descriptor selection.
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
- **Emit canonical, integrity-only tarball URLs.** When emitting streamed package
  frames and the returned lockfile, use the same route decision: public/anonymous
  routes keep upstream `dist.tarball` URLs; private proxied routes emit URLs
  canonical for the scope's configured uplink endpoint; pnpr-hosted packages emit
  pnpr-hosted tarball URLs. All three are canonical for the client's configured
  registry, so the lockfile writer stores them as integrity-only registry
  resolutions. The resolver must not return upstream auth headers, upstream
  tokens, or raw private upstream tarball URLs that are fetchable only with
  server-owned credentials, and must not emit non-canonical per-tarball gateway
  URLs.
- **Make lower metadata caches descriptor-scoped.** Pacquet's shared metadata
  mirror is currently keyed by registry/package, not by auth. Add a metadata
  cache scope to the npm resolver fetch context:
  - `Public` uses the current global mirror path unchanged.
  - `Private { access_descriptor }` uses a descriptor-scoped mirror path and
    matching in-memory key/fetch-lock key, preserving the current fast disk and
    request-coalescing paths without cross-descriptor sharing.

  (There is no descriptor-less private scope: off-allowlist routes never reach a
  fetch, so every fetched route is `Public` or `Private`.)
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
  use the same `Public` / `Private { access_descriptor }` metadata
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
    caller; a private candidate matches only when the current request can
    reproduce the same descriptor key inputs and pass every descriptor
    authorization gate in the footprint.
    This avoids knowing the footprint before the first resolve and avoids a
    second resolve or anonymous probe.
  - After a resolve, compute the footprint from the recorded routes; store an
    empty-footprint candidate for public resolutions, or a descriptor-keyed
    candidate for private resolutions.
  - For lockfile-seeded requests, include a stable digest of the canonical input
    lockfile plus the lockfile behavior flags in the base key. A different input
    lockfile, update mode, freshness policy, or trust policy must not match a
    prior candidate.
  - Store already-routed, integrity-only tarball URLs in cached resolutions. A
    private cache hit must replay the same canonical uplink-endpoint URLs, not
    raw upstream URLs that require the selected server-owned credential.
  - Bound the number of candidates stored under one base key and evict expired
    or least-recently-used private candidates first. Uplink credential rotation,
    route-policy changes, and unusual workspaces must not make lookup cost grow
    without bound.
  - The HMAC server secret is generated/configured at startup; reuse existing
    config plumbing.
  - `CachedResolution` may need to carry its footprint so lookups can apply the
    revocation-validation path when enabled.
- **Optional opt-in validation** for zero revocation window, behind config,
  addable after the base lands.
- **Lockfile/frozen path routing.** Fresh frozen-lockfile reuse and verifier-only
  paths can avoid a full resolve, but pnpr and its clients must still honor
  tarball route policy. A lockfile that contains raw private/unknown upstream
  URLs the client cannot authenticate to should be rerouted through the
  appropriate pnpr uplink endpoint or rejected for a fresh server-routed
  resolution instead of being materialized directly by the client.

Risk areas: the footprint must be complete (a missed private fetch would
mis-classify a resolution as public), so the recording must cover every metadata
fetch path, resolve-time tarball fetches (including direct HTTP tarball
dependencies fetched for manifest/integrity), auth-blind metadata mirror fast
paths, and uplink fallback ordering. Tests should cover: unscoped npmjs packages
are fetched without upstream auth and hit the shared cache even when the request
has a pnpr caller identity; private pnpr-hosted packages require caller package
access, including when the client registry points at pnpr itself; private
proxied packages require an authorized uplink and never use client-forwarded
upstream auth; mixed public/private resolves use the correct metadata namespace
per package fetch; invalid or unauthorized access misses and does not reuse
private metadata mirrors; `401`/`403` and private-route `404` responses do not
fall back to broader mirrors; inline URL credentials are rejected before fetch;
a tarball host that differs from its packument host never receives the packument
uplink's credential, and an unconfigured private tarball host fails closed rather
than leaking a credential or being proxied as an open relay;
verifier metadata fetches cannot read or populate the wrong cache scope;
different pnpr users authorized for the same uplink share resolution and metadata
cache entries; revoked uplink/package access stops matching private hits; public
tarballs keep upstream CDN URLs while private tarballs are emitted as canonical
uplink-endpoint URLs; **resolving a project through `/resolve` and resolving the
same project client-side against the same uplink-endpoint registry config produce
byte-identical, integrity-only lockfiles**; cached private resolutions do not
replay raw private upstream tarball URLs; lockfile/frozen paths do not bypass
tarball route policy; custom private default registry is not treated as public;
per-base-key candidate lists stay bounded.

## Prior Art

- **Verdaccio / Artifactory / corporate proxies** terminate upstream
  credentials at the proxy and apply per-package access control, but they cache
  package *metadata and tarballs* per upstream rather than caching whole
  *resolutions*; pnpr's resolution cache is a higher-level artifact that needs
  its own authorization model. This RFC reuses Verdaccio's **uplink** concept for
  the credential/registry-endpoint surface and adds the resolution-level cache on
  top.
- **HTTP caching with `Vary`** expresses "this cached response depends on these
  request attributes." The footprint key is the same idea specialized to
  access descriptors: public responses do not vary by auth; private responses
  vary by the descriptor and policy that can read them.
- **npm scopes** are often assumed to equal privacy; this RFC deliberately does
  not rely on that, because a private default registry makes unscoped packages
  private and a public registry makes scoped packages public.
- **Scoped private registries** (`@scope:registry=…`) are the standard npm
  mechanism this RFC leans on for private routing: pointing a scope at a pnpr
  uplink endpoint keeps the lockfile host-agnostic and integrity-only, exactly as
  it already is across a public registry and an internal mirror.

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
- **Uplink access config:** uplinks need a caller access policy in addition to
  their existing url/credential. Rotation needs no config — the private cache is
  namespaced by a digest of the credential, so editing it re-keys automatically.
  Should access be inline user lists, named groups/teams, reused package
  permissions, or all of the above?
- **Uplink registry endpoint prefix:** uplink endpoints share pnpr's root path
  namespace with package names (`/<name>` is a packument), so the endpoint path
  needs a prefix that cannot begin a package name. The proposed form is
  `/~<uplink>/`: `~` is a URL-unreserved character (never percent-encoded or
  re-decoded) and an invalid leading character for an npm package name, so it
  never collides with `/<name>` or `/@scope/...`, and mid-string it does not
  trigger shell tilde expansion. Rejected alternatives: `/<uplink>/` (collides
  with a package literally named `<uplink>`), `/+<uplink>/` (`+` is widely
  mis-decoded as a space in path segments), and `/$<uplink>/` (`$` triggers shell
  and `.npmrc` variable expansion). The npm-reserved `/-/uplink/<uplink>/`
  namespace is a bulletproof-but-clunkier fallback. Open: whether operators may
  override the prefix, and how clients are provisioned the scope→endpoint mapping
  in their `.npmrc`.
- **Default for revocation validation:** rely on the short TTL (proposed) vs.
  validate-on-private-hit by default. The latter is safer but adds a round trip
  to every private hit.
- **Split-domain tarball-origin config:** when a registry serves tarballs on a
  different origin than its packuments, is that origin declared as a **separate
  uplink** (its own credential + access policy) or as an **additional allowlisted
  origin on the same uplink** (sharing or explicitly overriding its auth)? Lean
  toward an additional origin for the common pre-signed/no-auth asset host, with
  a separate uplink when the asset host needs its own credential. Either way the
  credential rule holds: an origin receives an uplink credential only when that
  exact origin is explicitly configured.
- **Metrics:** should pnpr expose cache hit/miss split by public vs. private
  footprint and tarball routing decisions so operators can see the recovered hit
  rate and uplink-endpoint load?
