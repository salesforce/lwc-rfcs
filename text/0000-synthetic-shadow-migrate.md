---
title: Shadow DOM migration mode
status: DRAFTED
created_at: 2023-11-14
updated_at: 2023-11-14
pr: https://github.com/salesforce/lwc-rfcs/pull/83
---

# Shadow DOM migration mode

## Summary

This RFC introduces a new `shadowSupportMode` designed to ease the transition from synthetic shadow DOM
to native shadow DOM.

## Basic example

```js
export default class extends LightningElement {
  static shadowSupportMode = 'migrate'
}
```

```html
<template>
  <button class="slds-button slds-button_brand">
      Branded button
  </button>
</template>
```

The above component will be styled as an [SLDS branded button](https://www.lightningdesignsystem.com/components/buttons/),
if SLDS CSS is loaded in the `document`'s `<head>`. However, the component uses native shadow DOM under the hood.

[Proof-of-concept demo](https://stackblitz.com/edit/salesforce-lwc-foclkc?file=src%2Fmodules%2Fx%2Fapp%2Fapp.html&title=LWC%20playground)

## Motivation

Now that [mixed shadow DOM mode](https://developer.salesforce.com/docs/platform/lwc/guide/create-mixed-shadow.html) is released, component authors can switch from synthetic shadow to native shadow DOM:

```js
export default class extends LightningElement {
  static shadowSupportMode = 'native'
}
```

Removing synthetic shadow DOM entirely would bring several benefits, including improved performance and debuggability, better compatibility with other web component frameworks, access to new native features such as `::part`, fewer bugs, and less risk of breakage as browser standards evolve.

However, there are several incompatibilities between native and synthetic shadow, due to limitations of the polyfill itself.  Authors of existing components may not see a compelling reason to switch from synthetic to native shadow. After all, switching has the major downside of potentially breaking these components, with only a small marginal upside for the component author (mostly in terms of performance and new features such as `::part`).

The most serious incompatibility between native and synthetic shadow DOM is the lack of global styling for native shadow components. In synthetic shadow, any global stylesheets still affect elements inside the shadow root. As such, many component authors have taken dependencies on global stylesheets, notably on [SLDS](https://www.lightningdesignsystem.com/) in Lightning Experience. In fact, the Salesforce documentation explicitly [encourages this](https://developer.salesforce.com/docs/platform/lwc/guide/create-components-css-slds.html) in "getting started" guides, and community-authored content also frequently promotes `slds-*` classes as well.

This RFC proposes a new mode to help ease this migration, and to encourage more component authors to move from synthetic to native shadow, by emulating synthetic shadow's styling behavior on top of native shadow DOM.

## Detailed design

`shadowSupportMode` has two options – `'reset'` (synthetic) or `'native'` (native). Currently on Lightning Experience,
`'reset'` is the default, but developers may opt into `'native'`, which 1) uses native shadow DOM, and 2) cascades native
shadow DOM down into any descendants of the component.

This RFC proposes a third mode: `'migrate'`:

```js
static shadowSupportMode = 'migrate'
```

"Migrate" mode (ala [jquery-migrate](https://github.com/jquery/jquery-migrate)) has the following behaviors:

- Native shadow DOM is used (like `'native'`).
- The mode cascades to all descendants (like `'native'`).
- Unlike `'native'`, however, "migrate" mode copies global `<style>`s and `<link rel="stylesheet">`s from the `document`'s `<head>` into the shadow root of the component.

By copying the styles from the `<head>` into the shadow root, "migrate" mode sacrifices some performance in favor of backwards compatibility. In short, one of the major differences between synthetic and native shadow DOM is largely eliminated, because SLDS styles (or other global stylesheets added by the page author, e.g. [static resources](https://help.salesforce.com/s/articleView?language=en_US&id=sf.os_apply_custom_styling_to_omniscripts_with_static_resources_19016.htm&type=5)) can apply to elements within the shadow DOM.

This "style copying" technique is based on the [open-stylable polyfill](https://github.com/nolanlawson/open-stylable), which itself is based on the ["open-stylable" proposal](https://github.com/WICG/webcomponents/issues/909).

There are technical details of how this will work, which are beyond the scope of this document – e.g. whether we use constructable stylesheets, or whether we copy the styles once or repeatedly when the `<head>` is mutated. However, the core idea is simply this stylesheet-cloning approach.

Note that "migrate" mode does not rely on the existence of synthetic shadow DOM at all, and is supported even in environments where synthetic shadow DOM is not loaded. (This allows for portability of components between environments with and without synthetic shadow, e.g. Lightning Experience versus Lightning Web Runtime.)

### Transitivity

Like `'native'` mode, `'migrate'` mode cascades to all descendant components. This allows component authors to add `'migrate'` to a top-level component and have all descendant components automatically opt-in to migration mode.

However, if any descendant components use `'native'` mode, then `'migrate'` ceases propagating. If a component is already opting in to `'native'` mode, then it is assumed that it doesn't need any global styles for compatibility.

Like `'native'`, component authors can also use `'reset'` explicitly to override a superclass's `'migrate'` value. (This is simply a product of how JavaScript classes work.)

### Limitations

Copying styles from the `<head>` into the shadow root does not cover all possible CSS selectors. Most notably, descendant
selectors that cross shadow boundaries – e.g. `.foo .bar`, where `.foo` is in one shadow root and `.bar` is in another – will
not be supported.

However, this technique was already tested on a non-trivial internal Salesforce app, and there was only one component with incorrect styles. Therefore, "migrate" mode can be thought of as a "best effort" to emulate the behavior of synthetic shadow DOM, which will probably get component authors 99.9% of the way there, but without the effort required to _fully_ migrate to native shadow.

Also, it is understood that copying styles into shadow roots may lead to a performance cost. However, browsers already have
[optimizations for repeated stylesheets in shadow roots](https://github.com/whatwg/dom/issues/831#issuecomment-585489960), so this cost is not expected to be large.

Finally, `'migrate'` mode does not attempt to emulate non-stylesheet differences between native and synthetic shadow. For example, it does not attempt to emulate "lazy" slots, or the nonstandard behavior of `innerHTML`, or other quirks. These are all things that `'migrate'` _could_ do in a subsequent version, possibly gated by [component-level API versioning](https://lwc.dev/guide/versioning#component-level-api-versioning).

For a v1 of `'migrate'` mode, however, merely cloning stylesheets should provide the most bang for the buck. It is not a complicated feature to implement, and it is likely to cover the biggest hurdle that component authors would hit when migrating from synthetic to native shadow.

### Server-side rendering

In SSR mode, no attempt is made to inject particular stylesheets into the DOM at server-side render time.

This means that there may be a [flash of unstyled content](https://en.wikipedia.org/wiki/Flash_of_unstyled_content) when the components
hydrate on the client side. Also, LWC's hydration logic will have to explicitly ignore the missing `<link>`/`<style>` elements and not report
them as hydration warnings.

The downside is, of course, an ugly load experience and potential hit to [Cumulative Layout Shift](https://web.dev/articles/cls). The solution
for component authors who are concerned about this is to move fully from `'migrate'` to `'native'` shadow mode.

While it would be technically possible to have a handoff between the client and server where both are aware of which `<link>` and `<style>`
elements need to be rendered on both the client and the server, this is probably too complex, at least for a v1 of migration mode. Also,
the main goal of this proposal is to support pure-CSR environments such as Lightning Experience, so the additional complexity is probably
not worth it.

## Drawbacks

By offering a third mode, this proposal increases the complexity of explaining mixed shadow mode.

This proposal also makes it more likely that component authors will rely on global stylesheets forever, and never migrate their components off of `'migrate'` mode and onto `'native'` mode

## Alternatives

By not doing `'migrate'` mode, we risk entrenching synthetic shadow DOM further into the LWC and Salesforce ecosystem, making it impractical or even impossible for component authors to ever migrate, and therefore for Lightning Experience to ever deprecate or remove synthetic shadow.

Synthetic shadow DOM is known to have several problems with usability, debuggability, performance, and compatibility with the larger web component ecosystem. The longer it persists, and the more component authors rely on it, the more these issues will continue to pile up with no real recourse. For example, a [change to Chromedriver](https://github.com/salesforce/lwc/issues/2333) broke several LWC components due to a longstanding bug that required subsequent patching. Synthetic shadow DOM is a "bug factory" that keeps on giving.

Furthermore, with the [end of support for IE11](https://developer.salesforce.com/blogs/2022/06/ending-support-for-ie11-on-the-lightning-platform) in Lightning Experience and widespread adoption of modern browsers that have supported native shadow DOM for years, synthetic shadow DOM will continue to be an ugly eyesore in the LWC experience. This has the potential to reduce customer trust, as LWC risks being perceived as an outdated or "legacy" framework.

## Adoption strategy

Component authors will be warmly encouraged to switch from `'reset'` mode to `'migrate'` mode, test their components for breakages, and enjoy the benefits of native shadow DOM with only a lightweight shim for backwards compatibility.

At a certain point in the future, we may or may not use [component-level API versioning](https://lwc.dev/guide/versioning#component-level-api-versioning) to forcibly opt-in components to "migrate" mode. This would need to be done carefully, after thorough vetting of "migrate" mode, and after other strategies have reached their limit (e.g. asking component authors to voluntarily adopt "migrate" mode). It may also require further refinements of "migrate" mode to more extensively emulate quirks of synthetic shadow. At this point, though, it would theoretically be possible to remove synthetic shadow DOM entirely from Lightning Experience, which would bring immediate performance, usability, debuggability, and maintenance-cost improvements.

# How we teach this

The messaging for new component authors should remain the same: use `'native'` mode, and design your components for a world of pure native shadow, without relying on global stylesheets.

For existing component authors, the messaging should be something like:

> Switch your components into "migrate" mode and experience improved performance with minimal breaking changes.

We could also potentially leverage [lwc-codemod](https://github.com/salesforce/lwc-codemod) and/or IDE prompts to encourage adoption.

# Unresolved questions

- Do any non-stylesheet behaviors of synthetic shadow DOM need to be emulated?
- How should component-level API versioning apply to this, if at all? E.g., should the "forced" opt-in be done gradually, only for newer API versions?
