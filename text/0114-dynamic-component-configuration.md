# RFC: Dynamic Component Configuration

## Summary

This RFC introduces a set of principles and invariants required to dynamically configure a lazy loaded component

## Terminology


* *Lazy Loading*: resolve component definition/code at runtime - definition may need to be looked up from the backend

* *Dynamic Instantiation*: create a component instance when the underlying component constructor isn't known until runtime

* *Variable Dynamic import*: a dynamic import that uses JavaScript to create the component string parameter to pass into the `import` function (non-statically analyzable when the constructor is not a string literal)

* *Dynamic Configuration*: The process of setting a dynamically instantiated component's constructor, properties, attributes, or event handlers at runtime

## Interface Example

```html
<template>
   <c-my-cmp lwc:bind={configurations}></c-my-cmp>
</template>
```
```js
import { LightningElement, api } from "lwc";

export default class LazyCmp extends LightningElement {
	@api attributes;
	@api properties;
	@api handlers;

	get configurations() {
		return {
			attributes: this.attributes,
			properties: this.properties,
			eventHandlers: this.handlers,
		}
	}
}
```
## Working Example

This example shows how we render dynamic analytics dashboard with one type of chart. The component has dashboard data that it will assign to the chart item and it renders to the screen.

* *Properties*: Different chart type often requires different input property shapes. We set chart item's properties from dashboard data.

* *Attributes*: For application purposes, we assign different css class values based on chart type.

* *Events*: Different chart types support different operations on them. If we detect that a chart supports dashboard-wide filters, we bind the dashboard's handlers to it.

```html
<template>
	<c-lazy-cmp lwc:bind={chartItem} lwc:dynamic={ctor}></c-lazy-cmp>
</template>
```
```js
import { LightningElement, api } from "lwc";

export default class LazyCmp extends LightningElement {
	@api dashboardId;
	@api dashboardData;

	get chartItem() {
		return this.chartItem;
	}

	@wire(myAdapter, { id: 'someDashboardId' }) {
		chartDefinitions({ data, error }) {
			let chart = data.chart;

			// Populate 'ctor'
			// For simplicity assume they've been loaded async elsewhere
			this.ctor = chart.ctor;
			let attributes = {};
			let props = {};
			let eventHandlers = {};
			
			// Populate 'attributes', change style based on some config data
			attributes.style = {
				classname: chart.type === 'donut' ? 'chart-small' : 'chart-medium'
			};

			// Populate 'props', set the dimensions into charts that support them.
			if (chart.supportsDimensions) {
				props.dimensions = dashboardData.dimensions;
			}

			// Populate 'events', bind the event handler if it supports that
			if (chart.canFilter) {
				eventHandlers.onfilter = this.handleFilter;
			}

			chartItem =  {
				attributes,
				properties: props,
				events: eventHandlers
			}
		}
	}
}
```
## Motivation

Before this proposal, dynamic component creation was done using the `lwc:dynamic` directive. This directive required a component author to know component configuration details like the properties and event names at design time, which breaks down for use cases that need true dynamic component creation. For instance, in the case of property values or event handlers one would have to keep a reference to the element and set the information retrieved at runtime manually.

Also, when connetedCallback called on child and the component has been inserted into DOM, but its properties/attributes/eventHandlers have not been initiated, we also need to wait for another cycle to get them rendered with the correct properties. This breaks the framework's life cycle.

```js
export default class LazyCmp extends LightningElement {
	element;

	@api moduleName;

	connectedCallback() {
		const ctor = await import(this.moduleName);
		this.element = createComponent('c-lazy-cmp', { is: ctor.default });
		// element must be manually inserted into the dom
		this.template.querySelector('div').appendChild(this.element)
	}

	@api
	set properties(newProperties) {
		this.props = newProperties;
		Object.assign(this.element, this.props);
	}

	get properties() { return this.props; }
}

```
A new mechanism must be introduced that will allow component authors to configure their components with information retrieved at runtime without relying on maintaining element references or manually manipulating the DOM.

## How other frameworks are approaching this issue

### Vue.js

In the wrapper component(currentComponent) level, they do "bind" to props, and they return the corresponding props for different child level components(e.g. myComponent). In this way, they don't need to rerender to get the component with the correct props. But different child components' props are exposed at wrapper component level.

example:

```html
<template>
	<c-cmp :is="currentComponent" v-bind="currentProperties"></c-cmp>
</template>
```
```js
data: function () {
  return {
    currentComponent: 'myComponent',
  }
},
computed: {
  currentProperties: function() {
    if (this.currentComponent === 'myComponent') {
      return { foo: 'bar' }
    }
  }
}
```

## Our Proposal

The proposal involves adding a new directive that captures the relevant information to configure a web component at runtime. The assumption is that this directive will be used in cases where the component constructor is unknown until runtime and zero or more of the following are unknown until runtime: properties, attributes, or event handlers.

The directive `lwc:bind` can retrieve component configuration data via a JavaScript object with a pre-defined shape. Constructor is the only required property in the configuration data.

## Design Decisions Based On Our Proposal

* Changes to the configuration object must be reflected in the component (ie changing the properties will cause component properties to be set again)

* Changing the constructor on a parent component will also trigger unmounting the existing child components and creating new child components.

* Ex: changing the "dashboard" constructor will create new child "chart" components

* Will work regardless of whether one uses dynamic imports to retrieve the component constructor

* How a component author retrieves the configurations is unrelated to the proposal

* In cases of invalid component ctor, same behavior as existing `lwc:dynamic` feature

* This should reuse the same mechanism as `lwc:dynamic` and must not require additional development time to add support in environments that currently support `lwc:dynamic`.

* `lwc:bind` directive can be used with other directives (including `lwc:dynamic`, `if:true|false`, `key`), attributes and event handlers. `lwc:bind` should not be used with array directives (including `for:each`, `for:item`, `for:index`) or `lwc:dom`

* When the lwc:bind contains an attribute that is already defined via the template, we overwrite that attribute with the bind value. For example, <x-foo lwc:bind={config} class="error"></x-foo> with config = { attributes: { class: 'success' } }, we will overwrite default class 'success' with 'error', which we get from "bind".

* Updating properties, attributes and eventHandler by updating the bind configurations.

* Semantic of the following attribute values: when it's null or undefined, we will use the default attribute value. When that attribute value is a boolean, then we treat false as the default value.

## General Invariants

* It must not break the reactivity model or the general framework lifecycle

* After the component is loaded, it should function within the lifecycle of the parent component

## Drawbacks

Identifying the dependencies for static analysis could be difficult especially if the configurations are not derived from @api properties or statically analyzable strings. Additionally, this proposal would enable true dynamic component creation which has been abused in the past - proper guardrails would need to be implemented to prevent improper use of this pattern.

## Adoption Strategy

This feature will be available in Salesforce Lightning Platform and LWR to certain teams depending on their use cases

## Unresolved Questions

* Naming for directive

* What can be statically analyzable when a component uses this feature?

* Implementation details

* Scope

* Should we start with only constructor & properties or is it trivial to include attributes and event handlers?

## Appendix for more complexed example

This example shows a watered-down example of rendering a dynamic analytics dashboard. The component has some dashboard data that it will assign to the chart items it renders to the screen.

* *Ctor*: each chart type is constructed via its own dedicated chart renderer LWC.

* *Properties*: The chart items often require different input property shapes. For instance, some may not support dimensions (categories). Some take a color palette (i.e. a mapping of category to color) and others determine color based on the number they display (breakpoints).

* *Attributes*: For application purposes, each chart is assigned a unique DOM element id. Additionally, we want to render some charts smaller than others so we assign different css class values based on chart type.

* *Events*: Some charts allow users to apply dashboard-wide filters when clicks occur on data points and some allow for data sorting. If we detect that a chart supports either, we bind the dashboard's handlers accordingly.

```html
<template>
   <template for:each={chartItems} for:item="chartItem">
      <c-lazy-cmp lwc:bind={chartItem} lwc:dynamic={ctor}></c-lazy-cmp>
   </template>
</template>
```
```js
import { LightningElement, api } from "lwc";

export default class LazyCmp extends LightningElement {
	@api dashboardId;
	@api dashboardData;
	@api colorPalette;
	@api defaultKpiBreaks;

	chartItems = [];

	@wire(myAdapter, { id: 'someDashboardId' }) {
		chartDefinitions({ data, error }) {
			chartItems = data.charts.map((chart) => {
				// Populate 'ctor'
				// For simplicity assume they've been loaded async elsewhere
				this.ctor = chart.ctor;
				let props = {};
				let attributes = {};
				let eventHandlers = {};

				// Populate 'attributes'
				// For example, change style based on some config data and pass
				// in a unique id
				attributes.style = {
					classname: chart.type === 'donut' ? 'chart-small' : 'chart-medium'
				};
				attributes.id = `${dashboardId}-${chart.id}`;

				// Populate 'props'
				// Set the dimensions into charts that support them.
				// For example, a KPI doesn't have a dimensions prop
				// and would throw if you tried to supply one:
				if (chart.supportsDimensions) {
					props.dimensions = dashboardData.dimensions;
				}
				// Some chart items, like a dimension filter card,
				// might not support measures as inputs:
				if (chart.supportsMeasures) {
					props.measures = dashboardData.measures
				}
				// Some charts support colors in different ways:
				if (chart.type === 'kpi' || chart.type === 'gauge') {
					props.breakpoints = chart.customBreaks || this.defaultKpiBreaks;
				} else if (chart.supportsColorPalette) {
					props.colorPalette = this.colorPalette;
				}
				// Populate 'events'
				if (chart.canFilter) {
					eventHandlers.onfilter = this.handleFilter;
				}
				if (chart.canSort) {
					eventHandlers.onsort = this.handleSort;
				}
				return {
					attributes,
					properties: props,
					events: eventHandlers
				}
			});
		}
	}
}
```