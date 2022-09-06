---
title: Scoped Slots in Light DOM
status: DRAFTED
created_at: 2022-06-10
updated_at: 2022-08-31
champion: Mohammed Abdul Sattar
pr: 
---

# Scoped Slots in Light DOM

## Summary

Scoped slot allows slotted content to accesss to data of a child component, in a parent component's HTML template in a declarative fashion.

*This proposal is for adding Scoped Slots capability in Light DOM components only.*

## Motivation
As of today, a parent component can have slot content referencing its own data. This works great when the parent has the data required to render the content and let the child component take care of slotting the content. However, there are use cases where the parent wants to reference the child's data in the slotted content.

One of the most common use-cases for this is to use a generic container component that provides a specilized capability, such as rendering an array of items, and let the parent component customize how each item is rendered. For example a generic infinite list component, a virtualized list component or, a data table.

To allow the parent to customize the slotted content, it requires access to the child component's data. 

A specific usecase for scoped slots is point and click app builders. The builder can provide container components like lists, grid etc. Users can drag and drop components into the grid, customize the component's rendering.

## Prior Art

### Vue.js
[Scoped slots](https://v3.vuejs.org/guide/component-slots.html#scoped-slots) was a concept first introduced in Vue.js [v2.1.0](https://github.com/vuejs/vue/releases/tag/v2.1.0). Vue.js allows passing individual properties and objects, default slots and named slots. 
Example: 
1. Passing properties to default slot
```html
<!-- Parent -->
<template>
    <Child v-slot="item">
        <span>{{ item.id }} {{ item.name }}</span>
    </Child>
</template>
```
```html
<!-- Child -->
<template>
    <slot :id="123" :name="Vue"></slot>
</template>
```
2. Passing object
```html
<!-- Parent -->
<template>
    <Child v-slot="item">
        <span>{{ item.id }} {{ item.name }}</span>
    </Child>
</template>
```
```html
<!-- Child -->
<template>
    <slot v-bind="item"></slot>
</template>
```

3. Named slot
```html
<!-- Parent -->
<template>
    <Child>
        <template v-slot:body="item">
            <span>{{ item.id }} {{ item.name }}</span>
        </template>
    </Child>
</template>
```
```html
<!-- Child -->
<template>
    <slot name="body" v-bind="item"></slot>
</template>
```
Note: LWC has a slightly different syntax for specifying the slot name for the slotted content in a parent component's markup. It does not require/allow the usage of a `<template>` to demark the slotted content. Instead, the name directly goes on the content. For example: 
```html
<x-child>
    <span slot="body">{ item.id } { item.name }</span>
</x-child>
```
To implement this proposal, LWC will add support for using `<template>` to demark a named slot content.

### Svelte
In [Svelte](https://svelte.dev/docs#template-syntax-slot-slot-key-value), a child component can values back to its parent using props(short for properties).
Example:
```html
<!-- Parent.svelte -->
<Child>
    <span slot="body" let:prop={item}>{ item.id } { item.name }</span>
</Child>
```
```html
<!-- Child.svelte -->
<div>
    <slot name="body" prop={item}></slot>
</div>
```
The svelte approach requires a lot less boiler plate and does not introduce any new directive to support this capability.

### Reactjs
Reactjs took a imperative approach to supporting sharing values between components. It does so by using [`render props`](https://reactjs.org/docs/render-props.html). Essentially the child component exposes a property called "render" which will accept a function as value. During its rendering, the child will then invoke the provided function by passing its own values. The parent customizes the output using a function set as the value of the `render` property.
Example:
```jsx
class Child extends React.Component {
  constructor(props) {
    super(props);
    this.item = { id: 123, name: 'react' };
  }

  render() {
    return (
      <div >
        {this.props.render(this.state)}
      </div>
    );
  }
}

class Parent extends React.Component {
  render() {
    return (
      <div>
        <Child render={item => (
          <span>{ item.id } { item.name }</span>
        )}/>
      </div>
    );
  }
}
```

### Lit
[Lit has a proposal](https://github.com/lit/lit/issues/1478) to expose similar capabilities in very similar to React. The issue was first filed in Dec 2020 and is still open.

### Web Components Community Group
Supporting scoped slots using standard web components technology is challenging. There is [an open issue](https://github.com/webcomponents-cg/community-protocols/issues/8) in the wcgg to support. Here are some quotes from that issue which talk about the challenge.
> I'm hesitant about implementing this as a protocol. Vue.js scoped slots are a framework-specific concept (please don't be confused by its usage of <slot> in the template syntax) that mixes two things: DOM composition and data flow.<br/>
> Standard based <slot> elements are only about DOM composition. Passing data to slotted components could be done using slotchange event. But in general it's up to library authors to decide how to handle the data flow on their side.

In summary, Vue and Svelte both use a declarative approach to support scoped slot. React, Lit and WCGG have taken/propose to take the imperative approach. Vue.js and Svelte are able to achieve DOM Composition and data access using `<slot>` elements easily because they rely on virtual DOM. Reach, List and WCGG's approach of using public props is more compatible with the Web Components model. It is standards based and less proprieraty.

### Lightning Web Components
Developers are able to mimic scoped slots using template passing. To achieve this, a parent component is able to pass a compiled template to a child component. The child component will use the compiled template in its own `render` method or a stub component. All the data required for rendering the template will be setup as the stub component's props. `lightning-datatable` uses this approach to allow customization of cells. [Here](https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.data_table_custom_types) is more information about customizing lightning-datatable.
Key differences between this proposal and template passing:
1. This technique does not use `slot` elements to render the passed template.
2. The templates are stored as separate `.html` files in the parent component. They are not part of the parent's own template. Thus, the nodes of the passed template are not queriable from the parent's template using `this.template.querySelector`. Rather, the parent has to query the child and then find the elements in the child's tree dependening on the render mode of the child.
    - An implied restriction is that the passed template only has access to bindings of the stub component and not of the parent component.
    - The parent cannot attach its own event handles in the passed template.
3. Customizing the template is achieved by parent and child exchanging metadata to interpret the data.

## Basic example

```html
// x/parent.html
<template> <!-- parent need not be light DOM -->
    <x-list items={items} lwc:slot-data="item">
        <span>{item.id} - {item.name}</span>
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
The `<x-parent>` component is slotting `<span>{item.name}</span>` into `<x-list>` but the `item` is supplied by `<x-list>`, the child component. It's essentially like `<x-parent>` is slotting a function that actually returns the slotted content.

This allows `<x-parent>` to control how each item renders while leaving the looping/repeating logic to `<x-list>`. `<x-list>` is now free to create as many slots as it wants while using the same "slotting function" passed from the parent.
This allows `<x-list>` to be a virtual list or an infinite list.

The final HTML will look like this:
```html
<x-parent>
    #shadow-root
    |    <x-list>
    |        <ul>
    |            <li>
    |                <span>1 - One</span>
    |            </li>
    |        </ul>
    |        <ul>
    |            <li>
    |                <span>2 - Two</span>
    |            </li>
    |        </ul>
    |    </x-list>
</x-parent>
```
It's important to note that for this to happen, slots need to be lazily rendered.

Scoped Slots are allowed only in Light DOM components for the following reasons:
* Native shadow DOM slots: the slotted content is eagerly rendered regardless of there being a slot to “accept” the content. The component slotting the content is in charge of appending and removing the content from the DOM.
* Scoped slots: the slotted content is lazily rendered; the slotted content will only be rendered if there is a slot to “accept” the content. The component with the <slot> elements is the one in charge of rendering the slotted content passed by the parent component.
However, some preliminary work has been done for adding support for scoped slots in Shadow DOM component. For more details refer to the [appendix](#appendix-1-scoped-slots-in-shadow-dom).

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

### Access to bindings
Scoped slots have access to component bindings and scope bindings
#### Component Bindings
```html
<template>
    {title}
    <x-list items={items} lwc:slot-data="item">
        <div>{label}</div> <!-- label is a component binding and will be repeated in every row of the list-->
        <span>{item.id} - {item.name}</span>
    </x-list>
</template>
```

#### Nested Scoped Slots
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

### Ownership and styling implication
Standard slots and scoped slots are identical with respect to ownership. The slotted content is owned by the parent component. This had direct implications in terms of styling. If the parent component has a scoped stylesheet associated with it, the styles also apply to the scoped slot content.

### Validations
Validation of scoped slot usage between parent and component is handled at runtime.
* **Missing default slot**: If a parent component slots content in to the child component's default slot and the child does not have a default slot, the content will be ignored.(TODO: Why don't we log an error in this case today for standard slots?)
* **Missing named slot**: If a parent component tries to set an unknown named slot of child component, the content will be ignored and an error will be logged in dev mode.

_The above behaviors are consistent with how the engine handles regular slot content._

* **Parent uses scoped slot but child has regular slot**: In this case the content will be ignored and an error will be logged in dev mode.
* **Parent uses regular slot but child has scoped slot**: This is valid usage. The content provided by the parent will be slotted into the child's scoped slots.

## New capabilities required in LWC
This section new capabilities that will need to be supported in LWC to support this proposal.

### Allowing `slot` attribute on `<template>` tag
To set a named slot in an LWC, a parent component specifies the `slot` attribute on the slotted content. This proposal will require setting named slots on a `<template>` tag. 
This capability adds a new concern of deviating from standards and will be challenging to implement in shadow DOM. As per Web Components  standards, `slot` attribute on the slotted content is used to set a named slot. However, `template` does not support `slot` attribute as [per standard](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template#attributes). So to support this syntax in shadow DOM, a container element will have to demark the slot content and the `slot` attribute will be set on it.

Further, the same support of using `<template slot="someslotname">` must also be extended to standard slots.

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
Scoped Slots in Shadow DOM aren't supported by any other web component framework. The reason boils down to slots in native
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

The scoped content fragment is transferred to the <x-list> component and the <x-list> component renders the items in its shadow tree. Example (https://gist.github.com/pmdartus/cd490a82617e025a2e73f79f7f9888b3#file-shadow-html)

```html
<x-list>
   #shadow-root
   |  <div>1 - Buy groceries</div>
   |  <div>2 - Clean up my desk</div>
   |  <div>3 - Do the laundry</div>
</x-list>
```

This approach is quite similar to the one explained above with the main advantage that the template contains the scoped slot definition. This approach allows the scoped slot to reference properties on its original component.

However, this approach introduces new issues. Since the scoped slot is rendered in the <x-list> shadow tree, the <x-list> shadow tree styles are applied to the rendered content. This approach also breaks shadow DOM encapsulation because event handlers attached to the scoped slot will be invoked on elements that are rendered in another shadow tree.

* *Approach #2: One named slot for each scoped slot item.*

Instead of rendering the scoped slot content in the <x-list> shadow tree, the scoped slot is rendered in its light DOM
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

This approach solves the 2 main drawbacks of approach #1 as the scoped slot content is rendered in the parent component shadow tree. The main drawback of this approach is that it will make the diffing algo more complex because it requires coordination between the parent and the child component.
A drawback of this approach is that it is not interoperable with other UI frameworks. The parent/child component protocol is proprietary to LWC.

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
to do the content allocation which makes it unusable for SSR.
