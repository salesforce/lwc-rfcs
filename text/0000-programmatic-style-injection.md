---
title: Programmatic style injection
status: DRAFTED
created_at: 2021-09-16
updated_at: 2021-09-30
pr: (leave this empty until the PR is created)
---

# Programmatic style injection

## Summary

This proposal describes how LWC components can programmatically associate CSS stylesheets to an LWC component.

## Basic example

```js
import { LightningElement } from 'lwc';
import myStyles from './styles.css';

export default Example class extends LightningElement {
    static stylesheets = [
        myStyles
    ];
}
```

## Motivation

CSS custom properties and CSS `::part()` are the standard mechanisms by which applications can influence the look and feel of shadow DOM components. Component authors are required to explicitly define those extension points that can be styled from outside the shadow tree. While this approach gives a certain level of control over the component appearance, in certain cases developers might want to take full control over the styles injected in the shadow tree. And this without compromising the component encapsulation. 

Component libraries are a prime example of this use case. Component authors want to expose high-quality base components with minimal styles and let application developers add their branding.

LWC implicit template and stylesheet resolution system makes it impossible today to extend from a base LWC component and override its styles at the same time. This proposal addresses this issue by exposing a mechanism to associate stylesheets directly with the LWC component class. 

## Detailed design

A new `stylesheets` static property is added on the `LightningElement` constructor. This property allows developers to define which stylesheets the component should use. It accepts an array of [LWC stylesheets](#lwc-stylesheets-modules). The base `LightningElement.stylesheets` static property returns an empty array.

The `stylesheets` property works on both light DOM and shadow DOM LWC components. This stylesheet injection mechanism complements the existing implicit template/stylesheet resolution mechanism.

The LWC engine caches the static `stylesheets` value for the lifetime of the application during the component class definition. Modifying the `stylesheets` property after the component code is evaluated will not affect the stylesheet injected by this component. 

For component inheritance, the LWC engine doesn't attempt to merge ancestor components stylesheets. Developers are in charge of merging (or not) the ancestors' stylesheets with their stylesheets.

```js
import { LightningElement } from 'lwc';
import ancestorStylesheet from './ancestor.css';
import childStylesheet from './child.css';

class Ancestor extends LightningElement {
    static stylesheets = [
        ancestorStylesheet
    ];
}

class Descendent extends Ancestor {
    static stylesheets = [
        ...super.stylesheets, 
        childStylesheet
    ];
}
```

### Injection order

The LWC engine injects stylesheets in the following order: first, the implicit stylesheet associated with the template and then all the stylesheets associated with the `stylesheets` property on the component class. This order allows the styles that are programmatically associated with the component to override the styles that are implicitly loaded via the template.

In the following example the `<x-foo>` stylesheets will be injected in the following order: `foo.css`, `custom-a.css` and `custom-b.css`.

```
x/
└── foo/
    ├── custom-a.css
    ├── custom-b.css
    ├── foo.js
    ├── foo.html
    └── foo.css
```

```js
// x/foo.js
import { LightningElement } from 'lwc';
import customStyleA from './custom-a.css';
import customStyleB from './custom-b.css';

export default Foo class extends LightningElement {
    static stylesheets = [
        customStyleA,
        customStyleB
    ];
}
```

### LWC stylesheets modules

Programmatically importing a stylesheet module from JavaScript using an import declaration is not a new concept introduced by this RFC. Such import is already supported by the LWC compiler. In fact, the template compiler is already using this mechanism to import the stylesheet implicitly associated with the template.

However, until now, the compile-time behavior and runtime behavior for stylesheet modules have never been fully specified. This section fills this specification gap.

Stylesheets modules are identifiable by their `.css` extension. This kind of module only exposes a single default export. The exported value is an opaque object, in other words, the internal implementation of the stylesheet is not exposed to developers. Developers shouldn't rely on the shape of those objects as it may change.

```js
import stylesheetA from './styles-a.css';
import stylesheetB from './styles-b.css';
```

Scoped stylesheet modules work similarly than standard stylesheet modules. They can be imported using the `.scoped.css` extension.

```js
import scopedStylesheet from './styles.scoped.css';
```

#### Comparison with CSS module script

LWC stylesheet modules work similarly to native [CSS module script](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/css-modules-v1-explainer.md). This feature isn't fully standardized yet and is today only implemented in Chromium-based browsers. There are few differences between the two proposals worth calling out:
- LWC stylesheet modules can be imported using a standard import statement. The LWC compiler infers from the `.css` file extension that the file has to be treated as a CSS module. On the other hand, CSS module scripts require the use of [import assertions](https://github.com/tc39/proposal-import-assertions).
- LWC stylesheet modules default export is an opaque object. CSS module scripts default export is an instance of [`CSSStyleSheet`](https://developer.mozilla.org/en-US/docs/Web/API/CSSStyleSheet).

```js
// LWC stylesheet modules
import styles from "./styles.css";
console.log(styles + '') // [object LWCCSSStyleSheet]

// CSS modules
import styles from "./styles.css" assert { type: "css" };
console.log(styles + '') // [object CSSStyleSheet]
```

An alternative to exporting LWC stylesheet module as an opaque object is to have a utility function `registerStylesheet` that registers compiled stylesheets so that component authors cannot modify or replace stylesheet functions. This would be a similar approach on component templates ([`isTemplateRegistered`](https://github.com/salesforce/lwc/blob/1b85b4606f8cc8a18c368d9ce3b9c3616043f10c/packages/@lwc/engine-core/src/framework/secure-template.ts#L17)).

#### LWC stylesheet import enforcement

As mentioned above, the LWC compiler implicitly loads the stylesheet associated with each template. Internally, the template compiler generates an import statement with the template file name and the `.css` extension. Since stylesheets are optional, the LWC module resolver currently ignores non-existent stylesheet imports.

This behavior is problematic for explicit stylesheet imports as the compilation doesn't fail if the imported stylesheet can't be resolved. To solve this issue, the LWC compiler should treat component's implicit stylesheet import as optional and other explicit stylesheet imports as required. For DX improvement, LWC compiler should fail compilation or at least warn developers if an explicit stylesheet import cannot be resolved before treating non-existent stylesheet imports as empty imports. 

A straightforward implementation is to check whether a stylesheet import is implicit by checking if the importee's path is the same as importer's path. 

Another possible implementation is that a new `optional` query string can be added to the import identifier to indicate whether or not the LWC compiler should report an error when the file is missing.

```js
// Required LWC stylesheet import. Fails during compilation when missing.
import styles from "./styles.css";

// Optional LWC stylesheet import. Resolve to an empty stylesheet if missing.
import styles from "./styles.css?optional=true";
```

This query string should only be used by the LWC template compiler to differentiate between the optional stylesheet imported from templates and the required stylesheet explicitly imported by developers.

The import assertion proposal leaves the the door open for [potential future extensions](https://github.com/tc39/proposal-import-assertions#follow-up-proposal-evaluator-attributes) to avoid bloating the query string.

```js
// This syntax was discussed as part of the import assertion proposal and might be added in a
// follow-up proposal.
import styles from "./styles.css" assert { type: "lwc/css" } with { optional: true };
```

## Drawbacks

With this proposal, developers have now the capability to extends any other component to inject their own styles. This is an issue if those components aren't meant to be extended. 

## Alternatives

TBD

## Adoption strategy

This proposal solves a narrow use-case. It is not expected for this API to be widely adopted.

# How we teach this

A new section covering this topic will be added to the documentation.

# Unresolved questions

None