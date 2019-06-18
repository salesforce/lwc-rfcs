- Start Date: 2019-06-01
- RFC PR: (leave this empty)
- Lightning Web Component Issue: (leave this empty)

# Summary

Context provides a way to pass data through the component tree without having to pass props down manually at every level.

# Motivation

In a web component, data is passed top-down (host to child elements on its shadowRoot) via attributes, but this can be cumbersome for certain types of attributes (e.g. UI theme) that are required by many components within an application. Context provides a way to share values like these between components without having to explicitly pass a prop through every level of the tree.

* *Common use case*: receiving values from a component higher “up” the DOM tree
when you don't own the components in between.
* A context allows a provider component to provision a stream of immutable
values to multiple components “lower” in the DOM.

# Goals

The primary goal of this RFC is to define how to consume context, guarantee the right ergonomics and semantics for the consumer code while the provider can be an experimental API.

As a secondary goal is to provide ways for the Lightning Platform to control who can provide new context values, as a way to gate this feature.

A third goal is to support the provision of context values on any element, whether it is LWC or not.

# Prior Art

* React Context: https://reactjs.org/docs/context.html

# Detailed design

Important Notes:

* It is important for the consumer to be able to consume more than one context value from different context providers if needed.
* It is important to guarantee that the provisioned object is reactive, so the UI can
rely on it directly.

## Proposal: Context Consumer

This proposal provides a high-level API (an abstraction layer) for authors to consume a context value at any level in the DOM flatten tree. This must be a future-proof API, and must be easy to use and easy to maintain. Here is an example:

```js
import { LightningElement } from 'lwc';
import XFooContext from 'x/fooContext';
import XBarContext from 'x/barContext';

export default class MyComponent extends LightningElement {
    // context value provided by <x-foo-context> wired into a field
    @wire(XFooContext.Provider) a;
    // context value provided by <x-bar-context> wired into a method
    @wire(XBarContext.Provider) someMethod(contextValue) {
        // contextValue available after connecting
    }
}
```

Pros:
* it could be statically verified when wired to a field.
* it works well with inheritance since it relies on field declarations.
* it is familiar since it relies on the `wire` decorator.

Cons:
* the wire decorator is often seen as more complicated, and often requires authors to be defensive in case the value is never resolved (which is not the case of for context).
* it requires changes in the wire protocol to expand its capabilities to support wiring based on flatten DOM tree. (more on this below)

_Note: This is very similar to how React does it, but extending it to access multiple contexts._

_Note: since `@wire` supports wiring to a method, it will work the same way for context values._

## Proposal: Context Provider

This proposal provides a way for authors to provide a context value at any level in the flattened DOM tree that can be consumed by any node in the subtree.

### Defining a new Context Provider

The definition of a new context is equivalent to creation of a component that must extend `LightningContext`, e.g.:

```js
import LightningContext from 'lightning/context';

export default class ThemeContext extends LightningContext {
    @api set theme() {
        this.setContextValue(v);
    }
    get theme() {
        return this.getContextValue();
    }
}
```

Where `LightningContext` component is provided by the `lightning` namespace. This class is intended to be extended and provides two protected methods `setContextValue(newValue)` and `getContextValue()` to interact with the underlying implementation that takes care of provisioning  the context value to any node consuming this new context.

_Note: No html file is required, unless the author want to customize the UI of the context provider._

This class also provides a static accessor called `Provider`, which is intended to be used for the consumer to wire to the provided context, e.g.:

```js
import { LightningElement } from 'lwc';
import ThemeContext from 'x/themeContext';

// Consumer
export default class XBar extends LightningElement {
    @wire(ThemeContext.Provider) x;
}
```

### Providing a context value

Setting a context value must be an operation involving a custom element. Any element in the  light DOM subtree of that element or their descendent, can consume the value set for a context.

To provide a new context value, you must use the declarative form via an HTML template. Any structure that involves `<x-theme-context>` and `<x-bar>` components can share the `theme` value, e.g.: 

```html
<template>
    <x-theme-context theme="dark">
        <x-bar></x-bar>
    </x-theme-context>
</template>
```

It means that `<x-bar>` or any of its descendants can wire to `ThemeContext.Provider` to obtain the `theme` value.

## Implementation Details

In order to implement `LightningContext`, it requires a reform on the the wire protocol. This reforms has two folds:

1. We need to introduce a new event that can be dispatched by a wire adapter to create a side channel between the `eventTarget` with a context provider installed on an element in the the composed path. The new event is implemented via a new constructor called `LinkContextEvent`, and it expects a unique `token`, and a `callback` to finalize the linking process. E.g.:

```js
import { register, ValueChangedEvent, LinkContextEvent } from 'wire-service';

const adapter = Symbol('ContextProvider');
register(adapter, (eventTarget) => {
    let unsubscribeCallback;

    function callback(data, unsubscribe) {
        // provider calls this function at any given time
        eventTarget.dispatchEvent(new ValueChangedEvent(data));
        unsubscribeCallback = unsubscribe;
    }

    eventTarget.addEventListener('connect', () => {
        // attempt to link to a context provider identified by token
        const event = new LinkContextEvent(token, callback);
        eventTarget.dispatchEvent(event);
    });

    eventTarget.addEventListener('disconnect', () => {
        if (unsubscribeCallback !== undefined) {
            unsubscribeCallback();
        }
    });
});
```

2. The existing, and hacky way, to implement context with wire via an arbitrary event name `"WireContextEvent"`, must be removed. This cannot happen right now, we should move existing consumers to the new API.

# Adoption strategy

This is a brand new feature, we just need to document it.

# How we teach this

* The consumer aspect of it is very straight forward considering that we have plenty of documentation about the `@wire` decorator.

# Unresolved questions

* ~~How can the context be propagated when the fiber is cut (insertion of root elements inside a template)?~~ Events coordination solves this.
* ~~How can the context be provided to elements in the light DOM of the context element?~~ Events coordination solves this.
* Is the value always synchronous provisioned during the `wiring` stage of the component life-cycle?
* Is there a formal error state that the wire can provision? Will it participate in the component error boundary?
