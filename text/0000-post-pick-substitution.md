# Exact-version overrides apply to resolved versions

## Summary

Two related changes to pnpm's resolution, forming the client half of the
pnpr patch-provider design
([`pnpr/text/0000-patch-providers.md`](../pnpr/text/0000-patch-providers.md)):

1. **A semantics fix to overrides.** An override whose selector is an exact
   version also applies when version resolution **picks** that version.
   Today such an override only rewrites declared specs, matched by range
   subset — so `"ejs@2.7.4": "npm:@echo-patch/ejs@2.7.4"` covers a
   dependency declared as `2.7.4` but silently misses every `^2.7.0` that
   resolves to `2.7.4`, which is almost never what its author meant. The
   fix is always on: it strictly widens exact-selector overrides to the
   picks they were written for, and changes nothing else.
2. **Registry-supplied overrides.** A pnpr registry in substitute mode
   annotates a vulnerable version's packument entry (`_pnprPatch`) with the
   mapping to its patched build. The resolver treats such an annotation as
   a registry-supplied exact-version override at the **lowest precedence**:
   any workspace override claiming the same pick — including a self-mapping
   that keeps the original — outranks it.

A substituted dependency is recorded in the lockfile as an ordinary alias
with a canonical, host-free resolution; workspace-side substitutions are
ordinary overrides, recorded and change-detected by the machinery that
records overrides today. Version selection itself never changes — only
which *build* of the selected version is installed.

## Motivation

Vendors such as Echo and Seal ship security-patched builds of vulnerable
package versions that keep the **original version number** under a distinct
package name (`@echo-patch/ejs@2.7.4` is the patched build of `ejs@2.7.4`).
Adopting such a patch means: *wherever resolution would select `ejs@2.7.4`,
install the patched build of exactly that version instead.*

Today's override semantics cannot express this:

- **Overrides rewrite declared specs, before resolution**, and a selector
  matches a declared range by subset. An exact-version key covers only
  dependencies declared as exactly `2.7.4` — a dependency declared as
  `^2.7.0` slips past it even when resolution picks `2.7.4`, which is the
  common case.
- **Widening the selector changes version selection.** No range selector
  matches `^2.7.0` short of the bare name, and a bare-name override forces
  every `ejs` in the graph to the patched `2.7.4` — downgrading a `^2.7.0`
  that would have picked a genuinely fixed `2.7.5`, and freezing the graph
  to a stale patch until someone remembers to remove it.

The condition "resolution picked `2.7.4`" is only knowable *after the
pick*. Applying exact-version overrides there gives them a property no
spec-level encoding has: **they never freeze**. When upstream publishes a
fixed `2.7.5` and ranges start picking it, the `2.7.4` override simply goes
inert — the graph moves forward on its own, and the override becomes dead
config to clean up, not a pin holding the graph back.

## Detailed Explanation

### The fix

An override with an exact-version selector, `"ejs@2.7.4": <target>`,
applies in two situations:

1. **To declared specs, by subset** — exactly as today. (The only range
   that is a subset of `2.7.4` is `2.7.4` itself, so this leg is
   unchanged.)
2. **To the picked version, after resolution** — new. The resolver selects
   a version for a spec exactly as it does today, with every spec-level
   mechanism (including overrides' existing rewriting) already applied. If
   the picked `(name, version)` matches an exact-version override, the
   resolver resolves the override's target and returns *that* result — id,
   manifest, resolution — under the dependency's original alias. This rides
   the alias/resolved-identity split that `npm:` specs already use; nothing
   downstream of the resolver changes.

Rules that keep it predictable:

- **Version selection is never altered.** A `2.7.4` selector has no effect
  on which version a range picks; it replaces the build of the version that
  *was* picked. (A target naming a different version is expressible — it is
  the author's explicit, visible choice, same as today's overrides.)
- **Applied exactly once.** A substituted result is returned as-is and
  never re-matched against overrides, so mappings cannot chain or cycle.
  (npm's RFC 0036 solves the same problem by counting an override's *value*
  as a match; applied-once is the simpler equivalent.)
- **Frozen installs are untouched.** Overrides act only when a resolution
  is made; `--frozen-lockfile` reads no packuments and applies nothing.

### A fix, not a breaking change

The newly covered cases are precisely those where an exact-version override
previously did **nothing** — silently, with no warning — despite the
author's evident intent that "this version" means "this version, wherever
it comes from." npm's own overrides RFC specifies pick-matching intent for
version-keyed rules, and the miss is regularly reported as a bug. Shipping
the fix always-on, with a changelog note, is therefore the right default; a
workspace that genuinely relied on an exact-key override *not* covering
range-resolved picks (a contrived reading) can key by the declared spec
instead. No flag, no major-version gate.

### Registry-supplied overrides (pnpr substitute mode)

A pnpr registry in substitute mode serves the patch mapping as an
annotation on the vulnerable version's packument entry:

```jsonc
"2.7.4": {
  // ...original metadata, original dist...
  "_pnprPatch": {
    "patched": "npm:@echo-patch/ejs@2.7.4",
    "provider": "echo",
    "fixes": ["GHSA-..."]
  }
}
```

To the resolver this is an exact-version override supplied by the registry,
merged at the **lowest precedence**:

- if any workspace override claims the same pick, the workspace wins — in
  particular, a self-mapping (`"ejs@2.7.4": "npm:ejs@2.7.4"`) is the opt-out
  that keeps the vulnerable original, expressed in ordinary override
  vocabulary, visible and reviewable;
- otherwise the annotation applies exactly like the fix above, marked
  `resolvedVia: registry-patch` in reporting.

A client setting governs whether annotations are honored (and from which
registries); the default is an open question below. Note that client-side
substitution is a protection layer, not the enforcement boundary — a
deployment that must prevent vulnerable artifacts entirely uses pnpr's
registry-side `enforcement: refuse`, under which declining substitution can
only fail an install, never obtain the vulnerable bytes. pnpm's job there
is rendering the `403` reason well: the advisory, the patched alternative,
and the override that would adopt it.

### Resolution mechanics and cost

Matching is a map lookup per pick — free for the unpatched majority. A
matched pick resolves the target: one exact-version resolution against the
patch-scope packument, which is small (patched versions only), cached like
any other, and fetched in parallel. The target's packument is genuinely
needed — the substituted package's **version object** (dependencies, peers,
`bin`) drives subtree resolution, so the override target is a pointer, not
a manifest. This is a second resolution of *that dependency*, never of the
graph, and the original pick is not wasted work: it is the key that selects
the override.

### Lockfile

Nothing new is required:

- **The aliased entry** is ordinary: the importer/snapshot dependency entry
  maps `ejs` → `@echo-patch/ejs@2.7.4`, the package entry is keyed by the
  real identity with a canonical resolution (integrity only, tarball URL
  recomputed from registry config) — byte-for-byte what a hand-written
  alias override produces, because it *is* one.
- **Workspace-side change detection** is the existing overrides record: pnpm
  already stores the overrides a resolution used and re-resolves when they
  change; the fix rides that machinery untouched.
- **Annotation-produced substitutions** persist the way any locked
  resolution does: the lockfile is authoritative until an ordinary
  re-resolution event touches the entry. Whether the up-to-date check needs
  a minimal marker to avoid spurious re-resolution of an aliased entry it
  cannot attribute to workspace config — and how a *withdrawn* patch should
  revert — is left to implementation (see Unresolved Questions); no
  explanatory lockfile section is proposed.

### `pnpm audit --fix`

pnpr's audit enrichment carries the mapping per vulnerable version.
`--fix` writes it as an ordinary exact-version override. Because of the
fix's post-pick semantics this is correct for range-resolved picks, and
because exact selectors never freeze, the override ages gracefully: a
provider re-issue (`-sp2`) shows up in the next `pnpm audit` and `--fix`
updates the entry; an upstream fixed release makes it inert.

## Rationale and Alternatives

### A separate substitution-rules config (this RFC's earlier draft)

A dedicated post-pick rules list (with accept/ignore lists for registry
annotations) avoids touching override semantics — but it duplicates
overrides: a second way to say "replace this package," with its own
precedence rules, its own opt-out surface, and its own lockfile recording.
Once the exact-version fix is accepted as a fix, the existing override
surface provides identical power with none of the parallel machinery, and
workspace authority over registry annotations falls out of ordinary
override precedence.

### Keep today's override semantics unchanged

Rejected: spec rewriting cannot see the pick (Motivation), so exact-version
overrides remain silently inert for the dominant range-declared case, and
`pnpm audit --fix` has no correct output format.

### Gate the fix behind a setting or major release

Rejected per the "fix, not breaking change" argument above: the changed
cases are silent no-ops that contradicted author intent, the risk of
someone depending on the no-op is remote and has a trivial migration (key
by declared spec), and gating would leave `audit --fix` output broken for
everyone who has not opted in.

### Registry-dictated application posture

An earlier design had the annotation carry an `apply: default | opt-in`
field. Rejected: client posture is the workspace's business, not the
registry's. The registry's two postures are expressed by serving or not
serving annotations (substitute vs. advertise mode); the workspace's, by
override precedence.

## Implementation

1. **Resolver.** In `@pnpm/npm-resolver`, after the version pick: look up
   `(name, picked version)` in the effective exact-version override map
   (workspace overrides, then registry annotations if honored); on a match,
   resolve the target spec and return its result under the original alias
   with a `resolvedVia` marker. Never re-match a substitution result.
2. **Overrides plumbing.** Exact-version selectors flow to the resolver
   (today they live only in the manifest-rewrite hook); precedence:
   workspace over annotation.
3. **Settings.** Whether registry annotations are honored, and from which
   registries.
4. **`pnpm audit` / `--fix`.** Read the mapping extension from the enriched
   bulk-advisories response; render available/adopted patches; write and
   update exact-version overrides.
5. **Reporting.** `pnpm why`/install summary show substitutions and their
   source; render pnpr's `403` enforcement body with the adopting override.

Tests should cover, at minimum:

- a range-declared dependency (`^2.7.0`) whose pick lands on an
  exact-version override installing the substituted build, recorded as an
  alias with a canonical resolution;
- an exact-pinned declaration behaving identically (and identically to
  today);
- version selection unchanged: a range that picks `2.7.5` is untouched by a
  `2.7.4` override — no freezing;
- a workspace self-mapping override keeping the vulnerable original against
  a registry annotation (precedence opt-out);
- applied-once: no chaining through overrides whose targets are themselves
  covered by other overrides;
- `--frozen-lockfile` untouched;
- overrides-record change detection re-resolving when an exact-version
  override is added, changed, or removed;
- `pnpm audit --fix` writing and updating overrides from the mapping
  extension;
- rendering of the registry-enforcement `403` reason.

## Prior Art

- **npm's overrides (RFC 0036)** specify that a version-keyed rule matches
  "if the named package specifier would be satisfied by the dependency
  being considered" — pick-matching *intent* — but implement it during
  arborist tree construction, when nodes may lack versions, producing years
  of timing bugs (double-install lockfile flapping, fixed only in 11.2.0).
  Matching at one well-defined point after the pick is the same semantics
  with the failure mode removed — and npm's rule that an override's value
  also counts as a match is the analogue of applied-once.
- **Go's versioned `replace`** (`replace ejs v2.7.4 => patched v2.7.4`)
  applies only when MVS selects exactly that version and is honored only in
  the main module: post-selection exact-version substitution with workspace
  authority, alive and standard.
- **Cargo's `[replace]`**: exact package-id keyed, same-name-same-version
  source swap on the resolved graph; deprecated for `[patch]` because users
  needed *version-changing* overrides — which stay with pnpm's spec-level
  overrides here.
- **yarn's `resolutions`** (Berry) match declared descriptors only,
  confirming the gap is ecosystem-wide.
- The registry-side prior art (Seal, Echo, Assured OSS, Socket) is surveyed
  in the pnpr companion RFC.

## Unresolved Questions and Bikeshedding

- **Default for annotations.** Honored by default when a registry serves
  them, or behind a setting? From any registry, or only configured pnpr
  bases?
- **Up-to-date semantics for annotation-produced entries.** Does the
  existing lockfile suffice (authoritative until an ordinary re-resolution
  event), or does the up-to-date check need a minimal marker so it does not
  re-resolve an aliased entry it cannot attribute to workspace config? How
  does a withdrawn patch revert — next re-resolution, or an explicit
  gesture?
- **Adoption timing for locked picks.** When a new annotation covers an
  already-locked vulnerable pick, should the next non-frozen re-resolution
  adopt it silently, or should adoption require an explicit gesture
  (`pnpm update --patches`-style)?
- **Parent-scoped selectors.** Should `parent>child@1.2.3` selectors gain
  the same post-pick behavior, or is graph-wide-by-version the only sane
  granularity?
- **Annotation field name.** `_pnprPatch` vs. a vendor-neutral name other
  registries and resolvers could adopt.
- **Server-side resolution parity.** When pnpr's resolve endpoint performs
  resolution, substitutions arrive pre-applied and marked; the client-side
  outcome must be identical either way.
