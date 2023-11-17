---
title: List rendering reform
status: DRAFTED
created_at: 2023-11-14
updated_at: 2023-11-14
pr: https://github.com/salesforce/lwc-rfcs/pull/84
---

# List rendering reform

## Summary

This RFC proposes a new template directive, `lwc:each`, which resolves issues and provides enhancements compared to the existing `for:each` and `iterator:*` directives.

## Basic example

```html
<template>
  <ul>
    <template lwc:each={contacts} 
              lwc:item="contact"
              lwc:key={contact.id}
              lwc:index="index"
              lwc:first="first"
              lwc:last="last"
              lwc:even="even"
              lwc:odd="odd">
      <li>
        Name: {contact.name}
        Index: {index}
        First? {first}
        Last? {last}
        Even? {even}
        Odd? {odd}
      </li>
    </template>
  </ul>
</template>
```

## Motivation

This proposal supersedes the existing [`for:each`](https://lwc.dev/guide/html_templates#for%3Aeach) and [`iterator:*`](https://lwc.dev/guide/html_templates#iterator) directives and resolves several issues with them:

- We have two directives for lists instead of one, with different features available on each. The new directive unifies them.
- The existing directives do not use the `lwc:` prefix, unlike every other modern LWC directive. Using `lwc:` makes it clear which attributes are LWC-specific, and would allow for improvements in the future such as [shorthands](https://github.com/salesforce/lwc/issues/3303) (out of scope for this RFC).
- The key should be defined on the list root element, not on each element inside the list. The current behavior requires tediously repeating the `key` for each element inside the list, and also leads to [inconsistent behavior](https://github.com/salesforce/lwc/issues/3860) for text/comment nodes.
- There is no way to easily render based on even vs odd list items.

### Prior art

- [Vue `v-for`](https://vuejs.org/guide/essentials/list.html)
- [Svelte `{#each}`](https://svelte.dev/docs/logic-blocks#each)
- [Angular `*ngFor`](https://angular.io/api/common/NgForOf#local-variables)
- [Solid `<For>`](https://docs.solidjs.com/references/api-reference/control-flow/For)
- [Lit `repeat`](https://lit.dev/docs/templates/lists/#the-repeat-directive)

## Detailed design

The design of `lwc:each` is almost identical to that of `for:each`. There are only a few modifications.

First, all directives are prefixed with `lwc:`:

- `for:each` → `lwc:each`
- `for:item` → `lwc:item`
- `key` → `lwc:key`
- `for:index` → `lwc:index`

Second, the `lwc:key` must be declared on the iterator root, not on each item within the loop:

```html
<template lwc:each={items} lwc:item="item" lwc:key={item.id}>
    {item.name}
</template>
```

Third, four new convenience directives are added:

- `lwc:first` - boolean that is `true` if the item is first in the list
- `lwc:last` - boolean that is `true` if the item is last in the list
- `lwc:even` - boolean that is `true` if the item index is even
- `lwc:odd` - boolean that is `true` if the item index is odd

Each directive will be detailed separately.

### `lwc:each`

This functions nearly identically to `for:each`. (That's kind of the point – it should not be difficult for component authors to migrate.) Like `for:each`, it is supported in both the `<template>` format:

```html
<template lwc:each={items} lwc:item="item" lwc:key={item.id}>
    {item.name}
</template>
```

…as well as the shorthand, `<template>`-less format:

```html
<li lwc:each={items} lwc:item="item" lwc:key={item.id}>
    {item.name}
</li>
```

Identically to `for:each`, it supports any [Iterable](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols).

### `lwc:item`

Identical to `for:item`. It is required for `lwc:each`, and throws a compile-time error if it is not defined:

    lwc:each and lwc:item directives should be associated together.

(This is the same error thrown for `for:each` in the same situation.)

[Unlike `for:item`](https://stackblitz.com/edit/salesforce-lwc-m56edb?file=src%2Fmodules%2Fx%2Fapp%2Fapp.html,src%2Fmodules%2Fx%2Fapp%2Fapp.js&title=LWC%20playground), `lwc:item` cannot use the same variable name as `lwc:each`:

```html
<template lwc:each={foo} lwc:item="foo" lwc:key={foo.id}>
</template>
```

The above will throw a compile-time error, since otherwise the behavior may be unexpected and confusing, especially when considering the `lwc:key` on the same element.  (Which `foo` does `lwc:key={foo.id}` refer to above? Rather than guessing, we will throw a compile-time error.)

### `lwc:key`

Identical to `key`, except that it must be declared on the same element as `lwc:each`. This avoids the tedious repetition required for `key`:

```html
<template for:each={items} for:item="item">
  <div key={item.id}></div>
  <span key={item.id}></span>
  <button key={item.id}></button>
</template>
```

With the new directives, this becomes:

```html
<template lwc:each={items} lwc:item="item" lwc:key={item.id}>
  <div></div>
  <span></span>
  <button></button>
</template>
```

Also note that, in the shorthand `<template>`-less format, `lwc:ref` is defined not on a `<template>` but on the same element as the `lwc:each`:

```html
<li lwc:each={items} lwc:item="item" lwc:key={item.id}>
    {item.name}
</li>
```

Similar to `key`, `lwc:key` is required on elements with `lwc:each`, and throws an error if missing:

    Missing lwc:key for lwc:each. Iterators must have a unique, computed key value.

Unlike `key`, a missing `lwc:key` throws the error regardless of the content inside the iterator. (A missing `key` will not throw for text nodes, comment nodes, or empty markup – an apparent [inconsistency](https://github.com/salesforce/lwc/issues/3860) in that directive.)

Similar to `key`, `lwc:key` must not reference the variable defined by `lwc:index`. It also must not reference the variable defined by `lwc:first`, `lwc:last`, `lwc:even`, or `lwc:odd`. For example, this will throw a compile-time error:

```html
<template lwc:each={items} lwc:item="item" lwc:index="idx" lwc:key={idx}>
</template>
```

In short, `lwc:key` _must_ reference the `lwc:item` variable. Anything else throws a compile-time error.

### `lwc:index`

Identical to `for:index`. Returns a number for the current list index.

As with `for:index`, `lwc:index` cannot use the same string as the `lwc:item` on the same element:

```html
<template lwc:each={items} lwc:key={same.id} lwc:item="same" lwc:index="same">
</template>
```

The above will throw a compile-time error.

(Note that `for:index` only [incidentally](https://stackblitz.com/edit/salesforce-lwc-se51rr?file=src%2Fmodules%2Fx%2Fapp%2Fapp.html,src%2Fmodules%2Fx%2Fapp%2Fapp.js&title=LWC%20playground) throws an error for the same case, because the `key` ends up referencing the `for:index` variable. For `lwc:index`, the compile-time error should be explicit.)

### `lwc:first`, `lwc:last`, `lwc:even`, `lwc:odd`

These new directives take strings as their values. When referenced, they resolve to a boolean representing whether the list item is first, last, even, or odd. (This is heavily inspired by [Angular](https://angular.io/api/common/NgForOf#local-variables).)

The addition of `lwc:first` and `lwc:last` serve to unify the two existing directives – previously, `iterator:*` was the only way to get "first" or "last" behavior. The addition of `lwc:even` and `lwc:odd` are merely for convenience (especially in the absence of [complex template expressions](https://github.com/salesforce/lwc/pull/3376)).

### Constraints

#### Variable reuse

All new directives that take a string as their value (`lwc:item`, `lwc:index`, `lwc:first`, `lwc:last`, `lwc:even`, and `lwc:odd`) must have unique values when used on the same element.

For example, the following will throw a compile-time error, because `lwc:item` and `lwc:index` have the same values:

```html
<template lwc:each={items} lwc:key={same.id} lwc:item="same" lwc:index="same">
</template>
```

Note that this restriction does not apply to loops within loops. [Just like `for:each`](https://stackblitz.com/edit/salesforce-lwc-6usqmw?file=src%2Fmodules%2Fx%2Fapp%2Fapp.html,src%2Fmodules%2Fx%2Fapp%2Fapp.js&title=LWC%20playground), variables in inner loops simply shadow the same variable in outer loops:

```html
<template lwc:each={outer} lwc:key={item} lwc:item="item" lwc:index="index">
  <!-- These will be the outer loop variables -->
  Outer item: {item}
  Outer index: {index}
  <template lwc:each={inner} lwc:key={item} lwc:item="item" lwc:index="index">
    <!-- These will be the inner loop variables -->
    Inner item: {item}
    Inner index: {index}
  </template>
</template>
```

Note that the two loops have the same variable names for `lwc:item` and `lwc:index`, and this does not cause a compiler error.

Also note that the value of the `lwc:key` in each case refers to the `item` on the same `<template>` element.

#### Empty values

Empty values (e.g. `lwc:index=""`) will also throw a compile-time error. For example:

```html
<template lwc:each={items} lwc:key={item.id} lwc:item="item" lwc:index="">
</template>
```

Missing values (e.g. `lwc:index`) will also throw at compile-time:

```html
<template lwc:each={items} lwc:key={item.id} lwc:item="item" lwc:index>
</template>
```

#### Combining with `for:each` and `iterator:*`

All new directives in this RFC must _not_ be combined in the same template file with any directive associated with the
existing `for:each` or `iterator:*` directives. Doing so should throw a compile-time error.

New directives: `lwc:each`, `lwc:key`, `lwc:item`, `lwc:index`, `lwc:first`, `lwc:last`, `lwc:even`, `lwc:odd`.

Old directives: `for:each`, `for:item`, `for:index`, `key`, `iterator:*`.

If any of the directives from the first group are in a template file with any directive from the second group, then a compile-time error is thrown.

The reason for this is that mixing the new and old directives could easily lead to confusion, e.g. when `lwc:key` and `key` are mixed in the same file.

#### Other constraints

The directives may appear in any order on the element.

Any other constraints that apply to `for:each` must also apply to `lwc:each`. For example, `lwc:ref` is (currently) unsupported inside of `for:each` and `iterator:*` – the same applies to `lwc:each`. For another example, `<slot>`s cannot have any of the new directives, and cannot be repeated inside the new iterator.

## Drawbacks

This new directive may cause confusion, because we are introducing a third directive to try to unify the previous two (see [XKCD 927](https://xkcd.com/927/)).

The implementation also becomes more complex, as we will need to support all three directives for the foreseeable future. Perhaps they can share logic under the hood, but we will still need to test all three, at the very least.

We will also need to make an effort to advocate for adoption of the new directive, which may involve updating existing documentation and writing codemods or IDE tooling.

## Alternatives

### Enhancing `for:each`

One reasonable alternative would be to enhance the existing `for:each` rather than creating a new directive. However, this has several drawbacks:

1. There could be conflicts between the legacy `key` on within-iterator elements versus the `lwc:key` on the iterator element. In cases of nested iterators, it could be very tricky to determine which keys should apply to which iterators.
2. It does not move us closer to a world where all LWC directives use the `lwc:` prefix. Directives like `for:each`, `for:item`, `for:index`, and `key` would continue to be the way to write iterators, making it difficult to support a [directive shorthand](https://github.com/salesforce/lwc/issues/3303) in the future.

### Enhancing `iterator:*`

Similarly, enhancing `iterator:*` is another approach. However, this directive is seldom-used, and is rigid in how it allows for referencing the value, index, and first and last items in a list (the property names must always be `value`, `index`, `first`, and `last`).

### Making `lwc:key` optional

The key is a performance optimization, and not all developers need it. Some developers even use a getter with `Math.random()` to provide a random key, to work around the current LWC requirement that lists must have a `key`.

It is entirely possible to make `lwc:key` optional. However, this comes with a few downsides:

1. Developers may be encouraged to use a less-performant rendering pattern, since it takes less effort to omit the key.
2. It deviates from the existing behavior of `for:each`, and is one more thing to teach.
3. Developers may now assume that keys are actually a _bad_ idea, since the "improved" `lwc:each` makes them optional.

So, for this proposal, keys are still required. However, it is definitely something that can be revisited in the future.

## Adoption strategy

Similar to `lwc:if`/`lwc:elseif`/`lwc:else`, the messaging will be that `lwc:each` is the "new" way to use lists in LWC templates, and component authors should migrate from `for:each` and `iterator:*`.

Unlike `lwc:if` and friends, there are no feature gaps between `lwc:each` and the existing `for:each` and `iterator:*` directives. (`lwc:if` is missing an equivalent to `if:false`, at least in the absence of complex expressions.) Therefore, it should be relatively straightforward to migrate existing components – it could even be done with [lwc-codemod](https://github.com/salesforce/lwc-codemod).

In the future, we could also use [component-level API versioning](https://lwc.dev/guide/versioning#component-level-api-versioning) to disallow the legacy `for:each` and `iterator:*` directives.

# How we teach this

The new `lwc:each` directive is essentially a superset of `for:each`, so it can leverage most of the teaching material for that directive.

In addition, we can downplay or remove documentaiton for `for:each` and `iterator:*`, simplifying the LWC education narrative.

# Unresolved questions

None at this time.
