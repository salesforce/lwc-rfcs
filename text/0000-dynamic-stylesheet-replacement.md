---
title: Dynamic stylesheet replacement
status: DRAFTED
created_at: 2022-01-27
updated_at: 2022-01-27
champion: Nolan Lawson (nolanlawson)
pr: https://github.com/salesforce/lwc-rfcs/pull/76
---

# Dynamic stylesheet replacmeent

## Summary

[Programmatic stylesheets](https://lwc.dev/guide/css#associate-css-stylesheets-to-a-component-programmatically) are great, but they don't fully solve the styling needs of component authors. This proposal describes a complementary API that provides additional flexibility for associating stylesheets with templates.

## Basic example

A component with a dark and light mode:

```js
import template from './base.html';
import darkStylesheets from './dark.css';
import lightStylesheets from './light.css';

export default class extends LightningElement {
  @api darkMode = false;
  
  render() {
    const stylesheets = this.darkMode ? darkStylesheets : lightStylesheets;
    return {
      template,
      stylesheets
    };
  }
}
```

In this example, the component can render `dark.css` or `light.css`, depending on whether the `darkMode` property is true or false. Either way, it uses a single template: `base.html`.

## Motivation

Programmatic stylesheets use the `static stylesheets` property. This is ideal for cases where the component
author wants to extend another component and apply additional styles:

```js
import myStylesheets from './myStylesheets.css'

export default class extends SomeOtherComponent {
  static stylesheets = [...super.stylesheets, ...myStylesheets];
}
```

However, if authors want to render different stylesheets based on a _non-static property_ (e.g. `darkMode` in the above example), or some other non-statically-determined state (e.g. `this.template.synthetic`), then programmatic stylesheets would require an additional wrapper component whose only job is to render the correct child component class, as well as mixins to create those classes.

Below is an example of how you could do this using programmatic stylesheets:

<details><summary>Click to see</summary>

```js
// base.js
import template from './component.html';

export default class extends LightningElement {
    render() {
        return template;
    }
}
```

```js
// styleMixin.js
export const StyleMixin = (superclass, stylesheets) => {
    return class extends superclass {
        static stylesheets = stylesheets;
    };
};
```

```js
// dark.js
import Base from './base.js';
import { StyleMixin } from './styleMixin.js';
import darkStylesheets from './dark.css';

export default StyleMixin(Base, darkStylesheets);
```

```js
// light.js
import Base from './base.js';
import { StyleMixin } from './styleMixin.js';
import lightStylesheets from './light.css';

export default StyleMixin(Base, lightStylesheets);
```

```html
<!-- wrapper.html -->
<template lwc:if={darkMode}>
    <x-dark></x-dark>
</template>
<template lwc:else>
    <x-light></x-light>
</template>
```

```js
// wrapper.js
export default class extends LightningElement {
  @api darkMode;
}
```

</details>

This is a lot of code! Plus it requires the additional wrapper component.

Another goal of this RFC is to fully remove the [hacks that component authors are using](https://github.com/salesforce/lwc/issues/2826) to apply different stylesheets to the same template:

```js
import template from './template.html';
import darkStylesheets from './dark.css';
import lightStylesheets from './dark.css';

export default class extends LightningElement {
  @api darkMode;
  
  render() {
    // DEPRECATED - DO NOT DO THIS!
    template.stylesheets = darkMode ? darkStylesheets : lightStylesheets;
    template.stylesheetToken = darkMode ? 'dark' : 'light';
    return template;
  }
}
```

Programmatic stylesheets can replace some of these hacks, but not all. This proposal aims to replace _all_ of these use cases.

## Detailed design

LWC components are already able to [render multiple templates](https://lwc.dev/guide/html_templates#render-multiple-templates). However, the assumption is that component authors will
only switch between different HTML files:

```js
import a from './a.html';
import b from './b.html';

export default class extends LightningElement {
  @api shouldUseA;
  
  render() {
    return shouldUseA ? a : b;
  }
}
```

(These HTML files may have their own implicitly-associated stylesheets, but component authors cannot reuse the same HTML file with different stylesheets.)

This proposal augments the current behavior of the `render()` function. In addition to being able to return a template, the `render()` function can also return an object containing the `template` and `stylesheets` properties:

```js
import template from './template.html';
import a from './a.css';
import b from './b.css';

export default class extends LightningElement {
  @api shouldUseA;
  
  render() {
    return {
      template,
      stylesheets: shouldUseA ? a : b
    }
  }
}
```

The stylesheets are injected in the following order:

1. Stylesheets implicitly associated with the `template`
2. Stylesheets from the returned object's `stylesheets`
3. Static `stylesheets` on the component

This allows component authors to override the stylesheets that are implicitly associated with the `template`. For instance, they could have some shared default styles defined in `template.css`, whereas `a.css` and `b.css` would selectively override those defaults.

LWC still makes no attempt to remove stylesheets, or to ensure a particular ordering _between_ `render()` calls. (This is in line with existing behavior.)

### Restrictions

The returned object _must_ contain a `template` property pointing to a valid HTML template function.

The returned object _may_ contain a `stylesheets` property pointing to a valid stylesheet function or recursively deep array of stylesheet functions.

For compatibility with the `static stylesheets` property, the validation is exactly the same. The validation is skipped only if the `stylesheets` property does not exist on the object.

```js
return { template };              // No validation required
return { template, stylesheets }; // Stylesheets must be validated
```

In short (and recapitulating the validation of `static stylesheets`): if `stylesheets` is not a function or array of deeply-nested functions, then it will fail validation and be ignored (i.e. considered the same as if the `stylesheets` key did not exist).

> **Note:** At some point in the future, we may also validate that the stylesheet was registered with the LWC engine, or that it is an "opaque" object. But this is not a hard requirement for this feature.

Similarly, the value of `template` must be a registered template function. If not, a runtime error will be thrown. (This is [the same](https://stackblitz.com/edit/salesforce-lwc-dppg2e?file=src/modules/x/app/app.js) as if the `render()` function had directly returned an invalid object.)

## Drawbacks

This adds additional complexity to the codebase, and additional cognitive load for developers, as there are multiple ways to "do dynamic styles."

Additionally, we will need some way to vary the style token (for both synthetic shadow and scoped styles) to be reset when the stylesheets change. (This is due to the fact that LWC only injects stylesheets; it never removes them.) This will have to be done carefully so that we do not over-inject stylesheets (e.g. creating a random style token on every `render()`, so each `render()` injects brand-new `<style>`s that only grow with time). Most likely this will mean computing a unique scope token for a given _unique ordered set_ of flattened stylesheets. (Otherwise, unexpected behavior could occur if the user switches between stylesheet arrays whose contained stylesheets are partially shared.)

## Alternatives

The impact of not doing this is that component authors may continue to use template mutation hacks, since they won't have another option to achieve their design goal.

## Adoption strategy

This is net-new functionality and does not have backwards-compatibility risks. Returning a plain object from a `render()` function throws an error today, so developers cannot be relying on this.

# How we teach this

We will have to promote this feature alongisde programmatic stylesheets and explain when you would one or the other (or both). For component authors who are already using the template mutation hack, we can probably guide them directly to this solution, as it is the most direct replacement.

# Unresolved questions

None at this time.
