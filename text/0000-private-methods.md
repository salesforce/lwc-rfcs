---

## title: Private Methods for Lightning Web Components

status: DRAFTED
created_at: 2026-02-03
updated_at: 2026-02-06
pr: [https://github.com/salesforce/lwc-rfcs/pull/96](https://github.com/salesforce/lwc-rfcs/pull/96)

# Private Methods for Lightning Web Components

## Summary

The goal of this project is to enable [native private method support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_elements) into LWC. Private methods are created by using a `#` prefix and cannot be referenced outside of the declared class. This provides secure encapsulation. Elements and methods marked as private are not part of the typical inheritance chain and thus cannot be accessed by subclasses. The focus is explicitly only private methods, not private properties due to challenges with reactivity in LWCs ([reference](https://github.com/salesforce/lwc/issues/3537)). 

## Basic Example

We declare private methods directly on the component class using standard JavaScript private element syntax (the `#` prefix). This matches the native JavaScript authoring experience and requires no framework-specific APIs.

```javascript
class MyComponent extends LightningElement {
  #privateMethod() { ... }
}
```

Private methods can be referenced using dot notation: 

```javascript
this.#privateMethod();
```

Private methods cannot be accessed outside of the class body:

```javascript
(cmp) => cmp.#privateMethod(); // SyntaxError
```

## Motivation

The largest motivation for this feature is improving our security approach. Enabling private methods helps maintain the declared API contract of a component by restricting access to its internal methods. This will prevent unexpected context abuse and shadowing methods that alter the behavior of a component. Private methods are inherently secure by design, and using them eliminates entire classes of potential component vulnerabilities.

## Detailed Design

The following design proposal delegates to native browser behavior of private methods and is a backwards compatible solution, allowing us to make sensitive changes without impacting existing impelementations. [You can find a POC PR here.](https://github.com/salesforce/lwc/pull/5477)

If you were to try using a private method in an LWC today, it will fail with an error from [@babel/plugin-transform-class-properties](https://github.com/salesforce/lwc/blob/master/packages/%40lwc/compiler/src/transformers/javascript.ts).

![IDE tooltip showing an error: "LWC1007: /Users/a.chabot/qaFor258/privateMethodExample.js: Class private methods are not enabled. Please add `@babel/plugin-transform-private-methods` to your configuration."](https://github.com/user-attachments/assets/afbdb13f-4e26-4ccf-9a58-4b1fcd89dc64)

To implement private methods, the proposed solution involves transforming the method call both BEFORE and AFTER @babel/plugin-transform-class-properties. Both of these transformations will occur during compilation.

At the start, user code will look something like:

```javascript
#privateMethodCall() { ... }
```

We will transform that method declaration, via a newly written babel transform, to look like: 

```javascript
__lwc_component_class_internal_private_privateMethodCall()
```

Then it will go through the @babel/plugin-transform-class-properties. Since @babel/plugin-transform-class-properties does not see this as a private method, rather a regular method, it will not throw an error. Then the method name will be transformed back to match its original name, via another newly written babel transform: 

```javascript
#privateMethodCall() { ... }
```

**Round-Trip Validation**

The prefixed name is temporarily a plain method visible to every intermediate plugin, which means two things can go wrong silently: a developer could hand-write a method that happens to match the reserved prefix, and the reverse transform would incorrectly convert it to a private method or an intermediate plugin could drop/rename a prefixed method, leaving a mangled internal name in the final output. Either case would ship broken code with no compile-time signal.

To guard against this, the forward transform could record every prefixed name it produces in a `Set<string>` stored on `state.file.metadata`. The reverse transform would then check each prefixed method it encounters against that Set. If the name is absent, the method was user-authored, not generated, and the compiler should surface an error. After traversal, any name still in the forward Set that was never restored would indicate a lost method, and the compiler could report exactly which methods are missing.

**Section of Random Facts Relating to the Design**

- This feature is limited to implementing only private methods. It does not include not private properties or private accessors (getter/setters).
- Both of these transforms only get applied if enabled, using a compiler option

Private class members can only be accessed from within the class body itself, as defined by JavaScript's private fields specification. You cannot call `instance.#privateMethod()` from external code because it attempts to access the private method from outside the class context. Similarly, referencing `{#privateMethod}` in an LWC template violates this same encapsulation rule because the template exists outside the JavaScript class body. Both result in a SyntaxError caught before the code ever runs.  

## Drawbacks

**Drawback #1:** Performance.
This design requires AST traversal twice, once to rename the private functions to regular functions & then again to re-rename them as private functions. There is also a manual traversal of the tree to find the `Program` visitor. There is performance overhead to this AST traversal. 

**Drawback #2:** Complexity to LWC Compiler. 
As with all expansions, this project adds complexity and technical debt to the LWC compiler. This project is implementing two new babel transforms that will need to be maintained. 

## Alternatives

We have considered 4 other designs, which are explained below. 

**Alternate Design 1:** Implement [@babel/plugin-transform-private-methods](https://babeljs.io/docs/babel-plugin-transform-private-methods). 

As you can see in the screenshot above, the linter suggests to “Please add @babel/plugin-transform-private methods to your configuration” when using private methods in an LWC. This Babel transform lets you use JavaScript private methods in environments that don’t support them yet by transforming them into older, compatible JavaScript. In our use case, the Babel transform would perform compile time validation on private properties (anything without LWC decorators, such as @wire, @api, and @track). 

Implementation of this Babel plugin required using the `loose` flag due to it's dependency on @babel/plugin-transform-class-properties. When this flag is true, methods aren't truly private and private properties can get leaked to subclasses.

Furthermore, this design causes issues because class properties get renamed, thus becoming unusable in the template. While this solution does remove the error, it does not resolve the security issues within the components. It is also adding a polyfill for something that is already natively supported in browsers.

**Alternate Design 2:** Remove [@babel/plugin-transform-class-properties](https://babeljs.io/docs/babel-plugin-transform-class-properties).

@babel/plugin-transform-class-properties is the babel transform throwing the error in LWC right now. This alternative design suggests removing this babel transform from the LWC compiler completely. This Babel transform lets you use class fields in JavaScript and transforms them into code that works in older environments. By removing this transform, we would defer to the browser to handle class properties, including private elements. 

The main issue with this design is that it breaks reactivity on class properties, which is a core principle of LWC. [This problem is spelled out in more detail here](https://github.com/salesforce/lwc/issues/3537). 

**Alternate Design 3:** Add a newly created @private decorator. 

This alternative design is to introduce a new `@private` decorator for internal only components. Then use a custom babel plugin to convert `@private` annotated methods to be only accessible from within the class. This design offered the most promising path to unblocking internal teams, as well as allowing opportunities for growth to unblock external teams. However, it’s reiterating a design that is already native to JavaScript so feels repetitive. 

**Alternate Design  4:** Use WeakMaps. 

The final approach suggestion was to utilize WeakMaps within the internal components. This solution is a design at the component level, instead of at the compiler level. While component owners still need to migrate their LWCs once private methods are enabled, the overhead & boilerplate for the WeakMap implementation is much higher for component owners. 

## Adoption Strategy

New LWCs would need to implement their own private methods. Existing LWCs would need to migrate their existing methods to be private. We intend to create a linter to ensure consistent use of private methods. We will use this linter to enforce that any non-public method be private.

The design spelled out above is backwards compatible and does not break existing LWC code. It will be adopted internally before it is enabled and exposed for external customers.

A simple LWC example could be written for demonstration purposes.  

## How We Teach This

Documentation for private methods can link to the public MDN documentation, as we are enabling native JavaScript usage. 

## Unresolved Questions

N/A
