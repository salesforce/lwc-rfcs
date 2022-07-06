---
title: Native Web Components Support
status: DRAFTED
created_at: 2022-07-05
updated_at: 2022-07-05
pr: https://github.com/salesforce/lwc-rfcs/pull/66
---

# Native Web Components Support

## Summary

This proposal is about enabling developers to integrate third party web components by introducing
LWC support for native web components.

## Motivation

It is common for developers to want to integrate third party web components in an LWC application.
While there are several ways to do this today (e.g., through the use of iframes or the
`lwc:dom="manual"` directive), the preferred way would be to enable this use case through the
rendering of native web components. Browser support for native web components was still in the early
stages when LWC was first implemented, but it is quite good today. In addition, many frameworks now
support native web components as a compilation target so it makes sense to add native web component
support as a first class citizen.

## Detailed design

This section covers the identified work to add native web component support.

### Constructor resolution

By convention, LWC implicitly resolves component constructors for every custom element in its
template. This behavior needs to be modified to allow authors the flexibility to resolve their own
components.

#### Proposed solution

LWC would provide a directive (e.g., `lwc:external`) which signals that component resolution should
be delegated to the browser. LWC will not attempt to resolve components for custom elements tagged
with this directive. If it happens that the component was not registered, the custom element will
simply render as an HTMLUnknownElement, which is the default browser behavior.

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
to require an explicit signal in the form of a directive, this approach seems to strike a good
balance between developer experience and API semantics.

### Events

EventTarget polyfilled methods and properties such as `add/removeEventListener` and `target` go
through the synthetic shadow polyfill due to those methods and properties being globally patched.
This is the current status quo for native shadow roots and should not be an issue for supporting
third party web components.

LWC's declarative event bindings only support lowercase events. Events using anything other than
lowercase must be listened to imperatively. This would not need to change in order to support native
web components.

### Serverside rendering (SSR)

LWC's SSR implementation is tailored specifically for LWC and may not be compatible with a native
web component implementation. We would probably need to implement the entire synchronous DOM API if
we wanted to support SSR for native web components.

## Drawbacks

As with any new feature, adding support for native web components introduces additional complexity.

From the platform perspective, there are a few things that Locker vNext needs to implement in order
to GA this feature. 1) Lift the restriction on `customElements.define()` and 2) provide the
virtualization mechanism to allow different namespaces to use the same component name.

## Alternatives

No alternatives were considered for the constructor resolution portion of this proposal. An initial
uninformed idea to introduce error handling for implicit constructor imports was quickly eliminated
due to the fact that dynamic imports are promise-based.

Alternatives for passing data have already been discussed.

## Adoption strategy

Not much would be required to "sell" this feature as it is something that developers have been
asking for. Documentation of the new `lwc:external` directive would be required.

# Unresolved questions

- More of a platform-related question, but do we need to make metadata adjustments to exclude native
  web components from referential integrity?
- Investigate whether the synthetic shadow slotting implementation is compatible with native web
  components.
