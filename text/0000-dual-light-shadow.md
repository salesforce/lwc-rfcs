---
title: Dual light/shadow components
status: DRAFTED
created_at: 2022-02-08
updated_at: 2022-02-08
champion: Nolan Lawson (nolanlawson)
pr: https://github.com/salesforce/lwc-rfcs/pull/77
---

# Dual light/shadow components

## Summary

[Light DOM components](https://rfcs.lwc.dev/rfcs/lwc/0115-light-dom) are a useful feature in certain contexts.
However, LWC currently requires a component to be either light or shadow – it cannot be both at once.

This RFC proposes a "dual" mode where the same component can run in either light DOM mode or shadow DOM mode.

## Basic example

A component can declare its `lwc:render-mode` to be `"dual"`:

```html
<!-- component.html -->
<template lwc:render-mode="dual">
    <h1>Hello world</h1>
</template>
```

```js
// component.js
export default class extends LightningElement {
  static renderMode = 'dual';
}
```

Then, a parent component can use `lwc:render-mode` to choose to render the child as either light DOM or shadow DOM:

```html
<template>
    <x-component lwc:render-mode="light"></x-component>
</template>
```

```html
<template>
    <x-component lwc:render-mode="shadow"></x-component>
</template>
```

This will result in `<x-component>` being rendered as either light DOM:

```html
<x-component>
    <h1>Hello world</h1>
</x-component>
```

or shadow DOM:

```html
<x-component>
    #shadow-root
        <h1>Hello world</h1>
</x-component>
```

## Motivation

Light DOM and shadow DOM have tradeoffs relative to each other. Each may be useful in certain cases.

Also, the design of LWC lends itself to largely shadow-DOM-neutral component authoring. For instance, LWC supports scoped styles, light DOM `<slot>`s, and `lwc:ref` – all of which work roughly the same between shadow components and light components. So there is not a strong reason for a component to declare upfront whether it is light or shadow.

So naturally, a request we've heard from component authors is that they would like to be able to author a single component and have it render as either light or shadow DOM.

To be fair, there is already a workaround today: composition. I.e., a component could be authored in light DOM, and then a wrapper component could be authored in shadow DOM.

However, this workaround has several downsides:

1. All properties have to be reflected between the shadow parent and light child.
2. All slots have to be propagated from the parent to the child.
3. An extra wrapper component is required, which may be undesired for common base components (e.g. buttons, icons).
4. The two components have different external tag names (e.g. `my-button` and `my-button-shadow`).

This RFC has none of these downsides.

## Detailed design

### Declaring a dual-mode component

Currently the `lwc:render-mode` directive / `static renderMode` property only allows two values: `"light"` and `"shadow"`.

This proposal adds a third value: `"dual"`.

```html
<!-- component.html -->
<template lwc:render-mode="dual">
    <h1>Hello world</h1>
</template>
```

```js
// component.js
export default class extends LightningElement {
  static renderMode = 'dual';
}
```

### Switching between light and shadow mode

Currently, `lwc:render-mode` can only be applied to the top-level `<template>`. In this proposal,
it can also be applied to LWC components referenced inside a template:

```html
<template>
    <x-component lwc:render-mode="light"></x-component>
    <x-component lwc:render-mode="shadow"></x-component>
</template>
```

When used in this way, the value of `lwc:render-mode` can only be a static string. It can be set to `"light"` or `"shadow"`,
which controls the mode that the child component renders in.

If `"dual"` is used in this context, a compile-time error is thrown:

```html
<template>
    <!-- Invalid -->
    <x-component lwc:render-mode="dual"></x-component>
</template>
```

Only three values are accepted in this context: `"light"`, `"shadow"`, or `"inherit"`. Any other value will throw a compile-time error.

### Inheritance

`"inherit"` is a special value of `lwc:render-mode` that can only be used in the context of a child LWC component:

```html
<template lwc:render-mode="dual">
    <x-component lwc:render-mode="inherit"></x-component>
</template>
```

This instructs the `<x-component>` to render using whatever mode the parent component is using – either shadow or light. If `<x-component>` is not a dual-mode component, then a runtime error will be thrown.

For the parent component, however, `"inherit"` can be used in non-dual-mode components as well as dual-mode components:

```html
<template lwc:render-mode="light">
    <!-- x-component will render as light -->
    <x-component lwc:render-mode="inherit"></x-component>
</template>
```

In this case, the same rules apply – the child component uses the mode inherited from its parent.

The above example is equivalent to:

```html
<template lwc:render-mode="light">
    <!-- x-component will render as light -->
    <x-component lwc:render-mode="light"></x-component>
</template>
```

For the purposes of inheritance, `<slot>` contents are considered to be children of the slotting component, not the
slottable component.

```html
<template>
    <x-slottable>
        <!-- Mode is inherited from this template, not from x-slottable's template -->
        <x-component lwc:render-mode="inherit"></x-component>
    </x-slottable>
</template>
```

### Coherence

If a template is declared to be `lwc:render-mode="dual"`, then the corresponding component must also have
`static renderMode = 'dual'`, and vice versa.

If not, an error will be thrown at runtime. (This is the same as what currently happens if `"shadow"` and `"light"`
are mixed between the two.)

### Non-LWC components

`lwc:render-mode` can only be applied to LWC components. If it's applied to any other element, including an 
`lwc:external` component, then a compile-time error is thrown:

```html
<template>
    <!-- Invalid -->
    <third-party lwc:render-mode="light" lwc:external></third-party>
</template>
```

### No explicit `lwc:render-mode`

If a dual-mode component is referenced _without_ an explicit `lwc:render-mode`, then it is considered to be a shadow component:

```html
<template>
    <!-- Valid, defaults to shadow mode -->
    <x-component></x-component>
</template>
```

### Non-dual-mode components

If `lwc:render-mode` is applied to a component that is _not_ a dual-mode component, then a runtime error is thrown:

```html
<template>
    <!-- Invalid -->
    <x-not-dual lwc:render-mode="shadow"></x-not-dual>
</template>
```

```js
// notDual.js
export default class extends LightningElement {
}
```

(This error is thrown regardless of whether the component would have rendered in shadow DOM or light mode.)

### Dynamic components

`lwc:render-mode` is also supported on [dynamic components](https://github.com/salesforce/lwc-rfcs/pull/71), since it
is already known in advance that the dynamic component is an LWC component.

```html
<template>
    <!-- Valid if the component is always dual-mode -->
    <lwc:component lwc:is={ctor} lwc:render-mode="light"></lwc:component>
</template>
```

However, if a non-dual-mode component is rendered, then a runtime error is thrown.

### Shadow DOM mixed mode

A dual-mode component can also use the `static shadowSupportMode` property to control whether it renders in native or
synthetic shadow. This only applies when it is running in shadow mode.

```js
export default class extends LightningElement {
  static renderMode = 'dual';
  static shadowSupportMode = 'any'; // use native shadow when in shadow mode
}
```

### Restrictions

In short:

1. Any restrictions that apply to either light DOM components or shadow DOM components also apply to dual components.
2. Component authors are responsible for handling runtime differences, e.g. `this.querySelector` vs `this.template.querySelector`.

#### Light DOM restrictions that apply

Following [light DOM restrictions](https://rfcs.lwc.dev/rfcs/lwc/0115-light-dom), a dual-mode component cannot have
a `slotchange` event listener on a `<slot>`. This throws a compile-time error:

```html
<template lwc-render-mode="dual">
    <!-- Invalid -->
    <slot onslotchange={onSlotChange}></slot>
</template>
```

However, unlike light DOM components, a dual-mode component can use `lwc:dom="manual"` (see next section).

#### Shadow DOM restrictions that apply

`lwc:dom="manual"` must be used for manual DOM operations if a dual-mode component is running in synthetic shadow mode. If not,
an error will be logged (as is currently the case with normal shadow components).

[Scoped slots](https://github.com/salesforce/lwc-rfcs/pull/63) are not allowed, because they are not supported by
shadow DOM components. If a component author uses `lwc:slot-bind` in a dual-mode template, then a compile-time error is thrown
(the same as with normal shadow components).

#### Behavior that differs at runtime between light and shadow

Some behavior will differ at runtime between light DOM mode and shadow DOM mode. Dual-mode components are responsible
for handling these differences themselves.

A non-exhaustive list:

- `::slotted` selectors in CSS are permitted, but will only work when running in native shadow mode.
- `this.template` is undefined for light DOM components.
- Slots are lazy in light DOM and eager in native shadow DOM.
- Scoped and non-scoped stylesheets will both behave differently in light versus shadow mode.

#### Runtime modifications

A component cannot dynamically modify its `static renderMode`; changing it at runtime has no effect.

Due to the design of the `lwc:render-mode` property in this RFC, it will also be impossible for the LWC engine to render
a component in one mode, and then switch to another mode later for the same instance.

In other words, the following example will result in a complete destruction and recreation of the component in question
if the `lwc:if` condition changes:

```html
<template>
    <template lwc:if={useLightDom}>
        <x-component lwc:render-mode="light"></x-component>
    </template>
    <template lwc:else>
        <x-component lwc:render-mode="shadow"></x-component>
    </template>
</template>
```

This relies on the existing behavior of the LWC engine, and does not introduce any new modifications.

## Drawbacks

Implementing this feature adds additional complexity and may create confusion about which is better: shadow mode, light mode,
or dual mode.

It also introduces the possibility that a component author will only test in one mode (light/shadow)
and not realize that their component is broken in another mode.

## Alternatives

None considered.

## Adoption strategy

This is net-new functionality and developers can adopt it incrementally without backwards-compatibility issues.

# How we teach this

In addition to teaching the differences between light and shadow mode (which we already do), we would need to teach
why it might be useful to support dual mode.

In practice, dual mode is probably only going to be needed for specific use cases, namely generic components that
may be reused in a variety of environments, and so most component authors will probably not use it.

# Unresolved questions

None at this time.
