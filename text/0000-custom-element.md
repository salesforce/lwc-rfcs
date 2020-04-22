---
title: Custom Element Constructor
status: DRAFTED
created_at: 2019-07-05
updated_at: 2019-07-05
pr: https://github.com/salesforce/lwc-rfcs/pull/7
---

# Custom Element Constructor

## Summary

Facilitate the registration of LWC Constructor as Custom Elements.

## Motivation

As of today, the way to generate a qualifying Custom Element from a LWC Constructor, is via an experimental API: `buildCustomElementConstructor()`. Since this is a primary use case for LWC, we want to make it more ergonomic, and simpler. Additionally, the experimental API allow you to generate more than one Custom Element per LWC Constructors, which does not align with the web components semantics.

## Goals

1. Provide a dead simple way to register a custom element from a LWC Constructor.
2. Preserve the web components semantics.

## Proposal

This proposal provides a high-level API (an abstraction layer) for authors to access the custom element associated to a LWC Constructor:

```js
import XFoo from 'x/foo';
customElements.define('x-foo', XFoo.CustomElementConstructor);
// using tagName `x-foo`:
document.body.appendChild(document.createElement('x-foo'));
```

Pros:
* it is extremely simple.
* it reads very nicely in plain english.
* it preserves the semantics of web components in a way that you can't bend them.

Cons:
* it can't be statically analyzed (but since we don't use this in platform, that should be ok).
* it does not provide a way for you to control whether or not you want open or closed shadowRoot (this is fine because consumers should not be in control)

### Semantics and Invariants

* There is a 1-1 mapping between LWC Constructors and the corresponding Custom Element.
* Claiming the Custom Element for the abstract `LightningElement` must be disallow.

### Implementation Details

`LightningElement.CustomElementConstructor` is a static getter that can be accessed by any Constructor extending `LightningElement`. This getter can rely on the `this` value, which points to the sub class. This can be used in a Map to link the LWC to the Custom Element. This is very similar to the trick we use in  `LightningContextElement.Provider` for context.

As a consequence of this change, the existing experimental API can be deprecated.

## Adoption strategy

This is a brand new feature, we just need to document it.

## How we teach this

* The consumer aspect of it is very straight forward considering that they don't have to learn a new API, just a static field name in LWC Constructors.
