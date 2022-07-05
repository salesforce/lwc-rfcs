---
title: Light DOM components default to scoped styles
status: DRAFTED
created_at: 2022-07-05
updated_at: 2022-07-05
pr: https://github.com/salesforce/lwc-rfcs/issues/65
---

# Light DOM components default to scoped styles

## Summary

In the case of [light DOM components](https://rfcs.lwc.dev/rfcs/lwc/0115-light-dom), the default for `*.css` files will be [scoped](https://rfcs.lwc.dev/rfcs/lwc/0116-light-dom-scoped-styles) rather than unscoped. In addition, an explicit `*.unscoped.css` naming convention is added, for cases where unscoped styles are explicitly desired.

## Basic example

The current behavior for scoped styles is as follows:

| Filename       | Result   |
|----------------|----------|
| `*.css`        | Unscoped |
| `*.scoped.css` | Scoped   |

In this system, `*.css` means "unscoped" and `*.scoped.css` means "scoped."

(Note that this scoping mechanism is _not_ the same thing as shadow DOM style scoping, either in synthetic or native mode. What we're talking about here is [class-based style scoping](https://lwc.dev/guide/light_dom#implement-scoped-styles), which works in both light DOM and shadow DOM.)

The proposal is to change this to:

| Filename         | Shadow DOM | Light DOM |
|------------------|------------|-----------|
| `*.css`          | Unscoped   | Scoped    |
| `*.scoped.css`   | Scoped     | Scoped    |
| `*.unscoped.css` | Unscoped   | Unscoped  |

In this system, `*.css` defaults to "unscoped" for shadow DOM (i.e. the existing behavior, i.e. relying on shadow DOM for scoping), whereas the same filename in light DOM defaults to "scoped."

If developers prefer to be explicit, they can use `*.scoped.css`, which is always scoped, and `*.unscoped.css`, which is always unscoped.

## Motivation

Defaulting to unscoped styles in light DOM was a decision based on the browser's default behavior. If you drop a `<style>` into a light DOM web component, then those styles leak outside of the component.

However, this is a footgun for developers. LWC developers are used to using `*.css` and having those styles "scoped" (per shadow DOM scoping semantics). These developers may forget to use `*.scoped.css` for light DOM components, in which case their component will "work," but unknowingly, their styles will leak outside of the component, potentially across the entire document.

Changing the default behavior of `*.css` for light DOM components should nudge most developers towards the preferred solution – scoped styles. They will have to make an explicit decision to leak styles, by using `*.unscoped.css`.

## Detailed design

In the proposed system, developers can use any combination of the three filenames for the same component. E.g.:

```
component
  ├──component.html
  ├──component.css
  ├──component.scoped.css
  └──component.unscoped.css
```

These three filenames can be used with either shadow DOM or light DOM components.

As described above, `*.scoped.css` and `*.unscoped.css` always behave the same (scoped and unscoped, respectively), whereas `*.css` will vary depending on whether the component is light (scoped) or shadow (unscoped).

<details><summary>Implementation details</summary>

Currently we use `*.scoped.css` as a signal for whether a stylesheet should be scoped or not, and the `@lwc/template-compiler` appends a [`?scoped=true` suffix](https://github.com/salesforce/lwc/blob/cfee90b9b2f03fe0242cd1366b394e0ee7373227/packages/%40lwc/compiler/src/transformers/template.ts#L86-L87) to the URL of the import to signal this. E.g.:

```js
import _implicitStylesheets from './component.css';
import _implicitScopedStylesheets from './component.scoped.css?scoped=true';
```

In the new system, the `?scoped=true` suffix would vary depending on whether the importer is a light DOM or shadow DOM template. In light DOM, it would look like this:

```js
import _implicitStylesheets from './component.css?scoped=true';
import _implicitScopedStylesheets from './component.scoped.css?scoped=true';
import _implicitUnscopedStylesheets from './component.scoped.css';
```

...whereas in shadow DOM, it would be:

```js
import _implicitStylesheets from './component.css';
import _implicitScopedStylesheets from './component.scoped.css?scoped=true';
import _implicitUnscopedStylesheets from './component.scoped.css';
```

</details>

## Drawbacks

### Breaking existing behavior

Light DOM is considered "beta" and has not officially shipped in LEX yet. However, some folks may be using it off-core. In those cases, the behavior will change, and they will have to fix their existing components, if they were relying on `*.css` being unscoped for light DOM components.

### More complexity in the design

The old system (`*.css` is unscoped, `*.scoped.css` is scoped) was simple to describe. The new system has a 3x3 table to explain how it works.

Arguably the new system is "simpler," in that developers can mostly just use `*.css` and it will be "scoped." However, once you get into the weeds of what "scoped" actually means (e.g. the fact that scoping is useful for shadow DOM as well, e.g. in the case of a shadow-parent-light-child situation or [multiple templates](https://lwc.dev/guide/html_templates#render-multiple-templates)), then the existing system is actually much simpler.

### Breaking changes for bundlers/adapters

We will need to change the existing behavior in the LWC adapters for Rollup, Jest, Webpack, LWR, lwc-platform, and anywhere else that implements the LWC resolution algorithm. We may break existing unit tests and apps while getting the feature out the door. 

## Alternatives

### Make scoped styles the default everywhere

We cannot do this, because it would potentially break existing shadow DOM components. Even forgetting that light DOM exists, the addition of an extra class selector could change the specificity of the CSS rules, and thus the runtime styling behavior (in both synthetic and native).

### Keep the existing system

We could _not_ do this proposal, and teach developers how to use scoped styles in the existing system.

Arguably, this approach is not terrible because:

1) Developers already have a lot of new concepts to learn with light DOM (`lwc:render-mode`, `renderMode`, what `:host` means, security, etc.) – this is just one extra concept to learn.
2) 99% of the time, developers can just remember "Always use `*.scoped.css`," and it will match their intuition. For instance, even in the case of native shadow DOM, using scoped styles doesn't add a huge performance tax, and it avoids the aforementioned edge cases with shadow-parent-light-child and multi-template components.

## Adoption strategy

It will be a breaking change in some cases. We will have to communicate it very clearly.

# How we teach this

We will have to be very explicit about what "scoped styles" are, how they differ from shadow DOM style scoping, and how the naming conventions for CSS files affect which system you get, depending on whether your component is light or shadow. (Arguably we already have to teach this; now we are just teaching a different set of defaults and conventions.)

# Unresolved questions

None at this time.
