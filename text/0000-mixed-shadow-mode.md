---
title: Mixed Shadow Mode
status: DRAFTED
created_at: 2021-05-07
updated_at: YYYY-MM-DD
pr: https://github.com/salesforce/lwc-rfcs/pull/49
---

# Mixed Shadow Mode

## Introduction

LWC offers a collection of polyfills that implement Shadow DOM features. These polyfills are
collectively referred to as synthetic Shadow DOM and are published to npm under the
`@lwc/synthetic-shadow` package. Synthetic Shadow DOM provides a consistent experience for LWC web
components across the Salesforce-supported browser matrix by implementing Shadow DOM semantics in
older browsers that do not support it.

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
synthetic shadow mode. If the browser supports Shadow DOM the shadow mode will be native. If the
browser does not support Shadow DOM (e.g., IE11) the shadow mode will be synthetic. The test to
determine Shadow DOM support will be to check whether `window.ShadowRoot` is defined.

Setting `shadowSupportMode` to `default` is equivalent to not setting the property at all. A value
of `default` results in the default behavior where the shadow mode of the component depends only
on the inclusion of the `@lwc/synthetic-shadow` polyfill.

The only valid values are `default` and `any`. Any other value will result in a compiler error.

### Implementation

The value of this property will be read before component construction and cached for the lifetime of
the application. All instances of a component will operate in the same shadow mode and an error will
be thrown if the value of `shadowSupportMode` is unrecognized.

The `shadowSupportMode` property is implemented as an enum to keep the door open for additional
shadow modes. For example, we might implement a `native-only` shadow mode for components that only
support native Shadow DOM, or we might implement a `synthetic-only` shadow mode for components that
only support synthetic Shadow DOM.

### Composition

In terms of composition, a synthetic mode component can contain a native mode component, but the
inverse is not allowed. Not only does this make things easier to reason about, but many existing
components rely on workarounds in synthetic mode that are not possible to support in native mode.
Examples of these observable differences are further discussed below.

As this invariant cannot be asserted during compile-time, the engine will assert it during runtime.

### Testing

#### Framework

The LWC framework currently has integration tests for synthetic shadow mode and native shadow mode.
A third testing mode will be introduced for mixed shadow mode. This test environment will ensure
that components running in native mode will work correctly even when synthetic Shadow DOM polyfills
are present.

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

In the following example, `span` will not exist in the DOM in synthetic shadow, but it will exist in
native shadow.

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
exists for legacy reasons, is only possible to allow in a synthetic Shadow DOM environment, and
cannot be preserved in a native Shadow DOM context.

### Accessibility

The most common accessibility issue in LWC is related to id-referencing across shadow boundaries
(e.g., an aria-describedby attribute that references an element outside of its shadow tree). The
current workaround for this is to bypass the LWC engine's id-mangling by setting attributes
dynamically, but such a workaround would not work in native Shadow DOM.

### Instrumentation

Applications that rely on instrumentation libraries that don't yet support Shadow DOM are currently
able to obtain references to descriptors like `addEventListener()` before they are patched and use
them to override `@lwc/synthetic-shadow` polyfills where needed. Such workarounds would not work for
events originating from components that prefer native Shadow DOM.

### Styling

Applications currently rely on global stylesheets to apply deeply throughout the DOM. With native
Shadow DOM, this would not be possible. All of a component's styles would have to be generated and
added to the component bundle.

## Drawbacks

A potential (non-verified) drawback is that existing components may experience a slight performance
degradation due to the fact that mixed mode may require some logic-forking.

## Alternatives

Other alternatives have not been considered but contributions to this proposal are welcome.

## Adoption strategy

When setting `shadowSupportMode` to `any`, it will be the responsibility of the component author to
ensure that all components in the subtree are compatible with native Shadow DOM. Due to the nature
of this feature, it is expected that initial usage will start at the leaf components and work its
way up the component tree. If a component in the subtree does not support native Shadow DOM, the
component author should work with the other component author to make the appropriate updates.

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
