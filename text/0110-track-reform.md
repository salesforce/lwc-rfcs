---
title: Track reform
status: draft
created_at: July 1, 2019
updated_at: November 4, 2019
---

# Track Reform

## Status

- Start Date: 2019-05-08
- RFC PR: https://github.com/salesforce/lwc-rfcs/pull/4

## Summary

This RFC describes the reform for the tracking mechanism and the `@track` decorator.

## Goal

* Make template react to mutations in all fields (including private fields once enabled) without requiring the `@track` decorator.
* Use `@track` decorator only when you want track fields that can hold values other than primitives (react to mutations in the field value).

## Motivation

There is sometimes confusion about what to track and what not to track with the `@track` decorator. And even when you don't track a field, there is confusion about what happen with that field when it is mutated, and why sometimes the new value appears in the template. This confusion is unnecessary.

Another confusion is when for `@wire` decorator reactivity works fine even when no `@track` is used, it just work, making `@track` very confusing.

## Proposal

### Only use the `@track` decorator when you really need it.

The runtime/compiler will extract all fields that are defined by a class, and make the template observe mutations of any of those fields updating the UI accordingly.

Mutations on fields (ex: `this.x = 5;`) represent the majority of the use cases, but sometimes you need a field to hold a value other than a primitive, mutated ex: `this.address.city = 'San Francisco'`. Only in such cases you will need to use the `@track` decorator.


### Compiler changes

When the compiler compiles the class, it can extract any field that is not decorated with `@api`, `@track` or `@wire`, and pass the metadata through the registerDecorators call.

Notice that a vm is needed in order to observe a field mutation. registerDecorators will only save the class fields information in the decoratorsMeta, that will be used later on when the ComponentDefinition is created, only then we know with certain that the class represents a component.

For every class field that is present in the decoratorMeta of the ComponentDefinition, we will create a getter and a setter in the prototype in order to observe mutations in to the class field. No change is needed to the logic in the engine.

### Backwards Compatible

The reason why this change is backward compatible, and safe to do it, is the fact that today, a field that is not decorated, but used from the template is not a static value as some may assume, the field value is still accessed every time the component is re-rendered, making the value to be some sort of passive tracking. The only difference will be that, in some cases, the component will be re-rendered where in the past it wasn't, becoming deterministic.

The following is an example:

```js
export default class Foo extends LightningElement {
    // assume that x and y are used in the template
    @track x = 1;
    y = 2;
    foo() {
        // increment both
        this.x++;
        this.y++;
    }
    bar() {
        // increment only x
        this.x++;
    }
    baz() {
        // increment only y
        this.y++;
    }
}
```

In the example above, calling `foo()` and `bar()` will always render the latest on the next tick, while calling `baz()` might or might not update on the next tick, depending on others mutations in the current job. With this reform, they will always get the latest on the next tick, no matter what.

Also, non decorated class fields will maintain identity because in the proposed implementation we achieve observability of such fields without using proxies.

### Benefits

* No need to learn the `@track` decorator until you really need it, which is to track mutations in the value of the field. Ex:

```html
<template>
    <p>Your address is: {address.street}, {address.zipCode}</p>
</template>
```

```js
    // This statement mutates the field
    // it will be reflected on the template by default
    this.address = {
        street: 'Mission St',
        zipCode: '94102'
    }

    // this statament mutates the value of the field
    // and it won't be reflected in the template.
    // you need to use @track to properly reflect this in the template.
    this.address.zipCode = '94104';
```

* Performance improvement: since there is no need to use proxies to observe primitive values (majority of use cases). Suppose we have a base component that is heavily used today (ex: icon) with a tracked property (ex: name, and it receives a string); by removing the track, which is not needed anymore for this field, we will remove the extra checks of apply a proxy to the value.

* Updates in the UI after a mutating a field are more deterministic now because all fields are observed.

* No need to ask whether a field should be tracked or not (we get this question a lot). It should come naturally to developers: You start by adding class fields until you mutate the value of a property and you don't see it reflected on the template, then you read the documentation and see that in such cases where you need to react to changes in the value of the field, `@track` is your friend.
