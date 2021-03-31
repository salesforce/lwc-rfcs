---
title: Dynamic Imports
status: IMPLEMENTED
created_at: 2019-07-01
updated_at: 2021-02-01
pr: https://github.com/salesforce/lwc-rfcs/pull/42
---

# Dynamic Imports

## Summary

This RFC introduces a set of principles and invariants required to preserve a reliable and predictable system while allowing lazy loading of dependencies.

## Back Pointers

- Original PR: https://github.com/salesforce/lwc/pull/1397

## Terminology

**Lazy loading**: load JavaScript from the server that isn't initially sent to the client.

**Dynamic instantiation**: create a component instance when the underlying component constructor isn't known until run time.

**String-literal dynamic import**: a dynamic import that uses a static string-literal to reference another module. This category of dynamic imports is statically analyzable.

**Variable dynamic import**: a dynamic import that uses JavaScript to create the component string parameter to pass into the `import` function.

**Code splitting:** the capability to split the code base into multiple bundles, which can be loaded on demand.

## Basic example

```html
<template>
  <x-lazy lwc:dynamic="{customCtor}"></x-lazy>
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
    this.customCtor = ctor.default;
  }
}
```

Example in the [lwc-recipes-oss](https://github.com/trailheadapps/lwc-recipes-oss/tree/master/src/modules/recipe/compositionDynamic) repository.

## Motivation

Before this RFC, LWC didn't support dynamic imports or allow you to specify at build or design time different component configurations to pivot on at run time. We purposely made this decision to ensure that we have clean, statically analyzable, and predictable behavior to build on. Due to the complexity and dynamic nature of the Lightning Platform and OSS, as we expand LWC to new clouds, applications, and contexts, this strategy isn't sufficient. New primitives must be introduced to guarantee and allow for a more flexible and scalable model.

## Proposal

This proposal has two distinct parts (or you can think of them as two separate proposals):

- Usage of the **dynamic import** to lazy load modules on the client (dynamic imports).
- Usage of a component constructor that is unknown until runtime (template directive).

The `import()` refers to the code splitting and the template directive `lwc:dynamic` refers to the usage of a component that's unknown at compile time in the template.

### Mental model

> If we could snapshot, at a given point in time, the metadata of an app and precompute all the components generated server-side (if any), then the whole application would be a big monolithic and static component tree. Every route and every pivot is simply an if/else branch on the tree.

Though idealistic, it's conceptually sound (and even technically achievable). It allows us to then **decide when we really need those branches** and when to trim them. Each of those branches would be lazy loaded and dynamically created.

### Proposal by example

Let's imagine we need to build a chart component (`lightning-chart`).

A developer can choose between a bunch of types of charts (`lightning-chart-histogram`, `lightning-chart-bar`, `lightning-chart-pie`, and so on), but we only want to expose one and branch based on a `type` property. We could implement it today like this:

```html
<template>
  <section>
    <template if:true="{isTypePie}">
      <lightning-chart-pie data="{data}"></lightning-chart-pie>
    </template>
    <template if:true="{isTypeBar}">
      <lightning-chart-bar data="{data}"></lightning-chart-bar>
    </template>
    <template if:true="{isTypeHistogram}">
      <lightning-chart-histogram data="{data}"></lightning-chart-histogram>
    </template>
  </section>
  <template></template
></template>
```

> Note: The `lightning-input` component is implemented using this pattern.

Besides the verbosity of this implementation, note a couple of things:

- All dependencies are hard: **the component explicitly depends on `lightning-chart-histogram`, `lightning-chart-bar`, and `lightning-chart-pie`**, so they're sent as part of the definition (they're part of the dependency tree).

- The chart components aren't reachable (I can't do `querySelector`) until the component is rendered with the `if` condition evaluated to `true`.

When a developer uses this component, the DOM looks something like this:

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

Now let's imagine we have a lot of charts, like thirty. The template becomes verbose if we have to do all this conditional logic.

```html
<template>
  <lwc-chart-impl lwc:dynamic="{dynamicCtor}" data="{data}"></lwc-chart-impl>
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

Now let's look at an implementation that uses the directive `lwc:dynamic`, which tells LWC that the constructor isn't known until run time.

What is important is that both implementations are **semantically equivalent**.

- They have the **same dependencies** (template declared dependencies become static imports).

- You can't reach them until the constructor is set (same as the previous `if` statement).

> The directive waits for a constructor to be set. **How we get that constructor is unrelated to the use of the `lwc:dynamic` directive**.

The amazing properties of the examples above are they are **fully statically analyzable and predictable**; we **never have to make a request to the server** to fetch a particular implementation (this fact is an invariant later on), because they're part of the dependency tree (static imports).

The problem is, though, that maybe some of the dependencies are heavy. Let's look at a concrete example.

```html
<template>
  <lwc-chatter-impl
    lwc:dynamic="{dynamicCtor}"
    data="{data}"
  ></lwc-chatter-impl>
</template>

<script>
  import isPhone from "@salesforce/api/isPhone";

  export default class Chatter extends LightningElement {
    @track dynamicCtor;

    async connectedCallback() {
      if (isPhone) {
        this.dynamicCtor = await import("lwc-chatter-mobile");
      } else {
        this.dynamicCtor = await import("lwc-chatter-desktop");
      }
    }
  }
</script>
```

So `chatter-mobile` and `chatter-desktop` are **two fundamentally different implementations** and they have huge subtrees, let's say +50 kb compress and minified each. Now because of being **static dependencies** we bring both branches of the tree, which is undesirable for performance.

For this use case we want to be able to “cut the tree” (use soft-dependencies), and only bring the `chatter-phone` component tree on mobile and the `chatter-desktop` implementation on desktop. In other words, we want to make the dependencies “soft” and load them after the fact.

In order to do that, we use [dynamic imports](https://github.com/tc39/proposal-dynamic-import), which give us statically analyzability (although the dependency is soft, we still know that exists) and also allow us to abstract the loading mechanism and the module evaluation.

```html
<template>
  <lwc-chatter-impl
    lwc:dynamic="{dynamicCtor}"
    data="{data}"
  ></lwc-chatter-impl>
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

By using **string-lteral dynamic imports**, we make the dependencies **soft** (not part of the dependency tree), meaning that they're loaded at run time (a network call may be made). By forcing some invariants (described below) in the way we use those dynamic imports, we gather metadata at compile time that allows us to optimize **when** those resources can be prefetched.

### Guiding Principles

We want to avoid ad hoc solutions and proprietary non-standard ways to define dynamic imports. We want to build an API that is future-proof in case there are changes to the modules specification.

### General Invariants

The following invariants aren't necessarily constraints on LWC, but rather are explicitly defined for Lightning Platform use cases.

- A dynamic import must always be statically verifiable.
  - A dependency should never be discovered or learned of for the first time at run time.
  - The ability to analyze is orthogonal to when a component is loaded.
- It must not break the reactivity model or the general framework lifecycle.
  - After the component is loaded, it should function within the lifecycle of the parent component.
  - No special handling in user-land is needed.
- It must work regardless of the algorithm, even if nothing is preloaded at all.
  - The decision for when a dynamic dependency is loaded is orthogonal to its definition.
  - We might have a preload mechanism based on MRU or on a specific context (for example, mobile) but the component should work regardless of the algorithm even if nothing is preloaded at all.

These invariants follow the same pattern that we use for `@wire`: the ability to understand and resolve dependencies at compile, build, and design time.

### API

The API and ergonomics for dynamically loading a module has a similar mental model to the HTML `is` attribute proposal (which we can't use since Safari won't implement it in the foreseeable future): An arbitrary tag name on which you can assign any constructor.

```json
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
  <lazy-component
    lwc:dynamic="{lazyConstructor}"
    value="{value}"
    other="test"
  ></lazy-component>
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
       ? $api.c("lazy-component", $cmp.lazyConstructor)
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

As you can see from the implementation details, we've transformed the `lwc:dynamic` directive into a condition, and we've abstracted the dynamic import mechanism into a high-order loader call provisioned by the application container (this will be configurable).

#### API invariants

- A dynamic import can't be a top-level await - it must be on a block statement.
- You may only use your own namespace for the tag name of the lazy component.
- Arguments of the dynamic import function must be statically verifiable (requirement for offline)
  - We only support string literals
- The attribute bindings must be explicit.
  - No dynamic bag is allowed (for example, React `...props`)

> Note: The compiler options allow you to restrict and/or use the loader as described previously.

```js
// This code works
export async function example2() {
  const m = await loadModule("lightning/rtlFoo?pivots=rtl");
}

// Example of incorrect usage: non discoverable module
export async function example3() {
  const moduleName = "lightning/" + isPhone ? "phone" : "desktop";

  // This code throws since we can't statically verify the module name
  const m = await loadModule(moduleName);
}
```

The previous example uses a **variable dynamic import**, which isn't statically analyzable. Using this technique is discouraged and requires multiple levels of approvals.

#### Metadata

The compiler adds to its metadata all of the parsed dynamic imports, its source value, and the hints (`queryString` parameters). Here is an example of the output:

```json
{
  "dynamicImports": [
    {
      "moduleName": "lightning-mobile-foo"
    }
  ],
  "unknownDynamicImports": false
}
```

## Drawbacks

The biggest drawback of this feature is the historic abuse of dynamic imports in the Lightning Platform. We will add guardrails to prevent developers from causing problems for themselves.

## Alternatives

We considered several non-standard alternatives, but we came to the conclusion that they would put us on the wrong path.

# Adoption strategy

This feature will initially be implemented in OSS and will be rolled out to the Salesforce Lightning Platform depending on usage and feedback.

# How we teach this

Dynamic imports is a standard feature (in Stage 4). Our documentation around the `lwc:dynamic` directive should provide a comprehensive list of known anti-patterns along with details around when this feature should be used.

# Unresolved questions

- Naming for the directive can be changed.
- An equivalent fully declarative lazy loading.
