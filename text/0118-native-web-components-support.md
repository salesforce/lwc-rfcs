---
title: Native Web Components Support
status: ACTIVE
created_at: 2022-07-05
updated_at: 2022-09-08
pr: https://github.com/salesforce/lwc-rfcs/pull/66
---

# Native Web Components Support

## Summary

This proposal is about enabling developers to integrate third party web components by introducing
LWC support for native web components.

## Motivation

It is common for developers to want to integrate third party web components in an LWC application.
While there are several ways to do this today (e.g., through the use of iframes, or imperatively
using the `lwc:dom="manual"` directive), the preferred way would be to enable this use case through
first class support of native web components. Specifically speaking, this means enabling the
rendering of native web components in LWC templates. Browser support for native web components was
still in the early stages when LWC was first implemented, but it is quite good today. It makes sense
to do this work now since many frameworks, including LWC, now support native web components as a
compilation target.

## Detailed design

This section covers the identified work to add native web component support.

### Constructor resolution

By convention, LWC implicitly resolves component constructors for custom elements in its template.
This behavior needs to be modified to allow authors the flexibility to resolve their own components.

#### Proposed solution

LWC would provide a directive (e.g., `lwc:external`) which signals that component resolution should
be delegated to the browser. LWC will not attempt to resolve components for custom elements tagged
with this directive. If it happens that the component was not registered, the custom element will
simply render as an HTMLUnknownElement, which is the default browser behavior.

For example, given the following template:
```html
<template>
  <lwc-foo foo={bar}></lwc-foo>
  <native-foo lwc:external foo={bar}></native-foo>
</template>
```

The template compiler would output the following:
```js
import _lwcFoo from "lwc/foo";
import { registerTemplate } from "lwc";
function tmpl($api, $cmp, $slotset, $ctx) {
  const { c: api_custom_element, h: api_element } = $api;
  return [
    api_custom_element("lwc-foo", _lwcFoo, {
      props: {
        foo: $cmp.bar,
      },
      key: 0,
    }),
    api_element("native-foo", {
      attrs: {
        foo: $cmp.bar,
      },
      key: 1,
    }),
  ];
}
export default registerTemplate(tmpl);
tmpl.stylesheets = [];
```

If the `lwc:external` directive is used on a registered component, the engine will simply render the
associated custom element and defer the upgrading to the browser.

```html
<template>
  <lwc-foo foo={bar}></lwc-foo>
  <lwc-foo lwc:external foo={bar}></lwc-foo>
</template>
```

```js
import _lwcFoo from "lwc/foo";
import { registerTemplate } from "lwc";
function tmpl($api, $cmp, $slotset, $ctx) {
  const { c: api_custom_element, h: api_element } = $api;
  return [
    api_custom_element("lwc-foo", _lwcFoo, {
      props: {
        foo: $cmp.bar,
      },
      key: 0,
    }),
    api_element("lwc-foo", {
      attrs: {
        foo: $cmp.bar,
      },
      key: 1,
    }),
  ];
}
export default registerTemplate(tmpl);
tmpl.stylesheets = [];
```

### Passing data

By convention, custom elements are assumed to be backed by LWC components and values are set through
their properties. Given that a native web component might expect data passed as an attribute or as a
property, we would need to support both mechanisms. There are multiple approaches we could take
here, and the following options are how some popular frameworks do it today. We could default to
either attributes or properties and introduce a directive that would act as a signal to the engine
to use one or the other, we could implement a runtime heuristic to choose one over the other, or we
could default to attributes and set properties when setting a data structure.

#### Proposed solution

When passing data to a native web component, we can take a heuristical approach where if the
property exists, the property is set. Otherwise the attribute is set. While it may be advantageous
to require an explicit signal in the form of a directive (both in terms of static analysis and
performance), the proposed solution strikes a good balance between developer experience and API
semantics. This heuristical approach will require a check on the element's prototype to see whether
a property is defined. This will incur a runtime performance cost, which can be amortized over the
lifetime of the component by caching the result of the check.

### Events

Today, LWC's declarative event bindings only support lowercase events. Events using anything other
than lowercase can be listened for using the imperative `addEventListener()` API. There are several
historical discussions (e.g., [#1811][1811] and [#1904][1904]) about relaxing this constraint;
however, the current behavior would not need to change in order to support native web components. It
should be noted that adding support for events has always been about improved interoperability, as
the browser's imperative API has no restriction regarding the event type.

EventTarget polyfilled methods and properties such as `add/removeEventListener` and `target` go
through the synthetic shadow polyfill due to those methods and properties being globally patched.
This is the current status quo for native shadow roots and should not be an issue for supporting
third party web components.

### Serverside rendering (SSR)

If an external element is marked with a directive that identifies it as such, its custom element
should be rendered and its attributes should be set. Properties will not be set and no attempt will
be made to render the subtree.

#### Proposed solution

From the platform perspective, there are a few things that Locker vNext needs to implement in order
to GA this feature. 1) Lift the restriction on `customElements.define()` and 2) provide the
virtualization mechanism to allow different namespaces to use the same component name.

## Drawbacks

As with any new feature, adding support for native web components introduces additional complexity.
One such change will involve the prevention of external custom elements from being hoisted when
optimizing static content.

## Alternatives

No alternatives were considered for the constructor resolution portion of this proposal. An initial
uninformed idea to introduce error handling for implicit constructor imports was quickly eliminated
due to the fact that dynamic imports are promise-based.

Alternatives for passing data have already been discussed.

## Adoption strategy

Not much would be required to "sell" this feature as it is something that developers have been
asking for. Documentation of the new `lwc:external` directive would be required.

# Unresolved questions

- Investigate whether the synthetic shadow slotting implementation is compatible with native web
  components.
  - We should be fine assuming we can enforce the disabling of synthetic shadow slotting logic for
  external custom elements in the engine. Good test to have either way.

[1811]: https://github.com/salesforce/lwc/issues/1811
[1904]: https://github.com/salesforce/lwc/issues/1904
