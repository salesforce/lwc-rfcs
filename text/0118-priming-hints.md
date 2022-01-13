---
title: Priming Hints
status: DRAFTED
created_at: 2022-01-04
updated_at: 2022-01-04
pr: https://github.com/salesforce/lwc/pull/????
---

# Priming Hints

## Summary

Priming is an out-of-band mechanism for ensuring that context-specific LWC modules and dependencies are made available in client caches prior to rendering. Priming is useful for both speeding up rendering of components when online, and for ensuring the ability to render unvisited components when offline.

One priming mechanism available for LWC is [Komaci] (https://rfcs.lwc.dev/rfcs/komaci)

This document suggests an enhancement to LWC that would allow components to provide hints to help drive more complete priming.

## Problem description

### Motivation

For maximum flexibility, some LWC priming mechanisms are made available to run out-of-band in a minimal Javascript environment without a DOM. This would be helpful for use cases such as priming LWCs in a client before a user has viewed them, and evaluating LWC dependencies in a headless server context.

Out-of-band priming mechanisms rely on static analysis to traverse component trees and identify dependencies. However, expanding the full LWC component tree may rely upon conditions that are only properly evaluatable when rendered in a DOM. For example, private members may be used to control the presence of specific children, and these members may be initialized only within the DOM lifecycle. These conditions pose barriers to priming mechanisms that run without rendering LWCs.

In order to increase the scope of priming possible, we suggest allowing component developers to provide priming hints. Priming hints can guide priming mechanisms into artificially substituting in property values that introduce more component sub-trees into the component's markup. This would allow the priming mechanism to override property value calculations and mimick happy path tree expansion, introducing a deeper component sub-tree to prime.

## Example

Consider a "bar" container that acts like a tab set. It holds a collection of items, and allows the user to interactively choose an item to be displayed in its default slot:

```html
<!-- bar.html -->
<template>
    <div class={computedClass} title={title}>
        <bar-header></bar-header>
        <slot></slot>
    </div>
</template>
```

The items look like the following:

```js
export default class BarItem extends LightningElement {
    @track _loadContent = false;

    @api
    loadContent() {
        this._loadContent = true;

        this.dispatchEvent(new CustomEvent('active'));
    }

    ...
}
```

```html
<!-- barItem.html -->
<template>
    <template if:true={_loadContent}>
        <slot></slot>
    </template>
</template>
```

The container lazily loads the contents of its children only when they are first selected for view. The bar container can control the currently visible barItem by calling the loadContent() method to load its content and using CSS to hide any previously displayed barItem.

For a prefetch mechanism based on static analysis, this structure poses a barrier. The conditional loading of barItem is governed by a member _loadContent that is triggered by user interaction. If the prefetch mechanism extracts composition information from markup, and executes resolution based on that analysis without a DOM or user interaction, the best it could hope for it priming the contents of the barItem that is first displayed by default. For some use cases of priming that is reasonable (such as for speeding up render while online). However, for priming for an offline use case one may desire a greater scope of priming - priming the contents of all the barItems. In this case, it would be desirable to provide a hint to the priming mechanism to say "please assume this if:true condition is true for priming purposes".

## Design

We propose adding new meaningful comments to LWC which allow components to opt-in to marking properties that a priming mechanisms could assume to be true. This will allow priming mechanisms to prime a greater scope of component contents.

These comments would be used on property getters, and their format would be:

```js
/**
 * priming-hint: <scope> <value>
 */
```

With the following values for "scope":
- always: priming mechanisms should always evaluate the given property with the given value for the purposes of priming through the component tree
- offline: priming mechanisms should evaluate the given property with the given value for the purposes of priming through the component tree only if the priming is targeted at an offline use case

We differentiate between these 2 scopes because online and offline priming use cases can be qualitatively different. Online priming is aimed at enhancing performance of rendering, with the assumption that further resources and data can be pulled from the server later on as necessary. Offline priming aims to prime a complete useful set of resources and data from the server up-front with no additional requests later. In general, online priming is more sensitive to performance and offline priming is more sensitive to missing scope.

For our bar example, we could modify the barItem component by replacing the use of a @track member with a new @api property that uses could make use of this priming hint to guide a priming mechanism to always prime its contents:

```js
export default class BarItem extends LightningElement {

    /**
     * priming-hint: always true
     */
    @api
    viewContent() {
        return this._loadContent;
    }

    @api
    loadContent() {
        this._loadContent = true;

        this.dispatchEvent(new CustomEvent('active'));
    }

    ...
}
```

```html
<!-- barItem.html -->
<template>
    <template if:true={viewContent}>
        <slot></slot>
    </template>
</template>
```

## Other Possible Strategies

Here is a brief survey of other strategies we considered for priming hints.

### Contextual Import

```js
import {primingContext} from @salesforce/priming;
...
get loadContent {
    return primingContext?.isPriming || _loadContent;
}
```

Here we have the property itself explicitly import an indicator of priming status and use it to drive the value to return for the property. This feels like it would be too much work for developers and they could easily calculate the property incorrectly.

### Annotation

```js
import {forcePriming} from @salesforce/priming;
...

@forcePriming
get loadContent {
    return _loadContent;
}
```

Here the component opts in to the priming-specific override by using a decorator. Using decorators like this seems out-of-step with other parts of the LWC specification.

### Metadata XML

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="xmlns=http://soap.sforce.com/2006/04/metadata">
    <isExposed>true</isExposed>
    <primingHints>
        <primingHint value="true">loadContent</primingHint>
    </primingHints>
</LightningComponentBundle>
```

We could express the priming overrides within metadata XML. This feels a little awkward because it's located separately from the code. A developer may miss making the corresponding change here when refactoring their component logic.

## How we teach this

Teaching about priming hints should be part of more comprehensive user docuemntation around static analysis and priming in LWC.

# Survey of prior art

// TODO