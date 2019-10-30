- Start Date: 2019-10-28
- RFC PR: (leave this empty)
- Lightning Web Component Issue: (leave this empty)

# Summary

This RFC describes the formalization of the `lwc:dom` directive in LWC Templates. Specifically, it defines how to lift existing restrictions on `lwc:dom="manual"` and introduces `lwc:dom="synthetic-portal"` as a way to support legacy systems.

# Quick recap on the `lwc:dom="manual"` directive

This directive, which is LWC specific (hence the prefix), is intended to help with the following issues:

* facilitate the transition between synthetic shadow and native shadow.
* help to keep the dom updates somewhat deterministic because the in-memory representation of the dom (via a virtual dom tree structure) will always represent the current dom, except for the cases in which it a signal is used via the `lwc:dom="manual"` directive.
* style manually inserted elements with the shadowed styles defined by the host (which in synthetic dom will require magic attributes to be applied to the element in order to apply the scoped style rules).

These are the main three reasons why this directive was introduced. The determinism is enforce by throwing an error if a DOM node is inserted or removed from an element produced by a LWC Template, this is a dev-mode only enforcement.

Additionally, as a precaution, we decided to support `lwc:dom="manual"` in leaf elements only, and do not support it at all in custom elements or slot elements, and reevaluate if we could lift part of that restriction in the future.

# Motivation

There are 3 main motivations for this reform:

1. The existing restriction that only leaf elements can be marked with `lwc:dom="manual"` is limiting. There are valid use-cases that requires a mix of elements generated from a template plus manually inserted elements when the template syntax falls short.

2. Legacy systems, specifically flexipages in Aura applications, are in need to support rendering legacy DOM structures that are not suitable, or were not designed, to work inside a `ShadowRoot` instance. In such cases, it is imperative that from the perspective of those elements, they should behave as inserted in the document.

3. The formalization of this feature can lead to performance improvements, specifically, the considerations with respect to what elements need to be keyed, and which one can be safely ignored.

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

# Why do we need to lift restrictions for `lwc:dom="manual"`?

One of uses cases is the use of a graph library that can connect elements (visually) where the nodes in the graph are controlled via the template declarative mechanism, while the edges are manually generated and inserted by a 3rd party library.

__Note:__ the restriction about custom elements stays the same, this directive cannot be used in `<slot>` elements or custom elements.

# Why do we need `lwc:dom="synthetic-portal"`?

It turns out that in Aura platform, we have many Aura components that were designed to work inside an Aura flexipage, which means they can make assumptions about the page, about the structure of the page. Many of those assumptions will not work if they are ever used inside a LWC flexipage (we call this RRH - raptorized record home). This happens to be a very important piece of the salesforce platform. Additionally, any library or legacy system that assumes a regular dom structure without shadow dom boundaries will suffer similar consequences.

Until today, we have ignored this problem, but now that LWC Flexipages are here, we need to have a consistent solution that works seamless. Until now, the LWC Flexipage will insert manual elements inside the container without specifying `lwc:dom="manual"`, due to the implications of using that, which makes the inserted node to recognize the shadow boundary, which is not the intention. This works somehow, but introduces a bigger problem, the problem that LWC doesn't know what's going on there, and it is hard for LWC Team to predict the impact of changes in LWC and Synthetic Shadow. It also makes LWC Flexibpage an offender in dev-mode, reporting invalid insertion of nodes on an element generated from a template.

We are proposing to solve this problem by providing the correct hint via the template directive `lwc:dom="synthetic-portal"`, which could be used by LWC to support the interactions with the portal. At the same time, it can be used by Synthetic Shadow polyfill to implement the proper semantics for the portal.

# `lwc:dom="manual"` vs `lwc:dom="synthetic-portal"`

When using `lwc:dom="manual"`:

* the scoped style will be applied to any manually inserted element (in the micro-task via MO)
* the manually inserted nodes will respect the shadow dom semantics (they belong to the shadowRoot, just like template generated elements)
* works the same in native (the only difference is that the scope styles are applied immediately instead of having to way for the micro-task)
* elements can't be found via global query selectors (because they belong to a shadowRoot)

When using `lwc:dom="synthetic-portal"`:

* no scoped style will be applied to any manually inserted element (they only get global styles, no MO is installed)
* the manually inserted nodes will act like elements slotted into the outer most custom element (they belong to the `document.body`)
* does not work in native at all (we should decide if we should throw or just let it be)
* elements can be found via global query selectors (because they belong to `document.body`)

# Detailed design

## Detailed design about lifting restriction for leaf elements only for `lwc:dom="manual"` directive

TBD

## Detailed design of `lwc:dom="synthetic-shadow"` directive

TBD

## Performance optimizations

Considering that `lwc:dom` is only useful when running in synthetic shadow, it is a no-op on native shadow, which means no perf penalty. However, when in synthetic shadow, we must take into consideration few things:

* The amount of nodes inserted manually is non-deterministic, it might be few, it might be a big portion of the app.
* Nodes inserted manually are usually hard to track and stamp to speed up the resolution of whether or not it qualifies as an element inside a portal or not.
* Mutation Observer is slow, and async, making it almost impossible for the cases where a portal gets a big subtree manually inserted.
* Traversing up when we have no clue whether the element was inserted in a portal or not is also expensive, and requires crossing the bridge to `C` layer when invoking native APIs at every step of the way when traversing up, which makes this even more complicated.

It seems that a `Element.matches` (https://developer.mozilla.org/en-US/docs/Web/API/Element/matches) could provide a boost on the resolution time when analyzing an element. For example, if portal elements are decorated with a _magic attributes_, and host elements are decorated with another _magic attribute_, we could check, at any given time, if a element is within a portal directly, or indirectly, which is sufficient information to make a determination of what to do at any given time.

The cost of adding the magic attributes is almost depreciable since it happens once before inserting the custom elements and the portal elements into the DOM. No MO is needed for portals anymore because we can make determinations at any given time by matching the element, which crosses the bridge to `C` layer once and it is sufficiently optimized and cached in modern browsers.

Example:

For the following markup:

```
<body>
    <div>
        <x-foo lwc:host="synthetic-shadow">
            #shadowRoot
                <div lwc:dom="manual"> (portal)
                    <p class="first"> (manually inserted element)
                    <x-bar lwc:host="synthetic-shadow">
                        #shadowRoot
                            <p class="second">
```

At any given time, you can check if `p` qualifies as shadowed or not by running simple query matches:

* If `<p class="first">` matches query `"[lwc:dom] *"` it means it is inside a portal.
* If `<p class="first">` matches query `"*:not([lwc:host]) *"` it means it is not inside a custom element.

Note: similar performance optimization can be applied to many other areas of the synthetic shadow patches.

# Drawbacks

Why should we *not* do this? Please consider:

- `lwc:dom="manual"` is simply a no-op when using native shadow, but `lwc:dom="synthetic-portal"` does not work when using native shadow, that difference could be misleading.
- `lwc:dom="synthetic-portal"` is not getting us closer to use native shadow, in fact it could derail us from ever activating native shadow because features will be built on top of a feature that will never work in native shadow.

# Alternatives

No apparent alternative has been identified.

# Adoption strategy

The changes described in this RFC are backward compatible.

# How we teach this

The restriction of `lwc:dom="manual"` that will be lifted do not require a change in the mental model on how this feature is used. No side-effects were identified when combining declarative and manual elements. The only thing needed is to update the documentation to remove the restriction, and maybe adding an example that showcases the combination of declarative and manual elements.

On the other hand, the `lwc:dom="synthetic-portal"` must be documented, and probably gated.

# Unresolved questions

* Should `lwc:dom="synthetic-portal"` be gated or restricted for very specific cases?
* Should we include a generated global key for `lwc:host="generated-global-key-value"` in the queries, or using `[lwc:host]` is sufficient?
