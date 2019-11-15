---
title: Support for spellcheck and alike global attributes.
status: DRAFTED
created_at: 2019
---

# Support for spellcheck and alike global attributes.

### related Issue: https://github.com/salesforce/lwc/issues/1595

## Summary

When setting the attribute `spellcheck` of a **custom element** in a component template, the rendered element does not reflect the attribute properly. This proposal introduces support `spellcheck` and alike global attributes when used in LWC custom elements.

## Motivation

Setting the attribute `spellcheck` of a **custom element** in the template, the corresponding rendered element don’t reflect the attribute properly, ex:

```
<template>
  (a)<c-foo spellcheck="false"></c-foo> <!-- renders as <c-foo spellcheck="true"> -->
  (b)<c-bar spellcheck={expr}></c-bar>
</template>

// js
class ... extends LightningElement {
   get expr() {
       return 'false'; // any str result in spellcheck=true
       // null, undefined, 0, '', false results in spellcheck=false
   }
}
```

(a) and (b) only occurs for custom elements; regular elements like `div` or `textarea` works correctly.

## Why such attributes like spellcheck are special for LWC?

When rendering custom elements in the template, the LWC engine always needs to set properties instead of attributes, in that way you could declare the property as `@api` in the component class, and handle value changes from the parent accordingly.

Regarding `spellcheck` from https://html.spec.whatwg.org/#spelling-and-grammar-checking:

1. **(attribute)** The `spellcheck` attribute is an enumerated attribute whose keywords are the empty string, true and false. The empty string and the true keyword map to the `true` state. The false keyword maps to the `false` state. In addition, there is a third state, the default state, which is the missing value default and the invalid value default.
2. **(property)** Returns `true` if the element is to have its spelling and grammar checked; otherwise, returns false. Can be set, to override the default and set the spellcheck content attribute.
    1. On **getting**, must return `true` if the element's spellcheck content attribute is in the `true` state, or if the element's spellcheck content attribute is in the default state and the element's default behavior is true-by-default, or if the element's spellcheck content attribute is in the default state and the element's default behavior is inherit-by-default and the element's parent element's spellcheck IDL attribute would return `true`; otherwise, if none of those conditions applies, then the attribute must instead return `false`.
    2. On **setting**, if the new value is `true`, then the element's spellcheck content attribute must be set to the literal string `"true"`, **otherwise** it must be set to the literal string `"false"`.

Specifically **2.b** is the one involved in this case: since the property is always set when rendering the custom element, it will reflect on the attribute as `spellcheck="true"` or `spellcheck="false"`.

## Proposed solution

The following tables shows a comparison between rendering a regular `HTMLElement` (correct) and `LightningElement` using the current version when writing this RFC:  `v1.1.11`.

**Table A:** Static string value, case (a): `<textarea spellcheck="staticValue">` vs `<c-foo spellcheck="staticValue">`

|Value\Rendered Element	|textarea	|c-foo	|textarea.spellcheck / c-foo.spellcheck	|
|---	|---	|---	|---	|
|`""`	|spellcheck	|spellcheck="false"	|true / false	|
|`"false"` (case insensitive)	|spellcheck="false"	|spellcheck="true"	|false / true	|
|`"any other string"`	|spellcheck="any other string"	|spellcheck="true"	|true / true	|

**Table B:** Expression value, case (b): `<textarea spellcheck={expr}>` vs `<c-foo spellcheck={expr}>`

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

### Case (a) Static values:

In the case of static values, the solution is simple: When compiling the template, we can check if the attribute being set is spellcheck with a string value, and normalize its value to boolean.

With this solution Table A becomes:

|Value\Rendered Element	|textarea	|c-foo	|textarea.spellcheck / c-foo.spellcheck	|
|---	|---	|---	|---	|
|`""`	|spellcheck	|spellcheck="true"	|true / true	|
|`"false"` (case insensitive)	|spellcheck="false"	|spellcheck="false"	|false / false	|
|`"any other string"`	|spellcheck="any other string"	|spellcheck="true"	|true / true	|

### Case (B) Expression Values:

This case is more complicated because of all the invariants that involves, but we can group them in 2 general cases to be handled:

1. `cmp.expr == null` (expr result is `null` or `undefined`) This is the more complicated codepath, and the one that deserves more attention: 
    1. Given that we always set the property, the attribute is always going to be set to `true` or `false` 
    2. Since the spellcheck attribute is **true-by-default**, for `undefined` or `null`, we will set it to boolean `true`.
    3. Setting the attribute to `true` change the browser default behavior which is inherited, see [Drawbacks](##Drawbacks) section for details.
2. `cmp.expr != null` This case is relatively simple, the correct spellcheck value will be: 
3. cmp.expr().toString().toLowerCase() !== "false"

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

There is one specific case in which the proposed solution to expression values fall short:

In case **1.b**, since the `spellcheck` attribute is set to true change, the browser default behavior is inherited (meaning the attribute should not to be present) it leaves one uncovered case in which a parent element has set the `spellcheck` attribute to `false`, and explicitly setting the attribute to `true` will override this inherited behavior:

```html
<template>
    <div spellcheck="false">
        <c-foo spellcheck={expr}></c-foo>
        <textarea spellcheck={expr}></textarea>
    </div>
</template>
```

```js
class ... extends LightningElement {
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

In the rendered chunk, `c-foo.spellcheck` will return `true`, but the correct behavior is the one from the `textarea`, in which `textarea.spellcheck` is going to be false, the inherited value from the div.

