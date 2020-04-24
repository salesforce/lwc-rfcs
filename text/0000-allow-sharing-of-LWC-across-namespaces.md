---
title: Allow sharing of LWC across namespaces 
status: DRAFTED
created_at: 2020-03-29
updated_at: 2020-04-21
pr: (leave this empty until the PR is created)
---

# Allow sharing of LWC across namespaces

## Summary

There is a tight coupling (1:1) between namespace and Lighting Web Components (LWC). Developer can use isExposed flag to expose the component to the world or keep it private within the namespace. 

This RFC proposes functionality to allow sharing of LWC across namespaces.

## Motivation

Some teams have more than one namespaces and only desire to share their common components internally. However, it is not possible today, which forces such teams to make their component available globally. This is an undesired effect as it introduces maintenance burden. 

## Basic example

**Proposal #1** - Leverage IsExposed flag

We will leverage isExposed tag in *LightingComponentBundle.xml (**-meta.xml)* file. If it is set to true that implies that this component is exposed to other namespaces. We will introduce new xml file called *NamespaceBundle.xml* at namespace level. This file will list out namespaces were all the LWC can be shared. Alternatively, if namespace tag has * it implies that it is open to the world. 

```
        <!-- LightingComponentBundle.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>49.0</apiVersion>
    <isExposed>true</isExposed>
</LightningComponentBundle>
```

```
            <!-- NamespaceBundle.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<NamespaceBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <namespaces>
        <namespace>*</namespace> <!-- open to the world -->
        
        <!-- open to following namespaces
            <namespace>force</namespace>
            <namespace>forcechatter</namespace>
            <namespace>ui</namespace>
        -->
    </namespaces>
</NamespaceBundle>
```

##### Pros
- Leverages isExposed tag without introducing any new tag.
- Sharing is easy as you have to edit one single file to add or remove access to the namespaces.

##### Cons
- Individual component cannot be expose to different namespaces. It is controlled at namespace level so either all or none. 
- Learning curve as we are extending isExposed flag.


**Proposal #2** - Don't touch IsExposed flag

We will introduce new tag in *LightingComponentBundle.xml (**-meta.xml)* file isShared. If it is set to true that implies that this component is shared across namespaces. We will introduce new xml file called *NamespaceBundle.xml*. It will enlist namespaces were all the LWC can be shared. 

__Note__ : isExposed flag has higher precedence over isShared flag. 

|  isExposed 	| isShared   	        |   Comments  	                |
|---	        |---	                |---	                        |
|  True 	    |  True (ignored)	    | Open to the world  	        |
|  True 	    |  False (ignored)	    | Open to the world  	        |
|  False     	|  True 	            | Open to Limited Namespaces  	|
|  False     	|  False 	            | Open to Local Namespace  	    |

```
        <!-- LightingComponentBundle.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>49.0</apiVersion>
    <isExposed>false</isExposed>
    <isShared>true</isShared>
</LightningComponentBundle>
```

```
            <!-- NamespaceBundle.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<NamespaceBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <namespaces>
        <namespace>force</namespace>
        <namespace>forcechatter</namespace>
        <namespace>ui</namespace>
    </namespaces>
</NamespaceBundle>
```

##### Pros
- Honor isExposed flag.
- Sharing is easy as you have to edit one single file to add or remove access to the namespaces.

##### Cons
- Individual component cannot be expose to different namespaces. It is controlled at namespace level so either all or none. 
- Learning curve as we are introducing new flag.
- Complex implementation as we have to set precedence logic in the code. 

**Proposal #3** - Granual expose control 
Add <exposed> tag to existing LightingComponentBundle file. This tag can contain list of namespaces where this components can be accessible. 


__Note__ : isExposed flag has higher precedence over shared flag. 

|  isExposed 	|   shared   	        |   Comments  	                |
|---	        |---	                |---	                        |
|  True 	    |  (ignored)	        | Open to the world  	        |
|  True 	    |  (ignored)	        | Open to the world  	        |
|  False     	|  ! isEmpty             | Open to Limited Namespaces  	|
|  False     	|  isEmpty	            | Open to Local Namespace  	    |

```
        <!-- LightingComponentBundle.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>49.0</apiVersion>
    <isExposed>false</isExposed>
    <shared>
        <namespaces>
            <namespace>force</namespace>
            <namespace>forcechatter</namespace>
            <namespace>ui</namespace>
        </namespaces>
    </shared>
</LightningComponentBundle>
```

##### Pros
- No need to introduce new xml file at namespace level since we are using existing LightingComponentBundle file.
- Honor isExposed flag.
- Granual control as we can control sharing of individual LWC. 

##### Cons
- Changing access for all namespace components involves changing all LWC bundle files. Too much of work for the developer.
- Learning curve as we are introducing new flag.
- Complex implementation as we have to set precedence logic in the code. 

## Detailed design

**TBD**



## Drawbacks

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Lightning Web Components
- integration of this feature with other existing and planned features
- cost of migrating existing Lightning Web Components applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

## Alternatives

What other designs have been considered? What is the impact of not doing this?



## Adoption strategy

If we implement this proposal, how will existing Lightning Web Components developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

**TBD**

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Lightning Web Components patterns?

**TBD** 

Would the acceptance of this proposal mean the Lightning Web Components documentation must be
re-organized or altered? Does it change how Lightning Web Components is taught to new developers
at any level?

**TBD**

How should this feature be taught to existing Lightning Web Components developers?

**TBD**

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
