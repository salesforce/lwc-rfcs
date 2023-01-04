---
title: Dynamic Component Creation
status: DRAFTED
created_at: 2022-12-21
updated_at: 2022-12-21
pr: (leave this empty until the PR is created)
---

# Dynamic Component Creation

## Summary

This RFC supersedes the existing `lwc:dynamic` directive to introduce a new way of creating dynamic components without reusing the custom element name.

Dynamic components refer to custom elements where the constructor is not known at compile time.

*This proposal only focuses on the dynamic component creation process, lazy loading which was included in the [original proposal for dynamic components](https://rfcs.lwc.dev/rfcs/lwc/0110-dynamic-components) is not included.*

## Basic example

```html
<template>
    <div class="container">
        <lwc.dynamic lwc:is={lazyConstructor}></lwc.dynamic>
    </div>
</template>
```

```javascript
import { LightningElement } from 'lwc';

export default class extends LightningElement {
    lazyConstructor;

    async connectedCallback() {
        const { default: ctor } = await import('lightning/concreteComponent');
        this.lazyConstructor = ctor;
    }
}
```

Renders:

Before import completes:

```html
<div class="container"></div>
```

After import completes:

```html
<div class="container">
    <lightning-concrete-component></lightning-concrete-component>
</div>
```

## Motivation

The main issue with the current implementation of the `lwc:dynamic` directive is that it does not respect the 1:1 mapping of the custom element name to constructor in the `CustomElementRegistry`.  This directly conflicts with the [custom elements specification](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-element) and will make adopting the native `CustomElementRegistry` more difficult in the future.

For context, the current design requires a component author define an arbitrary custom element name which will be used as the name for all constructors passed to the `lwc:dynamic` directive, creating a 1:N mapping.

For example:

```html
<template>
  <x-lazy lwc:dynamic={customCtor}></x-lazy>
</template>
```

```javascript
import { LightningElement } from "lwc";

export default class DynamicCtor extends LightningElement {
  customCtor;

  connectedCallback() {
    this.loadCtor();
  }

  async loadCtor() {
    const { default: ctor } = await import("c/customConstructor");
    this.customCtor = ctor;
  }
}
```

Will always render the same tag regardless of the constructor.

```html
  <x-lazy>
    <!-- c-custom-constructor content -->
  </x-lazy>
```

This proposal aims to overcome the issues with the current design by exposing the custom element name at compile time and using it as the name of the dynamic component that is created at runtime.

## Prior art

Several other frameworks have the concept of dynamically rendering components:

- [Svelte: `<svelte:component this={expression}>`](https://svelte.dev/docs#template-syntax-svelte-component)
- [Vue: `<component :is="...">`](https://vuejs.org/guide/essentials/component-basics.html#dynamic-components)
- [Angular: viewContainerRef](https://angular.io/guide/dynamic-component-loader)
- [Lit: static expressions](https://lit.dev/docs/templates/expressions/#static-expressions)
- [Aura: $A.createComponent](https://developer.salesforce.com/docs/atlas.en-us.lightning.meta/lightning/js_cb_dynamic_cmp_async.htm)

Frameworks that use JSX such as [React](https://reactjs.org/docs/rendering-elements.html#rendering-an-element-into-the-dom) and [Stencil](https://stenciljs.com/docs/templating-jsx) can dynamically render components directly into the markup using expressions.

The goal of managing dynamic components in LWC is to follow the same component lifecycle as all other LWC custom elements.  Having said that, taking a similar approach as Svelte and Vue seems the most natural and intuitive path forward.

## Detailed design

A new property, `lightningElementName` on `LightningElementConstructor` is introduced to store the name of the custom element.

A special tag, `<lwc.dynamic>` along with a new template directive `lwc:is` are also introduced and serves as the anchor in the DOM where the dynamic component will be rendered.

Broken down, there are three main parts to this design:

1. Storing and retrieving a custom element's name.
2. Defining an anchor in the DOM.
3. Instantiating the dynamic custom element.

#### Storing and retrieving a custom element's name

In today's world, an LWC module's namespace and name are known at compile time by [examining the directory structure](https://rfcs.lwc.dev/rfcs/lwc/0101-module-unification) and through [module resolution](https://rfcs.lwc.dev/rfcs/lwc/0020-module-resolution).

The default folder structure for an LWC module looks as follows:

```
namespace/
└── name/
    ├── *.css
    ├── *.js
    ├── *.html
```

Leveraging this, the namespace and name can be accessed at compile time and ultimately stored on the constructor.  A new read-only property on `LightningElementConstructor` called `lightningElementName` will store the custom element name which will be of the form `namespace-name`.

```javascript
const { default: ctor } = await import("c/customConstructor");
// The tag name can be directly accessed on the constructor
ctor.lightningElementName
```

The value of `lightningElementName` should only be set by the LWC engine to prevent component authors from modifying it and producing unintended side effects.

Alternatively, the custom element name can be stored in a `weakMap` and a public API can be exposed to retrieve the constructor's custom element name. However, because the custom element name can be used as a selector with `this.template.querySelector` placing it on the constructor seems more ergonomic.

#### Defining an anchor in the DOM

##### `<lwc.dynamic>` tag

`<lwc.dynamic>` is a placeholder tag for the dynamic component and will not be rendered to the DOM.  It serves as a special signal to the compiler that the dynamic component will be rendered at the specific location in the DOM. (Similar to how `template` is not rendered)

Note the separation of the namespace and name using a '.' is intentional because of the following reasons:

1. It is neither a valid [tag name](https://html.spec.whatwg.org/multipage/syntax.html#syntax-tag-name) nor a valid [custom element name](https://html.spec.whatwg.org/multipage/custom-elements.html#valid-custom-element-name) which should clearly signal that this is a special element (which will not be rendered to the DOM).
2. Initially considered using a ':' instead of a '.' but ':' is the delimiter Aura uses to separate namespace and name.  To prevent confusion on whether this is an Aura tag, decided to use '.' instead.

Additionally, `<lwc.dynamic>` is intended to be used in tandem with `lwc:is`, otherwise `<lwc:dynamic>` serves no purpose. A warning should be issued when `<lwc.dynamic>` is used without the `lwc:is` template directive.

##### `lwc:is` directive

A new template directive called `lwc:is` will be used to signal to compiler that the constructor is unknown at compile time. 

The directive has the following properties:

1. It can only be used with `<lwc.dynamic>` tag.
    - An error will be issued when `lwc:is` is used on any other element.
2. At compile time the value supplied to `lwc:is` can only be an expression.
3. At runtime the expression supplied to `lwc:is` must be a `LightningElementConstructor`.
    - An error will be thrown if anything other than a `LightningElementConstructor` is passed to `lwc:is`.

#### Instantiation of the custom element

Dynamic component instantiation follows the same creation path as any other LWC custom element. When the dynamic component is ready to be rendered, the element will be created and mounted to the DOM.  If there is an existing element supplied to `lwc:is`, it will be unmounted as well.

This means that when the value provided to `lwc:is` changes the component and all of its children will be destroyed and recreated using the new constructor.  In addition, when the value provided to `lwc:is` is evaluated, the LWC engine will render depending on the following scenarios:

- **The constructor is `undefined` or `null`**
    - When the constructor is undefined the `<lwc.dynamic>` tag along with all of its children are not rendered to the DOM.
- **The constructor is defined and is not a `LightningElementConstructor`**
    - When the value provided to `lwc:is` is not a `LightningElementConstructor` then an error is thrown indicating the constructor value must be `LightningElementConstructor`.
- **The constructor is defined and is a `LightningElementConstructor`**
    - When the constructor is a `LightningElementConstructor`, the component will be rendered in place of `<lwc.dynamic>`. The tag name used for the dynamic component is the value of `lightningElementName` on `LightningElementConstructor`.

_Note in the case of lazy loading, the component author is responsible for resolving the promise.  The only value that can be given to `lwc:is` is a `LightningElementConstructor`._

#### Selecting the dynamic component

It is important to note that `<lwc.dynamic>` will not be rendered to the DOM.  As such, it is not available to be selected using `this.template.querySelector`.  The actual custom element must be selected once it has been rendered to the DOM.  The dynamic component can be selected either through the `lightningElementName` property on the constructor or through another attribute assigned to the component.

Component authors can use the `connectedCallback` of the dynamic component to signal when the component has been rendered to the DOM.

#### Assigning attributes and event listeners

Attributes and event listeners can be passed to the `<lwc.dynamic>` tag declaratively or can be assigned directly to the custom element imperatively.

Attributes and event listeners assigned declaratively will be assigned to the dynamic component once it has been created.

Declaratively:

```html
<template>
    <lwc.dynamic lwc:is={ctor} class="container" onclick={handleClick}></lwc.dynamic>
</template>
```

Imperatively:

```javascript
import { LightningElement } from "lwc";

export default class DynamicCtor extends LightningElement {
  customCtor;

  connectedCallback() {
    this.loadCtor();
  }

  async loadCtor() {
    const { default: ctor } = await import("c/customConstructor");
    this.customCtor = ctor;
    this.template.querySelector(ctor.lightningElementName).setAttribute('title', 'hawaii');
    this.template.querySelector(ctor.lightningElementName).addEventListener('click', () => console.log('hello world'));
  }
}
```
#### Usage with other LWC directives

All [LWC custom element directives](https://lwc.dev/guide/reference#directives-for-nested-templates) with the exception of `lwc:external` are available to use on `<lwc.dynamic>` and their functionality will be passed through to the custom element once it has been created.

`lwc:external` will not be available because the constructor provided to the `lwc:is` directive must be a `LightningElementConstructor`.

#### Children of the dynamically created element

The `<lwc.dynamic>` tag accepts child elements, first creating the dynamic component and subsequently the children afterwards.  Each time the dynamic component changes, the existing dynamic component will be unmounted along with all of its children.  The new dynamic component will be created along with the children.

```html
<template>
    <lwc.dynamic lwc:is={ctor}>
        <span>child</span>
    </lwc.dynamic>
</template>
```

#### Light DOM vs shadow DOM

A dynamic component can switch between light DOM and shadow DOM and should follow the same semantics as currently exists with other forms of dynamic rendering components.

For example

```html
<template>
    <template lwc:if={renderLight}>
        <light-component></light-component>
    <template lwc:else>
        <shadow-component></shadow-component>
    </template>
</template>
```

Is semantically the same as

```html
<template>
    <!-- The ctor swaps between light and shadow DOM components -->
    <lwc.dynamic lwc:is={ctor}></lwc.dynamic>
</template>
```

In both cases the dynamic component will be fully mounted and unmounted.

#### Styles

Something to keep in mind is that [light DOM styles are injected to the closest root node](https://rfcs.lwc.dev/rfcs/lwc/0115-light-dom#styles) and are not removed once the components are unmounted. This means that repeated mounting and mounting of a dynamic component could cause styles to be overwritten.  This can be avoided by using either scoped styles or shadow dom, as the styles will be scoped.

## Drawbacks

The main drawback of this design is that when the constructor changes, the entire component hierarchy will need to be re-rendered. This means that the children of the custom element will also need to re-rendered.  Careful consideration should be taken by the component author as this may cause performance issues.

## Alternatives

### Use a `<template>` tag instead of `lwc.dynamic`.

Reference to [POC](https://github.com/salesforce/lwc/pull/3218)

Decided against this approach because the LWC team has placed restrictions on being able to add attributes to `<template>` as it does not have any semantic meaning if no LWC directives are associated. In the past the LWC team has seen component authors attempt to write components wrapped in `<template>` believing it would be rendered to the DOM.

For example the following will not render in the DOM:

```html
<template>
    <template class="slds-border">
        <span>hello world</span>
    </template>
</template>
```

As a result the team has placed restrictions on non-root `<template>` tags to discourage component authors from confusing it as an HTML element that will be rendered in the DOM. Because dynamic components should allow attributes and event listeners to be assigned declaratively this is may further confuse component authors about how to use `<template>` element.

### LightningDynamicElement

Full details can be found [here](https://github.com/salesforce/lwc/pull/3204)

#### Brief summary

`LightningDynamicElement` is a special custom element provided by LWC that serves as a wrapper component around dynamic LWC custom elements. `LightningDynamicElement` tackles the issue of reusing the same custom element with multiple definitions by serving as a wrapper around dynamic components that renders their content to the DOM.

`LightningDynamicElement` works by synthetically handling the lifecycle of the dynamic component, which means the dynamic component does not go through the same mounting and unmounting process as other custom elements.  Additionally, `LightningDyanmicElement` does not unmount when the dynamic constructor changes.

_This is similar to the [Angular approach to dynamic components](https://angular.io/guide/dynamic-component-loader)._

#### Comparison with `<lwc.dynamic lwc:is={expression}>`

With `LightningDynamicElement` the framework keeps track of references to the dynamic components and also manages its lifecycle.  In contrast with `<lwc.dynamic>` the component author is responsible for keeping track of the custom element.  This means that any dynamic component must be retrieved each time the component changes.  Additionally, any imperatively assigned attributes or event listeners will need to be reapplied once the component changes as well.

## Adoption strategy

This is a breaking change for existing user of `lwc:dynamic` as it supersedes the directive.  However, `lwc:dynamic` directive has not reached GA and all consumers of the directive are internal.  They will need to switch their components to change the tag names to `lwc.dynamic` and the directives from `lwc:dynamic` to `lwc:is`.

The behavior of `lwc:is` is the same as `lwc:dynamic` when the constructor is not provided, both will not render the dynamic component of its children. The behavior is different when the components are rendered though, as `lwc:is` will render the custom element name rather than the custom element name used with `lwc:dynamic`.  Component authors may need to adjust any testing that relies on this.

# How we teach this

This concept is fairly intuitive and is similar to the same concepts in other frameworks (Svelte, Vue, Angular).  Providing documentation around how the to use `<lwc.dynamic>` and `lwc:is` with a small playground example should be sufficient.

Examples of [documentation](https://svelte.dev/docs#template-syntax-svelte-component) and [playground](https://svelte.dev/tutorial/svelte-component) from Svelte.

# Unresolved questions

Will there be issues with collisions?  For example if a dynamic constructor is imported but has the same custom element name as a custom element that's already on the page?
    - Does the `upgradeableCallback` take care of this?