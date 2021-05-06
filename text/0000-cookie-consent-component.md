---
title: ePrivacy Cookie Consent
status: DRAFTED
created_at: YYYY-MM-DD
updated_at: YYYY-MM-DD
pr: (leave this empty until the PR is created)
---

# ePrivacy Cookie Consent LWC Component

## Summary

This LWC component provide utility functions which allow teams to incorporate
Cookie Consent mechanism in their components. This abstracts all the cookie 
fetching, reading, writing, updating logic and provide library functions to 
achieve the functionality.

## Basic example

If the proposal involves a new or changed API, include a basic code example.
Omit this section if it's not applicable.

## Motivation

We want customers to be able to manipulate the Consent cookie in LWC.
For customers and internal developers, we want to give them support to
be able to check the state of the cookie before setting any additional 
cookies.
We would like to have a versioned (and ideally an abstraction) of the 
format so that customers don’t need to worry about our internal 
serialization formats.

Use Cases
* As a customer, I’d like to be able to check if a user has granted 
consent before I set a cookie
* As a customer, I’d like to set the consent state
* As a customer, I’d like to be able to build a UI component where I 
display the current state of consent.
* As a customer, I build a component where I can modify the state of 
the consent, for example revoke access to Performance cookies.
* As a customer, I’d like to be able to build a native banner in 
Salesforce for managing cookie compliance, with an Opt-In option.
* As a customer, I only need my cookie consent code to run in 
Experiences (Communities)
* As a customer, I need to be able to use the consent cookie for 
authenticated and unauthenticated (guest) users.
* As a customer, I’d like to be able to set the duration that the 
consent cookie is granted for.


## Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Lightning Web Components to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## Drawbacks

Why should we *not* do this? Please consider:

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

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Lightning Web Components patterns?

Would the acceptance of this proposal mean the Lightning Web Components documentation must be
re-organized or altered? Does it change how Lightning Web Components is taught to new developers
at any level?

How should this feature be taught to existing Lightning Web Components developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
