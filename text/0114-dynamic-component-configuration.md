{\rtf1\ansi\ansicpg1252\cocoartf2513
\cocoatextscaling0\cocoaplatform0{\fonttbl\f0\fswiss\fcharset0 Helvetica;}
{\colortbl;\red255\green255\blue255;}
{\*\expandedcolortbl;;}
\margl1440\margr1440\vieww10800\viewh8400\viewkind0
\pard\tx720\tx1440\tx2160\tx2880\tx3600\tx4320\tx5040\tx5760\tx6480\tx7200\tx7920\tx8640\pardirnatural\partightenfactor0

\f0\fs24 \cf0 # RFC: Dynamic Component Configuration\
\
## Summary\
\
This RFC introduces a set of principles and invariants required to dynamically configure a lazy loaded component\
\
## Terminology \
\
* *Lazy Loading*: resolve component definition / code at runtime \'97 definition may need to be looked up from the backend\
* *Dynamic Instantiation*: create a component instance when the underlying component constructor isn't known until runtime\
* *Variable Dynamic import*: a dynamic import that uses JavaScript to create the component string parameter to pass into the `import` function \'97 non-statically analyzable when the constructor is not a string literal\
* *Dynamic Configuration*: The process of setting a dynamically instantiated component\'92s constructor, properties, attributes, or event handlers at runtime\
\
## Interface Example\
\
```\
<template>\
   <c-lazy-cmp lwc:bind=\{configurations\}></c-lazy-cmp>\
</template>\
```\
\
```\
import \{ LightningElement, api \} from "lwc";\
export default class LazyCmp extends LightningElement \{\
  customCtor;\
  \
  @api \
  attributes;\
  \
  @api \
  moduleName;  \
\
  connectedCallback() \{\
    this.loadCtor();\
  \}\
\
  async loadCtor() \{\
    const ctor = await import(this.moduleName);\
    this.customCtor = ctor.default;\
  \}\
  \
  get configurations() \{\
    return \{\
        ctor: this.customCtor,\
        attributes: this.attributes,\
        properties: this.properties,\
        eventHandlers: this.handlers,\
    \}\
  \}\
\}\
```\
\
## Working Example\
\
This example shows a watered-down example of rendering a dynamic analytics dashboard. The component has some dashboard data that it will assign to the chart items it renders to the screen. \
\
* *Ctor*: each chart type is constructed via its own dedicated chart renderer LWC.\
* *Properties*: The chart items often require different input property shapes. For instance, some may not support dimensions (categories). Some take a color palette (i.e. a mapping of category to color) and others determine color based on the number they display (breakpoints).\
* *Attributes*: For application purposes, each chart is assigned a unique DOM element id. Additionally, we want to render some charts smaller than others so we assign different css class values based on chart type.\
* *Events*: Some charts allow users to apply dashboard-wide filters when clicks occur on data points and some allow for data sorting. If we detect that a chart supports either, we bind the dashboard\'92s handlers accordingly.\
\
```\
<template>\
    <template for:each=\{chartItems\} for:item="chartItem">\
       <c-lazy-cmp lwc:bind=\{chartItem\}></c-lazy-cmp>\
   </template>\
</template>\
```\
\
```\
import \{ LightningElement, api \} from "lwc";\
import \{  \} from "lds/...";\
\
export default class LazyCmp extends LightningElement \{\
\
   @api dashboardId;\
   @api dashboardData;\
   @api colorPalette;\
   @api defaultKpiBreaks;\
\
   chartItems = [];\
\
   @wire(myAdapter, \{ id: 'someDashboardId' \})\
   chartDefinitions(\{ data, error \}) \{\
       chartItems = data.charts.map((chart) => \{\
           let ctor;\
           let props = \{\};\
           let attributes = \{\};\
           let eventHandlers = \{\};\
           \
           // Populate 'ctor'\
           // For simplicity assume they've been loaded async elsewhere\
           ctor = chart.ctor;\
           \
           // Populate 'attributes'\
           // For example, change style based on some config data and pass\
           // in a unique id\
           attributes.style = \{\
               classname: chart.type === 'donut' ? 'chart-small' : 'chart-medium'\
           \};\
           attributes.id = `$\{dashboardId\}-$\{chart.id\}`;\
           \
           // Populate 'props'\
           // Set the dimensions into charts that support them.\
           // For example, a KPI doesn't have a dimensions prop\
           // and would throw if you tried to supply one:\
           if (chart.supportsDimensions) \{\
              props.dimensions = dashboardData.dimensions;\
           \}\
           // Some chart items, like a dimension filter card,\
           // might not support measures as inputs:\
           if (chart.supportsMeasures) \{\
              props.measures = dashboardData.measures\
           \}\
           \
           // Some charts support colors in different ways: \
           if (chart.type === 'kpi' || chart.type === 'gauge') \{\
              props.breakpoints = chart.customBreaks || this.defaultKpiBreaks;\
           \} else if (chart.supportsColorPalette) \{\
              props.colorPalette = this.colorPalette;\
           \}\
           \
           // Populate 'events'\
           if (chart.canFilter) \{\
                eventHandlers.onfilter = this.handleFilter;\
           \}\
           if (chart.canSort) \{\
                eventHandlers.onsort = this.handleSort;\
           \}\
           \
           return \{\
               ctor,\
               attributes,\
               properties: props,\
               events: eventHandlers\
           \}\
       \});\
   \}\
\}\
```\
\
## Motivation\
\
Before this proposal, dynamic component creation was done using the `lwc:dynamic` directive. This directive required a component author to know component configuration details like the property and event names at design time which breaks down for use cases that need true dynamic component creation. For instance, in the case of property values or event handlers one would have to keep a reference to the element and set the information retrieved at runtime manually.\
\
```\
element;\
\
@api \
moduleName;\
\
connectedCallback() \{\
   const ctor = await import(this.moduleName);\
   this.element = createComponent('c-lazy-cmp', \{ is: ctor \});\
   // element must be manually inserted into the dom\
   this.template.querySelector('div').appendChild(this.element)\
\}\
\
@api \
set properties(newProperties) \{\
   this.props = newProperties;\
   Object.assign(this.element, this.props);\
\}\
\
get attributes() \{ return this.props; \}\
```\
\
A new tool must be introduced that will allow component authors to configure their components with information retrieved at runtime without relying on maintaining element references or manually manipulating the DOM.\
\
## Proposal\
\
The proposal involves adding a new directive that captures the relevant information to configure a web component at runtime. The assumption is that this directive will be used in cases where the component module is unknown at runtime (therefore used alongside dynamic imports) and zero or more of the following are unknown at runtime: properties, attributes, or event handlers.\
\
The directive `lwc:bind` can retrieve component configuration data via a JavaScript object with a pre-defined shape. Constructor is the only required property in the configuration data.\
\
## General Invariants\
\
* It must not break the reactivity model or the general framework lifecycle\
    * After the component is loaded, it should function within the lifecycle of the parent component\
    * Changes to the configuration object must be reflected in the component (ie changing the properties will cause component properties to be set again)\
* Changing the constructor on a parent component will also trigger a cascade re-render of any child components \
    * Ex: changing the \'93dashboard\'94 constructor will re-render any child \'93chart\'94 components\
* Will work regardless of whether one uses dynamic imports to retrieve the component constructor\
    * How a component author retrieves the configurations is unrelated to the proposal\
* In cases of invalid component ctor, same behavior as existing `lwc:dynamic` feature\
* This should reuse the same mechanism as `lwc:dynamic` and must not require additional development time to add support in environments that currently support `lwc:dynamic`.\
\
## Drawbacks\
\
Identifying the dependencies for static analysis could be difficult especially if the configurations are not derived from @api properties or statically analyzable strings. Additionally, this proposal would enable true dynamic component creation which has been abused in the past - proper guardrails would need to be implemented to prevent improper use of this pattern.\
\
## Adoption Strategy\
\
This feature will be available in Salesforce Lightning Platform and LWR to certain teams depending on their use cases\
\
## Unresolved Questions\
\
* Naming for directive\
* What can be statically analyzable when a component uses this feature?\
* Implementation details\
* Scope\
    * Should we start with only constructor & properties or is it trivial to include attributes and event handlers?\
\
\
\
}