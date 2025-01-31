---
title: `lwc:on` directive 
status: DRAFTED
created_at: 2025-01-13
updated_at: 2025-01-31
pr: https://github.com/salesforce/lwc-rfcs/pull/92

---

# `lwc:on` Directive

## Summary

This proposal introduces a mechanism to dynamically add a collection of event listeners to elements in an LWC template using a new directive, `lwc:on`.

## Basic example

```js
// x/myComponent.js
export default class MyComponent extends LightningElement {

    _someField = 'some value';

    childEventHandlers = {
        foo: function (event) {console.log('foo');} ,
        bar: function (event) {console.log(this._someField);}
    };

}
```

```html
 <!-- x/myComponent.html -->
<template>
  <x-child lwc:on={childEventHandlers}></x-child>
</template>
```

The `lwc:on` directive uses the properties of the `childEventHandlers` object to set up event listeners on `<x-child>`. Each property key in the object specifies an event type to listen for, and when that event occurs, the corresponding property's value will be invoked with `this` set to the `MyComponent` instance.
In this example, when `<x-child>` receives a 'foo' event, it logs 'foo' to the console. When it receives a 'bar' event, it logs 'some value' to the console because `this` is the `MyComponent` instance.


## Motivation

LWC's current support for declarative event listeners is limited to scenarios where event types are known at the time of authoring the owner component's HTML template. There is no way to declaratively add event listeners for event types that are determined dynamically.

A common use case arises when using the `lwc:component` directive, where event listeners may depend on the current value of the constructor passed to `lwc:is`. In such cases, the event types cannot be known in advance when authoring the owner component's HTML template.

As an alternative, it is possible to add event listeners imperatively. However, this requires the owner component to have a reference to the element on which the listener needs to be added. This reference is only available after the element has been connected, and hence, it is typically only available after the `connectedCallback` of the corresponding component has terminated. This makes it impossible for the owner component to handle events dispatched by the `connectedCallback` of the corresponding component.

## Prior Art

- React ([JSX Spread Operator](https://stackblitz.com/edit/vitejs-vite-wvbk6vet?file=src%2FApp.jsx))
- Vue ([`v-on`](https://vuejs.org/api/built-in-directives.html#v-on))
- Svelte ([Spread Events](https://svelte.dev/docs/svelte/basic-markup#Events:~:text=you%20can%20spread%20them%3A%20%3Cbutton%20%7B...thisSpreadContainsEventAttributes%7D%3Eclick%20me%3C/button%3E))
- Lit ([Discussion for support](https://github.com/lit/lit/issues/923))
- Solid ([JSX Spread Operator](https://stackblitz.com/edit/solidjs-templates-k1nr5puc?file=src%2FApp.jsx))

## Detailed design

The `lwc:on` directive will enable the addition of a collection of event listeners whose names may not be known at the time of HTML template authoring.

In this document, `element` shall refer to the rendered `Element` and `component` shall refer to the instance of `LightningElememt`.
In this document, the value of an object's property shall refer to the result of calling the object's `[[Get]]` internal method with the property's key as the argument.

### Directive behaviour 

The `lwc:on` directive would accept an object. For each property of the object, it would add a event listener to the `element` that listens for the event type specified by property's key. The property's value with `this` set to the owner `component` would be used for handling the event.

### Considered Properties

Only own enumerable string-keyed properties of object passed to `lwc:on` will be considered. Other properties, i.e. inherited properties, non-enumerable properties and symbol-keyed properties shall be ignored. 

### Caching

#### For static components, i.e. components declared in template directly using its selector
Since it is uncommon for event listeners to change after the start of the owner component's intial render, We can cache them to improve performance. Note that this is same as how `on{eventType}` in template HTML works currently. For consumers, the implication of this would be that any changes made to the object passed to `lwc:on` after the first `connectedCallback` of owner component would cause no effect.


#### For dynamic components, i.e. components created using `lwc:component` directive
Since it is uncommon for event listeners to change if the constructor passed to `lwc:is` doesn't change, We can skip patching of event listeners if the `element` doesn't change. For consumers, the implication of this would be that after the first `connectedCallback` of owner component, any changes made to the object passed to `lwc:on` would cause no effect until the constructor passed to `lwc:is` itself is changed.


### Overriding

`lwc:on` will always be applied last. That means it will take precedence over whatever event listeners are declared in the template directly. In the below example, `x-child`, only one listener for event type `foo` would be added and `childEventHandlers.foo` would be used for event handler.
Additionally, there can be only one `lwc:on` directive on an element

```js
// x/myComponent.js
export default class MyComponent extends LightningElement {

    childEventHandlers = {
        foo: function (event) {console.log('lwc:on');} ,
    };

    fooHandler(){
        console.log('onfoo');
    }

}
```

```html
 <!-- x/myComponent.html -->
<template>
  <x-child lwc:on={childEventHandlers} onfoo={fooHandler}></x-child>
</template>
```

## Drawbacks

One downside of this proposal is that it would prevent us from being able to statically analyze the names of event listeners added by this directive.

## Alternatives

### lwc:spread - Just spreading it in template

An alternative design would be for `lwc:spread` to behave as a directive that appears to just spread the object's properties onto the template.

Currently, a component can execute logic in its `connectedCallback` based on the value of an attribute in its corresponding `element`. If an owner component uses `lwc:component` to consume this component, it may encounter issues similar to those described in the [# Motivation](#motivation). Although this is considered an anti-pattern, the author of the owner component might not have control over the consumed component, leading to a noticeable feature gap. The proposed design would address this issue.
 
Currently, `lwc:spread` can be used to listen for standard events like `click` or `focus` by assigning event handlers to the `onclick` or `onfocus` properties. This works because these properties are applied to `element`, meaning the event handlers are bound to the `elements` themselves, not to the owner `component` as `onclick` or `onfocus` on the template does. This behavior might be unexpected for consumers. The distinction between `element` and `component` is an implementation detail. For consumers, it is simply an Lwc, and this behavior cannot be explained as the application of properties to Lwc. The proposed design breaks this behaviour. However, any component that relies on the current behavior can be rewritten to accommodate the new design.

Although properties and event listeners use the same HTML attribute syntax, the distinction between them is a part of public API. Consumers are expected to understand their differences and reason about them separately. Combining them into a single directive does not seem to significantly enhance the developer experience. In fact, a combined directive might lead consumers to treat them as a unified concept, causing confusion. Additionally, the usage patterns of properties and event listeners are different, and a combined directive would complicate consumer code.

Currently, the segregation of properties, attributes, and event listeners is handled solely by the template compiler. This design would require implementing a runtime segregator, causing additional runtime overhead. Any changes in segregation rules would necessitate updates in both the compiler and the runtime engine, leading to increased maintenance effort.

### Event Handler Properties

Another alternative design considered is for the framework to create new properties named `on{eventType}` on the `component`. The setter for these properties would add an event listener to the corresponding `element`. This listener would listen for events with the type `eventType` and handle them using the input value bound to the `component`. In this design, the template-compiler output shall not distinguish between properties and event listeners, they shall all be treated as properties. This would enable us to use `lwc:spread` for dynamic addition of event listeners,

This approach presents significant implementation challenges. Since we can't know all the required event listeners during component construction, we would need to create properties dynamically when they are first encountered. The framework can handle `on*` properties defined in the HTML template easily because it manages the assignment. However it would require a proxy based mechanism for the framework to create these properties when first encountered elsewhere.

Similar to previous alternative, this design would be a breaking change and it doesn't seem to improve the developer experience significantly.

## Adoption strategy

If we implement this proposal, how will existing Lightning Web Components developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Lightning Web Components patterns?

Would the acceptance of this proposal mean the Lightning Web Components documentation must be
re-organized or altered? Does it change how Lightning Web Components is taught to new developers
at any level?

How should this feature be taught to existing Lightning Web Components developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?

Need Input on all parts of design.