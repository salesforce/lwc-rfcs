---
title: Passing Dynamic Attributes/Properties
status: DRAFTED
created_at: 2021-07-05
updated_at: YYYY-MM-DD
champion: Mohammed Abdul Sattar (abdulsattar)
pr: https://github.com/salesforce/lwc-rfcs/pull/51
---

# Passing Dynamic Attributes/Properties

## Summary
This proposal adds mechanism to pass a dynamic attributes or properties bag to elements in
an LWC template.

## Example
```js
// x/myComponent.js
export default class MyComponent extends LightningElement {
    childProps = { 
        name: 'My Child',
        style: 'width: 50px',
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

The new `lwc:spread` directive passes `childProps` as properties/attributes to `x-child`.

## Motivation

In LWC, there is no way to pass props/attributes to a child component when the component author
doesn't know in advance all the props/attributes that are needed.
Adding this feature can be helpful in creating wrapper components that extend the functionality
of an existing component while being oblivious to all the props that component requires.

### Prior Art
- React ([JSX Spread Operator](https://reactjs.org/docs/jsx-in-depth.html#spread-attributes))
- Vue ([`v-bind`](https://v3.vuejs.org/api/directives.html#v-bind))
- Svelte ([Spread Props](https://svelte.dev/tutorial/spread-props))
- Lit ([Discussion for support](https://github.com/lit/lit/issues/923))

## Detailed Design

A new LWC directive, `lwc:spread` will be introduced that can be used to spread the props/attributes
to the children elements.

### Structure
It accepts an object with keys as the attribute names and values as the
corresponding attribute values.

LWC Template compiler separates all attributes into:
1. Attributes
2. Properties
3. `style` attribute
4. `class` attribute

The same algorithm will be used for separating those attributes at runtime.

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

The order in which attributes are specified matters. If the explicit attribute is passed
before the attribute bag, the attribute bag takes precedence. Otherwise, the explicit attribute
takes precedence. Much like `Object.assign`.

## Open Questions
   1. How do we handle deleting properties? i.e., if a property is missing in an updated property
bag, how do we "remove" the property of the child component?