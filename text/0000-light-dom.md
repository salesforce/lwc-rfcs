---
title: Light DOM Component
status: DRAFTED
created_at: 2020-09-10
updated_at: 2020-09-10
champion: Philippe Riand (priand), Ted Conn (tconn)
pr: https://github.com/salesforce/lwc-rfcs/pull/44
---

# RFC: Light DOM Component

## Summary

As of today, all the LWC components inheriting from `LightningElement` render their content to the shadow DOM. This proposal introduces a new kind of component which renders its content as children in the Light DOM.

## Basic example

When the Shadow DOM is turned off for a component, its content is not attached to its shadow-root, but to the element itself. Here is an example, showing whenever Shadow DOM is on or off:

_Shadow DOM_

```html
<app-container-blue-shadow>
  #shadow-root (open)
  | <div>
  |   <b>Blue Shadow:</b>
  |   <span class="counter">...</span>
  |   <button type="button">Add one</button>
  | </div>
</app-counter-blue-shadow>
```

_Light DOM_

```html
<app-container-blue-light>
    <div>
      <b>Blue Light:</b>
      <span class="counter">...</span>
      <button type="button">Add one</button>
    </div>
</app-counter-blue-light>
```

As a result, when the content of a component resides in the Light DOM, it can be accessed like any other content in the Document host, and thus behave like any other content (styling, APIs, accessibility, third party tooling...).

## Motivation

Consumer applications require DOM traversal and observability of an application’s anatomy from the document root. Without this, theming becomes hard and 3rd party applications do not run properly:

- **Theming and branding**

  Because of the CSS isolation, theming is made harder. Typically, theming is done via APIs (CSS properties) and/or CSS theming parts (`::part`). Both come with caveats and issues. Theming is critical for consumer apps, where custom branding is a must have. Some new APIs, like `::theme`, are investigated but they won’t be pervasively available before years.
  [Styling is critical to web component reuse, but may prove difficult in practice](https://component.kitchen/blog/posts/styling-is-critical-to-web-component-reuse-but-may-prove-difficult-in-practice)

- **Third party integrations**

  Third party tools need to traverse the DOM, which doesn't work with Shadow DOM and standard browser query APIs such as `querySelector` and `querySelectorAll`. This affects analytics tools, personalization platforms, commerce tools, etc.

- **Testing software**

  Tools like Selenium, Cypress etc. face the same issues as third party tools when it comes to traversing the DOM.

Introducing components rendering to the Light DOM opens new possibilities where applications can render most of their content in the Light DOM, but also keep individual widgets (e.g. `select` tag) and leaf components that render in the Shadow DOM. This approach combines the best of both worlds by solving the issues presented above and offering strong encapsulation when needed.
![shadow spectrum](./shadow-spectrum.png?raw=true "Shadow Spectrum")

### Prior Art

Most of the libraries designed to support Shadow DOM also propose a Light DOM option, with a variety of Shadow DOM features (slots, scoped styles, etc.). It includes:

- [StencilJS](https://stenciljs.com/docs/styling#shadow-dom-in-stencil)
- [LitElement](https://lit-element.polymer-project.org/api/classes/_lit_element_.litelement.html#createrenderroot)
- [MS Fast Element](https://fast.design/docs/fast-element/working-with-shadow-dom#shadow-dom-configuration)

## Detailed design

### `LightningElement.renderMode`

Toggle between light DOM and shadow DOM is done via a new `renderMode` static property on the `LightningElement` constructor. `renderMode` is an enum with `shadow` (default) and `light` as values. When the value is `shadow`, the LightningElement creates a shadow root on the host element and renders the component content in the shadow root. When set to `light` the component renders its content directly in the host element light DOM.

```js
import { LightningElement } from "lwc";

// Example of a shadow DOM component
class ShadowDOMComponent extends LightningElement {
  static renderMode = `shadow`; // can be omitted since it's default
}

// Example of a light DOM component
class LightDOMComponent extends LightningElement {
  static renderMode = `light`;
}
```

The `renderMode` property is looked up by the LWC engine when the component is instantiated to determine how it should render. Changing the value of the `renderMode` static property after the instantiation doesn't influence whether components are rendered in the light DOM or in the shadow DOM.

It also means that developers should be careful not to override the `renderMode` static property value when inheriting from another component. Switching a component mode from shadow DOM to light DOM (and vice-versa) in the child class would certainly break logic in the base class.

```js
import { LightningElement } from "lwc";

class Base extends LightningElement {}

class Child extends Base {
  // ⚠️ Changing the shadow static property value in a component class inheriting from a base
  // will certainly class some unexpected issue.
  static renderMode = `light`;
}
```

### `LightningElement.prototype.template`

When rendering to the shadow DOM, `LightningElement.prototype.template` returns the component associated `ShadowRoot`. When rendering to the light DOM, `LightningElement.prototype.template` value is `null`.

### DOM querying

This proposal doesn't change the way component authors query the light DOM. Component can query their children elements via `LightningElement.prototype.querySelector` and `LightningElement.prototype.querySelectorAll`. It means that turning a shadow DOM component to a light DOM one, all the occurrences of `this.template.querySelector` have to replaced `this.querySelector`.

### Template

Any template associated with a light DOM component should use `lwc:render-mode="light"` to indicate that it is a light DOM template:

```html
<template lwc:render-mode="light">
  <!-- light DOM content -->
</template>
```

Similarly, shadow DOM components can use `lwc:render-mode="shadow"`, but it's not necessary since it's the default value for that attribute.

For light DOM components, this directive is needed to allow the compiler to behave differently for light DOM components versus shadow DOM components. For example, the `lwc:dom` directive is disallowed within light DOM templates (since it's only needed for synthetic shadow DOM).

### Styles

Until now styles used in LWC components were scoped to the component thanks to shadow DOM (or synthetic shadow DOM) style scoping. In the light DOM, component styles naturally leak out of the component; LWC doesn't do any style scoping out of the box. Developers are in charge of writing selectors that are specific enough to target the intended element or pseudo-element.

To support the cases where a shadow DOM element composes a light element, light DOM styles are required to be injected to the closest root node. For a given light DOM element, if all the ancestor components are also light DOM components, the component style sheet will be injected in the document `<head>`. Otherwise, if any of the ancestors is a shadow DOM component, the style has to be injected into the closest shadow root. Upon insertion of a light DOM element, LWC does the following steps:

1. look up for the closest root node (`Node.prototype.getRootNode()`)
1. insert the stylesheet if not already present:
   - if the root node is the HTML document, the style sheet is inserted as a `<style>` element in the `<head>`.
   - if the root node is a shadow root, the stylesheet is inserted as a `<style>` element as the first shadow root child.

> As an optimization, the above algorithm may be substituted by the equivalent usage of [constructable stylesheets](https://developers.google.com/web/updates/2019/02/constructable-stylesheets) in supporting browsers. This means that developers should not rely on the `<style>` elements being inserted at a specific place in the DOM; it's an implementation detail.

It is important to notice that the order in which light DOM components are rendered impacts the order in which stylesheets are injected into the root node and directly influences CSS rule specificity.

#### `:host` and `:host-context()`

Also worth noting is that `:host` and `host-context()` are inserted as-is. Therefore, a light DOM component can style its shadow host. For example:

```css
/* x-light.css */
:host {
  background: blue;
}
```

```html
<x-shadow>
  #shadow-root
  <style>
    :host {
      background: blue;
    }
  </style>
  <x-light></x-light>
</x-shadow>
```

In this case, `<x-light>` is able to style `<x-shadow>` via `:host`. Unlike shadow components, `:host` does not refer to the component but instead to its actual shadow host.

This behavior is according to the specification – `:host` anywhere in the DOM refers to the nearest containing shadow root. At the top (document) level, it does nothing.

#### Synthetic shadow

In the case of synthetic shadow, no attempt is made to scope light DOM styles as described above. The styles are simply inserted as-is, and within a synthetic shadow root they will bleed into the rest of the page, including components using synthetic shadow DOM.

This has some precedent – synthetic shadow always allowed global styles to leak into components (although styles could not leak out of those components). So in a synthetic shadow world, light DOM components' CSS will essentially act like
`<style>`s inserted into the document `<head>`.

#### Scoped styles

Different approaches to layer style scoping on top have been discussed while designing Light DOM, such as introducing a new file extension for automatic style scoping (`.scoped.css`) or using a `<style scoped>` element in the template. This can be addressed in a future RFC.

### Slots

As mentioned before, the component composition model in LWC is provided by slots. Light DOM will provide the same mental model for developers building Light DOM components.

In Light DOM, `<slot>` will denote the place where the slotted component will be attached. The `<slot>` element itself won't be rendered. The slotted content (or the fallback content) will be flattened to the parent element at runtime.

Since the `<slot>` element itself isn't rendered, adding attributes or event listeners to the `<slot>` element in the template will throw a compiler error.

A template cannot contain both light DOM and shadow DOM slots. Inside of `<template lwc:no-shadow>`, all `<slot>`s are light,
and inside of `<template>`, all `<slot>`s are shadow.

In terms of timing, slots in light DOM will behave similarly to the current synthetic shadow slots.

<details>
  <summary>Light DOM vs. shadow DOM timing comparison</summary>
For example:

```html
<!-- x/slottable.html -->
<template lwc:render-mode="light">
  <slot name="foo"></slot>
  <slot></slot>
  <slot name="bar"></slot>
</template>
```

```html
<!-- app.html -->
<template>
  <x-slottable>
    <x-baz></x-baz>
    <x-foo slot="foo"></x-foo>
    <x-bar slot="bar"></x-bar>
  </x-slottable>
</template>
```

(Note that the ordering of the named slots and default slot is different between the slottable component and the component using it.)

This results in the timing:

- `slottable` `constructor`
- `slottable` `connectedCallback`
- `foo` `constructor`
- `foo` `connectedCallback`
- `foo` `renderedCallback`
- `baz` `constructor`
- `baz` `connectedCallback`
- `baz` `renderedCallback`
- `bar` `constructor`
- `bar` `connectedCallback`
- `bar` `renderedCallback`
- `slottable` `renderedCallback`

Note that this timing is different from the equivalent for slots in native shadow DOM:

- `slottable` `constructor`
- `slottable` `connectedCallback`
- `baz` `constructor`
- `baz` `connectedCallback`
- `baz` `renderedCallback`
- `foo` `constructor`
- `foo` `connectedCallback`
- `foo` `renderedCallback`
- `bar` `constructor`
- `bar` `connectedCallback`
- `bar` `renderedCallback`
- `slottable` `renderedCallback`

In native slots, the ordering is based on the order in the _consumer_ component, whereas in synthetic and light DOM slots, it's based on the ordering in the _slotted_ component.

</details>

#### Lazy slots

The main reason for this difference is that synthetic shadow slots and light DOM slots are both _lazy_. For example:

```html
<!-- x/slottable.html -->
<template lwc:render-mode="light">
  <template if:true="{foo}">
    <slot></slot>
  </template>
</template>
```

```js
// slottable.js
import { LightningElement, track } from "lwc";
export default class Slottable extends LightningElement {
  foo = false;
}
```

```html
<!-- app.html -->
<template>
  <x-slottable>
    <x-foo></x-foo>
  </x-slottable>
</template>
```

In this case, `<x-foo>` will never render at all, because it's within a `<template if:true>` block where the condition is false. Similarly, the slot contents would not render if there was no receiving slot for a given slot name, or if the target component contained no slots at all.

These are all differences with native shadow slots, which render eagerly. However, these differences align with the current implementation of synthetic shadow slots. This behavior is maintained in light DOM slots for simplicity's sake in the LWC engine, as well as for the performance benefit of lazy slots.

#### Events and other slot features

The `slotchange` event and `::slotted` CSS pseudo-selector are both unsupported for light DOM slots.

In the case of `slotchange`, it's not clear what `event.target` would be. For `::slotted`, it's not really relevant or useful since light DOM slots can be styled the same as any other light DOM content.

### Security

Shadow DOM (in combination with Lightning Locker) encapsulates components and prevents unauthorized access into the shadow tree. With Light DOM though, the DOM is open for traversal by other components. Since Light DOM is not the default, the component author has to opt-in to it, the burden of security falls on them. Developers need to understand that they are "opening" their component for access from outside when they opt in to Light DOM.

Light DOM will be behind a feature flag that can be set for the runtime. It can be turned on/off on a per container basis. The LWC engine will throw at runtime is a component attempts to render in the light DOM when the feature flag is disabled.

It's important to note that there isn't a way to enable light DOM for a restricted set of namespaces. When the feature flag is turned on, the capability is made available to all components on the page.

### Server Side Rendering

The engine-server module should provide the SSR capability to seamlessly render Shadow DOM or Light DOM. It should include the component children, as well as generated styles.

## Adoption strategy

This new feature does not break any existing components, it simply adds a new feature that developers have to opt for.

There is no automated migration of the existing components. As discussed above, Light DOM components differs from Shadow DOM components from the API and runtime standpoint. If a component author wishes to use Light DOM for any of their existing components, they will have to make the relevant changes manually.

## How we teach this

Shadow DOM and Light DOM are already names accepted by the industry, see: [Terminology: light DOM vs. shadow DOM](https://developers.google.com/web/fundamentals/web-components/shadowdom?hl=en). We need to provide the proper documentation to educate the LWC developers:

- What are the differences between Shadow DOM and Light DOM.
- When to use one compared to the other.

Some tradeoffs we might make explicit to developers:

- In LEX, using light DOM exposes potentially sensitive information to DOM scraping.
- With light DOM, you have to do your own style scoping, e.g. using classes or attributes.
- With light DOM, styles can bleed in or out of the component.
- With light DOM, the slotting mechanism is slightly different than with shadow DOM, and may have performance/timing implications (especially for native shadow DOM).
- With light DOM, there is no event retargeting, so e.g. `<button>`s contained within multiple layers of light DOM components will still trigger `click` events that can be caught at the `document` level, and whose `event.target` is the `<button>` itself rather than the containing component.
- With light DOM, IDs are not scoped to the shadow, so for accessibility purposes, two separate components can reference each other's IDs. For instance, a `<label for="foo">` can reference an `<input type="text" id="foo">` even if the two live in separate components.
- If you have complex, deeply-nested components, you may prefer to have a single shadow component at the top level and light DOM components within, to avoid the overhead of shadows-within-shadows where they're not needed. For instance, a data table component can probably just have one big shadow component around the whole thing, rather than a shadow for every table row and table cell.

## Open questions

**How to deal with different namespaced packages claiming the same custom element name?**

As of today on the Salesforce platform, all the LWC components are authored under the `c` namespace. This namespace dictates the prefix for of the custom elements rendered in the DOM. When delivered via a namespaced package, the components are referenced using their actual namespace (eg. `lightning`, `my_ns`, etc.). However internal component references in the package is done using the `c` namespace.

Because of this if 2 managed packages publish a `button` component, the 2 components will be rendered as `c-button` in DOM. This is problematic because a custom element name can only be bound to a single element constructor. Even if this is not a new issue, the [scoped custom element registry proposal](https://github.com/WICG/webcomponents/issues/716) would solve this issue for shadow DOM components. This is still an issue for light DOM components because scoped custom elements registry can only be attached to a shadow root.
