# Release notes: what changed in a version, served by the registry

## Summary

pnpr stores release notes at two levels, in the same way for every ecosystem
it hosts: a note for each package version, and a note for each
[publication](./0000-atomic-workspace-publication.md), the set of versions,
npm packages, crates, and wheels alike, that one publish made visible
together. Notes are Markdown, sent with the publish and editable later. A
version's notes are served next to its other routes in its ecosystem, one
version at a time or all the versions between two, so a client upgrading from
`1.2.0` to `1.4.0` reads what changed in one request. A publication's note is
served from the server root, outside every ecosystem. Notes are not part of
any package document, release record, or archive, so an install never
downloads them and editing them changes nothing a lockfile pins. pnpm sends
notes with `pnpm publish` and shows them where it offers an upgrade.

## Motivation

npm has no place for release notes, and neither does crates.io or PyPI. A
publisher who writes them puts them in a GitHub release, a `CHANGELOG.md`
inside the package, or both, and every reader goes looking:

- `pnpm update --interactive` and `pnpm outdated` list versions with nothing
  about what changed, so the user leaves the terminal to find out whether an
  upgrade is safe.
- Renovate and Dependabot fetch notes from the GitHub or GitLab repository
  named in the package's metadata. They need a token and API quota for that
  host, guess the release tag from conventions that monorepos break, and find
  nothing when the repository is private, moved, hosted elsewhere, or not
  named.
- A `CHANGELOG.md` in the archive can be read only after downloading the
  version it describes, and it cannot be corrected once published.
- A package published to a private pnpr registry often has no public
  repository at all, so its users have no notes unless the publisher runs a
  second publishing pipeline to a release page.

A release is also more than its packages. A workspace that ships an npm CLI,
its Rust core as a crate, and Python bindings as a wheel writes one story
("12.0: a new resolver, …") that today lives on a GitHub release, apart from
every registry the packages came from, and three changelogs that each tell
part of it.

pnpr already receives the release, publishes it across ecosystems in one
transaction, knows who published it, and serves every reader of the
packages, with their access rules. It is the one place where notes can be
published in the same step as the versions and read by every client the same
way, whatever ecosystem the package is in.

## Detailed Explanation

### Model

There are two kinds of notes, and both are UTF-8 Markdown documents with the
time they were last changed and the identity that changed them:

- A **version note** belongs to one package version in one ecosystem. It
  covers all of the version's archives, including its platform variants, and
  a [registry revision](./0000-integrity-addressed-patch-revisions.md) of the
  version does not replace it; a revision states its own reason in its
  history record.
- A **publication note** belongs to one publication: the committed set of
  versions, possibly in several ecosystems, that
  [atomic workspace publication](./0000-atomic-workspace-publication.md)
  records. It tells what the release as a whole changes, and its readers reach
  it from any of the publication's members.

Every version belongs to exactly one publication, a single-package publish
being a publication of one member, so the two levels never compete: a
publisher of one package writes a version note, and a workspace writes a
publication note and, where a package has its own story, version notes too.

Notes are mutable. The publisher can fix a typo, add a migration step found
after release, or remove a note, and readers see the change after the note's
cache lifetime. Nothing that is integrity-addressed refers to a note, so
editing one never invalidates a package document, a release record, or a
lockfile.

### Content rules

pnpr accepts a note when it:

- is valid UTF-8 and at most 64 KiB;
- contains no control characters other than line feed and tab, no carriage
  returns, and no bidirectional override characters, so a note cannot move a
  terminal's cursor, recolor it, or hide text from someone reading it;
- is Markdown in the CommonMark and GitHub Flavored Markdown dialect. pnpr
  does not render it or check its syntax.

Raw HTML inside the Markdown is stored as written. A web interface that
renders notes must sanitize them as it would any untrusted Markdown. pnpm
never renders HTML and never fetches images.

### Version note routes

A version note lives under the version's path in its ecosystem's registry:

```text
GET    <registry-base>/<name>/<version>/notes
PUT    <registry-base>/<name>/<version>/notes
DELETE <registry-base>/<name>/<version>/notes
GET    <registry-base>/<name>/<version>/notes?since=<version>
```

The registry base is the one the package's other routes use, so the same
shape serves every ecosystem:

```text
https://pnpr.example.com/npm/@example/cli/12.0.0/notes
https://pnpr.example.com/cargo/example-core/12.0.0/notes
https://pnpr.example.com/pypi/example-bindings/12.0.0/notes
```

A [named registry](./0000-pnpr-registry-mounts.md) serves the notes of its
own packages under its `~<name>` base. A name is written as the ecosystem
writes it in its own routes: an npm scope keeps its `/`, and a Python name is
in its normalized form. No existing route has a segment after a version. An
unscoped npm `fast-thing/1.4.0/notes` has as many segments as a scoped
version document, `@scope/fast-thing/1.4.0`, and is told apart by its first
segment: only a scope starts with `@`.

Without `since`, `GET` answers `text/markdown; charset=utf-8` with the note,
and `404` when the version has none. The `Last-Modified` header carries the
time of the last change and the `ETag` header identifies the text. In an
ecosystem with dist-tags, a `GET` may name a tag instead of a version, as
npm's version document route allows: `fast-thing/latest/notes?since=1.2.0`
is what changed since the installed version.

With `since`, it answers `application/json` with the notes of every version
`v` of the package with `since < v ≤ <version>`, ordered by the ecosystem's
own version order (semver for npm and Cargo, PEP 440 for Python), newest
first. Each version is listed when it has a note or its publication has one:

```json
{
  "notes": [
    {
      "version": "12.0.0",
      "modified": "2026-10-28T09:12:44Z",
      "text": "- `install` reads lockfiles from 11.x.\n",
      "publication": { "id": "example/9f2c41", "name": "v12.0.0" }
    },
    { "version": "11.4.0", "modified": "2026-09-30T17:01:02Z", "text": "…" }
  ],
  "next": null
}
```

`publication` is present when the version's publication has a note, so a
client fetches each publication note once, however many of the upgraded
packages, in whatever ecosystems, came out of it. `text` is absent when only
the publication has a note.

A range includes prereleases only when `<version>` is one, so an upgrade
between stable versions does not list the betas in between. A response holds
at most 100 notes and 1 MiB; when more match, `next` is the URL of the
following page. Unpublished versions are left out. A deprecated or yanked
version keeps its note.

### Publication note routes

A publication belongs to no ecosystem, so its note is served from the server
root, next to the publication itself:

```text
GET    <pnpr>/-/publications/<org>/<id-or-name>/notes
PUT    <pnpr>/-/publications/<org>/<id-or-name>/notes
DELETE <pnpr>/-/publications/<org>/<id-or-name>/notes
```

For example, `https://pnpr.example.com/-/publications/example/v12.0.0/notes`.
`GET` answers as for a version note. The publication's own document lists its
members, filtered by what the caller may read, so a reader can see which
packages, in which ecosystems, the release changed.

### Writing notes

Notes are written with the publication, or at any time after it.

**With the publication.** The cross-ecosystem publish body carries the
publication's name and note at the top, and each entry's version note beside
the entry's `ecosystem`:

```json
{
  "name": "v12.0.0",
  "notes": "## A new resolver\n\n…",
  "packages": [
    { "ecosystem": "npm", "notes": "- `install` reads lockfiles from 11.x.\n", … },
    { "ecosystem": "cargo", "notes": "…", … },
    { "ecosystem": "pypi", … }
  ]
}
```

pnpr removes `notes` from each entry before handing it to its ecosystem's
publish path, as it removes `ecosystem`, so no ecosystem's own format gains a
field. A direct npm publish of one package is a publication of one entry: its
document may carry a top-level `notes` field, the version note, and npm's
registry ignores it. Staged publishing holds the notes with the stage and
stores them when the stage is approved.

pnpr stores the notes in the same transaction as the versions. A rejected
note rejects the publication, so a release is never visible without the notes
it was published with.

**After the publication.** `PUT` replaces a note with the request body
(`text/markdown`), and `DELETE` removes it. A request for a version or
publication that does not exist is answered `404`; notes are never created
ahead of what they describe.

A version note may be written by an identity that the package's `publish`
rule allows, the rule that published the version. A publication note may be
written by an identity that the `publish` rule of every member allows. A note
adds text and removes nothing the install depends on, so neither needs the
`unpublish` permission that rewriting a version's entry, such as deprecating
it, needs. The CI identity that published a release can therefore write its
notes once a later changelog step has run.

pnpr records each change in its audit log with the identity, the time, and
the digests of the old and new text. It does not keep or serve earlier texts.

### Reading notes

A version note has the access rules of its package: whoever can read the
package can read its notes, and a caller who cannot gets the response that the
package's own routes give it. A publication note is readable by a caller who
can read at least one of its members, as the publication itself is.

Notes follow the cache rules that the
[split packuments RFC](./0000-split-packuments.md) gives to mutable
documents. A note of public packages is `public` with a short lifetime, five
minutes by default, and is revalidated with its `ETag`; a note gated by the
caller's credentials is `private, no-store` with `Vary: Authorization`.

### Upstream and mirrored packages

npm's registry, crates.io, and PyPI have no notes, so an upstream registry
that proxies one serves `404` for a single note and an empty list for a
range. An upstream registry whose origin is another pnpr forwards notes
requests to it and caches the responses under the same rules. pnpr never
fetches notes from GitHub or another release page on a reader's behalf.

### pnpm

**Publishing.** `pnpm publish` sends notes for what it publishes, in every
ecosystem it publishes to. In a workspace that uses
[pnpm's versioning](../../text/0006-monorepo-versioning.md), the notes come
from its change intents (see the next section). Elsewhere:

- the publication note comes from `--release-notes <file>`;
- each package's version note comes from its `CHANGELOG.md`, taking the
  section under the heading whose text is the version, as changesets
  (`## 1.4.0`) and Keep a Changelog (`## [1.4.0] - 2026-10-28`) write it, up
  to the next heading of the same level;
- `--no-release-notes` sends none.

When pnpm publishes one package, `--release-notes` gives that package's
version note instead, since the publication has nothing else in it. pnpm
prints which source it used for each note. It sends notes only to registries
declared with `serverType: pnpr`, and sends the same documents without them
elsewhere.

**Reading.** pnpm shows notes where it offers an upgrade:

- `pnpm update --interactive` shows, for the package under the cursor, the
  notes between the installed version and the chosen one, each publication
  note once above the version notes from it;
- `pnpm outdated --notes` prints them below each outdated package;
- `pnpm view <name>@<version> releaseNotes` prints one version's note and its
  publication's.

pnpm prints notes as plain text after removing any control characters a
registry let through, shows link targets next to their text, and never
treats a note as an instruction: the notes are the publisher's words, not
pnpm's.

### Notes from pnpm's versioning

pnpm's [native monorepo versioning](../../text/0006-monorepo-versioning.md)
already composes each released package's changelog section from the change
intents that `pnpm version -r` consumed and recorded in its ledger. By
default it keeps no committed `CHANGELOG.md`: it packs the section, chained
onto the previous version's `CHANGELOG.md`, into the new tarball. When the
publish target is a pnpr registry, that `registry` storage mode writes notes
instead:

- A version note holds the intents that name only that package, plus the
  intents shared with packages outside the publication. It is the section
  versioning would have packed, without the history below it: a reader gets
  the history from a range.
- The publication note holds the intents that name more than one package of
  the publication, once each, with the packages each applies to, and the
  text of `--release-notes <file>` above them for a summary written by hand.
- The publication is named after the plan: the version of the epic's lead
  package or of the single fixed group that the release contains, or
  `--publication-name <name>`, and it has no name otherwise.
- `pnpm publish -r --batch` sends the whole plan as one publication, so the
  versions, their notes, and the publication note become visible together.

The tarball then holds only what the tagged sources produce, and no previous
tarball is fetched. A later `pnpm version -r` deletes a consumed intent once
the registry holds the notes it went into, which it checks by comparing the
notes' text with the text it composes. A version that another tool published
has no such notes, so it blocks deletion, as a tarball without the section
does today. Registries other than pnpr keep the tarball chain.

A package on a release lane gets a version note for each prerelease, and its
graduation's stable version gets the note composed from every intent the
lane consumed, as versioning composes the stable section today. A range
between stable versions leaves the prereleases out, so it reads that
aggregated note once.

## Rationale and Alternatives

### A changelog inside the archive

pub.dev and the VS Code Marketplace show the `CHANGELOG.md` of the package,
and NuGet shows the `releaseNotes` element of the package's `.nuspec`. That
needs no new route, and the notes are as immutable and as well attested as
the archive.

It also means a note cannot be fixed, and a reader must download the version
it describes to learn whether to install it, which is the wrong way round for
an upgrade prompt. pnpm reads `CHANGELOG.md` at publish time instead, so
publishers who keep one need to do nothing new.

### A field of the package document

A `releaseNotes` field in each version's entry would reach clients without a
new route. Every resolution would then download every version's notes, the
same weight the [split packuments RFC](./0000-split-packuments.md) removed,
and every edit would advance the document's generation and purge it from
caches. Notes are read when a person decides on an upgrade, not when a
resolver picks a version, so they get a route of their own.

### A link to the notes

RubyGems' `changelog_uri` metadata and PyPI's `project_urls` name a page
where the notes are. A link is cheap but does not answer the problem: the
text behind it has no defined format, may need a token for another host, and
is gone when the host moves, which is what Renovate deals with today.

### The release record

The release record at `<name>/<version>+r<revision>` is fetched for the
version a resolver picks. It is immutable and addressed by revision, while a
note is edited and belongs to the version, so a note there would either be
frozen or force a new revision for a typo.

### Notes per ecosystem

Each ecosystem's publish path could take notes in its own way: a field of
npm's publish document, a Cargo publish metadata field, a form field of a
Python upload. That would give three formats for one concept, a note per
ecosystem for a release that spans several, and no place for the release as
a whole. One model, carried beside each entry rather than inside its
ecosystem's format, keeps the notes of a release in one place and every
ecosystem's protocol unchanged.

## Implementation

1. pnpr: notes keyed by ecosystem, package, and version, and by
   publication, in the SQL metadata storage, written in the publication's
   transaction, with the content rules, the routes above, and audit records.
   Publication notes need the publications of
   [atomic workspace publication](./0000-atomic-workspace-publication.md);
   version notes do not, and can come first.
2. pnpr: accept `notes` in the cross-ecosystem publish body, in a direct npm
   publish, and in staged publishing.
3. pnpr: forward notes routes of an upstream registry whose origin is a pnpr.
4. pnpm: `--release-notes` and `--no-release-notes` for `pnpm publish`, and
   reading the version's section of each package's `CHANGELOG.md`, in every
   ecosystem pnpm publishes.
5. pnpm: in versioning's `registry` storage mode, compose version and
   publication notes from the ledgered intents for a pnpr registry, name the
   publication, and check the notes before deleting intents.
6. pnpm: notes in `pnpm update --interactive`, `pnpm outdated --notes`, and
   `pnpm view`.
7. Documentation of the routes for other clients, such as Renovate.

Tests must cover: a publish whose note is rejected (nothing published), a
staged publish whose notes appear only on approval, a publication of npm,
Cargo, and Python packages whose note each member's range lists, a range
across prereleases with a stable and a prerelease target in npm and Python
version orders, a dist-tag target, pagination, a gated package read without
credentials, a publication note read by a caller who can read only some
members, an edit by an identity outside the
`publish` rule (rejected), control and bidirectional override characters
(rejected by pnpr, removed by pnpm), and changelog sections in the changesets
and Keep a Changelog forms.

## Prior Art

- **GitHub and GitLab releases** attach editable Markdown notes to a tag.
  They are the de facto home of npm release notes today, and Renovate and
  Dependabot read them from there.
- **NuGet** stores `releaseNotes` in the package manifest and shows it on the
  package page. It cannot be changed without a new version.
- **pub.dev** and the **VS Code Marketplace** render the package's
  `CHANGELOG.md`.
- **RubyGems** (`changelog_uri`) and **PyPI** (`project_urls`) link to notes
  hosted elsewhere.
- **npm** and **crates.io** have no release notes; their package pages show
  the README.

## Unresolved Questions and Bikeshedding

- Container images have tags rather than ordered versions. Should an image
  in a publication get version notes, and how would a range be defined?
- Outside pnpm's versioning, should `pnpm publish` read `CHANGELOG.md` by
  default, or only when asked? A default helps every publisher who keeps
  one, but publishes text the publisher may not have meant as release notes.
- Should the range route also accept a semver range, for a reader that wants
  the notes of `^1` without knowing the versions?
- Should pnpr import notes from a package's GitHub releases when it mirrors a
  package from npm, so that public packages have notes too? That fetches
  from a third host on pnpr's side, with its own rate limits and failure
  modes.
- Should a package document entry say whether a version has a note, so a
  client can skip the request? That puts a mutable flag back into the
  document.
- Should earlier texts of an edited note be served, as a history, rather than
  kept only as digests in the audit log?
