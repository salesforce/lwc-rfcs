---
title: Native Web Component Support
status: DRAFTED
created_at: 2021-10-21
updated_at: YYYY-MM-DD
pr: (leave this empty until the PR is created)
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

Once the specification for [Scoped Custom Element
Registries](https://github.com/WICG/webcomponents/issues/716) reaches consensus.

This proposal is about adding support for native web components for off-platform users, with the
additional goal of exploring how we might do the same for platform users.

## Motivation

The long term goal of LWC is to provide a thin abstraction layer on top of native browser APIs.
Adding support for the custom element registry would be a step towards that goal. In addition, if we
are able to figure out a secure way to support native web components on the platform through
forward-compatible abstractions, that would allow a smooth migration path for customers tasked with
migrating existing web components to the platform.

## Detailed design

### OSS restriction

In order for LWC to support native web components registered via `customElements.define()`, the
following needs to happen.

1. Prevent LWC components from using the same custom element as a native web component.
2. Prevent native web components from using the same custom element as an LWC component.
3. Remove the restriction that LWC must know about all available components before rendering.

In order to prevent namespace collisions between LWC components and native web components, LWC
components should be registered in the same custom element registry that native web components will
be registered in.

The third item about relaxing the restriction that LWC must know about all available components
before rendering would be a big change. LWC currently needs to know about all available components
when rendering an application. If it encounters an unknown component, an exception is thrown. This
behavior could be modified so that unrecognized custom elements are rendered as-is with the
expectation that they will be natively upgraded. To ensure that only off-platform applications can
use this feature, it should be implemented behind a feature flag.

It would seem that the reason we have this restriction in
place is to disallow components to be defined during runtime. (Look into whether this is a
platform-specific restriction.)

### Platform restriction

1. Prevent the registering of custom elements in different namespaces.


This behavior allowed us to avoid the problem where different namespaces compete to register the
same custom element as the page is rendered, so to ensure that

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

Optional, but suggested for first drafts. What parts of the design are still
TBD?
