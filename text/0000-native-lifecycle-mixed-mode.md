---
title: Native lifecycle mixed mode
status: DRAFTED
created_at: 2024-06-04
updated_at: 2024-06-04
pr: https://github.com/salesforce/lwc-rfcs/pull/89
---

# Native lifecycle mixed mode

## Summary

LWC ships with a polyfill for two [custom element lifecycle hooks](https://developer.mozilla.org/en-US/docs/Web/API/Web_Components/Using_custom_elements#custom_element_lifecycle_callbacks): `connectedCallback` and `disconnectedCallback`. This polyfill was originally designed for IE11, but has persisted due to backwards compatibility concerns.

The polyfill (otherwise known as "synthetic lifecycle") also affects component rendering and `renderedCallback`, since the LWC engine only does an initial render when `connectedCallback` is invoked.

An attempt was made during the LWC v6 release (Salesforce Summer '24) to [upgrade components to native lifecycle based on API versioning](https://github.com/salesforce/lwc/releases/v6.0.0#native-lifecycle). This turned out to be non-viable, so it was rolled back.

The reason it was non-viable is that mixing components running in two modes (synthetic vs native lifecycle) causes observable changes even for components that were not expecting their lifecycle callback timings to change, due to changes in their slot container. (An example of this is spelled out in detail [here](https://github.com/salesforce/lwc/issues/4249).)

To resolve this, this RFC proposes a new "native lifecycle mixed mode," inspired by [mixed shadow DOM mode](https://github.com/salesforce/lwc-rfcs/blob/master/text/0115-mixed-shadow-mode.md). It would work in much the same way, allowing components to gradually opt-in to native custom element lifecycle, while also maintaining consistency within the descendant DOM tree for that component.

## Basic example

Opting in:

```js
export default class extends LightningElement {
  static lifecycleSupportMode = 'native';
}
```

Similar to shadow DOM mixed mode, you can also opt out:

```js
export default class extends LightningElement {
  static lifecycleSupportMode = 'reset';
}
```

Like shadow DOM mixed mode, this `'reset'` is mostly useful if you have a superclass that declares `'native'`, which you would like to override.

## Motivation

The synthetic lifecycle polyfill is technical debt. If LWC were designed from scratch today, it wouldn't ship with its own implementation of `connectedCallback` or `disconnectedCallback` – it would just defer to the native browser behavior.

Unfortunately, due to subtle differences between native and synthetic lifecycle, and due to the Lightning Platform's backwards compatibility guarantees, it is not trivial to swap one implementation out for the other.

At the same time, component authors often complain about bugs in synthetic lifecycle ([and](https://github.com/salesforce/lwc/issues/2609) [there](https://github.com/salesforce/lwc/issues/3361) [are](https://github.com/salesforce/lwc/issues/1452) [many](https://github.com/salesforce/lwc/issues/3823)). The most common complaint is that `disconnectedCallback` does not consistently fire, which can lead to memory leaks or buggy behavior. Unfortunately, fixing these bugs would necessarily create observable behavior that would break some other component that relies on the bug (see [Hyrum's Law](https://www.hyrumslaw.com/)).

To unblock component authors who _do_ want non-buggy behavior for `connectedCallback`/`disconnectedCallback`, we should allow them to opt-in to native lifecycle. This paves the way for a polyfill-free future, while still preserving backwards compatibility and giving a migration path for authors who need time to upgrade.

## Detailed design

### The `lifecycleSupportMode` property

Like `shadowSupportMode`, `lifecycleSupportMode` has two possible values:

- `'native'`
- `'reset'` (default, i.e. synthetic mode)

Any other value throws an error at runtime.

### Polyfill behavior

Similar to [`@lwc/synthetic-shadow`](https://www.npmjs.com/package/@lwc/synthetic-shadow) and [`@lwc/aria-reflection`](https://www.npmjs.com/package/@lwc/aria-reflection), we will isolate the code necessary for synthetic lifecycle into a standalone package: `@lwc/synthetic-lifecycle`. The polyfill can be loaded using simply:

```js
import '@lwc/synthetic-lifecycle';
```

This allows `@lwc/engine-dom` to run in two modes:

1. **Purely native**. This is the case where the polyfill is not loaded. All components run with native lifecycle.
2. **Mixed mode**. This is the case where the polyfill _is_ loaded. This is the only case where `lifecycleSupportMode` actually does anything. Components that specify `static lifecycleSupportMode = 'native'` will opt-out of the polyfill.

This has a few advantages:

1. `@lwc/engine-dom` is "pure" and polyfill-free by default, which gives off-core consumers the best possible LWC experience.
2. It aligns neatly with the existing behavior for LWC's other polyfills.
3. If consumers do not need the polyfill, then they do not need to load the code for the polyfill (which admittedly is small – roughly 385 bytes minified+gzipped).
4. It does not enshrine `lwcRuntimeFlags` as a user-facing API surface, which is the current ad-hoc solution to toggling native vs synthetic lifecycle.

### Transitivity

Like shadow DOM mixed mode, lifecycle mixed mode will need some concept of transitivity. In particular, it will enforce the following rules:

1. Any descendants of a native-lifecycle component are forced into native lifecycle mode. (Same as shadow DOM mixed mode.)
2. Any _slotted content_ of a native-lifecycle component is also forced into native lifecycle mode. (Different from shadow DOM mixed mode.)

The reason for rule #2 is that slots in particular proved to be a source of observable changes for native-lifecycle components. It is simply the case that a slotted component can observe the lifecycle mode of the component it is being slotted into, due to connect/disconnect events triggering downstream callbacks in the tree.

The reason for rule #1 is that, exactly like slotted content, children can observe the mode of their parents.

So rather than trying to emulate synthetic-lifecycle-within-native-lifecycle (which is [a fool's errand](https://github.com/salesforce/lwc/issues/4249)), we should force the entire tree into native lifecycle mode so that `connectedCallback` and `disconnectedCallback` at least consistently fire.

For component authors, this means that they will have to be careful when enabling native lifecycle mode. Similar to native shadow mode, they will have to ensure that their children are all native-shadow ready. But in addition, they must communicate to their consumers that any _slotted_ content that consumers provide is also beholden to this rule.

For components with a longstanding history and wide variety of consumers, this might not be feasible. Component authors will have to decide for themselves whether they are ready to ship their components in native lifecycle or not.

> [!NOTE]
> There is no scenario where LWC will "move" a component from one component's slot to another component's slot. Instead, it will unmount (destroy) and mount (create) a new component in this case. This means that LWC does not have to re-evaluate a component's lifecycle mode after the initial mount. However, if a component is manually removed/inserted, then all bets are off. (Component authors typically do not do this.)

### Testing

For Jest testing, we can follow the same model as the existing [`nativeShadow`](https://github.com/salesforce/lwc-test/blob/master/packages/%40lwc/jest-preset/README.md#nativeshadow) config option:

```json
{
    "globals": {
        "lwc-jest": {
            "nativeLifecycle": true
        }
    }
}
```

In other words, Jest tests will assume synthetic lifecycle as the default, but will allow opting-in to lifecycle mixed mode.

> [!NOTE]
> The decision of which mode is used by default can be debated later. After all, Jest is not production – it is just for informational purposes. But for now, it makes sense to align with the existing `nativeShadow` option.

## Drawbacks

The main drawbacks of this proposal are:

- It creates two new confusing modes for LWC to run in (with a confusing name as well – although "synthetic lifecycle" seems to have stuck).
- It adds complexity without actually removing global polyfills from LWC (which will still be required as long as any component on the page uses synthetic lifecycle).
- It will add some complexity to the code to extract out the polyfill to a separate package, and to create a handshake between that package and `@lwc/engine-dom`.
- Component authors may not migrate as fast as we'd like them to, since it is opt-in and provides abstract benefits compared to a very real risk of breakage.

## Alternatives

Obviously API versioning was considered, but rolled back for reasons spelled out above.

Another alternative is to just _never_ ship native lifecycle, but due to the widespread bugs in the polyfill and volume of user complaints about it, this alternative does not seem reasonable.

## Adoption strategy

We can certainly write [a codemod](https://github.com/salesforce/lwc-codemod) for this – it is trivial to add a static property. Unfortunately, the codemod cannot detect or fix any bugs caused by the migration from synthetic to native lifecycle.

On the other hand, we do already have runtime warnings that detect when synthetic lifecycle is executing `connectedCallback` on a disconnected DOM tree, and we can use this warning message as an opportunity to point component authors to native lifecycle.

The adoption strategy for native lifecycle can also be folded into the strategy for native shadow DOM, since the shape of the solution largely rhymes.

# How we teach this

The names "synthetic" and "native" here were deliberately chosen to resemble the [existing documentation around native shadow DOM](https://developer.salesforce.com/blogs/2024/01/get-your-lwc-components-ready-native-shadow-dom). Teaching one should give us a perfect opportunity to teach about the other.

# Unresolved questions

None at this time.
