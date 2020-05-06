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

**Proposal** - Don't touch IsExposed flag for now

We will introduce new tag, **namespaceScope** in LightingComponentBundle.xml (**-meta.xml) file. It’s a string which can take following value viz. global, targeted or local. This tag can be missing in that case we treat it as MISSING internally. 


**Global** : Acts similar to isExposed to true flag. It opens the LWC to the world.
**Targeted** : It implies that we are opening this LWC to limited namespaces. 
**Local** : This LWC is only open to local namespace

We will also introduced new xml file called NamespaceBundle.xml. This file is at namespace level. It will enlist namespaces where all the LWCs can be shared. This will allow sharing for LWCs to targeted namespaces. 

For now we will honor isExposed flag and display error for in valid scenarios listed below. However, in the future we will completely rely on namespaceScope tag. 

**Valid Cases**

| namespaceScope    | isExposed       | Comments                                                      |
|----------|-----------------|---------------------------------------------------------------|
| Global   | TRUE            | Open to the world                                             |
| Targeted | FALSE           | Open to the namespaces listed in the NamespaceBundle.xml file |
| Local    | FALSE           | Open to local name space only                                 |
| MISSING  | TRUE            | Open to the world                                             |
| MISSING  | FALSE           | Open to local name space only                                 |
| MISSING  | MISSING (FALSE) | Open to local name space only                                 |

**Error Cases**

| namespaceScope    | isExposed | Comments                   | Errors             |
|----------|-----------|----------------------------|--------------------|
| Global   | FALSE     | Failed during compile time | Compile time error |
| Targeted | TRUE      | Failed during compile time | Compile time error |
| Local    | TRUE      | Failed during compile time | Compile time error |


```
<!-- LightingComponentBundle.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>49.0</apiVersion>
    <isExposed>false</isExposed>
    <namespaceScope>global/targeted/local</namespaceScope> 
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

Pros

* Honor isExposed flag.
* Sharing is easy as you have to edit one single file to add or remove access to the namespaces.

Cons

* Individual component cannot be expose to different namespaces. It is controlled at namespace level so either all or none.
* Learning curve as we are introducing new flag.
* Complex implementation as we have to set precedence logic in the code.

## High level design

In order to honor the sharing of LWC across namespaces, we will be doing validation at difference stages viz.

#### Compile Time Validation : 

During compile time all we do is compile the component either using RINO (Aura) or NodeJS (LWC) compiler and load it into registry (BundleAwareDefRegistry). Since compiler has no context of namespace there is nothing much we can do here. During compile time we can validate Bundle xml file and error out as listed in the table above. 

#### On Boot :

During boot we try to warmup caches so that we can improve performance at the runtime. During this stage we try to retrieve the def, build dependency tree and try to validate them. Here we also link the definitions and check for its access.
Here we will also do the validation for  the namespaces.   

#### Run Time Validation : 

We may have to retrieve the def at runtime in some case when we are not able to fetch the def from registry. This will follow similar work flow as On Boot and will do access check at run time. We do log a warning for such look ups.

#### Detail level design

Skipped the parsing of  XML files for bravity


During the linking phase we have the def and its dependency tree. We also have its direct dependencies (Assumption is that we also retrieve its indirect dependency if that is not true we have to recurse to n level for below logic). 

Algo: 
loop for all the dependencies for given def
```
for given referencing def compare its namespace against allowed/targeted namespaces.

    if (true)
        continue;
    else
        QFE (fail fast)
``` 
Here we are using in memory map as cache for future use. Performance profiling can help us for the optimization either by using global cache or local cache later.

#### Alternatives

**Alernative #1**

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


**Alernative #2** - Granular expose control 
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
- Granular control as we can control sharing of individual LWC. 

##### Cons
- Changing access for all namespace components involves changing all LWC bundle files. Too much of work for the developer.
- Learning curve as we are introducing new flag.
- Complex implementation as we have to set precedence logic in the code. 

## Drawbacks

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Lightning Web Components
- integration of this feature with other existing and planned features
- cost of migrating existing Lightning Web Components applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.


## Adoption strategy

* Since we are only focusing on Internal LWC scope should be limited


## How we teach this

* Update the documentation and communicate internally about isExposed flag. 
* Since we are only focusing on Internal LWC scope should be limited

## Unresolved questions

* Does compiler brings in direct and in direct dependencies ? Not sure how does sub definition plays in to this.
* Can we do anything better than in memory map for storing the access check result ? 


