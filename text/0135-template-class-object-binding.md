---
title: Template class object binding
status: DRAFTED
created_at: 2024-01-04
updated_at: 2024-01-04
champion: Pierre-Marie Dartus (pmdartus)
pr: None
---

# Template class object binding

## Summary

This proposal aims to improve the Developer Experience (DX) around managing components with complex styles by enabling developers to describe the classes applied to an element using JavaScript objects.

## Basic example

```js
export default class CustomButton extends LightningElement {
  variant = null;
  position = null;

  fullWidth = false;
  stretch = false;

  get className() {
    return [
      "slds-button__icon",
      this.variant ?? `slds-button_${this.variant}`,
      this.position ?? `slds-button_${this.position}`,
      {
        "slds-button_full-width": this.fullWidth,
        "slds-button_stretch": this.stretch,
      },
    ];
  }
}
```

```html
<template>
  <button class="{className}">
    <slot></slot>
  </button>
</template>
```

## Motivation

Manipulating an element's class list is a very common use case for UI frameworks. Since `class` is a standard Element attribute, component authors currently bind those attributes with a string value. However, generating complex class names through string concatenation can be hard to read and error-prone. This is the reason why this RFC proposes enhancements to the element `class` attribute binding.

Popular UI frameworks offer such capability out of the box:

- _Vue_: Use the standard [`class`](https://vuejs.org/guide/essentials/class-and-style) HTML attribute as described in this proposal.
- _Svelte_: Use a custom template directive called [`class:name`](https://svelte.dev/docs/element-directives#class-name), to conditionally apply classes.
- _Angular_: Use a custom template directive called [`[ngClass]`](https://angular.io/api/common/NgClass), with a similar API as described in this proposal.

It is also worth mentioning that React has no built-in support for conditional style application. However, most React applications use the [classnames](https://www.npmjs.com/package/classnames) NPM package to fill this gap (6.3M weekly downloads).

The `classSet` utility method exposed from the Salesforce internal `lightning/utils` module offers similar capabilities as the public classnames NPM package. This utility function is used by +500 components in Salesforce's internal code base. As this LWC module isn't exposed on the platform, platform components have to re-implement such utility methods for managing complex styles.

As a side note, modern UI frameworks also have built-in capabilities to manipulate element styles. This capability is out of the scope of this RFC and will be the subject of a follow-up proposal.

## Detailed design

This RFC proposes to change the semantics of the template `class` attribute. On top of accepting a string value, the `class` attribute would now accept objects (plain objects and arrays) representing the CSS classes applied to the elements. The LWC engine would then interpret the object to determine which class names should be applied to the element.

The list of class names applied to a given element is defined as follows:

1. If the value is a `string`, apply all the classes in the value.
2. If the value is an `object`, iterate over its keys and apply the classes with the keys associated with a truthy value.
3. If the value is an `array`, iterate over its entries and apply this algorithm recursively for each of its entries.
4. Ignore all other values: `null`, `undefined`, other primitive values, and complex objects.

| Input                                       | Output          |
| ------------------------------------------- | --------------- |
| `'foo bar'`                                 | `'foo bar'`     |
| `{foo: true, bar: false, 'fiz buz': true }` | `foo fiz buz`   |
| `[false, 'bar', null]`                      | `'bar'`         |
| `['foo', { bar: true }, ['baz']]`           | `'foo bar baz'` |

**Is this a breaking change?**

In theory, this proposal doesn't introduce a breaking change. However, it introduces a user-land observable change. Even if the `class` attribute only accepts `string` values today, the LWC engine doesn't warn or throw when passing a value of a different type. When passing a non-string value, the LWC engine automatically coerces the value to a string.

| Input                       | Before               | After   |
| --------------------------- | -------------------- | ------- |
| `{ foo: true, bar: false }` | `'[object Object]'`  | `'foo'` |
| `[false, 'foo', null]`      | `'false,'foo',null'` | `'foo'` |

> As a side note, it's possible to match against an element with its class attribute set to `[object Object]` using the `.\005Bobject` CSS selector. 🤯

Because of this change in behavior, this change will be conditionally enabled using LWC versioning.

## Adoption strategy

This new feature will be gated through LWC built-in API versioning. Rolling it through API versioning would at the same time prevent breaking changes from sneaking in and also offers an additional incentive for developers to bump their components API version.

# How we teach this

As it is a new backward-compatible feature, updating the LWC documentation and sample apps should be sufficient. The documentation should also encourage developers to use this feature when dealing with complex class names as it is more maintainable and less error-prone compared to string concatenation.

# Unresolved questions

None at this time.