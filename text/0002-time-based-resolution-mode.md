# Time-based resolution mode

## Summary

A new resolution mode that resolves versions not to their highest versions but to versions that were published in a given time frame.

## Motivation

The resolution mode that is currently used by pnpm is not optimized for caching. During installation we need to always fetch the metadata of each package as we don't know if the metadata in the cache has the highest version of the package. Also, installation of the highest version is dangerous as someone may hijack a subdependency, publish a new version, and pnpm will immediately install it.

## Detailed Explanation

Resolution will be divided into two stages. The first stage will resolve all the direct dependencies of all the workspace projects. This stage may work the same way as it works now (highest version is picked). When all the direct dependencies are resolved, we check when were the picked versions released. This information is present in the package metadata at the "time" field. For example, if we install webpack and eslint, we get webpack resolved to v5.74.0 and eslint resolved to v8.22.0. `webpack@5.74.0` was released at "2022-07-25T08:00:33.823Z". `eslint@8.22.0` was released at "2022-08-14T01:23:41.730Z". Now we compare the dates of each released packages and pick the nearest date. In this case, the nearest date is the date eslint was released: "2022-08-14T01:23:41.730Z". We add some delta (e.g. 1 hour) to this time and call the resulting date T. The delta is needed because in some monorepos packages may be published in random order, so dependent packages may be released after dependency packages.

At the second stage, we resolve all the subdependencies. At this stage, instead of resolving a range to the highest available version, we filter out any versions that were released after T and pick the highest version from those. Let's say we need to resolve `ajv@^6.10.0`. We already have a metadata of ajv in cache and it was saved after T, so we don't need to redownload it. This are the versions of ajv that match `^6.10.0`:

| version | release date |
|--|--|
| 6.10.0 | 2021-01-08 |
| 6.10.1 | 2021-12-22 |
| 6.10.2 | 2022-01-25 |
| 6.11.0 | 2022-04-05 |
| 6.11.1 | 2022-08-16 |

As you can see, v6.11.1 was released after T, so we pick v6.11.0, which is the highest version that satisfies the range and the time frame.

## Rationale and Alternatives

An alternative solution might be to resolve to the lowest version. This is a resolution mode that [Maël Nison](https://github.com/arcanis) came up with and he already works on the [implementation in Yarn](https://github.com/yarnpkg/berry/pull/4351). It is called "stable" resolution mode. This resolution mode is harder to implement in pnpm. pnpm starts to fetch a package immediately after resolving it. This is fine with "hightest" resolution mode because different ranges are resolved to the same highest version. But when the lowest version is picked, the package manager should do some deduplication before starting the fetching process. For instance, in the root, `foo@^1.0.0` might be in dependencies. pnpm will immediately resolve it to `1.0.0` and fetch it from the registry. But then in the subdependencies there might be `foo@^1.1.0` in the dependencies, so pnpm will need to download a second version of foo even though 1.1.0 would have satisfied the root as well. This is easier to do with Yarn, which has a separate stage for resolution.

## Implementation

{{Give a high-level overview of implementation requirements and concerns. Be specific about areas of code that need to change, and what their potential effects are. Discuss which repositories and sub-components will be affected, and what its overall code effect might be.}}

{{THIS SECTION IS REQUIRED FOR RATIFICATION -- you can skip it if you don't know the technical details when first submitting the proposal, but it must be there before it's accepted}}

## Prior Art

{{This section is optional if there are no actual prior examples in other tools}}

{{Discuss existing examples of this change in other tools, and how they've addressed various concerns discussed above, and what the effect of those decisions has been}}

## Unresolved Questions and Bikeshedding

{{Write about any arbitrary decisions that need to be made (syntax, colors, formatting, minor UX decisions), and any questions for the proposal that have not been answered.}}

{{THIS SECTION SHOULD BE REMOVED BEFORE RATIFICATION}}

