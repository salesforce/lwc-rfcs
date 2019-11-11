---
title: Module resolution
status: DRAFTED
created_at: 2019-11-08
updated_at: 2019-11-08
pr: https://github.com/salesforce/lwc/pull/1602
---

# Module Resolution

This RFC formalizes the module resolution for LWC. It defines the way that we can resolve modules in all host environments (node, browser, rollup, webpack) and in a multi-author multi-version way that is compliant with the latest mental models and specificatons in the different platforms.

## Motivation

As the LWC ecosystem organically grows both internally at Salesforce and externally as OSS, we want to provide and encourage a way to share web components, npm packages and other artifacts, regardless of the host environment, technologies or tools used.

## Basic example

Let's look at a very simple scenario: We have a repository of LWC components we want to open source. We want to use other 3party packages as the foundation for my components, in this case `@ui/components@2.0.0` and `fancy-components@1.0.0`. Its important to note that the `fancy-components` package is using internally `@ui/components@1.0.0`, which is a different version of the package I'm using.

Here is how the structure of the repo looks like  you can take a look at [ a real implementation implementation and test here](https://github.com/salesforce/lwc/pull/1602/files#diff-5cfc908f1b15979cf59306304092ddb5):

```markup
rootDir
├── src/
│   └── modules/
│       └── root/
│           └── foo/
│               └── foo.js ("root/foo")
└── node_modules/
    ├── @ui/
    │   └── components/
    │       ├── package.json [ui-components@2.0.0]
    │       └── src/
    │           └── modules/
    │               └── ui/
    │                   └── button/
    │                       └── button.js ("ui/button")
    └── fancy-components/
        ├── package.json     [fancy-components@1.0.0]
        ├── src/
        │   └── modules/
        │       └── fancy/
        │           └── bar/
        │               └── bar.js ("fancy/bar")
        └── node_modules/
            └── @ui/
                └── components/
                    ├── package.json [ui-components@v1.0.0]
                    └── src/
                        └── modules/
                            └── ui/
                                └── button/
                                    └── button.js ("ui/button")
```

When importing those components from a given module here is what we would expect:

```js
// importer: root/foo/foo.js
import button from "ui/button"; // should resolve against ui/button@2.0.0
```

```js
// importer: fancy/bar/bar.js
import button from "ui/button"; // should resolve against ui/button@1.0.0
```

This means that from a developers perspective you just import the module you need.

## State of the art

An important topic to understand first is how module resolution works today in different technologies and host environments.

JavaScript did not have built-in support for modules, but the community has created impressive work-arounds over the years. The two most important (and unfortunately incompatible) standards are: CommonJS (used fundamentally by Node - with some extra semantics) and AMD (Asynchronous Module Definition), used by RequireJs and a lot of other bundlers.

Note that there are other non-standard formats, loaders and tools like UMD, SystemJS, IIFE, etc, which are interesting for you know and understand but are out of the scope for this proposal.

### Node.js resolution

If you ever used Node, you know that in order to import a module you need to use the `require()` method, and if you want to export something to some other module or file you must use `module.exports`. That is know nas [CommonJS format](https://en.wikipedia.org/wiki/CommonJS). Now if you want to import a 3party package how does that works? Well Node decided to implement its own resolution by looking at the content of a the node_modules folder based on some dependencies defined on `package.json`. You can look at the full algorithm [here](https://nodejs.org/docs/latest/api/modules.html#modules_all_together).

All of this being said after years of very hard work, Node is finnaly addopting ES6 modules (which we will see in the next section) since LTS v12 and they are hoping to [fully transition to it soon](https://twitter.com/MylesBorins/status/1189618753065144322)

Just to put all of this in a simple example:

```js
// This will be resolved by looking a node_modules/ folder "foo-bar" 
// recursively upwards, and the check the package.json to find the entry point.
const x = require('foo-bar') // This is using CommonJS syntax

// This indicates that we are exporting a default function 
// to whoever is requesting this module
module.exports = function () { return x() }; // This is using CommonJS syntax

```

### ES6 Modules and browser resolution

At the end of July 2014, TC39 (Javascript standarization comitee) had another meeting, during which the last details of the ECMAScript 6 (ES6) module syntax were finalized. [Here is the specification text](https://www.ecma-international.org/ecma-262/6.0/#sec-imports).

The fundamental goal for ECMAScript 6 modules was to create a standard format that all users from the previous formats were comfortable and happy about, and lacked none of the caveats and issues - and since was part of the standard a lot of things could be improved like

1. The structure which can be statically analyzed (for static checking, optimizations, etc.)
2. Better support for cyclic dependencies

Here is how ES6 modules look like:

```js
import { foo } from "bar";
import buzz from "buzz";

export function namedExport() { return foo(); }

// re-export a module
export { buzz };
```

Along with that there is another proposal (Stage 4) for [dynamic imports](https://tc39.es/proposal-dynamic-import/) to allow modules to be loaded lazily.

As a final note, remember that the resolution of ES6 can be done in multiple ways.
  - Browsers have implemented a URL schema base way of resolving modules with [import maps](https://wicg.github.io/import-maps/) (We will be basing our implementation on this)
  - Node.js as stated before has integrated ES6 modules along side the old one (see link above).
  - Tools like Webpack or RollupJs allow you to configure and customize the way we load bundle and resolve modules both run build-time and run-time.

## Why we need our own module resolution

Web Components are composed of multiple pieces (CSS, HTML and JS) and there is no canonical or standard way to bundle them natively yet. Although there are several proposals to standarize the way by which each of these pieces can interoperate and be integrated in the future (see [CSS modules](https://github.com/w3c/webcomponents/issues/759), [HTML templates](https://github.com/w3c/webcomponents/issues/839)), we must build a future-proof, standards compliant way to do so while we help the standardization pieces to land.

Even when all these pieces become available (and without going into too much detail here), it is likely that we will have to do some preprocessing or bundling anyway, which mean that we will have to take over the resolution one way or another - whether at build time to collapse multiple pieces in on, or at runtime by using import-maps with some extra semantics on top. Moreover remember that a give package might contain many components (1:N) relationship, so we'll always have to have a configuration on where to find those components.

To summarize: We will always need an abstraction layer which dictates where a particular module specifier must be resolved from (adapted to different host environments).

## Detailed design

In order to resolve modules in LWC that work on different host environments (Salesforce platform, Node.js, npm, git, et all) we will be relying on our own LWC configuration file (similar approach as import-maps, babel, webpack rollup or package.json).

### LWC configuration

A configuration file will be located at the root of the project, optionally at the component namespace level and in the future (this is yet to be spec out) at the module bundle level, where specificity will be resolved from inner to outer (merging upwards - similar to node_modules).

At the package root level there are two ways on which you can add a configuration file: Creating a `lwc.config.json` or adding an `lwc` key to package.json (this is mostly for simplicity and easy of use - specially when publishing a package). Here are some examples:

```json5
// package.json
{
    "name: "foo",
    "version": "bar",
    "lwc": {
        "modules": [
            /* list with modules records goes here */
        ]
    }
}
```

```json5
// lwc.config.json
{
    "modules:" [
        /* list with modules records goes here */
    ]
}
```

Its important that **any plugin, tool or project honor this configuration**, otherwise the resolution will not work correctly in most environments.

Moreover, note that this type of configuration is not something new, many tools use an equivalent configuration (babel, lerna, rollup, webpack, ...)

### Module resolution configuration

The `modules` key on the configuration will hold an Array of `ModuleRecords`. A module record is an object with a specific shape depending on the type of module record it holds. The type will depend on the mode of resolution, we will be defining three types of `ModuleRecords` for now:
  - AliasModuleRecord: Define an specifier that points to a specific module.
  - DirectoryModuleRecord: Defines a folder where many namespaced modules can be defined
  - NpmModuleRecord: Defines a set of packages that will hold more modules to be resolved.

### Why Objects instead of raw strings?

In previous draft versions, `module` entries used to be just *strings* which had slightly better ergonomics. However, raw strings present a lot of ambiguities, let's look at one example:

```json
{
    "modules": [
        "just-string"
    ]
}
```

On the example below, it will be impossible to know if `just-string` is the name of an npm package, or a name of a folder that contains components, hence creationg resolution ambiguity.

We considered other alternatives such as forcing specific prefix like `npm:package-name` or require a final slash (ex. `folder-name/`) for folders, but we believed that those were more cumbersome to learn and future-proof, so standarizing on an consistent object shape as ModuleRecord was a preferable and superior approach.

### Types of ModuleRecord

#### AliasModuleRecord

This type of resolution maps a given module specifier to a particular path within the root of that module.

```json
{
    "modules": [
       {
            "name": "ui/button",
            "path": "src/modules/ui/button/button.js"
        }        
    ]
}
```

`AliasModuleRecord` must contain `name` and a `path` keys.

#### DirectoryModuleRecord

This type of resolution allows to specify a folder that contain LWC modules. The structure of the folder its very specific (LWC has a very opinionated module structure). It must contain a namespace folder, then a folder per module bundle (moduleName) and then a moduleEntry file that matches the name of the bundle (the entry must be of type .css, .html, .js or .ts)

::: note
  The folder structure is something that the core LWC teams wants to enforce as a convention. There is no technical limitation on the compiler or other tools to resolve different folder structures. However we do want to provide and enforce as much as possible a canonical way so we can in the future keep iterating, improving and expanding the ergonomics of it.
:::


```json
{
    "modules": [
       { "dir" : "src/modules" }
    ]
}
```

Here is an example of a valid module resolution given the previous DirectoryModuleRecord
```
src
└── modules/ 
    └── ui/ (namespace)
        └── button/ (moduleName)
            └── button.js (moduleEntry)
```

`DirectoryModuleRecord` must contain `dir` key.

#### NpmModuleRecord

This type of resolution tells the resolver to find an npm package with that name and resolve the modules it might have inside. This process can be recursive (see example at the top).

```json
{
    "modules": [
       { "npm" : "@ui/components" }     
    ]
}
```

`NpmModuleRecord` must contain `npm` key.

## Algorithm

Here is the algoritm for module resolution (the instructions are not in a rigurous specification format for brevity and time)

```
  Let `rootDir` be the initial path for the application, `initModules` 
  the list of ModuleRecords to be resolved and ModuleRecordEntryList 
  an empty array.
  
  1. Load the LWC configuration file in the rootDir
    1.1 Search for `lwc.config.js`, if it exist read it.
    1.2 If lwc.config.js is not found search in package.json
    (If both are found the configuration file has precedence).

  2. Merge all the ModuleRecords found in the config with 
  the list of initModules. If an alias with the same name 
  exist pick initModules over the ones resolved in the config.

  3. For each ModuleRecord:

    3.1  Validate that its a valid ModuleRecord type, skip it otherwise.

    3.2 Resolve ModuleRecord entry:

        3.2.1 If is an AliasModuleRecord validate the path and add create 
        a ModuleRecordEntry with: 
          - `specifier` as the `name` value.
          - `scope` with the closest configuration path.
          - `entry` with the `path` value.

        3.2.2 If is a DirModuleRecord validate the path and find of the 
        modules that match the structure:
        [namespace]/[componentName]/[componentName.{html|css|js|ts}]

        3.2.3 If is a NpmModuleRecord, validate that the npm package is 
        installed and can be resolved (by using Node `node_modules` 
        algo) and `GoTo 1` recursively merging all the ModuleRecordEntries.

    3.3 Return all the ModuleRecordEntries collected.
```

#### Future ModuleRecord types

We will not discard adding another types of module resolutions, like for example, find modules directly from git or even from a url.
This schema allows us to add 

## Scoping

Its an invariant of the module resolution system to allow for multiple versions of multiple components within a particular application configuration. As the modules are resolved, we keep track of the context on which those modules appear, basically who is the importer (and its path). You can check again the [basic example](#basic_example) on the top.

This concept of scope matches the [current specification of import maps](https://github.com/WICG/import-maps#multiple-versions-of-the-same-module) which will allow us to create the equivalent resolution for the browser.

## Resolving modules at build time.

This algorithm can be used and integrated with tools like Webpack or rollup, in fact we have created both the `@lwc/module-resolver` and the `@lwc/rollup-plugin` packages that we officially support. Other projects in the ecosystem such as `create-lwc-app` also leverage those primitives.

## Alternatives

We discussed different module resolution alteratives previously, but to summarize:

  - We need a custom module resolution to: 
    - Support the 1:N relationship on packages
    - Have different build steps
    - Align with standards in different host environments
  - Our module resolution strategy is based on 3 types of module records
    - Other alternative semantics can be discussed as part of this RFC as long as it fullfill the invariants described in this doc.

## Adoption strategy

We need to coordinate all projects that are leveraging the old module resolution. However given that we haven't fully exposed the module resolution API the surface is relatively small.

All of the tools on the ecosystem should follow the algorithm described here to be fully compliant and work well in different integrations.

## How we teach this

Its important to understand the previous state of the art on how different platforms implement module resolution and the current work being done in specification land to might shape the transformation of this rules.

### Nomenclature

- `ModuleRecord`: An entry module to be resolved.
- `[Type]ModuleRecord`: And ModuleRecord with specific schema and semantics.
- `ResolvedModuleRecord` The result of resolving a ModuleRecord.
- Scope: The closest importer of a particular ModuleRecord.
- Specifier: Name of the module that can be used to reference a module.
- Entry: The entry point for a particular module bundle.

### Ecosystem

Developers must learn about `lwc.config.json` for their package or component to be distributed discoverable and used by 3parties (via npm or other forms of registry).

Note that in the Salesforce platform developers already familiar with our metadata file which hold information about the component, altough this will be irrelevant for them.

# Unresolved questions

This RFC should cover all the use cases for module resolutions.
