# Execution Environments

## Summary

This RFC introduces settings for controlling what execution environment (Node.js, Bun, Deno) will be used for a package during:

* runnings its lifecycle scripts
* building
* running it as a CLI app

## Motivation

Running multiple versions of Node.js on the same computer isn't easy. Also, there is currently no way for a package to tell the package manager that it needs to be executed with a specific version of Node.js. Node.js versions should be locked the same way as other dependencies of projects are locked for reproducibility.

## Detailed Explanation

We will support a new field in `package.json`: `pnpm.executionEnv.jsRuntime`. This field will be similar to the `packageManager` field introduced by Corepack but will feature `<js runtime>@<version>` instead. For example:

```json
{
  "pnpm": {
    "executionEnv": {
      "jsRuntime": "node@20.16.0"
    }
  }
}
```

or

```json
{
  "pnpm": {
    "executionEnv": {
      "jsRuntime": "bun@1.1.26"
    }
  }
}
```

or

```json
{
  "pnpm": {
    "executionEnv": {
      "jsRuntime": "deno@1.46.1"
    }
  }
}
```

When pnpm sees this setting, it will load the specified runtime and use it for:

* running scripts locally (via the `pnpm run` and `pnpm exec` command)
* running build scripts, when installed as a dependency. If the package has a postinstall script, it will be executed by the specified runtime.
* running the package, when it is executed as a CLI.

If we don't want to control the execution env of the published package, set the optional `localOnly` setting to `true`. For instance:

```json
{
  "name": "cowsay",
  "version": "1.0.0",
  "bin": "bin.js",
  "pnpm": {
    "executionEnv": {
      "jsRuntime": "node@20.16.0",
      "localOnly": true
    }
  }
}
```

In this case, pnpm will remove the `executionEnv` setting from the `package.json` file on publish and the binary of the package will be executed with whatever runtime will be installed globally on the target machine.

Some environments might not want to allow pnpm to control the js runtime. For thes cases we need to support a setting that will instruct pnpm to ignore all the execution env settings: `ignore-execution-env=true`.

## Rationale and Alternatives

The alternative would be to use a third party tool for this (like Volta) but then we would have one more prerequisite for using pnpm.

## Implementation

The implementation can leverage the logic that is already present in pnpm for the `pnpm env` command, the `pnpm.executionEnv.nodeVersion` setting, the `use-node-version` setting.

Binding CLI apps to specific Node.js versions can be done via command shims. This currently works for globally installed packages. pnpm links globally installed packages to the active Node.js version.

## Prior Art

We already have some functionality for managing Node.js versions:

* the [pnpm env](https://pnpm.io/cli/env) command
* the [use-node-version](https://pnpm.io/npmrc#use-node-version) setting
* the [pnpm.executionEnv.nodeVersion](https://pnpm.io/package_json#pnpmexecutionenvnodeversion) setting.

## Unresolved Questions and Bikeshedding

{{Write about any arbitrary decisions that need to be made (syntax, colors, formatting, minor UX decisions), and any questions for the proposal that have not been answered.}}

{{THIS SECTION SHOULD BE REMOVED BEFORE RATIFICATION}}

