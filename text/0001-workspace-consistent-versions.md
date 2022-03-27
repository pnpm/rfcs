## Summary

Instead of every workspace package declaring a version range on a dependency, a new `workspace-consistent` version specifier protocol would allow packages to delegate to a range defined at the root `package.json`.

## Motivation

A common workflow in monorepos is the need to synchronize on the same version of a dependency.

For example, the `foo` and `bar` packages of a monorepo may declare the same version of `react` in their `package.json` files.

```json5
// packages/foo/package.json
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "^17.0.2"
  }
}
```

```json5
// packages/bar/package.json
{
  "name": "@monorepo/bar",
  "dependencies": {
    "react": "^17.0.2"
  }
}
```

Multiple versions of the same dependency can cause different flavors of problems.

- In projects that bundle dependencies, multiple versions inflate the size of the final result deployed to users.
- Differing versions result in multiple copies that may not interact well at runtime, especially if features like [`Symbol()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) are used. For example, [React hooks will error if a component is rendered with a different copy of React in the same app](https://reactjs.org/warnings/invalid-hook-call-warning.html#duplicate-react).
- For TypeScript, multiple copies of the same `@types` package causes compile errors from mismatching definitions. The compiler diagnostic for this is usually: *"argument of type `Foo` is not assignable to parameter of type `Foo`"*. For developers that have seen this before, they may realize this diagnostic is due to a dependency version mismatch. For developers newer to TypeScript, *"`Foo` is not assignable to `Foo`"* is very confusing.

While there are situations making differing versions become unavoidable, this is usually accidental. Multiple differing versions arise from not reviewing `pnpm-lock.yaml` file changes or not searching for existing dependency specifiers before adding a new one. The later is typically unwritten convention in most monorepos.

## Detailed Explanation

### Minimal Example

<details open>
<summary>A sample root `package.json` configuration would appear as such:</summary>

```json5
// package.json
{
  "name": "example-monorepo",
  "private": true,
  "pnpm": {
    "workspaceConsistent": {
      "versions": {
        "jest": "^27.5.1",
        "react": "^17.0.2",
        "react-dom": "^17.0.2"
      }
    }
  }
}
```
</details>

<details open>
<summary>Workspace packages can then refer to the configured root defined version ranges:</summary>

```json5
// packages/foo/package.json
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "workspace-consistent",
  }
}
```

```json5
// packages/bar/package.json
{
  "name": "@monorepo/bar",
  "dependencies": {
    "react": "workspace-consistent",
    "react-dom": "workspace-consistent"
  },
  "devDependencies": {
    "jest": "workspace-consistent"
  }
}
```
</details>

<details open>
<summary>The resulting `pnpm-lock.yaml` file would persist as:</summary>

```yaml
lockfileVersion: X.X # TBD

workspaceConsistent:
  # Spaces after each block are intentional to help git auto merge version changes.
  jest:
    specifier: ^27.5.1
    dependency: 27.5.1

  react:
    specifier: ^17.0.2
    dependency: 17.0.2

  react-dom:
    specifier: ^17.0.2
    dependency: 17.0.2

importers:

  .:
    specifiers: {}

  packages/foo:
    specifiers:
      react: workspace-consistent
    dependencies:
      react: workspace-consistent

  packages/foo:
    specifiers:
      jest: workspace-consistent
      react: workspace-consistent
      react-dom: workspace-consistent
    dependencies:
      jest: workspace-consistent
      react: workspace-consistent
      react-dom: workspace-consistent_react@workspace-consistent

# No proposed changes to the packages section.
packages:

  /jest/27.5.1:
    # Omitted for brevity

  /react/17.0.2:
    # Omitted for brevity

  /react-dom/17.0.2_react@17.0.2:
    # Omitted for brevity

  # Additional transitive dependencies omitted for brevity
  
```
</details>

### Peers Suffix Elaboration

It's worth elaborating on the lockfile resolved version for `react-dom`, which was recorded as: `workspace-consistent_react@workspace-consistent` for the [_minimal example_](#minimal-example) above.

The `react@workspace-consistent` portion (after the underscore) is referred to as the *"peers suffix"* in pnpm nomenclature. A "_peers suffix_" appears in this case since `react-dom` declares a peer dependency on `react`.

Without workspace consistent versions, this would normally render as `17.0.2_react@17.0.2`. The `17.0.2` portions are intentionally replaced with `workspace-consistent` to keep changes to the lockfile small. The extra indirection allows edits to be limited to the `workspaceConsistent` and `packages` section whenever the workspace changes its `react` or `react-dom` version. See [_Merge conflicts in `pnpm-lock.yaml` are reduced_](#1-merge-conflicts-in-pnpm-lockyaml-are-reduced) for the problem this solves.

## Rationale for First-Class Support

While there's existing tooling in the frontend ecosystem for consistent versions ([`syncpack`](https://github.com/JamieMason/syncpack) being one great option), builtin support from pnpm allows several improvements not otherwise possible.

### 1. Merge conflicts in `pnpm-lock.yaml` are reduced

When upgrading a dependency intended to be consistent across workspace packages, any blocks under the `importers` key containing that dependency will have line changes in `pnpm-lock.yaml`.

Suppose a commit upgrades `react` and `react-dom` to `^18.0.0-rc.3` in the [_minimal example_](#minimal-example) without `workspace-consistent` usage. This results in git merge conflicts if another commit:

- <details close>
  <summary>Changes the version of a dependency line-adjacent to the upgraded dependency.</summary>

  Suppose the current `HEAD` commit upgrades `jest`. On `git merge`, the following conflict manifests.

  ```yaml
  # pnpm-lock.yaml
  packages:

    packages/bar:
      specifiers:
  <<<<<<< HEAD
        jest: ^28.0.0-alpha.7
        react: ^17.0.2
        react-dom: ^17.0.2
  =======
        jest: ^27.5.1
        react: ^18.0.0-rc.3
        react-dom: ^18.0.0-rc.3
  >>>>>>> upgrade-react
  ```
  </details>

- <details>
  <summary>Add or removes a dependency line-adjacent to the upgraded dependency.</summary>

  Suppose the current `HEAD` commit deletes `jest` from `bar`. On `git merge`, the following conflict manifests.

  ```yaml
  # pnpm-lock.yaml
  packages:

    packages/bar:
      specifiers:
  <<<<<<< HEAD
        react: ^17.0.2
        react-dom: ^17.0.2
      dependencies:
        react: 17.0.2
        react-dom: 17.0.2_react@17.0.2
  =======
        jest: ^27.5.1
        react: ^18.0.0-rc.3
        react-dom: ^18.0.0-rc.3
      dependencies:
        react: 18.0.0-rc.3-next-de516ca5a-20220321
        react-dom: 18.0.0-rc.3-next-de516ca5a-20220321_1d81fe1bff57e86f30d2d405110ea7f4
      devDependencies:
        jest: 27.5.1
  >>>>>>> upgrade-react
  ```

  A similar merge conflict arises if `redux` was added since it would be declared below `react`.

  ```yaml
  # pnpm-lock.yaml
  packages:

    packages/foo:
      specifiers:
  <<<<<<< HEAD
        react: ^17.0.2
        redux: ^4.1.2
      dependencies:
        react: 17.0.2
        redux: 4.1.2
  =======
        react: ^18.0.0-rc.3
      dependencies:
        react: 18.0.0-rc.3-next-de516ca5a-20220321
  >>>>>>> upgrade-react
  ```
  </details>

- <details>
  <summary>Adds a new declaration of the upgraded dependency.</summary>

  Suppose the current `HEAD` commit adds a `react-dom` declaration to `foo`:

  ```yaml
  # pnpm-lock.yaml
  packages:

    packages/foo:
      specifiers:
  <<<<<<< HEAD
        react: ^17.0.2
        react-dom: ^17.0.2
      dependencies:
        react: 17.0.2
        react-dom: 17.0.2_react@17.0.2
  =======
        react: ^18.0.0-rc.3
      dependencies:
        react: 18.0.0-rc.3-next-de516ca5a-20220321
  >>>>>>> upgrade-react
  ```
  </details>

As monorepos grow in workspace package count, merge conflicts become increasingly more probable. On GitHub, any merge conflict in `pnpm-lock.yaml` blocks pull requests due to lack of custom merge driver support.

With first-class support for workspace consistent versions, edits on upgrades are limited to the root `package.json` and the `workspaceConsistent` portion of `pnpm-lock.yaml`. This reduces churn in `pnpm-lock.yaml`, and **prevents merge conflicts in all the cases listed above**.

Previous discussion: https://github.com/pnpm/pnpm/discussions/4324#discussioncomment-2166292

### 2. Consistent versions can be declared directly in `package.json` files.

Existing tooling rely on a synchronization step that edits all `package.json` files on a dependency upgrade. For example, developers may find + replace `"react": "^17.0.1"` to `"react": "^17.0.2"` across a repository.

- These edits lead to churn in `package.json` files and merge conflicts with other commits editing `package.json` files.
- Repository maintainers have to remember to run this synchronization step periodically, or create a Continuous Integration step to verify the repo is in a good state.

### 3. Intention becomes clear

There might be a tight relationship between `foo` and `bar`.

```json5
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "^17.0.2"
  }
}
```

```json5
{
  "name": "@monorepo/bar",
  "dependencies": {
    "react": "^17.0.2"
  }
}
```

A developer working primarily in `@monorepo/bar` may not realize the implied coupling and upgrade `@monorepo/bar` to `react@18` without realizing an edit to `@monorepo/foo` was also required.

The `workspace-consistent` protocol makes it more clear from just reading `package.json` when a dependency is intended to be consistent across the monorepo. Ideally this person would search "workspace-consistent package.json` online and find pnpm.io docs.

## Implementation

The primary implementation changes happen in `pnpm-lock.yaml` read and write.

On read, any `workspace-consistent` references will be replaced with the aliased version when deserializing the lockfile into memory. On write, the in-memory versions would be replaced with `workspace-consistent`.

This process may not be possible if the lockfile in a broken state.

  - If the `workspace-consistent` is specified for a dependency in the lockfile not present in the root `package.json`, installation will abort.
  - If the `workspace-consistent` resolution is specified for a dependency not present in the `workspaceConsistent` lockfile block, a best case resolution will be made following existing semver range to concrete version logic.

### Other Changes

Similar to the [`workspace:` protocol](https://pnpm.io/workspaces#workspace-protocol-workspace), `pnpm publish` will need to dynamically replace instances of `workspace-consistent` with the value specified in the root `package.json`.

There may be changes to `pnpm update` to make sure it respects `workspaceConistent` versions, but the existing version deduplication logic may do that already.

## Elaboration

### Semver Range Specifier in Workspaces

Readers familiar with the [`workspace:` protocol](https://pnpm.io/workspaces#workspace-protocol-workspace) may be curious as to why the `workspace-consistent:` protocol does not allow a semver range specifier for publishing like the `workspace:` protocol does. This is because the behavior on publish may be confusing.

Walking through an example of the confusing behavior:

```json5
// package.json
{
  "pnpm": {
    "workspaceConsistent": {
      "versions": {
        "react": "^17.0.0",
      }
    }
  }
}
```

Assume the `react` specifier `^17.0.0` resolved to `17.0.2`.

```yaml
# pnpm-lock.yaml
workspaceConsistent:
  react:
    specifier: ^17.0.0
    dependency: 17.0.2
```

Suppose semver ranges were indeed allowed on `workspace-consistent:`. If a package were to specify `workspace-consistent:^`:

```json5
// packages/foo/package.json
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "workspace-consistent:^",
  },
}
```

On publish this is expected to become:

```json5
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "^17.0.0",
  },
}
```

It would be surprising if this became `^17.0.2` (created by adding `^` to the resolved version) since that may be overly specific and leave authors with no way to relax that.

On the other hand, suppose a package were to specify `workspace-consistent:*`:

```json5
// packages/foo/package.json
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "workspace-consistent:*",
  },
}
```

On publish, the least suprising behavior would be for this to become `17.0.2`. This is because `17.0.2` was the resolved version actually tested on in the monorepo.

```json5
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "17.0.2",
  },
}
```

This leads to a discrepancy between what `workspace-consistent:*` and `workspace-consistent:^` should append semver range types to on publish. (The original specifier vs the resolved version.)

The semantics may become more clear if `workspaceConsistent` configured versions must be a concrete version, but this eliminates the ability to `pnpm update` `workspaceConsistent` values.

### Lockfile `workspaceConsistent` block format

The proposed format is:

```yaml
workspaceConsistent:
  jest:
    specifier: ^27.5.1
    dependency: 27.5.1

  react:
    specifier: ^17.0.2
    dependency: 17.0.2

  react-dom:
    specifier: ^17.0.2
    dependency: 17.0.2
```

An alternative format that represents the same information would be:

```yaml
workspaceConsistent:
  specifiers:
    jest: ^27.5.1
    react: ^17.0.2
    react-dom: ^17.0.2
  dependencies:
    jest: 27.5.1
    react: 17.0.2
    react-dom: 17.0.2
```

The alternative format better matches the existing shape of `importers`. However the proposed format is more resilient to merge conflicts. Separate commits changing the version of `react` and `jest` would merge conflict with the alternative format, while the proposed format would merge those changes cleanly.

## Extensions

### Transitive Consistency

Multiple versions of a package can still end up in the final product due to versions declared as a dependency of a dependency.

It would be useful to declare two extra fields to solve the duplicate dependency problem completely.

```json5
{
  "name": "example-monorepo",
  "private": true,
  "pnpm": {
    "workspaceConsistent": {
      "groups": {
        "default": {
          "@types/react": "^17.0.43",
          "react": "^17.0.2",
          "react-dom": "^17.0.2",
        }
      },
      "enforceConsistencyTransitively": true,
      "allowTransitiveMismatches": ["@types/react"]
    }
  }
}
```

`enforceConsistencyTransitively` will error if there is ever more than one item in the `packages` block of `pnpm-lock.yaml` for a dependency, even duplicates on the same version but appearing multiple times due to peer dependencies.

The `allowTransitiveMismatches` option will provide an escape hatch in case specific dependencies are acceptable. In the case above, duplicate `@types/react` dependencies may be acceptable since it's usually under `devDependencies` and package authors may declare a semver range without any overlap on the monorepo workspace consistent range.

### Allowed Deviations

The default behavior is to show an error if a workspace package does not use `workspace-consistent` for a configured dependency. However, sometimes a workspace package may not be able to use the same version of a dependency as the other packages in the monorepo. The `allowedDeviations` option can be used as an escape hatch.

Once specified, `@monorepo/baz` will be allowed to declare a version range that does not begin with `workspace-consistent` for `@types/react`.

```json5
{
  "name": "some-monorepo",
  "pnpm": {
    "workspaceConsistent": {
      "groups": {
        "default": {
          "@types/react": "^17.0.43",
          "react": "^17.0.2",
          "react-dom": "^17.0.2",
        }
      },
      "allowedDeviations": {
        "@types/react": ["@monorepo/baz"]
      },
    }
  }
}
```

### Version Groups

Users may want to declare different subset partitions of consistent version sharing. Borrowing the phrase [_version groups_ from syncpack](https://github.com/JamieMason/syncpack#versiongroups), this could be defined as such:

```json5
{
  "name": "example-monorepo",
  "private": true,
  "pnpm": {
    "workspaceConsistent": {
      "groups": {
        "default": {
          "react": "^17.0.2",
          "react-dom": "^17.0.2",
          "redux": "^4.1.2"
        },
        "react-rc": {
          "react": "^18.0.0-rc.3",
          "react-dom": "18.0.0-rc.3"
        },
        "react-next": {
          "react": "18.0.0-rc.3-next-033fe52b4-20220325",
          "react-dom": "18.0.0-rc.3-next-033fe52b4-20220325"
        },
      }
    }
  }
}
```

Workspace packages would then use `workspace-consistent:<version-group>` as the protocol. Simply `workspace-consistent` would refer to the `default` group.

An edge case arises if a package overlaps its version group usage.

```json5
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "workspace-consistent:react-rc",
    "react-dom": "workspace-consistent:react-next"
  }
}
```

These scenarios are likely mistakes. While installation can continue without problems, an error should be emitted to be helpful. To produce the error a check algorithm would record usages of a non-default version group and the dependencies declared inside of it. If another version group is selected for a dependency present in a previously declared group, installation will fail.

Note that the algorithm above would permit this alternative scenario, which is likely not a problem.

```json
{
  "name": "@monorepo/foo",
  "dependencies": {
    "react": "workspace-consistent:react-rc",
    "react-dom": "workspace-consistent:react-rc",
    "redux": "workspace-consistent"
  }
}
```

## Alternatives

### Comparison to overrides/resolutions

An alternative mechanism for declaring workspace consistent versions is the [`pnpm.overrides` feature](https://pnpm.io/package_json#pnpmoverrides). While mechanically this allows you to set the version of a dependency across all workspace packages, it can be a bit unexpected when if `pnpm.overrides` rewrites a dependency's dependency to an incompatible version silently.

`pnpm.overrides` is ultimately intended for a different purpose. The NPM RFC for a similar feature explicitly states that it should be used as a short-term hack to fix vendor problems.

> Using this feature should be considered a hack in most cases, something that is done temporarily while waiting for a bug to be fixed, or to avoid excessive duplication caused by an overly strict meta-dependency specifier.

https://github.com/npm/rfcs/blob/main/accepted/0036-overrides.md

The `workspace-consistent` protocol is conversely intended for long-lived usage.
