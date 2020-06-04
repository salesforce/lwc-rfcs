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

## Definitions
* Component class: Any class declared using the ES6 class syntax and extending 'LightningElement' binding imported from the built-in 'lwc' module.
* Static resource: For the purpose of this RFC, any resource referenced by a fully qualified uri will be considered a static resource. This RFC does not attempt to analyze static resource determined at runtime, for example, by the usage salesforce scoped modules like `@salesforce/resourceUrl`.

## Metadata shape

### TypeScript

#### Bundle metadata shape
```js
interface BundleMetadata {
    version: string;
    // Full canonical name in camel case
    moduleSpecifier: string;
    name: string;
    namespace: string;
    // Define the type of module that is represented by the given metadata object.
    moduleType?: 'HTML' | 'CSS' | 'Javascript';

    /**
     * For a component bundle, this property is a reference to the main component class in the bundle.
     * This class must be the default export of the javascript file. The file must be named the same
     * as the bundle
     */
    defaultComponentClass?: Class;

    // Information about HTML template files in the bundle.
    templates?: HTMLTemplateFile[];

    // Information about javascript files in the bundle.
    scripts?: ScriptFile[];

    // Information about css files in the bundle.
    css?: CSSFile[];
}
```
Other alternatives:
- Indexed by file type and file name
```ts
interface BundleMetadataOption2 {
    version: string;
    moduleSpecifier: string;
    name: string;
    namespace: string;
    // Define the type of module that is represented by the given metadata object.
    moduleType?: 'HTML' | 'LightningElement' | 'CSS' | 'Javascript' | 'LightningEvent';

    /**
     * For a component bundle, this property is a reference to the main component class in the bundle.
     * This class must be the default export of the javascript file. The file must be named the same
     * as the bundle
     */
    defaultComponentClass?: Class[];

    // Information about HTML template files in the bundle, indexed by file name
    templates?: { [key: string]: HTMLTemplateFile[] };

    // Information about javascript files in the bundle, indexed by file name
    scripts?: { [key: string]: ScriptFile[] };

    // Information about css files in the bundle, indexed by file name
    css?: { [key: string]: CSSFile[] };
}
```

#### HTML Template file metadata shape
```js
// Root metadata object for an HTML template file
interface HTMLTemplateFile {
    fileType: 'html';
    fileName: string;
    // List of unique components referenced in the template
    componentReferences?: ComponentReference[];
    /**
     * Static resources referenced in a module. For example: 'src' attribute value of an <img>, <source> tag.
     * Will be collected only if the url is fully statically analyzable
     */
    staticResources?: StaticResourceReference[];
    // Template in AST format starting with the root <template> node. Note: this is a partial AST
    experimentalAST?: Node;
}

// Represents a component referenced in the template.
// Use the 'tagName' property for kebab case and 'name' property for the reference in canonical form
interface ComponentReference extends ModuleReference {
    // Component reference always has a namespace, is a required property
    namespace: string;
    // No moduleIdentifier for a component
    moduleIdentifier: never;
    // Components are always imported from an external bundle
    type: 'external';
    // element's tag name as it appears in the template, retains the kebab casing
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
```js
// Root Metadata object for a file authored in javascript(or its variants like TypeScript) in an LWC bundle.
interface ScriptFile {
    fileType: 'js';
    fileName: string;
    // A list of unique ModuleReference objects referenced by this script file
    moduleReferences: ModuleReference[];
    // List of class metadata, covers only component class and any exported class
    // class has to be created using the ES6 'class' syntax
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
    // EventTarget.dispatchEvent invocations
    eventDispatches?: EventDispatch[];
    /**
     * Static resources referenced in a module. Includes any static urls referenced in javascript.
     * The urls have to be fully qualified.
     * Does not include resources loaded via lightning/platformResourceLoader or a url determined at runtime.
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
    // Location starts with the "class" key word and ends at the closing curly braces of the class body
    location: SourceLocation;
    id: ID;
    // A string literal representing the leading comments of the class declaration
    doc?: string;
}

interface ClassMember {
    type: 'property' | 'method';
    // According to javascript specs
    propertyFieldType: 'public' | 'private' | 'static';
    // Location starts with the first character of the member's identifier and ends at the semi-colon in case of a property and closing curly in case of a method
    location: SourceLocation;
    name: string;
    // A string literal representing the leading comments of the class member
    doc?: string;
}

// Information about a class property of a component class.
interface ClassProperty extends ClassMember {
    // To make it easy to discover
    propertyType: 'accessor' | 'dataProperty';
    // True, if the property is an instance property
    dataProperty?: DataProperty;
    // Preset if the property has a getter
    getter?: Getter;
    // Present if the property has a setter
    setter?: Setter;
    decorators?: [TrackDecorator | WireDecorator | ApiDecorator];
}

interface DataProperty {
    location: SourceLocation;
    // If the initial value is statically analyzable. If not analyzable, property will be omitted.
    initialValue?: number | string | boolean | null | object | undefined;
}

// Information about presence of getter
// Value property is added if its statically analyzable
interface Getter {
    // If the initial value is statically analyzable. If not analyzable, property will be omitted.
    value?: number | string | boolean | null | object | undefined;
    location: SourceLocation;
    // A string literal representing the leading comments of the getter block
    doc?: string;
}

// Information about presence of setter
// Do not infer any information about return value, setters are expected to return void
interface Setter {
    // A string literal representing the leading comments of the setter block
    doc?: string;
    location: SourceLocation;
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
    type: 'wire';
    adapterId: string;
    // module name in canonical form
    adapterModule?: string;
    adapterConfig?: object;
}

// Information about a static import statement in a module.
interface Import {
    bindings: (DefaultBinding | NamedImport | NameSpaceImport)[];
    // module name in canonical form
    moduleSpecifier: string;
    // Location starts from the import key word to the end of statement
    location: SourceLocation;
    refId: ID; // back reference to the ModuleReference object
}

// Represents a Default import binding
// e.g: import defaultExport from "module-name";
interface DefaultBinding {
    importType: 'DefaultBinding';
    // The import name will reflect the name at the source and not the aliased name
    name: string;
    location: SourceLocation;
}

// e.g: import { export1 as alias1 } from "module-name";
interface NamedImport {
    importType: 'NamedImport';
    name: string;
    aliasName?: string;
    location: SourceLocation;
}

// For bindings in import statement like 'import * as name from "module-name";'
// This object captures information about the '* as name'
interface NameSpaceImport {
    importType: 'NameSpaceImport';
    aliasName: string;
    location: SourceLocation;
}

// Information about a dynamic import statement in a module.
interface DynamicImport {
    // module name in canonical form
    moduleSpecifier: string;
    moduleNameType: 'string' | 'unresolved';
    location: SourceLocation;
    hints?: DynamicImportHint[];
    refId: ID; // back reference to the ModuleReference object
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
    type: 'export';
    exportsList?: NamedExport[] | DefaultExport;
    // For easy discover-ability of default export
    default?: boolean;
    location: SourceLocation;
}

// Information about a export statement in a module.
// e.g. export
interface NamedExport {
    exportType: 'NamedExport';
    value: ClassDeclaration | FunctionDeclaration | VariableDeclaration;
    // local name of the export
    name: string;
    // aliased name of the export
    aliasName?: string;
    location: SourceLocation;
}

interface DefaultExport {
    exportType: 'DefaultExport';
    value: ClassDeclaration | FunctionDeclaration | VariableDeclaration;
    location: SourceLocation;
}

interface ClassDeclaration {
    type: 'class';
    refId: ID; // Ties back to the class object at the root metadata object
}

interface FunctionDeclaration {
    type: 'function';
    name?: string;
    async?: boolean;
    parameterNames?: (DefaultParameters | RestParameters)[];
    returnType: string;
    // If the return value is statically analyzable. If not analyzable, property will be omitted.
    returnValue?: number | string | boolean | null | object;
    location: SourceLocation;
    // A string literal representing the leading comments of the function declaration
    doc?: string;
}

interface VariableDeclaration {
    type: 'variableDeclaration';
    name: string;
    // If the initial value is statically analyzable. If not analyzable, property will be omitted.
    initialValue?: number | string | boolean | null | object | undefined;
    location: SourceLocation;
}

// Information about an export statement used to reexport bindings imported from another module.
interface ReExport {
    type: 'reexport';
    exportsList: {
        name: string;
        aliasName?: string;
    }[];
    // module name in canonical form
    moduleSpecifier: string;
    location: SourceLocation;
    refId: ID; // back reference to the ModuleReference object
}

// Information about a dom event instantiated in script.
interface DOMEvent {
    eventType: string;
    isCustomEvent?: boolean;
    options?: {
        bubbles?: boolean;
        composed?: boolean;
    };
    location: SourceLocation;
    id: ID;
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

interface EventDispatch {
    targetType: 'host' | 'shadowRoot' | 'Node';
    event: {
        refId: ID; // Back pointer to the metadata for the DOMEvent
    };
    location: SourceLocation;
}

// A parent component class including the 'LightningElement' class provided by the lwc module.
type ParentClass =
    | {
          name: string; // Another class declared in the same file
          refId: ID; // Back pointer to the metadata for the parent class
      }
    // or a class imported from an external module
    | {
          // Parent class name. This could be the aliased import binding, use the module specifier
          // refId to check the name of the binding in the source module
          name: string;
          // module name in full canonical form, the ModuleReference object can be looked up at
          // the root ScriptFile object using this name
          moduleSpecifier: string;
          refId: ID; // Back pointer to the metadata for the parent class
          // Location of the parent class identifier
          location: SourceLocation;
      }
    | 'unresolved'; // When the parent class is not statically analyzable, for example mixins

// Information about a module imported into a module using the 'import' statement.
interface ModuleReference {
    // Full canonical name, eg. lightning/input, lightning/recordForm
    // name will be in camel case
    moduleSpecifier: string;
    // name of the bundle or module, will retain camel case
    name: string;
    // namespace if it exists
    namespace?: string;
    // Only present for salesforce scoped modules, else property is omitted
    sfdcResource?: {
        scoped:
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
        namespaceId: string; // scoped module namespace normalization
    };
    // whether the module reference is the built-in lwc module or the @salesforce scoped module or
    // a local file in the current bundle('internal') or an module imported from an external source('external', eg. a component or a lib)
    type: 'lwc' | '@salesforce' | 'internal' | 'external';
    location: SourceLocation;
    id: ID;
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

// Unique identifier per object, other metadata objects of the same module can have a back reference
// to this id. The uniqueness of ids is only guaranteed within the module. It cannot span multiple modules
type ID = string;
```
Examples:
Samples of the metadata for script files can be viewed [here](https://github.com/salesforce/lwc-metadata/pull/2)

#### CSS file metadata shape
```js
// Root metadata object for a css file
interface CSSFile {
    fileType: 'css';
    fileName: string;
    imports?: ModuleReference[];
    customProperties?: CSSCustomProperty[];
    staticResources?: StaticResourceReference[];
}

// Information about css custom property.
interface CSSCustomProperty {
    name: string;
    fallbackValue?: CSSCustomPropertyFallback;
    location: SourceLocation;
}

// Information about css custom property declaration
// Information about css custom property fallback values.
type CSSCustomPropertyFallback = string | CSSCustomProperty;
```
Examples:
Samples of the metadata for css files can be viewed [here](https://github.com/salesforce/lwc-metadata/pull/3)

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
    type: 'image' | 'css' | 'html' | 'js' | 'svg' | 'other';
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

## Metadata Shape Option 2
Overview:
The vision is to allow expert users to retrieve file based metadata and for the regular users to retrieve an easy, self-contained metadata object that exposes all publicly available/accessible properties of the current bundle.

The main differences in proposed shape #2:
- no decorators: external consumer shouldn't be concerned if the class fields are decorated or not. All they need is the information that describes whether the class member is public. With that, instead of collecting decorators, each class member will carry 'isPublic' property to indicate its access. 
- success: an indication of successful collection. If errors occurred during the metadata collection, then 'success' will be false and diagnostics will be reported.
- diagnostics: even though metadata collection is not responsible for validating the syntax, any unexpected failures during the collection should be handled gracefully and surfaced to the user
- javascript declarations: file metadata revolves around declarations (both exported and private) - that can be functions, classes, consts, etc. The reason for this is that we don't know the content of the file and cannot make an assumption about its shape, therefore the metadata is collected from declarations.
- bundle interface: an interpretation of the bundle shape. It is an object that contains public properties, slots, events or anything that is accessible from the outside of the component/module. The reason for doing so is that individual file metadata doesn't have the notion of other files in the bundle and therefore cannot make assumption about an overall shape. With that, the interface is an additional step that attempts to deduce the shape and provide a union of all public properties. 

### TypeScript

#### Root Metadata Object
```js
interface BundleMetadata {
    success: boolean;
    version: string;
    diagnostics: Array<Diagnostic>;
    entry: string;
    results: { [name: string]: FileMetadata } // metadata for each source
    interface: BundleInterface; // public methods, events, css properties, exports, slots.
}

interface Diagnostic {
    message: string;
    code: number;
    filename?: string;
    location?: Location;
    level: DiagnosticLevel;
}

enum DiagnosticLevel { Fatal, Error, Warning, Log }

// A bundle interface; which contains a union of all file-based
// metadata objects with some wits about inheritance and overrides. 
interface BundleInterface {
    type: 'module' | 'component';
    eventListener: Array<EventListener>;
    emittedEvents: Array<ClassEvent>;
    publicProperties: Array<ClassMember>;
    publicMethods: Array<ClassMember>;
    slots: Array<SlotMeta>;
    exports: Array<ClassMetadata | FunctionMetadata | VariableMetadata>
}
```

<details>
<summary>Click to view format</summary>

#### Root Metadata Object Per File
```ts
interface FileMetadata {
    name: string,
    path: string;
    type: 'script' | 'html'| 'css',
    dependencies: Array<Reference>, // dynamic import is treated as a module dependency type of 'dynamic'
}

interface Reference {
    type: ReferenceType;
    id: string;
    namespacedId?: string;
    file: string;
    locations: Location[];
}
interface ReferenceReport {
    references: Reference[];
    diagnostics: Diagnostic[];
}
type ReferenceType = 'apexClass' | 'apexMethod' | 'apexContinuation' | 'client' | 'community' | 'component' | 'contentAssetUrl' | 'customPermission' | 'dynamicComponent' | 'slds' | 'messageChannel' | 'i18n' | 'gate' | 'label' | 'metric' | 'module' | 'internal' | 'resourceUrl' | 'schema' | 'sobjectClass' | 'sobjectField' | 'user' | 'userPermission';
```

For every source in the bundle there will be a corresponding typed class which extends from the base FileMetadata.

#### Script Metadata (.js|.ts)
```js
interface ScriptMetadata extends FileMetadata {
    type: 'script';
    declarations: Array<ClassMetadata | FunctionMetadata | VariableMetadata>
} 

interface FunctionMetadata {
    name?: string;
    params?: Array<any>;
    returnValue?: ClassMemberValue;
    context?: ClassMemberValue; // invocation context
}

interface VariableMetadata {
    name?: string;
    value?: ClassMemberValue;
}

interface ClassMetadata {
   name?: string; // optional due to anonymous class
   extends: ExtendsMeta; // id; resource; location of the super 
   classMembers: Array<ClassMember>; // includes private/public props/methods
   documentation: ClassDocumentation;
   events: Array<ClassEvent>;
   listeners: Array<ClassListener> // programmatic or declarative event listeners found in the file
}

interface ClassEvent {
    name: string;
    className: string; // class the event is fired from 
    location: Location;
}

interface ClassListener {
    name: string;
    handlerName: string;
    className: string; // class the event is fired from 
    location: Location;
}

interface ClassDocumentation {
    classDescription: string;
    html: string;
    metadata: Object;
}

interface ExtendsMeta {
    id: string;
    type: 'class' | 'function';
    resource: string;
    location: Location;
}

interface ClassMember {
    name: string;
    type: string;
    isPublic: boolean;
    documentation?: ClassMemberDocumentation;
    wire?: WireDependency
}

interface ClassMemberDocumentation {
    description: string;
    defaultValue: string;
    dataType: string;
    isRequired: boolean
}

interface ClassProperty extends ClassMember {
    hasGetter: boolean;
    hasSetter: boolean;
    value?: ClassMemberValue;
}

interface ClassMethod extends ClassMember {
    returnValue?: ClassMemberValue;
}

interface ClassMemberValue {
    type: ClassMemberValueType; 
    value: any;
    importedName: string;
}

enum ClassMemberValueType {
    ARRAY, BOOLEAN, MODULE, NUMBER, NULL, OBJECT, STRING, UNDEFINED, UNRESOLVED,
}

interface WireDependency {
    adapter: WireTargetAdapter;
    params?: { [name: string]: ClassMemberValue }; // value has type and actual value
    
}

interface WireTargetAdapter {
    adapterId: string;
    moduleSpecifier: string; // points to the adapter resource
}

interface SlotMeta {
    name: string;
    scope: string; // parent element name
}
```

### HTML Metadata
```ts
interface HTMLMetadata extends FileMetadata {
    type: 'html';
    tags: Array<CustomElementMetadata>;
    slots: Array<SlotMeta>;
    staticResources: Array<Reference>, //images, urls, etc with location
}

interface CustomElementMetadata {
    attributes: {
        [name: string]: DependencyParameter;
    };
    properties: {
        [name: string]: DependencyParameter;
    };
    events: Array<HTMLEventListener>, // not sure if the even belongs to a js or html meta
}

interface DependencyParameter {
    type: 'literal' | 'expression';
    value: string | boolean;
}

interface HTMLEventListener {
    name: string;
    handlerName: string;
    tagName: string; // custom element the handle is attached to
    location: Location;
}
```

### CSS Metadata 
```js
interface CSSMetadata extends FileMetadata {
    type: 'css';
    imports: Array<Reference>; // css only module imports
    customProperties: Array<CssCustomProperty>;
}

interface CssCustomProperty {
    external: boolean; // determines if the custom property is defined in the current file
    name: string;
    fallback: string;
    value?: {
        type: string;
        value: string;
        fallback: string;
    }
}
```
</details>

# Open questions
* Should gathering metadata about configuration(`*.js-meta.xml`) file in a bundle be in the scope of this RFC?
  * **Answer:** Dean Moses from Builder Framework team says it is not required.
* Is documentation(.md) in the scope of this RFC? Will it overlap with [this RFC](https://github.com/salesforce/lwc-rfcs/pull/26)
* Does svg file have any useful metadata?
* For a script file, should the metadata gathering be restricted to only ComponentClasses or be more broad and gather metadata about all classes?
  * Should it collect data about only exported classes?
  * **Answer:** Collect metadata only about component classes and any exported class. A component class is one that extends LightningElement.

# Future work
* Make usage of `lightning/platformResourceLoader` statically analyzable. A separate RFC will be required for this work. Initial suggestion is to take the same approach as hints for dynamic imports.

# References

- [JSON Schema type system](https://salesforce.quip.com/SH2FA614DXQO)<sup>*</sup>
- [RFC - Canonical View Metadata](https://salesforce.quip.com/ZJ93A0XQZvnU)<sup>*</sup>

<sup>*</sup> _restricted access_
