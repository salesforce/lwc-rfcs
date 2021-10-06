---
title: LWC Dynamic with Attrs/Props/Event listeners
status: DRAFTED
created_at: 2021-10-01
updated_at: 2021-10-01
pr: (leave this empty until the PR is created)
---

# LWC Dynamic with Attrs/Props/Event listeners

## Summary

This proposal augments `lwc:dynamic` to not just accept the constructor, but also
the props that are passed to the component. This allows the property set to be changed
dynamically.

## Basic example

```html
// x/app.html
<template>
    <x-cmp lwc:dynamic={cmp}></x-cmp>
</template>
```

```js
// x/app.js
import { LightningElement } from 'lwc';

export default class App extends LightningElement {
    cmp;
    async loadCtor() {
        const ctor = await import("x/child");
        const propertySet = {
            style: 'color: red;',
            name: 'John Foo',
            onchange: handleChange,
        }
        
        this.cmp = { constructor: ctor, propertySet: propertySet };
    }
}
```
In the above example, `<x-cmp>` is created with the constructor `x/child`. The `<x-cmp>` element will have
the `style` as an attribute, `name` assigned as a property and `handleChange` added as an event listener of type `change`.

## Motivation

`lwc:dynamic` allows component authors to lazily instantiate components. However,
the props that we pass to the component can't be dynamic: we need to pass them
in the template HTML itself.

This is a problem when the component constructor is changed at runtime: the new constrcutor,
might need a different set of properties than the earlier one.

We can, of course, not pass any props and once the element renders, query the
element and set the props programmatically. We lose reactivity this way and also force
the component to handle `null` props which might not be desirable.

This proposal allows `lwc:dynamic` to accept a dynamic property set along with the constructor.

## Detailed design

`lwc:dynamic` currently accepts a `LightningElementConstructor` to be passed. With this
proposal, we modify it to accept `LightningElementConstructor | { constructor: LightningElementConstructor, propertySet: { [key: string]: any }`.

LWC engine should continue working as before when `lwc:dynamic` receives a `LightningElementConstructor`.
However, if it receives an object, it should further split the `propertySet` object into attributes, properties and event listeners.

1. *Attributes*: keys `style`, `class`, `key`, `slot` and `data-*` are attributes
2. *Event Listeners*: keys that start with `on` are Event Listeners
3. *Properties*: remaining keys are properties

The newly created component is instantiated with the above attributes, keys and event listeners. This is inline with what
the template compiler does with attributes of an element in the template.

It is worth noting that event listeners declared in the template are bound to the component. Event listeners declared
via `lwc:dynamic` in this proposal should also be bound to the component.

### Reactivity

The object passed to `lwc:dynamic` is reactive. If the `constructor` changes, it is equivalent to how `lwc:dynamic` works
currently: a new component is created along with the `propertySet`.

However, if the `propertySet` changes, we again split it into three objects, attributes, properties and event listeners.
This significantly differs from what we had earlier because:
1. Attributes and Properties can't change shape: we can't have new attributes that we didn't have in the previous render, currently. Same for properties.
2. Event listeners can't be changed at all: we can't have new event listeners or even change earlier ones.

The above limitations need to be removed for accepting a propertySet in `lwc:dynamic`.

**Attributes change**
1. Key in new render, but not in old render: attribute is added.
2. Key in old render, but not in new render: attribute is removed.
3. Key in old render and also in new render: attribute is present and is subject to current diffing algo.

**Properties change**
1. Key in new render, but not in old render: property is set to the value passed.
2. Key in old render, but not in new render: property is set to `undefined`.
3. Key in old render and also in new render: property is set to the value passed.

**Listeners change**
1. Key in new render, but not in old render: listener is added.
2. Key in old render, but not in new render: listener is removed.
3. Key in old render and also in new render: if the value is same as older one, do nothing else remove old listener and add new listener.

### Duplicate properties
Duplicate properties cannot arise within the `propertySet` since it is an object. However, a property can be declared in
both the template and the `propertySet`.

```html
<template>
    <x-cmp style="color: red;" lwc:dynamic={cmp}></x-cmp>
</template>
```
```js
// ... assuming the rest of the class
loadCtor = async () => { 
    this.cmp = {
        constructor: await import("x/child"),
        propertySet: { style: "color: blue; " }
    }
}
```
One solution is to throw on detecting any attributes in the template when `lwc:dynamic` accepts an object. Since, the`propertySet`
can contain all the attributes/properties/event listeners, it's not required to declare any in the template. This needs to be a runtime check,
since we can't know at compile time if `lwc:dynamic` is accepting an object or a function (in which case
it can accept attributes/properties/event listeners).

Another solution is to always give precedence to the properties in the `propertySet`.

## Drawbacks

One of the biggest drawbacks is that it prevents static analysis of the component. However, if components were using the
programmatic way of assigning properties, then those were not statically analyzable anyway.

## Adoption strategy

`lwc:dynamic` itself is behind a gate which needs to be enabled. This feature will be part of the same gate: if you have
access to `lwc:dynamic`, you can use it in the earlier way (passing a function) or in this way.

# How we teach this

Since `lwc:dyanamic` is currently allowed only for internal developers, an internal doc describing this should be sufficient.