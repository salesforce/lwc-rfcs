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

// TODO: example based on lightning-tab/lightning-tabset

## Design

// TODO: exact format

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

// TODO

# Survey of prior art

// TODO