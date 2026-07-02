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
applies it in one of two modes: **advertise**, where consumers adopt patches
explicitly via package alias overrides
(`"ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"`) surfaced through audit
enrichment, and **substitute**, where the registry compiles the manifest into
a ready-made **overrides object** served at a well-known endpoint, which
patch-aware clients fetch and apply automatically at resolution — so *fresh
resolutions* receive the patched artifact as an ordinary recorded alias with
a canonical, host-free lockfile resolution, while already-pinned installs are
never altered.

Notably, this requires **no new mount kind** and no changes to how packuments
or tarballs are served. Both shapes are expressible with
hosted/upstream/router mounts as already specified; what this RFC adds is the
manifest protocol, a registry-served overrides document, and the
audit/advisory machinery around it.

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
- **Integrity-pinned.** The manifest's integrity for the patched artifact must
  match what the patch source serves; a mismatch is an error, never a
  fall-through to the vulnerable original.
- **Patch re-issues stay inside the namespace.** If a patch itself needs a
  second revision (a new CVE lands on the same base version), the provider
  publishes `@echo-patch/ejs@2.7.4-sp2` and updates the manifest entry.
  Suffix ordering quirks are harmless inside a dedicated patch namespace,
  because adoption is always an exact pin produced from the manifest — no
  range resolution against upstream expectations ever happens there.

### Advertise mode: explicit adoption via alias overrides

In `advertise` mode, adoption is a **package alias override** in the consuming
workspace — the mechanism pnpm already has for substituting one package for
another:

```jsonc
{
  "pnpm": {
    "overrides": {
      "ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"
    }
  }
}
```

Three adoption paths, from least to most automatic:

1. **Hand-written overrides**, as above. Works today with no pnpr or pnpm
   changes; the alias installs the patched bytes at every point in the graph
   where `ejs@2.7.4` appeared, including transitive positions.
2. **Audit enrichment.** pnpr implements the npm audit endpoints
   (`/-/npm/v1/security/advisories/bulk`) against OSV plus the pinned
   manifests, and reports, per vulnerable version, the exact override spec
   available *on this registry*. The stock response format has no field for
   "an aliased fix exists", so pnpr adds a namespaced extension field to each
   advisory entry carrying the suggested override; clients that do not know
   the field ignore it.
3. **`pnpm audit --fix` (pnpm follow-up, out of scope here).** pnpm reads the
   enriched audit response and writes the alias overrides mechanically. This
   is deliberately a *write-my-config* flow: the override is visible in the
   workspace manifest and lockfile, reviewable in the PR that introduces it,
   and removable when the team upgrades away from the vulnerable base
   version. (Registry-side resolution-time behavior is substitute mode,
   below.)

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

Advertise mode's limit is that it is **opt-in per workspace**: a fresh
`pnpm install` in a repository that has not written the override resolves
`^2.7.0` to the vulnerable original, exactly as before. Repositories that
never run `pnpm audit` never find out. When the deployment's intent is "no
fresh resolution may pick a version we have a patch for", that is substitute
mode.

### Substitute mode: a registry-provided overrides object

Advertise mode's audit enrichment already computes, per vulnerable version,
the exact override a workspace should adopt. `substitute` mode serves that
same result wholesale: the registry compiles the pinned, policy-filtered
manifests into one **overrides object** — in pnpm's own overrides format —
at a well-known endpoint on each registry base that enables it:

```jsonc
// GET <registry base>/-/pnpr/overrides
{
  "schema": "pnpr-overrides/1",
  "overrides": {
    "ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4",
    "lodash@4.17.20": "npm:@echo-patch/lodash@4.17.20"
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
      minSeverity: high      # only include fixes at/above this severity
```

A patch-aware client (a small pnpm follow-up) fetches the document once per
resolution run and merges it at the **lowest precedence** — anything the
workspace declares wins — exactly as if the workspace had written those
overrides by hand. Everything downstream is pnpm's already-shipped alias
machinery, and that is the point:

- **The lockfile stays canonical and host-free.** A fresh resolution of
  `^2.7.0` picks `2.7.4`, the override redirects it, and the lockfile records
  the aliased package `@echo-patch/ejs@2.7.4` with a canonical resolution —
  integrity only, tarball URL recomputed from registry config at install
  time. Nothing deployment-specific is persisted; portability is exactly that
  of any aliased dependency.
- **No pinned install ever changes.** Overrides act only when a resolution is
  made. An existing lockfile entry is never touched, and a
  `--frozen-lockfile` install neither fetches nor applies the document.
  Substitute at resolution, never at fetch.
- **Provenance is legible and diffable.** The lockfile shows `ejs` resolving
  to `@echo-patch/ejs@2.7.4`; when a manifest refresh moves a patch to
  `-sp2`, the next non-frozen resolution produces a reviewable lockfile diff.
- **Opting out is ordinary config.** A workspace that must keep the
  vulnerable original pins it with its own override, which outranks the
  registry-provided one — visible and reviewable, like every other override.

Clients that are not patch-aware never ask for the document and resolve the
vulnerable original — no worse than today. Per-mount policy can escalate:
`unawareClients: refuse` makes the tarball route answer a manifest-covered
vulnerable artifact with the explicit `403`-plus-suggested-override body
instead of bytes. That is hard enforcement, stated honestly: the tarball
route cannot distinguish a fresh resolution from an old pinned one, so
`refuse` also blocks existing lockfiles that pin the vulnerable original —
which is precisely what "no known-patchable vulnerable artifact leaves this
registry" means. The default is `serve`.

Costs and requirements, stated honestly:

- **Automatic protection reaches only patch-aware clients.** npm and yarn
  users get advertise-mode discovery (and `refuse`, where enabled). pnpr is
  pnpm-first, and the document is deliberately pnpm's overrides shape so
  other tools can consume it too — an org can even paste it into a shared
  config today.
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

Determinism and conflicts: the served overrides object is a pure function of
the mount's `patching:` config plus the pinned manifest snapshots — the same
determinism contract as router routes. A refresh that changes it is logged
and diffable, and the document carries a digest/ETag so clients cache it
cheaply. Within one mount, at most one provider may claim a given original
`name@version`; two manifests colliding is a config error, not a precedence
guess. And substitution composes with advertise-mode machinery — the audit
endpoint still reports the mapping, so a workspace can see which of its
resolutions came from the registry's overrides and pin them explicitly if it
prefers.

**Document size.** The document grows linearly with the provider's patch
catalog, and that catalog is advisory-driven: an entry exists only for an
exact `name@version` that has a backported fix, so growth tracks CVE history
in patched packages, not the registry. Realistic catalogs are on the order of
10³–10⁴ entries; at roughly 80 bytes per entry that is sub-megabyte raw, and
the shape (one scope prefix, repeated names) compresses to tens of kilobytes.
Policy filtering (`minSeverity`, `requireAdvisoryMatch`) trims it further.
Delivery cost is bounded by the caching contract rather than the size: the
document changes only on manifest refresh or config reload, so a client
caches it machine-wide per registry base (shared across workspaces, like the
store) and revalidates with `If-None-Match` — the steady-state cost of
substitute mode is one `304` per resolution run. Should a catalog ever
outgrow single-document delivery, the escape hatch is a filtered bulk lookup
shaped like the audit bulk endpoint (the client posts the names it is
resolving); that variant is deliberately out of scope until real catalog
sizes demand it.

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

- **It synthesizes metadata.** The overlay serves a packument that no single
  origin published — base metadata with foreign *version entries* spliced
  into the version list. The alias model never modifies a served packument in
  any mode: the overrides document lives beside the registry surface, not
  inside another origin's metadata — yet grafting buys neither automatic
  protection (last bullet below) nor a cleaner adoption story.
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

### Serving the substitution inside packuments

Two earlier shapes of substitute mode put the mapping into the served
packument itself instead of a separate overrides document:

- **Rewrite the vulnerable version's `dist`** to the patched artifact's
  tarball URL and integrity. Its appeal is universality — every npm client
  gets auto-patched with no client feature — but the patched URL is
  non-canonical for the original name, so clients persist the full URL,
  baking the deployment host into lockfiles: exactly what the mounts RFC's
  canonical-URL round-tripping exists to prevent. The lockfile records an
  opaque URL where the alias encoding records a first-class package identity
  with a canonical, recomputable resolution.
- **Annotate the vulnerable version** with an extension field
  (`_pnprPatch: {patched, integrity, ...}`) for patch-aware clients to act
  on. Lockfile-clean, but it still injects nonstandard fields into another
  origin's metadata on the serving path, and the client must discover
  patches one packument at a time.

The registry-provided overrides object subsumes both: the same information,
one standard-shaped document beside the registry surface, zero changes to
packument or tarball serving — and the client-side behavior it enables is
identical to overrides the workspace could have written itself.

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

All registry-side changes are in pnpr-server, and none of them touch the
mount graph, routing, or the packument/tarball serving paths:

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
4. **Per-mount `patching:` policy and the overrides endpoint.** Validate the
   policy block (known provider, mode, severity gate, `unawareClients`, one
   substituting provider per original `name@version` per mount); compile the
   pinned, policy-filtered manifests into the `/-/pnpr/overrides` document
   with a digest/ETag, recompiled only on manifest refresh or config reload;
   implement `unawareClients: refuse` on the tarball route with the `403`
   reason body.

Client-side follow-ups (separate pnpm RFC): `pnpm audit --fix` writing alias
overrides from the enriched audit response, and fetch-and-merge of the
registry-provided overrides document at lowest precedence during resolution.

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
- the overrides document compiled from pinned manifests, filtered by
  `minSeverity`, stable between refreshes, and served with a digest/ETag;
- packuments and tarball responses byte-identical with `patching:` enabled
  and disabled — substitute mode touches neither serving path;
- an existing lockfile pinning the vulnerable original installing unchanged
  through a substitute-mode mount (with `unawareClients: serve`);
- `unawareClients: refuse` answering manifest-covered vulnerable tarballs
  with `403` plus the suggested override, and leaving uncovered artifacts
  untouched;
- the one-substituting-provider-per-original rule enforced at config load;
- a manifest refresh changing the overrides document being logged, and
  previously aliased patched artifacts continuing to serve as long as the
  provider retains them;
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
  overrides to many repositories through an installed package; the
  registry-provided overrides document is the same idea with the registry as
  the distribution channel, and the two can coexist (an org can compile the
  document into a config dependency for clients that predate the endpoint).
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
  overrides; verify the exact behavior in npm's `overrides` and yarn's
  `resolutions` so the audit-suggested spec works (or degrades clearly)
  across clients.
- **Audit extension field shape.** The exact name and structure of the
  namespaced field carrying the suggested override in the bulk-advisories
  response, and whether `pnpm audit` should render it without the `--fix`
  follow-up.
- **SBOM/upstream identity mapping.** License and SBOM tooling will record
  `@echo-patch/ejs`; should pnpr expose the manifest mapping (patched name →
  upstream identity) as an endpoint so downstream tooling can resolve it, or
  is the provider's attestation the right carrier?
- **Named-registry encoding timing.** Revisit `echo:ejs@2.7.4`-style
  overrides once the pnpm lockfile registry-identity follow-up lands; the
  overrides document could then carry registry-qualified specs as an
  alternative encoding.
- **Overrides endpoint discovery and trust.** The exact path
  (`/-/pnpr/overrides` vs. a capabilities document), whether the document is
  signed with the pnpr instance key in addition to being served over the
  authenticated registry connection, and how a client maps multiple
  configured registries to multiple documents (fetch from each base? default
  registry only?).
- **pnpm-side application semantics.** Does pnpm apply a registry-provided
  overrides document by default when the registry offers one, or behind a
  setting? How is its provenance surfaced (`pnpm why`, lockfile comment,
  install summary)? These belong in the pnpm follow-up RFC, but the answers
  shape the document format.
- **Scale threshold for a filtered variant.** At what measured catalog size
  (entries, compressed bytes) does the single cached document stop being the
  right delivery, and is the fallback a name-filtered bulk lookup, a
  chunked/delta encoding, or both?
- **Default mode.** Is `advertise` the right default for `patching:`, with
  `substitute` as the explicit escalation? (This RFC assumes yes: changing
  what fresh resolutions receive should be a deliberate, visible deployment
  decision.)
- **Interaction with package screening.** The separate screening RFC's block
  responses could advertise "a patched artifact is available" via the same
  manifest data. That integration is optional in both directions; neither RFC
  depends on the other.
