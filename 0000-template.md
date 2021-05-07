---
title: Mixed Shadow Mode
status: DRAFTED
created_at: 2021-05-07
updated_at: YYYY-MM-DD
pr: (leave this empty until the PR is created)
---

# Mixed Shadow Mode

## Introduction

LWC implements a collection of polyfills that implement Shadow DOM features. These polyfills are
collectively referred to as "synthetic Shadow DOM" and are published to npm under the
`@lwc/synthetic-shadow` package. Synthetic shadow DOM provides a consistent experience for LWC web
components across the Salesforce-supported browser matrix.

Shadow DOM support has greatly increased since the start of the LWC project but applications
continue to rely on the synthetic shadow polyfill for various backwards-compatibility reasons. These
reasons include consistency of Shadow DOM features across targeted browsers, or reliance on escape
hatches for styling and accessibility.

Due to how `@lwc/synthetic-shadow` globally polyfills native APIs, applications that wish to use
LWC components in a native Shadow DOM context simply need to omit the inclusion of the polyfill;
however, this is not a realistic option for existing applications that cannot afford a complete
rewrite.

This proposal opens up an incremental migration path for applications moving to native Shadow DOM by
allowing the usage of both native and synthetic Shadow DOM in the same document.

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
