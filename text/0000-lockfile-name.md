# RFC: Add a new `lockfile-name` config to the `.npmrc` file\*\*

## Summary

This RFC proposes to add a new `lockfile-name` config to the `.npmrc` file. This would allow users to specify the name of the pnpm lockfile.

## Motivation

The current pnpm lockfile is named `pnpm-lock.yaml`. However, some users may want to change the name of the lockfile for various reasons, such as to match their own naming conventions.

## Detailed Explanation

The `lockfile-name` config would be a string value. The default value would be `pnpm-lock.yaml`.

If the `lockfile-name` config is specified, pnpm would use that name for the lockfile instead of the default name.

## Rationale and Alternatives

There are a few other ways to allow users to specify the name of the pnpm lockfile. One option would be to add a new command-line option, such as `--lockfile-name`. Another option would be to allow users to set the `lockfile-name` config in the `package.json` file.

However, I believe that adding a new config to the `.npmrc` file is the best solution. This is because the `.npmrc` file is already used to configure other aspects of pnpm, such as the package registry. Adding the `lockfile-name` config to the `.npmrc` file would make it consistent with other pnpm configuration options.

## Implementation

The following are some of the steps that would need to be taken to implement the `lockfile-name` config:

1. Add a new `lockfile-name` field to the `Config` interface in the `@pnpm/config` package, and also define a default value for `lockfile-name` in the getConfig() method, and define `lockfile-name` constructor in the `types` object.
2. Update the pnpm code to use the `lockfile-name` config when generating and reading the lockfile instead of using the constant value `WANTED_LOCKFILE` or the hard-coded value `pnpm-lock.yaml`.
3. Write tests to ensure that the `lockfile-name` config works as expected.

## Unresolved Questions and Bikeshedding

One unresolved question is whether or not to add a new command-line option to allow users to specify the lockfile name. I believe that this is not necessary, since users can already specify the lockfile name in the `.npmrc` file.

Finally, we need to decide on a name for the new config. I have proposed the name `lockfile-name`, but other suggestions are welcome.

## Additional Considerations

Add support for a `.pnpm` config folder:

We may also want to consider adding support for a `.pnpm` config folder. This would allow users to centralize all of their pnpm configuration options in a single place.

The `.pnpm` config folder would be created in the workspace root directory. The `.pnpm` config folder would contain all of the pnpm configuration files, such as `lock.yaml` and `workspace.yaml`

## Conclusion

I believe that adding a new `lockfile-name` config to the `.npmrc` file would be a valuable addition to pnpm. It would give users more flexibility to name their lockfile however they want. It would also avoid conflicts with other tools and allow users to match their own naming conventions.

I welcome feedback on this proposal.
