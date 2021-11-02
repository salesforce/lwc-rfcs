---
title: Native Web Component Support
status: DRAFTED
created_at: 2021-10-21
updated_at: YYYY-MM-DD
pr: https://github.com/salesforce/lwc-rfcs/pull/58
---

# Native Web Component Support

## Summary

It is possible to author LWC components that can be used natively in an off-platform application by
registering the `CustomElementConstructor` getter.

```js
import { LightningElement } from 'lwc';
class HelloWorld extends LightningElement {}
customElements.define('hello-world', HelloWorld.CustomElementConstructor);
```

```html
<html>
  <body>
    <hello-world></hello-world>
  </body>
</html>
```

The usage of native web components is generally not allowed on the platform so the main use case is
off-platform applications.

The following example attempts to use native web components in an LWC application, which is not
supported.

```js
customElements.define('hello-world', class HelloWorld {});
```

```html
<template>
  <hello-world></hello-world>
</template>
```

This is because, by design, LWC relies on a custom module loader and never defers to the custom
element registry when resolving components. The Salesforce platform is based on a multi-tenant
architecture, and as such, it relies on a directory-based module resolution system that is
compatible with how developers create components on the platform. This basically guarantees that no
two parties can define the same component, allowing LWC to side-step a variety of security concerns.

This proposal is about adding native web component support for off-platform users, with the
additional goal of exploring how we might do the same for platform users. Any solution should be
forward-compatible with [Scoped Custom Element Registries].

## Motivation

The long term goal of LWC is to provide a thin abstraction layer on top of native browser APIs.
Adding support for the custom element registry for off-platform applications would be a step towards
that goal.

For platform applications, the major motivation is to facilitate platform integration and allow
developers to migrate existing tooling and solutions without having to utilize LWC.

## Detailed design

### OSS restriction

In order for LWC to support native web components for OSS applications, the following needs to
happen.

1. Prevent LWC components from using the same custom element as a native web component.
2. Prevent native web components from using the same custom element as an LWC component.
3. Remove the restriction that LWC must know about all available components before rendering.

In order to prevent namespace collisions between LWC components and native web components, LWC
components should be registered in the same custom element registry that native web components will
be registered in.

The third requirement of relaxing the restriction that LWC must know about all available components
before rendering, would be a significant change. Currently, if an unknown component is encountered,
an exception is thrown. This behavior should be modified so that unrecognized custom elements are
rendered as-is with the expectation that they will be natively upgraded. This opens up the
possibility of rendering custom elements that are `HTMLUnknownElement`, which is basically an
extension of `HTMLElement` without any additional fields or methods.

To ensure that only off-platform applications can use this feature, it should be implemented behind
a feature flag (e.g., `ENABLE_CUSTOM_ELEMENT_REGISTRY_RESOLUTION`).

### Platform restriction

In addition to the OSS restrictions outlined above, in order for LWC to support native web
comopnents for platform applications, the following needs to happen.

1. Prevent registration of custom elements in a namespace different from the current one.

As mentioned earlier, platform applications at Salesforce are based on a multi-tenant architecture.
This means that, on any given page, components can be authored by internal developers, customers,
and third party developers. In such an environment, keeping data secure is a major concern, and
exposing a global registry is problematic because a bad actor could hijack an existing component
by registering it first. This concern can be alleviated by restricting the registration of custom
elements to the current namespace.

## Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Lightning Web Components
- integration of this feature with other existing and planned features
- cost of migrating existing Lightning Web Components applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

## Alternatives

What other designs have been considered? What is the impact of not doing this?

## Adoption strategy

If we implement this proposal, how will existing Lightning Web Components developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Lightning Web Components patterns?

Would the acceptance of this proposal mean the Lightning Web Components documentation must be
re-organized or altered? Does it change how Lightning Web Components is taught to new developers
at any level?

How should this feature be taught to existing Lightning Web Components developers?

# Unresolved questions

1. Are there any concerns with allowing the rendering of custom elements that are
   `HTMLUnknownElement`?


[Scoped Custom Element Registries]: https://github.com/WICG/webcomponents/blob/gh-pages/proposals/Scoped-Custom-Element-Registries.md
