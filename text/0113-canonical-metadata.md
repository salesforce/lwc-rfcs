---
title: Canonical specification of metadata for LWC modules
status: DRAFT
created_at: 2020-05-07
updated_at: 2020-05-07
rfc: 
champion: Aliaksandr Papko (@apapko) | Ravi Jayaramappa [@ravijayaramappa](https://github.com/ravijayaramappa)
implementation: https://git.soma.salesforce.com/lwc/lwc-api/pull/42 & https://git.soma.salesforce.com/lwc/lwc-platform/pull/361
---

# Canonical specification of metadata for LWC modules

Table of Content:
Summary


# Summary
> The [dictionary meaning](https://www.merriam-webster.com/dictionary/canonical%20form) of **Canonical form** means `the simplest form of something`. 

This RFC aims to document the metadata collected by statically analyzing the source code of Lightning Web Component(LWC) modules. It documents the standardized shape of metadata.

# Goals

- Completeness
- Eliminate redundancy: There should be only one way to get a desired information about a module.
- Consitency in property names
- Flexibility: Allow for new data to be collected in the future.

# Use Cases
## Referential Integrity
One of the key features of the Salesforce platform is referential integrity capabilities. Referential integrity is how the platform allows a given resource to be safely updated and deleted without adverse cascading effects.

## Editor support
Metadata about components can be used augment standard developer experience in code editors with code completion, attribute name & type validation, peek definition.

## Documentation
LWC components Metadata can be used to automatically generate documentation about components. Standard javascript documentation formats like JSDoc provided in component can also be gathered as metadata.

## Dependency analysis and prefetching
Components depend on static resources such as images, javascript files, css files and component definitions to name a few. For offline usage, knowing these dependencies at compile time allows the dependencies to be prefetched and avoid a network call.

## Code generation
Finally, metadata can facilitate low code development environment by allowing developer to declaratively define component. This metadata can be used to generate component stubs and the developer can add the specific business logic into the stubs thus reducing the amount of manually authored boiler plate code.
Metadata can also be transformed to generate components in various eco-systems like LWC, native web components etc. This would require adding an additional transformation of the metadata to another format like the custom-elements-json in the case of web components.

# Current state of art
## Web Components community
Since web components are relatively new in the industry, there is no well established standard or format for declaratively defining a web component. [custom-elements-json](https://github.com/webcomponents/custom-elements-json) is making an attempt to define a standard. It is in a very early stage. However there is a ecosystem developing around this standard. 
[web-component-analyzer](https://www.npmjs.com/package/web-component-analyzer) can analyze web components written in vanilla javascript and other popular web component libraries. It also supports custom-elements-json as an output format.

## LWC Platform
Currently, LWC platform gathers metadata as part of the compilation process. This metadata shape and implementation is currently private. This RFC is an attempt to make the metadata shape public as a first step. This [document](https://salesforce.quip.com/DdncANrJA0ko)<sup>*</sup> captured the information collected and the metadata shape currently gathered.

# Detailed Design

## Format
Metadata shape of the various objects we wish to analyse will be represented using in [JSON Schema](https://json-schema.org/) format. This is also augmented with a TypeScript definition of the shape.

[JSON](https://tools.ietf.org/html/rfc7159) is the IETF approved standard for representing objects. [JSON Schema](https://json-schema.org/) is on the path to become an IETF approved standard for describing JSON.

This format was also influenced by standardization choices made with in Salesforce UI platform.

## Entities that will be analyzed
These are the entities that will be analzed and metadata gathered about:
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
    * Template directives
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

## Metadata shape

```json
{
    "$schema": "http://json-schema.org/draft-07/schema#",
    "description": "A representation of metadata gathered for a LWC bundle file",
    "type": "object",
    "properties": {
        "fileType": { "$ref": "#/definitions/FileType" },
        "fileName": { "type": "string" }
    },
    "definitions": {
        "Class": {
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "extends": { "$ref": "#/definitions/Class" },
                "properties": { 
                    "type": "array",
                    "items": { "$ref": "#/definitions/ClassProperty" },
                    "default": []
                },
                "methods": {
                    "type": "array",
                    "items": { "$ref": "#/definitions/ClassMethod" },
                    "default": []
                }
            }
        },
        "ClassMemberType": {
            "type": "string",
            "enum": [ "property", "method"]
        },
        "ClassMethod": {
            "type" : "object",
            "properties": {
                "name": { "type": "string" },
                "isStatic": { "type": "boolean"},
                "decorator": { "$ref": "#/definitions/Decorator" },
                "returnType": { "type": "string" },
                "returnValue": { "type": [ "number", "string", "boolean", "undefined", "null", "unresolved" ] }
            }
        },
        "ClassProperty": {
            "type" : "object",
            "properties": {
                "hasGetter": { "type": "boolean"},
                "hasSetter": { "type": "boolean"},
                "name": { "type": "string" },
                "decorator": { "$ref": "#/definitions/Decorator" },
                "initialValue": { "type": [ "number", "string", "boolean", "undefined", "null", "unresolved" ] }
            }
        },
        "DynamicImport": {
            "type" : "object",
            "properties": {
                "moduleName": { "type": "string" },
                "moduleNameType": { "type": [ "unresolved", "string"] },
                "location": { "$ref": "#/definitions/SourceLocation" },
                "hints": { 
                    "type": "array",
                    "items": { "$ref": "#/definitions/DynamicImportHint" },
                    "default": []
                }
            }
        },
        "DynamicImportHint": {
            "type": "object",
            "properties": {
                "rawValue": { 
                    "type": "string",
                    "description": "Hint value with leading and trailing spaces of hint phrase trimmed"
                },
                "key": { "type": "string" },
                "value": { "type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "FileType": {
            "type": "string",
            "enum": [ "html", "js", "css" ]
        },
        "DecoratorType": {
            "type": "string",
            "enum": [ "wire", "track", "api" ]
        },
        "SourceLocation": {
            "type": "object",
            "required": [ "filename" ],
            "properties": {
                "fileName": { "type": "string" },
                "startLine": { "type": "integer" },
                "startColumn": { "type": "integer" },
                "endLine": { "type": "integer" },
                "endColumn": { "type": "integer" }
            }
        }
    }
}
```
# Reference

- [JSON Schema type system](https://salesforce.quip.com/SH2FA614DXQO)<sup>*</sup>

<sup>*</sup> _restricted access_