---
title: Light DOM Support
status: DRAFTED
created_at: 2020-09-10
updated_at: 2020-09-10
champion: Philippe Riand (priand), Ted Conn (tconn)
pr: https://github.com/salesforce/lwc-rfcs/pull/44
---

# RFC: Light DOM Support

## Summary

LWC currently enforces the use of Shadow DOM for every component. This proposal aims to provide a new option, a toggle, which instead lets a component attach its content as children in the Light DOM.

## Basic example

When the Shadow DOM option is turned off for a component, then its content is not attached to its shadow-root, but to the Element itself. Here is an example, showing whenever Shadow DOM is on or off:

_Shadow DOM_

```html
<app-container-blue-shadow>
  ▼ #shadow-root (open)
      <div>
        <b>Blue Shadow:</b>
        <span class="counter">...</span>
        <button type="button">Add one</button>
      </div>
</app-counter-blue-shadow>
```

_Light DOM_

```html
<app-container-blue-light>
  ▼ <div>
      <b>Blue Light:</b>
      <span class="counter">...</span>
      <button type="button">Add one</button>
    </div>
</app-counter-blue-light>
```

As a result, when the content of a component lives in its children, it can be accessed like any other content in the Document host, and thus behave like any other content (styling, APIs, accessibility, third party tooling...).

Shadow DOM provides a wonderful, native component encapsulation and composition model.

The two main reasons for using Shadow DOM are:

**Native Component Encapsulation**

The Shadow DOM encapsulation model provides a way for the component author to keep the component internals hidden, with no effect from the broader environment. This means that global styles have no effect on those internals and those internals cannot be queried from outside of the component. This provides a way for widget-type components to be more portable (eg: embedding into 3rd party apps) at the cost of some complexity that makes the creation of a component or an application harder.

**Native Component Composition**

The Shadow DOM gives custom element developers a way to allow consumers to compose some content that it should render. Think about native elements like `<select>`. This is done via `<slot>` elements that are automatically filled by the content from the light DOM. Without Shadow DOM there is no native component composition and this functionality must be provided by the framework.

> A single component that needs to stand on its own with its own set of functionality is a good candidate for shadow DOM. Salesforce Lightning components are a good example of that. While one or more components as part of an application might not need Shadow DOM, as their intended use is much clearer and their markup less fragile.

## Motivation

Consumer applications require DOM traversal and observability of an application’s anatomy from the document root. Without this, theming becomes hard while 3rd party applications do not run properly:

- **Theming and branding**

  Because of the CSS isolation, theming is made harder. Typically, theming is done via APIs (CSS properties) and/or CSS theming parts (`::part`). Both come with caveats and issues. Theming is critical for consumer apps, where custom branding is a must have. Some new APIs, like `::theme`, are investigated but they won’t be pervasively available before years.
  [Styling is critical to web component reuse, but may prove difficult in practice](https://component.kitchen/blog/posts/styling-is-critical-to-web-component-reuse-but-may-prove-difficult-in-practice)

- **Third party integrations**

  Third party tools need to traverse the DOM, which breaks with Shadow DOM and the existing browser APIs (querySelector, ...). Note that the use of Light DOM fixes for Light DOM components, but for for native Shadow DOM ones if the page contains some.

  - Analytics tools, Personalization platforms, Commerce tools like PriceSpider or Honey...

- **Testing software**

  They faced the same issues than third party tools when it comes to traverse the DOM

Furthermore, **the goal of Shadow DOM is not to enclose the entire application in a single shadow-bound element**. We want to build UIs which are comprised of multiple web components, not UIs which are a single, top-level web component. Web components shouldn’t be the fundamental mechanism for building everything in an app, otherwise they negate the usefulness of standard semantic HTML. This is why we think the current model is fundamentally flawed.
![shadow spectrum](./shadow-spectrum.png?raw=true "Shadow Spectrum")

### Prior Art

Most of the libraries designed to support Shadow DOM also propose a Light DOM option, with a variety of Shadow DOM features (slots, scoped styles, etc.). It includes:

- [StencilJS](https://stenciljs.com/docs/styling#shadow-dom-in-stencil)
- [LitElement](https://lit-element.polymer-project.org/api/classes/_lit_element_.litelement.html#createrenderroot)
- [MS Fast Element](https://fast.design/docs/fast-element/working-with-shadow-dom#shadow-dom-configuration)
- ...

Or frameworks that are built for Light DOM offer a Shadow DOM option:

- [Angular](https://angular.io/guide/component-styles#view-encapsulation)
- [React](https://github.com/Wildhoney/ReactShadow) (community library)
- [Vue.js](https://github.com/karol-f/vue-custom-element) (community library)

## Detailed design

### Selecting Light DOM vs Shadow DOM

The selection of Light DOM vs Shadow DOM is under the control of the component developer. It is done at the component class level, by inheriting a different base class (`MacroElement`, not `LightningElement`). The LWC compiler can also use this information to check any issue and warn the developer.

```js
import { MacroElement } from "lwc";

export default class MyComponent extends MacroElement {}
```

### Component features when using Light DOM

Some of the LWC component capabilities are directly inherited from Shadow DOM, or emulated by the synthetic-shadow. Despite the use of Light DOM, we’d like to keep these features available, even if their behavior is slightly adapted to the Light DOM:

- **Slots**

  As we mentioned before, the component composition model in LWC is provided by slots. Light DOM will provide the same mental model for developers building Light DOM components.

  In Light DOM, `<slot>` will denote the place where the slotted component will be attached. The `<slot>` element itself won't be rendered. The slotted content (or the placeholder content) will be flattened to the parent element at runtime.

  Since the `<slot>` element itself isn't rendered, adding any event listeners to the `<slot>` element in the template will throw a compiler error.

- **Scoped Styles**

  Shadow DOM styles are scoped to the component they belong. In Native shadow, it is enforced by the browser while in Synthetic shadow the styles are scoped by some attributes which are also added to the element at runtime.

  In Light DOM, the styles will be scoped not just to the component they belong, but also to its children. This is similar to how Aura styles its components.

  E.g.

  ```javascript
  <x-a>
    <style>
      p { color: blue }
    </style>

    <p item=1></p>

    <x-b>
      <style>
        p { color: red }
      </style>

      <p item=2></p>
      <p item=3></p> <!-- This paragraph is coming from A and slotted into B -->
    </x-b>
  </x-a>
  ```

  In this case, item 1 will be in blue and item 2 and 3 will be in red. Also note that in `<x-a>`, we can write a selector of higher specificity `x-b p` to override the styles in `x-b`.

  Light DOM components can have scoped style assigned to their content. Today, all styles are scoped by default. The compiler generates some scope attributes to the Elements in the template and attaches a style sheet to the document (or parent shadow).

  Global scoping is not supported. If desired, a global stylesheet can be injected manually.

- **`this.template`**

  In `LightningElement`, `this.template` returns the shadow-root. It will return `null` in `MacroElement`.

### Security (WIP)

- In some applications, light-dom components may not be allowed... it’s up to the app context
  - **what is the behavior when it’s not allowed?**
  - Some applications might disable light-dom as a whole
  - Some applications might disable light-dom selectively using a “privileged code” model

### Component Migration

There is no migration of the existing components needed. The behaviors of existing components using Shadow DOM remain the same.

### Synthetic Shadow DOM

The selection of Light DOM should not be impacted by the use of synthetic shadow instead of native shadow. Now, the goal is to get rid of synthetic shadow, but this is hardly possible today because of:

- Salesforce Lightning Components do not work with native shadow today
- Shadow DOM has accessibility issues

Could we think of a lightweight synthetic shadow that do not override the global methods but provides enough functions to the base components to work while letting third party integration tools work? This can be a different DOM option, which is an hybrid between Shadow DOM and Light DOM.

## Internal Implementation

Fortunately, most of the code already exists in the LWC core runtime, as it has been implemented to support synthetic shadow. This makes the implementation much easier, and only touching a few code blocks.

### Class Structure

LWC will be refactored into a base class `BaseLightningElement` that will be used throughout the `lwc` codebase. `LightningElement` class will inherit `BaseLightningElement` and will have only Shadow DOM specific logic. For example, `renderer.attachShadow(elm, ...)` will only exist in `LightningElement`.

`MacroElement` will also inherit `BaseLightningElement` and will have light dom specific logic. For example, `this.template` will return `null`.

Developers can't inherit from `BaseLightningElement` directly - compiler will prevent that from happening.

### View Model

Each LWC component has a VM (View Model) associated to it which carries the component runtime information. The VM class has an attribute `cmpRoot` which is of type `ShadowRoot`. We'll have to change it to `ShadowRoot | null` to denote that it will be null in case of `MacroElement`.

```typescript
export interface VM<N = HostNode, E = HostElement> {
  cmpRoot: ShadowRoot | null;
}
```

This can also be used to check whether a `vm` is associated with a Shadow DOM element or a Light DOM element.

### Template

Template compiler, when compiling a template that belongs to a Light DOM component, should throw compiler errors when event listeners are found on `slot` elements or when `lwc:dom="manual"` directive is found.

Currently, the template compiler has no context of the Javascript file and thus can't know whether the template being compiled is a Light DOM component template or a Shadow DOM component template.

To solve the above problem, the template in a Light DOM component, will have a special `macro` attribute at the root `<template>` tag.

```html
<template macro>
  <h1>My Light DOM element</h1>

  <slot onslotchange="{handleSlotChange}">
    <!-- throws error -->
    <div lwc:dom="manual"></div>
    <!-- throws error -->
  </slot>
</template>
```

In the presence of the `macro` attribute, the template-compiler will do additional validations. Currently, there doesn't seem to be a need to modify the compiler output in anyway.

### Scoped Styles

Style scoping will be done at the host level. The host will have a special attribute (similar to the one that is used in Synthetic shadow) and that attribute selector will be used to prefix all the CSS selectors in the style.

`:host` and `:host()` will also be replaced with the host token.

e.g. for the following CSS,

```css
p {
  color: red;
}
```

the output of style-compiler will be (note the addition of `macroSelector`)

```js
function stylesheet(hostSelector, shadowSelector, nativeShadow, macroSelector) {
  return [macroSelector, "p", shadowSelector, " {color: red;}"].join("");
}
export default [stylesheet];
```

the CSS output of which when evaulated as follows:

```js
const macroSelector = isMacroElement ? `[${tokens.hostAttribute}]` : '';

content = evaluateStylesheetsContent(
    stylesheets,
    hostSelector,
    shadowSelector,
    !syntheticShadow,
    macroSelector
```

will result in

```css
[x-test_test_a] p {
  color: red;
}
```

Runtime will also add the attribute `x-test_test_a` to the host element here:

```js
if (renderer.syntheticShadow || !cmpRoot) {
  updateSyntheticShadowAttributes(vm, html);
}
```

### Slotting

Slotting will mostly reuse synthetic shadow slotting. However, there is one major difference: the slot element itself won't be rendered.

Instead of the slot, two empty text nodes will be inserted to denote the start and end of the slot. This can be done by introducing a new `VNode` called `VFakeSlot`:

```js
export interface VFakeSlot extends VElement {
  start: VText;
  end: VText;
}
```

Creating a node for `VFakeSlot` will mean creating the two text nodes. Adding/removing children to the slot will result in adding or removing children between the two text nodes.

For example:

```js
// updating dynamic children
const endElm = vnode.end.elm!;
const parentElm = endElm.parentNode!;
updateDynamicChildren(parentElm, oldCh, newCh, endElm); // inserted before endElm

// in updating static children
const startIndex = Array.prototype.indexOf.call(parentElm, startElm);
addVnodes(parentElm, null, newCh, startIndex, newChLength);
```

`updateDynamicChildren` here is the same function that is used for synthetic shadow.

### Server Side Rendering

The engine-server module should provide the SSR capability to seamlessly render Shadow DOM or Light DOM. It should include the component children, as well as the scoped styles.

### POC

More implementation details available through this POC:

- Git repo: [https://github.com/salesforce/lwc/tree/abdulsattar/spikes/light-dom](https://github.com/salesforce/lwc/tree/abdulsattar/spikes/light-dom)

**Earlier POC by Philippe Riand**

- Demo: [https://priandsf.github.io/lwc-light-dom-static](https://priandsf.github.io/lwc-light-dom-static)
- Git repo: [https://github.com/priandsf/lwc/blob/light-dom-1.7.7/packages/sample-app/README.md](https://github.com/priandsf/lwc/blob/light-dom-1.7.7/packages/sample-app/README.md)

## Adoption strategy

This new feature does not break any existing components, it simply adds a new feature that developers have to opt for. There is no migration of the existing components needed.

This feature should be exposed and explained to the component library developers as they might change how they develop their components internally.

## How we teach this

Shadow DOM and Light DOM are already names accepted by the industry, see: [Terminology: light DOM vs. shadow DOM](https://developers.google.com/web/fundamentals/web-components/shadowdom?hl=en).
We need to provide the proper documentation to educate the LWC developers:

- What are the differences between Shadow DOM and Light DOM
- We need a guide on when to use one or the other
