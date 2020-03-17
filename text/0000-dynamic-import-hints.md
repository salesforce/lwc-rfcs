---
title: Dynamic Import Hints and Its Metadata
status: DRAFTED
created_at: 2020-03-16
updated_at: 2020-03-16
rfc: https://github.com/salesforce/lwc-rfcs/pull/27
champion: Aliaksandr Papko (@apapko) | LWC Team
implementation: 228
---

# Hints Authoring Syntax and Its Metadata

## Summary
This RFC suggests authoring syntax for specifying dynamic import hints and describes its metadata shape.

## Motivation
Using hints allow component authors to declare the component preload context by parameterizing dynamic imports. The values from the hints can be used to optimize dependency resolution and pre-fetching.

## Hint Invariants
* hint values must be globally available and resolvable on the server side 
* hint values must be statically verifiable
* hint values must be finite and enumerable (ex: no Math.random())

## Detailed design

### Hints Syntax
Hints are expressed as a comment inside a dynamic import expression. 
The hint syntax is a block string with a *key *as a resource and a *value* as a resolved resource value

```js
/* <key>=<value> */
```
Hint comment ex:
```js
/* @salesforce/client/formFactor=Small */
```

Dynamic import with a hint ex:
```js
export async function example1() {
    loadedModule = await import("lightning-foo"/* @salesforce/client/formFactor=Small */);
}
```

### Reasons
The hint syntax in a form of a comment was selected for the following reasons:
* complies with ‘what you write is what you see’ concept - an originally authored code remains untouched
* eliminates any additional code transformations at compile time
* eliminates any additional processing during module resolution at runtime (to strip out or pre-process module name)


### Dynamic Import Metadata
The proposed metadata format will include dynamically imported component name and a hint object, which in-itself contains the original query and a hint value. A hint value will be an array of objects with raw hint query, expected hint condition, and hint resource information.

Proposed Shape
```js
{
  dynamicImports: [{
    moduleName: "lightning-foo",
    file: "foo.js"
    hints: [
                rawValue: "/* @salesforce/client/formFactor=Small */"
                resource: {
                    id: "@salesforce/client/formFactor",
                    namespacedId: undefined, // scoped modules are namespaced at compile time
                    type: "client",
                    file: "foo.js",
                    location: {...}
                },
                value: "Small", 
    ]
  }]
}
```

## Other Options Considered

### Hints as a query attached to the component name
The hint syntax is a parameterized query appended to the component name:
```js
export async function example1() {
    loadedModule = await import("lightning-foo*?*@salesforce/client/formFactor=Small");
}
```

*Pros:*

* intuitive syntax 

*Cons:*

* requires transform to remove the query 
* using a query string syntax to discover modules to prefetch doesn't align with the way standard import() works. JavaScript VM caches modules by URL (including the query string), this means that lightning-foo lightning-foo?1 and lightning-foo?2 are 3 different JavaScript module instances. 

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

* dedicated syntax
* full control of the hint shape
* having to explicitly define a resource reference will allow for ref integrity support if required

*Cons:*

* requires an additional transform
* requires knowledge of syntax existence
* maintanance burden

