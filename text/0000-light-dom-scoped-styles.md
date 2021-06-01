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
<template lwc:render-mode="light">
  <div>Hello</div>
</template>
```

```js
// x/foo.js
import { LightningElement } from 'lwc'

export default class Foo extends LightningElement {
  static renderMode = 'light'
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

### File format

A new `*.scoped.css` file can be used alongside an existing `*.css` file. The component author can include either one, both, or neither. `*.scoped.css` is only supported for light DOM templates (marked with `lwc:render-mode="light"`).

The `*.scoped.css` file is automatically picked up by the compiler, similar to the existing `*.css` file.

In terms of ordering, `*.css` stylesheets are injected before `*.scoped.css` stylesheets.

### Scoped region

Inside of `*.scoped.css`, the CSS selectors are scoped to all elements defined in the template HTML file. For instance:

```html
<!-- foo.html -->
<div>
  <span></span>
  <button class="my-button"></button>
</div>
```

```css
/* foo.scoped.css */
div {}
span {}
button {}
div > span {}
div button {}
.my-button {}
```

In the above CSS, all selectors would match the relevant elements in the template, and only those elements.

In addition, the root element (in this case `x-foo`) can also be targeted with scoped CSS:

```css
x-foo {}
```

More on this [below](#targeting-the-root-element).

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
- Generating CSS strings at runtime rather than compile time.
- Using `lwc:dom="manual"` and `MutationObserver` to dynamically update the scoping token for new DOM nodes.

The classes vs attribute issue is already addressed above. So let's cover the other differences.

#### `@import` and runtime scoping

On the server, synthetic shadow scoped styles are compiled to a JavaScript function which takes the scoping tokens as input and outputs a string. This means that styles are scoped at runtime rather than compile time.

The main reason for this is that two component templates can contain `*.css` files that `@import` the same shared CSS file. This shared CSS file would need to be separately scoped for the two components. Since these shared CSS files could be very large (e.g. a CSS framework), and since they may `@import` further CSS files, it would be impractical to scope all of these files at compile time for each component. It would lead to a lot of duplicated code being generated on the server and sent to the client.

Many of these design decisions were made to emulate native shadow DOM, which is perfectly content to `@import` shared CSS files within separate shadow roots, and to scope them accordingly. However, for light DOM style scoping, we don't have the same constraint of needing to match native shadow DOM semantics.

So for light DOM style scoping, we can have a simpler system: disallow `@import` entirely, and generating the scoped CSS string at compile time. This has performance benefits, at the cost of a probably-niche feature for developers.


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

### Targeting the root element

With global light DOM styles, the `:host` selector is inserted as-is. This means that a light DOM child component can use `:host` to target its shadow parent's host container.

With scoped styles, the intuition around `:host` should be a bit different. Rather than inserting the CSS as-is, scoped styles imitate the encapsulation of shadow DOM components. So it makes sense for `:host` to follow shadow DOM-like semantics. This means that `:host` should refer to the root element of the light DOM component – i.e., the root of the scoped DOM region.

So for instance:

```css
/* light.css */
:host {
    background: red;
}
```

```css
/* light.scoped.css */
:host {
    background: blue;
}
* {
    background: green;
}
```

```html
<!-- light.html -->
<template>
  <div>Hello</div>
</template>
```

This would render:

```html
<x-shadow> <!-- red background -->
  #shadow-root
    <style>
      :host { background: red; }
      .x-light_light-host { background: blue; }
      *.x-light_light { background: green; }
    </style>
    <x-light class="x-light_light-host"> <!-- blue background -->
      <div class="x-light_light">Hello</div> <!-- green background -->
    </x-light>
</x-shadow>
```

In the above example, observe that `:host` is inserted as-is for the global style, whereas `:host` is transformed for the scoped style. Also note that the styling token is different for `:host` compared to other selectors, such as `*`.

Note that `:host-context()` is not supported because it [lacks buy-in from Apple and Mozilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1082060#c76).

## Drawbacks

Conceptually, it's a bit awkward that `foo.css` in a shadow DOM component is scoped, whereas `foo.css` is unscoped for a light DOM component, so you need `foo.scoped.css` instead. However, this default behavior actually matches the native DOM behavior: when you insert a `<style>` into a shadow-DOM-using component, it's scoped, whereas it's unscoped if the component doesn't use shadow DOM.

It's also a bit awkward that scoped light DOM styles behave differently from shadow DOM styles. Developers will have to understand the difference, and in some cases perhaps prefer shadow DOM over light DOM (e.g. to support dynamically-inserted elements being scoped).

However, our goal here is not to maintain synthetic shadow DOM for all eternity. So if light DOM scoped styles differ from native shadow DOM in favor of simplicity, then this will be better in the long run for performance and maintainability.

## Alternatives

### Not scoping

We could simply not implement scoped CSS for light DOM. However, given how popular it is in other frameworks (Svelte, Vue, the wide ecosystem of React CSS-in-JS libraries), it seems like a shame for LWC not to support it.

### Descendant selectors

Another alternative is to use CSS selectors that can style all of a component's descendants. For instance:

```css
/* input */
div {}
```

```css
/* output */
.scope-token div {}
```

This approach was rejected because, although it's similar to the style scoping used in Aura, it's dissimilar from the style scoping used in native shadow DOM, Vue, Svelte, Stencil, `styled-components`, etc. Our hope is that the current approach will be more familiar to more developers.

If developers want a component to contain styles that affect its children, it's always possible to use non-scoped light DOM styles, and to target the child component's classes, attributes, etc.:

```css
/* parent.css */
.my-child {}
```

```html
<!-- child.html -->
<template lwc:render-mode="light">
  <div class="my-child"></div>
</template>
```

This is the same solution one might use with global styles in Vue (`<style>`) or Svelte (`:global()`).

### Targeting the root element with `x-component`

In theory, a light DOM component can target its own root element in both global and scoped styles by using its own tag name, e.g. `x-component`:

```css
x-component {
    background: red;
}
```

The problem with this approach is that templates do not really "own" their tag names. A subclass of a `LightningElement`, for instance, could end up with a totally different tag name. Or we may rename tags at the platform level. Therefore it's not safe for developers to use this technique to "guess" their own tag name.

This is why we support the `:host` selector in scoped CSS instead.

## Adoption strategy

Light DOM components are already opt-in (using `static renderMode = 'light'`), and scoped light DOM styles would also be opt-in, using `*.scoped.css`.

# How we teach this

Conceptually, scoped light DOM styles will have to be bundled up into a larger discussion of light DOM versus shadow DOM. Because switching between the two will never be as simple as flipping a boolean flag, and because the differences between the two can have a wide-ranging impact, we have to be careful about how we communicate the differences to developers.

# Unresolved questions

There are no unresolved questions at this time.
