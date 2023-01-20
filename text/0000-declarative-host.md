---
title: Declarative host element
status: DRAFTED
created_at: 2022-01-19
updated_at: 2022-01-19
champion: Nolan Lawson (nolanlawson)
pr: https://github.com/salesforce/lwc-rfcs/pull/72
---

# Declarative host element

## Summary

Provides a declarative way to apply reactive attributes, classes, and event listeners to the host (root) element.

## Basic example

In a component's template HTML file:

```html
<template>
    <lwc:host
      class="static-class"
      data-foo={dynamicAttribute}
      onclick={onRootClick}
      lwc:spread={otherAttributes}
    ></lwc:host>
    <h1>Hello world</h1>
</template>
```

Result:

```html
<x-component class="static-class" data-foo="foo" data-other="other">
    #shadow-root
      <h1>Hello world</h1>
</x-component>
```

Classes, attributes, and event listeners are applied from the `<lwc:host>` element to the
root `<x-component>` element. (The event listener is not shown above.)

## Motivation

A common pattern in LWC components is something like this:

```js
export default class extends LightningElement {
  connectedCallback() {
    this.template.host.classList.add('my-class');
  }
}
```

Developers may want to add a class, set an attribute, or add an event listener to the root (host) element
of the component. Today there is no declarative way to do this, so they resort to doing it manually in the
`connectedCallback`, `constructor`, or `renderedCallback`.

This has several downsides:

1. It does not work well with SSR, where it would require shims for DOM APIs like `classList`, `setAttribute`, etc., where `connectedCallback` timings [may differ](https://github.com/salesforce/lwc/issues/3009) between SSR and DOM, and where `renderedCallback` doesn't fire at all.
2. It is not easy to make it reactive, e.g. to set a class that updates based on a `@track`ed property.

Prior art in other frameworks:

- Stencil: [`<Host>` functional component](https://stenciljs.com/docs/host-element)
- Angular: [`@HostBinding`](https://angular.io/api/core/HostBinding)
- FAST: [Host directive template](https://www.fast.design/docs/fast-element/using-directives/#host-directives)
- Lit: [Nonexistent, but considered](https://github.com/lit/lit/issues/1825)

## Detailed design

The `<lwc:host>` element is a synthetic element defined in a component's template HTML file. Similar to [dynamic components](https://github.com/salesforce/lwc-rfcs/pull/71), it does not actually render in the DOM.

```html
<template>
    <lwc:host class="foo"></lwc:host> <!-- Doesn't actually render -->
</template>
```

Instead, `<lwc:host>` refers to either the shadow host element (in the case of shadow DOM components) or the root element (in the case of light DOM components). (In other words, if you are writing a component whose JavaScript file is `x/foo.js`, it refers to `<x-foo>`.)

Many attributes and directives that can be applied to a normal `HTMLElement` can also be applied to `<lwc:host>`. These include:

- Standard [global HTML attributes](https://developer.mozilla.org/en-US/docs/Web/HTML/Global_attributes), such as `class` and `tabindex`
- Custom HTML attributes, such as `data-*`
- Event listeners, such as `onclick` or `onfocus`
- The `lwc:spread` directive

Other directives that apply to normal `HTMLElement`s, such as `lwc:inner-html`, `lwc:ref`, and `key`, are not supported.

A bare `<lwc:host></lwc:host>` with no attributes/listeners/directives is allowed, but effectively does nothing.

### Restrictions

#### Placement

A `<lwc:host>` element must be placed at the top level (root) of a `<template>`, and may not be preceded by other elements:

```html
<!-- Valid -->
<template>
    <lwc:host></lwc:host>
    <div></div>
</template>
```

```html
<!-- Invalid -->
<template>
    <div>
        <lwc:host></lwc:host>
    </div>
</template>
```

```html
<!-- Invalid -->
<template>
    <div></div>
    <lwc:host></lwc:host>
</template>
```

Comments are allowed to precede `<lwc:host>`:

```html
<!-- Valid -->
<template>
    <!-- Comment -->
    <lwc:host></lwc:host>
</template>
```

However, in the case of `lwc:preserve-comments`, comments may not precede `<lwc:host>`:

```html
<!-- Invalid -->
<template lwc:preserve-comments>
    <!-- Comment -->
    <lwc:host></lwc:host>
</template>
```

#### Contents

The `<lwc:host>` element may not have any contents other than whitespace and comments:

```html
<!-- Valid -->
<template>
    <lwc:host>
        
    </lwc:host>
</template>
```

```html
<!-- Valid -->
<template>
    <lwc:host>
        <!-- Comment -->
    </lwc:host>
</template>
```

However, in the case of `lwc:preserve-comments`, comments inside of `<lwc:host>` are not allowed:

```html
<!-- Invalid -->
<template lwc:preserve-comments>
    <lwc:host>
        <!-- Comment -->
    </lwc:host>
</template>
```

This restriction is why `lwc:inner-html` is not supported on `<lwc:host>`.

### Timing

In terms of timing, any attributes added to or removed from the root element should follow the timing applied
to elements inside of the template:

```html
<template>
    <lwc:host class={clazz}></lwc:host>
    <div class={clazz}></div>
</template>
```

In other words, in the above example, if `clazz` changes, then the root element's `class` attribute should be changed
in the same tick as when the `<div>`'s `class` changes.

The above also applies to the timing of when event listeners are attached using `on*`.

No guarantees are made about the ordering of changes made to `<lwc:host>` compared to the ordering of changes made to elements inside of the template. I.e. there is no guarantee that changes to the root occur before changes to an in-template element, or vice-versa.

The ordering of changes made to the element itself (i.e. between the categories of `class`, other attributes, and event listeners) is also not guaranteed. (Nor is it guaranteed for in-template elements in any other RFC.)

### Precedence

Attributes and event listeners added using `<lwc:host>` may conflict with those added by the parent component:

```html
<!-- parent.html -->
<template>
    <x-child class="foo" 
             data-foo="bar" 
             onclick={onClick} 
             lwc:spread={others}>
    </x-child>
</template>
```

```html
<!-- child.html -->
<template>
    <lwc:host class="quux" 
              data-foo="toto" 
              onclick={onClick} 
              lwc:spread={others}>
    </lwc:host>
</template>
```

In cases of conflict, the following rules apply:

- `lwc:spread` precedence applies [as normal](https://github.com/salesforce/lwc-rfcs/pull/52) within the `<lwc:host>` element itself.
- For event listeners such as `onclick`, both event listeners are attached.
- For the `class` attribute, strings are concatenated with a single whitespace (`" "`) character (e.g. `"foo quux"` in the above example). No attempt is made to deduplicate duplicate strings.
- For all other attributes, the parent overrides the child's attribute (e.g. `data-foo` would be `"bar"` in the above example).

The reason for this is to allow a component to set defaults for its root's behavior, while still allowing consumers to override those defaults.

## Drawbacks

Implementing this feature does require additional complexity, and moves us further away from standard custom element syntax.

## Alternatives

### Calling it "host" instead of something else ("root," "toplevel," etc.)

Other frameworks call this feature "host." And there is some precedent in LWC, since even [light DOM scoped styles](https://github.com/salesforce/lwc-rfcs/pull/50) allow for styling the root tag of a light DOM component using `:host` in `*.scoped.css`.

"Root" may also sound like it refers to the [shadow root](https://developer.mozilla.org/en-US/docs/Web/API/ShadowRoot), so it could be even more confusing.

For non-shadow components, it is a bit odd to call it "host," but this seems to be the least confusing name overall.

### Placing attributes on the root `<template>` tag.

Historically in LWC, `<template>` is used to designate a reusable tree of HTML, e.g. in `for:each` and [scoped slots](https://github.com/salesforce/lwc-rfcs/pull/63). The root `<template>` does not actually refer to the root/host element.

Also, because `<lwc:host>` supports attributes and directives that generally would apply to normal `HTMLElement`s, it makes
sense to represent it as a pseudo-normal HTML element.

### Supporting `lwc:ref`

Supporting `lwc:ref` is technically feasible, but doesn't make much sense, as there are already alternatives. In shadow DOM, developers can use `this.template.host`, and in light DOM, they can use `this`.

Also, the whole point of this feature is to avoid needing programmatic access to the root/host element. So adding `lwc:ref` would be counter-productive.

## Adoption strategy

This proposal can be adopted as a net-new feature and does not have backwards-compatibility implications. `lwc:host` is
not a pre-existing built-in HTML element, and it's extremely unlikely that anyone is creating runtime elements with this name.

It may be possible to write codemods that look for simple usages of e.g. `this.template.host.classList.add('foo')`
and replace it with `<lwc:host>`. This may be too complex to be worthwhile, though.

# How we teach this

This feature is LWC-specific (not based on an existing web standard) and will have to be taught as such. The existence of
similar mechanisms in other frameworks (e.g. Stencil and Angular) does provide some reference points that help with teaching.

However, the fact that, for the most part, we can just say "`<lwc:host>` behaves like a normal element" makes it easier to teach this. Developers can reuse their existing knowledge to understand how it works.

# Unresolved questions

None at this time.
