# Release notes: what changed in a version, served by the registry

## Summary

pnpr stores release notes for each version of a hosted package: Markdown text
the publisher sends with the publish and may edit later. Notes are served at
`<registry-base>/<name>/<version>/notes`, one version at a time or all the
versions between two, so a client upgrading from `1.2.0` to `1.4.0` reads
what changed in one request. They are not part of the package document, the release record, or
the archive, so an install never downloads them and editing them changes
nothing a lockfile pins. pnpm sends notes with `pnpm publish` and shows them
where it offers an upgrade.

## Motivation

npm has no place for release notes. A publisher who writes them puts them in
a GitHub release, a `CHANGELOG.md` inside the package, or both, and every
reader goes looking:

- `pnpm update --interactive` and `pnpm outdated` list versions with nothing
  about what changed, so the user leaves the terminal to find out whether an
  upgrade is safe.
- Renovate and Dependabot fetch notes from the GitHub or GitLab repository
  named in the package's `repository` field. They need a token and API quota
  for that host, guess the release tag from conventions that monorepos break,
  and find nothing when the repository is private, moved, hosted elsewhere, or
  not named.
- A `CHANGELOG.md` in the archive can be read only after downloading the
  version it describes, and it cannot be corrected once published.
- A package published to a private pnpr registry often has no public
  repository at all, so its users have no notes unless the publisher runs a
  second publishing pipeline to a release page.

pnpr already receives the release, knows who published it, and serves every
reader of the package, with that package's access rules. It is the one place
where notes can be published in the same step as the version and read by
every client the same way.

## Detailed Explanation

### Model

A version of a hosted package has at most one release note: a UTF-8 Markdown
document, with the time it was last changed and the identity that changed it.
A note belongs to the version, not to an archive: it covers the version's
platform variants, and a
[registry revision](./0000-integrity-addressed-patch-revisions.md) of the
version does not replace it. A revision states its own reason in its history
record.

A note is mutable. The publisher can fix a typo, add a migration step found
after release, or remove the note, and readers see the change after the
note's cache lifetime. Nothing that is integrity-addressed refers to a note,
so editing one never invalidates a package document, a release record, or a
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

### Routes

A note lives under its version's own path, next to the version document
npm clients already read at `<registry-base>/<name>/<version>`:

```text
GET    <registry-base>/<name>/<version>/notes
PUT    <registry-base>/<name>/<version>/notes
DELETE <registry-base>/<name>/<version>/notes
GET    <registry-base>/<name>/<version>/notes?since=<version>
```

For example, `https://registry.example.com/fast-thing/1.4.0/notes` or
`https://registry.example.com/@scope/fast-thing/1.4.0/notes`. The paths are
relative to the registry base, so each
[named registry](./0000-pnpr-registry-mounts.md) serves the notes of its own
packages. No existing route has a segment after a version. An unscoped
`fast-thing/1.4.0/notes` has as many segments as a scoped version document,
`@scope/fast-thing/1.4.0`, and is told apart by its first segment: only a
scope starts with `@`.
A `GET` may name a dist-tag instead of a version, as the version document
route allows: `fast-thing/latest/notes?since=1.2.0` is what changed since the
installed version.

Without `since`, `GET` answers `text/markdown; charset=utf-8` with the note,
and `404` when the version has none. The `Last-Modified` header carries the
time of the last change and the `ETag` header identifies the text.

With `since`, it answers `application/json` with the notes of every version
`v` of the package with `since < v ≤ <version>` that has one, newest first:

```json
{
  "notes": [
    { "version": "1.4.0", "modified": "2026-10-28T09:12:44Z", "text": "## Breaking\n\n…" },
    { "version": "1.3.0", "modified": "2026-09-30T17:01:02Z", "text": "…" }
  ],
  "next": null
}
```

A range includes prereleases only when `<version>` is one, so an upgrade
between stable versions does not list the betas in between. A response holds
at most 100 notes and 1 MiB; when more match, `next` is the URL of the
following page.

Versions that are unpublished are left out. A deprecated version keeps its
note.

### Writing notes

A note is written with the version, or at any time after it.

**With the version.** An npm publish document may carry a top-level
`releaseNotes` field, keyed by version:

```json
{
  "name": "fast-thing",
  "versions": { "1.4.0": { … } },
  "releaseNotes": { "1.4.0": "## Breaking\n\n- Drops Node.js 18.\n" },
  "_attachments": { … }
}
```

pnpr stores the note in the same transaction as the version. A rejected note
rejects the publish, so a release is never visible without the notes it was
published with. Batch publishing carries the field in each npm entry, and
staged publishing holds it with the stage and stores it when the stage is
approved. npm's registry ignores unknown top-level fields of a publish
document, so a document with notes still publishes there, without them.

**After the version.** `PUT` replaces a version's note with the request body
(`text/markdown`), and `DELETE` removes it. A request for a version that does
not exist is answered `404`; notes are never created ahead of their version.

Both are allowed to an identity that the package's `publish` rule allows,
the rule that published the version. A note adds text and removes nothing
the install depends on, so it does not also need the `unpublish` permission
that rewriting a version's entry, such as deprecating it, needs. The CI
identity that published the version can therefore write its notes once a
later changelog step has run.

pnpr records each change in its audit log with the identity, the time, and
the digests of the old and new text. It does not keep or serve earlier texts.

### Reading notes

A note has the access rules of its package: whoever can read the package can
read its notes, and a caller who cannot gets the response that the package's
own routes give it.

Notes follow the cache rules that the
[split packuments RFC](./0000-split-packuments.md) gives to mutable
documents. A note of a public package is `public` with a short lifetime, five
minutes by default, and is revalidated with its `ETag`; a note of a package
gated by the caller's credentials is `private, no-store` with
`Vary: Authorization`.

### Upstream and mirrored packages

npm's registry has no notes, so an upstream registry that proxies it serves
`404` for a single note and an empty list for a range. An
upstream registry whose origin is another pnpr forwards notes requests to it
and caches the responses under the same rules. pnpr never fetches notes from
GitHub or another release page on a reader's behalf.

### pnpm

**Publishing.** `pnpm publish` sends notes for the version it publishes:

- from `--release-notes <file>` when given;
- otherwise from the package's `CHANGELOG.md`, taking the section under the
  heading whose text is the version, as changesets (`## 1.4.0`) and
  Keep a Changelog (`## [1.4.0] - 2026-10-28`) write it, up to the next
  heading of the same level;
- not at all with `--no-release-notes`, or when neither source has a note for
  the version.

pnpm prints which source it used. It sends notes only to registries declared
with `serverType: pnpr`, and sends the same document without them elsewhere.

**Reading.** pnpm shows notes where it offers an upgrade:

- `pnpm update --interactive` shows the notes between the installed version
  and the chosen one for the package under the cursor, fetched when the
  package is selected;
- `pnpm outdated --notes` prints them below each outdated package;
- `pnpm view <name>@<version> releaseNotes` prints one version's note.

pnpm prints notes as plain text after removing any control characters a
registry let through, shows link targets next to their text, and never
treats a note as an instruction: the notes are the publisher's words, not
pnpm's.

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

## Implementation

1. pnpr: a notes table keyed by package and version in the SQL metadata
   storage, written in the publish transaction, with the content rules, the
   routes above, and audit records.
2. pnpr: accept `releaseNotes` in direct, batch, and staged publishing.
3. pnpr: forward notes routes of an upstream registry whose origin is a pnpr.
4. pnpm: `--release-notes` and `--no-release-notes` for `pnpm publish`, and
   reading the version's section of `CHANGELOG.md`.
5. pnpm: notes in `pnpm update --interactive`, `pnpm outdated --notes`, and
   `pnpm view`.
6. Documentation of the routes for other clients, such as Renovate.

Tests must cover: a publish whose note is rejected (nothing published), a
staged publish whose note appears only on approval, a range across
prereleases with a stable and a prerelease target, a dist-tag target,
pagination, a gated
package read without credentials, an edit by an identity outside the
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

- The name of the `releaseNotes` publish field.
- Should `pnpm publish` read `CHANGELOG.md` by default, or only when asked? A
  default helps every publisher who keeps one, but publishes text the
  publisher may not have meant as release notes.
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
