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
Below is an example how other LWC and Aura components can use this service
component.
## How a LWC component can use this service component ?
LWC Component JS file
```js
// Component JS file
import { LightningElement,track } from 'lwc';
import {isCategoryAllowedForCurrentConsent,setCookieConsent} from 'force/ePrivacyConsentCookie'
export default class HelloIteration extends LightningElement {
    @track
    categories = [
        {
            Id: 1,
            Name: 'Preferences',
            Title: 'Preference Cookies',
            consent: isCategoryAllowedForCurrentConsent("Preferences")
        },
        {
            Id: 2,
            Name: 'Marketing',
            Title: 'Marketing cookies',
            consent: isCategoryAllowedForCurrentConsent("Marketing")
        },
        {
            Id: 3,
            Name: 'Statistics',
            Title: 'Statistic Cookies',
            consent: isCategoryAllowedForCurrentConsent("Statistics")
        },
    ];
}
```

LWC Component HTML file
```html
// Component HTML file
<template>
    <lightning-card title="Consent Information" icon-name="custom:custom14">
        <div class="slds-m-around_medium">
           <template for:each={categories} for:item="category">
              <div key={contact.Id}>
                  <li>
                    {category.Name} <br>
                    {category.Title} <br>
                    {category.consent}<br>
                  </li>
                  <br>
              </div>
           </template>
        </div>
     </lightning-card>
</template>
```

## How an Aura component can use this service component
Aura component file
```js
<aura:component>
    <aura:handler name="init" value="{!this}" action="{!c.doInit}"/>
    <p>Aura component calling the utils lib</p>
    <!-- add the lib component -->
    <force:ePrivacyConsentCookie aura:id="ePrivacyConsentCookie" />
</aura:component>
```

Aura Component controller file
```js
({
    doInit: function(cmp) {
        // Call the lib here
        var libCmp = cmp.find('ePrivacyConsentCookie');
        var result = libCmp.isCategoryAllowedForCurrentConsent("Preferences");
        console.log("Is Preferences Category Allowed ?: " + result);
    }
})
```

## Motivation

We want to provide the cookie consent mechanism to our customers.
For customers and internal developers, we want to give them support to
make the cookies regulated and give them the ability to check the state 
of the cookie before setting any additional cookies.

We would like to have a versioned (and ideally an abstraction) of the 
format so that customers don’t need to worry about our internal 
serialization formats.

### Use Cases
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

This LWC component provide utility functions which allow teams to incorporate 
Cookie Consent mechanism in their components. This abstracts all the cookie 
fetching, reading, writing, updating logic and provide library functions to 
achieve the functionality. The component primarily export two functions :


- *isCategoryAllowedForCurrentConsent(categoryName*) - to check consent of 
specified category based on preferences set in “CookieConsent”.

- *setCookieConsent(cookieClassifications)*- This cookie will be provided by 
Salesforce and you can set preferences using setCookieConsent(cookieClassifications).

The goal is to enable this component to be exported by other teams, so they can utilize 
the cookie consent mechanism using the libraries provided. Teams can build their own 
components by using the library functions and embed the cookie consent functionality in 
their component without actually dealing with cookie values.




## Drawbacks
The biggest drawback of this feature is the improper use of consent cookies in the 
Lightning Platform and relying on cookies which can be modified. We have added guardrails 
which comply with Lightning Locker and will be updating them in the future.


## Alternatives

The alternative approach was to accomplish the functionality by using a WireAdapter and 
making API call to the server.

However, our approach has following pros over the alternative approach
- Reduced load on backend server
- Low latency due to no calls being made to server
- Same architecture as other market leaders for Cookie Consent.


## Adoption strategy

This is a brand-new feature and will be initially used by very limited number of customers.
Eventually, any customer using cookies in their component and wants to have cookie regulation
should use this component.

The impact of not having this service component would be that customers would not be able to
implement and use the cookie consent mechanism and would have to deal with cookies directly
in order to be compliant with privacy laws. This would add a lot of overhead to our customers.

Public documentation will be provided to customers on how to utilize this service component.

# How we teach this

The consumer aspect of it is very straight forward and customers will be provided with a
public document on how to use the libraries exposed. In the future, the team also plans to 
release a trailhead module explaining how to utilize this component for cookie consent.

# Unresolved questions

- Naming for the component can be changed.
- Some exposed APIs can be removed or added in future.
- Future functionality changes may be required based on Locker developments.

