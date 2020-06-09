---
title: Canonical specification of metadata for LWC modules
status: DRAFT
created_at: 2020-05-07
updated_at: 2020-05-19
rfc: https://github.com/salesforce/lwc-rfcs/pull/33
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
    * [API](#api)
    * [Format](#format)
    * [Entities that will be analyzed](#entities-that-will-be-analyzed)
    * [Definitions](#definitions)
    * [Metadata shape](#metadata-shape)
        * [Bundle metadata shape](#bundle-metadata-shape)
        * [HTML Template file metadata shape](#html-template-file-metadata-shape)
        * [Script file metadata shape](#script-file-metadata-shape)
        * [CSS file metadata shape](#css-file-metadata-shape)
    * [Metadata Shape Option 2](#metadata-shape-option-2)
* [Open questions](#open-questions)
* [Future work](#future-work)
* [References](#reference)

# Summary
> The [dictionary definition](https://www.merriam-webster.com/dictionary/canonical%20form) of **Canonical form** is `the simplest form of something`. 

This RFC aims to document the shape of metadata collected by statically analyzing the source code of Lightning Web Component(LWC) modules. The scope of this RFC is strictly the shape of the metadata and does not cover the implementation details.

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

## Dependency analysis and pre-fetching
Components depend on static resources such as images, javascript files, css files and component definitions to name a few. For offline usage, knowing these dependencies at compile time allows the dependencies to be pre-fetched and avoid a network call.

## Code generation
Finally, metadata can facilitate low code development environment by allowing developer to declaratively define component. This metadata can be used to generate component stubs and the developer can add the specific business logic into the stubs thus reducing the amount of manually authored boiler plate code.
Metadata can also be transformed to generate components in various eco-systems like LWC, native web components etc. This would require adding an additional transformation of the metadata to another format like the custom-elements-json in the case of web components.

# Current State of Art
## Web Components community
There is no well established standard or format for declaratively defining a web component. [custom-elements-json](https://github.com/webcomponents/custom-elements-json) is making an attempt to define a standard. It is in a very early stage. However there is a ecosystem developing around this standard. 
[web-component-analyzer](https://www.npmjs.com/package/web-component-analyzer) can analyze web components written in vanilla javascript and other popular web component libraries. It also supports custom-elements-json as an output format.

## LWC Platform
Currently, LWC platform gathers metadata as part of the compilation process. This metadata shape and implementation is currently private. This [document](https://salesforce.quip.com/DdncANrJA0ko)<sup>*</sup> captures the information collected and the metadata shape currently gathered. Here is a summary of problems with the current implementation:
* Replicated type system in java and javascript that is maintained manually
* Dependency(reference) analysis is done outside of metadata gathering
* Change management to accommodate new metadata is cumbersome because the implementation is spread across two repositories

This RFC is an attempt to define the new metadata shape that adheres to the above-stated goals. The desire is to make this a stand alone npm package and make it public.

# Detailed Design

## API

For a bundle:
```js
type collectBundleMetadata =(
    ...files: {
        filename: string;
        source: string;
    }[]
) => BundleMetadata;
```

For an HTML file:
```js
type collectTemplateMetadata =(
    file: {
        filename: string;
        source: string;
    }
) => HTMLTemplateFile;

/** Experimental API, reusing existing ast from @lwc/template-compiler **/
import { IRElement} from '@lwc/template-compiler';
type collectTemplateMetadata =(
    file: {
        filename: string;
        root: IRElement;
    }
) => HTMLTemplateFile;
/** End Experimental API **/
```

For a script file:
```js
type collectScriptMetadata =(
    file: {
        filename: string;
        source: string;
    }
) => ScriptFile;

import { File } from '@babel/types';
type collectScriptMetadata =(
    file: {
        filename: string;
        astRoot: File;
    }
) => ScriptFile;
```

For a CSS file:
```js
type collectCssMetadata =(
    file: {
        filename: string;
        source: string;
    }
) => CSSFile;

import { Root } from 'postcss';
type collectCssMetadata =(
    file: {
        filename: string;
        root: Root;
    }
) => CSSFile;

```

## Format
There was an initial experiment to use JSONSchema and TypeScript as a format to describe the metadata shape. Based on the feedback from reviewers, TypeScript was adopted. This will make the RFC readable and concise.

A [JSON Schema](https://json-schema.org/) representation of the shape is provided for reference. [JSON](https://tools.ietf.org/html/rfc7159) is the IETF approved standard for representing objects. [JSON Schema](https://json-schema.org/) is on the path to become an IETF approved standard for describing JSON. The JSON Schema can be used to validated generated json metadata against the specification. 

There are libraries that can convert TypeScript to JSONSchema and vice versa. The conversion is not seamless and needs manual editing post-conversion. 

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
    * Event listeners
    * Slots
* Javascript file
    * Component classes
        * Inheritance
        * Public & private class members
        * Decorators
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
        * Re-exports
    * JSDoc
        * Class
        * Properties
        * Methods
    * Programmatically generated events
        * type
        * options: composed, bubbles
    * Programmatically fetched static resources
        * `<script>`
        * `<img>`
        * `<link>`
* CSS file
    * Custom property declaration and usage([--*](https://developer.mozilla.org/en-US/docs/Web/CSS/--*))
    * Static resources
    * CSS imports

## Definitions
* **Component class:** Any class declared using the ES6 class syntax and extending 'LightningElement' binding imported from the built-in 'lwc' module.
* **Static resource:** For the purpose of this RFC, any resource referenced by a fully qualified uri will be considered a static resource. This RFC does not attempt to analyze static resource determined at runtime, for example, by the usage salesforce scoped modules like `@salesforce/resourceUrl`.

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
```js
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

    // Template in AST format starting with the root <template> node. 
    // Note: this part of the metadata is retaining the existing shape with a
    // few additions(slot names, event listeners, lwc:dom, lwc:dynamic)
    experimentalAST?: string;
}

// Represents a component referenced in the template.
// Use the 'tagName' property for kebab case and 'name' property for the reference in canonical form
interface ComponentReference extends ModuleReference {
    // Component reference always has a namespace, is a required property
    namespace: string;
    // No sfdcResource for a component
    sfdcResource: never;
    // Components are always imported from an external bundle
    type: 'external';
    // element's tag name as it appears in the template, retains the kebab casing
    tagName: string;
    // ComponentReference object is only for reference gathering
    // Since a template can have multiple usages of same child component, individual locations should be
    // discovered via the AST
    location: never;
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

    // A list of ModuleReference objects referenced by this script file
    moduleReferences: ModuleReference[];

    // List of class metadata, covers only component class and any exported class.
    // class has to be created using the ES6 'class' syntax
    classes?: Class[];

    // Static import statements in a javascript module.
    imports?: Import[];

    // Dynamic import statements in a javascript module, additionally information about hints if provided.
    dynamicImports?: DynamicImport[];

    // Exports of local bindings
    exports: Export[];
    // Reexporting bindings from another module.
    // e.g. export * from …;
    reExports: ReExport[];

    // DOM Event objects instantiated in a module. Includes Events and CustomEvents.
    domEvents?: DOMEvent[];

    // Programmatically added event listeners in a module
    eventListeners?: ProgrammaticEventListener[];

    // EventTarget.dispatchEvent invocations
    eventsDispatched?: EventDispatch[];

    /**
     * Static resources referenced in a module.
     * Includes any fully qualified urls referenced in javascript.
     * Does not include resources loaded via lightning/platformResourceLoader or a url determined at runtime.
     */
    staticResources?: StaticResourceReference[];
}

// Information about a class
interface Class {
    // Is optional because class can be anonymous
    name?: string;

    // A class is considered a component class if it extends
    // 'LightningElement' class available in the standard 'lwc' module.
    isComponentClass?: boolean;

    // A parent can be another class declared locally, imported identifier from an external module.
    // When not statically analyzable, will be 'unresolved'(e.g. Mixin)
    extends: ParentClass;

    properties?: ClassProperty[];

    methods?: ClassMethod[];

    // Location starts with the "class" key word and ends at the closing curly braces of the class body
    location: SourceLocation;

    // A string literal representing the leading comments of the class declaration
    doc?: string;
    id: ID;
}

interface ClassMember {
    // Note: getter/setter types are considered a property
    type: 'property' | 'method';
    // According to javascript specs
    propertyFieldType: 'public' | 'private' | 'static';
    name: string;
    // Location starts with the first character of the member's identifier and ends at the semi-colon
    // in case of a property and closing curly in case of a method
    // For a accessor property type, the getter and setter property has the specific location
    location?: SourceLocation;
    // A string literal representing the leading comments of the class member
    doc?: string;
}

// Information about a class property of a component class.
interface ClassProperty extends ClassMember {
    type: 'property';
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

// The naming is derived from data descriptors v/s accessor descriptor
interface DataProperty {
    // If the initial value is not statically analyzable, type will be 'unresolved'
    initialValue: {
        type:
            | 'number'
            | 'string'
            | 'boolean'
            | 'null'
            | 'object'
            | 'undefined'
            | 'array'
            | 'unresolved';
        value?: any;
    };
}

// Information about presence of getter
interface Getter {
    // If the initial value is not statically analyzable, type will be 'unresolved'
    initialValue: {
        type:
            | 'number'
            | 'string'
            | 'boolean'
            | 'null'
            | 'object'
            | 'undefined'
            | 'array'
            | 'unresolved';
        value?: any;
    };
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

// Information about a class method
interface ClassMethod extends ClassMember {
    type: 'method';
    decorators?: [ApiDecorator | TrackDecorator | WireDecorator];
    parameterNames?: (DefaultParameters | RestParameters)[];
    // If the return value is not statically analyzable, type will be 'unresolved'
    returnValue: {
        type:
            | 'number'
            | 'string'
            | 'boolean'
            | 'null'
            | 'object'
            | 'undefined'
            | 'array'
            | 'unresolved';
        value?: any;
    };
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
    adapterConfig?: {
        // Configuration where a value is prefixed with '$' to reference a property on the component instance
        reactive: { [name: string]: string };
        // Configuration where value is a static
        static: {
            [name: string]: {
                type:
                    | 'number'
                    | 'string'
                    | 'boolean'
                    | 'null'
                    | 'object'
                    | 'undefined'
                    | 'array'
                    | 'unresolved';
                value?: any;
            };
        };
    };
}

// Information about a static import statement in a module.
interface Import {
    defaultBinding?: DefaultBinding;
    namedImports?: NamedImport[];
    namespaceImport?: NameSpaceImport;
    // module name in canonical form
    moduleSpecifier: string;
    // Location starts from the import key word to the end of statement
    location: SourceLocation;
    refId: ID; // back reference to the ModuleReference object
}

// Represents a Default import binding
// e.g: import defaultExport from "module-name";
interface DefaultBinding {
    // The import name will reflect the imported binding's local name
    name: string;
    location: SourceLocation;
}

// e.g: import { export1 as alias1 } from "module-name";
interface NamedImport {
    name: string;
    aliasName?: string;
    location: SourceLocation;
}

// For bindings in import statement like 'import * as name from "module-name";'
// This object captures information about the '* as name'
interface NameSpaceImport {
    aliasName: string;
    location: SourceLocation;
}

// Information about a dynamic import statement in a module.
interface DynamicImport {
    // module name in canonical form
    moduleSpecifier: string;
    refId: ID; // back reference to the ModuleReference object
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
    namedExports?: NamedExport[];
    defaultExport?: DefaultExport;
    location: SourceLocation;
}

// Information about a export statement in a module.
// e.g. export let name1, name2, …, nameN;
//      export { variable1 as name1, variable2 as name2, …, nameN };
interface NamedExport {
    value: ClassDeclaration | FunctionDeclaration | IdentifierDeclaration;
    // local name of the export
    name: string;
    // aliased name of the export
    aliasName?: string;
    location: SourceLocation;
}

// A default export
// e.g. export default function (…) { … }
interface DefaultExport {
    value: ClassDeclaration | FunctionDeclaration | IdentifierDeclaration | 'unresolved';
    location: SourceLocation;
}

interface ClassDeclaration {
    type: 'class';
    name?: string;
    refId: ID; // Ties back to the class object at the root metadata object
}

interface FunctionDeclaration {
    type: 'function';
    name?: string;
    async?: boolean;
    parameterNames?: (DefaultParameters | RestParameters)[];
    returnValue: {
        type:
            | 'number'
            | 'string'
            | 'boolean'
            | 'null'
            | 'object'
            | 'undefined'
            | 'array'
            | 'unresolved';
        value?: any;
    };
    location: SourceLocation;
    // A string literal representing the leading comments of the function declaration
    doc?: string;
}

interface IdentifierDeclaration {
    type: 'identifierDeclaration';
    name: string;
    // If the initial value is not statically analyzable, type will be 'unresolved'
    initialValue?: {
        type:
            | 'number'
            | 'string'
            | 'boolean'
            | 'null'
            | 'object'
            | 'undefined'
            | 'array'
            | 'unresolved';
        value?: any;
    };
    location: SourceLocation;
}

// Information about an export statement used to reexport bindings imported from another module.
// e.g. export * from …;
interface ReExport {
    exportSpecifiers: {
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
    event:
        | {
              refId: ID; // Back pointer to the metadata for the DOMEvent
          }
        | 'unresolved';
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
          moduleSpecifier: string;
          refId: ID; // Back pointer to the metadata for external module reference
          location: SourceLocation;
      }
    | 'unresolved'; // When the parent class is not statically analyzable, for example mixins

// Information about a module reference
interface ModuleReference {
    // Full canonical name, eg. lightning/input, lightning/recordForm
    // name will be in camel case
    moduleSpecifier: string;
    // name of the bundle or module, will retain camel case
    name?: string;
    // namespace if it exists
    namespace?: string;
    // Only present for salesforce scoped modules, else property is omitted
    sfdcResource?: {
        scoped:
            | 'accessCheck'
            | 'apex'
            | 'apexContinuation'
            | 'client'
            | 'community'
            | 'component'
            | 'contentAssetUrl'
            | 'customPermission'
            | 'dynamicComponent'
            | 'messageChannel'
            | 'i18n'
            | 'gate'
            | 'label'
            | 'metric'
            | 'internal'
            | 'resourceUrl'
            | 'schema'
            | 'slds'
            | 'user'
            | 'userPermission';
        namespaceId?: string; // scoped module namespace normalization
    };
    type:
        | 'lwc' // reference is the built-in lwc module
        | '@salesforce' // reference is a @salesforce scoped module
        | 'internal' // reference is relative import, "internal" to the current bundle
        | 'external'; // reference is an external module like another component or a library
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
    customProperties?: {
        declarations?: CSSCustomPropertyDeclaration[];
        references?: CSSCustomPropertyReference[];
    }
    staticResources?: StaticResourceReference[];
}

// Information about css custom property.

interface CSSCustomProperty {
    name: string;
    location: SourceLocation;
}

interface CSSCustomPropertyDeclaration extends CSSCustomProperty {
    value: CSSCustomPropertyReference | string;
    scope: string; // the scope of the current custom property
}

interface CSSCustomPropertyReference extends CSSCustomProperty {
    fallback: CSSCustomPropertyFallback[] | null; // array to accommodate value with comma separated custom properties and strings: var(--default-border-radius, var(--border-top, 5px) 10px var(--border-bottom, 10 px) 10px);
}

type CSSCustomPropertyFallback = string | CSSCustomProperty;
```
Examples:
Samples of the metadata for css files can be viewed [here](https://github.com/salesforce/lwc-metadata/pull/3)

<details>
<summary>Click to view extended typescript definitions</summary>

```js
// Object to represent the start and end position of a code block.
interface SourceLocation {
    fileName: string;
    startLine: number;
    startColumn: number;
    endLine: number;
    endColumn: number;
}

// Information about reference to a static resource in a module. The resource can be a url specified
// as a string literal.
interface StaticResourceReference {
    // The type of static resource loaded, identifiable by the file extension.
    type: 'image' | 'css' | 'html' | 'js' | 'svg' | 'other';
    value: string;
    location: SourceLocation;
}

// !!Note: Retaining types from the existing data binding AST
enum BindingType {
    Boolean = 'boolean',
    Component = 'component',
    Expression = 'expression',
    ForEach = 'for-each',
    ForOf = 'for-of',
    Identifier = 'identifier',
    If = 'if',
    LwcDom = 'lwc:dom',
    LwcDynamic = 'lwc:dynamic',
    MemberExpression = 'member-expression',
    Root = 'root',
    String = 'string',
}

interface BindingBooleanAttribute {
    name: string;
    type: BindingType.Boolean;
    value: true;
}
interface BindingExpressionAttribute {
    name: string;
    type: BindingType.Expression;
    value: BindingExpression;
    location: SourceLocation;
}
interface BindingStringAttribute {
    name: string;
    type: BindingType.String;
    value: string;
    location: SourceLocation;
}

type BindingAttribute =
    | BindingBooleanAttribute
    | BindingExpressionAttribute
    | BindingStringAttribute;

interface BindingIdentifier {
    name: string;
    type: BindingType.Identifier;
    location: SourceLocation;
}

interface BindingMemberExpression {
    object: BindingIdentifier | BindingMemberExpression;
    property: BindingIdentifier;
    type: BindingType.MemberExpression;
    location: SourceLocation;
}

type BindingExpression = BindingIdentifier | BindingMemberExpression;

interface BindingForEachDirectiveNode extends BindingBaseNode {
    expression: BindingExpression;
    index?: BindingIdentifier;
    item: BindingIdentifier;
    type: BindingType.ForEach;
}

interface BindingForOfDirectiveNode extends BindingBaseNode {
    expression: BindingExpression;
    iterator: BindingIdentifier;
    type: BindingType.ForOf;
}

interface BindingIfDirectiveNode extends BindingBaseNode {
    expression: BindingExpression;
    modifier: string;
    type: BindingType.If;
}

interface BindingLwcDynamicDirectiveNode extends BindingBaseNode {
    expression: BindingExpression;
    type: BindingType.LwcDynamic;
}

interface BindingLwcDomDirectiveNode extends BindingBaseNode {
    value: string;
    type: BindingType.LwcDynamic;
}

type BindingDirectiveNode =
    | BindingForEachDirectiveNode
    | BindingForOfDirectiveNode
    | BindingIfDirectiveNode
    | BindingLwcDomDirectiveNode
    | BindingLwcDynamicDirectiveNode;

interface BindingBaseNode {
    children: BindingNode[];
    location: SourceLocation;
}

interface BindingEventTargetNode extends BindingBaseNode {
    eventListeners?: BindingExpressionAttribute;
}

interface BindingComponentNode extends BindingEventTargetNode {
    attributes: BindingAttribute[];
    tag: string;
    type: BindingType.Component;
    // Slots of this component node set by the parent
    slotReferences: string[];
}

interface BindingRootNode extends BindingBaseNode {
    type: BindingType.Root;
}

interface BindingElementNode extends BindingEventTargetNode {
    attributes: BindingAttribute[];
    type: BindingType.String;
}

type BindingNode =
    | BindingComponentNode
    | BindingDirectiveNode
    | BindingElementNode
    | BindingRootNode;
```
</details>

### JSONSchema(outdated - Typescript definitions are up to date)

[Click here](https://github.com/salesforce/lwc-metadata/pull/4) to view in JSONSchema format.

## Metadata Shape Option 2
**Overview:**
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
    customProperties?: {
        declarations?: CSSCustomPropertyDeclaration[];
        references?: CSSCustomPropertyReference[];
    }
}

// Information about css custom property.
interface CSSCustomProperty {
    name: string;
    location: SourceLocation;
}

interface CSSCustomPropertyDeclaration extends CSSCustomProperty {
    value: CSSCustomPropertyReference | string;
    scope: string; // the scope of the current custom property
}

interface CSSCustomPropertyReference extends CSSCustomProperty {
    fallback: CSSCustomPropertyFallback[] | null; // array to accommodate value with comma separated custom properties and strings: var(--default-border-radius, var(--border-top, 5px) 10px var(--border-bottom, 10 px) 10px);
}

type CSSCustomPropertyFallback = string | CSSCustomProperty;
```
</details>

# Open questions
1. Should gathering metadata about configuration(`*.js-meta.xml`) file in a bundle be in the scope of this RFC?
    - **Answer:** Dean Moses from Builder Framework team says it is not required.
2. Is documentation(.md) in the scope of this RFC? Will it overlap with [this RFC](https://github.com/salesforce/lwc-rfcs/pull/26)
3. Does svg file have any useful metadata?
4. For a script file, should the metadata gathering be restricted to only ComponentClasses or be more broad and gather metadata about all classes?
    - Should it collect data about only exported classes?
    - **Answer:** Collect metadata only about component classes and any exported class. A component class is one that extends LightningElement.
5. Should imports/module references/component references be renamed to "dependencies"?
6. Can a script file import a css module and should such an import have a separate ModuleReference.type?
7. At bundle level we should be able to to deduce if the bundle is a 'Component' or 'LightningElement'?
8. Should metadata collection phase produce diagnostics information? Diagnostics is information about errors that occurred during metadata collection. Is that the responsibility of the compilation phase?


# Future work
* Make usage of `lightning/platformResourceLoader` statically analyzable. A separate RFC will be required for this work. Initial suggestion is to take the same approach as hints for dynamic imports.

# References

- [JSON Schema type system](https://salesforce.quip.com/SH2FA614DXQO)<sup>*</sup>
- [RFC - Canonical View Metadata](https://salesforce.quip.com/ZJ93A0XQZvnU)<sup>*</sup>

<sup>*</sup> _restricted access_
