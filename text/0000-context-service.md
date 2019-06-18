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

The primary goal of this RFC is to define how the to consume context, guarantee the right ergonomics and semantics for the consumer code while the provider can be experimental APIs.

As a secondary goal is to provide ways for the Lightning Platform to control who can provide new context values, as a way to gate this feature.

A third goal is to support the provision of context values on any element, whether it is LWC or not.

# Prior Art

* React Context: https://reactjs.org/docs/context.html

# Detailed design

Important Notes:

* It is important for the consumer to be able to consume more than one context value
from different context providers if needed.
* It is important to guarantee that the provisioned object is reactive, so the UI can
rely on it directly.

## Proposal: Context Consumer

This proposal provides a high-level API (an abstraction layer) for authors to consume a context value at any level in the DOM flatten tree. This must be a future-proof API, and must be easy to use and easy to maintain. Here is an example:

```js
import { LightningElement } from 'lwc';
import { XFooContext } from 'x/fooContext';
import { XBarContext } from 'x/barContext';

export default class MyComponent extends LightningElement {
    @wire(XFooContext) a; // context object provided by an element up in the flatten tree
    @wire(XBarContext) b; // context object provided by an element up in the flatten tree
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

_Note: since `@wire` supports wiring to a method, it will work the same way for context._

## Proposal: Context Provider

This proposal provides a low-level API for authors to provide a context value at any level in the flattened DOM tree that can be consumed by any node in the subtree. This API is subject to change as we understand more and more about how context is used.

### Defining a new Context

The definition of a new context is equivalent to creation of a new wire adapter. It has an identity, and it has some internal implementation details that are necessary to work as the first argument of the `@wire` decorator.

This API must be experimental for the MVP:

```js
import { createContextExperimental } from 'lwc';

export const XFooContext = createContextExperimental({
    x: 0,
    y: 0,
});
```

Where the first argument to `createContextExperimental()` function receives the default payload to be provided to any node consuming this new context.

_Note: we could add a second argument for a context value validation routine if needed._

The consumer of this context is as described in the previous section of this document for `XFooContext`.

## Providing a custom value for a context

Setting the value of a context already defined must be an operation involving an element. Any element in the  flatten DOM subtree of that element or its shadowRoot, can consume the value set for a context.

This API must be experimental for the MVP:

```js
import { setContextExperimental } from 'lwc';
import { XFooContext } from 'x/fooContext';

setContextExperimental(elm, XFooContext, {
    x: 1,
    y: 2,
});
```

Where the first argument is the element that provides the new context value into its flatten DOM subtree, the second argument is the defined context reference returned from `createContextExperimental` function call, and the 3rd argument is the _context value_ to be provided.

Invariants:

* `setContextExperimental()` must be called at least once before the insertion into the DOM phase is completed.

Observations:
* LWC is not opinionated on how a context definition is going to be shared. User-land abstractions are allowed.
* LWC is not opinionated on how a developer will share the capabilities to set a new context value on an element. User-land abstractions are allowed.

## Other examples

### How to implement a React-like context HOC in LWC?

```js
import { LightningElement, createContextExperimental, setContextExperimental } from 'lwc';

// creating a new context definition
const Context = createContextExperimental({
    x: 0,
    y: 0,
});

export default class XFooContext extends LightningElement {
    _value = {};

    @api
    get theme() {
        return this._value;
    }
    set theme(v) {
        // you can validate `v` here if you want
        this._value = v;
        // setting a new context value 
        this.setNewContextValue();
    }

    connectedCallback() {
        // setting the initial value of the context
        this.setNewContextValue();
    }
    setNewContextValue() {
        setContextExperimental(this, Context, this._value);
    }
    static Consumer = Context;
}
```

The way a provider is used in a template is very straight forward:

```html
<template>
    <x-foo-context theme="dark">
        <c-consumer></c-consumer> 
    </x-foo-context>
    <x-foo-context value="lite">
        <c-other-consumer></c-other-consumer> 
    </x-foo-context>    
</template>
```

Where `<c-consumer>` and `<c-other-consumer>` or any of their descendent will be able to consume the context value by using the `@wire(XFooContext.Consumer)` decorator, e.g.:

```js
import { LightningElement } from 'lwc';
import XFooContext from 'x/fooContext';

export default class MyComponent extends LightningElement {
    @wire(XFooContext.Consumer) foo; // context object provided by <x-foo-context> element
}
```

# Other Alternatives

* The two new APIs can be imported from `@lwc/context` or `@lwc/wire-service` rather than `lwc` directly. This might help to eventually allow others to use contexts with vanilla web components. It will also make it easier for Lightning Platform to restrict who has access to these APIs in platform.
* Make the new APIs a lot more generic, instead of context specific, where you might want to use them to provide contextual wiring per adapter. The fear here is that we will give them a very powerful experimental API.

# Adoption strategy

This is a brand new feature, we just need to document it.

# How we teach this

* The consumer aspect of it is very straight forward considering that we have plenty of documentation about the `@wire` decorator.

# Unresolved questions

* Should the context value be validated?
* Should the context value be type any? or must be an object?
* Should the context value be serializable to support locker?
* ~~How can the context be propagated when the fiber is cut (insertion of root elements inside a template)?~~ Events coordination solves this.
* ~~How can the context be provided to elements in the light DOM of the context element?~~ Events coordination solves this.
* Should the context definition be a default export or a named export?
