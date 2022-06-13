---
title: Scoped Slots in Light DOM
status: DRAFTED
created_at: 2022-06-10
updated_at: 2022-06-22
pr: 
---

# Scoped Slots in Light DOM

## Summary

[Scoped slots](https://v3.vuejs.org/guide/component-slots.html#scoped-slots) is a feature from the Vue
framework, that allows passing data from a child component to the content slotted from a parent component
in a declarative fashion.

One of the most common use-cases for this is to use a generic component to render an array of items and
let the parent component customize how each item is rendered. For example a generic infinite list
component, a virtualized list component or, a data table.

*This proposal is for adding Scoped Slots only to Light DOM.*

## Basic example

```html
// x/parent.html
<template> <!-- parent need not be light DOM -->
    <x-list items={items} lwc:slot-data="item">
        <span>{item.name}</span>
    </x-list>
</template>

// x/list.html
<template lwc:render-mode="light">
    <ul>
        <template for:each={items} for:item="item">
            <li key={item.id}>
                <slot lwc:slot-bind={item}></slot> <!-- delegating rendering of children to the parent -->
            </li>
        </template>
    </ul>
</template>
```
The `<x-parent>` component is slotting `<span>{item.name}</span>` into `<x-list>` but the `item` is supplied
by `<x-list>`, the child component. It's essentially like `<x-parent>` is slotting a function that actually
returns the slotted content.

This allows `<x-parent>` to control how each item renders while leaving the looping/repeating logic to `<x-list>`.
`<x-list>` is now free to create as many slots as it wants while using the same "slotting function" passed from the parent.
This allows `<x-list>` to be a virtual list or an infinite list.

It's important to note that for this to happen, slots need to be lazily rendered.

* Native shadow DOM slots: the slotted content is eagerly rendered regardless of there being a slot to “accept” the content. The component slotting the content is in charge of appending and removing the content from the DOM.
* Scoped slots: the slotted content is lazily rendered; the slotted content will only be rendered if there is a slot to “accept” the content. The component with the <slot> elements is the one in charge of rendering the slotted content passed by the parent component.

It is for this reason that Scoped Slots are allowed only in Light DOM components.

## Detailed design

Two new directives, `lwc:slot-bind` and `lwc:slot-data` are introduced that are used to bind data to the slotted content
and to specify, in the parent, a placeholder to accept the bound data respectively.

* ### `lwc:slot-bind`
It has to be on a `<slot>` tag and should be an expression that is bound to a value within the child's scope. It can only
be present on a Light DOM template. The compiler will throw an error if `lwc:slot-bind` is used in a shadow DOM template.
* ### `lwc:slot-data`
It has to be present on a custom element and should be a string literal that represents the variable name that will be
used in the slotted content to reference the passed data. The custom element has to be a Light DOM LWC element. This check,
though, will be performed at runtime because the LWC compiler lacks the necessary information to throw at compile-time.

### Named Slots
Accepting scoped slots for named slots presents a few logistical challenges illustrated below.
```html
<template>
    <x-child lwc:slot-data="defaultSlotData">
        <h1>Content that goes into default Slot {defaultSlotData.title}</h1>
        <h2 slot="namedSlot">Named slot content {namedSlotContent.title}</h2> // where should namedSlotContent be declared?
        <h3 slot="otherNamedSlot">{defaultSlotData.title}</h3> // `defaultSlotData` is actually out of scope, but *looks* in scope for a template author
    </x-child>
</template>
```

To address the above concerns, content slotted into named slots needs to be grouped together using a `<template>` tag
and `lwc:slot-data` should be used on the `<template>`. 
```html
// x/parent.html
<template>
    <x-child> <!-- no `lwc:slot-data` on the component tag -->
        <template slot="slotname1" lwc:slot-data="slot1data">
            <p>{slot1data.name}</p>
        </template>

        <template slot="slotname2" lwc:slot-data="slot2data">
            <p>{slot2data.title}</p>
        </template>

        <template lwc:slot-data="defaultdata"> <!--// for default slot-->
            <p>{defaultdata.title}</p>
        </template>
    </x-child>
</template>

// x/child.html
<template lwc:render-mode="light">
    <slot name="slotname1" lwc:slot-bind={slot1data}></slot>
    <slot name="slotname2" lwc:slot-bind={slot2data}></slot>
    <slot lwc:slot-bind={defaultdata}></slot>
</template>
```

When using named slots, the child custom element does not accept `lwc:slot-data`. There cannot be any elements other than
`<template>` as direct children of the custom element. The check for these conditions will happen at compile time.
The template author can either use the named slot syntax or regular default slot syntax, but cannot combine both.

### Nested Scoped Slots
Scoped slots that are nested within other scoped slots bring in new levels of flexibility for component authors.
```html
<template>
    <x-table data={data} lwc:slot-data="row">
        <x-row row={row} lwc:slot-data="column"> <!-- this is rendered for every row in the table -->
            <span> <!-- this is rendered for every column in the row -->
                Coordinates: {row.number} - {column.number} <!-- can refer to both `row` and `column` -->
            </span>
        </x-row>
    </x-table> 
</template>
```


## Drawbacks

There are two main drawbacks: (1) not being available in Shadow DOM and (2) it brings in a new syntax for named slots.

There aren't any features that are exclusive to Light DOM (scoped styles is one, but Shadow DOM has an equivalent). Scoped
Slots is the first feature where the framework branches off into building something that works for only Light DOM components
which, at the time of writing this proposal, are very less in number when compared to Shadow DOM components.

Scoped Slots can't be used with custom elements that are *not* written in LWC. The new syntax for Named Slots especially
deviates from the standards.

## Adoption strategy

This new feature does not break any existing components, it simply adds a new feature that developers have to opt for.
Components that use workarounds to implement some use-cases that Scoped Slots need to be re-written to benefit from
Scoped Slots.

## Unresolved questions

- What should the directives be called instead of `lwc:slot-data` and `lwc:slot-bind`?
- Should Shadow DOM be supported?

## Appendix 1: Scoped Slots in Shadow DOM
Scoped Slots in Shadow DOM aren't supported by any other web component framework. The reasons boil down to slots in native
shadow DOM being eager, and it being the browser that does the slotting leaving us with little maneuverability.

That said, there are a few ways we can implement scoped slots in Shadow DOM. Let’s take the infinite list example to
illustrate this.

```html
<template>
   <x-list>
       <div slot:scope="item">
            {item.id} - {item.name}
       </div>
   </x-list>
</template>
```

* *Approach #1: Render the scoped content in the <x-list> shadow tree.*

Passing the scoped content fragment is transferred to the <x-list> component and the <x-list> component renders the items in its shadow tree. Example (https://gist.github.com/pmdartus/cd490a82617e025a2e73f79f7f9888b3#file-shadow-html)

```html
<x-list>
   #shadow-root
   |  <div>1 - Buy groceries</div>
   |  <div>2 - Clean up my desk</div>
   |  <div>3 - Do the laundry</div>
</x-list>
```

This approach is quite similar to the one explained above with the main advantage the template contains the scoped slot definition. This approach allows the scoped slot to reference properties on its original component.

However, this approach introduces new issues. Since the scoped slot is rendered in the <x-list> shadow tree, the <x-list> shadow tree styles are applied to the rendered content. This approach also breaks shadow DOM encapsulation because event handlers attached to the scoped slot will be invoked on elements that are rendered in another shadow tree.

* *Approach #2: One named slot for each scoped slot item.*

Instead of rendering the scoped slot content in the <x-list> shadow tree, the scope slot is rendered in its light DOM
and named slots are generated in the shadow tree. The named slots are then matched with the scoped slot content by
applying the slot attribute. [Example](https://gist.github.com/pmdartus/cd490a82617e025a2e73f79f7f9888b3#file-declarative-html)

```html
<x-list>
   #shadow-root
   |  <slot name="scoped-1"></slot>
   |  <slot name="scoped-2"></slot>
   |  <slot name="scoped-3"></slot>

   <div slot="scoped-1">1 - Buy groceries</div>
   <div slot="scoped-2">2 - Clean up my desk</div>
   <div slot="scoped-3">3 - Do the laundry</div>
</x-list>
```

This approach solves the 2 main drawback from approach #1 as the scoped slot content is rendered in the parent component shadow tree. The main drawback of this approach is that it will make the diffing algo more complex because it require coordination between the parent and the child component.

* *Approach #3: Manual slot allocation.*

This approach is similar to approach #2 with the only difference that slots content are imperatively assigned. This
leverage the new [Imperative shadow slot distribution API](https://github.com/WICG/webcomponents/blob/gh-pages/proposals/Imperative-Shadow-DOM-Distribution-API.md), currently only implemented in Chrome. [Example](https://gist.github.com/pmdartus/cd490a82617e025a2e73f79f7f9888b3#file-imperative-html)

```html
<x-list>
   #shadow-root
   |  <slot></slot>     <!-- imperative assigned to 1st element -->
   |  <slot></slot>     <!-- imperative assigned to 2nd element -->
   |  <slot></slot>     <!-- imperative assigned to 3rd element -->

   <div>1 - Buy groceries</div>
   <div>2 - Clean up my desk</div>
   <div>3 - Do the laundry</div>
</x-list>
```

Same advantages and drawback as approach #2. The main show-stopper with this approach is that it requires JavaScript
to do the content allocation which make it unusable for SSR.
