---
title: Virtual Script Container
status: DRAFTED
created_at: 2020-07-15
pr: https://github.com/salesforce/lwc-rfcs/pull/39
champion: Caridy Patiño, Ted Conn
---

# Virtual Script Container for LWC Apps

## Summary

We need a way for application developers to integrate common third-party libraries into their Web Component based apps (specifically, LWC applications). These libraries, such as Google Analytics, are severely hindered by the Shadow DOM which prevents global interaction with any element on the page.

By providing a virtual container for scripts to run, see, and interact with all the Shadow DOM trees as if it were the light DOM, application developers can continue to run their existing integrations until these libraries support Shadow DOM semantics natively.

## Basic example

```html
<virtual-script src="//cdn.optimizely.com/js/12345678.js"></virtual-script>
```

`virtual-script` is a custom element which acts the same as the `script` tag, but any code which is evaluated inside this container is able to traverse the entire Shadow  DOM tree using light DOM semantics. So if the above script runs `document.querySelectorAll('button')`, it will return all `<button>` elements regardless if they are encapsulated inside a `ShadowRoot` or not.

```html
<virtual-script>
    (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
    (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
    m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
    })(window,document,'script','https://www.google-analytics.com/analytics.js','ga');

    ga('create', 'UA-XXXXX-Y', 'auto');
    ga('send', 'pageview');
</virtual-script>
```

Inline scripting would also be supported, such as the above example which uses script injection to fetch its required resources.

## Motivation

Web component based applications need the ability to integrate with third party libraries. Because
web components rely on the Shadow DOM for encapsulation, this development paradigm does not work
with libraries who are expecting to interact with the application globally. Before there was
component encapsulation via Shadow DOM it was possible for library authors to:

- Listen for events and know who fired them
- Define global styling rules that would get applied on every element
- Query the entire document for any element on the page

These libraries are typically added as global scripts to the document root rather than deeper 
integrations into the components themselves. When every component in the application has a ShadowRoot attached to them, these libraries will not work as expected.

Some examples of these libraries are:

- **Analytics tools**: Google Tag Manager / Google Analytics, Pendo
- **Personalization platforms**: Optimizely, Evergage, Google Optimize
- **Instrumentation tools**: New Relic, Sentry.io

The goal of this proposal is to provide a way for application developers to continue to use their existing integrations until library authors support Shadow DOM integration.

## Design

A new element called `<virtual-script>` encapsulates the various mechanisms to evaluate the scripts in a virtual environment, which solves the problem when using synthetic shadow and subsequently, when using the native Shadow DOM. We can implement this solution in two phases:

### Phase I: Synthetic Shadow

Many LWC applications use the [synthetic-shadow polyfill](https://github.com/salesforce/lwc/tree/master/packages/%40lwc/synthetic-shadow/). This polyfill emulates the native Shadow DOM behavior while still allowing global styles to cascade into the shadow trees. Synthetic Shadow patches many of the DOM APIs that allow you to interact and traverse the DOM tree, (e.g.: `document.querySelector` and `document.querySelectorAll`), these APIs will prevent developers from penetrating the Shadow DOM boundaries defined by their LWC components. In this phase, we want to solve the LWC + Synthetic Shadow scenario.

To ensure we can solve the problem while continuing to guarantee the benefits that are provided by the synthetic-shadow polyfill for all other components running on the main window, we are currently proposing the following characteristics for the virtual-script design:

* Create a JS sandbox before the Synthetic Shadow Polyfill is evaluated.
  * The sandbox is created using a same-domain iframe.

* Use this JS sandbox to virtualize the DOM APIs in order to prevent these APIs to be subject to the Synthetic Shadow Polyfill patches:
  * This can be done by caching references to the to-be-patched APIs inside the sandbox before the synthetic shadow is evaluated.

* Evaluate one or more third party libraries inside this sandbox
  * This allows the code to run in a non-patched environment, while the rest of the document is running synthetic shadow.

* Package the above functionality in a web component which semantically matches the `<script>` tag API.

* Provide an additional mechanism to remap globals when needed.
    * The sandbox does NOT include any global name defined in the main window, whether it is defined before or after the creation of the sandbox. But the `<virtual-script>` can opt-in to share a global value between the sandbox and the main window, this must be done via the `extra-globals` argument passed to the `<virtual-script extra-globals="dataLayer">` element.

Other considerations for Phase I:

 * due to the explicit order of evaluation required for this to function, we plan to only support one sandbox that can be shared by multiple virtual scripts.
 * budget is around 5kb (minify/gzip).
 * performance penalties will be added to the virtual scripts as they run inside a membrane:
   * evaluation budget: 30% slower than evaluating the same script in the main window.
   * operation budget: 50% slower than executing the same task is the main window.
   * 3rd party scripts are rarely part of the critical path for the application to function.

### Phase II: Native Shadow

For applications which are using the native shadow, the underlying implementation will be different, a more complicated one. We are currently proposing the following characteristics for the virtual-script design:

* During the evaluation of the virtual script component class, patch `Element.prototype.attachShadow`, to record any shadow created in the app.

* When evaluating code via `<virtual-script>` element, create a new sandbox with patched DOM APIs to simulate a flatten tree.
  * All APIs that allow you immediate traversal up and down (e.g.: `elm.parentNode`) must be patched.
  * All Event retargeting mechanism must be unwind to flatten them up.
  * All deep traversal (e.g.: `elm.querySelector`) must be patched, and DOM filtering must be implemented by simulating the flatten representation of the application. This will require a query selector mechanism that can translate regular selectors into cross shadow selections (which is not trivial work)

* Package the above functionality in a web component which semantically matches the `<script>` tag API.

* Provide an additional mechanism to remap globals when needed.
    * The sandbox does NOT include any global name defined in the main window, whether it is defined before or after the creation of the sandbox. But the `<virtual-script>` can opt-in to share a global value between the sandbox and the main window, this must be done via the `extra-globals` argument passed to the `<virtual-script extra-globals="dataLayer">` element.

Other considerations for Phase II:

 * the virtual script web component "should" be registered before any shadow root is created, it is not longer a "must", the flip side of this is that pre-existing shadowRoots are just not going to be inspected by virtual scripts.
 * budget is less than 10kb (minify/gzip).
 * performance penalties will be added to the virtual scripts as they run inside a membrane:
   * evaluation budget: 30% slower than evaluating the same script in the main window.
   * operation budget: 1X slower than executing the same task is the main window.
   * 3rd party scripts are rarely part of the critical path for the application to function.
 * synthetic shadow and native shadow combinations can be allowed, and supported, although it will require an explicit order to run before synthetic shadow polyfill does it.

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

If we evaluate the above snippets with `<virtual-script>` instead, then `ga` global value will only be available into the sandbox. So `ga` will not be usable by anyone else in the entire application. We can however, allow for globals to be exposed into the main window. A syntax like the following could be provided:

```html
<virtual-script extra-globals="ga"></virtual-script>
```

- `virtual-script` container would not have access to globals defined by other scripts in the outer window unless they were defined in a `virtual-script` using `extra-globals`.
- Globals defined in `extra-globals` will be redefined in any `virtual-script` containers.
- They will also be available to any Locker-created sandboxes

Note: these global values are controlled side-channels between the main window and the sandbox, and should be used with caution. For example, creating a component that depends on `ga` to be defined, is an anti-pattern.

### Supported Attributes

`crossorigin`: indicates that the browser should provide more information to window.onerror when handling CORS errors

`defer`: indicates that the script should be executed after the document has parsed (has no effect if `src` is absent)

`integrity`: enables the browser to verify that a fetched resource is delivered without unexpected manipulation

`nomodule`: the script should not be executed in browsers that support ES2015 modules

`nonce`: user provided nonce values for CSP will be supported

`referrerpolicy`: indicates which referrer to send when fetching the script

`src`: the address of the resource

**async**

TBD describe asynchronous nature of `virtual-script`.

### Unsupported Attributes

`type`: all `virtual-script` resources will be `text/javascript`. `type="module"` will not be supported.

`charset`: this attribute is deprecated

`language`: this attribute is deprecated



### Known Limitations

* Currently there is an outstanding issue with Synthetic Shadow in that the Event object patching happens lazily, so we are unable to prevent its patching at the time of creating the sandbox, which means that events are still re-targeted in some cases. [See LWC PR #1569](https://github.com/salesforce/lwc/pull/1569).

### Testing

We will want to gather the top 20 common libraries and implement their behavior inside of `virtual-script` to ensure that we are properly accounting for all these libraries' use cases. This will be our integration test suite.

## Drawbacks

- The element may end up being anywhere from 5kb - 20kb.
- Debugging these 3rd party libraries will become harder due to the javascript membrane.

## Alternatives

- **Allow Shadow DOM to be toggled at a more granular level.** There is a forthcoming RFC for allowing component authors to decide if they want to opt in to Shadow DOM or not. This would mean we can keep top-level elements and layout components in the light DOM, but base components or managed components can choose to safeguard their internals. Until we allow this, `virtual-script` serves as a mitigation for application developers looking to add third party libraries to their apps.
- Do not apply the Shadow DOM polyfill at all: this would allow component authors to violate Shadow DOM encapsulation which is not an option.
- If we don't do this, then companies / websites building on our platform will not be able to integrate with common third party libraries.

## Adoption strategy

TBD

## How we teach this

* In OSS, just teach that replacing `<script>` with `<virtual-script>` is sufficient to allow bypassing the Shadow DOM encapsulation.

## Unresolved questions

- Do we want to support all the existing `script` tag attributes like `type`, `async` and `defer`?
- Is product security ok with 3rd party scripts running inside the virtual environment but without the same level of sandboxing used by Locker vNext?
