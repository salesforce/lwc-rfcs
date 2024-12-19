---
title: Stylesheet deduplication for SSR
status: DRAFTED
created_at: 2024-12-04
updated_at: 2024-12-04
pr: https://github.com/salesforce/lwc-rfcs/pull/90
---

# Stylesheet deduplication for SSR

## Summary

In Server-Side Rendering (SSR) of web components, a common problem is that `<style>`s are duplicated when rendering multiple Declarative Shadow DOM (DSD) trees. This can contribute to performance problems, in particular for HTML parsing and style calculation. 

This RFC proposes a new API to address this problem by deduplicating rendered `<style>`s into special
vanilla web components that push deduplicated styles into `shadowRoot.adoptedStyleSheets`.

## Basic example

```ts
import { renderComponent } from '@lwc/ssr-runtime'

const html = await renderComponent('x-foo', Constructor, props, {
    stylsheetCache: new Set()
});
```

The above API allows the caller of `renderComponent` to specify a `Set` to be used for deduplicating styles. This `Set` can be pre-populated (e.g. across renders of multiple islands of components), or created fresh.

After `renderComponent` is called, the `stylesheetCache` (if specified) will be populated with any stylesheets (as strings) that were encountered during rendering.

If, during rendering, a stylesheet was _not_ found in the `stylesheetCache`, then something like the following will be rendered:

```html
<template shadowrootmode="open">
    <style id="<checksum>"> div { color: red; } </style>
    <lwc-style style-id="<checksum>"></lwc-style>
</template>
```

If a stylesheet _is_ found in the `stylesheetCache`, or for subsequent renders during `renderComponent`, then the following is rendered:

```html
<template shadowrootmode="open">
    <lwc-style style-id="<checksum>"></lwc-style>
</template>
```

This `<lwc-style>` is a special vanilla web component that is based on [Rob Eisenberg's excellent design](https://eisenbergeffect.medium.com/sharing-styles-in-declarative-shadow-dom-c5bf84ffd311).

## Motivation

In benchmarks, we have seen [significant performance costs](https://github.com/nolanlawson/declarative-shadow-dom-style-vs-link-benchmark) in page load scenarios when using duplicate `<style>`s. This can take two forms:

- Shadow DOM components containing `<style>`s in their `<template shadowrootmode="open">`s (i.e. Declarative Shadow DOM).
- Light DOM components containing direct inline `<style>`s.

The reason for the duplication is straightforward: LWC components have individual stylesheets associated with them, and those stylesheets, in SSR, are rendered as `<style>` tags. (In client-side rendering, [constructable stylesheets](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet/CSSStyleSheet) are used, which inherently avoids duplication issues.) 

While browsers do have [optimizations under the hood for duplicate stylesheets](https://github.com/whatwg/dom/issues/831#issuecomment-585489960), and while compression algorithms like Brotli can greatly reduce the over-the-wire transfer cost of duplicated text, we believe that significant perf gains could be made by deduplicating these styles.

## Detailed design

### Options

At a high level, there are multiple options on the table:

1. Bet on the native browser implementation and use it or a polyfill.
2. Detect unused CSS rules on the server side (i.e. based on the rendered CSS selectors and HTML) and remove them.
3. Extract duplicate stylesheets into `<link rel=stylesheet>`s and host the individual `.css` files externally.
4. Use an inline `<script>` or web component to read styles from a global location (e.g. the `<head>`) and copy into individual shadow roots.

After researching, especially after observing benchmark results, we've decided on option #4. The main reasons are:

1. It is the fastest option by far. Depending on the browser, it is anywhere from 20% to 100% faster than `<link rel=stylesheet>`s, the runner-up choice.
2. It would work for both shadow DOM and light DOM components.
3. It does not require waiting for browsers to implement a new API.
4. It can operate transparently – by removing both the `<style>` and itself, the `<lwc-style>` component can avoid virtual DOM-diffing or hydration mismatch issues by more closely matching what a pure Client-Side-Rendered LWC component would do (i.e. use `adoptedStyleSheets` or hoist `<style>`s into the `<head>`.
5. It avoids any issues that removing "unused" CSS rules may incur (e.g. if a rule ends up being used dynamically later on the client side).
6. Compared to a `<script>`, it results in less duplicated content, and also avoids issues with cross-origin, CSP nonces, etc. The consumer is expected to define the web component themselves once, in the `<head>` preferably.

### Design

At a high level, we need to give users of the `renderComponent` API a way to specify which stylesheets already exist on the page, and which can be considered by `<lwc-style>` to be available for reuse.

There were various options here, including adding a hook for generating HTML or virtual DOM, or even just asking consumers to use a regex on the generated HTML.

The main complexity here is the need to render something different on the first vs subsquent encounter of a given stylesheet, and to allow reuse across multiple islands. The `Set` neatly solves both cases, as it communicates very clearly whether a stylesheet was already used or not.

### API shape

A new property bag is added to the `renderComponent` API of `@lwc/ssr-runtime`.

```ts
function renderComponent(
    tagName: string, 
    Constructor: LightningElement, 
    props: any,
    // The below is new
    options?: {
        stylesheetCache?: Set<String>;
    }
): void;
```

For the client side, consumers are expected to import the `registerLwcStyleComponent` function and call it once:

```js
import { registerLwcStyleComponent } from '@lwc/style-component'
registerLwcStyleComponent()
```

Under the hood, this will call `customElements.define()` with a vanilla `<lwc-style>` component whose job is to identify nearby `<style>`s and push them into either `shadowRoot.adoptedStyleSheets` or `document.adoptedStyleSheets` (for top-level light DOM).

Consumers should most likely put this `<script>` into the `<head>` so that it registers the web component as soon as possible.

### Prior art

#### Web component frameworks

Lit has [toyed with the idea](https://github.com/lit/lit/issues/4357) of deduplicating styles, but not committed to anything. The two high-level ideas they have explored are the `<link rel=stylesheet>` solution and the "inline web component" solution.

FAST has implemented a ["styled component"](https://github.com/microsoft/fast/pull/5600) approach relying on an inline web component to copy styles from a global source into individual components. (Notably this approach has led to [Flash of Unstyled Content issues](https://github.com/microsoft/fast/issues/5906), but this seems to be an implementation issue and not a problem inherent in the design.)

The Microsoft Edge team has [an explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ShadowDOM/explainer.md) proposing a solution to this problem at the level of HTML, but this is still far from being available cross-browser. And when it is, it is unclear how backwards compatibility with older browsers should work.

#### Non-web-component frameworks

Most non-web-component using frameworks (such as Vue, Svelte, and React) have a system where styles can be extracted from compiled components and are assumed to be hoisted to the document `<head>`. In fact, in Vite, that is what this does:

```js
import './stylesheet.css' // auto-hoists
```

Needless to say, this is insufficient for our use case since LWC has first-class support for shadow DOM components.

## Drawbacks

This solution will add significant complexity both to the LWC engine (which will have to deal with supporting the new API in SSR, as well as potentially deduplicating styles client-side w.r.t the current CSR implementation) and on the consumer side (which will have to do the bulk of the work of managing and hosting the stylesheets themselves).

In terms of performance, we may be making a small performance gain at the cost of a large amount of complexity and tech debt. This tech debt could grow once browsers have implemented a blessed API, and if we decide to pivot to it.

## Alternatives

Most alternatives were already described above. The main alternative is to not do this entirely.

Another alternative was `<link rel=stylesheet>`, which was the original proposal in this RFC. This ended up being scrapped when we realized that it was much slower than an inline web component, and added significant complexity.

Another alternative was to allow for removing `<style>`s entirely, or optionally keeping them inlined if they are below a particular size. However, this adds a lot of complexity for unknown gains – YAGNI reigns here.

Yet another alternative considered was to add this functionality to `@lwc/engine-server` as well as `@lwc/ssr-runtime`. However, since `@lwc/engine-server` is intended to be deprecated in favor of the latter, it seems like a bad bet to implement the same functionality in both.

Still another alternative considered was to use a style extraction system more in line with how Svelte/Vue components work – i.e. when the component is compiled, its styles can be extracted out (so that e.g. Rollup/Webpack/Vite can hoist them to the `<head>`). This doesn't work well for LWC, though, since 1) our components may be shadow DOM rather than light DOM, and 2) our components' CSS can only be determined at runtime rather than compile time. (E.g. a scoped stylesheet's `:host` selector behaves differently when used in light DOM components versus shadow DOM components.)

## Adoption strategy

This will be implemented by downstream consumers of LWC (notably LWR). It will be largely transparent to component authors. (The only exception is if component authors are manually manipulating their rendered `<style>`s, which they should be avoiding.)

# How we teach this

Since this should not affect the component authoring experience, there is no need to teach this outside of Salesforce. It is a low-level detail of the LWC framework.

# Unresolved questions

How can we future-proof this API against potential new browser APIs? We may need to have a [sophisticated fallback strategy](https://github.com/whatwg/html/issues/10673#issuecomment-2453512552).
