---
title: Element Internals and Form Association
status: DRAFTED
created_at: 2022-03-02
updated_at: 2022-03-02
champion: Nolan Lawson (nolanlawson)
pr: https://github.com/salesforce/lwc-rfcs/pull/78
---

# Element Internals and Form Association

## Summary

This proposal provides component authors with a way to use the [`ElementInternals`](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals) API, as well as the related [Form-Associated Custom Elements](https://web.dev/more-capable-form-controls/#lifecycle-callbacks) (FACE) lifecycle API.

## Basic example

Using `ElementInternals`:

```js
export default class extends LightningElement {
  constructor() {
    super();
    const internals = this.attachInternals();
    internals.role = 'button';
    internals.ariaLabel = 'My button';
  }
}
```

Using FACE lifecycle callbacks:

```js
export default class extends LightningElement {
  static formAssociated = true;

  formAssociatedCallback(form) {
    console.log('Form:', form);
  }
}
```

## Motivation

`ElementInternals` and Form-Associated Custom Elements (FACE) are an emerging feature of web components that are gaining browser support across [Chromium](https://chromestatus.com/feature/5962105603751936), [Firefox](https://developer.mozilla.org/en-US/docs/Web/API/ElementInternals#browser_compatibility), and [Safari](https://webkit.org/blog/13711/elementinternals-and-form-associated-custom-elements/).

These APIs give web component authors several benefits, including:

1. **Default ARIA semantics.** For instance, `<my-button>` can have a default `role` of `button` that is communicated to screen readers, even without an explicit `role="button"` attribute on the host.
2. **Form association.** A custom element can communicate its states (e.g. valid/invalid, checked/unchecked) as part of a [form](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/form), as well as use lifecycle callbacks corresponding to the form itself (form associated, form reset, etc.).

## Detailed design

In web component frameworks such as [Lit](https://lit.dev/), [Stencil](https://stenciljs.com/), and [FAST](https://www.fast.design/), component authors have access to the `class` that is registered with `customElements.define()`, as well as the `HTMLElement` instance via `this`. In LWC, the framework mediates these interactions, so we must provide an explicit API for these features.

Additionally, calling `attachInternals()` is inherently risky, because it can unintentionally leak private data. For instance, [it is possible](https://codepen.io/nolanlawson-the-selector/pen/gOdWPQJ) with vanilla custom elements to call `attachInternals()` from outside a component and gain access to its (closed) shadow root:

```js
const { shadowRoot } = document.querySelector('my-component').attachInternals();
```

This pitfall should be avoided in LWC's implementation.

### Accessing ElementInternals

Within an LWC component, component authors can call `this.attachInternals()` to create an `ElementInternals` object:

```js
export default class extends LightningElement {
  constructor() {
    super();
    const internals = this.attachInternals();
    internals.role = 'button';
  }
}
```

Most commonly, developers will probably assign the internals to a class property:

```js
export default class extends LightningElement {
  internals;
  constructor() {
    super();
    this.internals = this.attachInternals();
    this.internals.role = 'button';
  }
}
```

`this.attachInternals` is available at any point in the custom element lifecycle, similar to `this.template`. It can also be directly called from a class property:

```js
export default class extends LightningElement {
  internals = this.attachInternals();
}
```

(Note that vanilla custom elements allow this as well.)

To avoid leaking private component details, accessing internals from outside a component is not allowed:

```js
const component = document.querySelector('my-component');
console.log(component.attachInternals); // undefined
```

`this.attachInternals()` can only be called once, and will throw an error if called twice. (This is how vanilla custom elements work.) So component authors are encouraged to guard against this by avoiding calling `this.attachInternals()` in a lifecycle hook that may be called multiple times, such as `connectedCallback` or `renderedCallback`.

In Server-Side Rendering (SSR), `this.attachInternals` will be undefined.

### Form association

To associate an LWC component with a form, the standard APIs are exposed from within the component:

- `static formAssociated`
- `formAssociatedCallback`
- `formDisabledCallback`
- `formResetCallback`
- `formStateRestoreCallback`

For example:

```js
export default class extends LightningElement {
  static formAssociated = true;

  formAssociatedCallback(form) {
  }

  formDisabledCallback(disabled) {
  }

  formResetCallback() {
  }

  formStateRestoreCallback(state, mode) {
  }
}
```

And naturally, this feature can be paired with `ElementInternals` to read or write form states:

```js
export default class extends LightningElement {
  static formAssociated = true;
  
  constructor() {
    super();
    const internals = this.attachInternals();
    internals.states.add('--checked');
    console.log(internals.validity.valid);
  }
}
```

Neither `formAssociated` nor the FACE callbacks are available outside of a component:

```js
const component = document.querySelector('my-component');
console.log(component.formAssociatedCallback); // undefined
console.log(component.constructor.formAssociated); // undefined
```

This matches the current behavior in LWC of similar APIs such as `static observedAttributes` and `connectedCallback`.

### Polyfills

No attempt is made by LWC to polyfill either `ElementInternals` or FACE callbacks in browsers that don't support these APIs. If a browser does not support `ElementInternals`, then `this.attachInternals()` will throw an error. All other APIs will behave as they would for a vanilla custom element running in those browsers.

## Drawbacks

There is a risk of backwards incompatibility with this change. Notably, if a component author has already defined properties such as `attachInternals`, `formResetCallback`, or `static formAssociated`, then these may conflict with LWC's newly added APIs. Although the risk is low, we may need to use API versioning or backwards-compatibility techniques like those employed for [template refs](https://rfcs.lwc.dev/rfcs/lwc/0122-refs#this.refs).

Also, opening up `ElementInternals` and FACE callbacks introduces additional API surface. In particular, directly exposing the `ElementInternals` object may open up new APIs in the future as browsers add them.

Furthermore, we will have to be careful to implement this correctly in SSR. Neither `ElementInternals` nor FACE is currently serializable as HTML, but [they may be in the future](https://github.com/WICG/webcomponents/issues/972), and we may have to add that capability later.

## Alternatives

### `this.internals`

Instead of exposing `this.attachInternals()`, we could have a more direct syntactic sugar for accessing the internals, such as `this.internals`.

However, this has several downsides:

1. Less standards-based, with respect to vanilla custom elements and other frameworks. 
2. More likely to introduce backwards-compatibility concerns, since `this.internals` is more likely than `this.attachInternals` to have been squatted by existing components.
3. Lack of clarity as to when the internals are actually attached. LWC could proactively call `attachInternals` for every component, but this may have a performance overhead. Whereas calling it lazily when `this.internals` is first accessed would require additional machinery in LWC to manage the internals, avoid calling `attachInternals()` twice, etc.

### Sprouting attributes on the host

Without `ElementInternals`, it is possible to simiulate some of the same functionality by sprouting attributes on the `host`. For instance:

```js
export default class extends LightningElement {
  connectedCallback() {
    this.template.host.setAttribute('role', 'button');
    this.template.host.setAttribute('aria-label', 'My button');
  }
}
```

This has several downsides:

1. The component author may end up overriding attributes set by the consumer of the component.
2. Setting attributes dynamically on the host makes SSR and hydration less predictable and more difficult for the framework to manage.
3. A component author has to be careful to call `setAttribute` only when the DOM is available during the proper lifecycle callback, which is not the case with `attachInternals`.

For these reasons, this alternative is not recommended to component authors, although it may be useful as a polyfill for browers that don't support `ElementInternals`.

## Adoption strategy

This is a net-new feature, and component authors can opt-in as they find they need the exposed features.

# How we teach this

Because LWC will not offer a polyfill, we will need to advertise this feature as an _enhancement_, not as core functionality. Otherwise, it the functionality is required, we will need to make it clear to developers that they will need to include a polyfill.

Because this RFC has been designed with the most standards-compliant behavior in mind, we can largely teach this by pointing to existing documentation on [MDN](https://developer.mozilla.org), [web.dev](https://web.dev), etc.

# Unresolved questions

None at this time.
