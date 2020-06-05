---
title: LWC server runtime
status: DRAFTED
created_at: 2020-01-06
updated_at: 2020-01-06
pr: https://github.com/salesforce/lwc-rfcs/pull/23
---

# LWC server runtime

## Summary

> TODO: Add more details

## Goals

Server-side Rendering (SSR) is a popular technique for rendering pages and components that would usually run in a browser on the server. SSR takes as input a root component and a set of properties and renders an HTML fragment and optionally one or multiple stylesheets. This technique can be used in many different ways: improve initial page render time, generate static websites, …

Due to the size and complexity of this feature, the rollout of SSR is broken up in 3 different phases and is spread over multiple releases:

-   **Phase 1:** Decouple the LWC engine from the DOM and allow it to run in a JavaScript environment without access to the DOM APIs (like Node.js). As part of this phase, the LWC server engine will also generate a prototype version of the serialized HTML (Phase 2).
-   **Phase 2:** Settle on an HTML serialization format.
-   **Phase 3:** Rehydrate in a browser the DOM tree resulting from the serialization into an LWC component tree.

As part of phase 1, the goal is to provide a capability for LWC to render components transparently on the server. From the component standpoint, the LWC SSR and DOM versions provide identical APIs and will run the component in the same fashion on both platforms.

## Proposal Scope

This proposal will be only focussing on phase 1 of the SSR project. Phase 2 and 3 will be covered in subsequent proposals. This proposal cover 2 main aspects of the LWC SSR:

-   What are the different approach in implement SSR in a UI framework?
-   How to create 2 different implementations of the LWC engine while keeping the core logic shared?
-   What new APIs need to be exposed to render components on the server?
-   What constraints need to be put in place to ensure that SSR is reliable and performant?

## Detailed design

### Approaching SSR in UI framework

Many popular UI frameworks already provide options to enable server-side rendering. There is no single way to implement SSR in a UI framework and each framework approaches this in a different way. That being said all the major frameworks converge around 2 main approaches.

The first approach is to **polyfill the server environment with DOM APIs and to reuse the same core UI framework code on the server and the client**. This is the approach [Stencil](https://github.com/ionic-team/stencil/master/tree/src/mock-doc), [Ember](https://github.com/ember-fastboot/simple-dom) and [lit-ssr](https://github.com/PolymerLabs/lit-ssr).

This approach is convenient because it requires almost no change to the core UI framework to enable SSR. Since most of the DOM APIs used by the UI framework are low-level API, most of them are easy to polyfill. Popular DOM APIs implementations like [domino](https://github.com/fgnass/domino) or [jsdom](https://github.com/jsdom/jsdom) can be used to avoid having to write to polyfills.

This approach also suffers from multiple drawbacks. The main issue is that the polyfill has to stay in sync with the core UI framework. As the core UI framework evolves and the DOM APIs used by the engine changes, developers have to always double-check that the newly introduced APIs are also polyfilled properly. The other main drawback is that applying polyfills to evaluate the engine the same DOM interfaces and methods are also available from the component code (eg. `window`, `document`, ...). This gives a false sense that the component is running in the browser.

Using the polyfilling approach, the existing `@lwc/engine` stays untouched and the entry point of the SSR version of LWC would look something like this.

**`index.ts`:**

```ts
// Evaluate fist the DOM polyfills, then the DOM engine version of the engine. Reexport all the
// exported properties from the DOM engine.
import './dom-environment';
export * from '@lwc/engine';
```

**`dom-environment.ts`:**

```ts
class Node {
    children: Node[] = [];

    insertBefore(child: Node, anchor: Node): void {
        const anchorIndex = this.children.indexOf(anchor);

        if (anchorIndex === -1) {
            this.children.push(child);
        } else {
            this.children.splice(anchorIndex, 0, child);
        }
    }
}

class Element extends Node {
    tagName: string;

    constructor(tagName: string) {
        super();
        this.tagName = tagName;
    }
}

const document = {
    createElement(tagName: string): Element {
        return new Element(tagName);
    },
};

const window = {
    document,
};

// Assigning the synthetic DOM APIs to the current environment global object.
Object.assign(globalThis, {
    window,
    document,
    Node,
    Element,
});
```

The second approach is to **abstract the DOM APIs and the core UI framework**. This approach is used by [React](https://github.com/facebook/react/tree/master/packages/react-dom), [Vue](https://github.com/vuejs/vue-next/tree/master/packages/runtime-dom) and [Angular](https://github.com/angular/angular/tree/master/packages/platform-browser/src). This approach consists of creating an indirection between the rendering APIs and the core framework code. The indirection is injected at runtime depending on the environment. When loaded in the browser the rendering APIs create DOM `Element` and set DOM `Attribute` on the elements. While when loaded server the rendering APIs are manipulating string to create the serialized version of the components on the fly.

When used with a type-safe language like TypesScript, it is possible to ensure at compile time that all the rendering APIs are fulfilled in all the environments. By injecting at runtime this indirection, it also ensures that component code doesn't have access to any DOM APIs when running on the server.

The main drawback of this approach is the amount of refactoring that needs to happen in LWC engine code specifically. So far the LWC engine has been designed to run in JavaScript environment with direct access to the DOM APIs. This means that if we were to adopt this approach a large chunk of the LWC engine code will need to be rewritten. For example some of the LWC engine code store a reference for DOM interfaces (eg. `HTMLElement.prototype`, `Node.prototype`) at evaluation time. This will not be possible with this approach as the APIs will be injected into the core engine after its evaluation.

To inject the rendering APIs depending on the environment, we would need to create a different entry point per environment. You can find below some pseudo-code for each entry point. The 2 entry points use the same `createComponent` abstraction to create an LWC component but pass a renderer specific to the targeted environment.

**`entry-points/dom.js`:**

```ts
import { LightningElement, createComponent } from '@lwc/engine-core';

const domRenderer = {
    createElement(tagName: string): Element {
        return document.createElement(tagName);
    },
    insertBefore(node: Node, parent: Element, anchor: Node): void {
        return parent.insertBefore(node, anchor);
    },
};

// The method dedicated to create an LWC component in a dom environment. It returns a DOM element
// that can then inserted into the document to render.
export function createElement(
    tagName: string,
    opts: { Ctor: typeof LightningElement },
): Element {
    const elm = document.createElement(tagName);
    createComponent(elm, Ctor, domRenderer);
    return elm;
}
```

**`entry-points/server.js`:**

```ts
import { LightningElement, createComponent } from '@lwc/engine-core';

interface ServerNode {
    children: ServerNode[];
}

interface ServerElement extends ServerNode {
    tagName: string;
}

const serverRenderer = {
    createElement(tagName: string): ServerElement {
        return {
            tagName,
            children: [],
        };
    },
    insertBefore(
        node: ServerNode,
        parent: ServerElement,
        anchor: ServerNode,
    ): void {
        const anchorIndex = parent.children.indexOf(anchor);

        if (anchorIndex === -1) {
            parent.children.push(child);
        } else {
            parent.children.splice(anchorIndex, 0, child);
        }
    },
};

// The method dedicated to create an LWC component in a server environment. It returns a string
// resulting of the serialization component tree
export function renderComponent(
    tagName: string,
    opts: { Ctor: typeof LightningElement },
): string {
    const elm: ServerElement = {
        tagName,
        children: [],
    };

    createComponent(elm, Ctor, serverRenderer);

    return serializeElement(elm);
}
```

**Conclusion:** After considering the pros and cons of the 2 approaches described above, **we decided to go with the second approach when the rendering APIs are injected lazily at runtime**.

### Splitting `@lwc/engine`

As discussed in the previous section, an abstraction of the underlying rendering APIs into the core framework depending on the target environment. To share the core UI framework between the different environments the existing `@lwc/engine` will be split into 3 packages.

#### `@lwc/engine-core`

This package contains core logic shared by the different runtimes environment including the rendering engine and the reactivity mechanism. It should never be consumed directly in an application. It only provides internal APIs for building custom runtimes. This package exposes the following platform-agnostic public APIs:

-   `LightningElement`
-   `api`
-   `track`
-   `readonly`
-   `wire`
-   `setFeatureFlag`
-   `getComponentDef`
-   `isComponentConstructor`
-   `getComponentConstructor`
-   `unwrap`
-   `registerTemplate`
-   `registerComponent`
-   `registerDecorators`
-   `sanitizeAttribute`

On top of the public APIs, this package also exposes new internal APIs that are meant to be consumed

-   `getComponentInternalDef(Ctor: typeof LightningElement): ComponentDef`: Get the internal component definition for a given LightningElement constructor.
-   `createVM(elm: HostElement, Ctor, options: { mode: 'open' | 'closed', owner: VM | null, renderer: Renderer }): VM`: Create a new View-Model (VM) associated with an LWC component.
-   `connectRootElement(elm: HostElement): void`: Mount a component and trigger a rendering.
-   `disconnectRootElement(elm: HostElement): void`: Unmount a component and trigger a disconnection flow.
-   `getAssociatedVMIfPresent(elm: HostElement): VM | undefined`: Retrieve the VM on a given element.
-   `setElementProto(elm: HostElement): void`: Patch an element prototype with the bridge element.

The current `@lwc/engine` code relies on direct DOM invocation. The list of all the current DOM APIs the engine depends upon can be found in the [DOM APIs usage](#dom-apis-usage). Those direct DOM APIs invocations will be replaced by a [`Rendered` interface](#renderer-interface) that will be injected at runtime into the `createVM`. All the sub-components created from the root VM will share the same `Renderer` interface.

#### `@lwc/engine-dom`

A runtime that can be used to render LWC component trees in a DOM environment. On top of re-exporting all the public APIs from the `@lwc/engine-core` package, this package also exposes the following DOM environment specific APIs:

-   `createElement(name: string, ctor: typeof LightningElement): HTMLElement`
-   `isNodeFromTemplate(node: Node): boolean`
-   `getComponentConstructor(element: HTMLElement): typeof LightningElement | null`
-   `buildCustomElementConstructor(ctor: typeof LightningElement): typeof HTMLElement` (deprecated)

#### `@lwc/engine-server`

A runtime that can be used to render LWC component trees as strings. Like the `@lwc/engine-dom`, this package re-exports all the public APIs from the `@lwc/engine-core` package along with:

-   `renderComponent(name: string, ctor: typeof LightningElement, props?: { [key: string]: any }): string`: This method creates an renders an LWC component synchronously to a string. It will be discussed further in the following section.

### Rendering an LWC component on the server

#### Constraints

To make the LWC SSR predictable and performant, only a certain subset of the LWC engine capabilities available on the client will be present on the server. As a side-effect, LWC components that need to be rendered on the server will have to observe the following constraints.

**No access to web platform APIs on the server:** When running in the server environment, LWC will not polyfill web platform-specific APIs. Because of this, accessing any of those APIs as the component renders on the server, will result in a runtime exception. For example, if a server-side renderer component wants to attach event listeners to the `document` when it is rendered on the client, it needs to check first if the `document` object is present in the current runtime environment.

**The entire component tree will be rendered in a single pass synchronously:** This behavior matches the current behavior of the LWC engine. Connecting an LWC component to a document triggers a synchronous rendering cycle. This means that reactivity is unnecessary on the server. Disabling the reactive membrane on the server will also improve the overall SSR performance by avoiding creating unnecessary JavaScript `Proxy`. This also means that if a component need to do an asynchronous operation to fetch data to render, it will not be possible to do so when rendered on the server. All the asynchronous data dependencies for a component subtree needs to be retrieved prior rendering the component and needs to be passed via public property for it to be rendered.

**The `renderedCallback` lifecycle hook will not execute on the server:** When running a browser, this hook is the first life cycle hook which give the component author access to the rendered DOM elements. If the component were to attempt to access those APIs on the server it would results in a runtime error since the DOM APIs are not polyfilled.

**Wire adapters will not be invoked:** The Wire protocol emits a stream of data to a component. The current protocol doesn't define today a way today to indicate that the stream is done emitting new data. Because of this, the first version of SSR will not invoke any of the wire adapter. The protocol will need to be changed and new primitives will need to be added to LWC to make wire adapter compatible with SSR.

#### The `renderComponent` API

The `renderComponent` is the only new public API exposed with this proposal. This new API is only available in `@lwc/engine-server`. It renders a component and returns synchronously the rendered content. This proposal accepts 3 arguments:
 - `name` (type: `string`) - the tag name of the component host element.
 - `ctor` (type: `typeof LightningElement`) - the root LWC component constructor.
 - `props` (optional, type: `{ [key string]: any}`) - an object representing the different properties set on the root component.

This method returns a the serialized HTML content rendered by the component. The serialization format is out of the scope of this proposal and will be covered in a different RFC. However in the first version of the `@lwc/engine-server`, the `renderComponent` will produce an HTML string matching the [declarative shadow DOM proposal](https://github.com/mfreed7/declarative-shadow-dom/blob/master/README.md) format.

#### Component Authoring format

The existing `lwc` package stays untouched and will be used to distribute the different versions of the engine. From the developer perspective, the experience writing a component remains identical. Since the LWC engine exposes different API depending on the environment, the application owner will be in charge of creating a different entry point for each environment.

**`c/app/app.js`:**

```js
import { LightningElement } from 'lwc';

export default class App extends LightningElement {}
```

**`client.js`:**

```js
import { createElement } from 'lwc'; // Resolves to `@lwc/engine-dom`
import App from 'c/app';

const app = createElement('c-app', { is: App });
document.body.appendChild(app);
```

**`server.js`:**

```js
import { renderComponent } from '@lwc/engine-server'; // Resolves to `@lwc/engine-server`
import App from 'c/app';

const str = renderComponent('c-app', { is: App });
console.log(str);
```

## How we teach this

- Updating the documentation for the newly added server only APIs should be enough.
- Creating a set of a new linting rules prevent obvious cases where components can't be rendered on the server.

## Unresolved questions

-   **Remove `@lwc/engine-core` on TypeScript `dom` library?** The `@lwc/engine-core` package relies heavily on the ambient DOM TypeScript interfaces provided by the `dom` library. To ensure that the `@lwc/engine-core` is not leveraging any of the DOM APIs we will need to remove the `dom` lib from the `tsconfig.json`. It is currently unclear how all the ambient types can be removed on this package while ensuring the type safety.
-   **How to implement LWC Context for SSR?** Context uses eventing for registration between the provider and the consumer. Since `@lwc/engine-server` will only implement a subset of the DOM eventing, we will need to evaluate how we can replace the current registration mechanism.

---

## Appendix

### DOM APIs usage

We can break-down the current LWC DOM usages into 3 different categories:

-   [DOM constructors used by the engine](#dom-constructors-used-by-the-engine)
-   [DOM methods and accessors used by the engine](#dom-methods-and-accessors-used-by-the-engine)
-   [DOM constructors used by component authors](#dom-constructors-used-by-component-authors)

#### DOM constructors used by the engine

##### `HTMLElement`

-   **DOM usage:** Used by the engine to extract the descriptors and reassign them to the `LightningElement` prototype.
-   **SSR usage:** 🔵 NOT REQUIRED
-   **Observations:** We can create a hard-coded list of all the needed accessors: HTML global attributes, aria reflection properties and HTML the good part.

#### DOM methods and accessors used by the engine

On top of this the engine also rely on the following DOM methods and accessors:

##### `EventTarget.prototype.dispatchEvent()`

-   **DOM usage:** Exposed via `LightningElement.prototype.dispatchEvent()`
-   **SSR usage:** 🔴 REQUIRED
-   **Observations:** Components may dispatch event once connected.

##### `EventTarget.prototype.addEventListener()`

-   **DOM usage:** Exposed via `LightningElement.prototype.addEventListener()`. Used by the rendering engine to handle `on*` directive from the template.
-   **SSR usage:** 🔴 REQUIRED

##### `EventTarget.prototype.removeEventListener()`

-   **DOM usage:** Exposed via `LightningElement.prototype.removeElementListener()`.
-   **SSR usage:** 🔴 REQUIRED

##### `Node.prototype.appendChild()`

-   **DOM usage:** Used by the upgrade mechanism, synthetic shadow styling and to enforce restrictions.
-   **SSR usage:** 🔵 NOT REQUIRED
-   **Observations:** This API can be replaced by `Node.prototype.insertBefore(elm, null)`.

##### `Node.prototype.insertBefore()`

-   **DOM usage:** Used by the upgrade mechanism, by the rendering engine, and to enforce restrictions.
-   **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.removeChild()`

-   **DOM usage:** Used by the upgrade mechanism, by the rendering engine, and to enforce restrictions.
-   **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.replaceChild()`

-   **DOM usage:** Used by the upgrade mechanism, and to enforce restrictions
-   **SSR Usage:** 🔴 REQUIRED

##### `Node.prototype.parentNode` (getter)

-   **DOM usage:** Used to traverse the DOM tree
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** Depending on how the error boundary is implemented, we might be able to get rid of this API.

##### `Element.prototype.hasAttribute()`

-   **DOM usage:** Used by the aria reflection polyfill
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** For SSR, we will not need this polyfill

##### `Element.prototype.getAttribute()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getAttribute()`. Used by the aria reflection polyfill
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** We might use `Element.prototype.getAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.getAttributeNS()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getAttributeNS()`
-   **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.setAttribute()`

-   **DOM usage:** Exposed via `LightningElement.prototype.setAttribute()`. Used by the rendering engine, for synthetic shadow styling and by the aria reflection polyfill.
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** We might use `Element.prototype.setAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.setAttributeNS()`

-   **DOM usage:** Exposed via `LightningElement.prototype.setAttributeNS()`. Used by the rendering engine.
-   **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.removeAttribute()`

-   **DOM usage:** Exposed via `LightningElement.prototype.removeAttribute()`. Used by the rendering engine, for the synthetic shadow styling and by the aria reflection polyfill
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** We might use `Element.prototype.removeAttributeNS(null, name)` to reduce the API surface needed by the engine.

##### `Element.prototype.removeAttributeNS()`

-   **DOM usage:** Exposed via `LightningElement.prototype.removeAttributeNS()`.
-   **SSR Usage:** 🔴 REQUIRED

##### `Element.prototype.getElementsByTagName()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getElementsByTagName()`.

-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `HTMLCollection` when running on the server.

##### `Element.prototype.getElementsByClassName()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getElementsByClassName()`.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `HTMLCollection` when running on the server.

##### `Element.prototype.querySelector()`

-   **DOM usage:** Exposed via `LightningElement.prototype.querySelector()`. Used to enforce restrictions.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `NodeList` when running on the server.

##### `Element.prototype.querySelectorAll()`

-   **DOM usage:** Exposed via `LightningElement.prototype.querySelectorAll()`. Used to enforce restrictions.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** This method is only exposed to select elements from the component light DOM, which is only available from the `renderedCallback`. Since SSR doesn't run `renderedCallback`, we can always returns an empty `NodeList` when running on the server.

##### `Element.prototype.getBoundingClientRect()`

-   **DOM usage:** Exposed via `LightningElement.prototype.getBoundingClientRect()`.
-   **SSR Usage:** 🔵 NOT REQUIRED
-   **Observations:** Running a layout engine during SSR, is a complex task that will not bring much values to component authors. Returning an empty `DOMRect` (where all the properties are set to `0`), might be the best approach here.

##### `Element.prototype.classList` (getter)

-   **DOM usage:** Exposed via `LightningElement.prototype.classList`. Used by the rendering engine.
-   **SSR Usage:** 🔴 REQUIRED

##### HTML global attributes (setters and getters) and Aria reflection properties

-   **DOM usage:** Exposed on the `LightningElement.prototype` Used by the rendering to set the properties on custom elements.
-   **SSR Usage:** 🔴 REQUIRED

#### DOM constructors used by component authors

Finally, on top of all the APIs used by the engine to evaluate and run, component authors need to have access to the following DOM constructors in the context of SSR.

##### `CustomEvent`

-   **DOM usage:** Used to dispatch events
-   **SSR Usage:** 🔴 REQUIRED

##### `Event`

-   **DOM usage:** Used to dispatch events
-   **SSR Usage:** 🔶 MIGHT BE REQUIRED
-   **Observations:** This might not be needed because `CustomEvent` inherits from `Event` and because `CustomEvent` is the recommended way to dispatch non-standard events.

### `Renderer` interface

```ts
export interface Renderer<HostNode, HostElement> {
    syntheticShadow: boolean;
    insert(node: HostNode, parent: HostElement, anchor: HostNode | null): void;
    remove(node: HostNode, parent: HostElement): void;
    createElement(tagName: string, namespace?: string): HostElement;
    createText(content: string): HostNode;
    nextSibling(node: HostNode): HostNode | null;
    attachShadow(element: HostElement, options: ShadowRootInit): HostNode;
    setText(node: HostNode, content: string): void;
    getAttribute(
        element: HostElement,
        name: string,
        namespace?: string | null,
    ): string | null;
    setAttribute(
        element: HostElement,
        name: string,
        value: string,
        namespace?: string | null,
    ): void;
    removeAttribute(
        element: HostElement,
        name: string,
        namespace?: string | null,
    ): void;
    addEventListener(
        target: HostElement,
        type: string,
        callback: EventListenerOrEventListenerObject,
        options?: AddEventListenerOptions | boolean,
    ): void;
    removeEventListener(
        target: HostElement,
        type: string,
        callback: EventListenerOrEventListenerObject,
        options?: AddEventListenerOptions | boolean,
    ): void;
    dispatchEvent(target: HostNode, event: Event): boolean;
    getClassList(element: HostElement): DOMTokenList;
    getStyleDeclaration(element: HostElement): CSSStyleDeclaration;
    getBoundingClientRect(element: HostElement): ClientRect;
    querySelector(element: HostElement, selectors: string): HostElement | null;
    querySelectorAll(element: HostElement, selectors: string): NodeList;
    getElementsByTagName(
        element: HostElement,
        tagNameOrWildCard: string,
    ): HTMLCollection;
    getElementsByClassName(element: HostElement, names: string): HTMLCollection;
    isConnected(node: HostNode): boolean;
    tagName(element: HostElement): string;
}
```