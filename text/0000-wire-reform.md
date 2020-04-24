---
title: Wire Reform
status: APPROVED
created_at: 2019-08-01
updated_at: 2019-09-10
pr: https://github.com/salesforce/lwc-rfcs/pull/14
---

# Wire reform

This RFC describes the way to decouple the wire service from LWC entirely, and implement reactive tracking for wired configuration and wired methods.

# Motivations

There is a dual-dependency between `@lwc/engine` and `@lwc/wire-service`, even though neither of those two packages are importing each other, it is the responsibility of the adapter author to connect them via `registerWireService(register)` where `registerWireService()` is provided by `@lwc/wire-service` and `register()` is provided by `@lwc/engine`. This is, by itself, complex and confusing. Additionally, there is another `register()` method from `@lwc/wire-service` that is used by authors to link their wire adapters with their adapter ID (identity). This process also poses a limitation, and unnecessary dependency, making adapters to tied to LWC.

Additionally, there are various situations where the wired field or method is not behaving correctly, the most notable example is when a configuration value that uses a member expression might not trigger the update on the config. (e.g. `wire(foo, { x: '$foo.bar' }) data`, if `foo.bar` changes, config is not updated). This is because the wire decorator is not relying on the reactivity system used by LWC, and instead it relies on getter and setters that are slower, intrusive, complex and do not cover the whole spectrum of mutations that could occur on a component.

Finally, keeping the wire service tied to LWC means that when needed, wire adapters will not be very useful beyond LWC, when in reality they are not tied to the component system.

# Goals

The primary goal of this RFC is to decouple the wire service from LWC and the LWC Wire Decorator implementation.

As a secondary goal, to embrace reactivity for the configuration payload and the wired method in LWC.

A third goal is to support the provision of wire adapters via wire service on any object, whether it is LWC component or not.

# No-goals

* This reform does not change the wire decorator syntax.
* This reform does not change the wire adapter API for Lightning Platform (we should be able to keep that intact).

# Proposal

This reform is focused on the refactor of the wire decorator code, and the wire service code. As part of the separation process, there are certain responsibilities that must be well defined:

## Responsibilities of the wire decorator

* To install a prototype descriptor to handle the wired field value from the `vm` of the component.
* To create an instance of an adapter and link it to the host.
* To signal to the adapter instance when the component is connected or disconnected via `connect()` and `disconnect()` APIs.
* To signal to the adapter instance when the config has changed by providing the new config object via `update()` API.
* To extract the config value from the host object by relying on the compiler's wire metadata.
* To signal to the adapter instance when a configured context is available by providing the new context value via `update()` API. 

## Responsibilities of the wire service

* To define the wire adapter protocol.
* To provide a reference implementation of the wire adapter protocol.
* To provide an optional abstraction for existing "legacy" wire adapter factories.

## Implementation Details

* `@lwc/engine` does not know about `@lwc/wire-service` and vice versa.
* `@lwc/engine` will install a descriptor on the prototype for every wired field during the decorators registration routine.
* `@wire` decorator will create an instance of the provided `WireAdapter` during the component initialization routine by providing the data callback as argument.
* `@wire` decorator will invoke the adapter's `connect` and `disconnect` based on the internal hooks used per component.
* `@wire` decorator will create a mutation tracking phase to track any access executed during the computation of the config before calling the `update` routine to be able to detect mutations on those values and issue another update on the adapter instance.
* `@wire` decorator will detect if the adapter is expecting contextual information, and if there is a context provider defined to carry on the hand-shaking protocol between the consumer and the provider.
* `@lwc/compiler` will provide a config function per `@wire()` declaration to produce a new config object when invoked with the `component` as the first argument. The `@wire` adapter can rely on that config function to produce a new config object at will.
* `@lwc/wire-service` becomes Lightning Platform-specific for the most part (`register()` method) since anyone can implement the wire adapter protocol.

_Backwards Compatibility Notes:_

* As today, the descriptor installed on the prototype of the component by the `@wire` decorator was identical to `track` when decorating a field. This means that the component author could change the value of the field, and such change will be tracked. This is no longer the case, and even though the author can still change the value, it will not be reactive, causing no side effects on the UI of the element. This is a non-backwards compatible change, but we believe this is a very low risk for something that was not working as expected.

### Wire Adapter Protocol

The formalization of the wire adapter protocol is important because that enables the interoperability aspect of this feature. The adapter's code should not be aware of the component system, or even the application framework. It only cares about very specific hints to produce a stream of data. The following describes the proposed protocol:

```ts
interface WireAdapter {
    update(config: ConfigValue, context?: ContextValue);
    connect();
    disconnect();
}
interface WireAdapterConstructor {
    new (callback: DataCallback): WireAdapter;
    configSchema?: Record<string, WireAdapterSchemaValue>;
    contextSchema?: Record<string, WireAdapterSchemaValue>;
}
type DataCallback = (value: any) => void;
type ConfigValue = Record<string, any>;
type ContextValue = Record<string, any>;
type WireAdapterSchemaValue = 'optional' | 'required';
```

_Notes:_

* Not all environments will support or need context (e.g.: preloading LDS data), but does supporting it can rely on the static field called `contextSchema` to provide the context value when available.
* Some environments might choose to implement validation rules for `configSchema` and `contextSchema` to guarantee compliance.
* we favored the `DataCallback` over a promised based on `update()` calls because the callback can be invoked sync and async, but most important, because update might never be called by the environment.

### Semantic changes for `@wire` decorator IDL

There exist a few restrictions and ambiguities with the IDL for the config object in `@wire` decorator declarations. This section will describe the changed semantics. Most use cases of `@wire` are unaffected. 

* `$token` can only appear as the value of a top level member property, e.g.: `@wire(foo, { x: '$prop1' })` will continue to be valid, while `@wire(foo, { x: { y: '$prop1' } })` will throw a compiler error, while today it doesn't but the value is never transformed, and remains as a string value. This is a non-backwards compatible change, but we believe this is a very low risk for something that was not working as expected.
* there will be no identity for inline JSON objects when assigned to a property in the config object, e.g.: `@wire(foo, { x: { y: 1 } })` where the value of `x` will be computed every time, instead of cached per instance or per class as today.
* there will be identity preserved when assigning a reference values in the config object, e.g.: `@wire(foo, { x: someValue })` where the value of `x` will be a reference to `someValue` during the class declaration.
* every time that `adapter.update()` is invoked, a new config object will be provided as a first argument, no identity is preserved in this case.
* `adapter.update()` must be invoked initially regardless of the value of the config. e.g.: `@wire(foo, { x: $foo })` will invoke the update even if `this.foo` resolves to `undefined` or was never set.
* on any change to the values used to generate a wire configuration, `adapter.update()` will be called with the new config object, even if results in the same config values.
* the reactive tracking for wire configuration can’t track changes on a wire configuration that depends on component's instance "expandos", it only reacts to changes on class' declared fields.

### Context Provider for Wire Adapters

For LWC, we can introduce a new API that allows the creation of a `Contextualizer`, which is a function that can be used to install a Context Provider on any `EventTarget`. This `Contextualizer` has very specific semantics, and allows LWC engine to do the bridging between `ContextProvider` and `ContextConsumer` (Wire Adapters used via `@wire` decorator when defining `contextSchema` as a static field on the adapter).

When installing a `Contextualizer` in an `EventTarget`, you can provide a set of options that will allow pushing context values to each individual `ContextConsumer` via a very simple API. Lets see an example:

```js
import { createContextProvider } from 'lwc';
import { MyAdapter } from 'my/adapter';
// creating a new contextualizer for `MyAdapter`
const contextualizer = createContextProvider(MyAdapter);

// finding the element to be used as the provider
const elm = document.querySelector('container');
// installing contextualizer on `elm`
contextualizer(elm, {
    consumerConnectedCallback(consumer) {
        consumer.provide({ x: 1 });
    },
});
```

The example above guarantees that any component connected under `elm`'s subtree, and wired to `MyAdapter` will receive a context of `{ x: 1 }` in the adapter via the Adapter's `update()` API.

The following is the specification of the `Contextualizer`:

```ts
interface ContextConsumer {
    provide(newContext: ContextValue): void;
}
interface ContextProviderOptions {
    consumerConnectedCallback: (consumer: ContextConsumer) => void;
    consumerDisconnectedCallback?: (consumer: ContextConsumer) => void;
}
type Contextualizer = (elm: EventTarget, options: ContextProviderOptions) => void;
```

Invariants:

* Only one `Contextualizer` can be created per `WireAdapter`, otherwise throws.
* Only a `WireAdapter` with `contextSchema` can be contextualized, otherwise throws.
* A `Contextualizer` can only be installed once on a given `EventTarget`, otherwise throws.
* Each individual `ContextConsumer` has its own identity, and it can't be forged.
* The identity of a `ContextConsumer` is bound to the provider.

_Notes_:
* `Contextualizer`'s options allow the control of the consumers, and can provide the same data for all consumers, or data based on the identity of each consumer, both cases are valid and supported.
* The consumer disconnect flow is optional.
* `Contextualizer` is a LWC specific mechanism, and it is only relevant for LWC `@wire` decorator.

### Backwards Compatibility

This RFC does introduce minor (or minimal) breaking changes:

* Minor semantic changes for `@wire` decorator IDL as described above.
* Removal of the experimental `LinkContextEvent` constructor in `@lwc/wire-service` in favor of the new `ContextProvider` API.
* Minor semantic change on the identity of the first argument passed into `@wire`, it is now a wire adapter instead of a symbol.
* Removal of LWC's `wire` services via `register()`, which was only needed for `@lwc/wire-services` to plug registered Wire Adapters.
* Minor semantic change in the descriptor installed on the prototype of the component by the `@wire` decorator. The component author could change the value of the field, and it will not be reactive, causing no side effects on the UI of the element.
* Removal of the experimental `decorate` function exposed in `lwc`.
* `@lwc/wire-services`'s `register()` cannot accept a symbol as the adapter id anymore, authors will have to replace that with a function or an arrow function must likely.

### Forward Compatible Changes

* Lifting the restrictions around manual invocation of Wire Adapters.

### Proposed Restrictions for Lightning Platform

* `ContextProvider` should be gold-filed to prevent proliferation of contextual information in the first release.
* Preserve the current restrictions to only allow certain wire adapters in `@wire` decorators for one more release.

## Interop

If you have an adapter, you should be able to use it with any component system, not just LWC. This is an example of how to use this with React:

```js
// shared adapter
import { MyWireAdapter } from 'some-module';

class Foo extends React.Component {
    constructor(props) {
        super(props);
        // The wire adapter instance is bound to the host object via the callback for data
        this.adapter = new MyWireAdapter((data) => {
            this.setState(() => {
                // stream of data from wire adapter to be used to update the component's state
                return data;
            });
        });
        // calling for the initial update of the config since componentDidUpdate() is not
        // invoked for the first time, but all props are ready.
        this.adapter.update({ x: 1, y: this.props.valueOfY });
    }

    componentDidUpdate(prevProps) {
        if (this.props.valueOfY !== prevProps.valueOfY) {
            // recompute the config by extracting `this.props.valueOfY`
            this.adapter.update({ x: 1, y: this.props.valueOfY });
        }
    }

    componentDidMount() {
        this.adapter.connect();
    }

    componentWillUnmount() {
        this.adapter.disconnect();
    }

    render() {
        return (
            <div>{this.state.valueProducedByMyWireAdapter}</div>
        );
    }
}
```

# Adoption strategy

Since there is a need to support callable adapters that behave differently depending on who uses that (wire adapter vs user invoking the function directly), we have added a simple mechanism to support such feature via `adapter` property member expression on the callable. This opens the door to transition existing adapters to the new form. E.g.: APEX adapters are all callable objects.

Additionally, those callable objects can implement forking logic based on the type of argument, if there is a desire to avoid the `adapter` property member expression. E.g.:

```js
export function invokeApex(...args) {
    if (new.target) {
        // invocation via new, return a WireAdapter instance
        const [ dataCallback ] = args;
        // ...
    } else {
        // standard function call, return a Promise of some Apex controller result
        const [ apexControllerParams ] = args;
        // ...
    }
}
```

# How we teach this

* For adapter consumers, nothing changes.
* For adapter author, the wire protocol no longer needs registration, which means it is easier to reason about compared to the existing mechanism.
* The new formalized wire protocol is a lot simpler to reasoning about, and simpler to implement.
* As for existing adapters based on `@lwc/wire-service`, they can remain the same until after they get refactored and simplified when possible.
* Testing components using wire: in the current implementation of the wire protocol, rendering a component that uses an invalid wire adapter (ex: `undefined`) never throws; this implementation will break such tests if the wire adapter mocks are not valid. For platform tests, wire-adapter stubs are provided and they will run fine, but for off-platform tests, a valid adapter needs to be provided.

# Unresolved questions

* ~~In the current implementation, a wired field is `writable`, which means the component author can alter the value of the field at will. What should we do? a) throw on setter, b) do nothing on setter, c) preserve the current semantics. This is a breaking change if we do a) or b), while the current behavior is weird.~~ (resolution is described in Backwards Compatibility section)
* ~~How context providers can be provisioned? In theory, a context provider is bound to a particular framework/system, while the context consumer is abstracted out in the wire adapter, and specific implementations per framework can provide the piping into the wire adapter protocol via the `contextSchema` static field on the wire adapter constructor. Is this sufficient?~~ `createContextProvider` from LWC seems sufficient to implement context.
