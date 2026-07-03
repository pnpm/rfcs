# Post-pick dependency substitution (patch-aware resolution)

## Summary

pnpm's resolver gains one primitive: a **post-pick substitution rule** —
"when version resolution picks `name@version`, resolve this alias spec in
its place." Rules come from two sources: **workspace config** (explicit
adoption of registry-advertised security patches, written by hand or by
`pnpm audit --fix`) and **registry annotations** (`_pnprPatch` fields served
by a pnpr registry in substitute mode, applied by default and refusable per
package via an ignore list). A substituted dependency is recorded in the
lockfile as an ordinary alias with a canonical, host-free resolution, plus a
record explaining why the alias exists. Version selection itself never
changes — only which *build* of the selected version is installed.

This is the client half of the pnpr patch-provider design
([`pnpr/text/0000-patch-providers.md`](../pnpr/text/0000-patch-providers.md));
the registry half (patch scopes, signed manifests, annotation serving, audit
enrichment, registry-side enforcement) is specified there. This RFC is
useful on its own — workspace-declared rules require no pnpr — but its
motivating consumer is registry-delivered security patches.

## Motivation

Vendors such as Echo and Seal ship security-patched builds of vulnerable
package versions that keep the **original version number** under a distinct
package name (`@echo-patch/ejs@2.7.4` is the patched build of `ejs@2.7.4`).
Adopting such a patch means: *wherever resolution would select `ejs@2.7.4`,
install the patched build of exactly that version instead.*

pnpm's existing substitution surface cannot express this:

- **Overrides rewrite declared specs, before resolution**, and a selector
  matches a declared range by subset. An exact-version key
  (`"ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"`) covers only dependencies
  declared as exactly `2.7.4` — a dependency declared as `^2.7.0` slips
  past it even when resolution picks `2.7.4`, which is the common case.
- **Widening the selector changes version selection.** No range selector
  matches `^2.7.0` short of the bare name, and a bare-name override forces
  every `ejs` in the graph to the patched `2.7.4` — downgrading a `^2.7.0`
  that would have picked a genuinely fixed `2.7.5`, and freezing the graph
  to a stale patch after upstream moves on.

The condition "resolution picked `2.7.4`" is only knowable *after the
pick*. That is the gap this RFC fills — with a mechanism deliberately
separate from overrides, so existing override semantics change in no way.

## Detailed Explanation

### The rule

A post-pick substitution rule is an exact original version mapped to an
alias spec:

```jsonc
{ "ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4" }
```

Semantics:

- **Applies after the pick.** The resolver selects a version for a spec
  exactly as today — overrides and every other spec-level mechanism have
  already done their work. If the picked `(name, version)` matches a rule,
  the resolver resolves the rule's target spec and returns *that* result
  (id, manifest, resolution) under the dependency's original alias, marked
  `resolvedVia: registry-patch`. This rides the alias/resolved-identity
  split that `npm:` specs already use; nothing downstream of the resolver
  changes.
- **Version selection is never altered.** A rule keyed `ejs@2.7.4` has no
  effect on which version a range selects; it replaces the build of the
  version that *was* selected. (A rule whose target names a different
  version is expressible but is the author's deliberate choice, visible in
  config.)
- **Applied exactly once.** A substituted result is returned as-is and
  never re-matched against the rules, so rules cannot chain or cycle. (npm
  RFC 0036 solves the same problem by counting an override's *value* as a
  match; the applied-once rule is the simpler equivalent.)
- **Frozen installs are untouched.** Rules act only when a resolution is
  made; `--frozen-lockfile` reads no packuments and consults no rules.

### Two sources of rules

1. **Workspace config.** The workspace declares rules directly (config key
   name is bikeshedding, e.g. `securityPatches.accept` or
   `substitutions`). Each entry is self-contained — it carries the full
   mapping — so it works against any registry and pins the patch revision
   the workspace reviewed. This is the adoption surface for pnpr's
   *advertise* mode: `pnpm audit` reports available patches (the registry's
   audit enrichment carries the mapping), and `pnpm audit --fix` copies
   them into config, reviewable in the PR that introduces them.
2. **Registry annotations.** A pnpr registry in *substitute* mode serves the
   mapping as a `_pnprPatch` annotation on the vulnerable version's
   packument entry. To the resolver an annotation is a registry-supplied
   rule, applied by default and refused per package (or per provider)
   through an **ignore list** in workspace config.

When both sources cover the same pick, workspace rules win (they should
normally agree; a workspace rule pinning an older patch revision than the
annotation is respected — it was reviewed).

### Resolution mechanics and cost

Matching is a map lookup per pick — free for the unpatched majority. A
matched pick resolves the target: one exact-version resolution against the
patch-scope packument, which is small (patched versions only), cached like
any other, and fetched in parallel. The target's packument is genuinely
needed — the substituted package's **version object** (dependencies, peers,
`bin`) drives subtree resolution, so the rule is a pointer, not a manifest.
This is a second resolution of *that dependency*, never of the graph, and
the original pick is not wasted work: it is the key that selects the rule.

### Lockfile

Two things are recorded:

- **The ordinary aliased entry.** The importer/snapshot dependency entry
  maps `ejs` → `@echo-patch/ejs@2.7.4`; the package entry is keyed by the
  real identity with a canonical resolution (integrity only, tarball URL
  recomputed from registry config at install time — nothing host-specific).
  Byte-for-byte what a hand-written alias override would have produced.
- **A substitution record** in a dedicated lockfile section: the applied
  mapping plus its source (workspace rule, or annotation + registry). This
  is required for *correctness*, not only provenance: without it, the
  up-to-date check would see `ejs: ^2.7.0` resolved to an alias the
  workspace's own config does not explain and re-resolve it. The record
  lets satisfiability checks accept the entry, lets a withdrawn patch
  re-resolve exactly the affected packages, distinguishes registry
  substitutions from hand-written config, and tells humans and SBOM tooling
  why the alias exists.

### Security model

Rules and ignore lists are **trusted workspace configuration**, the same
trust class as overrides — which can already redirect any package to any
spec, so this feature grants no redirection power that did not exist.
Client-side substitution is a protection and compatibility layer, **not an
enforcement boundary**: a deployment whose policy is "the vulnerable
artifact must not be used" enforces at the registry (pnpr's
`enforcement: refuse` answers the vulnerable tarball with `403` plus the
suggested patch, for every client). Under registry enforcement, an ignore
list can choose failure over substitution but can never obtain the
vulnerable bytes. pnpm's part of that story is rendering the `403` reason
body well: name the advisory, the patched alternative, and the config that
would adopt or ignore it.

### UX surfacing

- `pnpm why ejs` and the install summary show the substitution and its
  source ("`ejs@2.7.4` → `@echo-patch/ejs@2.7.4` via registry patch
  (echo)").
- `pnpm audit` shows available-but-unadopted patches (advertise mode) and
  the adopted ones' status (a `-sp2` re-issue shows as an available
  update to an adopted rule; `--fix` updates it).
- `pnpm licenses`/SBOM tooling see the real patched name, which is the
  point: provenance is legible wherever the name is recorded.

## Rationale and Alternatives

### Extend override semantics instead (exact-version selectors match picks)

Considered at length in the pnpr RFC's history: let an override whose
selector is an exact version also match the *picked* version. It is
strictly additive for exact selectors (the only subset of `2.7.4` is
`2.7.4` itself), but it silently changes the behavior of existing configs —
exact-keyed overrides written under today's semantics would start covering
picks they previously missed — and it entangles adoption with override
precedence. A separate primitive with its own config surface changes
nothing that exists and keeps "what substitutes my picks" answerable from
one place. (If the override extension is ever wanted for its own sake —
hand-written exact-version overrides that catch range-resolved picks — it
can be proposed independently; nothing here conflicts.)

### Apply registry annotations without a workspace-rule counterpart

Then advertise-mode adoption would have to reuse overrides, which cannot
express it (Motivation), or a registry-dictated "opt-in posture" flag on
annotations — rejected because client posture is the workspace's business,
not the registry's. Self-contained workspace rules keep the primitive
symmetric: the registry can *suggest* (annotation) and the workspace can
*declare* (rule), and both meet in the same resolver code path.

### Do nothing (blunt overrides)

Range-keyed overrides (`"ejs@^2.7.0": "npm:@echo-patch/ejs@2.7.4"`) and
bare-name overrides work today and remain documented, but they freeze or
force version selection, silently miss differently-shaped ranges added
later, and go stale when upstream ships a real fix. Acceptable escape
hatches; not a mechanism to build a security workflow on.

## Implementation

All changes are in pnpm (and later pacquet):

1. **Resolver hook.** In `@pnpm/npm-resolver`, after the version pick and
   before the result is assembled: look up `(name, picked version)` in the
   effective rule map (workspace rules ∪ annotations − ignore list, with
   workspace precedence); on a match, resolve the target alias spec and
   return its result under the original alias with the `resolvedVia`
   marker. Enforce applied-once by never rule-matching a substitution
   result.
2. **Config.** Workspace substitution rules and the ignore list (names are
   bikeshedding); a setting governing whether registry annotations are
   honored (and from which registries).
3. **Lockfile.** The substitution-record section; satisfiability-check
   integration; re-resolution of affected packages when a recorded rule or
   annotation disappears or changes.
4. **`pnpm audit` / `--fix`.** Read the mapping extension field from the
   enriched bulk-advisories response; render available and adopted patches;
   `--fix` writes/updates workspace rules.
5. **Errors and reporting.** Render pnpr's `403` reason body (registry
   enforcement) with the suggested adoption config; `pnpm why`/summary
   surfacing.

Tests should cover, at minimum:

- a range-declared dependency (`^2.7.0`) whose pick lands on a ruled
  version installing the substituted build, recorded as an alias with a
  canonical resolution;
- an exact-pinned declaration behaving identically;
- version selection unchanged by rules (a range that picks `2.7.5` is not
  substituted by a `2.7.4` rule);
- the ignore list suppressing an annotation; workspace rules winning over
  a conflicting annotation;
- applied-once: a rule targeting a package that is itself covered by
  another rule does not chain;
- `--frozen-lockfile` neither consulting rules nor changing anything;
- lockfile round-trip: substitution records satisfying the up-to-date
  check, a withdrawn annotation re-resolving only affected packages, and
  hand-written-override results distinguishable from substitutions;
- `pnpm audit --fix` writing rules from the mapping extension and updating
  them on a patch re-issue;
- rendering of the registry-enforcement `403` reason.

## Prior Art

- **Go's versioned `replace`** (`replace ejs v2.7.4 => patched v2.7.4`):
  applies only when MVS selects exactly that version, honored only in the
  main module — post-selection exact-version substitution with workspace
  authority, alive and standard.
- **Cargo's `[replace]`**: exact package-id keyed, same-name-same-version
  source swap on the resolved graph; deprecated in favor of `[patch]`
  because users needed *version-changing* overrides — a need that stays
  with pnpm's spec-rewriting overrides here, keeping post-pick substitution
  for the same-version case where even Cargo's docs equate the two.
- **npm's overrides (RFC 0036)**: version-keyed overrides that try to match
  resolved versions during tree construction — a timing entanglement that
  produced years of bugs (double-install lockfile flapping, fixed only in
  npm 11.2.0). Substituting at one well-defined point inside a single
  resolve call is the lesson applied.
- **yarn's `resolutions`** (Berry): declared-descriptor matching,
  confirming that every JS package manager's substitution surface is
  declaration-level today.
- The registry-side prior art (Seal, Echo, Assured OSS, Socket) is surveyed
  in the pnpr companion RFC.

## Unresolved Questions and Bikeshedding

- **Config naming and shape.** `securityPatches.accept`/`ignore`,
  `substitutions`, or something else; per-package vs. per-provider ignore
  granularity; where the settings live (`pnpm-workspace.yaml`,
  `package.json#pnpm`).
- **Default for annotations.** Honored by default when a registry serves
  them, or behind a setting? From any registry, or only configured pnpr
  bases?
- **Adoption timing for locked picks.** When a new annotation covers an
  already-locked vulnerable pick, should the next non-frozen re-resolution
  adopt it silently, or should adoption require an explicit gesture
  (`pnpm update --patches`-style) while only *withdrawn* patches act
  automatically?
- **Lockfile section format** for substitution records, and its interplay
  with the existing `overrides` record.
- **Annotation field name.** `_pnprPatch` vs. a vendor-neutral name other
  registries and resolvers could adopt.
- **Parent-scoped rules.** Should rules support `parent>child@version`
  scoping like overrides do, or is graph-wide-by-version the right (and
  only) granularity for security patches?
- **Server-side resolution parity.** When pnpr's resolve endpoint performs
  resolution, substitutions arrive pre-applied and marked; the lockfile
  record must come out identical either way.
