---
title: InnerHTML Bindings template binding
status: DRAFTED
created_at: 2019-10-03
updated_at: 2021-03-08
pr: https://github.com/salesforce/lwc/pull/2256
champion: Philippe Riand, Pierre-Marie Dartus
---

# InnerHTML Bindings for SSR

## Summary

Applications dealing with rich content, like commerce apps, need to render this content as raw HTML, coming from a database or a CMS. Today, there is no way in LWC to rendered raw HTML content both on the browser and the server. This proposal introduces the `lwc:inner-html` template directive to inject HTML content.

### Basic example

The following template renders raw HTML content inside the `<div>` HTML element:

**example.js**
```js
import { LightningElement } from 'lwc';

export default class Example extends HTMLElement {
  content = 'Hello <strong>world</strong>!';
}
```

**example.html**
```html
<template>
  <div lwc:inner-html={content} />
</template>
```

**Output:**

```html
<x-example>
  # shadow-root
  |   <div>
  |     Hello <strong>world</strong>
  |   </div>
</x-example>
```

## Motivation

It is common in customer-facing apps to render rich content coming from a CMS. As of today, this capability is supported by acting directly with the DOM within `renderedCallback()`. The templating syntax does not provide any binding capability that renders some content as raw HTML.

The reliance on the `renderedCallback()` life-cycle hook to render raw HTML content is a real issue for server-side rendering. In SSR, the `renderedCallback()` is never invoked. There is currently no straightforward way to inject raw HTML content for server-side rendered components.

This proposal introduces a new `lwc:inner-html` directive to inject raw HTML content that would work both in the browser and on the server.

## Detailed design

### Runtime behavior

At runtime the `lwc:inner-html` directive binds the directive value to the `Element.innerHTML` property. During each rendering cycle, the LWC engine diff the current value with its previous value. If the value changes between two rendering cycles, the LWC engine sets the element `innerHTML` property to the new value.

`Element.innerHTML` is a wellknown XSS sink. To prevent malicious content injection via this `lwc:inner-html` directive, a new `sanitizeHtmlContent` hook is introduced on the LWC engine. This hook is invoked during the LWC component rendering cycle and can be used to strip out malicious code from the content to be injected. The hook accepts a single `content` argument, which is the value passed to the `lwc:inner-html` directive. The `sanitizeHtmlContent` hook is expected to return the sanitized HTML content as a `string`. By default, the `sanitizeHtmlContent` hook will throw an error indicating that it needs to be overridden.

You can override the `sanitizeHtmlContent` hook by calling the `setHooks` API. For example:

```js
import { setHooks } from 'lwc';
import DOMPurify from 'dompurify';

setHooks({
  sanitizeHtmlContent(content) {
      return DOMPurify.sanitize(content);
  }
});
```

When running in native shadow, the shadow DOM style automatically gets applied to injected content. In synthetic shadow, the `lwc:inner-html` relies on the same mechanism than `lwc:dom="manual"` to apply the scoped styles to the manually injected content. The synthetic shadow DOM attaches a MutationObserver on the root element and watches for DOM changes in the subtree to apply the styling attributes.

### Compilation restrictions

The `lwc:inner-html` directive can be applied to elements in the template. The directive accepts a string or an expression value. The LWC template compiler enforces the following restrictions:

-   The `lwc:inner-html` directive can't be used as a boolean attribute.
-   The `lwc:inner-html` directive cannot be applied to LWC components.
-   The `lwc:inner-html` directive cannot be applied to the `<slot>` and `<template>` HTML elements.
-   The `lwc:inner-html` directive cannot be applied to HTML element with child nodes.

```html
<template>
  <!-- Valid -->
  <div lwc:inner-html={content}></div>
  <div lwc:inner-html="<h1>hello</h1>"></div>

  <!-- Invalid -->
  <div lwc:inner-html></div>
  <div lwc:inner-html={content}>With content</div>
  <slot lwc:inner-html={content}></slot>
  <template lwc:inner-html={content}></template>
  <x-foo lwc:inner-html={content}></x-foo>

</template>
```

### Caveats

#### Usage of custom elements in the injected content

The LWC engine will not attempt to upgrade custom elements injected via `lwc:inner-html` even if the custom element maps to a known LWC component.

```js
import { LightningElement } from 'lwc';

export default class extends LightningElement {
  content = '<x-button></x-button>';
}
```

```html
<template>
  <x-button></x-button>
  <div lwc:inner-html={content}></div>
</template>
```

In the above example, the `<x-button>` element defined in the template is treated as a potential LWC component. The `<x-button>` custom element defined in the `content` is treated like any other custom element. If `x-button` is registered as a custom element in the global registry, it will be upgraded by the user agent.

#### Usage of `<slot>` in the injected content

The LWC compiler transforms the `<slot>` elements in the template to support slotting in synthetic shadow. `<slot>` elements injected via the `lwc:inner-html` will not participate in the slotting when LWC with running in synthetic shadow.

It is also important to call out the opposite effect of injecting `<slot>` in native shadow. In native shadow, LWC relies on the user agent for slotting. If the content injected via `lwc:inner-html` contains slot elements, those new slots might conflict with existing slots defined in the template.

## Alternatives

Different syntaxes for the `innerHTML` attribute are possible, but they finally lead to the same results. It is then more a matter of preference and consistency.

Another solution would define a new binding syntax, like for example `${{myhtml}}` or `${html:myhtml}}`. But such a syntax could be used everywhere a binding is possible, including attribute values. This could lead to undefined behaviors why not providing any value. The proposed syntax makes sure that the `innerHTML` binding always applies to the content of an element.

## Prior Art

-   React: [dangerouslySetInnerHTML attribute](https://reactjs.org/docs/dom-elements.html#dangerouslysetinnerhtml).
-   Angular: [[innerHTML]](https://angular.io/guide/property-binding).
-   VueJS: [v-html attribute](https://vuejs.org/v2/guide/syntax.html#Raw-HTML).

## Adoption strategy

Because the attribute uses the reserved `lwc:` prefix, there is no backward compatibility issue, and there is no potential name conflicts.

## How we teach this

Documentation has to be updated with this new directive. 

Usage of `lwc:dom="manual"` for dynamic content injection is now obsolete and developers should transition over to `lwc:inner-html`. This should greatly reduce the number of components using `lwc:dom="manual"` directive. The `lwc:dom="manual"` documentation should be updated to reflect this. 
