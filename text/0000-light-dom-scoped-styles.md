---
title: Light DOM Scoped Styles
status: DRAFTED
created_at: 2021-05-14
updated_at: YYYY-MM-DD
champion: Nolan Lawson (nolanlawson), Abdulsattar Mohammed (abdulsattar)
pr: https://github.com/salesforce/lwc-rfcs/pull/50
---

# Light DOM Scoped Styles

## Summary

This proposal adds a mechanism for light DOM components ([#44](https://github.com/salesforce/lwc-rfcs/pull/44)) to have co-located CSS files that only apply to that component.

## Basic example

Since there is already `*.css` for unscoped light DOM CSS files, we add `*.scoped.css` for scoped light DOM CSS:

```html
<!-- x/foo.html -->
<template>
  <div>Hello</div>
</template>
```

```js
// x/foo.js
import { LightningElement } from 'lwc'

export default class Foo extends LightningElement {
  static shadow = false
}
```

```css
/* x/foo.scoped.css */
div {
  color: green;
}
```

The result will be:

```html
<x-app>
  <style class="x-foo_foo" type="text/css">
    div.x-foo_foo { color: green; }
  </style>
  <div class="x-foo_foo">Hello</div>
</x-app>
```

## Motivation

For Light DOM ([#44](https://github.com/salesforce/lwc-rfcs/pull/44)), the default assumption is that styles are unscoped. In other words, a light DOM component essentially just contains `<style>` tags that are inserted into the DOM _in situ_. (The actual implementation may be slightly different, but this is the basic mental model.)

Many frameworks, however, have a concept of _scoped styles_, even without using shadow DOM. These offer good developer ergonomics, because developers can concentrate on the CSS co-located with a particular component, without worrying how that CSS might affect other components.

Prior art:

- [Vue: `<style scoped>`](https://vue-loader.vuejs.org/guide/scoped-css.html)
- [Svelte: scoped styles](https://svelte.dev/docs#style)
- [React: CSS Modules in `create-react-app`](https://create-react-app.dev/docs/adding-a-css-modules-stylesheet/)
- [Stencil: scoped styles in light DOM](https://stenciljs.com/docs/styling#scoped-css)
- [BEM: Block, Element, Modifier](https://en.bem.info/)

## Detailed design

### Scoping token

For the purposes of this document, a _scoping token_ is some string that we use to scope CSS selectors to a particular region of the DOM.

In general, there are two approaches for applying scoping tokens to the DOM: classes or attributes. Both have [the same CSS specificity](https://alistapart.com/article/braces-to-pixels/#section4), but [historically](https://github.com/threepointone/glamor/issues/339) and [presently](https://github.com/salesforce/lwc/pull/2329), classes are faster than attributes in terms of the browser's [style calculation](https://developers.google.com/web/fundamentals/performance/rendering/#the_pixel_pipeline) process. So we prefer classes.

This means that, with scoped styles, all CSS selectors inside of the `*.scoped.css` file will have an added class, which scopes the rules to that particular component.

For example, the user might author:

```css
div > .bar {}
```

This is compiled to:

```css
div.x-foo_foo > .bar.x-foo_foo {}
```

Any existing classes in the template will be merged with the scoping token classes.

### Comparisons with synthetic shadow DOM

The current design of LWC's synthetic shadow scoped styles has several features that we do not want to emulate with light DOM scoped styles:

- Using attributes over classes.
- Allowing `@import` to import other CSS files.
- Evaluating scoped CSS at runtime rather than compile time.
- Using `lwc:dom="manual"` and `MutationObserver` to dynamically update the scoping token for new DOM nodes.

The classes vs attribute issue is already addressed above. So let's cover the other differences.

#### `@import` and runtime scoping

On the server, synthetic shadow scoped styles are compiled to a JavaScript function which takes the scoping tokens as input and outputs a string. This means that styles are scoped at runtime rather than compile time.

The main reason for this is that two component templates can contain `*.css` files that `@import` the same shared CSS file. This shared CSS file would need to be separately scoped for the two components. Since these shared CSS files could be very large (e.g. a CSS framework), and since they may `@import` further CSS files, it would be impractical to scope all of these files at compile time for each component. It would lead to a lot of duplicated code being generated on the server and sent to the client.

Many of these design decisions were made to emulate native shadow DOM, which is perfectly content to `@import` shared CSS files within separate shadow roots, and to scope them accordingly. However, for light DOM style scoping, we don't have the same constraint of needing to match native shadow DOM semantics.

So for light DOM style scoping, we can have a simpler system: disallow `@import` entirely, and evaluate the scoped CSS at compile time. This has performance benefits, at the cost of a probably-niche feature for developers.

#### `lwc:dom="manual"` and `MutationObserver`

Like the issue with `@import`, this one arises from the need to match native shadow DOM semantics. Consider this example:

```html
<!-- example.html -->
<template>
  <div lwc:dom="manual"></div>
</template>
```

```css
/* example.css */
span {
    color: green;
}
```

```js
// example.js
class Example extends LightningElement {
  renderedCallback() {
    this.template.querySelector('div').appendChild(document.createElement('span'))
  }
}
```

In this case, the `<span>` is dynamically inserted into the component's shadow DOM, but synthetic shadow DOM needs some way to monitor the DOM for changes so that it can apply the styling token. This is where `lwc:dom="manual"` comes in – it's a signal to add a `MutationObserver` to track changes.

This is a lot of extra machinery to support style scoping. Users need to know about `lwc:dom="manual"`, and the framework needs to create and disconnect a `MutationObserver`.

We would like to avoid this for light DOM style scoping, so we have a simpler system: dynamically-inserted elements are not scoped. Incidentally, this is how Vue and Svelte scoped styles work – there's no expectation that you can mutate the DOM with vanilla DOM APIs and still have the scoping token applied.

## Drawbacks

Conceptually, it's a bit awkward that `foo.css` in a shadow DOM component is scoped, whereas `foo.css` is unscoped for a light DOM component, so you need `foo.scoped.css` instead. However, this default behavior actually matches the native DOM behavior: when you insert a `<style>` into a shadow-DOM-using component, it's scoped, whereas it's unscoped if the component doesn't use shadow DOM.

It's also a bit awkward that scoped light DOM styles behave differently from shadow DOM styles. Developers will have to understand the difference, and in some cases perhaps prefer shadow DOM over light DOM (e.g. to support dynamically-inserted elements being scoped).

However, our goal here is not to maintain synthetic shadow DOM for all eternity. So if light DOM scoped styles differ from native shadow DOM in favor of simplicity, then this will be better in the long run for performance and maintainability.

## Alternatives

We could simply not implement scoped CSS for light DOM. However, given how popular it is in other frameworks (Svelte, Vue, the wide ecosystem of React CSS-in-JS libraries), it seems like a shame for LWC not to support it.

## Adoption strategy

Light DOM components are already opt-in (using `static shadow = false`), and scoped light DOM styles would also be opt-in, using `*.scoped.css`.

# How we teach this

Conceptually, scoped light DOM styles will have to be bundled up into a larger discussion of light DOM versus shadow DOM. Because switching between the two will never be as simple as flipping a boolean flag, and because the differences between the two can have a wide-ranging impact, we have to be careful about how we communicate the differences to developers.

# Unresolved questions

What should `:host` and `:host-context()` do in the context of scoped shadow DOM styles?

- Throw an error?
- Refer to the "host" (i.e. the light DOM component's root DOM node)?

Similarly, what should `:root` do?
