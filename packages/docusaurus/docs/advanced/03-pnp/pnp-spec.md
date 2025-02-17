---
category: advanced
slug: /advanced/pnp-spec
title: "PnP Specification"
description: In-depth documentation of the PnP spec.
---

## About this document

To make interoperability easier for third-party projects, this document describes the specification we follow when installing files on disk under the [Plug'n'Play install strategy](/features/pnp). It also means:

- any change we make to this document will follow semver rules
- we'll do our best to preserve backward compatibility
- new features will be intended to gracefully degrade

## High-level idea

Plug'n'Play works by keeping in memory a table of all packages part of the dependency tree, in such a way that we can easily answer two different questions:

- Given a path, what package does it belong to?
- Given a package, where are the dependencies it can access?

Resolving a package import thus becomes a matter of interlacing those two operations:

- First, locate which package is requesting the resolution
- Then retrieve its dependencies, check if the requested package is amongst them
- If it is, then retrieve the dependency information, and return its location

Extra features can then be designed, but are optional. For example, Yarn leverages the information it knows about the project to throw semantic errors when a dependency cannot be resolved: since we know the state of the whole dependency tree, we also know why a package may be missing.

## Basic concepts

All packages are uniquely referenced by **locators**. A locator is a combination of a **package ident**, which includes its scope if relevant, and a **package reference**, which can be seen as a unique ID used to distinguish different instances (or versions) of a same package. The package references should be treated as an opaque value: it doesn't matter from a resolution algorithm perspective that they start with `workspace:`, `virtual:`, `npm:`, or any other protocol.

### Portability

For portability reasons, all paths inside of the manifests:
- must use the unix path format (`/` as separators).
- must be relative to the manifest folder (so they are the same regardless of the location of the project on disk).

:::caution
All algorithms in this specification assume that paths have been normalized according to these two rules.
:::

## Fallback

For improved compatibility with legacy codebases, Plug'n'Play supports a feature we call "fallback". The fallback triggers when a package makes a resolution request to a dependency it doesn't list in its dependencies. In normal circumstances the resolver would throw, but when the fallback is enabled the resolver should first try to find the dependency packages amongst the dependencies of a set of special packages. If it finds it, it then returns it transparently.

In a sense, the fallback can be seen as a limited and safer form of hoisting. While hoisting allows unconstrainted access through multiple levels of dependencies, the fallback requires to explicitly define a fallback package - usually the top-level one.

## Package locations

While the Plug'n'Play specification doesn't by itself require runtimes to support anything else than the regular filesystem when accessing package files, producers may rely on more complex data storage mechanisms. For instance, Yarn itself requires the two following extensions which we strongly recommend to support:

### Zip access

Files named `*.zip` must be treated as folders for the purpose of file access. For instance, `/foo/bar.zip/package.json` requires to access the `package.json` file located within the `/foo/bar.zip` zip archive.

If writing a JS tool, the [`@yarnpkg/fslib`](https://yarnpkg.com/package/@yarnpkg/fslib) package may be of assistance, providing a zip-aware filesystem layer called `ZipOpenFS`.

### Virtual folders

In order to properly represent packages listing peer dependencies, Yarn relies on a concept called [Virtual Packages](/advanced/lexicon#virtual-package). Their most notable property is that they all have different paths (so that Node.js instantiates them as many times as needed), while still being baked by the same concrete folder on disk.

This is done by adding path support for the following scheme:

```
/path/to/some/folder/__virtual__/<hash>/<n>/subpath/to/file.dat
```

When this pattern is found, the `__virtual__/<hash>/<n>` part must be removed, the `hash` ignored, and the `dirname` operation applied `n` times to the `/path/to/some/folder` part. Some examples:

```
/path/to/some/folder/__virtual__/a0b1c2d3/0/subpath/to/file.dat
/path/to/some/folder/subpath/to/file.dat

/path/to/some/folder/__virtual__/e4f5a0b1/0/subpath/to/file.dat
/path/to/some/folder/subpath/to/file.dat (different hash, same result)

/path/to/some/folder/__virtual__/a0b1c2d3/1/subpath/to/file.dat
/path/to/some/subpath/to/file.dat

/path/to/some/folder/__virtual__/a0b1c2d3/3/subpath/to/file.dat
/path/subpath/to/file.dat
```

If writing a JS tool, the [`@yarnpkg/fslib`](https://yarnpkg.com/package/@yarnpkg/fslib) package may be of assistance, providing a virtual-aware filesystem layer called `VirtualFS`.

:::note
The `__virtual__` folder name appeared with Yarn 3.0. Earlier releases used `$$virtual`, but we changed it after discovering that this pattern triggered bugs in softwares where paths were used as either regexps or replacement. For example, `$$` found in the second parameter from [`String.prototype.replace`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace) silently turned into `$`.
:::

## Manifest reference

When [`pnpEnableInlining`](/configuration/yarnrc#pnpEnableInlining) is explicitly set to `false`, Yarn will generate an additional `.pnp.data.json` file containing the following fields.

This document only covers the data file itself - you should define your own in-memory data structures, populated at runtime with the information from the manifest. For example, Yarn turns the `packageRegistryData` table into two separate memory tables: one that maps a path to a package, and another that maps a package to a path.

:::info
You may notice that various places use arrays of tuples in place of maps. This is mostly intended to make it easier to hydrate ES6 maps, but also sometimes to have non-string keys (for instance `packageRegistryData` will have a `null` key in one particular case).
:::

import pnpSchema from '@yarnpkg/docusaurus/static/configuration/pnp.json';
import theme     from 'prism-react-renderer/themes/vsDark';
import {JsonDoc} from 'react-json-doc';

<JsonDoc theme={theme} extraTheme={{
  container: {borderRadius: `var(--ifm-code-border-radius)`},
  inactiveHeader: {},
  activeHeader: {borderRadius: `var(--ifm-code-border-radius)`, background: `#3d437c`},
  annotation: {borderRadius: `var(--ifm-code-border-radius)`, background: `#383944`, color: `#ffffff`},
  anchor: {scrollMarginTop: 60},
  section: {fontFamily: `SFMono-Regular, Menlo, Monaco, Consolas, "Liberation Mono", "Courier New", monospace`},
}} data={pnpSchema}/>

## Resolution algorithm

:::info
For simplicity, this algorithm doesn't mention all the Node.js features that allow mapping a module to another, such as [`imports`](https://nodejs.org/api/packages.html#imports), [`exports`](https://nodejs.org/api/packages.html#exports), or other vendor-specific features.
:::

### NM_RESOLVE

```
NM_RESOLVE(specifier, parentURL)
```

1. This function is specified in the [Node.js documentation](https://nodejs.org/api/esm.html#resolver-algorithm-specification)

### PNP_RESOLVE

```
PNP_RESOLVE(specifier, parentURL)
```

1. Let `resolved` be **undefined**

2. If `specifier` is a Node.js builtin, then

    1. Set `resolved` to `specifier` itself and return it

3. Otherwise, if `specifier` is either an absolute path or a path prefixed with "./" or "../", then

    1. Set `resolved` to [`NM_RESOLVE`](#nm_resolve)`(specifier, parentURL)` and return it

4. Otherwise,

    1. Note: `specifier` is now a bare identifier

    2. Let `unqualified` be [`RESOLVE_TO_UNQUALIFIED`](#resolve_to_unqualified)`(specifier, parentURL)`

    3. Set `resolved` to [`NM_RESOLVE`](#nm_resolve)`(unqualified, parentURL)`

### RESOLVE_TO_UNQUALIFIED

```
RESOLVE_TO_UNQUALIFIED(specifier, parentURL)
```

1. Let `resolved` be **undefined**

2. Let `ident` and `modulePath` be the result of [`PARSE_BARE_IDENTIFIER`](#parse_bare_identifier)`(specifier)`

3. Let `manifest` be [`FIND_PNP_MANIFEST`](#find_pnp_manifest)`(parentURL)`

4. If `manifest` is null, then

    1. Set `resolved` to [`NM_RESOLVE`](#nm_resolve)`(specifier, parentURL)` and return it

5. Let `parentLocator` be [`FIND_LOCATOR`](#find_locator)`(manifest, parentURL)`

6. If `parentLocator` is null, then

    1. Set `resolved` to [`NM_RESOLVE`](#nm_resolve)`(specifier, parentURL)` and return it

7. Let `parentPkg` be [`GET_PACKAGE`](#get_package)`(manifest, parentLocator)`

8. Let `referenceOrAlias` be the entry from `parentPkg.packageDependencies` referenced by `ident`

9. If `referenceOrAlias` is **null** or **undefined**, then

    1. If `manifest.enableTopLevelFallback` is **true**, then

        1. If `parentLocator` **isn't** in `manifest.fallbackExclusionList`, then

            1. Let `fallback` be [`RESOLVE_VIA_FALLBACK`](#resolve_via_fallback)`(manifest, ident)`

            2. If `fallback` is neither **null** nor **undefined**

                1. Set `referenceOrAlias` to `fallback`

10. If `referenceOrAlias` is still **undefined**, then

    1. Throw a resolution error

11. If `referenceOrAlias` is still **null**, then

    1. Note: It means that `parentPkg` has an unfulfilled peer dependency on `ident`

    2. Throw a resolution error

12. Otherwise, if `referenceOrAlias` is an array, then

    1. Let `alias` be `referenceOrAlias`

    2. Let `dependencyPkg` be [`GET_PACKAGE`](#get_package)`(manifest, alias)`

    3. Return `path.resolve(manifest.dirPath, dependencyPkg.packageLocation, modulePath)`

13. Otherwise,

    1. Let `reference` be `referenceOrAlias`

    2. Let `dependencyPkg` be [`GET_PACKAGE`](#get_package)`(manifest, {ident, reference})`

    3. Return `path.resolve(manifest.dirPath, dependencyPkg.packageLocation, modulePath)`

### GET_PACKAGE

```
GET_PACKAGE(manifest, locator)
```

1. Let `referenceMap` be the entry from `parentPkg.packageRegistryData` referenced by `locator.ident`

2. Let `pkg` be the entry from `referenceMap` referenced by `locator.reference`

3. Return `pkg`

    1. Note: `pkg` cannot be **undefined** here; all packages referenced in any of the Plug'n'Play data tables [**MUST**](#must) have a corresponding entry inside `packageRegistryData`.

### FIND_LOCATOR

```
FIND_LOCATOR(manifest, moduleUrl)
```

Note: The algorithm described here is quite inefficient. You should make sure to prepare data structure more suited for this task when you read the manifest.

1. Let `bestLength` be **0**

2. Let `bestLocator` be **null**

3. Let `relativeUrl` be the relative path between `manifest` and `moduleUrl`

    1. Note: The relative path must not start with `./`; trim it if needed

4. If `relativeUrl` matches `manifest.ignorePatternData`, then

    1. Return **null**

5. Let `relativeUrlWithDot` be `relativeUrl` prefixed with `./` or `../` as necessary

6. For each `referenceMap` value in `manifest.packageRegistryData`

    1. For each `registryPkg` value in `referenceMap`

        1. If `registryPkg.discardFromLookup` **isn't true**, then

            1. If `registryPkg.packageLocation.length` is greater than `bestLength`, then

                1. If `relativeUrl` starts with `registryPkg.packageLocation`, then

                    1. Set `bestLength` to `registryPkg.packageLocation.length`

                    2. Set `bestLocator` to the current `registryPkg` locator

6. Return `bestLocator`

### RESOLVE_VIA_FALLBACK

```
RESOLVE_VIA_FALLBACK(manifest, ident)
```

1. Let `topLevelPkg` be [`GET_PACKAGE`](#get_package)`(manifest, {null, null})`

2. Let `referenceOrAlias` be the entry from `topLevelPkg.packageDependencies` referenced by `ident`

3. If `referenceOrAlias` is defined, then

    1. Return it immediately

4. Otherwise,

    1. Let `referenceOrAlias` be the entry from `manifest.fallbackPool` referenced by `ident`

    2. Return it immediately, whether it's defined or not

### FIND_PNP_MANIFEST

```
FIND_PNP_MANIFEST(url)
```

Finding the right PnP manifest to use for a resolution isn't always trivial. There are two main options:

- Assume that there is a single PnP manifest covering the whole project. This is the most common case, as even when referencing third-party projects (for example via the [`portal:` protocol](/features/protocols#whats-the-difference-between-link-and-portal)) their dependency trees are stored in the same manifest as the main project.

  To do that, call [`FIND_CLOSEST_PNP_MANIFEST`](#find_closest_pnp_manifest)`(require.main.filename)` once at the start of the process, cache its result, and return it for each call to [`FIND_PNP_MANIFEST`](#find_pnp_manifest) (if you're running in Node.js, you can even use `require.resolve('pnpapi')` which will do this work for you).

- Try to operate within a multi-project world. **This is rarely required**. We support it inside the Node.js PnP loader, but only because of "project generator" tools like `create-react-app` which are run via `yarn create react-app` and require two different projects (the generator one `and` the generated one) to cooperate within the same Node.js process.

  Supporting this use case is difficult, as it requires a bookkeeping mechanism to track the manifests used to access modules, reusing them as much as possible and only looking for a new one when the chain breaks.

### FIND_CLOSEST_PNP_MANIFEST

```
FIND_CLOSEST_PNP_MANIFEST(url)
```

1. Let `manifest` be **null**

2. Let `directoryPath` be the directory for `url`

3. Let `pnpPath` be `directoryPath` concatenated with `/.pnp.cjs`

4. If `pnpPath` exists on the filesystem, then

    1. Let `pnpDataPath` be `directoryPath` concatenated with `/.pnp.data.json`

    2. Set `manifest` to `JSON.parse(readFile(pnpDataPath))`

    3. Set `manifest.dirPath` to `directoryPath`

    4. Return `manifest`

5. Otherwise, if `directoryPath` is `/`, then

    1. Return **null**

6. Otherwise,

    1. Return [`FIND_PNP_MANIFEST`](#find_pnp_manifest)`(directoryPath)`

### PARSE_BARE_IDENTIFIER

```
PARSE_BARE_IDENTIFIER(specifier)
```

1. If `specifier` starts with "@", then

    1. If `specifier` doesn't contain a "/" separator, then

        1. Throw an error

    2. Otherwise,

        1. Set `ident` to the substring of `specifier` until the second "/" separator or the end of string, whatever happens first

2. Otherwise,

    1. Set `ident` to the substring of `specifier` until the first "/" separator or the end of string, whatever happens first

3. Set `modulePath` to the substring of `specifier` starting from `ident.length`

4. Return `{ident, modulePath}`
