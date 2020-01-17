---
title: Partial Template
status: DRAFTED
created_at: 2020-01-16
updated_at: 2020-01-17
pr: (leave this empty until the PR is created)
---

# Patial Template

## Summary

Partial templates allow the ability to import html templates into another template via
the normal variable syntax `{template}`.

## Basic example

```js
import partialTemplate from './partialTemplate.html';

export default class Example extends LightningElement {
    partialTemplate = partialTemplate;
}
```

```html
<template>
  Parent
  {partialTemplate}
</template>
```

## Motivation

In investigating performance issues in components that are used hundreds of times on a
page it was noted that creating reusable child components; while organizationally nice,
were not as performant as inlining the HTML into the template;

In other instances it was noted that large conditonal blocks of HTML could be more organized
into seperate template files.

## Detailed design

Templates can be rendered into a component one of two ways currently.

- By matching the name of the component. Ex: `cmp.js` / `cmp.html`
- By returning a default import in the `render() {}` method.
  - Ex: `import cmpTemplate from './cmp.html'` -> `render() { return cmpTemplate; }`

A template is a constant and cannot be modified. Partials will inherit this same behavior.

This simplifies things, but still may lead to complications in optimizations [unfamiliar with
what this entails, so may need LWC context here]. Example:

```
get dynamicTemplate() {
    return this.conditonal ? template1 : template2;
}
```

## Drawbacks

Why should we *not* do this? Please consider:

- Conditionally swapping out partial templates could be performantly bad in themselves negating
  the benfit if we're not able to optimize for this.
- Recursion good/bad.

> Note: Developers coming from other frameworks would expect this to work this way.

## Alternatives

The other design choice would be to compile time import via an include syntax.

```
<template lwc:include="./template.html"></template>
```

This syntax greatly cuts down on what would be possible and introduces verbose
conditional blocks of `if:true` that could lend themselves to unnecessary rerenders.

## Adoption strategy

Since this is not a breaking change and purely an enhancement to an existing syntax developers
adoption will be optional. If they decide to reorganize existing components they'll go in with
expectations partials are the way going forward to make complex templates more readable.

In the examples above we may want to recommend reoganization of existing components if one was
currently heavily nesting components.

# How we teach this

For any developer coming from JSX environment this pattern is already normal practice. Showing a
simple example is enough to explain how partials work.

Currently a developer would assume this syntax would include a partial and not `.toString()` the
JS template function.

# Unresolved questions

- As with all feature this will depend heavily on performance.
