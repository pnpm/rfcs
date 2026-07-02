# Patch-provider integration for pnpr

## Summary

Vendors such as Echo, Seal Security, and Socket ship security-patched builds
of vulnerable open-source package versions and want to deliver them through
the registry the organization already uses. pnpr should support this in two
shapes: a vendor registry as an ordinary **upstream mount** (works today, the
vendor is the declared origin), and a new **patch overlay mount** that grafts
a provider's signed patch manifest onto a base mount, substituting per-version
and deterministically.

Both shapes extend the invariants of the registry-mounts RFC rather than
weakening them:

> **Provenance is declared, never inferred — now per version.** Every
> `name@version` resolves to exactly one declared concrete origin, selected by
> pinned configuration and manifest data, never by request-time existence or
> availability.
>
> **Within one mount, a version's bytes are immutable.** pnpr never serves
> different bytes under a `name@version` a lockfile may already pin. Choosing
> patched bytes means choosing either a different declared origin (mount) or a
> distinct version identifier — never a silent byte swap.

## Motivation

### The registry is the right choke point

Every artifact an organization installs passes through its registry — every
developer machine, every CI job, every transitive dependency. A remediation
applied there is applied once, centrally, with no per-repository configuration
and no gaps for the repo that forgot to install a wrapper CLI or adopt an
override.

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
identity another origin defined.

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

### Patch overlay mounts

For the "only the patches" contract, this RFC adds one new composite mount
kind. An **overlay** wraps a base concrete mount and grafts in patched
versions from a patch source, driven by a signed, pinned **patch manifest**:

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

  safe-npm:
    type: overlay
    base: npmjs
    patchSource: echo-patches
    manifest:
      url: https://npm.echo.example/pnpr-manifest.json
      publicKeys:
        - ed25519:AAAA...
      refreshInterval: 1h
    policy:
      requireAdvisoryMatch: true   # a patch must claim to fix a known advisory
```

The **patch manifest** is a signed document published by the provider mapping
original versions to patched artifacts:

```jsonc
{
  "schema": "pnpr-patch-manifest/1",
  "entries": {
    "ejs@2.7.4": {
      "patchedVersion": "2.7.4-sp1",       // MUST differ from the original
      "integrity": "sha512-...",            // of the patched tarball
      "fixes": ["GHSA-...", "CVE-2024-..."],
      "attestations": ["https://..."]       // provenance, VEX, build info
    }
  }
}
```

Semantics, in the order the invariants demand:

- **Distinct identity, always.** A manifest entry whose `patchedVersion`
  equals the original version is rejected at manifest validation. The overlay
  never swaps bytes under an existing version identifier; the patched artifact
  is a new version (`2.7.4-sp1`-style) with its own integrity, visible as
  itself in packuments and lockfiles. Same-version substitution is the
  vendor-as-origin shape above, never an overlay behavior.
- **Pinned, not probed.** pnpr fetches and signature-verifies the manifest on
  its refresh interval and pins a snapshot. Between refreshes, routing is a
  pure function of config plus the pinned snapshot — a request never asks the
  provider "do you have a patch for this?" at resolution time. A manifest
  refresh is a logged, diffable event, like an operator config change.
- **Per-version, single origin.** The overlay serves the base packument with
  the manifest's patched versions grafted in as additional versions. A
  tarball request for a patched version routes to `patchSource`; every other
  version routes to `base`. Each version has exactly one origin, and the
  overlay owns no bytes of its own (like a router).
- **No fall-through, either direction.** If `patchSource` is down or returns
  not-found for a manifest-listed artifact, that is an error — pnpr does not
  quietly serve the vulnerable base artifact instead. If the fetched patched
  tarball fails the manifest integrity, that is an error. The base being down
  is equally final for base-routed versions.
- **dist-tags are the base's.** The overlay does not repoint `latest` or any
  other tag at patched versions. Adoption is explicit (below), not a tag
  surprise.
- **Read-only.** Publish, dist-tag, and unpublish through an overlay are
  rejected with the same "name a hosted mount" error a router gives for
  upstream-routed writes.
- **Access composes from the sources.** Consistent with
  authorize-at-the-concrete-source: `base` and `patchSource` each enforce
  their own access policy, and the overlay may add its own `access` for the
  composed surface, but a gated patch source is protected at its own URL
  regardless of the overlay in front of it.

### How clients adopt patched versions

Grafted versions are deliberately inert until a client asks for them. Three
adoption paths, from least to most automatic:

1. **Explicit ranges/overrides.** A workspace pins `ejs@2.7.4-sp1` or writes a
   pnpm override, exactly as Seal customers do today. Works with no client
   changes.
2. **Audit enrichment.** pnpr implements the npm audit endpoints
   (`/-/npm/v1/security/advisories/bulk`) against OSV plus the pinned
   manifests, and reports, per vulnerable version, the patched version
   available *on this registry*. `pnpm audit` then shows an actionable fix
   that does not require a semver upgrade.
3. **Client resolution preference (pnpm follow-up, out of scope here).** A
   `preferPatched` resolution mode in pnpm/pacquet: when a range resolves to a
   version for which the registry advertises a patched variant, resolve the
   variant instead. Opt-in, recorded in the lockfile as the concrete patched
   version, so the choice is visible and reproducible. This needs care with
   semver ordering (below) and belongs in a pnpm RFC.

One sharp edge to standardize: **semver pre-release ordering.**
`2.7.4-sp1 < 2.7.4`, so ordinary ranges like `^2.7.0` do *not* match patched
versions — which is precisely why they are safe to graft into packuments (no
existing range resolution changes), and why adoption paths 1–3 are explicit.

A second sharp edge: **advisory screening of patched versions.** OSV ranges
like `affected: < 2.7.5` *do* match `2.7.4-sp1` textually. pnpr's advisory
screening must subtract the manifest's `fixes` list (the provider's VEX claim)
for that exact artifact — pinned by integrity — so a patched artifact is not
blocked or flagged by the very advisory it fixes, while remaining subject to
every other advisory.

## Rationale and Alternatives

### Same-version byte substitution in overlays (Echo's model, overlay-shaped)

Rejected. Serving provider bytes under the base origin's `name@version` means
the same identity yields different integrity depending on when it was first
resolved — breaking lockfile verification for existing consumers the moment a
patch appears, poisoning shared caches with ambiguity about which bytes are
"the" version, and hiding the substitution from every human reading a
lockfile. Organizations that want same-version patching have a coherent home
for it: make the vendor the declared origin (upstream mount), where the vendor
owns the identity end to end. The overlay's value is precisely that patched
artifacts are *visible, distinct, opt-in* versions.

### Proxy the provider's packument and merge it live with the base

Rejected. Live-merging two origins' metadata re-creates the
existence-based-merge model the mounts RFC removed — the composition would
depend on the provider's availability and current contents at request time.
The pinned signed manifest keeps composition deterministic between refreshes,
verifiable, and diffable.

### Leave patching entirely client-side (overrides, Socket-style repo patches)

Not sufficient alone. Repo-local patching (Socket Certified Patches, pnpm
`patchedDependencies`, overrides) is excellent for application teams and needs
no infrastructure — but it protects one repository at a time, drifts across
hundreds of repos, and gives a security organization no central enforcement or
inventory. Registry-side delivery and repo-side adoption compose; this RFC
supplies the registry half and the audit enrichment that makes the client half
discoverable.

### Only support vendor-as-origin, no overlay

Simplest possible scope — and it is the shipped baseline regardless. But it
forces an all-or-nothing trust decision: to get one vendor's patches, the
deployment must make that vendor the origin for every routed package. The
overlay exists so the base origin (say, npmjs) keeps serving everything except
the specific, signed, advisory-matched artifacts the provider patched.

## Implementation

All changes are in pnpr-server; no pnpm client change is required for the
baseline or for overlay adoption path 1 (explicit versions/overrides).

1. **Overlay mount kind.** Extend the mount config model with `type: overlay`
   (`base`, `patchSource`, `manifest`, `policy`); validate at load that base
   and patchSource are concrete mounts (not routers/overlays), and that no
   router route uses an overlay as a write target. Manifest
   fetch/verify/pin machinery with logged refresh diffs; reject entries with
   `patchedVersion == version` or missing integrity.
2. **Packument grafting and per-version routing.** Serve base packuments with
   pinned-manifest versions grafted in; `dist.tarball` stays canonical for the
   addressed base (mounts RFC rules); tarball requests route patched version
   ids to `patchSource` and verify manifest integrity, all other versions to
   `base`. No fall-through on error in either direction.
3. **Audit endpoint.** Implement/extend the npm advisories bulk endpoint from
   OSV data, with VEX subtraction for manifest-fixed advisories and "patched
   version available" enrichment.

Tests should cover, at minimum:

- overlay config validation (non-concrete base/patchSource, same-version
  manifest entries, unsigned/tampered manifests, write rejection);
- grafted packuments carrying both origins' versions with canonical tarball
  URLs, and per-version routing to the correct source;
- patch-source not-found/unavailable and integrity mismatch surfacing as
  errors, never serving the vulnerable base artifact;
- manifest refresh changing routing only at refresh, with the change logged;
- dist-tags passing through from the base unmodified;
- access enforced independently at base, patchSource, and overlay;
- VEX subtraction: patched artifact not flagged by fixed advisories, still
  flagged by unrelated ones;
- audit bulk endpoint advertising the patched version for a vulnerable
  version with a manifest entry.

## Prior Art

- **Seal Security** is the closest model for the overlay: patched versions as
  distinct `-spN` identifiers served from a dedicated artifact source and
  adopted explicitly by the consumer. The overlay generalizes this into a
  provider-neutral signed-manifest protocol.
- **Echo Libraries** validates the vendor-as-origin shape: a hardened,
  signed artifact pipeline that mirrors into existing repository managers,
  patching on the same version number. That same-version model is exactly why
  it maps to a declared origin rather than an overlay.
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
- **Patched version identifiers.** Adopt the provider's identifier verbatim
  (`-sp1`, `-echo.1`) or normalize to a pnpr-defined suffix? Pre-release
  suffixes have the right semver ordering properties but interact with
  tooling that treats any pre-release as unstable. npm rejects build
  metadata (`+meta`) on publish, so that alternative is likely closed —
  verify.
- **`preferPatched` client resolution** and `pnpm audit --fix` writing
  overrides to patched versions belong in a pnpm-side RFC; what does pnpr
  need to expose (beyond audit enrichment) to make those trivial?
- **Interaction with package screening.** A separate RFC proposes a screening
  pipeline whose block responses could advertise "a patched version of this
  artifact is available" when an overlay manifest knows one. That integration
  is optional in both directions; neither RFC depends on the other.
