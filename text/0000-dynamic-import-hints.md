---
title: Dynamic Import Hints and Its Metadata
status: DRAFTED
created_at: 2020-03-16
updated_at: 2020-03-22
rfc: https://github.com/salesforce/lwc-rfcs/pull/27
champion: Aliaksandr Papko (@apapko) | LWC Team
implementation: https://git.soma.salesforce.com/lwc/lwc-api/pull/42 & https://git.soma.salesforce.com/lwc/lwc-platform/pull/361
---

# Hints Syntax and Its Metadata

Table of Content:
* [Summary](#summary)
* [Motivation](#motivation)
* [Detailed Design](#detailed-design)
    * [Grammar](#grammar)
        * [Hint Syntax](#hint-syntax)
        * [Static Semantics](#static-semantics)
        * [Semantics](#semantics)
        * [Examples](#examples)
        * [Separation of Concern](#separation-of-concern)
    * [Reasons](#reasons)
    * [Dynamic Import Metadata](#dynamic-import-metadata)
* [Alternate Syntax Considered](#alternate-syntax-considered)
* [Future work](#future-work)
* [References](#references)

## Summary
This RFC suggests authoring syntax for specifying dynamic import hints and describes its metadata shape.

## Motivation
Using hints allows component authors to specify a condition on which a dynamic import will be necessary. This is a way to describe the conditions required for this import to happen. An application framework can use these hints to optimize the loading of those resources (e.g.: pre-fetch at the time of resolving dependencies)

## Hint Invariants
* Hint key and value are string literals
* Hints do not influence the outcome of the compilation phase. It only produces additional metadata for the application framework to consume.

## Detailed Design

### Grammar
This section defines the grammar to be followed for expressing hints. The grammar establishes a contract between the component and the compiler to facilitate hint recognition by the compiler, gathering the metadata and providing that as part of the compilation result.

#### Hint Syntax
A hint is expressed as a comment inside a dynamic import expression. The hint syntax is either a line comment or a block comment with a *key* and *value* pair. The key and value will be separated by a colon(`:`) and delimited using double quotes(`"`).

```js
/* "<key>": "<value>" */
```
Hint comment e.g.:
```js
/* "@salesforce/client/formFactor": "SMALL" */
/* "MY_CONDITION": "VALID" */
// "MY_OTHER_CONDITION": "NOT_VALID"
```

Dynamic import with a hint e.g.:
```js
import formFactor from '@salesforce/client/formFactor';

export async function loadFoo() {
    if (formFactor === 'SMALL') {
        return import("lightning-foo-mobile" /* "@salesforce/client/formFactor": "SMALL" */);
    } else {
        return import("lightning-foo-desktop" /* "@salesforce/client/formFactor": "LARGE" */);
    }
}
```

#### Static Semantics
* Both key and value have to be string literal
* Is case sensitive
* Key and value have to be delimited with a double quote(`"`) to be explicit about the literal value. Single quote(`'`) is not considered a valid delimiter.
* Spaces before and after the colon(`:`) is optional
* Hint phrase can have leading and trailing spaces

##### Hint Key:
* Cannot contain spaces(to allow usage as object keys)
* Cannot combine multiple string literals using operators(i.e `"KEY1" & "KEY2": "true"` will not qualify as a valid hint).

##### Hint Value:
* Can contain space(s)
* Cannot combine multiple string literals using operators(i.e `"KEY": "LARGE" & "SMALL"` will not qualify as a valid hint).

##### Multiple Hints:
When multiple hint comments are specified for a single dynamic import statement, only the first hint is considered.
Example:
```js
import("c-bar-mobile"
    /* "@salesforce/client/formFactor": "SMALL"*/
    /* "@salesforce/userPermission/ViewSetup": "true"*/
);
```
In the above example, only the `formFactor` hint is considered.

#### Semantics
The [_ModuleSpecifier_](http://www.ecma-international.org/ecma-262/6.0/index.html#sec-imports) of the import statement should be a string literal. The compiler cannot analyze data about a module name that will be discovered or learned at runtime. For example, in the following case no hint metadata will be gathered:

```js
const bar = '/modules/my-module.js';
function foo() {
    import(
        bar
        /* "@salesforce/client/formFactor": "SMALL" */
    ).then(module => {
        success();
    });
}
```
If a hint does not follow the above grammar, metadata gathering for that specific hint will be skipped. However, if there are other hints in the same module that follow the grammar, metadata will be gathered.

#### Examples
Valid
```js
// Single hint
import("c-bar-mobile"/* "@salesforce/client/formFactor": "SMALL"*/);
import("c-bar-mobile"/* "KEY.SUBKEY": "VALUE"*/);
import("c-bar-mobile"/* "KEY/SUBKEY1": "VALUE"*/);
import("c-bar-mobile"/* "KEY": "true"*/);
import("c-bar-mobile"/* "KEY": "1"*/);
```
Invalid
```js
import("c-bar-mobile"/* "KEY SUBKEY": "SMALL"*/);
import("c-bar-mobile"/* @salesforce/client/formFactor = SMALL*/);

// Dynamic module name specifier
const bar = 'module-name';
import(bar/* "@salesforce/client/formFactor": "SMALL" */)
```

#### Separation of Concern
Dynamic imports hint metadata gathering happens during component compilation. An invariant of this process is that the metadata collection will **not** fail due to a bad hint. This could be due to a hint not following specified grammar.

Additional validation and post processing of the gathered metadata is out of the scope of this RFC. That logic will be the responsibility of the application framework that consumes the metadata.

Similarly, static analysis of dynamic import hints can be handled by linting module code. For example, on the salesforce platform, allowed hint keys might be restricted to only `@salesforce` scoped specifiers. The hint values will be validated based on the relevant value for the specifier. An ESLint rule will enforce this restriction via the [eslint-plugin-lwc](https://github.com/salesforce/eslint-plugin-lwc).

### Reasons
The hint syntax in the form of a comment was selected for the following reasons:
* Complies with ‘what you write is what you see’ concept - an originally authored code remains untouched
* Eliminates any additional code transformations at compile time
* Eliminates any additional processing during module resolution at runtime (to strip out or pre-process module name)

### Dynamic Import Metadata
The proposed metadata format will include dynamically imported component name and a hint object, which in-itself contains the original query and a hint value. A hint value will be an array of objects with raw hint query, expected hint condition, and hint resource information.

#### Proposed Shape
```ts
export interface SourceLocation {
    startLine: number;
    startColumn: number;
    endline: number;
    endColumn: number;
    file: string;
}

export interface DynamicImportHint {
    rawValue: string; // Leading and trailing spaces of hint phrase trimmed
    hintKey: string;
    hintValue: string;
    location: SourceLocation;
}

export interface DynamicImportMetadata {
    moduleName: string;
    file: string;
    hints: DynamicImportHint[];
}
```

Example:
```js
{
    dynamicImports: [{
        moduleName: 'lightning-foo',
        file: 'foo.js',
        hints: [{
            rawValue: '"@salesforce/client/formFactor": "SMALL"',
            hintKey: '@salesforce/client/formFactor',
            location: {...},
            hintValue: 'SMALL',
        }]
    }]
}
```

## Alternate Syntax Considered

### Hints as a query attached to the component name
The hint syntax is a parameterized query appended to the component name:
```js
export async function example1() {
    loadedModule = await import("lightning-foo*?*@salesforce/client/formFactor=Small");
}
```

*Pros:*

* Intuitive syntax

*Cons:*

* Requires transform to remove the query
* Using a query string syntax to discover modules to prefetch doesn't align with the way standard import() works. JavaScript VM caches modules by URL (including the query string), this means that lightning-foo lightning-foo?1 and lightning-foo?2 are 3 different JavaScript module instances

### Hints via proprietary LWC function
The hint syntax is expressed via a proprietary lwc function, where the first parameter is a component name and the second parameter is a query object - hintImport(cmpName, queryObj). At compile time, the function will be transformed into a regular import without a string. Note, the hint is consumed by metadata parser only, which is traversed outside of the compilation sequence on the original source. Therefore, it is safe to have a ‘throw-away’ wrapper that will be eliminated at compiler time.  

```js
import formFactor from '@salesforce/client/formFactor';
import permName from '@salesforce/customPermission/PermName';

export async function example1() {
    loadedModule = await hintImport(
        "lightning-foo",
        {
            query: "{0} IS Small AND {1} NOT true",
            fields: [{
                resource: formFactor,
                value: "Small",
            }, {
                resource: permName,
                value: true
            }]
        }
    );
}
```

*Pros:*

* Dedicated syntax
* Full control of the hint shape
* Having to explicitly define a resource reference will allow for ref integrity support if required

*Cons:*

* Requires an additional transform
* Requires knowledge of syntax existence
* Maintenance burden

## Future work
* Support for multiple hints for a dynamic import statement, with usage of operators to combine hints and hint values
* Support for other primitive data types like boolean, number as hint values
* Support for JSON string as hint value to represent an object

## References
* [RFC - Dynamic component creation](https://github.com/salesforce/lwc-rfcs/blob/master/text/0110-dynamic-components.md)
* [RFC - Pivots in LWC](https://salesforce.quip.com/En4UA5EtbKc0) (_restricted access_)
