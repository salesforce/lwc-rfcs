---
title: Declarative Binding for Non-Standard Event Names
status: DRAFTED
created_at: 2024-02-25
updated_at: 2024-02-25
pr: (leave this empty until the PR is created)
---

# Declarative Binding for Non-Standard Event Names

## Summary

LWC support for declarative event binding has always been limited to event names using lower-case
alphabetic characters, underscores, and numeric characters. Event names containing upper-case
characters (e.g., camelCase, CAPScase, PascalCase, etc) have never been supported in LWC due to
the case-insensitive nature of HTML. In addition, event names containing hyphens were never
supported due to the reluctance to avoid introducing ambiguity into the LWC convention of using
hyphens in attribute names to map to upper-case letters in property names.

This has worked well in practice with developers sticking to LWC conventions and using
`addEventListener()` when required as a workaround, but with the recent introduction of
`lwc:external` which provides a paved path for integrating third party web components, there is
greater value in providing a declarative way to listen for non-standard event names.

This RFC proposes the introduction of a new template directive `lwc:on`, which allows us to support
declarative event binding for third party web components, while sidestepping all the aforementioned
limitations and ambiguities.

## Basic example

```js
customElements.define('third-party', class extends HTMLElement{
    connectedCallback() {
        this.dispatchEvent(new CustomEvent('Foo-Bar-BAZ'));
    }
});
```

These are examples of some of the options that are currently on the table:

```html
<!-- Event as part of directive -->
<third-party lwc:external lwc:on:Foo-Bar-BAZ={handleConnected}></third-party>

<!-- Event as part of attribute name (special treatment for lwc:external) -->
<third-party lwc:external onFoo-Bar-BAZ={handleConnected}></third-party>

<!-- Event and listener as part of attribute value -->
<third-party lwc:external lwc:on="Foo-Bar-BAZ:handleConnected"></third-party>
```

This is an example of a less declarative approach but should also be useful for things like creating
components from metadata:

```js
// foo.js
class Foo extends LightningElement {
    handlers = {
        'kebab-case': (event) => console.log("'kebab-case' handled"),
        camelCase: (event) => console.log("'camelCase' handled"),
        CAPScase: (event) => console.log("'CAPScase' handled"),
        PascalCase: (event) => console.log("'PascalCase' handled"),
    };
}
```

```html
<!-- foo.html -->
<x-bar lwc:external lwc:on={handlers}></x-bar>
```

## Motivation

The primary motivation for this feature is to enable component authors to listen for non-standard
event names in a declarative manner without having to resort to calls to `addEventListener()`. This
should be especially useful when interfacing with non-LWC components that do not follow LWC
conventions, which should become more common with the recent introduction of native web component
support through the use of the `lwc:external` directive.

## Detailed design

The usage of this feature will be restricted to custom elements using `lwc:external`.

The design will be based on the agreed upon API shape, which is still under discussion.

## Drawbacks

A drawback of this feature is that component authors of LWC components may be encouraged to
implement non-standard event names that requires the use of `lwc:on` for declarative consumption.
This is not desirable because it makes static analysis more difficult. Conversely, consumers might
use this directive when they could just as easily have used a standard `on*` attribute in their
template. These scenarios could be avoided, however, by restricting the usage of `lwc:on` to custom
elements that use the `lwc:external` directive.

## Alternatives

### lwc:spread

An alternative design that was considered was to build this feature into the existing `lwc:spread`
directive which currently only supports the setting of property values. In fact, `lwc:spread` can be
used to listen for standard events like `click` or `focus` by assigning event listeners to the
`onclick` and `onfocus` properties. However, adding support for non-standard event names would add
complexity to `lwc:spread`. Introducing a new directive also makes it easier to restrict the usage
of `lwc:on` to `lwc:external` components.

## lwc:on=`${eventName}:${listenerName}`

Encoding both the event name and event handler name as a string using some delimiter would allow us
to work around the case-insensitive nature of HTML attributes. This was simply unpopular during an
initial informal survey.

## Adoption strategy

This feature should only be used for interoperability with third party web components. The main
benefit would be an improvement in static analysis when used over listening to events using
`addEventListener()` and we could document it as such.

# How we teach this

We can teach this by adding to the documentation we already have for third party web components.

# Unresolved questions

Would this be sufficiently declarative if the event name does not appear in the template but the
user does not have to invoke `addEventListener()`?
