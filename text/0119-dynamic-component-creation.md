---
title: Dynamic Component Creation
status: DRAFTED
created_at: 2022-12-21
updated_at: 2023-01-04
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
        <lwc:component lwc:is={lazyConstructor}></lwc:component>
    </div>
</template>
```

```javascript
import { LightningElement } from 'lwc';

export default class extends LightningElement {
    lazyConstructor;

    connectedCallback() {
        import('lightning/concreteComponent')
            .then(({ default: ctor }) => this.lazyConstructor = ctor)
            .catch(err => console.log('Error importing component'));
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
    <button onclick={loadCtor}>Click to change constructor</button>
    <x-lazy lwc:dynamic={customCtor}></x-lazy>
</template>
```

```javascript
import { LightningElement } from "lwc";
import About from 'c/about';
import Home from 'c/home';

const ctors = [About, Home]; 

export default class extends LightningElement {
    customCtor;
    index = 0;

    loadCtor() {
        const nextCtor = ctors[this.index++ % ctors.length];
        this.customCtor = nextCtor;
    }
}
```

Will always render the same tag regardless of the constructor.

```html
<button onclick={loadCtor}>Click to change constructor</button>
<x-lazy>
    <!-- c-custom-constructor content -->
</x-lazy>
```

This proposal aims to overcome the issues with the current design by resolving the custom element name at compile time and using it as the name of the dynamic component that is created at runtime.

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

A new internal `Map` called `componentRegisteredNameMap` is introduced to store the custom element name and an internal API `getComponentRegisteredName` is introduced to retrieve the custom element name.

A special tag, `<lwc:component>` along with a new template directive `lwc:is` are also introduced and serves as the anchor in the DOM where the dynamic component will be rendered.

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

The LWC module resolver also allows for namespace and name mappings that do not adhere to the default folder structure, such as the case with [alias module records](https://rfcs.lwc.dev/rfcs/lwc/0020-module-resolution#aliasmodulerecord).

Leveraging this, the namespace and name can be resolved at compile time and used to construct the custom element name, which will be in the form `namespace-name`.  

##### `componentRegisteredNameMap`

The custom element name will be stored on an internal `Map` called `componentRegisteredNameMap` where the key is the `LightningElementConstructor` and the value is the custom element name.

At compile time, LWC will inject a call to `registerComponent` providing the custom element name.  At runtime `registerComponent` will associate the custom element name to `componentRegisteredNameMap` in a similar way to how templates are associated to `LightningElementConstructor`.

If the custom element name is unable to be resolved at compile time and no value is provided to `registerComponent` the compiler will report an error.

##### `getComponentRegisteredName`

In contrast, the custom element name can be retrieved through an internal API called `getComponentRegisteredName`.  The LWC engine will use this API to retrieve the name when the dynamic component is ready to be instantiated.

_Note `getComponentRegisteredName` should be the only way to retrieve the custom element name._

The `componentRegisteredNameMap`, `getComponentRegisteredName`, and custom element name are internal APIs that should only be used by the LWC engine and will not be observable to component authors.  This is to prevent any unintended side effects.

_See the [Selecting the dynamic component](#Selecting-the-dynamic-component) for details on how to access the component once it has been instantiated._

##### Considerations

There is a possibility that a constructor is mapped to more than one alias, such as:

```javascript
{
    "modules": [
       {
            "name": "ui/button",
            "path": "src/modules/ui/button/button.js"
        },
        {
            "name": "ui/button2",
            "path": "src/modules/ui/button/button.js"
        }
    ]
}
```

Since there is no way to know which alias to use in this case, the custom element name will be the first alias that is resolved.  This should be an edge case that does not appear often.

#### Defining an anchor in the DOM

##### `<lwc:component>`

`<lwc:component>` is a placeholder tag for the dynamic component and will not be rendered to the DOM.  It serves as a special signal to the compiler that the dynamic component will be rendered at the specific location in the DOM. (Similar to how `<template>` is not rendered)

Note the separation of the namespace and name using a ':' is intentional. It is neither a valid [tag name](https://html.spec.whatwg.org/multipage/syntax.html#syntax-tag-name) nor a valid [custom element name](https://html.spec.whatwg.org/multipage/custom-elements.html#valid-custom-element-name) which should clearly signal that it is a special element (which will not be rendered to the DOM).

Additionally, `<lwc:component>` is intended to be used in tandem with `lwc:is`, otherwise `<lwc:component>` serves no purpose. The compiler will report an error when `<lwc:component>` is used without the `lwc:is` template directive.

##### `lwc:is`

A new template directive called `lwc:is` will be used as a signal to the compiler that the constructor is unknown at compile time. 

The directive has the following properties:

1. It can only be used with `<lwc:component>` tag.
    - An error will be issued when `lwc:is` is used on any other element.
2. At compile time the value supplied to `lwc:is` can only be an expression.
3. At runtime the expression supplied to `lwc:is` must be a `LightningElementConstructor`.
    - An error will be thrown at runtime if anything other than a `LightningElementConstructor` is passed to `lwc:is`.

#### Instantiation of the custom element

Dynamic component instantiation follows the same creation path as any other LWC custom element. When the dynamic component is ready to be rendered, the element will be created and mounted to the DOM.  If the component constructor supplied to `lwc:is` changes, the existing element will be unmounted.

This means that when the value provided to `lwc:is` changes the component and all of its children will be destroyed and recreated using the new constructor.  In addition, when the value provided to `lwc:is` is evaluated, the LWC engine will render depending on the following scenarios:

- **The constructor is falsy**
    - When the constructor is falsy the `<lwc:component>` tag along with all of its children are not rendered to the DOM.
- **The constructor is defined and is not a `LightningElement` constructor**
    - When the value provided to `lwc:is` is not a valid `LightningElement` constructor then an error is thrown.
- **The constructor is defined and is a `LightningElement` constructor**
    - When the constructor is a valid `LightningElement` constructor, the component will be rendered in place of `<lwc:component>`. The tag name used for the dynamic component is the value returned from `getComponentRegisteredName` for the given constructor.

_Note in the case of lazy loading, the component author is responsible for resolving the promise.  The only value that can be given to `lwc:is` is a `LightningElement` constructor._

#### Selecting the dynamic component

To select the dynamic component, the actual custom element must be selected once it has been rendered to the DOM by using the [`lwc:ref`](https://rfcs.lwc.dev/rfcs/lwc/0000-refs) directive or through another attribute assigned to the component such as a class name.

Here are some ways component authors can detect when a dynamic component has mounted:
- Use `connectedCallback` on the dynamic component to signal when it has mounted.
- A dynamic component constructor is guaranteed to be mounted in the next rendering cycle once it has been set.  When it is set, the parent component can wait until the `renderedCallback` lifecycle method is invoked to detect when the dynamic component is mounted.


```html
<template>
    <lwc:component lwc:is={lazyConstructor} lwc:ref="foo"></lwc:component>
</template>
```

```javascript
import { LightningElement } from 'lwc';

export default class extends LightningElement {
    lazyConstructor;

    connectedCallback() {
        import('lightning/concreteComponent')
            .then(({ default: ctor }) => this.lazyConstructor = ctor)
            .catch(err => console.log('Error importing component'));
    }

    renderedCallback() {
        // this.refs.foo will be available on the next rendering cycle after the constructor has been set.
        if (this.refs.foo) {
            // this.refs.foo will contain a reference to the DOM node
            console.log(this.refs.foo);
        }
    }
}
```

#### Assigning attributes and event listeners

Attributes and event listeners can be passed to the `<lwc:component>` tag declaratively or can be assigned directly to the custom element imperatively.

Attributes and event listeners assigned declaratively will be assigned to the dynamic component once it has been created.

Declaratively:

```html
<template>
    <lwc:component lwc:is={ctor} class="container" onclick={handleClick}></lwc:component>
</template>
```

Imperatively:

```html
<template>
    <lwc:component lwc:is={lazyConstructor} lwc:ref="foo"></lwc:component>
</template>
```

```javascript
import { LightningElement } from "lwc";

export default class extends LightningElement {
    lazyConstructor;

    connectedCallback() {
        import('lightning/concreteComponent')
            .then(({ default: ctor }) => this.lazyConstructor = ctor)
            .catch(err => console.log('Error importing component'));
    }

    renderedCallback() {
        if (this.refs.foo) {
            this.refs.foo.setAttribute('title', 'hawaii');
            this.refs.foo.addEventListener('click', () => console.log('hello world!'));
        }
    }
}
```
#### Usage with other LWC directives

All directives for [nested templates](https://lwc.dev/guide/reference#directives-for-nested-templates) are available to use on `<lwc:component>` and their functionality will be passed through to the custom element once it has been created.

Additionally, the following directives will also be supported:
- [`lwc:spread`](https://lwc.dev/guide/reference#lwc%3Aspread%3D%7Bchildprops%7D)
- [`lwc:ref`](https://rfcs.lwc.dev/rfcs/lwc/0000-refs)

_Note `lwc:external` will not be available because the constructor provided to the `lwc:is` directive must be a `LightningElement` constructor._

#### Children of the dynamically created element

The `<lwc:component>` tag accepts child elements, first creating the dynamic component and subsequently the children afterwards.  Each time the dynamic component changes, the existing dynamic component will be unmounted along with all of its children.  The new dynamic component will be created along with the children.

```html
<template>
    <lwc:component lwc:is={ctor}>
        <span>child</span>
    </lwc:component>
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
    <lwc:component lwc:is={ctor}></lwc:component>
</template>
```

In both cases the dynamic component will be fully mounted and unmounted.

#### Styles

Something to keep in mind is that [light DOM styles are injected to the closest root node](https://rfcs.lwc.dev/rfcs/lwc/0115-light-dom#styles) and are not removed once the components are unmounted. This means that repeated mounting and unmounting of dynamic component could cause styles to be overwritten.  This can be avoided by using either scoped styles or shadow dom, as the styles will be scoped.

## Drawbacks

The main drawback of this design is that when the constructor changes, the entire component hierarchy will need to be re-rendered. This means that the children of the custom element will also need to re-rendered.  Careful consideration should be taken by the component author as this may cause performance issues.

## Alternatives

### Use a `<template>` tag instead of `lwc:component`.

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

`LightningDynamicElement` works by synthetically handling the lifecycle of the dynamic component, which means the dynamic component does not go through the same mounting and unmounting process as other custom elements.  Additionally, `LightningDynamicElement` does not unmount when the dynamic constructor changes.

_This is similar to the [Angular approach to dynamic components](https://angular.io/guide/dynamic-component-loader)._

#### Comparison with `<lwc:component lwc:is={expression}>`

With `LightningDynamicElement` the framework keeps track of references to the dynamic components and also manages its lifecycle.  In contrast with `<lwc:component>` the component author is responsible for keeping track of the custom element.  This means that any dynamic component must be retrieved each time the component changes.  Additionally, any imperatively assigned attributes or event listeners will need to be reapplied once the component changes as well.

## Adoption strategy

This is a breaking change for existing user of `lwc:dynamic` as it supersedes the directive.  However, the `lwc:dynamic` directive has not reached GA and all consumers of the directive are internal.  They will need to switch their components to change the tag names to `lwc:component` and the directives from `lwc:dynamic` to `lwc:is`.

The behavior of `lwc:is` is the same as `lwc:dynamic` when the constructor is not provided, both will not render the dynamic component of its children. The behavior is different when the components are rendered though, as `lwc:is` will render the custom element name rather than the custom element name used with `lwc:dynamic`.  Component authors may need to adjust any testing that relies on this.

To help with the transition, the LWC team will support both `lwc:dynamic` and `lwc:is` directives for existing consumers.  Going forward, the compiler will report a warning when usage of the `lwc:dynamic` directive is detected until it is fully deprecated.

# How we teach this

This concept is fairly intuitive and is similar to the same concepts in other frameworks (Svelte, Vue, Angular).  Providing documentation around how the to use `<lwc:component>` and `lwc:is` with a small playground example should be sufficient.

Examples of [documentation](https://svelte.dev/docs#template-syntax-svelte-component) and [playground](https://svelte.dev/tutorial/svelte-component) from Svelte.

# Unresolved questions

Will there be issues with collisions?  For example if a dynamic constructor is imported but has the same custom element name as a custom element that's already on the page?
- Pivots in Locker and the `UpgradeableConstructor` should take care of this.

Should we log errors when component authors try to use `<lwc:*>` incorrectly?  For example should we prevent component authors from doing something like `document.createElement('lwc:other');`?
- In order to do this we would need to globally patch certain APIs which is something that we want to avoid to prevent bugs that are hard to maintain in the long run.  To handle this case, explicitly mention in the documentation that `<lwc:dynamic>` is an LWC compiler signal, and doesn't have any bearing at runtime.