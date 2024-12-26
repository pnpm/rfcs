# Configurational dependencies

## Summary

A new type of dependencies for installing configuration and hooks.

## Motivation

We want to make it possible to share some configurations between projects. Storing the configuration in regular dependencies is not an option as we might need the configuration during installation of the "regular" dependencies ("dependencies", "devDependencies", and "optionalDependencies"). Hence, we need to install these configurational dependencies before other types of dependencies.

Some examples of usage:

* Installing hooks used by [`.pnpmfile.cjs`](https://pnpm.io/pnpmfile).
* Installing the list of dependencies that are allowed to be built (the [pnpm.onlyBuiltDependenciesFile](https://pnpm.io/package_json#pnpmonlybuiltdependenciesfile)).
* Installing [catalogs](https://pnpm.io/catalogs).
* Loading patch file.

## Detailed Explanation

There will be a new field in `package.json` called `pnpm.configDependencies`. For example:

```json
{
  "name": "my-pkg",
  "version": "0.0.0",
  "pnpm": {
    "configDependencies": {
      "my-configs": "1.0.0+sha512-30iZtAPgz+LTIYoeivqYo853f02jBYSd5uGnGpkFV0M3xOt9aN73erkgYAmZU43x4VfqcnLxW9Kpg3R5LC4YYw=="
    }
  }
}
```

These new type of "configurational" dependencies will be npm packages with a lot of limitations:

* They won't have any dependencies. Even if they will have dependencies, pnpm will ignore them during installation.
* They will not have lifecycle scripts.
* They will only be installable via exact versions.

These dependencies will be installed into a new directory (name to be decided), not into `node_modules`.

## Rationale and Alternatives

The `"onlyBuiltDependencies"` list can currently be loaded from `node_modules`. However, this introduces "works on my machine" issues since the list isn't available during the link-from-store stage. As a result, if the dependency is not allowed to be built, it might still be loaded from the side-effects cache.

## Implementation

These dependencies will be installed as early as possible in order to be able to load settings from it. So this should happen ouside of the `@pnpm/core` module. When we call `mutateModules` we should already have all the configurations from `pnpm.configDependencies` loaded and ready.

## Prior Art

## Unresolved Questions and Bikeshedding

{{Write about any arbitrary decisions that need to be made (syntax, colors, formatting, minor UX decisions), and any questions for the proposal that have not been answered.}}

{{THIS SECTION SHOULD BE REMOVED BEFORE RATIFICATION}}

