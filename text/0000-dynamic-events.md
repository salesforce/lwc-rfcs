---
title: `lwc:on` directive 
status: DRAFTED
created_at: 2025-01-13
updated_at: 2025-01-17
pr: https://github.com/salesforce/lwc-rfcs/pull/92

---

# `lwc:on` Directive

## Summary

This proposal adds a mechanism to add a collection of event listeners to elements in an LWC template using a new directive `lwc:on`.

## Basic example

```js
// x/myComponent.js
export default class MyComponent extends LightningElement {

    childEventHandlers = {
        foo: function (event) {console.log('foo');} ,
        bar: function (event) {console.log('bar');}
    };

}
```

```html
 <!-- x/myComponent.html -->
<template>
  <x-child lwc:on={childEventHandlers}></x-child>
</template>
```

For each property in `childEventHandlers`, the new `lwc:on` directive adds an event listener to `x-child`. The event listener listens for the event specified by the property's key and uses the property's value, bound to the instance of `MyComponent`, as the event handler.

## Motivation

LWC's support for declarative event listeners is limited to scenarios where event name are known while authoring the owner component template. This is sufficient for most of the cases, however when using the `lwc:component` directive, it is common to require event listeners based on the current value of the constructor passed to `lwc:is`. In such cases, it is not possible to know the event name while authoring the owner component template.

As an alternative, it is possible to add event listeners imperatively. However it requires the owner component to have a reference of the element on which it needs to add event listener. The owner component can have a reference of a element only after the element has been connected and hence typically only after the connectedCallback of corresponding component has terminated , making it impossible for the owner component to handle events dispatched by the connectedCallback of the corresponding component.


## Detailed design

A new directive `lwc:on` will be introduced that can be used to add a collection of event listeners whose name may not be known while authoring the component.

### Structure

The `lwc:on` directive would accept an object. For each property of the object, it would add a event listener which would listen for the event specified by the property's key and handle it using the property's value bound to the owner component.

### Caching

#### For static components, i.e. components declared in template directly using its selector
Since it is uncommon for event listeners to change after the start of the owner component's intial render, We can cache them to improve performance. Note that this is same as how `onevent` in template HTML works currently. For consumers, the implication of this would be that any changes made to the object passed to `lwc:on` after the first `connectedCallback` of owner component would cause no effect.


#### For dynamic components, i.e. components created using `lwc:component` directive
Since it is uncommon for event listeners to change if the constructor passed to `lwc:is` doesn't change, We can skip patching of event listeners if the element doesn't change. For consumers, the implication of this would be that after the first `connectedCallback` of owner component, any changes made to the object passed to `lwc:on` would cause no effect until the constructor passed to `lwc:is` itself is changed.


### Overriding

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
  <x-child onfoo={fooHandler} lwc:on={childEventHandlers}></x-child>
</template>
```

`lwc:on` will always be applied last. That means it will take precedence over whatever event listeners are declared in the template directly. In the above example, `x-child`, only one listener for event `foo` would be added and `childEventHandlers.foo` would be used for event handler .
Additionally, there can be only one `lwc:on` directive on an element 


### Event names

The keys of object passed to `lwc:on` should conform to the requirement set by DOM Event Specification. There would be no other constraint on the object's keys.

## Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Lightning Web Components
- integration of this feature with other existing and planned features
- cost of migrating existing Lightning Web Components applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.


To add : Static Analyzability.

## Alternatives

What other designs have been considered? What is the impact of not doing this?

To add : reflecting on* properties as event listeners

### lwc:spread - Just spreading it in template

An alternative design would be for `lwc:spread` to behave as a directive that appears to just spread the object's properties on the template.

Currently, a component can execute logic in its `connectedCallback` based on the value of an attribute in its corresponding element. If an owner component uses lwc:component to consume this component, it may encounter issues similar to those described in the [# Motivation](#motivation). Although this is considered an anti-pattern, the author of the owner component might not have control over the consumed component, leading to a noticeable feature gap. The proposed design would address this issue
 
At present, `lwc:spread` can be used to listen for standard events like `click` or `focus` by assigning event handlers to the `onclick` or `onfocus` properties. This works because these properties are applied to the rendered element, meaning the event handlers are bound to the elements themselves, not to the owner component as `onevent` on the template does. This behavior might be unexpected for consumers. The distinction between the rendered element and the `LightningElement` component is an implementation detail. For consumers, it is simply an Lwc, and this behavior cannot be explained as the application of properties to Lwc. The proposed design would be a breaking change. However, any component that relies on the current behavior can be rewritten to accommodate the new design.

Although properties and event listeners use the same HTML attribute syntax, consumers understand their differences. They recognize whether a variable in an HTML template is a property, attribute, or event listener and reason about them separately. Combining them into a single directive does not enhance the developer experience. In fact, a combined directive might lead consumers to treat them as a unified concept, causing confusion. Additionally, the usage patterns of properties and event listeners are different, and a combined directive would complicate consumer code.

Currently, the segregation of properties, attributes, and event listeners is handled solely by the template compiler. This design would require implementing a runtime segregator, causing additional runtime overhead. Any changes in segregation rules would necessitate updates in both the compiler and the runtime engine, leading to increased maintenance effort. 

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