---
title: Dynamic Components
status: DRAFTED
created_at: 2019-07-01
updated_at: 2019-07-01
pr: https://github.com/salesforce/lwc-rfcs/pull/10
---

# Dynamic Components 

## Summary

This RFC introduces a set of principles and invariants required to preserve a reliable and predictable system while allowing lazy loading of dependencies.

## Back Pointers

* Original PR: https://github.com/salesforce/lwc/pull/1397

## Terminology

**Lazy loading**: loading javascript from the server that is not initially sent to the client.

**Dynamic instantiation**: creation of a new component instance where the underlying component constructor is not known until runtime.

**Code splitting:** Capability to split the code base in multiple bundles, that can be loaded on demand

## Basic example

```html
<template>
    <x-lazy lwc:dynamic={customConstructor}></x-lazy>
</template>
```

```js
import { LightningElement, track } from "lwc";
export default class DynamicCtor extends LightningElement {
    @track customCtor;

    connectedCallback() {
        this.loadCtor();
    }

    async loadCtor() {
        const ctor = await import("c/customConstructor");
        this.customCtor = ctor;
    }
}
```

## Motivation

Before this RFC, LWC does not support dynamic component creation nor allow you to specify at build or design time different component configurations to pivot on during runtime. We purposely made this decision to ensure we have a clean, statically analyzable and predictable behavior on which to build upon. Due to the complexity and dynamic nature of the Lightning Platform and OSS, as we expand LWC to new clouds, applications and contexts, this strategy isn't sufficient. New primitives must be introduced to guarantee and allow for a more flexible and scalable model.

## Proposal

This proposal has two distinct parts (or you can think of them as two separate proposals):

* The usage of the **dynamic import** to lazy load modules on the client (dynamic imports).
* Usage of a component constructor that is unknown until runtime (template directive).


The `import()` refers to the code splitting and the template directive `lwc:dynamic` refers to the usage of a component that is unknown at compile time in the template.

### Mental model

> If we could snapshot, at a given point in time, the metadata of an app and precompute all server side generated components (if any); then the whole application would be a big monolithic and static component tree, on which routes and every other pivot are simply just if/else branches.

Though idealistic it is conceptually sound (and even technically achievable). It allows us to then **decide when we really need those branches** and when to trim them. Each of those branches would be lazy loaded and dynamically created.

### Proposal by example

Let's imagine we need to build a chart component (`lightning-chart`).

A developer can choose between a bunch of types of charts (lightning-chart-histogram, lightning-chart-bar, lightning-chart-pie, ...), but we only want to expose one and branch based on a type property. We could implement it today like something like this:

```html
<template>
  <section>
    <template if:true={isTypePie}>
       <lightning-chart-pie data={data}></lightning-chart-pie>
    </template>
    <template if:true={isTypeBar}>
       <lightning-chart-bar data={data}></lightning-chart-bar>
    </template>
    <template if:true={isTypeHistogram}>
       <lightning-chart-histogram data={data}></lightning-chart-histogram>
    </template>
  </section>
<template>
```

> Note: This is exactly how `lightning-input` is implemented.

Besides the verbosity of this implementation, a couple of things to note:

- All dependencies are hard: **the component explicitly depends on lightning-chart-histogram, lightning-chart-bar, lightning-chart-pie**, hence will be sent as part of the definition (they are part of the dependency tree)

- The chart components will not be reachable (I can't do querySelector) until the component is rendered with the if condition evaluated to true

When a developer uses this component it will look something like this in the dom:

```html
<lightning-chart type="pie">
  #shadow-root
    <section>
      <lightning-chart-pie>
         #shadow-root
            <!-- fancy dom of lightning-chart-pie goes here -->
      </lightning-chart-pie>
    </section>
</lightning-chart>
```

Now let's imagine we have a lot of charts, like thirty. The template will become really verbose if we have to do all this conditional logic.

```html
<template>
  <lwc-chart-impl lwc:dynamic={dynamicCtor} data={data}></lwc-chart-impl>
</template>

<script>
import chartPie from "lightning-chart-pie";
import chartBar from "lightning-chart-bar";
import chartHistogram from "lightning-chart-histogram";
...

function getCtorByType(chartType) {
  switch(chartType) {
    case "pie": return chartPie;
    ...
  }
}

export default class Chart extends LightningElement {
  @api type;
  @track dynamicCtor;

  connectedCallback() {
    this.dynamicCtor = getCtorByType(this.type);
  }
}
</script>
```

In this implementation we have introduced the directive `lwc:dynamic` which tells LWC that the constructor will not be known until runtime.

What is important is that both codes are **semantically equivalent**

- They have the **same dependencies** (template declared dependencies become static imports anyway).

- You can't reach them until the constructor is set (same as the if statement above).

> Note that the directive is just waiting for a constructor to be set, **how do we get that constructor is orthogonal to the use of this directive**.

Ok so the amazing properties of the examples above are they are **fully statically analyzable and predictable**: We will **never have to do a request to the server** to fetch a particular implementation (this will be an invariant later on) because they are part of the dependency tree (static imports).

The problem is though that maybe some of this dependencies are really heavy. Let's go to a more concrete example:

```html
<template>
  <lwc-chatter-impl lwc:dynamic={dynamicCtor} data={data}></lwc-chatter-impl>
</template>

<script>
import chatterDesktop from "lwc-chatter-desktop";
import chatterPhone from "lwc-chatter-phone";
import isPhone from "@salesforce/api/isPhone";

export default class Chatter extends LightningElement {
  @track dynamicCtor;

  connectedCallback() {
    this.dynamicCtor = isPhone ? chatterPhone : chatterDesktop;
  }
}
</script>
```

So `chatter-mobile` and `chatter-desktop` are **two fundamentally different implementations** and they have huge subtrees, let's say +50kb compress and minified each. Now because of being **static dependencies** we will bring both branches of the tree, which is undesirable for perf.

For this use case we want to be able to “cut the tree” (soft-dependencies), and be able to only bring the `chatter-phone` component tree once we are mobile and the `chatter-desktop` implementation once we are in desktop. In other words we want to make the dependencies “soft” and load them after the fact.

In order to do that we will use [dynamic imports](https://github.com/tc39/proposal-dynamic-import) which give us statically analyzability (although the dependency is soft, we still know that exists) at the same time that allows us to abstract the loading mechanism and the module evaluation.

```html
<template>
  <lwc-chatter-impl lwc:dynamic={dynamicCtor} data={data}></lwc-chatter-impl>
</template>

<script>
export default class Chatter extends LightningElement {
  @track dynamicCtor;

  async connectedCallback() {
    let Ctor;
    if (isPhone) {
      Ctor = await import("lwc-chatter-mobile");
    } else {
      Ctor = await import("lwc-chatter-desktop");
    }

    this.dynamicCtor = Ctor;
  }
}
</script>
```

By doing **dynamic imports** we are making the dependencies **soft** (not part of the dependency tree), meaning that will be loaded at runtime (a network call may be made). That being said, by forcing some invariants (described below) in the way we use those dynamic imports we could gather a lot of metadata at compile time that will allows us to optimize **when** those resource can be prefetched (see Annex for pivots proposal).

### Guiding Principles

We want to avoid ad-hoc solutions and proprietary non-standard ways to define dynamic component loading. We want to build an API that is future proof, whenever more things land into the spec for modules.

### General Invariants

Note that the invariants defined below are not necessarily constraints on LWC in general but rather explicitly defined for the Lightning Platform use cases:

- A dynamic import must always be statically verifiable.
    - A dependency should never be discovered or learned of for the first time at runtime.
    - Note that the ability to analyze is orthogonal to when a component is loaded.
- It must not break the reactivity model or the general framework lifecycle.
    - Once the component is loaded it should function within the lifecycle of the parent component.
    - No special handling in user-land is needed.
- It must work regardless of the algorithm, even if nothing is preloaded at all.
    - The decision for when a dynamic dependency is loaded is orthogonal to its definition.
    - We might have a preload mechanism based on MRU or on a specific context (ex.mobile) but the component should work regardless of the algorithm even if nothing is preloaded at all.

You will notice that this follows the same pattern as what we have done for `@wire`: Bring the ability to understand and resolve dependencies at compile, build and design time.

### API

Below is the API and its ergonomics for dynamically loading a module. It has very similar mental model to the HTML `is` attribute proposal (which we can't use since Safari will not implement it in the foreseeable future): An arbitrary tag name on which you can assign any constructor.

```json5
{
    "dynamicComponent": {
        "loader": "@salesforce/loader",
        "strict": true // only allow string identifiers
    }
}
```

```html
<!-- original HTML source -->
<template>
  <lazy-component lwc:dynamic={lazyConstructor} value={value} other="test"></lazy-component>
</template>


<!-- original JS source -->
<script>
import { LightningElement, track } from "lwc";
export default class LazyTest extends LightningElement {
    @api value = "yay!";
    @track lazyConstructor;

    async connectedCallback() {
      const { default: Ctor } = await import("lightning/concreteComponent");
      this.lazyConstructor = Ctor;
    }
}
</script>

<!-- compiled code -->
<script>
  import { loader } from "@salesforce/loader";
  // this can't be imported in user-land

  function tmpl($api, $cmp) {
     return [$cmp.lazyConstructor != null // falsy for null or undefined
       $api.c("lazy-component", $cmp.lazyConstructor)
       : null
    }];
  }

  export default class LazyTest {
     async connectedCallback() {
        const { default: Ctor } = await loader("lightning/concreteComponent");
        this.lazyConstructor = Ctor;
     }
     render () {
       return tmpl;
     }
  }
</script>
```

As you can see in bold from the implementation details above we have transformed the *lwc:lazy* directive into a condition, and we have abstracted the dynamic import mechanism into a high order loader call provisioned by the application container (this will be configurable).

#### API invariants

* Dynamic imports can't be a top level await - It must be on a block statement.
* You may only use your own namespace for the tag name of the lazy component.
* Arguments of the dynamic import function must be statically verifiable (requirement for offline)
    * We only support string literals
* The attribute bindings must be explicit.
    * No dynamic bag will be allowed (ex. *...props* 'a la React')

> Note: The compiler options will allow you to restrict and or use the loader as decribed above.

```js
// This will work
export async function example2() {
  const m = await loadModule("lightning/rtlFoo?pivots=rtl");
}

// Example of incorrect usage: non discoverable module
export async function example3() {
  const moduleName = "lightning/" + isPhone ? "phone" : "desktop";

  // This will throw since we can't statically verify the module name
  const m = await loadModule(moduleName);
}
```

#### Metadata

The compiler add to its metadata all of the parsed dynamic imports, its source value and the hints (queryString parameters), here is an example of the output:

```json
{
  "dynamicImports": [{
    "moduleName": "lightning-mobile-foo",
  }],
  "unknownDynamicImports": false
}
```

## Drawbacks

The biggest drawback of this feature is the historic abuse of dynamic component creation in the Lightning Platform. We will add guardrails to prevent developers from shooting themselves in the foot.

## Alternatives

We considered several non-standard alternatives but we came to the conclusion that they would put us on the wrong path in the long term.

# Adoption strategy

This feature will initially be implemented in OSS and will be rolled out to the Salesforce Lightning Platform depending on usage and feedback.

# How we teach this

Dynamic imports is a standard feature (in Stage 4). Our documentation around the `lwc:dynamic` directive should provide a comprehensive list of known anti-patterns along with details around when this feature should be used.

# Unresolved questions

- Naming for the directive can be changed.
- An equivalent fully declarative lazy loading.
