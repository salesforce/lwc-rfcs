---
title: Reactivity Model with Signals in LWC
status: DRAFT
created_at: 2023-11-08
updated_at: 2023-11-08
champion: Caridy & Dale 
pr: https://github.com/salesforce/lwc-rfcs/pull/
---

# Reactivity Model with Signals in LWC

## Summary

This RFC proposes an enhancement to the reactivity model in Lightning Web Components (LWC) by introducing a Signals-based approach, inspired by similar mechanisms in libraries such as Preact. The goal is to provide a more flexible and optimized method for managing and tracking component state changes, facilitating better interaction with external data stores. The introduction of Signals aims to deprecate the current `@wire` and `@track` decorators, allowing developers to bring their own data stores and adhere to a simple protocol involving `.value`, `.subscribe`, and `.unsubscribe` properties. This change is designed to not only open up LWC to a broader development ecosystem but also to enable significant performance optimizations, especially when rehydrating UI components, potentially reducing the need for full component re-rendering through a more efficient subscription mechanism.

## Motivation

The current reactivity model in LWC relies on `@wire` and `@track` decorators, which are tightly coupled with LWC internals. This coupling leads to certain limitations:

1. __Limited External Store Interaction__: LWC components cannot directly interact with external state management systems without going through LWC-specific APIs. This restricts the framework's flexibility and developer freedom to integrate with the broader JavaScript ecosystem.
1. __Complexity and Overhead__: The use of decorators like `@wire` and `@track` introduces additional complexity and overhead in component design and state management, making it challenging for developers to reason about data flow and updates.
1. __Performance Constraints__: The current model requires components to re-render fully to reflect state changes, which can be inefficient, especially for large and complex component trees.

By adopting a Signals-based reactivity model, LWC can overcome these challenges:

1. __Framework Agnosticism__: With Signals, LWC components can interact with any external data store that follows the protocol. This change would make LWC more agnostic and flexible, allowing for easier integration with various state management patterns and libraries.
1. __Simplified State Management__: Signals provide a straightforward way to manage reactive data. By deprecating the @wire and @track decorators, we can reduce complexity, making it easier for developers to write and maintain component logic.
1. __Performance Optimizations__: The proposed model opens up possibilities for performance optimizations by allowing selective rehydration of the UI. Components could update the DOM directly in response to state changes without a full re-render, leading to faster updates and reduced resource consumption.

The motivation for this RFC is to align LWC with modern JavaScript practices, simplify component development, and improve performance, making LWC a more attractive option for developers and increasing its competitiveness in the frontend framework market.

## Detailed Design

The detailed design of the new reactivity model revolves around the concept of Signals, which are reactive primitives that allow for fine-grained control over state management and UI updates. Here's how it will look in practice:

### Reactivity with Signals

A Signal is an object that holds a value and allows components to react to changes to that value. It exposes a `.value` property for accessing the current value, and `.subscribe` and `.unsubscribe` methods for responding to changes.

```javascript
import { signal } from 'signals';

export default class ExampleComponent extends LightningElement {
    count = signal(0);

    increment() {
        this.count.value++;
    }
}
```

In the template, we can bind directly to the `.value` property:

```html
<template>
    <button onclick={increment}>Increment</button>
    <p>{count.value}</p>
</template>
```

### Compiler Enhancements

The LWC compiler will be updated to recognize when a Signal's `.value` is used within a template. Instead of accessing the property directly at runtime, it will generate code to subscribe to the Signal's changes during component's template initialization and unsubscribe upon component's template disposal.

### Store Protocol

A store must implement the following protocol to be compatible with the new reactivity model:

```javascript
class Store {
    get value() { /* return current value */ }
    subscribe(callback) { /* subscribe to changes */ }
    unsubscribe(callback) { /* unsubscribe from changes */ }
}
```

__NOTE:__ _We can work the details of this protocol. It could be more simple, or hopefully similar to what is out there._

### Integration with Components

Components should treat signals as internal reactive state mechanisms. While a signal can technically be passed down to child components, it's recommended to pass only the necessary data, typically the `.value` of the signal, to keep the child components decoupled from the parent's state management strategy.

Here’s how a parent component would pass the current value of a signal to a child component, which expects a number:

```javascript
// ParentComponent.js
import { signal } from 'signals';

export default class ParentComponent extends LightningElement {
    count = signal(0);
}
```

```html
<!-- ParentComponent's template -->
<template>
    <!-- Pass the value of the signal, not the signal itself -->
    <child-component count={count.value}></child-component>
</template>
```

The child component simply receives a number as a property and uses it as such, unaware of the signal mechanism in the parent:

```javascript
export default class ChildComponent extends LightningElement {
    @api count;
}
```

This ensures that child components remain agnostic of the parent's state management implementation, promoting better component encapsulation and reuse.

In terms of optimizations, the compiler can detect when a signal is passed down to a child component because of the `.value` in the template interpolation, and generate code to subscribe to the signal's changes. This would allow the parent component to update the child's public property `count` without a full re-render, leading to better performance. On the receiving end, because the child component's `count` property is not a signal, but a value, it will react to changes as usual, without any special handling.

### Object Reactivity

Objects can be wrapped in Signals to make them reactive. However, changes to nested properties must be handled by replacing the entire object:

```javascript
import { signal } from 'signals';

export default class ExampleComponent extends LightningElement {
    position = signal({ x: 0, y: 0 });

    move(x, y) {
        this.position.value = { ...this.position.value, x, y };
    }
}
```

### Connecting to External Data Stores

Components can integrate with external stores by using a pattern similar to the current `@wire`` mechanism, but with greater flexibility:

```javascript
import { RecordStore } from 'c/myStore';

export default class ExampleComponent extends LightningElement {
    record = new RecordStore("001XXXXXXXXXXXXXXX");
}
```

```html
<template>
    <p>{record.value.Name}</p>
</template>
```

__Note:__ _since the record might not be ready, it might require some additional logic to handle the case when the record is not available. But this is the same they do today with `@wire`._

### Reactive API Properties

The example above shows how to integrate with external stores using a static value. But what if the value is actually a property of the component that might change over time? To facilitate interaction with external stores, LWC will provide access to reactive API properties:

```javascript
export default class ExampleComponent extends LightningElement {
    @api recordId;
    record = new RecordStore(this.$api.recordId);
}
```

In this design, `this.$api` is a reactive wrapper around the component's API properties, enabling stores to react to changes in properties like recordId. This reensambles the internal mechanism used today in LWC via `vm.cmpProps` that is not exposed to developers.

This seems like a good compromise, where the developer only needs to know about the existant of `this.$api` as a way to access those signals, but they don't need to know about the internal implementation details. This is also backward compatible since the `$api` would be a new property of the base class that can be override by the subclass.

The alternative would be to ask developers to implement getter and setters for those properties in order to capture the incoming values, and place them into a private signal instance. This is not super bad, but it is more verbose and it requires more knowledge about the internal implementation details of the framework.

## Transition from `@track`

### Background

The `@track` decorator in LWC currently allows components to automatically track changes to the properties of an object, enabling reactive UI updates on the component tracking the object, but also on any child component that get access to the tracked object or a piece of it as read-only. While `@track` simplifies reactivity for objects, it creates a reactive proxy which can add complexity and overhead.

### New Approach

With the proposed Signals-based reactivity model, explicit reactivity replaces the implicit reactivity provided by `@track`. This shift encourages developers to consider the state as a series of discrete, observable changes rather than a mutable object graph.

__Advantages:__

* __Clarity and Predictability__: Developers will have a clearer mental model of their application's reactivity, as they must now explicitly define how and when the state changes.
* __Performance__: By removing the reactive proxy layer, applications can see performance improvements, particularly in the handling of complex state structures.

### Migration Path

To transition from `@track`, developers can:

* Identify which properties need to be reactive and create `Signals` for them.
* Refactor templates to use the `.value` property of Signals where tracked properties were previously used. This guarantees that the child component will receive the value, not the Signal itself.
* Update component logic to explicitly set the new value of a Signal when the state changes. This can be done by using the `.value` property of the Signal, or by replacing the entire Signal with a new one, which is slightly different from the current `@track` behavior, where you can change a property of the object and the component will react to it.

## Transition from `@wire`

### Background

The `@wire` decorator allows for a declarative approach to data fetching and reactive property updates based on changing inputs. It is closely tied to the Salesforce data layer and service layer.

### New Approach

Under the new design, external stores or Signals would take over the responsibility of managing reactive data fetching and updates. These stores would follow a standard protocol, making them framework-agnostic.

__Advantages__:

* __Flexibility__: Developers can use any data store that adheres to the protocol, not just Salesforce-specific services.
* __Consistency__: A consistent pattern for managing both local and external data simplifies the mental model for developers.

### Migration Path

#### Replacing a `@wire` property

We can go from this:

```javascript
import { RecordWireAdapter } from "c/wireAdapters";

export default class ExampleComponent extends LightningElement {
    @api recordId;
    @wire(RecordWireAdapter, { recordId: "$recordId" }) record;
}
```

To this:

```javascript
import { RecordStore } from "c/stores";

export default class ExampleComponent extends LightningElement {
    @api recordId;
    record = new RecordStore(this.$api.recordId);
}
```

This seems simple enough. No major differences, the `$` continues to be a thing in the new model so the store can access the value of the public property and react to it by fetching new data, it is just a different placement for the `$`. In terms of the usage of the record iself, it is almost the same, except that now we need to use `.value` to access the actual record in the template, e.g.:

```html
<template>
    <p>{record.value.Name}</p>
</template>
```

#### Replacing a `@wire` method

We can go from this:

```javascript
import { RecordWireAdapter } from "c/wireAdapters";

export default class ExampleComponent extends LightningElement {
    @api recordId;
    @wire(RecordWireAdapter, { recordId: "$recordId" }) updateRecord(record) {
      // do something with the record
    };
}
```

To this:

```javascript
import { RecordStore } from "c/stores";

export default class ExampleComponent extends LightningElement {
    @api recordId;
    record = new RecordStore(this.$api.recordId);

    updateRecord(record) {
      // do something with the record
    }

    connectedCallback() {
        this.record.subscribe(record => this.updateRecord(record));
    }

    disconnectedCallback() {
        this.record.unsubscribe(); // TODO: this is wrong
    }
}
```

It is important to highlight that this is the less common method of using a wire adapter. With the new approach, it becomes a little bit more cumbersome, but it is still possible to do it, and it offers a lot more flexibility since you can now use any store that follows the protocol, not just Salesforce-specific services.

## Backward Compatibility

There are two main concerns with backward compatibility:

1. A component receiving an object from a parent component that contains a `value` getter and a `subscribe` method. If the object is considered a POJO by our current test, then LWC would incorrectly avoid wrapping the object with a read-only reactive process. This is a minor concern, but it is something to consider.
2. A template using `x.value` as part of an interpolation where the value of `x` is not a signal, but contains a `value` getter and a `subscribe` method. In this case, the compiled code would attempt to call `subscribe`, if it fails, we can swallow the error and show a warning. In terms of functionality, everything remains the same, the `.value` is used to render the value into the DOM, and if it is reactive it will be updated automatically by triggering the re-rendering, which would call the `.value` getter again. Technically, this is not a non-backward compabitibility issue since we can swallow the error. But it is something to consider.

## Drawbacks

### Increased Complexity for Developers

* __Explicit Reactivity Management__: The shift from `@track`'s implicit reactivity to explicit signals and stores introduces additional complexity. For thos developers that are still using `@track`, they must now manage subscriptions and updates more directly, which could increase the cognitive load, compared to the simplicity of `@track`.
* __Learning Curve__: There's an inherent learning curve as developers familiarize themselves with the new patterns of Signals and stores, particularly around the nuances of subscription management and the .value convention.

### Migration Effort

* __Codebase Refactoring__: Transitioning existing code that relies on `@track` and `@wire` to the new model will require careful refactoring. This could be a significant effort for large and complex applications.
* __Backward Compatibility__: Ensuring backward compatibility during the transition period may necessitate maintaining dual systems, complicating the framework's implementation and usage.
* __Verbose Code__: Some scenarios, like converting `@wire` methods to store subscriptions, result in more verbose code, which might detract from the developer experience.

### Tooling and Documentation Updates

* __Development Tooling__: All tooling, including IDEs, linters, and debuggers, will need updates to accommodate and optimize for Signals and stores.
* __Educational Materials__: Comprehensive updates to documentation, tutorials, and example code will be essential to support developers in adopting the new model.

### Potential for Misuse

* __Overuse of Signals__: There's a risk that developers might overuse signals for state management, leading to unnecessarily reactive code, which can be less performant and harder to maintain. This is specially relevant because of the runtime detection of `.value` to maintain backward compatibility with existing components. Any misused on the signals can affect the runtime performance of the component and therefore the overall performance of the application.
* __Subscription Management__: Incorrect handling of subscriptions could introduce memory leaks or lead to unexpected behaviors if not managed correctly. This is specially important because the store implementation doesn't have access anymore to the component's lifecycle hooks, instead it would just stop receiving updates of the stores piped thru `this.$api`, meaning that we are at the mercy of the garbage collector.

### Framework Compatibility Concerns

* __Integration with Existing Libraries__: While the goal is to enable the use of any store that follows the Signal protocol, real-world usage may reveal compatibility issues, necessitating additional wrapper or adapter code. As of today, there are a number of popular Signal-like libraries out there, and we should make sure that we can integrate with them.
* __Interop with Non-Signal Components__: Components designed without signals in mind may require additional logic to work within the new reactivity model, potentially leading to a fragmented ecosystem during the transition period.

### Performance Implications

* __Granularity of Reactivity__: The fine-grained nature of Signals may introduce performance concerns if developers do not carefully consider the reactivity scope and frequency of updates.
* __Proxy Overhead Elimination__: While removing the proxy overhead of `@track` can lead to performance gains, it's essential to validate these improvements across a wide range of use cases to ensure that they hold true in practice.

## Alternatives

### 1. Continued Use of `@track` and `@wire`

The simplest alternative is to maintain the current reactivity model with @track and @wire. However, this approach does not address the limitations of the current system, such as the framework's tight coupling with Salesforce-specific services and the complexity introduced by reactive proxies. This hinders the ability for us to optimize the rendering and rehydration as well as the posibility to have more than one version of LWC in the same page.

### 2. Adoption of Other Reactivity Models

Other frameworks' reactivity models, such as Vue's reactivity system or Svelte's store contracts, could be considered. Each comes with its own set of trade-offs and would require significant adaptation to align with LWC's architecture and principles.

### Enhanced Compiler Optimizations

Enhancements to the LWC compiler could potentially abstract away some of the verbosity and complexity of managing reactive states. This solution would still necessitate a shift in developer practices but could minimize the learning curve and migration effort.

### Framework-Agnostic State Management Libraries

Leveraging existing state management libraries, like Redux or MobX, could be an option. These libraries offer mature ecosystems and patterns but would require integration work to behave reactively within the LWC component lifecycle.

## Adoption Strategy

### Phased Adoption

A gradual, phased approach is recommended to transition to the new reactivity model. This allows developers to adopt the changes incrementally and provides the framework maintainers with feedback loops to adjust the strategy as needed.

1. __Documentation and Examples__: Release comprehensive guides and examples showcasing the new model's benefits and usage.
1. __Tooling Support__: Update development tools to assist with the migration, such as linters for detecting `@track` usage or codemods for refactoring to Signals.
1. __Deprecation Warnings__: Introduce deprecation warnings for `@track` and `@wire` in development mode to inform developers of the upcoming changes.

Keep in mind that existing technology can still work side by side with the new reactivity model. For example, `@wire` and `@track` can still be used in the same component with Signals and stores, allowing developers to migrate incrementally.

### Support and Migration

1. __Migration Tools and Assistance__: Provide tools and resources to help developers migrate existing components to the new model. Potentially leveraging generative AI for that.
1. __Fallback Mechanisms__: Maintain legacy `@track` and `@wire` functionality to ensure that applications continue to function during the transition.
1. __Long-term Support__: Offer long-term support for the previous reactivity model to give enterprise users sufficient time to plan and execute their migration strategies. The `@wire` and `@track` will certainly get less performant, so developers are encouraged to migrate to the new model.

### Monitoring and Feedback
1. Feedback Channels: Establish clear channels for feedback from developers to monitor the adoption process's success and identify any areas needing additional support or clarification.
1. Performance Metrics: Track performance metrics before and after adopting the new model to validate its benefits and guide further optimizations.

## Unresolved Questions

[List any questions or concerns that you have not yet addressed.]

## Security Considerations

Minor security considerations:

1. Changing a property of a value from a store has no implications for other components with access to the same store, but we loose the read-only protection that we get with `@track`. This is not a big deal, but it is something to consider. The responsability shifts to the store developer to protect against others trying to make changes to it. LDS is doing some of that already by producing a new object whenever someone subscribe, but the fact that components are accessing the `.value` directly, makes it more difficult to prevent someone from changing the object that eventually someone else would rely on, unless a new value is emitted.
2. Exposing more internal implementation details to developers, like the `$api` property, could lead to misuse and potential security issues as described in 1 if the value is an object. An alternative here is to continue doing the readonly proxy for non-primitive values, but that would be a step back in terms of performance.

## Open Discussion Topics

* The signal/store protocol. Except for the `.value`, which seems to be a requirement, the rest of the protocol is open for discussion.
* The `this.$api` as a way to access internal signal objects created by LWC, that seems to be a good compromise, but we can discuss alternatives.
