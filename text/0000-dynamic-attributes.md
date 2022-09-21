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
    name: "My Child",
  };
}
```

```html
// x/myComponent.html
<template>
  <x-child lwc:spread={childProps}></x-child>
</template>
```

The new `lwc:spread` directive applies `childProps` as properties to `x-child`.

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

It accepts an object with keys as the property names and values. This object is merged with props passed in the template (essentially `Object.assign`). There is no special handling/transformation
of props whatsoever.

For example, event handlers like `onclick` aren't specially handled as `addEventListener` like LWC does if it sees an `onclick` in the template. They'll be assigned just like any other poperty. Also, LWC binds the component to the event handlers defined in the template. `lwc:spread` won't do any of that.

### Overriding props

```html
<template>
  <c-child name="Hello" lwc:spread={childProps}></c-child>
</template>
```

```js
childProps = { name: "World" };
```

`lwc:spread` will always be applied last. That means it will take precedence to whatever props are declared in the
template directly. In the above example, `c-child` is always passed `World` as `name`. Additionally, there can be only one `lwc:spread` on a directive.

## Locker Integration

Since `lwc:spread` allows setting any property, there is a risk of component authors trying to assign properties like `innerHTML`.
To prevent this, usage of `lwc:spread` will mark the element as risky and add the `renderer` key to the element
in the template so that locker can sanitize those properties.

## Drawbacks

One of the biggest drawbacks is that it prevents static analysis of the component. However, if components were using the
programmatic way of assigning properties, then those were not statically analyzable anyway.

## Adoption strategy

This feature will be behind a perm. It'll be allowed to customers on a case-by-case basis and when the team is fully confident
of the design, it'll be open to all.
