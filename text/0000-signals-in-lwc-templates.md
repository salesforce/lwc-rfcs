---
title: Supporting Signals in LWC Templates
status: DRAFT
created_at: 2023-11-15
updated_at: 2023-11-15
champion: Caridy & Dale 
pr: https://github.com/salesforce/lwc-rfcs/pull/
---

# Supporting Signals in LWC Templates

## Summary

This RFC proposes an enhancement to the rendering mechanism in Lightning Web Components (LWC) to support Signals-based objects in LWC Templates, this is inspired by similar mechanisms in libraries such as Preact, vue, etc. The goal is to provide a flexible mechanism to integrate stores or signals into the template in such a way that the rehydration of the UI is still possible even when the user is not relying on the observable membrane provided by LWC. The introduction of Signals in LWC opens the door for more ergonomic methods of observing state changes in the component and allows developers to bring their own data stores if they adhere to a simple protocol involving `.value`, and `.subscribe` properties. This change can also accelerate the work on performance optimizations within LWC Rendering process, especially when rehydrating the template, potentially reducing the need for full component re-rendering through a more efficient subscription mechanism.

## Motivation

The current reactivity model in LWC relies solely in the observable membrane, via proxies, which are tightly coupled with LWC internals. This coupling leads to certain limitations:

1. __Limited External Store Interaction__: LWC components cannot directly interact with external state management systems without going through LWC-specific APIs. This restricts the framework's flexibility and developer freedom to integrate with the broader JavaScript ecosystem.
1. __Complexity and Overhead__: The use of a membrane based on proxies introduces additional complexity and overhead in component design and state management, from read-and-write to read-only objects, it can be challenging for developers to debug and reason about data flow and updates.
1. __Performance Constraints__: The current model relies on components to re-render fully to reflect state changes, which can be inefficient, especially for large and complex component trees.

By adopting a Signals-based reactivity in templates, LWC can overcome these challenges:

1. __Framework Agnosticism__: With Signals, LWC components can interact with any external data store that follows the protocol. This change would make LWC more agnostic and flexible, allowing for easier integration with various state management patterns and libraries.
1. __Simplified State Management__: Signals provide a straightforward way to manage reactive data. By using signals, developers can reduce complexity, making it easier to write and maintain component logic.
1. __Performance Optimizations__: The proposed model opens up possibilities for performance optimizations by allowing selective rehydration of the UI. Components could update the DOM directly in response to state changes without a full re-render, leading to faster updates and reduced resource consumption. This has been a goal of the LWC team for a long time, small incremental improvements has been made, but this effort could accelerate the work in this area.

The motivation for this RFC is to enable LDS, and developers in general, to bring stores and signal-like objects as first class citizens into the LWC ecosystem without introducing significant overhead. To improve the ergonomics of signals in LWC Template, to align LWC with more modern JavaScript practices, simplify component development, and improve performance, making LWC a developers friendly.

## Detailed Design

The detailed design of the new reactivity model revolves around the concept of Signals, which are reactive primitives that allow for fine-grained control over state management and UI updates. Here's how it will look in practice:

### Reactivity with Signals

A Signal is an object that holds a value and allows components to react to changes to that value. It exposes a `.value` property for accessing the current value, and `.subscribe` methods for responding to changes.

```javascript
import { signal } from 'some/signals';

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

```ts
class Store {
    get value(): any { /* return current value */ }
    subscribe((newValue: any) => void): () => void { /* subscribe to changes */ }
}
```

__NOTE:__ _We can work the details of this protocol. It could be more simple, or hopefully similar to what is out there._

### Integration with Components

Components should treat signals as internal reactive state mechanisms. While a signal can technically be passed down to child components, it's recommended to pass only the necessary data, typically the `.value` of the signal, to keep the child components decoupled from the parent's state management strategy.

Here’s how a parent component would pass the current value of a signal to a child component, which expects a number:

```javascript
// ParentComponent.js
import { signal } from 'some/signals';

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

Components can integrate with external stores by using a pattern similar to the current `@wire` mechanism:

```javascript
import { record } from 'app/stores';

export default class ExampleComponent extends LightningElement {
    currentRecord = record;
}
```

```html
<template>
    <p>{currentRecord.value.Name}</p>
</template>
```

__Note:__ if the currentRecord's value is not available yet, it might require some additional logic to handle that case. This is similar to what developers have to do today when using `@wire`.

### LWC Internals

LWC will NOT provide a built-in store implementation. Instead, it relies on a well defined protocol to interpret data in templates. This will allow developers to bring their own stores and integrate them with LWC components. It is very likely that LDS implements its own flavor of signals to work with LWC but the purpose of this RFC is to define the protocol, not the implementation.

## Backward Compatibility

There are two main concerns with backward compatibility:

1. A component receiving an object from a parent component that contains a `value` getter and a `subscribe` method. If the object is considered a POJO by our current test, then LWC would incorrectly avoid wrapping the object with a read-only reactive process. This is a minor concern, but it is something to consider.
2. A template using `x.value` as part of an interpolation where the value of `x` is not a signal, but contains a `value` getter and a `subscribe` method. In this case, the compiled code would attempt to call `subscribe`, if it fails, we can swallow the error and show a warning. In terms of functionality, everything remains the same, the `.value` is used to render the value into the DOM, and if it is reactive it will be updated automatically by triggering the re-rendering, which would call the `.value` getter again. Technically, this is not a non-backward compabitibility issue since we can swallow the error. But it is something to consider.

## Drawbacks

### Increased Complexity for Developers

* __Explicit Reactivity Management__: The shift from `@track`'s implicit reactivity to explicit signals and stores introduces additional complexity. For thos developers that are still using `@track`, they must now manage subscriptions and updates more directly, which could increase the cognitive load, compared to the simplicity of `@track`.
* __Learning Curve__: There's an inherent learning curve as developers familiarize themselves with the new patterns of Signals and stores, particularly around the nuances of updates to `.value` instead of changing deep properties.

### Framework Compatibility Concerns

* __Integration with Existing Libraries__: While the goal is to enable the use of any store that follows the Signal protocol, real-world usage may reveal compatibility issues, necessitating additional wrapper or adapter code. As of today, there are a number of popular Signal-like libraries out there, and we should make sure that we can integrate with them.

## Alternatives

### 1. Adoption of Other Reactivity Models

Other frameworks' reactivity models, such as starbeam, Vue's reactivity system or Svelte's store contracts, could be considered. Each comes with its own set of trade-offs and would require significant adaptation to align with LWC's architecture and principles.

### Enhanced Compiler Optimizations

Enhancements to the LWC compiler could potentially abstract away some of the verbosity and complexity of managing reactive states. This solution would still necessitate a shift in developer practices but could minimize the learning curve and migration effort.

### Framework-Agnostic State Management Libraries

Leveraging existing state management libraries, like Redux or MobX, could be an option. These libraries offer mature ecosystems and patterns but would require integration work to behave reactively within the LWC component lifecycle.

## Adoption Strategy

Most of this section is really about how LDS and others would adopt this new technology. A gradual, phased approach is recommended to transition to the new reactivity model. This allows developers to adopt the changes incrementally and provides the framework maintainers with feedback loops to adjust the strategy as needed. Release comprehensive guides and examples showcasing the new model's benefits and usage.

## Security Considerations

Minor security consideration. Changing a property of a value from a store has no implications for other components with access to the same store, because the change is not going to trigger rehydration of templates, e.g.: `{currentRecord.value.x}` in a template, while someone else with access to the same record can do `currentRecord.value.x++`. In this case, LWC would not pick up the change, and update the UI, as described above, but anyone using the currentRecord in another turn, is subject to the incremented value. This is not such a big risk, but it is something to consider. The responsability shifts to the store developer to protect against others trying to make changes to it. LDS is doing some of that already, but the fact that components are accessing the `.value` directly, makes it more difficult to prevent someone from changing the object that eventually someone else would rely on, unless a new value is emitted. Since LWC is not in control, there is no much we can do to help in this area.

## Open Discussion Topics

* The signal/store protocol. Except for the `.value`, which seems to be a requirement, the rest of the protocol is open for discussion.
* The test to determine when an object is a signal during runtime is tricky, we should discuss that in details.
