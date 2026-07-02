# Patch-provider integration for pnpr

## Summary

Vendors such as Echo, Seal Security, and Socket ship security-patched builds
of vulnerable open-source package versions and want to deliver them through
the registry the organization already uses. pnpr should support this in two
shapes: a vendor registry as an ordinary **upstream mount** (works today, the
vendor is the declared origin), and a **patch scope** — the provider publishes
patched artifacts under its own package namespace (`@echo-patch/ejs@2.7.4`),
served through ordinary mounts and routes. A signed, pinned **patch
manifest** ties the two names together, and a per-mount `patching:` policy
applies it in one of two modes. Both serve the same per-version
**annotation** on the vulnerable version's packument entry, which a
patch-aware resolver acts on **after version selection**: whenever
resolution picks an annotated version, it returns the provider's aliased
artifact instead — recorded like an ordinary alias, with a canonical,
host-free lockfile resolution — while already-pinned installs are never
altered. The modes differ only in the client posture the annotation
requests: **advertise** substitutes only for patches the workspace has
explicitly accepted (discovered through audit enrichment, accepted by hand
or via `pnpm audit --fix`), while **substitute** substitutes by default,
with a per-package ignore list to refuse.

Notably, this requires **no new mount kind**. Both shapes are expressible
with hosted/upstream/router mounts as already specified; the only
serving-path addition is the annotation, layered onto affected packuments
on egress over a pristine cache. What this RFC adds is the manifest
protocol, that annotation, and the audit/advisory machinery around them.

Both shapes extend the invariants of the registry-mounts RFC rather than
weakening them:

> **Provenance is declared, never inferred.** Every `name@version` resolves to
> exactly one declared concrete origin. A patched artifact is a *different
> name* from a *declared provider namespace* — never a request-time
> substitution behind the original name.
>
> **An identity's bytes are immutable.** pnpr never serves different bytes
> under a `name@version` — or a tarball URL — a lockfile may already pin.
> Choosing patched bytes means choosing either a different declared origin
> (mount) or a distinct package identity — never a silent byte swap.
>
> **Substitute at resolution, never at fetch.** Patching may change what a
> *new resolution* is directed to — the same kind of event as a new publish
> or a dist-tag move — and the result is recorded explicitly in the client's
> lockfile. Patching never changes what an already-recorded resolution
> fetches.

## Motivation

### The registry is the right choke point

Every artifact an organization installs passes through its registry — every
developer machine, every CI job, every transitive dependency. A remediation
applied there is applied once, centrally, with no per-repository configuration
and no gaps for the repo that forgot to install a wrapper CLI. The registry is
also the only place that can *tell* every consumer, through the audit
endpoint, that a patched artifact exists for the exact version they depend on.

### Patch vendors exist and want registry distribution

A class of products now sells *backported security patches for the exact
version you already depend on*, so a critical CVE can be remediated without a
major-version migration:

- **Echo Libraries** runs npm/PyPI/Maven/etc. packages through a hardening
  pipeline (sandboxing, CVE remediation, signing) and delivers them as an
  artifact source that mirrors into Nexus, Artifactory, and custom
  repositories. Echo patches *on the same version number* the application
  requires.
- **Seal Security** publishes sealed versions under distinct identifiers —
  `ejs@2.7.4-sp1` is the first seal-patch of `ejs@2.7.4` — served from Seal's
  artifact server and adopted via overrides in the consuming project.
- **Socket** ships Certified Patches as repo-local patch files applied at
  install time (deliberately not a registry proxy). Its registry product,
  Socket Firewall, blocks malicious packages but does not substitute patched
  ones.
- **Google Assured OSS** rebuilds, patches, and signs popular open-source
  packages and serves them from Google's own registry endpoint.

These vendors want a way to plug into pnpr, and pnpr deployments want to
consume them without handing the vendor authority over every package. The
mount model gives us the vocabulary to do that precisely.

## Detailed Explanation

### Baseline: the vendor as a declared origin

This works with mounts as already specified, with zero new machinery:

```yaml
mounts:
  echo:
    type: upstream
    url: https://npm.echo.example/org-acme/
    auth:
      tokenEnv: ECHO_NPM_TOKEN
    access: team:acme

  main:
    type: router
    routes:
      - patterns: ['@acme/*']
        source: acme
      - patterns: ['**']
        source: echo
```

The organization has chosen Echo as its artifact source for public packages.
Echo's same-version patching model is *coherent* in this shape: the vendor is
the declared origin, the bytes are the vendor's bytes, the lockfile integrity
is the vendor's integrity, and that origin is expected to be internally
consistent about those bytes forever. Nothing is being substituted behind an
identity another origin defined. Seal's model fits the same shape: Seal's
artifact server serves the `ejs` packument with its `-spN` versions included,
and Seal is simply the declared origin for it.

Two consequences are worth stating:

- **Trust is coarse.** The vendor can serve arbitrary bytes for *every*
  package routed to it, not only the ones it patched. That is the correct
  contract when the vendor is bought as "our vetted artifact source", and the
  wrong one when the organization only wants the vendor's patches.
- **This makes the lockfile registry-identity follow-up urgent.** The same
  `name@version` now routinely exists with different bytes on `~npmjs` and
  `~echo`. pnpm's current `name@version` package key cannot represent both in
  one graph (see the mounts RFC's motivation); a patched-registry deployment
  will hit that gap on day one when a workspace mixes mounts.

### Patch scopes: patched artifacts under the provider's namespace

For the "only the patches" contract, the provider publishes each patched
artifact as its own package under a provider-owned scope, keeping the
**original version number**:

```text
ejs@2.7.4                    # the vulnerable original, on npmjs
@echo-patch/ejs@2.7.4        # Echo's patched build of exactly that version
```

The patched package's identity is distinct by *name*, so nothing about the
original — its packument, versions, dist-tags, or bytes — is touched. pnpr
serves the patch scope through completely ordinary configuration:

```yaml
mounts:
  npmjs:
    type: upstream
    url: https://registry.npmjs.org/
    public: true

  echo-patches:
    type: upstream
    url: https://npm.echo.example/patched/
    auth:
      tokenEnv: ECHO_NPM_TOKEN
    access: team:acme

  main:
    type: router
    routes:
      - patterns: ['@echo-patch/*']
        source: echo-patches
      - patterns: ['**']
        source: npmjs

patchProviders:
  echo:
    scope: '@echo-patch'
    source: echo-patches
    manifest:
      url: https://npm.echo.example/pnpr-manifest.json
      publicKeys:
        - ed25519:AAAA...
      refreshInterval: 1h
    policy:
      requireAdvisoryMatch: true   # a patch must claim to fix a known advisory
```

No new mount kind and no fall-through anywhere: the patch scope is just
packages, routed by the existing router rules, cached in the existing
per-mount namespaces, gated by the existing access policies. The new
configuration surface is `patchProviders:`, which registers the **manifest**
that ties the two namespaces together — and which a mount's `patching:`
policy (below) chooses how to apply.

### The patch manifest

The manifest is a signed document published by the provider mapping original
package versions to their patched counterparts:

```jsonc
{
  "schema": "pnpr-patch-manifest/1",
  "entries": {
    "ejs@2.7.4": {
      "patched": "@echo-patch/ejs@2.7.4",
      "integrity": "sha512-...",            // of the patched tarball
      "fixes": ["GHSA-...", "CVE-2024-..."],
      "attestations": ["https://..."]       // provenance, VEX, build info
    }
  }
}
```

- **Pinned, not probed.** pnpr fetches and signature-verifies the manifest on
  its refresh interval and pins a snapshot. Everything derived from it (audit
  enrichment, advisory mapping) is a pure function of config plus the pinned
  snapshot. A manifest refresh is a logged, diffable event, like an operator
  config change.
- **Scope-bound.** Every `patched` entry must name a package inside the
  provider's declared scope; an entry pointing anywhere else is rejected at
  validation. A provider can describe only its own namespace.
- **Integrity-pinned, enforced at the serving path.** The patched content
  originates from the provider's infrastructure — an origin the deployment
  does not control — so pnpr is the boundary where the provider's signed
  promise is enforced. For every manifest-listed version, the patch-scope
  packument's `dist.integrity` must equal the manifest's pin, and fetched
  tarball bytes are verified against it; a mismatch is an error, never a
  fall-through to the vulnerable original. A compromised patch source can
  therefore refuse to serve, but cannot serve different bytes through pnpr
  than the signed manifest promised.
- **Patch re-issues stay inside the namespace.** If a patch itself needs a
  second revision (a new CVE lands on the same base version), the provider
  publishes `@echo-patch/ejs@2.7.4-sp2` and updates the manifest entry.
  Suffix ordering quirks are harmless inside a dedicated patch namespace,
  because adoption is always an exact pin produced from the manifest — no
  range resolution against upstream expectations ever happens there.

### Advertise mode: discovery and explicit acceptance

In `advertise` mode the registry surfaces available patches but substitutes
nothing by default: adoption is an explicit, reviewable workspace decision.
The mechanics are the same annotation and patch-aware resolver as substitute
mode (below) — the annotation simply requests an **opt-in posture**. Three
adoption paths:

1. **Audit enrichment.** pnpr implements the npm audit endpoints
   (`/-/npm/v1/security/advisories/bulk`) against OSV plus the pinned
   manifests, and reports, per vulnerable version, the patched artifact
   available *on this registry*. The stock response format has no field for
   "an aliased fix exists", so pnpr adds a namespaced extension field
   carrying the mapping (`ejs@2.7.4 → npm:@echo-patch/ejs@2.7.4`); clients
   that do not know the field ignore it.
2. **Explicit acceptance (patch-aware clients).** The workspace lists
   accepted patches in its config — per entry, or per provider — and the
   patch-aware resolver substitutes exactly as in substitute mode, but only
   for accepted entries. `pnpm audit --fix` (pnpm follow-up, out of scope
   here) becomes mechanical: read the enriched audit response, append
   accept-list entries — reviewable in the PR that introduces them,
   removable after upgrading away. Because acceptance is applied by the
   resolver **after version selection**, it correctly covers picks reached
   through ranges — the case overrides cannot express, below.
3. **Hand-written overrides (any npm-compatible client, today).** An alias
   override adopts a patch with no new client feature — but overrides
   rewrite *declared specs* by subset matching, and that has teeth:
   `"ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"` covers only dependencies
   declared as exactly `2.7.4`, silently missing every `^2.7.0` that
   resolves to `2.7.4`. Covering ranges requires keying by the declared
   ranges themselves (`"ejs@^2.7.0": "npm:@echo-patch/ejs@2.7.4"`), which
   *freezes* everything inside that range to the patched build — including
   after upstream releases a fixed `2.7.5` — and misses differently-shaped
   ranges added later; a bare-name key is the same trade graph-wide. This
   path is honest and available, but it is a blunt instrument, not the
   mechanism this RFC builds on — and it is why `pnpm audit --fix` writes
   accept-list entries rather than overrides.

What the alias buys over other encodings of "patched artifact":

- **The original version number is preserved** (`2.7.4`, not `2.7.4-sp1`), so
  everything keyed on version — semver ranges inside the patched namespace,
  `require('ejs/package.json').version` probes, version-gated behavior —
  sees the version the application was tested against. This is the property
  Echo's product leads with, delivered without same-version byte swapping.
- **Provenance is legible in every artifact that records the name.** The
  lockfile, `node_modules`, SBOMs, and audit logs all show `@echo-patch/ejs`
  — nobody has to know that `-sp1` means "Seal patch" to notice a
  substitution happened.
- **No prerelease semantics.** Tools that treat any prerelease as unstable —
  `npm outdated`, Renovate/Dependabot version filtering, semver-range
  tooling — never see a prerelease identifier on the adoption path.

Advertise mode's limit is deliberate: it is **opt-in per workspace**. A
fresh `pnpm install` in a repository that has accepted nothing resolves
`^2.7.0` to the vulnerable original, exactly as before, and repositories
that never run `pnpm audit` never find out. When the deployment's intent is
"no fresh resolution may pick a version we have a patch for", that is
substitute mode.

### Substitute mode: annotated packuments selected by the resolver

Both modes put the mapping where the resolver is already looking: on the
vulnerable version's entry in the packument. For every manifest entry the
mount's policy accepts, pnpr serves the base packument with an annotation on
that version — original metadata and `dist` untouched — carrying the mode's
requested posture:

```jsonc
// GET /~main/ejs — versions["2.7.4"], annotated on egress
"2.7.4": {
  // ...original metadata, original dist...
  "_pnprPatch": {
    "patched": "npm:@echo-patch/ejs@2.7.4",
    "provider": "echo",
    "fixes": ["GHSA-..."],
    "apply": "default"        // advertise mode serves "opt-in"
  }
}
```

```yaml
mounts:
  main:
    type: router
    routes:
      - patterns: ['@echo-patch/*']
        source: echo-patches
      - patterns: ['**']
        source: npmjs
    patching:
      provider: echo
      mode: substitute
      minSeverity: high      # only annotate fixes at/above this severity
```

The annotation is applied **on egress**: the cached upstream packument stays
byte-identical to what the origin served, and the annotation is layered on
at response time as a pure function of the pinned manifest snapshot. The
field is inert for any client that does not know it. In advertise mode the
identical annotation is served with `apply: opt-in` and the resolver
substitutes only workspace-accepted entries (previous section); the rest of
this section describes the default-on posture that gives substitute mode its
name.

A patch-aware resolver (a pnpm follow-up RFC) selects the annotation
*inside* the resolve call. pnpm's resolver already separates a dependency's
**alias** — the key in the dependent's manifest — from the **resolved
identity** it returns (`id`, `manifest`, `resolution`); that split is how
`npm:` aliases work today. The patched flow rides it unchanged: resolving
`ejs@^2.7.0`, the resolver picks `2.7.4` from the packument, sees
`_pnprPatch`, resolves `@echo-patch/ejs@2.7.4`, and returns *that* result
(marked `resolvedVia: registry-patch`), while the dependency keeps its
alias `ejs`. Version selection is never changed — the pick is the key that
selects the patch — only which *build* of the selected version is
installed. A substituted result is returned as-is, never re-matched, so
substitutions cannot chain. And no override semantics change anywhere:
workspace overrides keep exactly their current meaning, applied to declared
specs before resolution, and continue to apply here (an override that
rewrites the spec away from `ejs@2.7.4` means that version is never picked
and the annotation never consulted). Everything downstream is pnpm's
already-shipped alias machinery, and that is the point:

- **The lockfile stays canonical and host-free.** A fresh resolution of
  `^2.7.0` picks `2.7.4`, the annotation redirects that pick, and the
  lockfile records the aliased package `@echo-patch/ejs@2.7.4` under the
  dependency key `ejs` with a canonical resolution — integrity only,
  tarball URL recomputed from registry config at install time. It is
  byte-for-byte the lockfile a hand-written alias override would have
  produced.
- **No pinned install ever changes.** The annotation is consulted only when
  a resolution is made; an existing lockfile entry is never touched, and a
  `--frozen-lockfile` install reads no packuments at all. Substitute at
  resolution, never at fetch.
- **Substitutions are recorded so the lockfile stays self-consistent.** To
  pnpm's up-to-date check, `ejs: ^2.7.0` resolving to an alias it cannot
  explain from the workspace's own config would look like an entry needing
  re-resolution. The follow-up therefore records substituted entries
  (mapping plus source registry) in a dedicated lockfile section — only
  applied entries, so the lockfile grows with the workspace, not the
  provider's catalog. The record lets satisfiability checks accept the
  aliased entry, lets a withdrawn patch re-resolve exactly the affected
  packages, and tells humans and SBOM tooling *why* the alias exists.
- **Provenance is legible and diffable.** The lockfile shows `ejs` resolving
  to `@echo-patch/ejs@2.7.4` and the recorded entry that caused it; when a
  manifest refresh moves a patch to `-sp2`, the next non-frozen resolution
  produces a reviewable lockfile diff in both places.
- **Opting out is an explicit client setting.** With no override merge
  involved, precedence cannot express refusal; the follow-up defines an
  ignore list (per package, or per provider) that makes the resolver skip
  the annotation — declared in workspace config, visible and reviewable
  like an override.

Clients that are not patch-aware ignore the unknown field and resolve the
vulnerable original — no worse than today. Per-mount policy can escalate:
`unawareClients: refuse` makes the tarball route answer a manifest-covered
vulnerable artifact with the explicit `403`-plus-suggested-override body
instead of bytes. That is hard enforcement, stated honestly: the tarball
route cannot distinguish a fresh resolution from an old pinned one, so
`refuse` also blocks existing lockfiles that pin the vulnerable original —
which is precisely what "no known-patchable vulnerable artifact leaves this
registry" means. The default is `serve`.

**Resolution cost.** Discovering patches costs nothing: the mapping rides
the packument the resolver already fetched, so unpatched packages — and
workspaces that never resolve a patched version — pay zero, and there is no
separate catalog to download, cache, or discover. A matched pick resolves
the alias target: one exact-version resolution against the patch-scope
packument, which is small (patched versions only), cached like any other,
and fetched in parallel with the rest of resolution — a second resolution
of *that dependency*, never of the graph. The original pick is not wasted
work; it is the key that selects the patch, and there is no way to learn
that `^2.7.0` lands on `2.7.4` without performing the pick. The annotation
could additionally carry the patched tarball's integrity and manifest
essentials so even that one fetch disappears — deliberately not required:
the trust boundary it would police (the patch source's infrastructure,
which the deployment does not control) is already policed server-side,
where pnpr verifies both the patch-scope metadata and the tarball bytes
against the provider's *signed manifest* before serving them. Client-side
verification against annotation-carried integrity would defend only against
pnpr itself failing to enforce that check — defense in depth, parked as an
open question.

**Server-side and client-side resolution apply the same mapping at the same
point.** The flow above is client-side resolution: pnpm reads annotated
packuments and substitutes inside its own resolve calls. When resolution
happens in pnpr itself — the resolution endpoint of the auth-aware
resolution-cache design — the server applies the same annotation logic at
the same logical point, so returned resolutions are natively aliased and a
server-side substitution is exactly as traceable as a client-side one: the
aliased resolution *is* the trace, carrying the same `resolvedVia` marker
for the lockfile record. One requirement keeps the paths coherent:
server-side **resolution-cache keys include the mount's patching policy and
the pinned snapshot digest** — otherwise a manifest refresh could keep
serving pre-refresh substitutions (or none) out of the cache.

Costs and requirements, stated honestly:

- **Automatic protection reaches only patch-aware clients.** For npm and
  yarn the annotation is inert; they get advertise-mode discovery (and
  `refuse`, where enabled). pnpr is pnpm-first, and the annotation is
  deliberately trivial — an alias spec on a version entry — so other
  resolvers could adopt it, and an org can still derive blunt bare-name
  overrides from the audit endpoint for clients that predate the feature.
- **Provider retention becomes load-bearing.** A lockfile aliasing
  `@echo-patch/ejs@2.7.4` installs only while that artifact remains
  available. Providers must treat their patch namespaces as immutable and
  permanent (the manifest protocol should say so), and pnpr's per-mount cache
  keeps serving cached patched artifacts through provider outages, like any
  other mount content.
- **Same range, different policies, different bytes.** Two workspaces
  resolving the same range through mounts with different `patching:` policies
  get the same version as different builds. That is the deployment's declared
  intent — but it is one more reason registry identity belongs in package
  identity.

Determinism and conflicts: the annotations are a pure function of the
mount's `patching:` config plus the pinned manifest snapshots — the same
determinism contract as router routes — and a refresh that changes them is
logged and diffable. Within one mount, at most one provider may claim a
given original `name@version`; two manifests colliding is a config error,
not a precedence guess. And substitution composes with advertise-mode
machinery — the audit endpoint reports the same mapping, so a workspace can
see which of its resolutions were substituted and pin them explicitly if it
prefers.

**Delivery is proportional by construction.** There is no patch catalog to
ship: a workspace only ever sees mappings for packages it actually
resolves, embedded in metadata it was fetching anyway — a few hundred bytes
on affected version entries of affected packuments. The catalog-scale
questions a standalone document raises (size bounds, endpoint discovery,
separate caching and revalidation) do not arise.

### Advisory screening of patched artifacts

Aliasing moves the advisory problem rather than removing it, and the manifest
is what solves it. OSV advisories are keyed by the *original* package name:

- `@echo-patch/ejs@2.7.4` matches no `ejs` advisory textually — including
  future ones. Without correction this is a **blind spot**: a brand-new CVE
  against `ejs@2.7.4` would flag the vulnerable original but silently miss
  the patched copy of it, which is equally affected unless the provider says
  otherwise.
- pnpr therefore screens every patch-scope artifact **as its upstream
  identity**, resolved through the pinned manifest: `@echo-patch/ejs@2.7.4`
  is evaluated against `ejs@2.7.4`'s advisories, *minus* the manifest's
  `fixes` list (the provider's VEX claim) for that exact integrity-pinned
  artifact. Fixed advisories are subtracted; everything else — including
  advisories published after the patch — applies normally.
- The same mapping feeds the audit endpoint, so a workspace that adopted a
  patched artifact still hears about new advisories against its base version.

## Rationale and Alternatives

### Prerelease-version grafting behind the original name

The main alternative — and this RFC's own earlier draft — is an **overlay
mount** that serves the base origin's packument with provider-patched versions
grafted in as distinct prerelease-style versions (`ejs@2.7.4-sp1`), routing
those versions to the patch source and everything else to the base.

Its advantage is discoverability in one namespace: the patched versions appear
in `npm view ejs versions`, and adoption is a version-only override with no
name change. It is rejected as the primary mechanism because:

- **It synthesizes load-bearing metadata.** The overlay serves a packument
  that no single origin published — base metadata with foreign *version
  entries* spliced into the version list, entries every client acts on. The
  substitute-mode annotation is categorically smaller: an inert extension
  field on an existing version entry, acted on only by resolvers that opted
  in — yet grafting buys neither automatic protection (last bullet below)
  nor a cleaner adoption story.
- **It requires a new composite mount kind**, with its own validation, routing,
  write-rejection, and access-composition rules. The alias model is ordinary
  packages through ordinary mounts.
- **Prerelease identifiers fight the ecosystem.** `2.7.4-sp1 < 2.7.4` in
  semver — useful for keeping ranges from matching accidentally, but the same
  property makes every prerelease-aware tool treat the patched artifact as
  unstable, and OSV ranges like `< 2.7.5` match it textually, requiring VEX
  subtraction just to serve it.
- **The version number lies about itself either way.** `2.7.4-sp1` is neither
  the version the application pinned nor a version upstream ever published.
  `@echo-patch/ejs@2.7.4` keeps the real version and moves the difference
  into the name, where provenance belongs.
- **It cannot protect fresh resolutions either.** Because prereleases sort
  below their base version and match no ordinary range, `^2.7.0` never
  resolves to `2.7.4-sp1` — grafting has exactly the same opt-in-only limit
  as advertise mode, while paying all the costs above. Substitute mode covers
  fresh resolutions without prerelease identifiers.

Providers that already ship same-name prerelease patches (Seal's `-spN`) are
still fully served: their artifact server is a vendor-as-origin upstream
mount, unchanged.

### Fetch-time byte substitution behind the original identity

Rejected in any pnpr-mediated form — and worth distinguishing carefully from
substitute mode. Fetch-time substitution serves provider bytes for the
*original* canonical tarball URL, so the same recorded resolution yields
different integrity depending on when it is fetched: existing lockfiles break
the moment a patch appears, shared caches become ambiguous about which bytes
are "the" version, and nothing in any lockfile records that a substitution
happened. Substitute mode has none of these properties: the original
canonical URL serves original bytes forever, and the patched artifact enters
only through *new resolutions*, under its own aliased identity and integrity,
recorded in the lockfile. Organizations that want fetch-level same-version
patching have
a coherent home for it: make the vendor the declared origin (upstream mount),
where the vendor owns the identity end to end.

### Rewriting `dist` inside served packuments

An earlier shape of substitute mode rewrote the vulnerable version's `dist`
to the patched artifact's tarball URL and integrity. Its appeal is
universality — every npm client gets auto-patched with no client feature —
but the patched URL is non-canonical for the original name, so clients
persist the full URL, baking the deployment host into lockfiles: exactly
what the mounts RFC's canonical-URL round-tripping exists to prevent, and a
silent bytes-for-version change for clients that never asked for one. The
chosen annotation differs in both respects: it *adds* an inert field
instead of altering `dist`, only a resolver that understands it acts on it,
and the result is recorded as a first-class aliased identity with a
canonical, recomputable resolution.

### Distributing the mapping as spec-rewriting overrides, unchanged

The simplest client story would be serving the mapping *in overrides
format* for clients to merge alongside workspace overrides, with no
semantic change anywhere. Rejected because today's override semantics
cannot express the goal. Overrides rewrite **declared specs** before
resolution, and a selector matches a declared range by subset:
`"ejs@2.7.4"` applies to a dependency declared as `2.7.4`, but not to
`^2.7.0` — whose resolution to `2.7.4` is exactly the case that needs
protection. Widening the selector cannot fix it: no range selector matches
`^2.7.0` short of the bare name, and a bare-name override
(`"ejs": "npm:@echo-patch/ejs@2.7.4"`) *changes version selection* — it
forces every `ejs` in the graph to the patched `2.7.4`, downgrading a
`^2.7.0` that would have picked a genuinely fixed `2.7.5`. Any correct
mechanism must act after version selection — spec rewriting cannot see the
pick. The same fact rules out `pnpm audit --fix` emitting overrides: an
exact-version key would not apply to range declarations, and range or
bare-name keys freeze or force version selection — which is why the fix
flow writes accept-list entries for the patch-aware resolver instead.

### A standalone patched-versions document plus an override extension

The previous draft of substitute mode compiled the manifests into a
standalone document at a well-known endpoint
(`/-/pnpr/patched-versions`), applied by the client through a semantic
extension of overrides: an override whose selector is an exact version
would also match the *picked* version at resolution time, and the
document's exact-version→alias entries would merge beneath the workspace's
own overrides. It satisfies the act-after-selection requirement, and its
opt-out story is elegant (workspace override precedence). The annotation
was chosen over it because:

- **Delivery and discovery.** A standalone catalog is a whole protocol
  surface: endpoint path and discovery, trust/signing, machine-wide
  caching and revalidation, size bounds and a scale escape hatch. The
  annotation needs none of it — mappings arrive with exactly the
  packuments being resolved, proportionally, over the already-authenticated
  registry connection.
- **No override-semantics change.** The extension was carefully additive
  (for an exact-version selector, post-pick matching only covers picks that
  previously slipped through), but it was still a behavior change to
  existing workspace configs, with real questions about how it ships.
  Resolver-side annotation selection leaves override semantics completely
  untouched.

What the document did better is worth recording: workspace opt-out fell out
of ordinary override precedence (annotations need an explicit ignore
setting instead), and the override extension also empowered *hand-written*
exact-version overrides to catch range-resolved picks — a genuinely useful
capability that the annotation approach does not deliver. If that proves
worth having, it can be proposed independently in pnpm; nothing in this
design conflicts with it.

### Named-registry aliases: same name, same version, different registry

The most elegant encoding is arguably pnpm's named registries: override
`ejs@2.7.4` with `echo:ejs@2.7.4` — identical name and version, provenance
carried entirely by the registry component of the package identity, exactly
the model vlt's DepID uses and the mounts RFC already endorses for lockfiles.

Deferred, not rejected: it depends on the lockfile registry-identity
follow-up (today pnpm's `name@version` package key cannot hold `ejs@2.7.4`
from two registries in one graph — precisely this situation), and it is
expressible only by clients with named-registry support, where the scoped
alias works in any npm-compatible client. When the lockfile work lands, the
manifest's `patched` field can name a registry-qualified spec as an
alternative encoding, and the adoption machinery above carries over
unchanged.

### Leave patching entirely client-side (overrides, Socket-style repo patches)

Not sufficient alone. Repo-local patching (Socket Certified Patches, pnpm
`patchedDependencies`, overrides) is excellent for application teams and needs
no infrastructure — but it protects one repository at a time, drifts across
hundreds of repos, and gives a security organization no central enforcement or
inventory. Note that the alias model *ends* in a client-side override too —
the registry's role is serving the patched namespace, vouching for the
manifest, and making patches discoverable through audit; the client's role is
adopting them visibly.

### Only support vendor-as-origin, no patch scopes

Simplest possible scope — and it is the shipped baseline regardless. But it
forces an all-or-nothing trust decision: to get one vendor's patches, the
deployment must make that vendor the origin for every routed package. The
patch scope exists so the base origin (say, npmjs) keeps serving everything
except the specific, signed, advisory-matched artifacts the provider patched.

## Implementation

All registry-side changes are in pnpr-server. The mount graph and routing
are untouched; substitute mode adds one egress annotation step layered over
pristine cached packuments:

1. **`patchProviders:` config and manifest machinery.** Parse/validate the
   provider block (scope, source mount, manifest URL, keys, policy);
   fetch/verify/pin manifests on the refresh interval with logged diffs;
   reject entries outside the provider's scope, without integrity, or (under
   `requireAdvisoryMatch`) without a known advisory in `fixes`.
2. **Advisory identity mapping.** Screen patch-scope artifacts as their
   manifest-mapped upstream identity, minus the `fixes` list for the pinned
   integrity; apply the same mapping wherever advisories are evaluated
   (screening policy, audit responses).
3. **Audit endpoint enrichment.** Implement/extend the npm advisories bulk
   endpoint from OSV data, adding the namespaced extension field with the
   suggested override spec when a pinned manifest covers a vulnerable
   version.
4. **Per-mount `patching:` policy and the egress annotation.** Validate the
   policy block (known provider, mode, severity gate, `unawareClients`, one
   substituting provider per original `name@version` per mount); annotate
   matched version entries of served packuments on egress with the mode's
   requested posture (`apply: default` / `opt-in`), as a pure function of
   the pinned snapshots, leaving cached upstream documents and every `dist`
   object untouched; implement `unawareClients: refuse` on the tarball
   route with the `403` reason body.
5. **Server-side resolution integration.** Where pnpr resolves on the
   client's behalf (the resolution endpoint of the auth-aware
   resolution-cache design), apply the same annotation logic inside
   resolution so returned resolutions are natively aliased and carry the
   substitution marker, and include the patching policy and snapshot digest
   in resolution-cache keys.

Client-side follow-ups (separate pnpm RFC): the patch-aware resolver —
selecting `_pnprPatch` inside the resolve call, returning the aliased
identity (`resolvedVia: registry-patch`), recording substituted entries in
a dedicated lockfile section so satisfiability checks accept them, and
honoring the posture through accept lists (opt-in annotations) and ignore
lists (default annotations) — plus `pnpm audit --fix` writing accept-list
entries from the enriched audit response.

Tests should cover, at minimum:

- provider config validation (unknown source mount, scope collisions between
  providers, unsigned/tampered manifests, out-of-scope entries, missing
  integrity, `requireAdvisoryMatch` violations);
- manifest refresh changing derived data only at refresh, with the change
  logged;
- patch-scope packages serving through ordinary routing with the source
  mount's access policy enforced;
- integrity mismatch between manifest and patch source surfacing as an error;
- advisory mapping: a patched artifact not flagged by fixed advisories,
  flagged by unrelated ones, and flagged by advisories published *after* the
  patch against the same base version;
- audit bulk endpoint carrying the override-spec extension for a vulnerable
  version with a manifest entry, and omitting it otherwise;
- matched version entries annotated on egress only for manifest-covered,
  policy-accepted versions, as a pure function of the pinned snapshot,
  carrying `apply: opt-in` in advertise mode and `apply: default` in
  substitute mode;
- cached upstream packuments staying byte-identical to origin (annotation
  on egress only), every `dist` object and all tarball responses identical
  with `patching:` enabled and disabled;
- the patch-scope packument's `dist.integrity` for a manifest-listed version
  equaling the manifest's pinned integrity (the client-visible pin);
- an existing lockfile pinning the vulnerable original installing unchanged
  through a substitute-mode mount (with `unawareClients: serve`);
- `unawareClients: refuse` answering manifest-covered vulnerable tarballs
  with `403` plus the suggested override, and leaving uncovered artifacts
  untouched;
- the one-substituting-provider-per-original rule enforced at config load;
- server-side resolution returning aliased resolutions from the same pinned
  snapshot as the served annotations, and a manifest refresh re-keying
  (never silently reusing) cached resolutions;
- a manifest refresh changing annotations being logged, and previously
  aliased patched artifacts continuing to serve as long as the provider
  retains them;
- vendor-as-origin deployments (Seal-style `-spN` packuments) proxying
  unchanged through an upstream mount.

## Prior Art

- **Seal Security** proves the adopt-via-override flow: patched versions
  under distinct identifiers, adopted by explicit overrides in the consuming
  project. This RFC keeps the override flow and moves the distinct identifier
  from the version to the name.
- **Echo Libraries** validates the vendor-as-origin shape and the
  same-version requirement: a hardened, signed artifact pipeline that mirrors
  into existing repository managers, patching on the version the application
  already requires. The patch scope delivers that property without byte
  swapping.
- **npm package aliases** (`npm:name@version`) are the established substitution
  mechanism this design rides on; pnpm supports aliases both as dependencies
  and as override targets, which is what makes adoption a config edit rather
  than a client feature.
- **pnpm's config dependencies** (RFC 0004) already distribute shared
  overrides to many repositories through an installed package;
  annotation-driven substitution serves the same organization-wide goal with
  the registry as the channel, and the two can coexist (an org can derive
  blunt overrides from the audit endpoint into a config dependency for
  clients that predate the resolver feature).
- **vlt's DepID** (registry as a component of package identity) is the prior
  art for the deferred named-registry encoding.
- **Socket Certified Patches** demonstrates the repo-local patching shape;
  its docs explicitly argue *against* registry-proxy patching for application
  repos — a fair argument at repo scope that does not hold for
  organization-wide enforcement, which is pnpr's scope.
- **Google Assured OSS** — rebuilt, patched, signed packages behind Google's
  own registry endpoint — is vendor-as-origin at cloud scale.
- **Snyk's deprecated patch protocol** (per-CVE diff files applied at install
  time) is a cautionary tale: patch-at-install couples remediation to client
  tooling and rots; serving real artifacts from a registry does not.
- **npm audit's bulk advisory endpoint** is the existing client-visible
  surface this RFC enriches rather than replaces.
- **Go's `replace` directive with a left-side version**
  (`replace ejs v2.7.4 => patched v2.7.4`) is the closest living precedent
  for substituting the selected pick: it applies **only when minimal
  version selection selects exactly that version**, other versions are
  untouched, and it is honored only in the main module — the same
  substitute-the-selected-pick shape the patch-aware resolver performs
  here.
- **Cargo's `[replace]`** was post-resolution exact-version substitution
  almost verbatim: keyed by an exact package id, it swapped the *source* of
  an already-resolved node, and the replacement was required to keep the
  same name and version — precisely the same-version-different-build
  contract of a security patch. It was deprecated in favor of `[patch]`,
  which instead augments the candidate set a source offers *before*
  resolution — because users wanted *version-changing* overrides
  (unpublished bumps `[replace]` could not express), and because mutating
  the graph behind the resolver's back broke resolver-derived behavior such
  as feature unification. Neither of `[patch]`'s two moves transfers to
  npm's model: candidate *replacement* (same version, different bytes)
  works in Cargo because source is part of package identity in
  `Cargo.lock`, which npm's `(name, version)` identity cannot represent —
  the rejected dist-rewrite; candidate *addition* works because a Cargo
  patch carries a real version that ranges select, whereas an npm patched
  artifact pinned to the same version number can only be distinguished by a
  prerelease suffix that ranges exclude — the rejected grafting. That is
  why this design moves identity into the *name* (patch scope) and keeps
  `[replace]`-style post-pick substitution, with the applied-exactly-once
  rule standing in for the graph-mutation opacity that bit Cargo, and the
  version-changing need left with pnpm's spec-rewriting overrides.
- **npm's overrides (RFC 0036)** are the cautionary tale for *where* to
  match. npm's version-keyed overrides do try to match resolved versions
  ("a match if the named package specifier would be satisfied by the
  dependency being considered"), but the matching runs during arborist tree
  construction, when nodes may not have versions yet — a timing entanglement
  that produced years of bugs, including two consecutive `npm install` runs
  emitting different lockfiles (broken from 8.3.0, fixed in 11.2.0). The
  mechanism proposed here substitutes at one well-defined point inside a
  single resolver call, after the pick. npm's rule that the override *value*
  also counts as a match (so versions cannot be swapped in cycles) is the
  analogue of this design's substituted-results-are-never-re-matched rule.
- **yarn's `resolutions`** (Berry) match the *declared descriptor* — a
  `foo@^1.0.0` key matches dependents that declared `^1.0.0`, not picks that
  land in it — confirming that every JS package manager's override surface
  is declaration-level today, and none can express "the version resolution
  actually chose" without resolver participation as proposed here.

## Unresolved Questions and Bikeshedding

- **Manifest schema and vendor buy-in.** No vendor-neutral patch-manifest
  standard exists. Do we define `pnpr-patch-manifest/1` and ask vendors to
  emit it, ship per-vendor adapter code, or both? What signature scheme
  (raw ed25519, DSSE envelopes, sigstore)?
- **Patch-scope naming.** Is the scope purely provider-chosen
  (`@echo-patch`, `@seal`), or should pnpr recommend a convention? Scope
  squatting on the public registry is a consideration when the patch source
  is also reachable as a plain npm registry.
- **Alias-override support matrix.** pnpm supports `npm:` aliases in
  overrides; verify the exact selector-matching and alias behavior in npm's
  `overrides` and yarn's `resolutions` so the documented hand-written path
  (range-keyed alias overrides) works, or degrades clearly, across clients.
- **Audit extension field shape.** The exact name and structure of the
  namespaced field carrying the patch mapping in the bulk-advisories
  response, and whether `pnpm audit` should render it without the `--fix`
  follow-up.
- **SBOM/upstream identity mapping.** License and SBOM tooling will record
  `@echo-patch/ejs`; should pnpr expose the manifest mapping (patched name →
  upstream identity) as an endpoint so downstream tooling can resolve it, or
  is the provider's attestation the right carrier?
- **Named-registry encoding timing.** Revisit `echo:ejs@2.7.4`-style
  overrides once the pnpm lockfile registry-identity follow-up lands; the
  annotation could then carry a registry-qualified spec as an alternative
  encoding.
- **Patch-aware resolver design.** The pnpm-side follow-up must settle
  whether annotations are honored by default when present or behind a
  setting, whether they are honored from *any* registry or only from
  configured pnpr bases, the shape of the opt-in and opt-out surfaces
  (accept list for `apply: opt-in` annotations, ignore list for
  `apply: default` ones — per package or per provider), how provenance is
  surfaced (`pnpm why`, install summary, the `resolvedVia` marker), and the
  annotation field name (`_pnprPatch` vs. something vendor-neutral other
  resolvers could adopt).
- **Lockfile recording shape.** The name and format of the lockfile section
  recording substituted entries — it must let satisfiability checks accept
  an aliased resolution the workspace's own config does not explain, and
  distinguish it from a hand-written override — and the adoption-timing
  default: is silently adopting a newly published patch for an
  already-locked pick on the next non-frozen re-resolution the right
  behavior, or should new adoptions require an explicit gesture
  (`pnpm update --patches`-style) while only *withdrawn* patches act
  automatically?
- **Annotation-carried integrity as defense in depth and fast path.** The
  patched content originates from the provider's infrastructure, and pnpr's
  serving-path verification is what stands between a compromised patch
  source and clients. Should the annotation optionally carry the patched
  artifact's manifest-pinned integrity (and manifest essentials) — letting
  the resolver construct the substituted resolution without fetching the
  patch-scope packument, and double-check the bytes against the provider's
  signed manifest even if pnpr's enforcement failed? A redundant check and
  a metadata-size cost; deliberately undecided.
- **Default mode.** Is `advertise` the right default for `patching:`, with
  `substitute` as the explicit escalation? (This RFC assumes yes: changing
  what fresh resolutions receive should be a deliberate, visible deployment
  decision.)
- **Interaction with package screening.** The separate screening RFC's block
  responses could advertise "a patched artifact is available" via the same
  manifest data. That integration is optional in both directions; neither RFC
  depends on the other.
