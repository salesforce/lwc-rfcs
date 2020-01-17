---
title: Focus Behavior
status: DRAFTED
created_at: 2020-01-15
updated_at: 2020-01-15
pr:
---

# Focus Behavior

## Summary

When discussing focus behavior, we need to consider the three ways you can
focus on elements: click focus, programmatic focus, and sequential focus. In
general, anything that is click focusable or sequentially focusable is
programmatically focusable, and focus behavior for clicks and sequential
navigation can depend on the user agent and system settings. For example,
Safari will not apply focus on buttons when clicked, but it will apply focus on
buttons when the focus method is invoked [1].

As such, the set of click focusable and sequentially focusable elements is not
something that we can know a priori. What we do know when observing our
supported user agent matrix is that, the set of _potentially_ focusable
elements is the same. This proposal makes the assumption that it is acceptable
for the framework to introduce shadow semantics via various focus management
polyfills as long as we don't prevent the user from reaching a focusable
element.

## Motivation

Generally speaking, it is outside the scope of LWC to normalize different
behaviors across its supported user agent matrix. However, in the case of focus
delegation, the framework does need to make informed decisions to normalize
focus behaviors in order to avoid a broken user experience. This is especially
true in the case of sequential focus navigation.

The primary goal of this proposal will be to minimize differences between its
synthetic Shadow DOM and native Shadow DOM implementations to facilitate the
transition of components to native Shadow DOM.

## Detailed design

For simplification, unless otherwise stated, it can be assumed that all
components are delegating focus.

For simplification, subsequent usage of the term "focusable" will refer to
whether or not an element is programmatically focusable (i.e., whether or not
an element can receive focus when its focus method is invoked or it has the
`autofocus` attribute).

```
focusable = programmatically-focusable = click-focusable ∪ sequentially-focusable
```

### Focusable elements

The set of sequentially focusable elements is currently defined by this
[monster selector]. This has been sufficient for our needs so far but can
probably be improved. We should go through the exercise of checking it against
the results of focusable elements listed on [this test
page](https://boom-bath.glitch.me/tabindex.html).

The set of click focusable and programmatic focusable elements is a superset of
the set of sequentially focusable elements, and should include all elements
with tabindex values of -1.

### Programmatic focus

In the case of programmatic focus on a custom element, if the custom element is
a shadow-including ancestor of the currently focused element, then we don't do
anything and the focused element should remain focused. Otherwise, the
specification states that the first programmatically focusable element should
receive focus. Since there is no DOM API that identifies such an element for
us, we can instead:

1. Invoke the focus method on the first element in our set of focusable
   elements, where the set of focusable elements is the set of sequentially
   focusable elements plus elements with a non-negative tabindex content
   attribute value.
1. Verify that the element received focus via `activeElement`.
1. If the element did not recieve focus then keep invoking the focus method on
   subsequent elements until we verify that focus was transferred.

The verification step is useful as a workaround for implementation bugs around
elements that should be focusable but aren't (e.g., in Safari, `area` with no
`href` is click focusable and sequentially focusable but not programmatically
focusable) and is worth adding since it's simple and cheap.

Note that if a custom element without tabindex has no focusable elements in its
shadow, then invoking the focus method should result in a NOOP.

### Click focus

In the case of click focusability, we can pretty much let the browser do its
thing. The only case we need to handle is the one where the user directly
clicks on the host element itself. When this happens, the specification says
that the first click focusable element should receive focus. Since there is no
DOM API that tells us what is click focusable, the next best thing would be to
fall back to invoking the focus method on the host element.

### Sequential focus

In practice, sequential focus is the class of focusability for which behavior
varies the most between user agents. For example, macOS users are able to
customize the ability to skip certain focusable elements via a combination of
system settings and browser settings.

In this proposal, we choose to ignore such settings in order to implement a
sequential focus polyfill for LWC components which delegate focus. Non-Safari
browsers on macOS also do this so there is precedence. Our existing
implementation already does this ([delegates focus
RFC](0106-delegates-focus.md)) but it is worth reviewing our coverage of
focusable elements since we will be relying on that for all three
classifications of focus.

## Drawbacks

The biggest drawback to this approach is the difference in the focus behavior
when comparing synthetic and native Shadow DOM. For example, in LWC's synthetic
shadow, a Safari user would have to tab through all focusable areas of a
component regardless of whether they configured their system to skip certain
elements. In native shadow, the same user with the same system configuration
would be tabbing through a smaller set of focusable areas.

## Alternatives

No alternatives have been considered. This proposal is simply an attempt to
better align the framework with specifications.

## Adoption strategy

Existing LWC components that have already implemented a focus method can
optionally remove their implementation in favor of the one provided by the
framework. Keeping the existing implementation in place will not break existing
functionality. Our recommendation that users implement their own focus method
until the specs around this area became well-defined, has paid off!

# How we teach this

There is nothing unique to LWC that we need to teach. Developers can reference
the [WHATWG interaction specification] to learn how focus works.

# Unresolved questions

- What should happen when a component that is delegating focus, has a tabindex
  value, and contains no focusable elements, is focused?



[1]: Safari will actually remove focus from a focused button if you click it.

[monster selector]: https://github.com/salesforce/lwc/blob/dec08b50c02cc69141c1833db9406b9d66ce8c1b/packages/%40lwc/synthetic-shadow/src/faux-shadow/focus.ts#L48-L58
[WHATWG interaction specification]: https://html.spec.whatwg.org/multipage/interaction.html
