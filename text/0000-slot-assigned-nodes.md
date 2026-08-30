---
title: Slot Assigned Nodes
status: DRAFTED
created_at: 2026-02-05
updated_at: 2026-02-05
pr: https://github.com/salesforce/lwc-rfcs/pull/97
---

# Slot Assigned Nodes

## Summary

To observe slot data requires a bit of boilerplate. The goal would be to remove this boilerplate especially around conditional slot parents or nodes.

## Basic example

> Note: Normalize `nodes` by removing all `node.type === Node.TEXT_NODE` with `node.nodeValue === ''`. This simplifies logic for devs.
> Note2: Using `slot` as the placeholder name, but could copy lit's `queryAssignedNodes()`.

```typescript
import { LightningElement, slot } from 'lwc';

export default class Playground extends LightningElement {
    showDefaultSlot = true;
    showNameSlot = true;

    @slot()
    slotDefault(nodes) {
        this.showDefaultSlot = nodes.length > 0;
    }

    @slot('name')
    slotName(nodes) {
        this.showNameSlot = nodes.length > 0;
    }

    // alternatively match lit's or support both formats
    @slot()
    bodyNodes: Array<Node>;

    @slot('name')
    nameNodes: Array<Node>;

    get showBody() {
        return this.bodyNodes.length > 0;
    }

    get showName() {
        return this.nameNodes.length > 0;
    }
}
```

template...
```html
<div hidden={hasSlot}>
  <slot></slot>
</div>
```

## Motivation

Simplifies monitoring slots by removing boilerplate. See lit's [`queryAssignedNodes()`](https://github.com/lit/lit/blob/main/packages/reactive-element/src/decorators/query-assigned-nodes.ts)

## Detailed design

See: [`queryAssignedNodes()`](https://github.com/lit/lit/blob/main/packages/reactive-element/src/decorators/query-assigned-nodes.ts)

## Drawbacks

Why should we *not* do this? Please consider:

- implements another decorator for something that can be done with a few lines of code.
- devs may also request `queryAssignedElements()`.

## Alternatives

Alternatively one could also support `<template lwc=if={hasSlot}>`, but this would require even more code and `MutationObserver`. Prototyped this with synthetic shadow and native shadow and while it works it might be too heavy.

Not sure there is any benefit in the long run to this vs `hidden={}` syntax.

## Adoption strategy

Document in or below the template docs.

# How we teach this

Using standard naming like `lit` might make more sense. Maybe even 1:1 implementation.

# Unresolved questions

TBD?
