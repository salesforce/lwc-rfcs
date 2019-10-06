- Start Date: 2019-10-06
- RFC PR: 
- Lightning Web Component Issue: 

# Summary

Generation of PDFs from current screen of LWCs

# Basic example

Since current PDF generation compiler(Slicer, I guess) does not support JS/dynamic generation of reports/charts; specially generated from LWCs. Hence, we need to rewrite the entire syntax in VF. This will be like just give a button - convert to PDF/PDF report,so we can reuse the existing tested and beautifully working LWCs. 

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

Since, in business there are lots of use cases, where we need to generate PDFs. Specially if there're lots of customization requires/visual reports to the end customer. However current Salesforce implementation only support VF with static html(even reports are manipulated by means of images). So, I am expecting a solution where lightning web/aura components should be able to mold themselves into PDF, so we can save effort of redesigning the entire thing into VF also lots of code base will be optimized(and development efforts will be reduced). 



# Detailed design

I have not gone through entire code-base/design, so I will not be a best person to suggest the same. 

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity -- Big 
- whether the proposed feature can be implemented in user space -- Possibly by means of other third party libraries
- the impact on teaching people Lightning Web Components -- No/Less
- integration of this feature with other existing and planned features --N/A
- cost of migrating existing Lightning Web Components applications (is it a breaking change?) -- Imrproved/Zero

There are tradeoffs to choosing any path. Attempt to identify them here.


# Alternatives



# Adoption strategy


# How we teach this

This is more intuitive part, so no need of the same. 

# Unresolved questions


