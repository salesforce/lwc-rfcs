---
title: portals proposal
status: DRAFTED
created_at: 2019-12-23
updated_at: 2019-12-23
pr:
---

# Short Title
Out of tree DOM insertion of custom elements via portals.

## Summary
Introduce a mechanism in LWC for inserting and managing an element outside of its natural dom tree.

## Basic example

The proposal suggests two new APIs:
1) A way to define where in the DOM the element would be inserted (eg. `lwc:portal-definition` attribute on an element)
2) A way to define the destination portal of an element (eg. `lwc:portal` attribute on an element)

One non-trivial example of such usage:

```xhtml
<body>
  <div lwc:portal-definition="global-modals"></div>

  <div lwc:portal-definition="popups"></div>
  ...
  <div id="tabs">
    <div id="tab-1">
      <div lwc:portal-definition="tab1-modals"></div>

      <div>
         <x-dialog lwc:portal="global-modals">Global Modal</x-foo>
         <x-dialog lwc:portal="tab1-modals">Scoped Modal</x-foo>
      </div>  
    </div>  
    <div id="tab-2">
      <div lwc:portal-definition="tab2-modals"></div>

      <div>
         <x-dialog lwc:portal="global-modals">Global Modal</x-foo>
         <x-dialog lwc:portal="tab2-modals">Scoped Modal</x-foo>
      </div>  
    </div>  
  </div>
</body>
```
since we wouldn't want for a user to specify the portal each and every time one would use a dialog, more realistically -- `<x-dialog>` would have a default behaviour based on the @Context, which can provide whether scoping is enabled or not, the name of the scoped portal and the name of the global portal.

## Motivation

Position of a node in the DOM determines its stacking context with the component having no 
real ability to change it besides creating a new stacking context. This becomes problematic
when one wants to present a popup or a modal dialog that appears on top of other existing
elements. 

Having a custom element be inserted out of its natural dom tree position would enable
one to have greater control over the stacking context, allowing a predictalbe popup
and modal implementation (including faux-modals that are scoped to tabs and are not per-se global).

## Detailed design

(not an actual detail designed just adding some possible requirements)
- Instead of being inserted at its natural position in the template, the element would be inserted in the destination portal,if one exists.
- When the host element is disconnected, the portalled element should also be disconnected. 
- The portalled component should still be @Context aware 

## Drawbacks

- new API
- additional complexity in LWC

## Security

- The portalled element content should not be accessible by anyone but the originally owning host element.

## Alternatives

In case of modals, some other strategies have been considered:
1. Event based creation of a modal with a modal manager defined at some parent node. 
Given that a modal could have up to 3 slots (header, default, footer), the ergonomics of usage become very cumbersome.
Moreover event-based modal creation is inferior to `<dialog>` like experience.
2. Exposing top layer by the browsers. This would allow one to effectively declare a new top layer and override any 
existing stacking context. It's unlikely that proposal would support scoping.

## Adoption strategy

It's not a breaking change, but it does introduce new API, adoption would initially be from component libraries with
support for modals and popups.

## How we teach this

This would allow implementation of components that can ignore the current stacking context. Teaching should be around what components can benefit from this.

## Unresolved questions

1. How would `querySelector` work ?
2. Is the definiton of the portal immutable (can you change its name, what if it gets removed?)
3. Can the destional portal be changed ?
4. What if the portal doesn't exist
5. What if there are more than 1 portal definition with the same name
6. Should there be implicit/reserved portals ? (eg. `body` which always implies the `<body>` element of the document) 
7. Should the events bubble out of the portalled component to the host element ?
