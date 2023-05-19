---
title: Light DOM Scoped Styles
status: IMPLEMENTED
created_at: 2021-05-14
updated_at: 2022-06-21
champion: Nolan Lawson (nolanlawson), Abdulsattar Mohammed (abdulsattar)
pr: https://github.com/salesforce/lwc-rfcs/pull/50
---

# Light DOM Scoped Styles

## Summary

This proposal adds a mechanism for components (including [light DOM components](./0114-light-dom.md)) to have co-located CSS files that are scoped to that component, but without relying on shadow DOM.

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

For ([light DOM components](./0114-light-dom.md)), the default assumption is that styles are unscoped. In other words, a light DOM component essentially just contains `<style>` tags that are inserted into the DOM _in situ_. (The actual implementation may be slightly different, but this is the basic mental model.)


Many frameworks, however, have a concept of _scoped styles_, even without using shadow DOM. These offer good developer ergonomics, because developers can concentrate on the CSS co-located with a particular component, without worrying how that CSS might affect other components.

Prior art:

- [Vue: `<style scoped>`](https://vue-loader.vuejs.org/guide/scoped-css.html)
- [Svelte: scoped styles](https://svelte.dev/docs#style)
- [React: CSS Modules in `create-react-app`](https://create-react-app.dev/docs/adding-a-css-modules-stylesheet/)
- [Stencil: scoped styles in light DOM](https://stenciljs.com/docs/styling#scoped-css)
- [BEM: Block, Element, Modifier](https://en.bem.info/)

Furthermore, scoped styles are useful even for shadow DOM. Consider a shadow parent component with a light child component: the parent's styles "bleed" into the child, which may be surprising for developers who are accustomed to the Vue/Svelte model. (Note that this only applies to native shadow DOM, not synthetic shadow DOM.)

## Invariants

Whatever we design, we would prefer for it to:

1. Use standard CSS syntax rather than non-standard CSS syntax.
2. Be compatible with the emergent [CSS Scoping Proposal](https://css.oddbird.net/scope/explainer/) (`@scope`).

## Detailed design

### File format

A new `*.scoped.css` file can be used alongside an existing `*.css` file. The component author can include either one, both, or neither.

The `*.scoped.css` file is automatically picked up by the compiler, similar to the existing `*.css` file.

In terms of ordering, `*.css` stylesheets are injected before `*.scoped.css` stylesheets.

### Scoped region

Inside of `*.scoped.css`, the CSS selectors are scoped to all elements defined in the template HTML file. For instance:

```html
<!-- foo.html -->
<template>
  <div>
    <span></span>
    <button class="my-button"></button>
  </div>
</template>
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

In addition, the root element (in this case `x-foo`) can also be targeted using `:host` (even in a light DOM component):

```css
:host {}
```

More on this [below](#targeting-the-root-element).

### Scoping token

For the purposes of this document, a _scoping token_ is a string used to scope CSS selectors to a particular region of the DOM.

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
- Using `lwc:dom="manual"` and `MutationObserver` to dynamically update the scoping token for new DOM nodes.

The classes vs attribute issue is already addressed above. So let's cover the other differences.

#### `@import`

For the time being, `@import` will be disallowed within `*.scoped.css` files. It may be enabled in the future, but for now, the invariant that "only files ending in `.scoped.css` are scoped" makes the implementation much simpler, and matches other scoped CSS implementations such as Vue and Svelte, which do not allow `@import`. (In addition, native constructable stylesheets [do not support `@import`](https://chromestatus.com/feature/4735925877735424).)

#### `lwc:dom="manual"` and `MutationObserver`

Consider this example:

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
```

This would render:

```html
<x-shadow> <!-- red background -->
  #shadow-root
    <x-light class="x-light_light-host"> <!-- blue background -->
      <style>
        :host { background: red; }
        .x-light_light-host { background: blue; }
      </style>
    </x-light>
</x-shadow>
```

In the above example, observe that `:host` is inserted as-is for the global style, whereas `:host` is transformed for the scoped style.

In order for `:host` to properly mimic shadow DOM semantics, it also needs a separate styling token from other selectors. Consider this example:

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

This should render the HTML:

```html
<x-light class="x-light_light-host"> <!-- blue background -->
  <style>
    .x-light_light-host { background: blue; }
    *.x-light_light { background: green; }
  </style>
  <div class="x-light_light">Hello</div> <!-- green background -->
</x-light>
```

This matches the same developer intuition at play with native shadow DOM, where `*` refers to elements defined inside the `<template>`, whereas `:host` refers to the "root" containing element.

Note that `:host-context()` is not supported because it [lacks buy-in from Apple and Mozilla](https://bugzilla.mozilla.org/show_bug.cgi?id=1082060#c76).

## Drawbacks

Conceptually, it's a bit awkward that `foo.css` in a shadow DOM component is "scoped," whereas `foo.css` is "unscoped" for a light DOM component (using naïve developer expectations about shadow DOM scoping). However, this default behavior actually matches the native DOM behavior: when you insert a `<style>` into a shadow-DOM-using component, it's scoped to that shadow root, whereas a light DOM component doesn't have shadow DOM to make the same guarantee.

It's also a bit awkward that scoped light DOM styles behave differently from shadow DOM styles. Developers will have to understand the difference, and in some cases perhaps prefer shadow DOM over light DOM (e.g. having one shadow root wrapper component around multiple light DOM components).

## Alternatives

### Not scoping

We could simply not implement scoped CSS for light DOM. However, given how popular it is in other frameworks (Svelte, Vue, the wide ecosystem of React CSS-in-JS libraries), it seems like a shame for LWC not to support it.

We have also had feedback from pilot customers that light DOM style scoping is a must-have.

### Descendant selectors

One alternative is to use CSS selectors that can style all of a component's descendants. For instance:

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

### `:global()`

[CSS Modules](https://github.com/css-modules/css-modules#exceptions) and [Svelte](https://svelte.dev/docs#style) both allow the `:global()` modifier to selectively disable scoping:

```css
:global(.foo) .bar {} /* .foo is global, .bar is scoped */
.foo :global(.bar) {} /* .foo is scoped, .bar is global */
:global(.foo .bar) {}  /* both .foo and .bar are global */
```

We choose not to adopt this modifier, because it is non-standard and may never be on the standards track. Whereas with the [CSS Scoping proposal](https://css.oddbird.net/scope/explainer/), we could potentially remove most of the CSS transforms and use the browser's built-in scoping behavior. (The exception is `:host`, but `:host` is at least a standard pseudo-class.)

### `*.global.css`

Another alternative is to make `foo.css` the default (scoped) stylesheet, and `foo.global.css` the global stylesheet.

This solution has some benefits, namely that styles are conceptually "scoped" by default for both light DOM and shadow DOM components. It also encourages developers to use "scoped" by default, which is a good practice for encapsulation.

However, the biggest drawback is that `*.global.css` wouldn't work well with enabling scoped styles for shadow DOM components.  This may seem redundant, but it's actually valuable in the case of a light child within a shadow parent:

```html
<x-shadow>
    #shadow-root
        <x-light></x-light>
</x-shadow>
```

A developer may naïvely assume that styles are "scoped" to both `<x-shadow>` and `<x-light>` as they are in frameworks like Vue and Svelte. However, that's not true – styles from `<x-shadow>` will "bleed" into `<x-light>`. (Sometimes this is desired, sometimes not.)

By using the `*.css` / `*.scoped.css` system, `*.css` acts as "global" (i.e. injected _in situ_ with no transformations) consistently for both light DOM and shadow DOM components. `*.scoped.css` is also consistently applied in both cases.

For developers who prefer to have styles scoped on a per-component basis, regardless of light or shadow DOM, they can use `*.scoped.css` in all their components, and everything will "just work." (Sometimes the scoping will be redundant, e.g. for shadow components within other shadow components, but it's just extra classes.)

To be clear: we will not disable `*.scoped.css` for shadow DOM components – it will work in all LWC components.

## Adoption strategy

Scoped styles would be opt-in, using `*.scoped.css`. Existing CSS (in `*.css` files) would not be affected.

# How we teach this

Conceptually, scoped light DOM styles will have to be bundled up into a larger discussion of light DOM versus shadow DOM. Because switching between the two will never be as simple as flipping a boolean flag, and because the differences between the two can have a wide-ranging impact, we have to be careful about how we communicate the differences to developers.

# Unresolved questions

There are no unresolved questions at this time.
