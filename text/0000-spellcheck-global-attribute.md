---
title: Spellcheck Attribute
status: DRAFTED
created_at: 2019/11/15
updated_at: 2019/11/20
pr: https://github.com/salesforce/lwc-rfcs/pull/21
---

# Support for spellcheck and alike global attributes.

## Summary

When setting the attribute `spellcheck` of a **custom element** in a component template, the rendered element does not reflect the attribute properly. This proposal introduces support `spellcheck` and alike global attributes when used in LWC custom elements.

## Back-pointers

 * Related issue: https://github.com/salesforce/lwc/issues/1595

## Motivation

First, when setting the attribute `spellcheck` of a **custom element** in the template, the corresponding rendered element don’t reflect the attribute into the DOM correctly.

Second, when it comes to built-in elements, the reflection works as expected, creating a difference in behavior between built-in elements and custom elements.

What is the expected outcome?

The `spellcheck` attribute should behave according to the spec:

Note: the spellcheck attribute and property are controlled by an internal state that could be false, true or default state. The following describes the behavior of the attribute and property:

1. **(attribute)** The `spellcheck` attribute is an enumerated attribute whose keywords are the empty string, `"true"` and `"false"`. The empty string and the `"true"` keyword reflects to the value `true` for the property `spellcheck`. The `"false"` keyword maps to the false state. In addition, there is a third state, the default state, which is the missing value default and the invalid value default.
2. **(property)** The `spellcheck` property returns `true` if the element is to have its spelling and grammar checked; otherwise, returns `false`. It can be set, to override the default and set the spellcheck content attribute.
    1. When **getting** the value of this property, it must return `true` if the element's `spellcheck` attribute is in the true state (which means that the value is set to some string value other than `"false"`), or if the element's spellcheck content attribute is in the default state and the element's default behavior is true-by-default, or if the element's spellcheck content attribute is in the default state and the element's default behavior is inherit-by-default and the element's parent element's spellcheck IDL attribute would return `"true"`; otherwise, if none of those conditions applies, then the property must instead return `false`.
    2. When **setting** the value of this property, if the new value is `true`, then the element's spellcheck content attribute must be set to the literal string `"true"`, **otherwise** it must be set to the literal string `"false"`.

## The problem

When rendering custom elements in a template, the LWC compiler transform attributes into properties, in case that you declare the property as `@api` in the component class. As for the built-in elements, they remain as attributes, which are handled by the browser directly. This different behavior introduces different outcomes for attributes like `spellcheck` when rendering a template.

Why is [spellcheck](https://html.spec.whatwg.org/#spelling-and-grammar-checking) so special?

Because `spellcheck` is an enumerable attribute that reflects into the boolean property `spellcheck`. Normally, boolean properties will reflect to boolean attributes, but this is one of the exceptions to that rule.

The following tables shows a comparison between rendering a regular `HTMLElement` (with the correct behavior) and `LightningElement` (with the incorrect behavior) using the current LWC version (`v1.1.11` as of 2019/11/20): 

**Table A:** Static string value, `<textarea spellcheck="staticValue">` vs `<c-foo spellcheck="staticValue">`

|Value\Rendered Element	|textarea	|c-foo	|textarea.spellcheck / c-foo.spellcheck	|
|---	|---	|---	|---	|
|`""`	|spellcheck	|spellcheck="false"	|true / false	|
|`"false"` (case insensitive)	|spellcheck="false"	|spellcheck="true"	|false / true	|
|`"any other string"`	|spellcheck="any other string"	|spellcheck="true"	|true / true	|

**Table B:** Expression value, `<textarea spellcheck={expr}>` vs `<c-foo spellcheck={expr}>`

|Value\Rendered Element	|textarea	|c-foo	|textarea.spellcheck / c-foo.spellcheck	|
|---	|---	|---	|---	|
|`undefined`	|not rendered (default state)	|spellcheck="false"	|**inherited **/ false	|
|`null`	|not rendered (default state)	|spellcheck="false"	|**inherited **/ false	|
|`0`	|spellcheck="0"	|spellcheck="false	|true / false	|
|`false`	|spellcheck="false"	|spellcheck="false"	|false / false	|
|`true`	|spellcheck="true"	|spellcheck="true"	|true / true	|
|`""`	|spellcheck	|spellcheck="false"	|true / false	|
|`"false"` (case insensitive)	|spellcheck="false"	|spellcheck="true"	|false / true	|
|`"any other string"`	|spellcheck="any other string"	|spellcheck="true"	|true / true	|
|object like	|spellcheck="val.toString()"	|spellcheck="true"	|true / true	|

## Detailed design

This RFC is attempting to solve this problem by changing the compiler to set the correct value into the `spellcheck` property when rendering the custom element, so it can reflect the value into the `spellcheck` attribute as `spellcheck="true"` or `spellcheck="false"`.

### Static Values:

In the case of static values, the solution is simple:

> When compiling the template, we can check if the `spellcheck` attribute is being set to a string value, and normalize its value to boolean.

With this solution Table A becomes:

|Value\Rendered Element	|textarea	|c-foo	|textarea.spellcheck / c-foo.spellcheck	|
|---	|---	|---	|---	|
|`""`	|spellcheck	|spellcheck="true"	|true / true	|
|`"false"` (case insensitive)	|spellcheck="false"	|spellcheck="false"	|false / false	|
|`"any other string"`	|spellcheck="any other string"	|spellcheck="true"	|true / true	|

### Dynamic Values (via template expressions):

This case is more complicated because of all the invariants that it involves, but we can group them in 2 general cases that can be handled differently for custom elements:

```html
<template>
  <c-foo spellcheck={expr}></c-foo> <!-- should reflect the attribute depending on the value of {expr} -->
</template>
```

1. `expr == null` (`expr` result is `null` or `undefined`) This is the more complicated code-path, and the one that deserves more attention: 
    1. Given that we always set the property, the attribute is always going to be set to `true` or `false` 
    2. Since the spellcheck attribute is **true-by-default**, for `undefined` or `null`, we will set it to boolean `true`.
    3. Setting the attribute to `true` change the browser default behavior which is inherited, see [Drawbacks](##Drawbacks) section for details.
2. `expr != null` This case is relatively simple, the correct spellcheck value will be: 
    1. `expr.toString().toLowerCase() !== "false"`

With this solution, Table B becomes:

|Value\Rendered Element	|textarea	|c-foo	|textarea.spellcheck / c-foo.spellcheck	|
|---	|---	|---	|---	|
|`undefined`	|not rendered (default state)	|spellcheck="true"	|**inherited **/ true	|
|`null`	|not rendered (default state)	|spellcheck="true"	|**inherited **/ true	|
|`0`	|spellcheck="0"	|spellcheck="true"	|true / true	|
|`FALSE`	|spellcheck="false"	|spellcheck="false"	|false / false	|
|`TRUE`	|spellcheck="true"	|spellcheck="true"	|true / true	|
|`""`	|spellcheck	|spellcheck="true"	|true / true	|
|`"false"` (case insensitive)	|spellcheck="false"	|spellcheck="false"	|false / false	|
|`"any other string"`	|spellcheck="any other string"	|spellcheck="true"	|true / true	|
|object like	|spellcheck="val.toString()"	|spellcheck="true"	|true / true	|

## Drawbacks

There is one specific case in which the proposed solution falls short:

In the case of **1.2**, when the `spellcheck` attribute is set to true, it changes the browser default behavior, which is inherited (meaning the attribute should not to be present) and leaves one uncovered case in which a parent element has set the `spellcheck` attribute to `false`, and explicitly setting the attribute to `true` will override this inherited behavior:

```html
<template>
    <div spellcheck="false">
        <c-foo spellcheck={expr}></c-foo>
        <textarea spellcheck={expr}></textarea>
    </div>
</template>
```

```js
export class Foo extends LightningElement {
   get expr() {
       return undefined; // or null
   }
}
```

will render this html chunk:

```html
<div spellcheck="false">
    <c-foo spellcheck="true"></c-foo>
    <textarea></textarea>
</div>
```

In the rendered chunk, `c-foo.spellcheck` will return `true`, but the correct behavior is the one from the `textarea`, in which `textarea.spellcheck` is going to be `false`, the inherited value from the div.

## Alternatives

We evaluated the alternative of fixing the issue at runtime by overwritting the `spellcheck` property descriptor for custom elements. This approach was dismissed because results in the same differences  between custom element and built-in element when programatically setting the `spellcheck` property.

## Adoption strategy

Component developers will not notice any difference in usage.

# How we teach this

This RFC is implementation specific. The only thing needed is to update the documentation adding a note that mention the limitations for the `spellcheck` attribute when used on custom elements (see [Drawbacks](##Drawbacks) section)

# Unresolved questions

