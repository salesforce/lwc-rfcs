---
title: `lwc:spread` Directive
status: DRAFTED
created_at: 2021-07-05
updated_at: YYYY-MM-DD
champion: Mohammed Abdul Sattar (abdulsattar)
pr: https://github.com/salesforce/lwc-rfcs/pull/52
---

# Passing Dynamic Attributes/Properties

## Summary
This proposal adds a mechanism to pass a dynamic attributes/properties/listeners to elements in
an LWC template using a new directive `lwc:spread`.

## Example
```js
// x/myComponent.js
export default class MyComponent extends LightningElement {
    childProps = { 
        name: 'My Child',
        class: 'my-class',
        onclick: this.handleClick,
    };
    
    handleClick() {
        console.log('Clicked')
    }
}
```
```html
// x/myComponent.html
<template>
    <x-child lwc:spread={childProps}></x-child>
</template>
```

The new `lwc:spread` directive passes `childProps` as properties/attributes/event listeners to `x-child`.

## Motivation

In LWC, there is no way to pass props/attributes to a child component when the component author
doesn't know in advance all the props/attributes that are needed.

One such usecase is `lwc:dynamic` which allows component authors to lazily instantiate components. However,
the props that we pass to the component can't be dynamic: we need to pass them in the template HTML itself.

This is a problem when the component constructor is changed at runtime: the new constructor
might need a different set of properties than the earlier one.

We can, of course, not pass any props and once the element renders, query the
element and set the props programmatically. We lose reactivity this way and also force
the component to handle `null` props which might not be desirable.

This proposal allows not just `lwc:dynamic` but all other elements to accept an object which is bound as props/attrs/listeners
at runtime.

### Prior Art
- React ([JSX Spread Operator](https://reactjs.org/docs/jsx-in-depth.html#spread-attributes))
- Vue ([`v-bind`](https://v3.vuejs.org/api/directives.html#v-bind))
- Svelte ([Spread Props](https://svelte.dev/tutorial/spread-props))
- Lit ([Discussion for support](https://github.com/lit/lit/issues/923), [PR](https://github.com/lit/lit/pull/1960))

## Detailed Design

A new LWC directive, `lwc:spread` will be introduced that can be used to spread the props/attributes/listeners
to children elements.

### Structure
It accepts an object with keys as the attribute/property/listener names and values as the corresponding attribute/property/listener values.

However, we need to split the object into three sub-objects: one with attributes, one with properties, and one with event listeners.
The following algorithm will be used:

**Property**
If the object key is present on the element (e.g. `key in element`), we assume it's a property.

We do whatever the engine currently does with properties: we assign them to the element.
Note that even if `undefined` or `null` as passed as value, we assign them to the property. We do not `delete elm[propKey]`.

**Event Listener**
If the object key starts with `'on'`, we assume it's an event listener.

The listener will be bound to the element and added using `addEventListener`. When the value is `undefined` or `null`, the previously added listener is removed.

**Attributes**
If the object key is neither a property nor an event listener, we assume it's an attribute. We do the same thing that engine does
as of this writing: we `setAttribute` if is truthy and `removeAttribute` if it is `undefined | null`.

### Reactivity
There's no precendent in LWC for change in the set of attributes/properties/listeners. An element can't have a property in one
render and not have it in another render. LWC Engine actually throws when it detects such a change. In the case of `lwc:spread`,
we'll have to relax this validation.

In every render, `lwc:spread` will merge with the older value of `lwc:spread`. Any keys not present in the new `lwc:spread` but
present in the old `lwc:spread` will be set to `null` explicitly in `lwc:spread`. With this merging logic, all the new
attributes/listeners will be set and missing attributes/listeners will be removed. Missing properties will be explicitly set to `null`. This results in a consistent, logical behavior for users. Please see the following example.

### Example
```html
// x/test.html
<template>
    <x-child lwc:spread={props}></x-child>
    <button onclick={toggle}>Toggle</button>
</template>
```

```js
import { LightningElement } from 'lwc';

export default class Test extends LightningElement {
    props = {
        attrA: 'a',
        attrB: 'b',

        propA: 'a',
        propB: 'b',

        listenerA() { },
        listenerB() { },
    }
    toggle() {
        this.props = {
            attrA: 'a', // remains same
            attrC: 'c', // new attribute added using setAttribute
            // attrB is removed using `removeAttribute`

            propA: 'a', // remains same
            propC: 'c', // new property assigned using elm.propC = 'c'
            // propB is "removed" using elm.PropB = null;

            listenerA() { }, // remains same
            listenerC() { }, // new listener added using `addEventListener`
            // listenerB is removed using `removeEventListener`
        }
    }
}
```

### Casing
LWC Template Compiler converts kebab-case attrs/props to camelCase to match the Javascript counterparts. Since props `lwc:spread`
are bound in Javascript, they need to be defined in camelCase. This means that `lwc:spread` won't do any case transformation.

### Disallowed keys
Some keys (e.g. 'for:each') alter the compiled HTML output itself, so they are not allowed.
`lwc:spread` does not allow LWC directives in general.

### Overriding attributes/props
```html
<template>
  <c-child name="Hello" lwc:spread={childArgs}></c-child>
</template>
```
```js
childArgs = { name: "World" };
```
`lwc:spread` will take precedence to whatever props/attrs are declared in the template directly. In the above example,
`c-child` is always passed `World` as `name`. Additionally, there can be only one `lwc:spread` on a directive.

## Drawbacks

One of the biggest drawbacks is that it prevents static analysis of the component. However, if components were using the
programmatic way of assigning properties, then those were not statically analyzable anyway.

## Adoption strategy

This feature will be behind a perm. It will be allowed on a case-by-case basis by URB.