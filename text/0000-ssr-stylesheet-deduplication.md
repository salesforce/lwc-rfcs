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

This RFC proposes a new API to address this problem by deduplicating rendered `<style>`s into `<link rel=stylesheet>`s.

## Basic example

```ts
import { renderComponent } from '@lwc/ssr-runtime'

const html = await renderComponent('x-foo', Constructor, props, {
    renderStylesheet(contents: string) {
        return {
            href: '/some-file.css',
            crossorigin: 'anonymous',
            fetchpriority: 'high',
            integrity: 'sha384-<hash>',
            referrerpolicy: 'no-referrer',
        }
    }
});
```

The above API allows the caller of `renderComponent` to specify a function that, for a given string of CSS contents, returns an object containing the following [attributes allowed on an HTML `<link rel=stylesheet>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/link#attributes):

- `href` (required)
- `crossorigin` (optional)
- `fetchpriority` (optional)
- `integrity` (optional)
- `referrerpolicy` (optional)

The caller can also return a falsy value to indicate that the `<style>` should be rendered as-is.

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

Upon [deliberation](https://salesforce.quip.com/gZIIA1Ngpc6i) we decided on 
option #3 for these reasons:

- It would work for both shadow DOM and light DOM components.
- It does not require waiting for browsers to implement a new API, or taking a dependency on a potentially heavyweight polyfill.
- It does not rely on JavaScript, avoiding Flash Of Unstyled Content (FOUC) issues.
- It avoids any issues that removing "unused" CSS rules may incur (e.g. if a rule ends up being used dynamically later on the client side).

### Design

At a high level, we need to give users of the `renderComponent` API a way to specify, for a given `<style>`, what `<link rel=stylesheet>` it should be transformed into.

There were a few options here:

1. Provide a generic API that allows returning a string of HTML.
2. Provide a specific API that allows constructing a `<link rel=stylesheet>`.
3. Provide a generic API to generate LWC virtual DOM nodes.

This RFC proposes option #2, for reasons described below.

Option #1 is great for flexibility (e.g. rendering something other than a `<link>`, in case browsers deliver a new API), but it would run into [hydration issues](https://github.com/salesforce/lwc/pull/4648#issuecomment-2429864929) at runtime, as the LWC engine would need to keep track of which client-side `<style>`s correspond to which server-rendered `<link>`s, including after a dynamic client-side `render()` that renders new stylesheets. LWC must instead control how the HTML is constructed, in case it needs to add `data-*` attributes or HTML comments instructing hydration how to deal with mismatches.

Option #3 offers the same flexibility, but also reveals too much of the internal details of the LWC engine. In the future, LWC may not even use a virtual DOM.

This leaves option #2. The idea behind option #2 is to allow a basic API for constructing a `<link rel=stylesheet>`, including attributes that may or may not be used, such as `crossorigin` and `integrity`.

### API shape

A new property bag is added to the `renderComponent` API of `@lwc/ssr-runtime`.

```ts
function renderComponent(
    tagName: string,
    Constructor: LightningElement,
    props: any,
    // The below is new
    {
        renderStylesheet(contents: string) {
            return {
                href: '/some-file.css', // required
                crossorigin: 'anonymous', // optional
                fetchpriority: 'high', // optional
                integrity: 'sha384-<hash>', // optional
                referrerpolicy: 'no-referrer', // optional
            }
        }
    }
)
```

The API allows for any number of these attributes to be specified in the returned object. The only required property is `href`. (`rel=stylesheet` is implied.)

The `renderStylesheet` function may also return a falsy value, in which case it is assumed that the `<style>` should _not_ be transformed into a `<link rel=stylesheet>`. This allows for flexibility in skipping the transformation if, for example, the stylesheet contents are sufficiently small.

The `renderStylesheet` function is passed in as part of a second options bag after the `props` passed in to `renderComponent`. All props are optional.

### Prior art

#### Web component frameworks

Lit has [toyed with the idea](https://github.com/lit/lit/issues/4357) of deduplicating styles, but not committed to anything. The two high-level ideas they have explored are the `<link rel=stylesheet>` solution and the "inline web component" solution.

FAST has implemented a ["styled component"](https://github.com/microsoft/fast/pull/5600) approach relying on an inline web component to copy styles from a global source into individual components. (Notably this approach has led to [Flash of Unstyled Content issues](https://github.com/microsoft/fast/issues/5906).)

The Microsoft Edge team has [an explainer](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/ShadowDOM/explainer.md) proposing a solution to this problem at the level of HTML, but this is still far from being available cross-browser. And when it is, it is unclear how backwards compatibility with older browsers should work.

#### Non-web-component frameworks

Most non-web-component using frameworks (such as Vue, Svelte, and React) have a system where styles can be extracted from compiled components and are assumed to be hoisted to the document `<head>`. In fact, in Vite, that is what this does:

```js
import './stylesheet.css' // auto-hoists
```

Needless to say, this is insufficient for our use case since LWC has first-class support for shadow DOM components.

## Drawbacks

This solution will add significant complexity both to the LWC engine (which will have to deal with supporting the new API in SSR, as well as dealing with hydration and mismatch detection on the client) and on the consumer side (which will have to do the bulk of the work of managing and hosting the stylesheets themselves).

In terms of performance, we may be making a small performance gain at the cost of a large amount of complexity and tech debt. This tech debt could grow once browsers have implemented a blessed API, and if we decide to pivot to it.

## Alternatives

Most alternatives were already described above. The main alternative is to not do this entirely.

We could also have a targeted strategy only for light DOM components (e.g. to dedupe them into `<style>`s and hoist a `<style>` to the document `<head>`). However, this doesn't address the large number of shadow DOM-using LWC components.

Plus, it would add significant complexity to the API to allow for multiple strategies (`<link>`, remove, keep), which would likely necessitate communicating in the `renderStylesheet` parameters whether a component is shadow or light DOM and whether its styles are scoped. To keep things simple, we instead only allow for `<link>`s to be outputted (or returning a falsy value, which indicates to retain the `<style>`). 

Another option is to allow the `renderStyleesheet` function to return a `Promise` that resolves with the expected object. This would theoretically allow for an additional optimization: if a `<style>` is detected as not duplicated (or not duplicated a significant number of times), then the user may opt to keep it inlined.

However, this wouldn't work with 1) the synchronous nature of the current `@lwc/ssr-runtime` implementation, or 2) with the fact that a `Promise` would need to be `await`ed, halting rendering and thus precluding the possibility of waiting for all `<style>`s to be accounted for before deciding what to do with them. In a future HTML streaming context, it would also be preferable to deal with `<style>`s on an ad-hoc individual basis rather than reading them all in at once.

Yet another alternative considered was to add this functionality to `@lwc/engine-server` as well as `@lwc/ssr-runtime`. However, since `@lwc/engine-server` is intended to be deprecated in favor of the latter, it seems like a bad bet to implement the same functionality in both.

## Adoption strategy

This will be implemented by downstream consumers of LWC (notably LWR). It will be largely transparent to component authors. (The only exception is if component authors are manually manipulating their rendered `<style>`s, which they should be avoiding.)

# How we teach this

Since this should not affect the component authoring experience, there is no need to teach this outside of Salesforce. It is a low-level detail of the LWC framework.

# Unresolved questions

How can we future-proof this API against potential new browser APIs? We may need to have a [sophisticated fallback strategy](https://github.com/whatwg/html/issues/10673#issuecomment-2453512552) which could involve rendering not just a `<link>` but also a new attribute in the `<template shadowrootmode>`.
