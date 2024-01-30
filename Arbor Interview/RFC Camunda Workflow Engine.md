
# RFC: Camunda Workflow Engine

**Authors:**

- [@chrisgaunt](https://github.com/cgtopher)
## Executive Summary

*In order to enable the creation of complex workflows that are dependent on state, this proposes the use of [Camunda's Workflow Engine](https://camunda.com/platform-7/workflow-engine/). A library provided by Camunda that uses Business Process Management Notation ([BPMN](https://www.bpmn.org/)) to define the "paths" a form can take*

## Background

Historically in this company, a home-brewed rules engine has been used along with a Json form definition that would be evaluated along with a context object on call to the service `POST /form/<uuid>` . While this solution works to power the forms our customers currently use, there are a number of issues with it.

- The service has no real concept as to where data is located in state, as a result it must send the entire context in body of requests to integrations, leaving the responsibility of figuring out the state of the form and what data is available to the receiving services/functions, resulting in buggy and difficult to test code
- These context evaluations, once a form is large enough, become expensive operations as the service would have to fetch and process the whole definition, which introduces latency
- As customer requests become more complex, issues with the state management are more frequently found and difficult to debug

Camunda's workflow engine helps to address these concerns
- 
## Proposed Implementation


*Consider:*

- *using diagrams to help illustrate your ideas.*
- *including code examples if you're proposing an interface or system contract.*
- *linking to project briefs or wireframes that are relevant.*

## Metrics & Dashboards

*What are the main metrics we should be measuring? For example, when interacting with an external system, it might be the external system latency. When adding a new table, how fast would it fill up?*

## Drawbacks

*Are there any reasons why we should not do this? Here we aim to evaluate risk and check ourselves.*

## Alternatives

*What are other ways of achieving the same outcome?*

## Potential Impact and Dependencies

*Here, we aim to be mindful of our environment and generate empathy towards others who may be impacted by our decisions.*

- *What other systems or teams are affected by this proposal?*
- *How could this be exploited by malicious attackers?*

## Unresolved questions

*What parts of the proposal are still being defined or not covered by this proposal?*

## Conclusion

*Here, we briefly outline why this is the right decision to make at this time and move forward!*

