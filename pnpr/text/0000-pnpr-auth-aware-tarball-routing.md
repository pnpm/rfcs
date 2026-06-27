# Auth-aware tarball routing for pnpr resolver installs

## Summary

pnpr resolver installs should keep third-party registry credentials server-side
without forcing every tarball byte through pnpr. The resolver should return
direct upstream tarball URLs only when pnpr can prove that the selected package
tarball is anonymously fetchable. Private, gated, or uncertain tarballs should
be rewritten to pnpr URLs and fetched through pnpr's configured uplinks. pnpr
must never return reusable upstream registry auth tokens to clients.

## Motivation

The pnpr resolver can accelerate installs by moving dependency resolution to a
server. That server often has access to upstream registries that clients should
not know about directly, especially private registries or organization-wide
tokens configured for the service.

There are two competing goals:

1. Keep upstream registry credentials out of clients.
2. Avoid making pnpr proxy every tarball, because that increases pnpr
   bandwidth, CPU, storage, and latency on installs that mostly use public
   packages.

A design that rewrites every tarball URL to pnpr satisfies the security goal
but makes pnpr a hot data path for public packages. A design that returns
third-party auth tokens to the client avoids that load but turns pnpr into a
token broker and exposes credentials that may be broader than the install.

The expected outcome is a hybrid path:

- Public packages keep the normal direct tarball download path.
- Private packages still install without exposing upstream tokens.
- Unknown cases fail closed by using pnpr as the tarball gateway.

## Detailed Explanation

The resolver endpoint should continue to accept only the client's pnpr
credentials. Upstream registry credentials are configured on pnpr's uplinks and
are used only by pnpr when it talks to third-party registries.

During resolution, pnpr chooses the tarball URL that will be emitted in both
streamed `package` frames and the returned lockfile:

- If pnpr proves that the selected tarball is anonymously fetchable, it may emit
  the upstream `dist.tarball` URL.
- If the tarball requires uplink credentials, if the check fails, or if pnpr is
  unsure, it emits a pnpr URL for that tarball.

The client does not decide which path is safe. The client consumes the tarball
URLs returned by pnpr and authenticates only to pnpr when the URL points back to
pnpr.

### Public tarball proof

pnpr must not infer public availability from a successful fetch that used an
uplink token. A tarball is eligible for direct upstream routing only after an
anonymous check succeeds.

Possible checks include:

- anonymous `HEAD` to the upstream tarball URL,
- anonymous `GET` with a small range request,
- anonymous full `GET` only when the registry does not support cheaper probes,
- an explicit uplink configuration that declares tarballs from that uplink to be
  publicly fetchable.

The safe default is fail closed:

- `401` and `403` mean proxy through pnpr,
- timeout means proxy through pnpr,
- unsupported or ambiguous response means proxy through pnpr,
- network errors mean proxy through pnpr,
- missing or non-string `dist.tarball` means proxy or reject according to the
  existing resolver behavior.

### Cache public verdicts

Anonymous public-tarball verdicts should be cached because probing every
resolved package can itself become a latency and upstream-load problem.

The cache key should include enough identity to avoid granting a public verdict
to the wrong artifact. At minimum it should include:

- uplink name,
- normalized tarball URL,
- expected integrity when available,
- package name and version.

The verdict should have a bounded TTL. Negative or ambiguous verdicts may also
be cached for a shorter TTL to avoid repeated slow failures.

### Lockfile shape

When pnpr chooses a pnpr-routed tarball, the returned lockfile must keep the
explicit pnpr tarball URL. The client must not reconstruct a third-party
registry URL from `{ integrity }` plus its local registry config.

When pnpr chooses a direct public tarball, the returned lockfile may keep the
explicit upstream tarball URL. This makes the chosen route stable across frozen
installs.

Frozen or verify-only client fast paths need the same guard: if an existing
lockfile contains registry-shaped entries or tarball URLs that do not match the
safe route pnpr would choose, the client should ask pnpr to resolve and return a
fresh routed lockfile instead of materializing the old one directly.

### Registry feature split

The resolver needs some npm registry read behavior to serve private tarballs
that cannot go direct. That does not necessarily mean pnpr must expose the full
registry product whenever the resolver is enabled.

pnpr should distinguish at least these surfaces:

- resolver endpoints,
- read-only install gateway for packuments and tarballs,
- full registry API for publish, login, dist-tags, unpublish, search, and other
  npm-compatible operations.

With that split, operators can disable the full registry API while keeping the
read-only gateway needed for private resolver installs.

## Rationale and Alternatives

### Proxy all tarballs through pnpr

This is the simplest secure design. Clients never receive upstream credentials
or upstream URLs for private packages. pnpr can enforce access policy,
integrity, vulnerability blocking, and caching in one place.

The drawback is operational cost. Public packages dominate many installs, and
proxying all public tarballs through pnpr can increase latency and turn pnpr
into a bandwidth bottleneck.

### Return upstream auth tokens to clients

This keeps pnpr out of the tarball byte path, but it weakens the security model.
Generic npm registry tokens are usually reusable bearer or basic credentials.
Returning them from `/-/pnpr/v0/resolve` would expose upstream registry access
to every authorized pnpr client and make pnpr a credential broker.

This could be reconsidered only for non-reusable delegated credentials, such as
short-lived, read-only, package-scoped, audience-bound URLs or tokens. Generic
npm registries do not consistently provide that mechanism, so it cannot be the
default design.

### Direct public tarballs, proxy private or uncertain tarballs

This RFC proposes this hybrid. It keeps upstream credentials server-side and
avoids proxying tarballs that are already public. The cost is more complexity:
pnpr needs anonymous public checks, verdict caching, route selection in the
resolver protocol, and careful frozen-lockfile handling.

The hybrid is the best fit because it preserves the key security invariant
while addressing the main performance concern.

## Implementation

Implementation touches pnpr, the Rust pnpr client used by pacquet, and the
TypeScript pnpr client used by pnpm.

At a high level:

1. Remove or reject resolver request fields that forward upstream auth from the
   client.
2. Resolve packages through pnpr's configured uplinks.
3. Add a route-selection step for each selected `dist.tarball`.
4. Add an anonymous public-tarball probe with bounded TTL caching.
5. Emit either the upstream public tarball URL or a pnpr tarball URL in streamed
   package frames and in the returned lockfile.
6. Ensure lockfile materialization uses the emitted URL and does not reconstruct
   third-party registry URLs for pnpr-routed tarballs.
7. Split pnpr feature toggles so resolver installs can depend on a read-only
   install gateway without enabling the full registry API.
8. Add observability for routing decisions: public direct, pnpr proxied,
   probe failed, probe skipped, cache hit, and cache miss.

Tests should cover:

- public packages returning upstream tarball URLs after anonymous proof,
- private packages returning pnpr tarball URLs,
- unknown probe failures falling back to pnpr URLs,
- no upstream auth token in resolver request or response bodies,
- frozen installs not bypassing pnpr when the existing lockfile contains
  third-party registry-shaped entries,
- named-registry aliases not bypassing pnpr's route selection,
- public-verdict cache key isolation by uplink, URL, package, version, and
  integrity.

## Prior Art

npm clients normally fetch public packages directly from the `dist.tarball` URL
advertised by registry metadata. Registry proxies and mirrors commonly sit in
front of upstream registries to cache or control access to tarballs. This RFC
combines those models: public tarballs keep the normal direct path, while
tarballs that require pnpr's configured credentials use pnpr as a read-only
gateway.

Some artifact systems support short-lived pre-signed URLs or scoped download
tokens. If an npm-compatible uplink can produce such delegated URLs safely, pnpr
could support that as an optional future optimization. It should not be required
for generic npm registries.

## Unresolved Questions and Bikeshedding

- Should anonymous public proof be automatic by default, or should each uplink
  opt into it?
- Which anonymous probe should be preferred: `HEAD`, ranged `GET`, or another
  registry-specific strategy?
- How long should positive and negative public-tarball verdicts be cached?
- Should pnpr cache public tarballs opportunistically even when it returns an
  upstream direct URL?
- What is the exact config shape for the read-only install gateway versus the
  full registry API?
- Should the resolver protocol expose route diagnostics to clients, or only to
  server logs and metrics?
- How should pnpr handle public metadata with private tarballs, or private
  metadata with public tarballs?
