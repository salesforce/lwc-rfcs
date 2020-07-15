---
title: Secure Script Container
status: DRAFTED
created_at: 2020-07-15
pr: (leave this empty until the PR is created)
champion: Caridy Patiño, Ted Conn
---

# Secure Script Container for LWC Apps

## Summary

We need a way for application developers to integrate common third-party libraries into their LWC applications. These libraries, such as Google Analytics, are severely hindered by the shadow DOM which prevents global interaction with any element on the page.

By providing a secure container for scripts to run, see, and interact with all the shadow DOM trees as if it were the light DOM, application developers can continue to run their existing integrations until these libraries support shadow DOM semantics natively.

## Basic example

```html
<secure-script src="//cdn.optimizely.com/js/12345678.js"></secure-script>
```

`secure-script` is a custom element which acts the same as the `script` tag, but any code which is evaluated inside this container is able to traverse the entire shadow tree using ligh DOM semantics. So if the above script runs `document.querySelectorAll('button')`, it will return all button elements regardless if they are in the shadow or not.

```html
<secure-script>
    (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
    (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
    m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
    })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

    ga('create', 'UA-XXXXX-Y', 'auto');
    ga('send', 'pageview');
</secure-script>
```

Inline scripting would also be supported, such as the above example which uses script injection to fetch its required resources.

## Motivation

Web component applications need the ability to integrate with third party libraries. Because
web components rely on the shadow DOM for encapsulation, this development paradigm does not work
with libraries who are expecting to interact with the application globally. Before there was
component encapsulation via shadow DOM it was possible for library authors to:

- Listen for events and know who fired them
- Define global styling rules that would get applied on every element
- Query the entire document for any element on the page

These libraries are typically added as global scripts to the document root rather than deeper 
integrations into the components themselves. When every component's root element is a shadow
root, there is no light DOM for them to work with, and their functionality is severely crippled.

Some examples of these libraries are:

- **Analytics tools**: Google Tag Manager / Google Analytics, Pendo
- **Personalization platforms**: Optimizely, Evergage, Google Optimize
- **Instrumentation tools**: New Relic, Sentry.io

In addition, developers are finding that this model is not compatible with their **design systems** and **testing frameworks**.

The goal of this proposal is to provide a way for application developers to continue to use their existing integrations until library authors can support web component based applications.

## Design

`secure-script` will solve for the synthetic shadow and subsequently the native shadow.

### Synthetic Shadow
Many LWC applications apply the [synthetic-shadow polyfill](https://github.com/salesforce/lwc/tree/master/packages/%40lwc/synthetic-shadow/). This polyfill emulates the native shadow behavior while still allowing global styles to cascade into the shadow trees. Synthetic shadow currently [patches](https://github.com/salesforce/lwc/tree/master/packages/%40lwc/synthetic-shadow/src/polyfills/document-shadow) the `document.querySelector` and `document.querySelectorAll` APIs which is what prevents developers from penetrating the shadow DOM boundaries either in their LWC components or any other scripts added to the document. We only want to solve for the latter scenario.

Ok, so what if we store a reference to the original, unpatched APIs before they get patched by our polyfill? And we let *only* our "secure scripts" access those APIs? That's the premise of this approach for synthetic shadow:

* Create a secure JS sandbox which doesn’t allow its object graph to be extended using [`Object.preventExtensions`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/preventExtensions)
  * The sandbox is created using a same-domain iframe provided by Locker v.Next
  * This has to be done before the synthetic shadow is imported

* Run or include the third party library script inside this sandbox
  * This allows the code to run in a non-patched environment, while the rest of the document is running synthetic shadow
* Package the above functionality in a web component which semantically matches the `<script>` tag.


### Native Shadow

For applications which are using the native shadow, the underlying implementation will be different. There are two approaches we can choose from:

1. Virtualize every shadow DOM tree in the outer window into a single DOM structure which this container would see as the default DOM.
2. Implement a `document.querySelector` API which traverses the shadow DOM naturally.

### Globals

Some scripts define globals which are meant to be used by the rest of the application. Let's take this example of Google Analytics:

```html
<!-- Google Analytics -->
<script>
    window.ga=window.ga||function(){(ga.q=ga.q||[]).push(arguments)};ga.l=+new Date;
    ga('create', 'UA-XXXXX-Y', 'auto');
    ga('send', 'pageview');
</script>
<script async src='https://www.google-analytics.com/analytics.js'></script>
<!-- End Google Analytics -->
```

If we included the above snippet inside our secure-script container, then `ga` will only be available to the *inner* `window`, i.e. the secure container's window. So `ga` will not be usable by anyone else in the entire application. We can however, allow for globals to be created inside our secure container, and redefined in any additional application sandboxes created by Locker (or the global outer window).

A syntax like the following could be provided:

```html
<secure-script extra-globals="ga"></secure-script>
```

### Known Limitations

* Currently there is an outstanding issue with synthetic shadow in that the Event object patching happens lazily, so we are unable to prevent its patching at the time of creating the secure container, which means that events are still retargeted. [See LWC PR #1569](https://github.com/salesforce/lwc/pull/1569).


### Testing

We will want to gather the top 20 common libraries and implement their behavior inside of `secure-script` to ensure that we are prooperly accounting for all these libraries' use cases. This will be our integration test suite. 

### Prototype Code

```js
import createSecureEnvironment from '@caridy/sjs/lib/browser-realm';

// creating the new sandbox with all global endowments
const evaluate = createSecureEnvironment(
    undefined,
    window,
);

evaluate(`
    // This initialization will prevent any of these APIs to be polyfilled
    // on the blue realm that can affect this sandbox.
    const pE = Object.preventExtensions;
    pE(HTMLElement.prototype);
    pE(Element.prototype);
    pE(Node.prototype);
    pE(Event.prototype);
    pE(Document.prototype);
    pE(EventTarget.prototype);
    pE(MutationObserver.prototype);
    pE(HTMLCollection.prototype);
    pE(NodeList.prototype);
    pE(ShadowRoot.prototype);
    pE(HTMLSlotElement.prototype);
    pE(Text.prototype);
`);

class SecureScript extends HTMLElement {
    connectedCallback() {
        const src = this.textContent;
        if (src) {
            evaluate(src);
        }
    }
}

customElements.define('secure-script', SecureScript);
```

## Drawbacks

- We are potentially prolonging the life of our synthetic shadow dom polyfill, instead of encouraging community library authors to adopt shadow DOM semantics.
- The element may end up being anywhere from 5kb - 20kb.

## Alternatives

- **Allow shadow DOM to be toggled at a more granular level.** There is a forthcoming RFC for allowing component authors to decide if they want to opt in to shadow DOM or not. This would mean we can keep top-level elements and layout components in the light DOM, but base components or managed components can choose to safeguard their internals. Until we allow this, `secure-script` serves as a mitigation for application developers looking to add third party libraries to their apps.
- Do not apply the shadow DOM polyfill at all: this would allow component authors to violate shadow DOM encapsulation which is not an option.
- If we don't do this, then companies / websites building on our platform will not be able to integrate with common third party libraries.

## Adoption strategy

TBD

## How we teach this

TBD

## Unresolved questions

- Do we want to support all the existing `script` tag attributes like `type`, `async` and `defer`?
