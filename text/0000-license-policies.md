# License policies

> **Status: proposed.** A prototype for pnpm v12 exists on a local branch and is not yet a pull request. An earlier TypeScript implementation was proposed in [pnpm/pnpm#11159](https://github.com/pnpm/pnpm/pull/11159).

## Summary

Add workspace settings that declare which licenses the dependencies may use. Add `pnpm licenses check` to check the installed dependencies against them, and `pnpm licenses allow` and `pnpm licenses disallow` to edit them. `pnpm add` and `pnpm update` run the check after they install. The check covers npm packages, Cargo crates, and Python distributions that pnpm installs.

## Motivation

`pnpm licenses list` reports licenses but enforces nothing. Teams that must keep copyleft or unlicensed code out of a product run a separate tool in CI. That tool reads `node_modules` only, so it misses the crates and Python distributions pnpm now installs. It also runs after the dependency is already committed, not when the developer adds it.

## Detailed Explanation

### Settings

```yaml title="pnpm-workspace.yaml"
allowLicenses:
  MIT: true
  Apache-2.0: true
  GPL-3.0-only: false
strictLicenses: true
licenseExceptions:
  some-package: [WTFPL]
  pkg:cargo/some-crate: [Zlib]
packageLicenses:
  missing-license@1.0.0: MIT
```

| Setting | Type | Default | Meaning |
| --- | --- | --- | --- |
| `allowLicenses` | `Record<string, boolean>` | `{}` | `true` allows a license. `false` always fails the check. |
| `strictLicenses` | `boolean` | `false` | A license that is not allowed, or an unknown license, fails the check instead of warning. |
| `licenseExceptions` | `Record<selector, string[]>` | `{}` | Extra licenses one package may use. Wins over `false`. |
| `packageLicenses` | `Record<selector, string>` | `{}` | The license to check in place of the one the package declares. |

`allowLicenses` follows the shape of `allowBuilds`: one map, a boolean per key, rather than separate allow and deny lists.

The keys of `allowLicenses` are single SPDX identifiers. A compound expression such as `MIT OR ISC` is rejected with `ERR_PNPM_LICENSES_COMPOUND_EXPRESSION`, as is one license listed in two letter cases.

The settings are read from `pnpm-workspace.yaml` and the environment only.

### Selectors

A selector is a package name, or a name with exact versions (`foo@1.0.0 || 2.0.0`), the same syntax `allowBuilds` takes. A crate is selected as `pkg:cargo/<name>` and a Python distribution as `pkg:pypi/<name>`, with the name normalized as PEP 503 specifies. An npm package and a crate with the same name are different packages.

### Matching

A package's license is parsed as an SPDX expression.

- `OR` passes when any alternative passes. `MIT OR GPL-3.0-only` passes when `MIT` is allowed, even if `GPL-3.0-only` is disallowed.
- `AND` passes when every part passes.
- A license that is not an SPDX expression is compared as text, ignoring case.
- A package with no license is `Unknown`. It warns when at least one license is allowed, and fails with `strictLicenses`.

### Where licenses come from

| Ecosystem | Packages | License |
| --- | --- | --- |
| npm | the lockfile's packages | `license` in the installed `package.json`, else the license file |
| Cargo | `Cargo.lock`, with `cargo.enabled` | `package.license` in the crate's `Cargo.toml`, else the license file |
| Python | `pylock.toml`, with `python.enabled` | `License-Expression` in the installed `METADATA`, else a one-line `License` |

### Commands

- `pnpm licenses check [--json] [-r]` prints the violations and warnings and exits with 1 on a violation.
- `pnpm licenses allow <license>...` and `pnpm licenses disallow <license>...` set keys of `allowLicenses` to `true` or `false`. They keep the spelling of a key that already exists.
- `pnpm add` and `pnpm update` run the check after they install and fail with `ERR_PNPM_LICENSE_VIOLATION`. A `--lockfile-only` run skips it, since nothing is installed to read.
- `pnpm install` does not run the check. A frozen install in CI should not start failing because the policy changed. CI runs `pnpm licenses check`.

## Rationale and Alternatives

- **Separate `allowedLicenses` and `disallowedLicenses` lists.** This is what pnpm/pnpm#11159 proposed. Two lists can name the same license, and they don't match how other pnpm settings are shaped.
- **Read Cargo's or Python's own license tooling, such as `cargo-deny`.** The user would configure the same policy twice. pnpm installs these dependencies, so it can check them under one policy.
- **Check on every `pnpm install`.** This is rejected for the CI reason above. It could be added later behind a setting.
- **Resolve licenses from registry metadata before install.** This would let `--lockfile-only` check too, but crates.io and PyPI metadata don't always match the installed package.

## Prior Art

- [`license-checker`](https://www.npmjs.com/package/license-checker) and its `--onlyAllow` and `--failOn` flags.
- [`cargo-deny`](https://github.com/EmbarkStudios/cargo-deny) `licenses` check, with allow lists, exceptions, and clarifications.
- [`pip-licenses`](https://pypi.org/project/pip-licenses/) `--allow-only` and `--fail-on`.

## Unresolved Questions and Bikeshedding

- Should the settings also be read from the global config, so `pnpm add -g` is checked?
- Should Python fall back to the `License ::` trove classifiers when `METADATA` has no `License-Expression`?
- Should `strictLicenses` default to `true` once `allowLicenses` names a license?
- Should `pnpm install` gain an opt-in check?
