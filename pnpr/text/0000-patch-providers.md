# Patch-provider integration for pnpr

## Summary

Vendors such as Echo, Seal Security, and Socket ship security-patched builds
of vulnerable open-source package versions and want to deliver them through
the registry the organization already uses. pnpr should support this in two
shapes: a vendor registry as an ordinary **upstream mount** (works today, the
vendor is the declared origin), and a **patch scope** — the provider publishes
patched artifacts under its own package namespace (`@echo-patch/ejs@2.7.4`),
served through ordinary mounts and routes, and consumers adopt them via
**package alias overrides** (`"ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"`). A
signed, pinned **patch manifest** ties the two names together, driving audit
enrichment and advisory mapping.

Notably, this requires **no new mount kind**. Both shapes are expressible with
hosted/upstream/router mounts as already specified; what this RFC adds is the
manifest protocol and the audit/advisory machinery around it.

Both shapes extend the invariants of the registry-mounts RFC rather than
weakening them:

> **Provenance is declared, never inferred.** Every `name@version` resolves to
> exactly one declared concrete origin. A patched artifact is a *different
> name* from a *declared provider namespace* — never a request-time
> substitution behind the original name.
>
> **An identity's bytes are immutable.** pnpr never serves different bytes
> under a `name@version` a lockfile may already pin. Choosing patched bytes
> means choosing either a different declared origin (mount) or a distinct
> package identity — never a silent byte swap.

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

No new mount kind, no synthesized packuments, no fall-through anywhere: the
patch scope is just packages, routed by the existing router rules, cached in
the existing per-mount namespaces, gated by the existing access policies. The
new configuration surface is `patchProviders:`, which registers the
**manifest** that ties the two namespaces together.

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

### How clients adopt patched versions

Adoption is a **package alias override** in the consuming workspace — the
mechanism pnpm already has for substituting one package for another:

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
   is deliberately a *write-my-config* flow, not a resolution-time behavior:
   the override is visible in the workspace manifest and lockfile, reviewable
   in the PR that introduces it, and removable when the team upgrades away
   from the vulnerable base version.

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
  origin published — base metadata with foreign versions spliced in. That is
  declared and deterministic, but it is still the one place in the design
  where pnpr manufactures a document blending two origins, and the alias
  model needs nothing like it.
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

Providers that already ship same-name prerelease patches (Seal's `-spN`) are
still fully served: their artifact server is a vendor-as-origin upstream
mount, unchanged.

### Same-version byte substitution behind the original name

Rejected in any pnpr-mediated form. Serving provider bytes under the base
origin's `name@version` means the same identity yields different integrity
depending on when it was first resolved — breaking lockfile verification for
existing consumers the moment a patch appears, poisoning shared caches with
ambiguity about which bytes are "the" version, and hiding the substitution
from every human reading a lockfile. Organizations that want same-version
patching have a coherent home for it: make the vendor the declared origin
(upstream mount), where the vendor owns the identity end to end.

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

All registry-side changes are in pnpr-server, and none of them touch the mount
graph, routing, or serving paths:

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

Client-side follow-ups (separate pnpm RFC): `pnpm audit --fix` writing alias
overrides from the enriched audit response.

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
  overrides once the pnpm lockfile registry-identity follow-up lands.
- **Interaction with package screening.** The separate screening RFC's block
  responses could advertise "a patched artifact is available" via the same
  manifest data. That integration is optional in both directions; neither RFC
  depends on the other.
