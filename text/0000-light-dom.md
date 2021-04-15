---
title: Light DOM Support
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

  Third party tools need to traverse the DOM, which breaks with Shadow DOM and the existing browser APIs (querySelector, ...). Note that the use of Light DOM fixes for Light DOM components, but not for native Shadow DOM ones if the page contains any.

  - Analytics tools, Personalization platforms, Commerce tools like PriceSpider or Honey...

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

### Rendering to the Light DOM

Toggle between light DOM and shadow DOM is done via a new `shadowDOM` static property on the `LightningElement` constructor. This accepts a `boolean` value and is `true` by default. When set to `true`, the LightningElement creates a shadow root on the host element and renders the component content in the shadow root. When set to `false` the component renders its content directly in the host element light DOM.

```js
import { LightningElement } from "lwc";

// Example of a shadow DOM component
class ShadowDOMComponent extends LightningElement {}

// Example of a light DOM component
class LightDOMComponent extends LightningElement {
  static shadowDOM = false;
}

// Default value
console.log(LightningElement.shadowDOM); // true
```

The `shadowDOM` property is looked up by the LWC engine when the component is instantiated for the first time and is cached for future instantiation. Changing the value of the `shadowDOM` static property after the first instantiation doesn't influence whether components are rendered in the light DOM or in the shadow DOM.

It also means that developers should be careful not to override the `shadowDOM` static property value when inheriting from another component. Switching a component mode from shadow DOM to light DOM (and vice-versa) in the child class would certainly break logic in the base class.

```js
import { LightningElement } from "lwc";

class Base extends LightningElement {}

class Child extends Base {
  // ⚠️ Changing the shadowDOM static property value in a component class inheriting from a base
  // will certainly class some unexpected issue.
  static shadowDOM = false;
}
```

### Component features when using Light DOM

Some of the LWC component capabilities are directly inherited from Shadow DOM, or emulated by the synthetic-shadow. Despite the use of Light DOM, we’d like to keep these features available, even if their behavior is slightly adapted to the Light DOM:

#### Slots

As we mentioned before, the component composition model in LWC is provided by slots. Light DOM will provide the same mental model for developers building Light DOM components.

In Light DOM, `<slot>` will denote the place where the slotted component will be attached. The `<slot>` element itself won't be rendered. The slotted content (or the fallback content) will be flattened to the parent element at runtime.

Since the `<slot>` element itself isn't rendered, adding attributes or event listeners to the `<slot>` element in the template will throw a compiler error.

Due to the above differences from regular `<slot>`, LWC compiler will enforce the presence of `<slot shadow="false">` on Light DOM slots to make it explicit to the user that these are different slots.

#### Styles

Until now styles used in LWC components were scoped to the component thanks to shadow DOM (or synthetic shadow DOM) style scoping. In the light DOM, component styles naturally leak out of the component; LWC doesn't do any style scoping out of the box. Developers are in charge of writing selectors that are specific enough to target the intended element or pseudo-element.

To support the cases where a shadow DOM element composes a light element, light DOM styles are required to be injected to the closest root node. For a given light DOM element, if all the ancestor components are also a light DOM components, the component style sheet will be injected in the document `<head>`. Otherwise, if any of the ancestors is a shadow DOM component, the style has to be injected in the closest shadow root. Upon insertion of a light DOM element, LWC does the following steps:

- look up for the closest root node (`Node.prototype.getRootNode()`)
- insert the stylesheet if not already present:
  - if the root node is the HTML document, the style sheet is inserted as a `<style>` element in the `<head>`.
  - if the root node is a shadow root, the stylesheet is inserted as a `<style>` element as the first shadow root child.

It is important to notice that the order in which light DOM components are rendered impact the order in which stylesheets are injected in the root node and directly influences CSS rule specificity.

**Note:** Different approaches to layer style scoping on top has been discussed while designing Light DOM, like introducing a new file extension for automatic style scoping (`.scoped.css`) or using a `<style scoped>` element in the template. This can be addressed in a future RFC.

#### `LightningElement.prototype.template`

When rendering to the shadow DOM, `LightningElement.prototype.template` returns the component associated `ShadowRoot`. When rendering to the light DOM, `LightningElement.prototype.template` value is `null`.

#### DOM querying

This proposal doesn't change the way component authors query the light DOM. Component can query their children elements via `LightningElement.prototype.querySelector` and `LightningElement.prototype.querySelectorAll`.

> It means that turning a shadow DOM component to a light DOM one, all the occurrences of `this.template.querySelector` have to replaced `this.querySelector`.

### Security

Shadow DOM (in combination with Locker) encapsulates components and prevents unauthorized access into the shadow tree. With Light DOM though, the DOM is open for traversal by other components.

Since Light DOM is not the default, the component author has to opt-in to it, the burden of security falls on them. They need to understand that they are "opening" their component for access from outside when they opt in to Light DOM.

Light DOM will be behind a feature flag that can be set for the runtime. It can be turned on/off by the container.

- **What is the behavior when it’s not allowed?**
  - It will fallback to rendering in Shadow DOM
- Some applications might disable light-dom as a whole
  - It can be disabled by turning the flag off.
- Some applications might disable light-dom selectively using a “privileged code” model
  - Not supported. It can be either turned on/off on the whole.

### Component Migration

There is no automated migration of the existing components. As discussed above, Light DOM components differs from Shadow DOM components from the API and runtime standpoint. If a component author wishes to use Light DOM for any of their existing components, they will have to make the relevant changes manually.

### Server Side Rendering

The engine-server module should provide the SSR capability to seamlessly render Shadow DOM or Light DOM. It should include the component children, as well as generated styles.

## Adoption strategy

This new feature does not break any existing components, it simply adds a new feature that developers have to opt for. There is no migration of the existing components needed.

This feature should be exposed and explained to the component library developers as they might change how they develop their components internally.

## How we teach this

Shadow DOM and Light DOM are already names accepted by the industry, see: [Terminology: light DOM vs. shadow DOM](https://developers.google.com/web/fundamentals/web-components/shadowdom?hl=en). We need to provide the proper documentation to educate the LWC developers:

- What are the differences between Shadow DOM and Light DOM
- We need a guide on when to use one or the other

## Open questions

**How to deal with different namespaced packages claiming the same custom element name?**

As of today on the Salesforce platform, all the LWC components are authored under the `c` namespace. This namespace dictates the prefix for of the custom elements rendered in the DOM. When delivered via a namespaced package, the components are referenced using their actual namespace (eg. `lightning`, `my_ns`, etc.). However internal component references in the package is done using the `c` namespace.

Because of this if 2 manage package publish a `button` component, the 2 components will be rendered as `c-button` in DOM. This is problematic because a custom element name can only be bound to a single element constructor. Even if this is not a new issue, the [scoped custom element registry proposal](https://github.com/WICG/webcomponents/issues/716) would solve this issue for shadow DOM components. This is still an issue for light DOM component as scoped custom elements registry can only be attached to a shadow root.
