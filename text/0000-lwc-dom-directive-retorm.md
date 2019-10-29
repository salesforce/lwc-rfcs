- Start Date: 2019-10-28
- RFC PR: (leave this empty)
- Lightning Web Component Issue: (leave this empty)

# Summary

This RFC describes the formalization of the `lwc:dom` directive in LWC Templates. Specifically, it defines how to lift existing restrictions on `lwc:dom="manual"` and introduces `lwc:dom="synthetic-portal"` as a way to support legacy systems.

# Basic example

This RFC lift some restrictions on the existing implementation and introduces the concept of `synthetic-portal`:

```HTML
<template>

    <h1>Existing feature (for leaf element only):</h1>
    <div lwc:dom="manual"></div>

    <h1>Lifting restrictions on existing feature:</h1>
    <div lwc:dom="manual">
        <p>can have child elements in template to be combined with manually inserted nodes</p>
    </div>

    <h1>Introducing new feature (for leaf element only):</h1>
    <div lwc:dom="synthetic-portal"></div>

</template>
```

# Motivation

There are 3 main motivations for this reform:

1. The existing restriction that only leaf elements can be marked with `lwc:dom="manual"` is limiting. There are valid use-cases that requires a mix of elements generated from a template plus manually inserted elements, one of them is the use of a graph library that can connect elements (visually) where the elements are controlled via the template declarative mechanism, while the edges are manually generated and inserted by a 3rd party library.

2. Legacy systems, specifically flexipages in Aura applications, are in needs to support rendering legacy DOM structures that are not suitable, or were not designed, to work inside a `ShadowRoot` instance. In such cases, it is imperative that from the perspective of those elements, they should behave as inserted in the document. 

3. The formalization of this feature can lead to performance improvements, specifically, the considerations with respect to what elements need to be keyed, and which one can be safely ignored.

# Detailed design

TBD

# Drawbacks

Why should we *not* do this? Please consider:

- `lwc:dom="manual"` is simply a no-op when using native shadow, but `lwc:dom="synthetic-portal"` does not work when using native shadow, that difference could be misleading.
- `lwc:dom="synthetic-portal"` is not getting us closer to use native shadow, in fact it could derail us from ever activating native shadow because features will be built on top of a feature that will never work in native shadow.

# Alternatives

No apparent alternative has been identified.

# Adoption strategy

If we implement this proposal, how will existing Lightning Web Components developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

The restriction of `lwc:dom="manual"` that will be lifted do not require a change in the mental model on how this feature is used. No side-effects were identified when combining declarative and manual elements. The only thing needed is to update the documentation to remove the restriction, and maybe adding an example that showcases the combination of declarative and manual elements.

On the other hand, the `lwc:dom="synthetic-portal"` must be documented, and probably gated.

# Unresolved questions

* Should `lwc:dom="synthetic-portal"` be gated or restricted for very specific cases?
