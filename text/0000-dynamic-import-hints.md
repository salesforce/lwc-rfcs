---
title: Dynamic Import Hints and Its Metadata
status: ACTIVE

created_at: 2020-03-16
updated_at: 2020-03-16
---

# Hints Authoring Syntax and Its Metadata

## Summary
This RFC suggests authoring syntax for specifying dynamic import hints and describes its metadata shape.

## Motivation
The Hints in dynamic components allows component authors to declare the hint context by parameterizing dynamic imports. The values from the hints can be used to optimize dependency resolution and pre-fetching.

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

Dynamic Import with Hint ex:
```js
export async function example1() {
    loadedModule = await import("lightning-foo"/* @salesforce/client/formFactor=Small */);
}
```

### Reasons
The hint syntax in a form of a comment was selected for the following reasons:
* complies with ‘what you write is what you see’ concept - an originally authored code remains untouched
* eliminates any additional code transformations at compiler time
* eliminates any additional processing during module resolution at runtime (to strip out or pre-process module name)

### Hint Invariants
* hint values must be globally available and resolvable on the server side 
* hint values must be statically verifiable
* hint values must be finite and enumerable (ex: no Math.random())

### Dynamic Import Metadata
The proposed metadata format will include a dynamically imported component name and a hint object, which in-itself contains the original query and a hint value. A hint value will be an array of objects with raw hint query, expected hint condition, and hint resource information.

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