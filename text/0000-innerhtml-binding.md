- Start Date: 2019-10-03
- RFC PR: (leave this empty)
- Lightning Web Component Issue: (leave this empty)

# Summary

Applications dealing with rich content, like commerce apps, need to render 
this content as raw HTML, typically coming from a database or a CMS. Today, 
the data binding capability always escapes the HTML for security reasons,
which makes the rich content data not rendered as expected.

# Basic example

The following template render some HTML content as innerHTML  

```js
function createMarkup() { return {__html: 'First &middot; Second'}; };

<template>
    <div lwc:innerHTML={createMarkup}/>
</template>
```

# Motivation

It is common in customer facing apps to render rich content coming from a
CMS or equivalent.  

As of today, and according to the LWC documentation, this capability is 
supported by acting directly with the DOM within `renderedCallback()`.
The templating mechanism does not provide any binding capability that renders
some content as raw HTNL.  

This breaks with SSR, as `renderedCallback()` and DOM manipulation won't be
allowed when a component is rendered on the server. Moreover, even if it was
possible, it would make the client side hydration more complex as the content
is not generated as part of the VDOM.  

Having to act on the DOM is also not user friendly compared to a binding
facility.  


# Detailed design

It seems that there is a consensus among the most popular libraries to expose the
innerHTML property of an element, although the exposed name for this property varies:

React: [dangerouslySetInnerHTML attribute](https://reactjs.org/docs/dom-elements.html#dangerouslysetinnerhtml).  
Angular: [https://angular.io/guide/template-syntax#property-binding-or-interpolation]().  
VueJS: [https://vuejs.org/v2/guide/syntax.html#Raw-HTML]().  

The Angular choice of `innerHTML` feels the most natural, while the React `dangerouslyInnerHTML`
has probably be named this way to not conflict with any potential component property.
To avoid any name collision, LWC can prefix it with the technical "lwc:" prefix.

React makes it even more secure by forcing the HTML to be added to a temporary 
object with a `__html` property. This avoids undesired assignment from a method
that does not sanitize the content:

```
<template>
    <div lwc:innerHTML={getUserName()}/>
</template>
```

It is up to the component developer to make sure that the injected HTML is properly
sanitized ([https://en.wikipedia.org/wiki/HTML_sanitization]()). Several client side
libraries, easily consumable by an LWC application, are available to do the job.

The compiler generates an error if an element has both an `innerHTML` property and
some content defined, like bellow:  

```
<template>
    <div lwc:innerHTML={myHTML()}>
      Adding content when innerMTML is defined leads to a compilation error
    </div>
</template>
```


# Drawbacks

- It might open security breaches if not used appropriately. The proposed API forces the developer to know what he/she is doing and avoids mistaken assignment.  
- Locker might prevent this assignment. To be verified.


# Alternatives

Different syntaxes for the `innerHTML` attribute are possible, but they finally lead to
the same results. It is then more a matter of preference and consistency.  

Another solution would define a new binding syntax, like for example `${{myhtml()}}` or
`${htnl:myhtml()}}`. But such a syntax could be used everywhere a binding is possible,
including attribute values. This could lead to undefined behaviors why not providing any
value. The proposed syntax makes sure that the `innerHTML` binding always applies to the
content of an element.


# Adoption strategy

Because the attribute uses the reserved `lwc:` prefix, there is not backward compatibility
issue, and there is no potential name conflicts.


# How we teach this

inner HTML or raw HTML binding are the industry adopted terms for such a feature.
