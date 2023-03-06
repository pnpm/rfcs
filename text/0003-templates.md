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

## Detailed Explanation

A "*Template*" is defined as a normal `package.json` file with the `pnpm.template` field set.

```json5
{
  "name": "@example/react-template",
  "pnpm": {
    "template": true,
  },
  "version": "0.1.0",
  "author": "Example Team <team@team.example>",
  "license": "MIT",
  "scripts": {
    "compile": "tsc"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
  }
}
```

Templates must opt into being a template by defining `pnpm.template`. This is a safety mechanism to:

1. Intentionally prevent existing packages from being referred to for that purpose. Templates have different obligations and usage than standard packages that authors of templates should be aware of.
2. Allow checks to be performed on the template before publishing and during consumption. Not all `package.json` fields may be valid on a template.

A package can reference the template to populate different portions of its definition. The specific feature is determined by `pnpm.templates`: `extends`, `toolkit`, and `catalog`.

### Extends

Using the `pnpm.templates.extends` mechanism, `package.json` fields from templates are copied with small exceptions. The `name`, `dist`, and underscore prefixed fields (e.g. `_npmUser`) are not copied since they're typically specific to the template package itself.

A package referencing the `@example/react-template` above will have the following on-disk and in-memory representations.

**On-Disk**

```json5
{
  "name": "@example/react-components",
  "pnpm": {
    "templates": {
      "extends": ["@example/react-template@0.1.0"],
    }
  }
}
```

**In-Memory and Publish Time**

```json5
{
  "name": "@example/react-components",
  "version": "0.1.0",
  "author": "Example Team <team@team.example>",
  "license": "MIT",
  "scripts": {
    "compile": "tsc"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
  }
}
```

Note that non-standard `package.json` fields such as `eslintConfig` will also be copied. Although these fields have no meaning to package managers, they will appear in the published manifest.

### Toolkit

The `pnpm.templates.toolkit` field copies the `dependencies` block of the template and spreads it into the `devDependencies` of the referencing package.

**On-Disk**

```json5
{
  "name": "@example/react-components",
  "pnpm": {
    "templates": {
      "toolkit": ["@example/react-template@0.1.0"],
    }
  },
  "devDependencies": {
    "jest": "^29.4.3"
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
  }
}
```

This allows package authors to share a reusable set of tools that can be imported or required in the referencing package. Dependencies of dependencies are not normally visible in this manner since pnpm sets up a semi-strict `node_modules` structure by default.

### Catalog

The `pnpm.templates.catalog` feature allows packages to declare a dependency using a version specifier from the referenced template. The `dependencies` block of the template will be available to reference through the `catalog:` version specifier protocol.

**On-Disk**

```json5
{
  "name": "@example/react-components",
  "pnpm": {
    "templates": {
      "catalog": ["@example/react-template@0.1.0"],
    }
  },
  "dependencies": {
    "react": "catalog:"
  }
}
```

**In-Memory and Publish Time**

```json5
{
  "name": "@example/react-components",
  "dependencies": {
    "react": "^18.2.0"
  }
}
```

We expect most monorepos to use this feature to keep dependency specifiers consistent between different in-repo packages.

### Combining Templates

Templates may not refer to other templates. Instead of an inheritance hierarchy, a package may refer to multiple templates with later entries in the list taking precedence.

For example, if both `@organization/authoring-metadata` and `@team/authoring-metadata` have an `author` field, the value from `@team/authoring-metadata` will be used.

```json5
{
  "name": "@example/simple",
  "pnpm": {
    "templates": {
      "extends": [
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

The existing design is inspired from the concept of "traits" in some programming languages, which provide reusability without inheritance.

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

## Implementation

### Fetching

Templates are not installed as standard dependencies and linked into `node_modules`. If a package refers to an external template on an NPM registry, only the metadata will be fetched. The fetch will be performed [without the abbreviated header](https://github.com/npm/registry/blob/master/docs/responses/package-metadata.md#abbreviated-metadata-format), so the full document is retrieved.

If only a subset of the version catalog is used, the full catalog of dependencies does not need to be installed.

### Lockfile

No explicit changes will be made to the `pnpm-lock.yaml` file. Any `importers` entries referencing a template will have their rendered result saved to the lockfile.

This may be revisited if it affects pnpm's performance when performing up to date checks.

### Portability

Usage of an template package is localized to a monorepo. When a package within the monorepo referencing a template is exported through `pnpm publish` or `pnpm pack`, the resulting `package.json` will be the merged contents of the original `package.json` and its references.

This allows the published `package.json` to be portable and consumed by other package managers.

A command to view the rendered result will be available to assist debugging.

## Prior Art

> This section is optional if there are no actual prior examples in other tools

> Discuss existing examples of this change in other tools, and how they've addressed various concerns discussed above, and what the effect of those decisions has been

- [RFC: First-Class Support for Workspace Consistent Versions](https://github.com/pnpm/rfcs/pull/1)
- [RFC for parent package.json npm/rfcs#165](https://github.com/npm/rfcs/pull/165)
- [[RRFC] Accepting version references within dependencies and devDependencies](https://github.com/npm/rfcs/issues/677)

## Unresolved Questions and Bikeshedding

- In prior discussions, this feature was referred to as "_Environments_". The initial draft proposes "_Templates_" to make it more clear that this feature is simply a `package.json` authoring mechanism. Templates/environments are not themselves installed.
- Should the `version` field be extended by `pnpm.templates.extends`? One on hand, this allows monorepo packages to be single-versioned easily. On the other hand, it can be very surprising when packages are published with the version from a template. Previously forgetting to specify a `version` results in a helpful error.
