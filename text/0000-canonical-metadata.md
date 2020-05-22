---
title: Canonical specification of metadata for LWC modules
status: DRAFT
created_at: 2020-05-07
updated_at: 2020-05-19
rfc: 
champion: Aliaksandr Papko (@apapko) | Ravi Jayaramappa (@ravijayaramappa)
implementation: 
---

# Canonical specification of metadata for LWC modules

Table of Content:
* [Summary](#summary)
* [Goals](#goals)
* [Use Cases](#use-cases)
* [Current State of Art](#current-state-of-art)
* [Detailed design](#detailed-design)
* [References](#reference)

# Summary
> The [dictionary definition](https://www.merriam-webster.com/dictionary/canonical%20form) of **Canonical form** is `the simplest form of something`. 

This RFC aims to document the metadata collected by statically analyzing the source code of Lightning Web Component(LWC) modules. It documents the standardized shape of metadata.

# Goals

- Completeness
- Eliminate redundancy: There should be only one way to get a desired information about a module.
- Consistency in property names
- Flexibility: Allow for new data to be collected in the future.

# Use Cases
## Referential Integrity
Referential integrity is how the platform allows a given resource to be safely updated and deleted without adverse cascading effects. For example, a bundle's metadata will include references to external modules imported, custom elements referenced in template and the properties set on such elements. The reference information can be indexed and used to validate if a component's properties can be safely renamed/removed, if a component can be safely deleted or needs to follow a deprecation process.

## Editor support
Metadata about components can be used to augment the standard developer experience in code editors with code completion, attribute name & type validation, peek definition, etc.

## Documentation
LWC components Metadata can be used to automatically generate documentation about components. Standard javascript documentation formats like JSDoc provided in component can also be gathered as metadata.

## Dependency analysis and prefetching
Components depend on static resources such as images, javascript files, css files and component definitions to name a few. For offline usage, knowing these dependencies at compile time allows the dependencies to be prefetched and avoid a network call.

## Code generation
Finally, metadata can facilitate low code development environment by allowing developer to declaratively define component. This metadata can be used to generate component stubs and the developer can add the specific business logic into the stubs thus reducing the amount of manually authored boiler plate code.
Metadata can also be transformed to generate components in various eco-systems like LWC, native web components etc. This would require adding an additional transformation of the metadata to another format like the custom-elements-json in the case of web components.

# Current State of Art
## Web Components community
There is no well established standard or format for declaratively defining a web component. [custom-elements-json](https://github.com/webcomponents/custom-elements-json) is making an attempt to define a standard. It is in a very early stage. However there is a ecosystem developing around this standard. 
[web-component-analyzer](https://www.npmjs.com/package/web-component-analyzer) can analyze web components written in vanilla javascript and other popular web component libraries. It also supports custom-elements-json as an output format.

## LWC Platform
Currently, LWC platform gathers metadata as part of the compilation process. This metadata shape and implementation is currently private. This RFC is an attempt to make the metadata shape public as a first step. This [document](https://salesforce.quip.com/DdncANrJA0ko)<sup>*</sup> captures the information collected and the metadata shape currently gathered. Here is a summary of problems with the current implementation:
* Replicated type system in java and javascript that is maintained manually
* Dependency(reference) analysis is done outside of metadata gathering
* Change management to accomodate new metadata is cumbersome because the implementation is spread across two repos

# Detailed Design

## Format
Metadata shape of the various objects we wish to analyse will be represented using in [JSON Schema](https://json-schema.org/) format. This is also augmented with a TypeScript definition of the shape.

[JSON](https://tools.ietf.org/html/rfc7159) is the IETF approved standard for representing objects. [JSON Schema](https://json-schema.org/) is on the path to become an IETF approved standard for describing JSON.

This format was also influenced by standardization choices made with in Salesforce UI platform.

## Entities that will be analyzed
These are the entities that will be analyzed and metadata gathered about:
* HTML Template file
    * Custom element references
        * Tag name
        * Attributes
        * Slot content - default and named slots
    * Static resource
        * Image urls
        * Javascript resource
        * CSS resource
    * Dynamic components
    * Nodes with special directives
        * for:each
        * if
        * iterator
        * key
        * lwc:dom="manual"
    * Event handlers
    * Slots
* Javascript file
    * Component classes
        * Inheritance
        * Public & private properties
        * Decorators
        * getters/setters
        * Public methods
    * Imports
        * Builtin modules
        * External modules
        * Salesforce scoped modules
        * Other modules(components and libraries)
        * Relative imports
        * Dynamic imports
    * Exports
        * Default exports
        * Named exports
        * Aggregating modules
    * JSDoc
        * Class
        * Properties
        * Public methods
    * Programmatically generated events
        * type
        * options: composed, bubbles
    * Programmatically fetched static resources
        * `<script>`
        * `<img>`
        * `<link>`
* CSS file
    * Tokens
    * Static resources

## Metadata shape

### TypeScript

#### Bundle metadata shape
```ts
interface BundleMetadata {
    version: string;
    name?: string;
    namespace?: string;
    // Define the type of module that is represented by the given metadata object.
    moduleType?: "HTML" | "CSS" | "Javascript";

    /**
     * For a component bundle, this property is a reference to the main component class in the bundle.
     * This class must be the default export of the javascript file. The file must be named the same
     * as the bundle
     */
    defaultComponentClass?: Class[];

    // Information about HTML template files in the bundle.
    templates?: HTMLTemplateFile[];

    // Information about javascript files in the bundle.
    scripts?: ScriptFile[];

    // Information about css files in the bundle.
    css?: CSSFile[]
}
```
Other alternatives:
- Indexed by file type and file name
```ts
interface BundleMetadataOption2 {
    version: string;
    name?: string;
    namespace?: string;
    moduleType?: "HTML" | "CSS" | "Javascript";

    defaultComponentClass?: ComponentClass[];

    templates?: { [key: string]: HTMLTemplateFile[]};

    scripts?: { [key: string]: ScriptFile[]};

    css?: { [key: string]: CSSFile[]};
}
```

#### HTML Template file metadata shape
```ts
// Root metadata object for an HTML template file
interface HTMLTemplateFile {
    fileType: 'html';
    fileName: string;
    // List of unique components referenced in the template
    componentReferences?: ComponentReference[];
    // Static resources referenced in a template. For example: 'src' attribute value of an <img>, <source> tag.
    staticResources?: StaticResourceReference[];
    // Template in AST format starting with the root <template> node. Note: this is a partial AST
    astRoot?: Node;
}

// Represents a component referenced in the template.
// Use the 'tagName' property for kebab case and 'name' property for the reference in canonical form
interface ComponentReference extends ModuleReference {
    // Component reference always has a namespace
    namespace: string;
    // No moduleIdentifier for a component
    moduleIdentifier?: never;
    // Components are always imported from an external bundle
    type: 'external';
    tagName: string;
}

// Information about a node in the html template
interface Node {
    tagName: string;
    location: SourceLocation;
    // Any properties set in the template, will include html attributes, public properties
    attributes?: ElementAttributeReference[];
    // Event listeners attached declaratively in the template with a 'on' prefix, eg: 'onclick'
    eventListeners?: DeclarativeEventListener[];
    children?: Node[];
}

// Information about a custom element referenced in a HTML template.
interface CustomElementNode extends Node {
    // Information about slots set by parent
    slots?: SlotReference[];
}

/**
 * Information about default and named slots being set by the parent component. The parent can
 * specify a 'slot' attribute in the slotted content in the case of named slot. Slot content that
 * do not have a 'slot' attribute will be passed as default slot content. The 'name' property in
 * the SlotReference object will be 'default' for such a case.
 */
interface SlotReference {
    name: string;
}

// Information about an element referenced in the template with a 'lwc:dynamic' attribute.
interface DynamicCustomElementNode extends CustomElementNode {
    ctor: ComputedValue;
}

// Information about for:each directive usage to render a list in a html template.
interface TemplateForEachDirective extends Node {
    tagName: 'template';
    directive: 'for:each';
    items: ComputedValue;
    itemName?: string;
    indexName?: string;
    key?: string;
}

// Information about if:true or if:false directive usage to perform conditional rendering in a html template.
interface TemplateIfDirective extends Node {
    tagName: 'template';
    directive: 'if:true' | 'if:false';
    value: ComputedValue | string;
}

// Information about iterator directive usage in a html template.
interface TemplateIteratorDirective extends Node {
    tagName: 'template';
    directive: 'iterator:it';
    items: ComputedValue;
    key?: string;
}

// Information about lwc:dom attribute usage in a html template to perform programmatic manipulation of dom tree.
interface TemplateLwcDomDirective extends Node {
    directive: 'lwc:dom';
    directiveValue: 'manual'; // enum of all allowed values for lwc:dom attribute
}

// Information about an event handled declaratively in the template.
interface DeclarativeEventListener {
    type: string;
    value: ComputedValue;
    // Location includes the event name starting with 'on' and the property assignment ending with '}'
    location: SourceLocation;
}
```
*Examples:*
Samples of the metadata for HTML template files can be viewed [here](https://github.com/salesforce/lwc-metadata/pull/1)

#### Script file metadata shape
```ts
// Root Metadata object for a file authored in javascript(or its variants like TypeScript) in an LWC bundle.
interface ScriptFile {
    fileType: 'js';
    fileName: string;
    // List of all class metadata, covers only classes created using the ES6 'class' syntax
    classes?: Class[];
    // Static import statements in a javascript module.
    imports?: Import[];
    // Dynamic import statements in a javascript module, additionally information about hints if provided.
    dynamicImports?: DynamicImport[];
    // Export statements in a module including reexporting bindings from another module.
    exports: [] | [Export] | [Export, ReExport];
    // DOM Event objects instantiated in a module. Includes Events and CustomEvents.
    domEvents?: DOMEvent[];
    // Programmatically added event listeners in a module
    eventListeners?: ProgrammaticEventListener[];
    /**
     * Static resources referenced in a module. Includes any static urls referenced in javascript and
     * resources loaded using the lightning platform resource loader, urls fetched via the salesforce
     * scoped url resolvers.
     */
    staticResources?: StaticResourceReference[];
}

// Information about a class
interface Class {
    name?: string;
    // A class is considered a component class if it extends
    // 'LightningElement' class available in the standard 'lwc' module.
    isComponentClass: boolean;
    // A parent can be another class declared locally, imported identifier from an external module
    // or an expression(think Mixin)
    extends: ParentClass;
    properties?: ClassProperty[];
    methods?: ClassMethod[];
    location: SourceLocation;
}

interface ClassMember {
    accessType: 'public' | 'private' | 'static';
    location: SourceLocation;
    name: string;
}

// Information about a class property of a component class.
interface ClassProperty extends ClassMember {
    // True, if the property is an instance property
    dataProperty?: boolean;
    // True, if the property has a getter
    hasGetter?: boolean;
    // True, if the property has a setter
    hasSetter?: boolean;
    decorators?: [TrackDecorator | WireDecorator | ApiDecorator];
    // If the initial value is statically analyzable. If not analyzable, property will be omitted.
    initialValue?: number | string | boolean | null | object | undefined;
}

// Information about a class method of a component class.
interface ClassMethod extends ClassMember {
    decorators?: [ApiDecorator | TrackDecorator | WireDecorator];
    parameterNames?: (DefaultParameters | RestParameters)[];
    returnType: string;
    // If the return value is statically analyzable. If not analyzable, property will be omitted.
    returnValue?: number | string | boolean | null | object;
}

interface Decorator {
    type: string;
    location: SourceLocation;
}
// Information about the usage of an '@api' decorator to designate a class member as public.
interface ApiDecorator extends Decorator {
    type: 'api';
}

// Information about an @track decorator usage.
interface TrackDecorator extends Decorator {
    type: 'track';
}

// Information about @wire decorator usage includes information about the data source.
interface WireDecorator extends Decorator {
    adapterId: string;
    adapterModule?: ModuleReference;
    adapterConfig?: object;
}

// Information about a static import statement in a module.
interface Import {
    importType?: ('DefaultBinding' | 'NamedImports' | 'NamespacedImport')[];
    importsList?: string[] | string;
    moduleSpecifier: ModuleReference;
    location: SourceLocation;
}

// Information about a dynamic import statement in a module.
interface DynamicImport {
    moduleSpecifier: ModuleReference;
    moduleNameType: 'string' | 'unresolved';
    location: SourceLocation;
    hints?: DynamicImportHint[];
}
interface DynamicImportHint {
    // Hint value with leading and trailing spaces of hint phrase trimmed
    rawValue: string;
    key: string;
    value: string;
    location: SourceLocation;
}

// Information about a export statement in a module.
interface Export {
    exportsList?: {
        type: 'class' | 'function' | 'expression';
        name?: 'string';
    }[];
    default?: {
        type: 'class' | 'function' | 'expression';
    };
    location: SourceLocation;
}

// Information about an export statement used to reexport bindings imported from another module.
interface ReExport {
    exportsList: {
        name: string;
        type: 'class' | 'function' | 'expression';
    }[];
    moduleSpecifier: ModuleReference;
    location: SourceLocation;
}

// Information about a dom event instantiated in script.
interface DOMEvent {
    eventType: string;
    isCustomEvent?: boolean;
    options?: {
        bubbles?: boolean;
        composed?: boolean;
    };
}

// Information about an event handled declaratively in the template.
interface ProgrammaticEventListener {
    type: string;
    targetType: 'host' | 'shadowRoot' | 'Node';
    options?: {
        capture?: boolean;
    };
    location: SourceLocation;
}

// A parent component class including the 'LightningElement' class provided by the lwc module.
type ParentClass =
    | Class // Another class declared in the same file
    | Mixin
    // or a class imported from an external module
    | {
          name: string;
          source?: ModuleReference;
      };

// Information about a module imported into a module using the 'import' statement.
interface ModuleReference {
    name: string;
    namespace?: string;
    moduleIdentifier?:
        | 'apexClass'
        | 'apexMethod'
        | 'apexContinuation'
        | 'client'
        | 'community'
        | 'component'
        | 'contentAssetUrl'
        | 'customPermission'
        | 'dynamicComponent'
        | 'slds'
        | 'messageChannel'
        | 'i18n'
        | 'gate'
        | 'label'
        | 'metric'
        | 'module'
        | 'internal'
        | 'resourceUrl'
        | 'schema'
        | 'sobjectClass'
        | 'sobjectField'
        | 'user'
        | 'userPermission';
    // whether the module reference is the built in lwc module or the @salesforce scoped module or
    // a local file in the current bundle or an module imported from an external source(eg. a component)
    type: 'lwc' | '@salesforce' | 'internal' | 'external';
    location: SourceLocation;
}

// Information about default function parameters.
// Also look at parameters represented using rest syntax.
interface DefaultParameters {
    type: 'DefaultParameters';
    name: string;
    defaultValue?: number | string | boolean | null | object;
}

// Information about function parameters received using the rest(...) syntax.
interface RestParameters {
    type: 'RestParameters';
    name: string;
    startIndex: number;
    endIndex?: number;
}

interface Mixin {
    // The identifier of the mixin expression
    identifier: {
        name: string;
        source?: string; // If the mixin is imported from another module
    };
    arguments: string[]; // We can add more types, starting with class name for now.
    location: SourceLocation;
}
```
Examples:
_TODO_

#### CSS file metadata shape
```ts
// TODO APapko: 
// Add support for declared properties v/s consumed properties
// Add a field for dependencies(think imports in the case of slds v3)

// Root metadata object for a css file
interface CSSFile {
    fileType: 'css';
    fileName: string;
    customProperties?: CSSCustomProperty[];
    staticResources?: StaticResourceReference[];
}

// Information about css custom property.
interface CSSCustomProperty {
    name: string;
    fallbackValue?: CSSCustomPropertyFallback;
    location: SourceLocation;
}

// Information about css custom property fallback values.
type CSSCustomPropertyFallback = string | CSSCustomProperty;
```
Examples:
_TODO_

<details>
<summary>Click to view extended typescript definitions</summary>

```ts
// Information about an expression, surrounded by curly braces, in a HTML template.
interface ComputedValue {
    expression: string;
    expressionType: 'ClassMemberExpression' | 'IteratorExpression';
}

// Information about css custom property fallback values.
type CSSCustomPropertyFallback = string | CSSCustomProperty;

// Information about default function parameters.
// Also look at parameters represented using rest syntax.
interface DefaultParameters {
    type: 'DefaultParameters';
    name: string;
    defaultValue?: number | string | boolean | null | object;
}

// Information about an element's attribute set in the template.
interface ElementAttributeReference {
    name: string;
    value?: (null | string | boolean) | ComputedValue;
}

// Information about function parameters received using the rest(...) syntax.
interface RestParameters {
    type: 'RestParameters';
    name: string;
    startIndex: number;
    endIndex?: number;
}

// Object to represent the start and end position of a code block.
interface SourceLocation {
    fileName: string;
    startLine: number;
    startColumn: number;
    endLine: number;
    endColumn: number;
}

// Information about reference to a static resource in a module. The resource can be a url specified
// as a string literal or a reference to a computed value.
interface StaticResourceReference {
    // The type of static resource loaded, identifiable by the file extension.
    type: 'image' | 'css' | 'html' | 'js' | 'other';
    value: string | ComputedValue;
    location: SourceLocation;
}
```
</details>

### JSONSchema(outdated - Typescript definitions are up to date)
<details>
<summary>Click to view in JSONSchema format</summary>

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "description": "A representation of metadata gathered for a LWC bundle",
    "type": "object",
    "properties": {
        "name": { "type": "string" },
        "namespace": { "type": "string" },
        "moduleType": { 
            "description": "Define the type of module that is represented by the given metadata object.",
            "type": "string",
            "enum": [ "Component", "Library", "CssOnly" ]
        },
        "templates": { 
            "description": "Information about HTML template files in the bundle.",
            "type": "array",
            "items": { "$ref": "#/definitions/HTMLTemplateFile"}
        },
        "scripts": {
            "description": "Information about javascript files in the bundle.",
            "type": "array",
            "items": { "$ref": "#/definitions/ScriptFile"}
        },
        "defaultComponentClass": {
            "description": "For a component bundle, this property is a reference to the main component class in the bundle. This class must be the default export of the javascript file. The file must be named the same as the bundle",
            "type": "array",
            "items": { 
                "$ref": "#/definitions/ComponentClass"
            }
        },
        "css": {
            "description": "Information about css files in the bundle.",
            "type": "array",
            "items": {
                "$ref": "#/definitions/CSSFile"
            }
        }
    },
    "definitions": {
        "HTMLTemplateFile": {
            "description": "Information about an HTML template file in an LWC bundle.",
            "type": "object",
            "properties": {
                "fileType": { "const": "html" },
                "fileName": { "type": "string" },
                "children": {
                    "description": "Custom elements referenced in a template.",
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/CustomElementReference"
                    }
                },
                "staticResources": {
                    "description": "Static resources referenced in a template. For example: 'src' attribute value of an <img>, <source> tag.",
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/StaticResourceReference"
                    }
                },
                "dynamicChildren": {
                    "description": "Custom elements in a template that are dynamically(lazy) loaded.",
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/DynamicComponentReference"
                    }
                },
                "directives": {
                    "description": "Elements in a template that have special directives.",
                    "type": "object",
                    "properties": {
                        "forEach": {
                            "type": "array", 
                            "items": { "$ref" : "#/definitions/TemplateForEachDirective" }
                        },
                        "iterator": {
                            "type": "array", 
                            "items": { "$ref" : "#/definitions/TemplateIteratorDirective" }
                        },
                        "if": {
                            "type": "array", 
                            "items": { "$ref" : "#/definitions/TemplateIfDirective" }
                        },
                        "lwcDom": {
                            "type": "array", 
                            "items": { "$ref" : "#/definitions/TemplateLwcDomDirective" }
                        }
                    }
                },
                "serializedDataBindingAST": {
                    "description": "Trimmed down AST(Abstract Syntax Tree) of the template. ",
                    "type": "string"
                }
            },
            "required": [ "fileName", "fileType" ]
        },
        "ScriptFile": {
            "description": "Metadata about a file authored in javascript(or its variants like TypeScript) in an LWC bundle.",
            "type": "object",
            "properties": {
                "fileType": { "type": "string", enum: ["js"]},
                "fileName": { "type": "string" },
                "componentClasses": {
                    "description": "Classes that extends the 'LightningElement' class directly or by extending another component class.",
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/ComponentClass"
                    }
                },
                "imports": {
                    "description": "Static import statements in a javascript module.",
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/Import"
                    }
                },
                "dynamicImports": {
                    "description": "Dynamic import statements in a javascript module, additionally information about hints if provided.",
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/DynamicImport"
                    }
                },
                "exports": {
                    "description": "Export statements in a module including reexporting bindings from another module.",
                    "type": "array",
                    "items": [
                        { "$ref": "#/definitions/Export"},
                        { "$ref": "#/definitions/ReExport"}
                    ]
                },
                "domEvents": {
                    "description": "DOM Event objects instantiated in a module. Includes Events and CustomEvents.",
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/DOMEvent"
                    }
                },
                "staticResources": {
                    "description": "Static resources referenced in a module. Includes any static urls referenced in javascript and resources loaded using the lightning platform resource loader, urls fetched via the salesforce scoped url resolvers.",
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/StaticResourceReference"
                    }
                }
            },
            "required": [ "fileName", "fileType", "exports" ]
        },
        "CSSFile": {
            "description": "Metadata about a css file in an LWC bundle.",
            "type": "object",
            "properties": {
                "fileType": { "const": "css" },
                "fileName": { "type": "string" },
                "customProperties": { 
                    "type": "array", 
                    "items": { 
                        "$ref": "#/definitions/CSSCustomProperty"
                    }
                },
                "staticResources": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/StaticResourceReference"
                    }
                }
            },
            "required": [ "fileName", "fileType" ]
        },
        "ReExport": {
            "description": "Information about an export statement used to reexport bindings imported from another module.",
            "type": "object",
            "properties": {
                "exportsList": { 
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "name": { "type": "string" },
                            "type": {
                                "type": "string",
                                "enum": ["class", "function", "expression"]
                            }
                        }
                    }
                },
                "moduleSpecifier": { "$ref": "#/definitions/ModuleReference" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["location", "exportsList", "moduleSpecifier"]
        },
        "ApiDecorator": {
            "description": "Information about the usage of an '@api' decorator to designate a class member as public.",
            "type": "object",
            "properties": {
                "type": { "const": "api" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "ComponentClass": {
            "description": "Information about a component class. A class is considered a component class if it extends 'LightningElement' class available in the standard 'lwc' module.",
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "extends": { "$ref": "#/definitions/ParentClass" },
                "properties": { 
                    "type": "array",
                    "items": { "$ref": "#/definitions/ClassProperty" },
                    "default": []
                },
                "methods": {
                    "type": "array",
                    "items": { "$ref": "#/definitions/ClassMethod" },
                    "default": []
                },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["extends", "location"]
        },
        "ClassMethod": {
            "description": "Information about a class method of a component class.",
            "type" : "object",
            "properties": {
                "name": { "type": "string" },
                "accessType": {
                    "type": "string",
                    "enum": ["public", "private", "static"]
                },
                "decorator": { 
                    "anyOf": [
                        {"$ref": "#/definitions/ApiDecorator" },
                        {"$ref": "#/definitions/TrackDecorator" },
                        {"$ref": "#/definitions/WireDecorator" }
                    ]
                },
                "parameterNames": { 
                    "type": "array",
                    "items": {
                        "anyOf": [
                            { "$ref": "#/definitions/DefaultParameters" },
                            { "$ref": "#/definitions/RestParameters"}
                        ]
                    }
                },
                "returnType": { "type": "string" },
                "returnValue": { "type": [ "number", "string", "boolean", "null", "object" ] },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": [ "name", "accessType", "location", "returnType" ]
        },
        "ClassProperty": {
            "description": "Information about a class property of a component class.",
            "type" : "object",
            "properties": {
                "accessType": {
                    "type": "string",
                    "enum": ["public", "private", "static"]
                },
                "dataProperty": { "type": "boolean"},
                "hasGetter": { "type": "boolean"},
                "hasSetter": { "type": "boolean"},
                "name": { "type": "string" },
                "decorator": { 
                    "type": "array",
                    "items": {
                        "anyOf": [
                            { "$ref": "#/definitions/TrackDecorator" },
                            { "$ref": "#/definitions/WireDecorator" },
                            { "$ref": "#/definitions/ApiDecorator"}
                        ]
                    },
                    "maxItems": 1
                },
                "initialValue": { "type": [ "number", "string", "boolean", "null", "object" ] },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": [ "name", "accessType", "location" ]
        },
        "CSSCustomProperty": {
            "description": "Information about css custom property.",
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "fallbackValue": { "$ref": "#/definitions/CSSCustomPropertyFallback"},
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": [ "name", "location" ]
        },
        "CSSCustomPropertyFallback": {
            "description": "Information about css custom property fallback values.",
            "anyOf": [
                { "type": "string"},
                { "$ref": "#/definitions/CSSCustomProperty"}
            ]
        },
        "ComputedValue": {
            "description": "Information about an expression, surrounded by curly braces, in a HTML template.",
            "type": "object", 
            "properties": {
                "expression": { "type": "string" },
                "root": { "$ref": "#/definitions/ClassProperty" }
            },
            "required": ["expression"]
        },
        "CustomElementReference": {
            "description": "Information about a custom element referenced in a HTML template.",
            "type": "object",
            "properties": {
                "tagName": { "type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" },
                "attributes": { 
                    "type": "array",
                    "items" : { "$ref": "#/definitions/ElementAttributeReference" }
                },
                "slots": {
                    "type": "array",
                    "items": { "$ref": "#/definitions/SlotReference" }
                },
                "eventListeners": {
                    "type": "array",
                    "items": { "$ref": "#/definitions/DeclarativeEventListener" }
                }
            },
            "required": [ "tagName", "location" ]
        },
        "DefaultParameters": {
            "description": "Information about default function parameters. Also look at parameters represented using rest syntax.",
            "type": "object",
            "properties": {
                "type" : { "const": "DefaultParameters" },
                "name": { "type": "string" },
                "defaultValue": {"type": [ "number", "string", "boolean", "null", "object" ] }
            }
        },
        "DOMEvent": {
            "description": "Information about a dom event instantiated in script.",
            "type": "object",
            "properties": {
                "eventType":  { "type": "string" },
                "isCustomEvent": { "type": "boolean" },
                "options": {
                    "type": "object",
                    "properties": {
                        "bubbles": { "type": "boolean" },
                        "composed": { "type": "boolean" }
                    }
                }
            },
            "required": ["eventType"]
        },
        "DynamicComponentReference": {
            "description": "Information about an element referenced in the template with a 'lwc:dynamic' attribute.",
            "type": "object",
            "allOf": [{"$ref": "#/definitions/CustomElementReference"}],
            "properties": {
                "ctor": { "$ref": "#/definitions/ComputedValue" }
            }
        },
        "DynamicImport": {
            "description": "Information about a dynamic import statement in a module.",
            "type" : "object",
            "properties": {
                "moduleSpecifier": { "$ref": "#/definitions/ModuleReference" },
                "moduleNameType": { "type": "string", "enum": ["string", "unresolved"] },
                "location": { "$ref": "#/definitions/SourceLocation" },
                "hints": { 
                    "type": "array",
                    "items": { "$ref": "#/definitions/DynamicImportHint" },
                    "default": []
                }
            },
            "required": ["moduleSpecifier", "moduleNameType", "location"]
        },
        "DynamicImportHint": {
            "description": "",
            "type": "object",
            "properties": {
                "rawValue": { 
                    "type": "string",
                    "description": "Hint value with leading and trailing spaces of hint phrase trimmed"
                },
                "key": { "type": "string" },
                "value": { "type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["rawValue", "key", "value", "location"]
        },
        "ElementAttributeReference": {
            "description": "Information about an element's attribute set in the template.",
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "value": { 
                    "anyOf": [
                        { "type": [ "null", "string", "boolean"] },
                        { "$ref": "#/definitions/ComputedValue"}
                    ]
                }
            },
            "required": ["name"]
        },
        "DeclarativeEventListener": {
            "description": "Information about an event handled declaratively in the template.",
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "value": { "$ref": "#/definitions/ComputedValue" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["name", "value", "location"]
        },
        "Export": {
            "description": "Information about a export statement in a module.",
            "type": "object",
            "properties": {
                "exportsList": { 
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "type": {
                                "type": "string",
                                "enum": ["class", "function", "expression"]
                            }
                        }
                    }
                },
                "default": { 
                    "type": "object",
                    "properties": {
                        "type": {
                            "type": "string",
                            "enum": ["class", "function", "expression"]
                        }
                    }
                },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["location"]
        },
        "Import": {
            "description": "Information about a static import statement in a module.",
            "type": "object",
            "properties": {
                "importType": { 
                    "type": "array",
                    "items": {
                        "type": "string",
                        "enum": [ "DefaultBinding", "NamedImports", "NamespacedImport"]
                    }
                },
                "importsList": { "type": [ "array", "string"] },
                "moduleSpecifier": { "$ref": "#/definitions/ModuleReference" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": [ "moduleSpecifier", "location" ]
        },
        "ModuleReference": {
            "description": "Information about a module imported into a module using the 'import' statement.",
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "namespace": { "type": "string" },
                "id": {
                    "type": "string",
                    "enum": [ 
                        "apexClass", "apexMethod", "apexContinuation", "client", "community",
                        "component", "contentAssetUrl", "customPermission", "dynamicComponent",
                        "slds", "messageChannel", "i18n", "gate", "label", "metric", "module",
                        "internal", "resourceUrl", "schema", "sobjectClass", "sobjectField",
                        "user", "userPermission"
                    ]
                },
                "type": {
                    "type": "string",
                    "enum": [ "lwc", "salesforce", "internal", "external", "local"]
                }
            },
            "required": ["name", "type"]
        },
        "ParentClass": {
            "description": "A parent component class including the 'LightningElement' class provided by the lwc module.",
            "oneOf": [
                {"$ref": "#/definitions/ComponentClass"},
                {            
                    "type": "object",
                    "properties": {
                        "name": { "type": "string" },
                        "source": { "$ref": "#/definitions/ModuleReference"}
                    }
                }
            ]
        },
        "RestParameters": {
            "description": "Information about function parameters received using the rest(...) syntax.",
            "type": "object",
            "properties": {
                "type": { "const": "RestParameters" },
                "name": { "type": "string" },
                "startIndex": { "type": "integer" },
                "endIndex": { "type": "integer" }
            },
            "required": [ "name", "startIndex" ]
        },
        "SlotReference": {
            "description": "Information about default and named slots being set by the parent component. The parent can specify a 'slot' attribute in the slotted content in the case of named slot. Slot content that do not have a 'slot' attribute will be passed as default slot content. The 'name' property in the SlotReference object will be 'default' for such a case.",
            "type": "object",
            "properties": {
                "name": { "type": "string"}
            },
            "required": [ "name" ]
        },
        "SourceLocation": {
            "description": "Object to represent the start and end position of a code block.",
            "type": "object",
            "required": [ "filename" ],
            "properties": {
                "fileName": { "type": "string" },
                "startLine": { "type": "integer" },
                "startColumn": { "type": "integer" },
                "endLine": { "type": "integer" },
                "endColumn": { "type": "integer" }
            },
            "required": ["fileName"]
        },
        "StaticResourceReference": {
            "description": "Information about reference to a static resource in a module. The resource can be a url specified as a string literal or a reference to a computed value.",
            "type": "object",
            "properties": {
                "type": { 
                    "description": "The type of static resource loaded, identifiable by the file extension.",
                    "type": "string",
                    "enum": [ "image", "css", "html", "js", "other"]
                },
                "value": { 
                    "anyOf": [
                        { "type": "string", "format": "uri" },
                        { "$ref": "#/definitions/ComputedValue"}
                    ]
                },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["type", "value", "location"]
        },
        "TemplateForEachDirective": {
            "description": "Information about for:each directive usage to render a list in a html template.",
            "type": "string",
            "properties": {
                "items": { "$ref": "#/definitions/ComputedValue"  },
                "itemName": { "type": "string"},
                "indexName": { "type": "string" },
                "key": {"type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["items", "location"]
        },
        "TemplateIfDirective": {
            "description": "Information about if:true or if:false directive usage to perform conditional rendering in a html template.",
            "type": "string",
            "properties": {
                "qualifier": {
                    "type": "string",
                    "enum": [ "true", "false" ]
                },
                "value": { "$ref": "#/definitions/ComputedValue"  },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "TemplateIteratorDirective": {
            "description": "Information about iterator directive usage in a html template.",
            "type": "string",
            "properties": {
                "items": { "$ref": "#/definitions/ComputedValue"  },
                "key": {"type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["items", "location"]
        },
        "TemplateLwcDomDirective": {
            "description": "Information about lwc:dom attribute usage in a html template to perform programmatic manipulation of dom tree.",
            "type": "string",
            "properties": {
                "value": {
                    "type": "string",
                    "enum": [ "manual" ]
                },
                "tagName": { "type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["value", "tagName", "location"]
        },
        "TrackDecorator": {
            "description": "Information about an @track decorator usage.",
            "type": "object",
            "properties": {
                "type": { "const": "track" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            },
            "required": ["location", "type"]
        },
        "WireDecorator": {
            "description": "Information about @wire decorator usage includes information about the data source.",
            "type": "object",
            "properties": {
                "type": { "const": "wire" },
                "location": { "$ref": "#/definitions/SourceLocation" },
                "adapterId": { "type": "string" },
                "adapterModule": { "$ref": "#/definitions/ModuleReference" },
                "adapterConfig": { "type": "object" }
            },
            "required": ["location", "type", "adapterId"]
        }
    }
}
```
</details>

# Open questions
* Should gathering metadata about configuration(`*.js-meta.xml`) file in a bundle be in the scope of this RFC?
  * Dean Moses from Builder Framework team says it is not required.
* Is documentation(.md) in the scope of this RFC? Will it overlap with [this RFC](https://github.com/salesforce/lwc-rfcs/pull/26)
* Does svg file have any useful metadata?
* For a script file, should the metadata gathering be restricted to only ComponentClasses or be more broad and gather metadata about all classes?
  * Should it collect data about only exported classes?

# References

- [JSON Schema type system](https://salesforce.quip.com/SH2FA614DXQO)<sup>*</sup>
- [RFC - Canonical View Metadata](https://salesforce.quip.com/ZJ93A0XQZvnU)<sup>*</sup>

<sup>*</sup> _restricted access_