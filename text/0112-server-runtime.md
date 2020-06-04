---
title: LWC server runtime
status: DRAFTED
created_at: 2020-01-06
updated_at: 2020-01-06
pr: https://github.com/salesforce/lwc-rfcs/pull/23
---

# LWC server runtime

## Summary

The LWC engine has been designed from the beginning to run in an environment with access to the DOM APIs. To accommodate server-side rendering (SSR) requirements, we need a way to decouple the engine from the DOM APIs. The existing `@lwc/engine` will be replaced by 3 new packages: 
- `@lwc/engine-core` exposes platform-agnostic APIs and will be used by the different runtime packages to share the common logic.
- `@lwc/engine-dom` exposes LWC APIs available on the browser.
- `@lwc/engine-server` exposes LWC APIs used for server-side rendering.

>> TODO: Add more details

## Goals

Server Side Rendering (SSR) is a popular technique for rendering pages and components that would usually run in a browser on the server. SSR takes as input a root component and a set of properties and renders an HTML fragment and optionally one or multiple stylesheets. This technique can be used in many different ways: improve initial page render time, generate static website, …

Due to the size and complexity of this feature, the rollout of SSR is broken up in 3 different phases and is spread over multiple releases:
- **Phase 1:** Decouple the LWC engine from the DOM and allow it to run in a JavaScript environment without access to the DOM APIs (like Node.js). As part of this phase the LWC server engine will also generate a prototype version of the serialized HTML (Phase 2).
- **Phase 2:** Settle on a HTML serialization format
- **Phase 3:** Rehydrate in a browser the DOM tree resulting of the serialization into an LWC component tree

As part of phase 1, the goal is to provide a capability for LWC to render components transparently on the server. From the component stand point, the LWC SSR and DOM versions provide identical APIs and will run the component in the same fashion on both platforms.

## Proposal Scope

This proposal will be only focussing on the phase 1 of the SSR project. The phase 2 and 3 will be covered in subsequent proposals. This proposal cover 2 main aspects of the LWC SSR:

- What are the differences between the DOM version and the SSR version of LWC?
- How to create 2 different implementation of the LWC engine while keeping the core logic shared.

## Detailed design

The existing `@lwc/engine` will be replaced by 3 new packages:

- `@lwc/engine-core`: Core logic shared by the different runtimes including the rendering engine and the reactivity mechanism. This package should never be consumed directly in an application. This package is agnostic on the underlying rendering medium. It only provides APIs for building custom runtimes.
- `@lwc/engine-dom`: Runtime that can be used to render LWC component trees in a DOM environment. This package is built on top of `@lwc/engine-core`.
- `@lwc/engine-server`: Runtime that can be used to render LWC component trees as strings. This package is built on top of `@lwc/engine-core`.

The existing `lwc` package stays untouched and will be used to distribute the different versions of the engine. From the developer perspective, the experience should be identical.

**`c/app/app.js`:**

```js
import { LightningElement } from "lwc";

export default class App extends LightningElement {}
```

**`client.js`:**

```js
import { createElement } from "@lwc/engine-dom";
import App from "c/app";

const app = createElement("c-app", { is: App });
document.body.appendChild(app);
```

**`server.js`:**

```js
import { createElement, renderToString } from "@lwc/engine-server";
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
- `isNodeFromTemplate`

This package injects the native DOM APIs into the `@lwc/engine-core` rendering engine.

### `@lwc/engine-server`

This package exposes the following API:

- `renderComponent(name: string, ctor: typeof LightningElement, props?: Record<string, any>): string`: This method creates an renders an LWC component synchronously to a string.

When a component is rendered using `renderComponent`, the following restriction applies: 
  - each created component will execute the following life-cycle hooks once `constructor`, `connectedCallback` and `render`
  - properties and methods annotated with `@wire` will not be invoked
  - component reactivity is disabled

Another API might be created to accommodate wire adapter invocation and asynchronous data fetching on the server.

This package injects mock DOM APIs in the `@lwc/engine-core` rendering engine. Those DOM APIs produce a lightweight DOM structure. This structure can then be serialized into a string containing the HTML serialization of the element's descendants. As described in the Appendix, this package is also in charge of attaching on the global object a mock `CustomEvent` and only implements a restricted subset of the event dispatching algorithm on the server (no bubbling, no event retargeting).

## Drawbacks

- Requires a complete rewrite of the current `@lwc/engine`
- Might require substantial changes to the existing tools (jest preset, rollup plugins, etc.) to load the right engine depending on the environment.

## Alternatives

### Using a DOM implementation in JavaScript before evaluating the engine

Prior the evaluation of the LWC engine, we would evaluate a DOM implementation write in JavaScript (like [jsdom](https://github.com/jsdom/jsdom) or [domino](https://github.com/fgnass/domino)). The compelling aspect of this approach is that it requires almost no change to the engine to work.

There are multiple drawbacks with this approach:

- Performance: The engine only relies on a limited set of well-known APIs, leveraging a full DOM implementation for SSR would greatly reduce the SSR performance.
- Predictability: By attaching the DOM interfaces on the global object, those APIs are not only exposed to the engine but also to the component author. Exposing such APIs to the component author might bring unexpected behavior when component code runs on the server.

## How we teach this

This proposal breaks up the existing `@lwc/engine` package into multiple packages. Because of this APIs that used to be imported from `lwc` might not be present anymore. With this proposal, the `createElement` API will be moving to `@lwc/engine-dom`. This change constitutes a breaking change by itself, so we need to be careful how this change will be released. The good news is that `createElement` is considered as an experimental API and should only used to create the application root and to test components.

Changing the application root creation consists in a one line change. Therefor it should be pretty straightforward updated as long as it is well documented. Updating all the usages of `createElement` in test will probably require more time. For testing purposes, the `lwc` module will be mapped to the `@lwc/engine-dom` instead of `@lwc/engine-core` for now. We will also add warning messages in the console to promulgate the new pattern. A codemod for test files can also be used rename the `lwc` import in the test files to `@lwc/engine-dom`.


Updating the documentation for the newly added server only APIs should be enough.

## Unresolved questions

* **Remove `@lwc/engine-core` on TypeScript `dom` library?** The `@lwc/engine-core` package relies heavily on the ambient DOM TypeScript interfaces provided by the `dom` library. To ensure that the `@lwc/engine-core` is not leveraging any of the DOM APIs we will need to remove the `dom` lib from the `tsconfig.json`. It is currently unclear how all the ambient types can be removed on this package while ensuring the type safety.
* **How to implement LWC Context for SSR?** Context uses eventing for registration between the provider and the consumer. Since `@lwc/engine-server` will only implement a subset of the DOM eventing, we will need to evaluate how we can replace the current registration mechanism.

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
