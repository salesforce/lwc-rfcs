---
title: LWC server runtime
status: DRAFTED
created_at: 2020-01-06
updated_at: 2020-01-06
pr: (leave this empty until the PR is created)
---

# LWC server runtime

## Summary

The LWC engine has been designed from the beginning to run in an environment with access to the DOM APIs. To accommodate server-side rendering (SSR) requirements, we need a way to decouple the engine from the DOM APIs. The existing `@lwc/engine` will be replaced by 3 new packages: 
- `@lwc/engine-core` exposes platform-agnostic APIs and will be used by the different runtime packages to share the common logic.
- `@lwc/engine-dom` exposes LWC APIs available on the browser.
- `@lwc/engine-server` exposes LWC APIs used for server-side rendering.

## Scope

LWC SSR support is a broad subject, this proposal only focusses on a subset of it. This proposal covers the following topics:

- How to evaluate the existing LWC engine module in a javascript context without DOM access.
- What LWC APIs should be exposed in both environments.
- How the LWC engine share the same logic when evaluated in both environments.

The following topics are considered out of the scope of this proposal:

- How the LWC component tree rendered on the server gets serialized and sent to the browser.
- How the serialized component tree gets rehydrated when processed by the browser.

## Motivation

The first step in enabling SSR support on LWC is to be able to run the engine in a non-DOM-enabled environment. It is currently impossible to evaluate the engine in such environment due to its hard dependencies on DOM APIs.

While we want to share as much logic as possible between the different environments and make SSR as transparent as possible for component authors, the runtime behavior and APIs exposed by the engine greatly differs between environments. In this regard, it makes sense to break up the current `@lwc/engine` package into multiple packages specifically tailored for each environment.

Distributing multiple versions of the LWC engine offers the capability to expose different APIs depending on the execution context. On one hand, the current implementation of `createElement` with reaction hooks doesn't make sense in Node.js since there is no DOM tree to attach the created element. In the same way, the `renderToString` method doesn't make sense to ship in a browser.

## Detailed design

The existing `@lwc/engine` will be replaced by 3 new packages:

- `@lwc/engine-core`: Core logic shared by the different runtime including the rendering engine and the reactivity mechanism. This package should never be consumed directly in an application. This package is agnostic on the underlying rendering medium. The only provide APIs for building custom renderers.
- `@lwc/engine-dom`: Runtime that can be used to render LWC component trees in a DOM environment. This package is built on top of `@lwc/engine-core`.
- `@lwc/engine-server`: Runtime used that can be used to render LWC component trees as strings. This package is built on top of `@lwc/engine-core`.

The existing `lwc` package stays untouched and will be used to distribute the different versions of the engine. From the developer perspective, the experience should be identical.

**`c/app/app.js`:**

```js
import { LightningElement } from "lwc";

export default class App extends LightningElement {}
```

**`client.js`:**

```js
import { createElement } from "lwc"; // Aliased to @lwc/engine-dom
import App from "c/app";

const app = createElement("c-app", { is: App });
document.body.appendChild(app);
```

**`server.js`:**

```js
import { createElement, renderToString } from "lwc"; // Aliased to @lwc/engine-server
import App from "c/app";

const app = createElement("c-app", { is: App });
const str = renderToString(app);

console.log(str);
```

### `@lwc/engine-core`

This packages exposes the following platform-agnostic APIs:

- `LightningElement`
- `api`
- `track`
- `readonly`
- `wire`
- `setFeatureFlag`
- `getComponentDef`
- `isComponentConstructor`
- `getComponentConstructor`
- `unwrap`
- `registerTemplate`
- `registerComponent`
- `registerDecorators`
- `sanitizeAttribute`

The DOM APIs used by the rendered engine are injected by the runtime depending on the environment. A list of all the DOM APIs the engine depends upon can be found in the [DOM APIs usage](#dom-apis-usage) section in the Appendix.

### `@lwc/engine-dom`

This package exposes the following APIs:

- `createElement` + reaction hooks
- `buildCustomElementConstructor`
- `isNodeFromTemplate`

This package injects the native DOM APIs into the `@lwc/engine-core` rendering engine.

### `@lwc/engine-server`

This package exposes the following APIs:

- `createElement(name: string, options: { is: typeof LightningElement }): ServerHTMLElement`: This method creates a new LWC component tree. It follows the same signature as the `createElement` API from `@lwc/engine-dom`. Instead of returning a native `HTMLElement`, this method returns a `ServerHTMLElement` with the public properties, aria reflected properties and HTML global attributed.
- `renderToString(element: ServerHTMLElement): string`: This method creates an LWC component tree synchronously and serialize it to string. It accepts a single parameter, a `ServerHTMLElement` returned by `createElement` and returns the serialized string.

This package injects mock DOM APIs in the `@lwc/engine-core` rendering engine. Those DOM APIs produces a lightweight DOM structure that can be serialized into a string by the `renderToString` method. As described in the Appendix, this package is also in charge of attaching on the global object a mock `CustomEvent`.

## Drawbacks

- Requires a complete rewrite of the current `@lwc/engine`
- Might require substantial changes to the existing tools (jest preset, rollup plugins, ...) to load the right engine depending on the environment.

## Alternatives

### Using a DOM implementation in JavaScript before evaluating the engine

Prior the evaluation of the LWC engine, we would evaluate a DOM implementation write in JavaScript (like [jsdom](https://github.com/jsdom/jsdom) or [domino](https://github.com/fgnass/domino)). The compelling aspect of this approach is that it requires almost no change to the engine to work.

There are multiple drawbacks with this approach:

- Performance: The engine only relies on a limited set of well-known APIs, leveraging a full DOM implementation for SSR would greatly reduce the SSR performance.
- Predictability: By attaching the DOM interfaces on the global object, those APIs are not only exposed to the engine but also to the component author. Exposing such APIs to the component author might bring unexpected behavior when component code runs on the server.

## Adoption strategy

TBD

## How we teach this

TBD

## Unresolved questions

**How to load the right version of the engine when resolving the `lwc` identifier?**

For example, when running in Node.js `lwc` might resolve to `@lwc/engine-server` for SSR purposes, but also to `@lwc/engine-dom` when running test via jest and jsdom.

One way to solve this would be to import the platform-specific APIs from the appropriate module: `createElement` from `@lwc/engine-dom`, `createElement` and `renderToString` from `@lwc/engine-server` and use. Platform-agnostic APIs should be imported from `@lwc` (which becomes an alias to `@lwc/engine-core`).

---

## Appendix

### DOM APIs usage

We can break-down the current LWC DOM usages into 3 different categories:

- [DOM constructors used by the engine](#dom-constructors-used-by-the-engine)
- [DOM methods and accessors used by the engine](#dom-methods-and-accessors-used-by-the-engine)
- [DOM constructors used by component authors](#dom-constructors-used-by-component-authors)

#### DOM constructors used by the engine

The engine currently relies on the following DOM constructors during evaluation and at runtime:

##### `HTMLElement`

- **DOM usage:** Used by the engine to extract the descriptors and reassign them to the `LightningElement` prototype.
- **SSR usage:** 🔵 NOT REQUIRED
- **Observations:** We can create a hard-coded list of all the needed accessors: HTML global attributes, aria reflection properties and HTML the good part.

##### `ShadowRoot`

- **DOM usage:** Used by the engine to traverse the DOM tree to find the error boundary and create the component call stack.
- **SSR usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** Framework implementing the error boundary concept (like React or Vue), use equivalent of the LWC `VM.owner` back-pointer to traverse the component tree up instead of relying on the DOM tree. This discussion might require another RFC.

#### DOM methods and accessors used by the engine

On top of this the engine also rely on the following DOM methods and accessors:

##### `EventTarget.prototype.dispatchEvent()`

- **DOM usage:** Exposed via `LightningElement.prototype.dispatchEvent()`
- **SSR usage:** 🔴 REQUIRED
- **Observations:** Components may dispatch event once connected.

##### `EventTarget.prototype.addEventListener()`

- **DOM usage:** Exposed via `LightningElement.prototype.addEventListener()`. Used by the rendering engine to handle `on*` directive from the template.
- **SSR usage:** 🔴 REQUIRED

##### `EventTarget.prototype.removeEventListener()`

- **DOM usage:** Exposed via `LightningElement.prototype.removeElementListener()`.
- **SSR usage:** 🔴 REQUIRED

##### `Node.prototype.appendChild()`

- **DOM usage:** Used by the upgrade mechanism, synthetic shadow styling and to enforce restrictions.
- **SSR usage:** 🔵 NOT REQUIRED
- **Observations:** This API can be replaced by `Node.prototype.insertBefore(elm, null)`.

##### `Node.prototype.insertBefore()`

- **DOM usage:** Used by the upgrade mechanism, by the rendering engine, and to enforce restrictions.
- **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.removeChild()`

- **DOM usage:** Used by the upgrade mechanism, by the rendering engine, and to enforce restrictions.
- **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.replaceChild()`

- **DOM usage:** Used by the upgrade mechanism, and to enforce restrictions
- **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.parentNode` (getter)

- **DOM usage:** Used to traverse the DOM tree
- **SSR Usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** Depending on how the error boundary is implemented, we might be able to get rid of this API.

##### `Element.prototype.hasAttribute()`

- **DOM usage:** Used by the aria reflection polyfill
- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** For SSR, we will not need this polyfill

##### `Element.prototype.getAttribute()`

- **DOM usage:** Exposed via `LightningElement.prototype.getAttribute()`. Used by the aria reflection polyfill
- **SSR Usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** We might use `Element.prototype.getAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.getAttributeNS()`

- **DOM usage:** Exposed via `LightningElement.prototype.getAttributeNS()`
- **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.setAttribute()`

- **DOM usage:** Exposed via `LightningElement.prototype.setAttribute()`. Used by the rendering engine, for synthetic shadow styling and by the aria reflection polyfill.
- **SSR Usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** We might use `Element.prototype.setAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.setAttributeNS()`

- **DOM usage:** Exposed via `LightningElement.prototype.setAttributeNS()`. Used by the rendering engine.
- **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.removeAttribute()`

- **DOM usage:** Exposed via `LightningElement.prototype.removeAttribute()`. Used by the rendering engine, for the synthetic shadow styling and by the aria reflection polyfill
- **SSR Usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** We might use `Element.prototype.removeAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.removeAttributeNS()`

- **DOM usage:** Exposed via `LightningElement.prototype.removeAttributeNS()`.
- **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.getElementsByTagName()`

- **DOM usage:** Exposed via `LightningElement.prototype.getElementsByTagName()`.

- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `HTMLCollection` when running on the server.

##### `Element.prototype.getElementsByClassName()`

- **DOM usage:** Exposed via `LightningElement.prototype.getElementsByClassName()`.
- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `HTMLCollection` when running on the server.

##### `Element.prototype.querySelector()`

- **DOM usage:** Exposed via `LightningElement.prototype.querySelector()`. Used to enforce restrictions.
- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `NodeList` when running on the server.

##### `Element.prototype.querySelectorAll()`

- **DOM usage:** Exposed via `LightningElement.prototype.querySelectorAll()`. Used to enforce restrictions.
- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `NodeList` when running on the server.

##### `Element.prototype.getBoundingClientRect()`

- **DOM usage:** Exposed via `LightningElement.prototype.getBoundingClientRect()`.
- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** Running a layout engine during SSR, is a complex task that will not bring much values to component authors. Returning an empty `DOMRect` (where all the properties are set to `0`), might be the best approach here.

##### `Element.prototype.classList` (getter)

- **DOM usage:** Exposed via `LightningElement.prototype.classList`. Used by the rendering engine.
- **SSR Usage:** 🔴 REQUIRED

##### `ShadowRoot.prototype.host` (getter)

- **DOM usage:** Used to traverse the DOM.
- **SSR Usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** Depending on how the error boundary is implemented, we might be able to get rid of this API.

##### `ShadowRoot.prototype.innerHTML` (setter)

- **DOM usage:** Used to reset the shadow root.
- **SSR Usage:** 🔵 NOT REQUIRED
- **Observations:** This is a performance optimization to quickly reset the ShadowRoot. We can use `Node.prototype.removeChild` as a replacement.

##### HTML global attributes (setters and getters) and Aria reflection properties

- **DOM usage:** Exposed on the `LightningElement.prototype` Used by the rendering to set the properties on custom elements.
- **SSR Usage:** 🔴 REQUIRED

#### DOM constructors used by component authors

Finally, on top of all the APIs used by the engine to evaluate and run, component authors need to have access to the following DOM constructors in the context of SSR.

##### `CustomEvent`

- **DOM usage:** Used to dispatch events
- **SSR Usage:** 🔴 REQUIRED

##### `Event`

- **DOM usage:** Used to dispatch events
- **SSR Usage:** 🔶 MIGHT BE REQUIRED
- **Observations:** This might not be needed because `CustomEvent` inherits from `Event` and because `CustomEvent` is the recommended way to dispatch non-standard events.
