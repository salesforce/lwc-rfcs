---
title: `lwc:spread` Directive
status: DRAFTED
created_at: 2021-07-05
updated_at: YYYY-MM-DD
champion: Mohammed Abdul Sattar (abdulsattar)
pr: https://github.com/salesforce/lwc-rfcs/pull/52
---

# `lwc:spread` Directive

## Summary
This proposal adds a mechanism to pass a dynamic properties to elements in
an LWC template using a new directive `lwc:spread`.

## Example
```js
// x/myComponent.js
export default class MyComponent extends LightningElement {
    childProps = { 
        name: 'My Child',
    };
}
```
```html
// x/myComponent.html
<template>
    <x-child lwc:spread={childProps}></x-child>
</template>
```

The new `lwc:spread` directive passes `childProps` as properties to `x-child`.

## Motivation

In LWC, there is no way to pass props to a child component when the component author
doesn't know in advance all the props that are needed.

One such usecase is `lwc:dynamic` which allows component authors to lazily instantiate components. However,
the props that we pass to the component can't be dynamic: we need to pass them in the template HTML itself.

This is a problem when the component constructor is changed at runtime: the new constructor
might need a different set of properties than the earlier one.

We can, of course, not pass any props and once the element renders, query the
element and set the props programmatically. We lose reactivity this way and also force
the component to handle `null` props which might not be desirable.

This proposal allows not just `lwc:dynamic` but all other elements to accept an object which is bound as props
at runtime.

### Prior Art
- React ([JSX Spread Operator](https://reactjs.org/docs/jsx-in-depth.html#spread-attributes))
- Vue ([`v-bind`](https://v3.vuejs.org/api/directives.html#v-bind))
- Svelte ([Spread Props](https://svelte.dev/tutorial/spread-props))
- Lit ([Discussion for support](https://github.com/lit/lit/issues/923), [PR](https://github.com/lit/lit/pull/1960))

## Detailed Design

A new LWC directive, `lwc:spread` will be introduced that can be used to spread the props to children elements.

### Structure
It accepts an object with keys as the property names and values.

This object is merged with props passed in the template (essentially `Object.assign`).

There is one difference however: the set of properties passed in the template doesn't change but it changes in `lwc:spread` i.e.
a property can't be present in one render and absent in the next render if it is defined in the template. With
`lwc:spread` though, it can be present in one render and not be present in the next. See example below.

**Example**

```html
<template>
    <x-child name={name}></x-child> <!-- in first render -->
    <x-child></x-child> <!-- in second render, name is missing. This is not possible with template compiler -->
</template>
```

```html
<template>
    <x-child lwc:spread={props}>
</template>
```

```js
export default class App extends LightningElement {
    handleClick() {
        this.props = showName ? {name: 'lwc'} : {}; // in one render we're passing `name` prop, but in another one there's no `name` property.
    }

}
```

In cases where a property from earlier render is missing, we have two options:
1. Explicity set it to `undefined` or `null`.
2. Leave it untouched. Meaning we just assign the properties we get in the current render without touching any of the earlier set properties.

We'll go ahead with Option 1. We'll explicity set them to `undefined`. The advantage of this is that it guarantees that at any point
the properties on an element are the ones that are present in the template and the ones present in `lwc:spread` in that render.

An interesting side effect of the above behavior can be observed when properties have default values:

```html
// x/app.html
<template>
    <x-child lwc:spread={props}></x-child>
</template>
```
```js
export default class Child extends LightningElement {
    @api name = "lwc";
}

export default class App extends LightningElement {
    props = {};
    renderedCallback() {
        // in initial render props is {} and `name` in child is 'lwc'
        this.props = {name: "aura"}; // `name` in child is "aura";
        this.props = {}; // `name` in child is `undefined`;
    }
}

```
Customers should be educated on the above behavior.

### Casing
LWC Template Compiler converts kebab-case attrs/props to camelCase to match the Javascript counterparts. Since props `lwc:spread`
are bound in Javascript, they need to be defined in camelCase. This means that `lwc:spread` won't do any case transformation.

### Overriding props
```html
<template>
  <c-child name="Hello" lwc:spread={childProps}></c-child>
</template>
```
```js
childProps = { name: "World" };
```
`lwc:spread` will take precedence to whatever props are declared in the template directly. In the above example,
`c-child` is always passed `World` as `name`. Additionally, there can be only one `lwc:spread` on a directive.

## Drawbacks

One of the biggest drawbacks is that it prevents static analysis of the component. However, if components were using the
programmatic way of assigning properties, then those were not statically analyzable anyway.

## Adoption strategy

This feature will be behind a perm. It will be allowed on a case-by-case basis by URB.