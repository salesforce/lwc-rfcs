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
the properties that are passed to the component. This allows them to be changed dynamically.
Support for attributes/event listeners will be added later if required.

## Terminology
*component config*: properties/constructor listeners on a component. Will be amended later to include attributes/event listeners.

## Basic example

```html
// x/app.html
<template>
    <x-cmp lwc:dynamic={cmpConfig}></x-cmp>
</template>
```

```js
// x/app.js
import { LightningElement } from 'lwc';

export default class App extends LightningElement {
    cmpConfig;
    async loadCtor() {
        const constructor = await import("x/child");

        const props = { name: 'John Foo' };

        this.cmpConfig = { constructor, props };
    }
}
```
In the above example, `<x-cmp>` is created with the constructor `x/child`. The `<x-cmp>` element will have
 `name` .

## Motivation

`lwc:dynamic` allows component authors to lazily instantiate components. However,
the props that we pass to the component can't be dynamic: we need to pass them
in the template HTML itself.

This is a problem when the component constructor is changed at runtime: the new constructor,
might need a different set of properties than the earlier one.

We can, of course, not pass any props and once the element renders, query the
element and set the props programmatically. We lose reactivity this way and also force
the component to handle `null` props which might not be desirable.

This proposal allows `lwc:dynamic` to accept a *config* along with the constructor.

## Detailed design

`lwc:dynamic` currently accepts a `LightningElementConstructor` to be passed. With this
proposal, we modify it to accept `LightningElementConstructor | LightningElementConfig` where

```ts
type LightningElementConfig = {
    constructor: LightningElementConstructor,
    props: Record<string, any>,
}
```

LWC engine should continue working as before when `lwc:dynamic` receives a `LightningElementConstructor`.

However, if it receives an object, it should consider it *config* The newly created component should be instantiated
with the above properties.

It is worth noting that event listeners declared in the template are bound to the component. Event listeners declared
via `lwc:dynamic` in this proposal should also be bound to the component.

### Reactivity

The object passed to `lwc:dynamic` is reactive. If the `constructor` changes, it is equivalent to how `lwc:dynamic` works
currently: a new component is created along with the *config*

Reactivity on the*config* significantly from what we had earlier because properties
can't change shape: we can't have new props that we didn't have in the previous render, currently.

The above restrictions need to be removed for accepting a *config* in `lwc:dynamic`.

**Properties change**
1. Key in new render, but not in old render: property is set to the value passed.
2. Key in old render, but not in new render: property is left untouched.
3. Key in old render and also in new render: property is set to the value passed.
Essentially, for properties, we just go over the property set everytime and set those properties.

### Duplicate properties

**Duplicate keys within *config***

Duplicate keys cannot exist within props since these are objects.

**Duplicate keys within config and template**
A property can be declared in both the template and the *config*.

```html
<template>
    <x-cmp name="lwc" lwc:dynamic={cmp}></x-cmp>
</template>
```
```js
// ... assuming the rest of the class
loadCtor = async () => {
    this.cmp = {
        constructor: await import("x/child"),
        props: { name: 'raptor' }
    }
}
```
One solution is to throw on detecting any properties in the template when `lwc:dynamic` accepts an object. Since, the *config*
can contain all the props, it's not required to declare any in the template.
This needs to be a runtime check, since we can't know at compile time if `lwc:dynamic` is accepting an object or
a function (in which case it can accept *config* the template itself).

Another solution is to always give precedence to what we have in the *config*.

## Drawbacks

One of the biggest drawbacks is that it prevents static analysis of the component. However, if components were using the
programmatic way of assigning properties, then those were not statically analyzable anyway.

## Adoption strategy

`lwc:dynamic` itself is behind a gate which needs to be enabled. This feature will be part of the same gate: if you have
access to `lwc:dynamic`, you can use it in the earlier way (passing a function) or in this way.

# How we teach this

Since `lwc:dyanamic` is currently allowed only for internal developers, an internal doc describing would be required.
In addition, updating the OSS doc on `lwc:dynamic` would be required.
