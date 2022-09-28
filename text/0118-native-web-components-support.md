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
first class support of native web components. To be precise, this means enabling the rendering of
native web components in LWC templates.

Browser support for native web components was still in the early stages when LWC was first
implemented, but it is quite good today. It makes sense to do this work now since many frameworks,
including LWC, now support native web components as a compilation target.

## Detailed design

This section covers the identified work to add native web component support.

### Constructor resolution

By convention, LWC implicitly resolves component constructors for every custom element found in its
template.

```html
<lwc-foo foo={bar}></lwc-foo>
```

```js
import _lwcFoo from "lwc/foo";
```

Any custom elements in the template for which a constructor cannot be resolved will result in an
invalid import runtime error.

This behavior needs to be modified to allow authors the flexibility to resolve their own components.

#### Proposed solution

LWC would provide a directive (e.g., `lwc:external`) which signals to the compiler that component
resolution should be delegated to the browser. LWC will not attempt to resolve components for custom
elements tagged with this directive. If it happens that the component was not registered, the custom
element will simply render as an instance of the native `HTMLUnknownElement` interface which extends
`HTMLElement` without adding any properties or methods. This is the default browser behavior for
custom elements before they are upgraded.

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

In general, a component's public API can be implemented via properties, attributes, or a combination
of both. By convention, LWC components expect data to be passed via properties. This allows for
complex data structures and a variety of data types as arguments to an LWC component. An API backed
solely by attributes can only accept strings as data.

Depending on a native web component's implementation details, it might expect data to be passed as
an attribute or as a property. We would need to support both mechanisms for the best developer
experience. There are multiple approaches we could take here, and the following options are how some
popular frameworks do it today.

1) Default to either attributes or properties and introduce a directive to override the default
behavior.
2) Implement a runtime heuristic that attempts to intelligently choose one over the other.
3) Default to attributes and only set properties in the case of a data structure.

#### Proposed solution

When passing data to a native web component, we can take a properties-if-available approach where
attributes are set by default, but properties are set when they exist. While it may be advantageous
in terms of static analysis and performance to require an explicit signal in the form of a
directive, the proposed solution strikes a good balance between developer experience and API
semantics. This heuristical approach will require a runtime check on the element's prototype to see
whether a property is defined. This preliminary check will incur a runtime performance cost, but
this cost can be amortized over the lifetime of the component by caching the result of the check.

The current LWC implementation of normalizing attribute names to property names will be used as-is
without modification.

### Events

Today, LWC's declarative event bindings only support lowercase events. Events using anything other
than lowercase can be listened for using the imperative `addEventListener()` API. There are several
historical discussions (e.g., [#1811][1811] and [#1904][1904]) about relaxing this constraint;
however, the current behavior would not need to change in order to support native web components. It
should be noted that adding support for more than lowercase events has always been about improving
interoperability, as the browser's imperative API has no such restrictions.

The current LWC implementation of declarative event binding will be used as-is without modification.

### Serverside rendering (SSR)

When rendering an LWC component on the server, we are able to drill down into the component subtree
because we have recursive access to component definitions. When rendering an external element marked
as such on the server, we will simply render the top-level custom element without the rest of the
component subtree because we don't know what that looks like. In other words, regardless of the size
of the component subtree, we will only be rendering the root component on the server. As for data,
we will only be setting attributes.

## Drawbacks

As with any new feature, adding support for native web components will introduce additional
complexity. One such change will involve the prevention of external custom elements from being
hoisted when optimizing static content.

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
