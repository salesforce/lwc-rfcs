---
title: Scoped custom elements
status: DRAFTED
created_at: 2021-02-12
updated_at: 2021-02-12
pr: (leave this empty until the PR is created)
---

# Scoped custom elements

## Summary

This feature allows to use scoped custom elements in the core platform as described in the WICG proposal: [Scoped Custom Element Registries](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/Scoped-Custom-Element-Registries.md).

## Motivation

MuleSoft has a pluggable application made with custom elements (you can extend it to any cloud having their non-LWC custom elements). This application is shared cross-platform (Anypoint, the core platform, 3rd-party platforms).
We see a need for this pluggable application (the components) to be shared with our customers in their on/off-core applications.
It is currently impossible to use custom elements in an on-core application. The other obstacle is including custom elements in an LWC package, which is scoped by nature. The custom elements should be able to work with different packages. 
Finally, different customers can use different version of the same components published by us in their applications which under normal circumstances would cause names conflicts.

This is where implementation of scoped custom element registries can solve all of the above problems.

1. In introduces custom elements inside the core platform
2. LWC based packages create their own registry and passes the reference to it to the component (see below in detailed design section)
3. Since packages uses custom registries there is no chance for name collision or a component hijacking.

## Detailed design

An LWC package defines own custom elements registry and keeps the reference to be passed to the components.

`registry.js`

```javascript
export const registry = new CustomElementRegistry();
```

The custom element used in an LWC has to support a way of passing the registry reference to the shadow root creation. This would require the included custom element to support a static property on the class that keeps the reference to the registry. When the element is being created it uses this reference to pass it to the shadow root creation.

`@scope/custom-element/src/CustomElement.js`

```javascript
export declare class CustomElement extends HTMLElement {
  constructor() {
    const registry = CustomElement.registry;
    this.attachShadow({mode: 'open', registry});
  }
}
```

The LWC component imports the custom element, passes the handler to the custom element, and registers it.

```javascript
import { LightningElement } from 'lwc';
import { CustomElement } from '@scope/custom-element';
import { registry } from './registry.js';
CustomElement.registry = registry;
registry.define('custom-element', CustomElement);

export default class Example extends LightningElement {
  ...
}
```

```html
<template>
  <custom-element></custom-element>
</template>
```

The whole registration process of external custom elements could be done in the `registry.js` file instead of inside an LWC component to avoid name duplicates.

Finally, the LWC component uses the same registry to register itself in the same registry as the rest of the components.

### Polyfills

There is [Scoped CustomElementRegistry polyfill](https://github.com/webcomponents/polyfills/tree/master/packages/scoped-custom-element-registry) that can be used as a base for the implementation. However, it is a work in progress.

## Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people the proposed feature
- integration of this feature with other existing and planned features
- cost of migrating existing applications (is it a breaking change?)

I see a problem with the static field with the registry on the element. Generally I hate an idea of having static fields to pass references to a crucial part of the registration process. Even though it can't be altered by another application, an author can easily override it with another registry. This could be potentially beneficial in cases when an LWC component is made to be sharable so this component's registry is also dynamic. But it could potentially mean that the scope of the registration change without author realizing what is happening.

## Alternatives

Moving existing applications from CE to LWC. However, the cost of doing so is huge. We also can't expect all customers to migrate their own stack to LWC. Furthermore, LWC is not particularly friendly towards being used as a component in non-LWC projects so in many cases migration to LWC is impossible.

## Adoption strategy

If we implement this proposal, how will existing Lightning Web Stack developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

TBD

# How we teach this

This is set to become a web platform standard. When it become a standard then there's no need for training materials other than code examples.

# Unresolved questions

What happens to components that are included into the shadow DOM of the imported components? Do they register in the custom registry? They are included into a component which registers in a custom registry but these components can be already registered in a global registry.
