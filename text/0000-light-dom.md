---
title: Light DOM Support
status: DRAFTED
created_at: 2020-09-10
updated_at: 2020-09-10
champion: Philippe Riand (priand), Ted Conn (tconn)
pr: 
---

# RFC: Light DOM Support

## Summary

LWC currently enforces the use of Shadow DOM for every component. This proposal aims to provide a new option, a toggle, which instead lets a component attach its content as children in the Light DOM.

## Basic example

When the Shadow DOM option is turned off for a component, then its content is not attached to its shadow-root, but to the Element itself. Here is an example, showing whenever Shadow DOM is on or off: 

*Shadow DOM*

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

*Light DOM*

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

* **Theming and branding**

    Because of the CSS isolation, theming is made harder. Typically, theming is done via APIs (CSS properties) and/or CSS theming parts (`::part`). Both come with caveats and issues. Theming is critical for consumer apps, where custom branding is a must have. Some new APIs, like `::theme`, are investigated but they won’t be pervasively available before years.
    [Styling is critical to web component reuse, but may prove difficult in practice](https://component.kitchen/blog/posts/styling-is-critical-to-web-component-reuse-but-may-prove-difficult-in-practice)
* **Third party integrations**

    Third party tools need to traverse the DOM, which breaks with Shadow DOM and the existing browser APIs (querySelector, ...). Note that the use of Light DOM fixes for Light DOM components, but for for native Shadow DOM ones if the page contains some.
    * Analytics tools, Personalization platforms, Commerce tools like PriceSpider or Honey...
* **Testing software**

    They faced the same issues than third party tools when it comes to traverse the DOM

Furthermore, **the goal of Shadow DOM is not to enclose the entire application in a single shadow-bound element**. We want to build UIs which are comprised of multiple web components, not UIs which are a single, top-level web component. Web components shouldn’t be the fundamental mechanism for building everything in an app, otherwise they negate the usefulness of standard semantic HTML. This is why we think the current model is fundamentally flawed.
[Image: shadow spectrum (1).png]

### Prior Art

Most of the libraries designed to support Shadow DOM also propose a Light DOM option, with a variety of Shadow DOM features (slots, scoped styles, etc.). It includes:

* [StencilJS](https://stenciljs.com/docs/styling#shadow-dom-in-stencil)
* [LitElement](https://lit-element.polymer-project.org/api/classes/_lit_element_.litelement.html#createrenderroot)
* [MS Fast Element](https://fast.design/docs/fast-element/working-with-shadow-dom#shadow-dom-configuration)
* ...

Or frameworks that are built for Light DOM offer a Shadow DOM option:

* [Angular](https://angular.io/guide/component-styles#view-encapsulation)
* [React](https://github.com/Wildhoney/ReactShadow) (community library)
* [Vue.js](https://github.com/karol-f/vue-custom-element) (community library)

## Detailed design

### Selecting Light DOM vs Shadow DOM

The selection of Light DOM vs Shadow DOM is under the control of the component developer. It is done at the component class level, through a class level directive. Ideally, this directive does pollute the class namespace and thus uses a symbol. It cannot be changed dynamically or by the execution context. The LWC compiler can also use this information to check any issue and warn the developer.

```js
import { LightningElement, ShadowDom } from 'lwc';

export default class MyComponent extends LightningElement {

    static [ShadowDom] = false;
}
```

### Component features when using Light DOM

Some of the LWC component capabilities are directly inherited from Shadow DOM, or emulated by the synthetic-shadow. Despite the use of Light DOM, we’d like to keep these features available, even if their behavior is slightly adapted to the Light DOM:

* **Slots**

    As we mentioned before, the component composition model in LWC is provided by slots. For components _not_ subscribing to Shadow DOM, this will not change.
    
    Because slot behavior is native to the LWC engine, there is little extra work required to support that.

    Also note that slotted content renders lazily in the Light DOM, while native Shadow DOM renders it eagerly. This is actually beneficial as it enables behaviors that are hard to implement using Shadow DOM, like a router as you don’t want all the routes to render when the page is loaded.
* **Scoped Styles**

    Light DOM components can have scoped style assigned to their content. Today, all styles are scoped by default.  The compiler generates some scope attributes to the Elements in the template and attaches a style sheet to the document (or parent shadow).

    However, we may want to consider adding a new mechanism for specifying whether I want my styles to be scoped or not. If I am in a sub-tree of Light DOM components, it might be expected that my styles will cascade to my children by default. I.e. component encapsulation is no longer a thing, so this extends to styles as well.

    That leave us with two options:

    1. Leave the styles scoped to the component as they are today, and they do _not_ cascade to descendants, even across Light DOM boundaries.
    2. Implement a syntax similar to Vue (`<style scoped="true|false">`) which would allow Light DOM components to have a choice.
* **`this.template`**

    Currently `this.template` returns the shadow-root. What _should_ `this.template` return in a Light DOM component?

    1. Returns the component Light DOM instead of the shadow-root. From an API standpoint, it means that it returns an `Element` instead of a `ShadowRoot`.
    2. Returns `null` because `this.template` is the shadow root associated with the component.
    

Also, Light DOM carries some specific behaviors:

* ID conflicts
    Because the component content is not isolated from the rest of the DOM, this can result to Elements’s id conflicts

### $Refs (WIP)
In order to make element querying in the Light DOM less error prone (slots, descendants toggling shadow), we can consider implementing a `$ref` mechanism similar to React or Vue. 

### Security (WIP)
* In some applications, light-dom components may not be allowed... it’s up to the app context
    * **what is the behavior when it’s not allowed?**
    * Some applications might disable light-dom as a whole
    * Some applications might disable light-dom selectively using a “privileged code” model



### Component Migration
There is no migration of the existing components needed. The behaviors of existing components using Shadow DOM remain the same.

### Internal Implementation

Fortunately, most of the code already exists in the LWC core runtime, as it has been implemented to support synthetic shadow. This makes the implementation much easier, and only touching a few code blocks.

**Render Root**
Each LWC component has a VM (View Model) associated to it which carries the component runtime information. The VM class is extended with a new attribute defining if the Shadow DOM is being used:

```typescript
export interface VM<N = HostNode, E = HostElement> {
    
    shadowDom: boolean;
}
```

This attribute is set when the VM is created, which is very early in the component creation. For this, it introspects the component class and reads its `ShadowDom` static member. It defaults to true, which means Shadow DOM enabled, if the member does not exist. If this attribute is not defined at the component class, it eventually uses the one from the parent class, thanks to the chaining prototypes.

Similarly, the `VM.cmpRoot` is either the component shadow root, created calling `attachShadow`, or the element itself:

```typescript
    const cmpRoot = vm.shadowDom ? renderer.attachShadow(elm, ...) : elm
    vm.cmpRoot = cmpRoot;
```

**Scoped Styles**
The synthetic-shadow implementation is reused, where the runtime now checks for either `shadowDom` and `syntheticShadow` where it used to only checks for `syntheticShadow`. For example:

```typescript
function createStylesheet(vm, stylesheets) {
  const { shadowDom, renderer } = vm;
  if (!shadowDom || renderer.syntheticShadow) {
    // some code
```

The DOM engine implementation should be extended to add the scoped styles to the closest `DocumentFragment` (`ShadowRoot` container) in the hierarchy if any. If none, then the styles are added to the document root as it does right now.

**Server Side Rendering**
The engine-server module providing the SSR capability can seamlessly render Shadow DOM or Light DOM. It includes the component children, as well as the scoped styles.

**Synthetic Shadow DOM**
The selection of Light DOM should not be impacted by the use of synthetic shadow instead of native shadow. Now, the goal is to get rid of synthetic shadow, but this is hardly possible today because of:

* Base components do not work with native shadow (https://sfdc-communitycloud.slack.com/archives/C5W3E40TC/p1596736376290800)
* Shadow DOM has accessibility issues

Could we think of a lightweight synthetic shadow that do not override the global methods but provides enough functions to the base components to work while letting third party integration tools work? This can be a different DOM option, which is an hybrid between Shadow DOM and Light DOM.

### POC

More implementation details available through this POC:

* Demo: https://git.soma.salesforce.com/pages/priand/pages-lwc-lightdom
* Git repo: https://github.com/priandsf/lwc/blob/light-dom-1.7.7/packages/sample-app/README.md

### **Potential issues**

**Slot Propagation**
Slot propagation is not available between components using Shadow and Light DOM intermingled.  Typically, the use case below will not work:

```html
<!-- App.html, shadow on/off, doesn't matter -->
<template>
  <Container>
    <div>Slotted content!</div>
  </Container>
</template>

<!-- Container.html, shadow on -->
<template>
  <Comp>
    <slot></slot>
  <Comp>
</template>

<!-- Comp.html, shadow off -->
<template>
  <slot>default content</slot>
</template>
```

This is an edge case that a developer could fix by ensuring that the `Comp` component is actually using Shadow DOM.


## Drawbacks

* Components are not isolated anymore from the rest of the DOM, so it has to be used carefully.
    We have to validate with security when, or not, the feature can be enabled.
* The feature cannot be implemented in the user space, this is core compiler/runtime option
* Some of the work has to be coordinated with the following:
    * On going work on synthetic-shadow
    * Proposal around parts
    * Future work using constructible stylesheet will also have to handle Light DOM
    * Phase #2 of SSR, including client side component rehydration, should handle this option as well

## Design Alternatives

### Light DOM Selection

There are alternate ways to instruct the LWC compiler/runtime to use Light DOM.

**As a component method**

LitElement enables Light DOM via a component method. This method returns the render root, which can be the shadow-root, the current component or anything else:

```js
class LightDom extends LitElement {
  createRenderRoot() {
    return this;
  }
}
```

This is very flexible and allows the creation of [portals](https://reactjs.org/docs/portals.html). But there are more security implications that are out of the of the scope of this RFC. Moreover, it doesn’t tell explicitly if Shadow DOM is in use or not, so this will make the implementation harder (scoped styles, slots, ...), while preventing a static analysis of the code.

**At the application level**

The compiler can use a list of namespaces, defining the list of components that should use Light DOM.

The main drawback of this approach is that the switch is not controlled by the component developer. Even if we try to keep the API behaviors similar when using Light DOM or Shadow DOM, some components will only work in one mode or the other. For example, the design of the HTML template using CSS styles might require Light DOM. So we believe that the use of Light DOM should be under the control of the developer.
On the other hand, we can prevent components using Light DOM to be created contextually: the UI builder doesn’t show them if they are not enabled, and the runtime prevents their instantiation if the context does not allow them.

**At the component instance level**

The component tag can feature a new directive, like `lwc:shadow=“off”,` to instruct the compiler to use the Light DOM for a component instance.

* Similarly to the application level, it doesn’t leave the control to the developers
* It exposes a `lwc:shadow=“off”` directive to the consumer, who might not understand how and when to use it
    This is particularly true for a UI builder user. We don’t want them to be exposed to such a technical option.
* It needs a option for when the component is instantiated outside of a template: `createElement()`, dynamic components, etc.

## Adoption strategy

This new feature does not break any existing components, it simply adds a new feature that developers have to opt for. There is no migration of the existing components needed.

This feature should be exposed and explained to the component library developers as they might change how they develop their components internally.

# How we teach this

Shadow DOM and Light DOM are already names accepted by the industry, see: [Terminology: light DOM vs. shadow DOM](https://developers.google.com/web/fundamentals/web-components/shadowdom?hl=en).
We need to provide the proper documentation to educate the LWC developers:

* What are the differences between Shadow DOM and Light DOM
* We need a guide on when to use one or the other

# Unresolved questions

How much do we want to make the use of Light DOM transparent and similar to Shadow DOM? As the selection of Light DOM is controlled by the developer, it feels ok to have some slight differences. Not only it makes the implementation easier and better performing, but the developer can fully take advantage of the Light DOM when it makes sense. We introduce the Light DOM to expand the power of LWC, not to provide a different implementation of the synthetic-shadow.
Here are some differences:

* Provide a pseudo `ShadowRoot` implementation to make the `this.template` API identical
* Synthetic shadow currently mangle the element ids so they do not conflicts with those in the Light DOM
    Should we do that? I would say no, as we want to behave like the rest of the light DOM and eventually expose elements with a known ID.