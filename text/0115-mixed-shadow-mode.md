---
title: Mixed Shadow Mode
status: IMPLEMENTED
created_at: 2021-05-07
updated_at: 2021-09-14
pr: https://github.com/salesforce/lwc-rfcs/pull/49
---

# Mixed Shadow Mode

## Introduction

LWC offers a collection of polyfills that implement Shadow DOM features. These polyfills are
collectively referred to as "the synthetic shadow polyfill" and are published to npm under the
`@lwc/synthetic-shadow` package. This polyfill implements what we refer to as "the synthetic shadow
DOM". The synthetic shadow DOM provides a consistent experience for LWC web components across the
Salesforce-supported browser matrix by implementing Shadow DOM semantics in older browsers that do
not support it.

Native Shadow DOM support has become more common since the start of the LWC project but applications
continue to rely on the synthetic shadow polyfills to maintain backwards-compatibility. The removal
of these polyfills can result in broken experiences for existing applications due to dependencies on
certain "escape hatches" that are not possible in a native Shadow DOM environment (e.g.,
instrumentation, styling, etc).

Applications that wish to use LWC components in a native shadow context simply need to omit the
inclusion of the `@lwc/synthetic-shadow` polyfill; however, this all-or-nothing approach is not a
realistic option for large applications that cannot afford a rewrite.

This proposal opens up an incremental migration path for applications moving to native Shadow DOM by
enabling the usage of both native and synthetic Shadow DOM in the same application.

## Detailed design

Up until now, the two choices of shadow semantics have been synthetic mode and native mode. All the
components in an application run in either synthetic mode or native mode, depending on whether the
synthetic shadow polyfill (i.e., `@lwc/synthetic-shadow`) is applied. Mixed mode introduces the
ability for a component to access native Shadow DOM APIs, even while the synthetic shadow polyfill
is applied. Mixed mode should be transparent to components and the framework should guarantee this
through integration testing, which is further discussed below.

### Opting in

A component can signal that it supports both synthetic and native Shadow DOM by setting a static
property called `shadowSupportMode` to `any`.

```js
export default class extends LightningElement {
    static shadowSupportMode = 'any';
}
```

This mode will result in the whole component subtree to operate in either native shadow mode or
synthetic shadow mode. If the browser supports Shadow DOM, the shadow mode will be native. If the
browser does not support Shadow DOM (e.g., IE11), the shadow mode will be synthetic. The test to
determine Shadow DOM support will be to check whether `window.ShadowRoot` is defined.

### Opting out

A component that is extending a component class that opts in to native shadow mode can opt out of
native shadow mode by setting its own static `shadowSupportMode` property to `reset`; however,
this does not allow it to opt out of native shadow mode if any of its ancestor components set their
static `shadowSupportMode` property to `any`.

Setting `shadowSupportMode` to `reset` acts to reset the shadow mode to the original default
behavior where the mode depends only on the inclusion of the `@lwc/synthetic-shadow` polyfill:
synthetic mode if the polyfill is present and native mode if it isn't.

The only valid values are `reset` and `any`. Any other value will result in a compiler error.

### Implementation

The value of this property will be read before component construction and cached for the lifetime of
the application. All instances of a component will operate in the same shadow mode and an error will
be thrown if the value of `shadowSupportMode` is unrecognized. Note that this is different for
components that transitively participate in native mode during runtime; the shadow mode of these
components depends on the structure of the component tree and may be different for each instance.

The `shadowSupportMode` property is implemented as an enum to keep the door open for additional
shadow modes. For example, we might implement a `native-only` shadow mode for components that only
support native Shadow DOM, or we might implement a `synthetic-only` shadow mode for components that
only support synthetic Shadow DOM.

### Composition

In terms of composition, the transitive nature of this feature implies that synthetic mode
components can contain native mode components, but the inverse is not possible. This invariant makes
things easier to reason about and allows us to completely remove polyfilled behavior from DOM APIs
being used. There is also a technical reason for this invariant--slot elements introduced by
components operating in synthetic mode will end up being "owned" by a native mode ancestor,
resulting in unintended consequences when distributing slotted content.

There is no need to assert this invariant during runtime, as it is built into the design.

### Testing

#### Framework

The LWC framework currently has integration tests for synthetic shadow mode and native shadow mode.
A third testing mode will be introduced for mixed shadow mode. This test environment will ensure
that components running in native mode will work correctly even when the synthetic shadow polyfill
is present.

The existing WPT (Web Platform Tests) test suite for Shadow DOM APIs will also be run to identify
any coverage gaps in LWC integration tests.

#### Component

Components that support both shadow modes will also need to be tested in both modes. LWC test
utilities will be updated to facilitate this.

The component must also be tested to ensure that all utilized APIs are available in both modes
across the supported browser matrix, as the level of Shadow DOM support will not be considered when
computing the shadow mode of a component. A Shadow DOM API should be considered unavailable if it is
not implemented in the synthetic polyfill or if it is not natively implemented across all supported
browsers.

## Observable differences

### Light DOM and assigned elements

In native Shadow DOM, elements are rendered in a component's light DOM and assigned to slots in a
child component's shadow DOM. This means that elements will exist in the DOM regardless of whether
they are assigned to a slot. This is not the case for LWC--elements that are passed down to child
components are never rendered unless they are assigned to a slot.

In the following example, `span` is not assigned to a slot in `x-child`. In native mode, the `span`
will exist in the DOM. In synthetic mode, the `span` will not exist in the DOM.

```html
<!-- x-parent -->
<template>
    <x-child>
        <span>foo</span>
    </x-child>
</template>

<!-- x-child -->
<template>
    <p>child</p>
</template>
```

This observable difference will not be remediated as such a change would be non-trivial and
backwards-incompatible with the current synthetic shadow implementation.

### Lifecycle timing

Due to the way that slotting is implemented in `@lwc/synthetic-shadow`, there is an observable
difference in the timing of lifecycle hooks for slotted elements.

As mentioned previously, in synthetic mode, slotted elements that are never assigned to a slot are
not rendered. This means that their lifecycle hooks are never invoked.

Another difference is that for synthetic mode, lifecycle hooks are invoked in the order of
appearance after they are assigned, whereas in native mode, they are invoked in the order of
appearance in the template.

### Listening for non-composed events above the root LWC node

There currently exists an escape hatch that allows listeners to handle non-composed events outside
of the root LWC node if the event originates from a non-LWC component in the subtree. This behavior
exists for legacy reasons, is only possible in a synthetic Shadow DOM environment, and cannot be
preserved in a native Shadow DOM context.

### Accessibility

A common accessibility workaround in LWC is related to id-referencing across shadow boundaries
(e.g., an aria-describedby attribute that references an element outside of its shadow tree). In
synthetic mode, it is possible to create references across shadow boundaries by dynamically setting
attributes in both the parent and child components. Such a workaround would not work in native
Shadow DOM because of native id-scoping.

The workaround for this would be to dynamically set text in the right places to keep id-referencing
contained in the same shadow root. Components in the `lightning` namespace have been doing this
successfully for some time now.

### Instrumentation

Applications that rely on instrumentation libraries that don't yet support Shadow DOM are currently
able to obtain references to descriptors like `addEventListener()` before they are patched by the
synthetic shadow polyfill, and use them to override `@lwc/synthetic-shadow` polyfills where needed.
This allows them to handle non-composed events that should never have been able to cross shadow
boundaries.

Such a workaround would not work for events originating from components operating in native shadow
mode because the composedness of an event is controlled natively by the browser.

### Styling

Applications currently rely on global stylesheets to apply deeply throughout the DOM. With native
Shadow DOM, this would not be possible. All of a component's styles would have to be generated and
added to the component bundle.

## Drawbacks

A potential (non-verified) drawback is that existing components may experience a slight performance
degradation due to the fact that mixed mode may require some logic-forking.

The code to implement mixed shadow mode will be non-trivial and will eventually be thrown away when
all supported browsers have mature support for Shadow DOM.

## Alternatives

One alternative which would allow us to sidestep the mixed mode feature is to simply deprecate LWC's
synthetic Shadow DOM polyfill.

This alternative is not feasible because Salesforce experiences are composed of components that are
implemented by multiple parties (e.g., Salesforce developers, customers, AppExchange partners, etc)
and coordinating such a switch is nearly impossible.

Introducing a smooth transition path would allow customers to prepare for the native Shadow DOM
rollout without disruption.

## Adoption strategy

When setting `shadowSupportMode` to `any`, the expectation is that all components in the subtree are
compatible with native Shadow DOM. Due to the transitive nature of this feature, it is expected that
initial usage will start at the leaf components and work its way up the component tree, but it is
also possible that things "just work". If a component in the subtree does not function in native
shadow mode, the component author should work with the other component author to make the
appropriate updates.

# How we teach this

Since we plan to eventually deprecate our synthetic shadow polyfill, we should have enough
documentation for developers to understand how mixed mode works before they start implementing new
components. This would allow them to produce components that are future-proof and minimize the
amount of tech-debt from the start.

Observable differences should be documented to assist in the decision of whether a component is a
candidate for native Shadow DOM.

# Unresolved questions

N/A

# Resolved questions

1. Light DOM components - What does the runtime assertion for preventing synthetic mode components
   in the native mode component subtree look like with Light DOM components in play?

   Enforcing the invariant that a synthetic mode component cannot have any native mode component
   ancestors would allow native mode components to contain light DOM components.
