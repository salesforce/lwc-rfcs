---
title: Documentation-related Metadata
status: DRAFTED
created_at: 2020-03-10
updated_at: 2020-03-10
pr: (leave this empty until the PR is created)
---

# Documentation-related Metadata

## Summary

Component authors need a way to provide documentation-related metadata-- metadata that is specifically useful for documentation purposes and not runtime behavior. This includes information like component categorization and supported Salesforce experiences.

This metadata is comprised of key-value pairs and will be put in the documentation markdown file, `docs/module.md`.

## Basic example
Example documentation markdown file with documentation metadata:

```markdown
---
category: Navigation
experience: Lightning, Communities, Mobile
---

descriptive documentation...
```

## Motivation

As part of our path towards allowing customers to add documentation for their modules, customers need to be able to add documentation-related metadata. This metadata will be useful for tooling and reference materials. For example, component categories and experiences are displayed in the Component Library documentation website and also used for filtering lists. This metadata can also be used for linting in the LSPs.

In addition, for existing Salesforce-authored modules there's a need to put component authors in control of this metadata. Currently a hardcoded list is maintained by a single team which is a roadblock for other teams creating new components or wishing to update the values.

## Detailed design

Key-value pairs will be placed as [frontmatter](https://github.com/jonschlinkert/gray-matter#what-does-this-do) (in YAML format) in the module markdown file that lives under `__docs__` in the module bundle. For example, this could be the content for `input/__docs__/input.md`:

```markdown
---
category: Data Entry
experience: All
---

descriptive documentation...
```

### Recognized Fields

LWC itself will not restrict the set of fields a component author could add to the frontmatter, however only specific fields will be documented as recognized by the Salesforce platform. Unrecognized fields may be present but they will be ignored.

The following keys will be recognized in the frontmatter:

* __category__  `string` \
    The main category for the module. Only one may be specified. Salesforce will maintain a list of recognized categories in public documentation and the VS Code extension for code completion/linting. For applications like the Component Library, values that match the whitelist will be displayed on the component’s reference page. Also see the filters on the homepage of the [Component Library](https://developer.salesforce.com/docs/component-library/overview/components) for other ways this information can be used.
* __experience__  `string[]` \
    One or more Salesforce experiences that the module works in. Salesforce will maintain a list of recognized experiences in public documentation and the VS Code extension for code completion/linting. For applications like the Component Library, values that match the whitelist will be displayed on the component’s reference page. Also see the filters on the homepage of the [Component Library](https://developer.salesforce.com/docs/component-library/overview/components) for other ways this information can be used.
* __isSubComponent__  `boolean` \
    This denotes that the component is only intended for use inside of another component, for example `lightning-breadcrumb` is only used inside of `lightning-breadcrumbs`. The Component Library may use this information to automatically cross-link pages or provide [component grouping](https://gus.lightning.force.com/lightning/r/a07B0000006HrUNIA0/view) and this information could be used for linting.
* __deprecated__  `boolean` \
    This does not impact runtime behavior. The Component Library may use this information to provide filtering capabilities or special UX treatment. Is there an existing way deprecation is already specified? Does it make more sense to use the `@deprecated` tag from JSDoc? Also, should we have another field for the replacement component?

#### Recognized Categories

The following category values will be documented as recognized by Salesforce:

* Analytics
* API
* Buttons
* Data Entry
* Displaying Data
* Feed
* Files
* Forms
* Layout
* Logic
* Messaging
* Navigation
* Process/Workflow
* Service
* Visual

#### Recognized Experiences

The following experience values will be documented as recognized by Salesforce:

* All
* Communities
* Lightning
* Mobile
* Lightning Out
* Standalone
* Snapins

### Parsing

_Please note the following implementation is already in place, but is included here for completion. That said, now is a good time to address any concerns before opening up usage on the platform._

The frontmatter will be parsed by the platform compiler using [greymatter](https://github.com/jonschlinkert/gray-matter) and included in the documentation section of the compiler output:

```typescript
export interface PlatformBundleDoc {
    classDescription?: string;
    html?: string;
    metadata?: DocumentationMetadata;
}

export interface DocumentationMetadata {
    [key: string]: any;
}
```

In Aura this information will be placed on the ModuleDefinition as a `Map<String, String>`.

#### Field Agnostic

The platform-compiler will be agnostic to the fields and their meanings. Correct YAML grammar will be enforced, but downstream code such as the Component Library will be responsible for determining which fields are applicable and which values may need further processing, such as splitting the experience field on commas.

The benefit of this approach is flexibility and reduced overhead on the platform-compiler. Also consumption of these values is not limited to Salesforce technologies such as the Component Library, so a whitelist at the compiler level could prove too restrictive.

The downside is that content-based errors (such as using an unrecognized category) won’t be caught until further downstream. This can be mitigated somewhat with linting.

Another downside of being agnostic (and not validating the allowed fields) is that the future introduction of new fields may “conflict” with existing fields. That is, a component author could add any arbitrary field, even if Salesforce does not use or display that field, which could conflict with Salesforce’s use of that field name later. This should be rare, however, and since this data does not affect runtime behavior the impact is minimal. Customers will have time to make any necessary updates at their convenience.

## Alternatives

- While this information could continue to live apart from the module bundle as it is today, that's not a sustainable approach and does not work for the platform.
- This data could be written as custom JSDoc tags in the js file, but those tags would be non-standard. This information feels better suited to the documentation file.

### Rejected Fields

* __image__ \
    The Component Library uses images for components on the home page. The map from component to image is currently hardcoded. If we want to allow the platform to specify similar images then a field here is one solution, but it poses many complications for the implementation details. Also such images are of limited use in other contexts outside of the Component Library, which may not always use such images even itself.

## Adoption strategy

Currently we maintain a hardcoded list of categories and experiences for lightning base components and other exposed components. This information will be transferred to the individual component bundles and be maintained by the component owners going forward. 

This is also part of the effort to enable platform developers to create documentation for their components, in which we should allow them to provide the same categorization information as we do for Salesforce components.

# How we teach this

In general, every developer creating public components will need to make sure of the following:

1. The component has a quality markdown file.
2. The component specifies necessary documentation metadata (experience, category, etc...)
3. The component has one or more examples.
4. The component has JSDoc on public members.

Separate RFCs will be filed for opening up other related documentation features to the platform such as examples, events, non-component modules, and more.

# Unresolved questions

As mentioned above, is this really the correct format to signify deprecation?