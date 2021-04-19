---
title: HTML comments for LWC templates
status: DRAFTED
created_at: 2021-03-31
updated_at: 2021-03-31
pr: https://github.com/salesforce/lwc/pull/2263
---

# HTML comments for LWC templates

## Summary

In most cases, HTML comments in templates only add information for other developers. There are some use cases where HTML comments convey some runtime semantic (e.g., MSO comments). As of today, the LWC template compiler strips HTML comments. This RFC proposes a way to preserve HTML comments in LWC templates.

### Basic example

```html
<template preserve-comments>
  <!-- Greeter container -->
  <div>
	Hello <b>{name}</b>!
  </div>
  <!-- eof: Greeter container -->
</template>
```
```html
<x-example>
  # shadow-root
  |   <!-- Greeter container -->
  |   <div>
  |     Hello <b>world</b>
  |   </div>
  |   <!-- eof: Greeter container -->
</x-example>
```

## Motivation

Microsoft Outlook's desktop email client leverage MSO (Microsoft Office) to render their HTML content. Because of this, the support for common HTML and CSS properties is limited, eg: `<div>`s are not supported.

Email developers rely on [MSO conditionals](https://docs.microsoft.com/en-us/previous-versions/windows/internet-explorer/ie-developer/compatibility/ms537512(v=vs.85)?redirectedfrom=MSDN)  to improve the UX of emails opened on Microsoft Outlook.

This RFC introduces a new opt-in mechanism at compile time to preserve HTML comments.

## Proposal

### Compile-time behavior

LWC compilation strips HTML comments because the browsers do not use them at runtime. The bundle's size is smaller by stripping the HTML comments, and the rehydration process does not need to take into account comments nodes.

To preserve the performance of the existing components, we propose to introduce two options to enable comments:

1. A new boolean attribute (`lwc:preserve-comments`) to the root template tag in the component template; `false` by default.

2. A new compile option (`preserveHTMLComments`) in the template compiler; `false` by default.

Both options will make the compiler invoke a runtime API adding the HTML comments in the component template to the output code that runs in the browser/server. Modifications in each environment's renderers will ensure that the comment node/text is present on the resulting HTML.

#### Invariants

* When option 1 is present, HTML comments will be enabled regardless of the value of option 2.

* When option 2 is set to true while compiling a LWC template, HTML comments will be enabled, regardless of the presence of option 1.

### Runtime behavior

A new API "`[co]`mment" will be added to the engine/core. The `co(string commentText): VNode` will receive the comment text and return a Comment vnode.

During rendering cycles, the LWC engine diffing algorithm will show/hide the comment based on the directive containing it (if:true/false, for:each, etc.)

#### Invariants

* HTML comments will be enabled based on the opt-in mechanism, either [option 1 or 2](#compile-time-behavior).
* HTML comments will not support bindings (`{foo}`). Note: if you need to render some dynamic content in your comment, you may use the [lwc:inner-html directive](https://github.com/salesforce/lwc-rfcs/pull/15/files?short_path=e9f0a56#diff-e9f0a56e79f3840c76e53b10b430f4cd21be3e4bda91f2d41856fd151e525e93).
* When enabled, comments must be rendered in engine/dom and engine/server.
* HTML comments will behave the same way as other nodes with respect to slots and directives (`if:true`, `for:each`, etc.).

### Backwards Compatible

This feature is backward compatible because is gated by the new opt-in mechanism.

### Prior art

* React: [dangerouslySetInnerHTML attribute](https://reactjs.org/docs/dom-elements.html#dangerouslysetinnerhtml).
* VueJS: [comments option](https://vuejs.org/v2/api/#comments).
* Svelte: [preserveComments compile option](https://svelte.dev/docs#svelte_compile)

### How we teach this

We must update the documentation to reflect this opt-in mechanism.
