# Templates: Reusable packages to share dependencies and configuration

## Summary

"_Templates_" are packages other package manifests (i.e. `package.json` files) may reference to populate sections of their own definition. This enables data normalization (or deduplication) of local `package.json` files for greater consistency, improved editing ergonomics, and reduced merge conflicts.

## Motivation

Large monorepos often contain many `package.json` files that repeat information. Across these different package manifests, the same metadata fields (e.g. `author`, `repository`, `license`) or dependency versions (e.g. `react`, `jest`) may be declared. When a field or dependency declaration for one package in a monorepo deviates from the others, this can be unintentional and adversely affect the behavior of the project.

### Dependencies

For dependencies specifically, inconsistent versions of the same dependency within a monorepo cause different flavors of problems:

- In projects that bundle dependencies, multiple versions inflate the size of the final result deployed to users.
- Differing versions result in multiple copies that may not interact well at runtime, especially if features like [`Symbol()`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol) are used. For example, [React hooks will error if a component is rendered with a different copy of React in the same app](https://reactjs.org/warnings/invalid-hook-call-warning.html#duplicate-react).
- For TypeScript, multiple copies of the same `@types` package causes compile errors from mismatching definitions. The compiler diagnostic for this is usually: *"argument of type `Foo` is not assignable to parameter of type `Foo`"*. For developers that have seen this before, they may realize this diagnostic is due to a dependency version mismatch. For developers new to TypeScript, *"`Foo` is not assignable to `Foo`"* is very confusing.

While there are situations differing versions are intentional, this is more often accidental. Multiple differing versions arise from not reviewing `pnpm-lock.yaml` file changes or not searching for existing dependency specifiers before adding a new one. The later is typically unwritten convention in most monorepos.

### Authoring Fields

Fields such as `author`, `license`, and `repository` are typically expected to be the same across all packages in a monorepo. It would be easier to set this in a singular source of truth rather than every package.

### Workspace Configuration

Workspace settings such as [`pnpm.packageExtensions`](https://pnpm.io/package_json#pnpmpackageextensions) can be shared across different repositories. For example, pnpm includes the [`@yarnpkg/extensions` database](https://github.com/yarnpkg/berry/blob/master/packages/yarnpkg-extensions/sources/index.ts) builtin. Templates allow users to create their own extensions database.

## Detailed Explanation

A "*Template*" is a normal `package.json` file with the `pnpm.template` field set.

```json5
{
  "name": "@example/frontend-catalog",
  "pnpm": {
    "template": "catalog"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "redux": "^4.2.0",
    "react-redux": "^8.0.0"
  }
}
```

A package can then reference the template to populate different portions of its definition. Templates are available in different flavors that provide distinct features.

- `catalog` — templates that share dependency specifiers
- `toolkit` — templates that share `devDependencies`
- `authoring` — templates that share `author`, `license`, `funding`, `repository`, `bugs`, `contributors`
- `scripts` — templates that share scripts
- `compatibility` — templates that can modify the package manifest of other dependencies through `pnpm.packageExtensions`. (Alternative to the compatibility DB.)
- `patches` — templates that can modify the package manifest and contents of dependencies through fields such as `pnpm.packageExtensions`, `pnpm.overrides`, `pnpm.patchedDependencies`, `pnpm.peerDependencyRules`, etc.

Templates must opt into being a template by defining `pnpm.template` and the flavor. This is a safety mechanism to:

1. Intentionally prevent existing packages from being referred to for this purpose. Templates have different obligations and usage than standard packages that authors of templates should be aware of.
2. Allow checks to be performed on the template before publishing and during consumption. Not all `package.json` fields may be valid on a template.

### Authoring

The below shows an example of a simple `authoring` template.

```json5
{
  "name": "@example/authoring-template",
  "version": "0.1.0",
  "pnpm": {
    "template": "authoring"
  },
  "author": "Example Organization <team@organization.example>",
  "license": "MIT",
  "repository": "git@organization.example/example/project"
}
```

A package referencing the `@example/authoring-template` above will have the following on-disk and in-memory representations.

**On-Disk**

```json5
{
  "name": "@example/react-components",
  "pnpm": {
    "templates": {
      "authoring": ["@example/authoring-template@0.1.0"],
    }
  }
}
```

**In-Memory and Publish Time**

```json5
{
  "name": "@example/react-components",
  "author": "Example Organization <team@organization.example>",
  "license": "MIT",
  "repository": "git@organization.example/example/project"
}
```

### Toolkit

The `pnpm.templates.toolkit` flavor copies the `dependencies` block of the template and spreads it into the `devDependencies` of the referencing package.

Given the following template:

```json5
{
  "name": "@example/react-toolkit",
  "dependencies": {
    "jest": "^29.4.3",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "redux": "^4.2.1"
  }
}
```

A package referencing the `@example/react-toolkit` above will have the following on-disk and in-memory representations.

**On-Disk**

```json5
{
  "name": "@example/react-components",
  "pnpm": {
    "templates": {
      "toolkit": ["@example/react-toolkit@0.1.0"],
    }
  }
}
```

**In-Memory and Publish Time**

```json5
{
  "name": "@example/react-components",
  "dependencies": {
    "redux": "^4.2.1"
  },
  "devDependencies": {
    "jest": "^29.4.3",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "redux": "^4.2.1"
  }
}
```

This allows package authors to share a reusable set of dependencies that can be imported or required in the referencing package. Dependencies of dependencies are not normally visible in this manner since pnpm sets up a semi-strict `node_modules` structure by default.

### Catalog

The `pnpm.templates.catalog` flavor allows packages to declare a dependency using a version specifier from the referenced template. The `dependencies` block of the template will be available to reference through the `catalog:` version specifier protocol.

```json5
{
  "name": "@example/frontend-catalog",
  "pnpm": {
    "template": "catalog"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "redux": "^4.2.0",
    "react-redux": "^8.0.0"
  }
}
```

A package referencing the `@example/frontend-catalog` above will have the following on-disk and in-memory representations.

**On-Disk**

```json5
{
  "name": "@example/react-components",
  "pnpm": {
    "templates": {
      "catalog": {
        "example-frontend": "@example/frontend-catalog@0.1.0"
      }
    }
  },
  "dependencies": {
    "react": "catalog:example-frontend",
    "redux": "catalog:example-frontend"
  }
}
```

**In-Memory and Publish Time**

```json5
{
  "name": "@example/react-components",
  "dependencies": {
    "react": "^18.2.0",
    "redux": "^4.2.0",
  }
}
```

We expect the `pnpm.templates.catalog` flavor to very popular for monorepos. This allows dependency specifiers to be consistent between different in-repo packages.

There are a few rules on how a catalog defines shared dependencies specifiers and how they can be consumed.

- The `dependencies` block of a catalog can be used in the `dependencies`, `devDependencies`, and `optionalDependencies` of a consuming `package.json`.
- The `peerDependencies` block of a catalog can only be used in the `peerDependencies` block of a consuming `package.json`.

### Compatibility and Patches

The `compatibility` and `patches` template flavors apply to the root `package.json` of a pnpm workspace and allow modifications to the dependency graph.

```json5
{
  "name": "@example/patches",
  "version": "0.1.0",
  "pnpm": {
    "template": "patches",

    // https://pnpm.io/package_json#pnpmpackageextensions
    "packageExtensions": {
      "react-redux": {
        "peerDependencies": {
          "react-dom": "*"
        }
      }
    },

    // https://pnpm.io/package_json#pnpmpatcheddependencies
    "patchedDependencies": {
      "express@4.18.1": "patches/express@4.18.1.patch"
    }
  }
}
```

Settings that reference local files (such as `patches/express@4.18.1.patch` above) will be looked up relative to the template producing `package.json` rather than the consuming `package.json`. If the referenced `patches` template is an external package, it will be fetched and linked into `node_modules/.pnpm`. This is a special case of the patches template flavor due to the need to reference files within the package. Other template flavors are not linked into `node_modules/.pnpm`.

```
node_modules/.pnpm/@example+patches/node_modules/@example/patches/patches/express@4.18.1.patch
```

### Combining Templates

Templates may not be templated from other templates. Instead of an inheritance hierarchy, a package may refer to multiple templates with later entries in the list taking precedence.

For example, if both `@organization/authoring-metadata` and `@team/authoring-metadata` have an `author` field, the value from `@team/authoring-metadata` will be used.

```json5
{
  "name": "@example/simple",
  "pnpm": {
    "templates": {
      "authoring": [
        "@organization/authoring-metadata@0.1.0",
        "@team/authoring-metadata@0.1.0"
      ],
    }
  }
}
```

Avoiding inheritance simplifies implementation and usability in several ways.

- Suppose a new version of pnpm adds `pnpm.templates` config options. In an inheritance model, using these new options on a template would force consumers to a greater minimum version of pnpm.
- Complex circular resolution during installation is avoided.
- Template loading is more performant. The requirement to declare all required templates up front enables fetching in a single parallelized step, rather than a waterfall fetch at each level of a theoretical template inheritance hierarchy.
- It becomes possible to reference multiple templates the package's author does not control with less ambiguity. In an inheritance-based mechanism, multiple inheritance would be necessary for such functionality. However, multiple inheritance can lead to [diamond-shaped problems](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem) and less clarity as to what the final value of the field should be.

A few of the problems above can be mitigated by rendering the template package itself before publishing, but this introduces other problems. Toolkit templates install dependencies in the referencing package as `devDependencies`, which isn't desirable in other templates.

The existing design is inspired by the concept of ["_composition over inheritance_"](https://en.wikipedia.org/wiki/Composition_over_inheritance), which provides reusability without defining difficult to change relationships between packages.

### Specifiers

Templates can be specified through:

1. The `name@version` syntax to refer to an external template fetched from a registry.
2. The `name@workspace:` syntax to refer to a template that's also a workspace package.
3. The `file:<path>` syntax to refer to an on-disk path.

## Rationale and Alternatives

### Syncpack

[Syncpack](https://github.com/JamieMason/syncpack/) is a great open source tool for keeping `package.json` dependency specifiers in sync on disk.

The proposed solution allows metadata to be defined in a singular file without copying definitions to other files on disk. This is a capability only possible by the package manager reading `package.json` files.

### Comparison to overrides/resolutions

An alternative mechanism for the version catalog is the [`pnpm.overrides` feature](https://pnpm.io/package_json#pnpmoverrides). While mechanically this allows you to set the version of a dependency across all workspace packages, it can be a bit unexpected when if `pnpm.overrides` rewrites a dependency's dependency to an incompatible version silently.

`pnpm.overrides` is ultimately intended for a different purpose. The NPM RFC for a similar feature explicitly states that it should be used as a short-term hack to fix vendor problems.

> Using this feature should be considered a hack in most cases, something that is done temporarily while waiting for a bug to be fixed, or to avoid excessive duplication caused by an overly strict meta-dependency specifier.
https://github.com/npm/rfcs/blob/main/accepted/0036-overrides.md

The `catalog:` protocol is conversely intended for long-lived usage.

### Copying all fields

An earlier draft of this RFC considered a `pnpm.templates.extends` flavor. This would allow a child `package.json` to copy all fields from its parent. This won't be present in the initial version of pnpm templates for a few reasons.

- **Security concerns**: An external template may initially provide dependencies to inherit, but maliciously override fields such as `license` or `funding` in an update. This introduces a new form of security vulnerability in the ecosystem. Authors of consuming packages would need to trust or review every update to an `extends` template, which is impractical to expect.
- **Copying ambiguity**: Certain fields such as `version` may not be desirable to inherit from a parent `package.json`. The `extends` flavor would need to exclude copying for some fields. This makes a theoretical `extends` flavor difficult to understand and result in surprises at publish time.
- **Merging ambiguity**: It's not always clear how to "_merge_" fields. For example, if a consuming package and template both define `pnpm.allowedDeprecatedVersions.express`, should pnpm replace the field from the template with the child package's value, or union the allowed deprecated versions? What should `extends` do for fields added in newer versions of pnpm that it doesn't know how to merge?

A future version of pnpm templates may provide this flavor, but significant thoughtfulness around the above would be necessary. For now, we believe all use cases involving an `extends` flavor could be addressed through the introduction of new template flavors.

## Implementation

### Fetching

Most templates are not installed as standard dependencies and linked into `node_modules`. If a package refers to an external template on an NPM registry, only the metadata will be fetched. The fetch will be performed [without the abbreviated header](https://github.com/npm/registry/blob/master/docs/responses/package-metadata.md#abbreviated-metadata-format), so the full document is retrieved. The metadata will be saved to the content-addressable store for fast subsequent lookups.

For the `catalog` template, not installing the template into `node_modules` provides a performance optimization. Only the subset of the version catalog that's used will be fetched and installed.

The exception is the `patches` template flavor since it may refer to local patch files. The full NPM package producing the `patches` template will be fetched, added to the pnpm store, and linked into `node_modules/.pnpm`.

### Lockfile

The relevant sections of each template will be saved to `pnpm-lock.yaml` under a new `templates` key. This allows users to more easily review changes to templates and pnpm to perform faster up-to-date checks.

```yaml
lockfileVersion: next

importers:
  # ...

templates:

  /@example/frontend-catalog@0.1.0:
    resolution: {integrity: sha512...}
    pnpm:
      template: catalog
    dependencies:
      react: ^18.2.0
      react-dom: ^18.2.0
      redux: ^4.2.0
      react-redux: ^8.0.0

  /@example/patches@0.1.0:
    resolution: {integrity: sha512...}
    pnpm:
      template: patches
    packageExtensions:
      react-redux:
        peerDependencies:
          react-dom: "*"
    patchedDependencies:
      express@4.18.1: patches/express@4.18.1.patch

packages:
  # ...
```

Similar to external dependencies under the `packages` block, external templates will store the `resolution.integrity` field provided from the registry in its lockfile entry. Note that only the `patches` template flavor actually uses the tarball associated with the integrity hash. The other template flavors will store the tarball integrity for consistency.

In the initial implementation, `importers` entries referencing a template will have their rendered result saved to the lockfile.

### Portability

Usage of an template package is localized to a monorepo. When a package within the monorepo referencing a template is exported through `pnpm publish` or `pnpm pack`, the resulting `package.json` will be the merged contents of the original `package.json` and its references.

This allows the published `package.json` to be portable and consumed by other package managers.

### Debugging and Tracing

A command to view the rendered result will be available to assist debugging. When `pnpm template view` is ran, the `package.json` file in the current working directory will be rendered. The `--trace` flag will show which template a field value was chosen from.

```
❯ pnpm template view --trace
```

<img width="1050" alt="Screenshot of pnpm view template command" src="https://github.com/pnpm/rfcs/assets/906558/7350f337-5b5a-4725-aa1a-e65bb367f31d">

## Prior Art

> This section is optional if there are no actual prior examples in other tools

> Discuss existing examples of this change in other tools, and how they've addressed various concerns discussed above, and what the effect of those decisions has been

- [RFC: First-Class Support for Workspace Consistent Versions](https://github.com/pnpm/rfcs/pull/1)
- [RFC for parent package.json npm/rfcs#165](https://github.com/npm/rfcs/pull/165)
- [[RRFC] Accepting version references within dependencies and devDependencies](https://github.com/npm/rfcs/issues/677)
