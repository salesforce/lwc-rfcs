---
title: Canonical specification of metadata for LWC modules
status: DRAFT
created_at: 2020-05-07
updated_at: 2020-05-13
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
    "description": "A representation of metadata gathered for a LWC bundle",
    "type": "object",
    "properties": {
        "name": { "type": "string" },
        "namespace": { "type": "string" },
        "moduleType": { 
            "type": "string",
            "enum": [ "Component", "Library", "CssOnly" ]
        },
        "templates": { 
            "type": "array",
            "items": { "$ref": "#/definitions/HTMLTemplate"}
        },
        "scripts": {
            "type": "array",
            "items": { "$ref": "#/definitions/Script"}
        },
        "defaultComponentClass": {
            "type": "array",
            "items": { 
                "$ref": "#/definitions/Class"
            }
        },
        "css": {
            "$ref": "#/definitions/CSSFile"
        }
    },
    "definitions": {
        "HTMLTemplate": {
            "type": "object",
            "properties": {
                "fileType": { "$ref": "#/definitions/FileType" },
                "fileName": { "type": "string" },
                "children": {
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/CustomElementReference"
                    }
                },
                "staticResources": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/StaticResourceReference"
                    }
                },
                "dynamicChildren": {
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/DynamicComponentReference"
                    }
                },
                "directives": {
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
                }
            }
        },
        "Script": {
            "type": "object",
            "properties": {
                "fileType": { "$ref": "#/definitions/FileType" },
                "fileName": { "type": "string" },
                "componentClasses": {
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/Class"
                    }
                },
                "imports": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/Import"
                    }
                },
                "dynamicImports": {
                    "type": "array",
                    "items": { 
                        "$ref": "#/definitions/DynamicImport"
                    }
                },
                "exports": {
                    "type": "array",
                    "items": [
                        { "$ref": "#/definitions/Export"},
                        { "$ref": "#/definitions/AggregatingExport"}
                    ]
                },
                "domEvents": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/DOMEvent"
                    }
                },
                "staticResources": {
                    "type": "array",
                    "items": {
                        "$ref": "#/definitions/StaticResourceReference"
                    }
                }
            }
        },
        "CSSFile": {
            "type": "object",
            "properties": {
                "fileType": { "$ref": "#/definitions/FileType" },
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
            }
        },
        "AggregatingExport": {
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
                "moduleSpecifier": { "$ref": "#/definitions/ModuleReference" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "ApiDecorator": {
            "type": "object",
            "properties": {
                "type": { "const": "api" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "Class": {
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
            }
        },
        "ClassMethod": {
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
            }
        },
        "ClassProperty": {
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
            }
        },
        "CSSCustomProperty": {
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "fallbackValue": { "$ref": "#/definitions/CSSCustomPropertyFallback"},
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "CSSCustomPropertyFallback": {
            "anyOf": [
                { "type": "string"},
                { "$ref": "#/definitions/CSSCustomProperty"}
            ]
        },
        "ComputedValue": {
            "type": "object", 
            "properties": {
                "expression": { "type": "string" },
                "root": { "$ref": "#/definitions/ClassProperty" }
            }
        },
        "CustomElementReference": {
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
                "eventHandlers": {
                    "type": "array",
                    "items": { "$ref": "#/definitions/EventHandlerReference" }
                }
            },
            "required": [ "tagName", "location" ]
        },
        "DecoratorType": {
            "type": "string",
            "enum": [ "wire", "track", "api" ]
        },
        "DefaultParameters": {
            "type": "object",
            "properties": {
                "type" : { "const": "DefaultParameters" },
                "name": { "type": "string" },
                "defaultValue": {"type": [ "number", "string", "boolean", "null", "object" ] }
            }
        },
        "DOMEvent": {
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
            "type": "object",
            "allOf": [{"$ref": "#/definitions/CustomElementReference"}],
            "properties": {
                "ctor": { "$ref": "#/definitions/ComputedValue" }
            }
        },
        "DynamicImport": {
            "type" : "object",
            "properties": {
                "moduleSpecifier": { "$ref": "#/definitions/ModuleReference" },
                "moduleNameType": { "const": "stringliteral" },
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
        "ElementAttributeReference": {
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "value": { 
                    "anyOf": [
                        { "type": [ "null", "string", "boolean"] },
                        { "$ref": "#/definitions/ComputedValue"}
                    ]
                }
            }
        },
        "EventHandlerReference": {
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "value": { "$ref": "#/definitions/ComputedValue" }
            }
        },
        "Export": {
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
            }
        },
        "FileType": {
            "type": "string",
            "enum": [ "html", "js", "css" ]
        },
        "Import": {
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
            }
        },
        "ModuleReference": {
            "type": "object",
            "properties": {
                "name": { "type": "string" },
                "namespace": { "type": "string" },
                "id": {
                    "type": "string",
                    "enum": [ "apexClass", "apexMethod", "apexContinuation", "client", "community", "component", "contentAssetUrl", "customPermission", "dynamicComponent", "slds", "messageChannel", "i18n", "gate", "label", "metric", "module", "internal", "resourceUrl", "schema", "sobjectClass", "sobjectField", "user", "userPermission"
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
            "oneOf": [
                {"$ref": "#/definitions/Class"},
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
            "type": "object",
            "properties": {
                "name": { "type": "string"}
            },
            "required": [ "name" ]
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
        },
        "StaticResourceReference": {
            "type": "object",
            "properties": {
                "type": { "$ref": "#/definitions/StaticResourceType" },
                "value": { 
                    "anyOf": [
                        { "type": "string", "format": "uri" },
                        { "$ref": "#/definitions/ComputedValue"}
                    ]
                },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "StaticResourceType": {
            "type": "string",
            "enum": [ "image", "css", "html", "js", "other"]
        },
        "TemplateForEachDirective": {
            "type": "string",
            "properties": {
                "items": { "$ref": "#/definitions/ComputedValue"  },
                "itemName": { "type": "string"},
                "indexName": { "type": "string" },
                "key": {"type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "TemplateIfDirective": {
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
            "type": "string",
            "properties": {
                "items": { "$ref": "#/definitions/ComputedValue"  },
                "key": {"type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "TemplateLwcDomDirective": {
            "type": "string",
            "properties": {
                "value": {
                    "type": "string",
                    "enum": [ "manual" ]
                },
                "tagName": { "type": "string" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "TrackDecorator": {
            "type": "object",
            "properties": {
                "type": { "const": "track" },
                "location": { "$ref": "#/definitions/SourceLocation" }
            }
        },
        "WireDecorator": {
            "type": "object",
            "properties": {
                "type": { "const": "wire" },
                "location": { "$ref": "#/definitions/SourceLocation" },
                "adapterId": { "type": "string" },
                "adapterModule": { "$ref": "#/definitions/ModuleReference" },
                "adapterConfig": { "type": "object" }
            }
        }
    }
}
```

# Open questions
* Should gathering metadata about configuration(`*.js-meta.xml`) file in a bundle be in the scope of this RFC?
* Is documentation(.md) in the scope of this RFC? Will it overlap with [this RFC](https://github.com/salesforce/lwc-rfcs/pull/26)
* Does svg file have any useful metadata?

# References

- [JSON Schema type system](https://salesforce.quip.com/SH2FA614DXQO)<sup>*</sup>
- [RFC - Canonical View Metadata](https://salesforce.quip.com/ZJ93A0XQZvnU)<sup>*</sup>

<sup>*</sup> _restricted access_