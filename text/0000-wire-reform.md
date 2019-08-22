- Start Date: 2019-08-01
- RFC PR: TBD
- Lightning Web Component Issue: (leave this empty)

# Summary

This RFC describes the way to decouple the wire service from LWC entirely, and implement reactive tracking for wired configuration and wired methods.

# Motivations

There is a dual-dependency between `@lwc/engine` and `@lwc/wire-service`, even though neither of those two packages are importing each other, it is the responsibility of the adapter author to connect them via `registerWireService(register)` where `registerWireService()` is provided by `@lwc/wire-service` and `register()` is provided by `@lwc/engine`. This is, by itself, complex and confusing. Additionally, there is another `register()` method from `@lwc/wire-service` that is used by authors to link their wire adapters with their adapter ID (identity). This process also posses a limitation, and unnecessary dependency, making adapters to tied to LWC.

Additionally, there are various situations where the wired field or method is not behaving correctly, the most notable example is when a configuration value that uses a member expression might not trigger the update on the config. (e.g. `wire(foo, { x: '$foo.bar' }) data`, if `foo.bar` changes, config is not updated). This is because the wire decorator is not relying on the reactivity system used by LWC, and instead it relies on getter and setters that are slower, intrusive, complex and do not cover the whole spectrum of mutations that could occur on a component.

Finally, keeping the wire service tied to LWC means that when needed, wire adapters will not be very useful beyond LWC, when in reality they are not tied to the component system.

# Goals

The primary goal of this RFC is to decouple the wire service from LWC and the LWC Wire Decorator implementation.

As a secondary goal, to embrace reactivity for the configuration payload and the wired method in LWC.

A third goal is to support the provision of wire adapters via wire service on any object, whether it is LWC component or not.

# No-goals

* This reform does not change the wire decorator syntax.
* This reform does not change the wire adapter API for lightning platform (we should be able to keep that intact).

# Proposal

This reform is focused on the refactor of the wire decorator code, and the wire-service code. As part of the separation process, there are certain responsibilities that must be well defined:

## Responsibilities of the wire decorator

* To install a prototype descriptor to handle the wired field value from the `vm` of the component.
* To create an instance of an adapter and link it to the host.
* To signal to the adapter instance when the component is connected or disconnected.
* To signal to the adapter instance when the config has changed by providing the new config object.
* To extract the config value from the host object by relying on the compiler's wire metadata.
* To signal to the adapter instance when a context is available by providing the new context value. 

## Responsibilities of the wire service

* To define the wire adapter protocol.
* To provide a referral implementation of the wire adapter protocol.
* To provide an optional abstraction for existing "legacy" wire adapter factories.

## Implementation Details

* `@lwc/engine` does not know about `@lwc/wire-service` and vice versa.
* `@lwc/engine` will install a descriptor on the prototype for every wired field during the decorators registration routine.
* `@wire` decorator will create an instance of the provided `WireAdapter` during the component initialization routine by providing the data callback, and the context callback as arguments. 
* `@wire` decorator will invoke the adapter's `connect` and `disconnect` based on the internal hooks used per component.
* `@wire` decorator will create a mutation tracking phase to track any access executed during the computation of the config before calling the `update` routine to be able to detect mutations on those values and issue another update on the adapter instance.
* `@wire` decorator will implement the logic to provide contextual information when requested by the wire adapter.
* `@lwc/compiler` will provide a config function per `@wire()` declaration to produce a new config object when invoked with the `component` as the first argument. The `@wire` adapter can rely on that config function to produce a new config object at will.
* `@lwc/wire-service` becomes lightning platform specific for most part (`register()` method), since anyone can implement the wire adapter protocol.

### Wire Adapter Protocol

The formalization of the wire adapter protocol is important because that enables the interoperability aspect of this feature. The adapter's code should not be aware of the component system, or even the application framework. It only cares about very specific hints to produce a stream of data. The following describes the proposed protocol:

```ts
interface WireAdapter {
    update(config: Record<string, any>);
    context(uid: string, value: any);
    connect();
    disconnect();
}

interface WireAdapterConstructor {
    new (callback: EmitDataCallback, contextualizer?: RequestContextCallback): WireAdapter;
}

type EmitDataCallback = (value: any) => void;

type RequestContextCallback = (uid: string) => void;
```

_Notes:_

* Not all environments will support or need contextualization (e.g.: preloading LDS data), the `requestContextCallback` is optional. All adapters should be able to work with and without a context provider.

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

Additionally, those callable objects can implement a forking logic based on the type of argument, if there is a desire to avoid the `adapter` property member expression.

# How we teach this

* For adapter consumers, nothing changes.
* For adapter author, the wire protocol is now well defined and does not need registration, which means it is easier to reason about compared to the existing mechanism.
* As for existing adapters based on `@lwc/wire-service`, they can remain the same until after they get refactored and simplified when possible.

# Unresolved questions

* In the current implementation, a wired field is `writable`, which means the component author can alter the value of the field at will. What should we do? a) throw on setter, b) do nothing on setter, c) preserve the current semantics. This is a breaking change if we do a) or b), while the current behavior is weird.
* How context providers can be provisioned? In theory, a context provider is bound to a particular framework/system, while the context consumer is abstracted out in the wire adapter, and specific implementations per framework can provide the piping into the wire adapter protocol via the second argument in the constructor and the `adapter.context(uid, value)` mechanism. Is this sufficient?
