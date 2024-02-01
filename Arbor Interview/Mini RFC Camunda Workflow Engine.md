
# Mini-RFC: Camunda Workflow Engine

**Authors:**

- [@chrisgaunt](https://github.com/cgtopher)
## Executive Summary

*In order to enable the creation of complex workflows that are dependent on state, this proposes the use of [Camunda's Workflow Engine](https://camunda.com/platform-7/workflow-engine/). A library provided by Camunda that uses Business Process Management Notation ([BPMN](https://www.bpmn.org/)) to define the "paths" a form can take*

## Background

Historically in this company, a home-brewed rules engine has been used to power the decision making around the paths the forms would travel.  It uses Json form definitions that would be evaluated along with context objects on each call to the service (submitting answers to a page) behind a single endpoint to determine the next page a customer would be shown . While this solution works to power the forms our customers currently use, there are a number of issues with it.

- The service has no real concept as to where data is located in state, as a result it must send the entire serialized context object to integrations, leaving the responsibility of figuring out the state of the form and what data is available to the receiving services/functions, resulting in buggy and difficult to test code
- There is no good way to query forms in progress, it often requires scripting to get a clear picture. Which isn't ideal particularly for monitoring 
- These context evaluations, once a form is large enough, become expensive operations as the service would have to fetch and process the whole definition, which introduces latency

Camunda's workflow engine helps to address these concerns
- The engine has the concept of [Process Variables](https://docs.camunda.org/manual/latest/user-guide/process-engine/variables/) and provides a Java API for querying them. These variables are scoped to the [Process Instances](https://docs.camunda.org/manual/latest/user-guide/process-engine/process-engine-concepts/#process-instances) and will make it possible to map data across pages and into integration points
- Above mentioned Process Instances are also easily queryable through the Java API, for which endpoints could be stood up to monitor instances in flight.
- It has a long development history with many enhancements for performance and reliability
## Proposed Implementation

The workflow engine accepts BPMN, which can be expressed both as XML or in code through the [builder api](https://docs.camunda.org/manual/latest/user-guide/model-api/bpmn-model-api/create-a-model/), to create [Process Definitions](https://docs.camunda.org/manual/latest/user-guide/process-engine/process-engine-concepts/#process-definitions). A form designer service will be made to create these definitions, attaching page rendering data to [User Tasks](https://docs.camunda.org/manual/7.20/reference/bpmn20/tasks/user-task/) (basically a unit of work that waits for an interaction before proceeding to the next task).

A workflow service will be created to manage the interactions between the customer and the engine. The service will query for the next task, and return the data to be rendered by the front end. Once a new submission comes in for the form, it will submit the data for that page, which kicks off the next task. 

In simplified terms, the code for creating an instance and fetching the data to render the page would look like this:

```Java
// Create new process instance based on id which could be mapped from a slug  
RuntimeService runtimeService = engine.getRuntimeService();  
ProcessInstance instance = runtimeService
    .createProcessInstanceById("<Form Definition ID>").execute();  
return instance.getProcessInstanceId();  
  
  
/*  
 Query for the task by the process instance ID, an ID can be stored in the  task's "formKey" field which is placed by default by Camunda 
*/
TaskService taskService = engine.getTaskService();
Task task = taskService.createTaskQuery()
    .processInstanceId(instance.getProcessInstanceId())
    .singleResult();

UUID pageId = UUID.fromString(task.getFormKey());  
return formRepository.getPageById(pageId);
```


Integrations can be attached to [Service Tasks](https://docs.camunda.org/manual/latest/reference/bpmn20/tasks/service-task/)  which are automatically executed while the service is querying for the next User Task. These tasks can be mapped to a [Java Delegate](https://docs.camunda.org/manual/7.20/user-guide/process-engine/delegation-code/#java-delegate) that will be responsible for making REST requests or executing AWS lambdas, and map the results back into the process variables in the instance.


# Drawbacks

Since the service task execution will happen while a thread is held open to query for the next task, the Java delegates will need to carefully handle timeouts and exceptions. If there are too many concurrent service tasks running, it could hog system resources and risk increased latency overall.

Considerations to make:
- Extra monitoring for execution time, number of service tasks running, as well as overall request latency
- Thinking through async approaches, to perhaps send an event from the server when the next page is ready, rather than holding the connection open

For now configuring the K8's deployment such that it autoscales at a relatively low cpu utilization threshold, should provide some reduction in impact of bursts of traffic.  

## Alternatives

##### [Flowable](https://www.flowable.com/)
- Works very similarly to Camunda (and was in fact created by some of Camunda's developers)
- Has more support for tying in Java executables to process definitions

The automations team is currently using Camunda to orchestrate automations for customers. Since Flowable is such a similar product, it doesn't make much sense to use it as consistent tooling should make it easier to tie the products together in the future.

##### [Spring State Machine](https://spring.io/projects/spring-statemachine)
- First class Spring Boot support
- Simple API for interacting with states

Using Spring State Machine would grant a lot of flexibility in how forms are interacted with, however states are defined in code only, so is not as easily serializable. This library is also more general purpose than for just powering workflows, so there would be a need for more underlying custom logic, adding complexity and hurting maintainability. 

## Unresolved questions

- What format should the pages use for rendering data? This should be something something that could be deserialized into Java objects for validation purposes.
	- [JSONForms](https://jsonforms.io/) is being considered for this purpose as there are Java libraries for interacting with JSON schema, which it operates on 
- Should page data be embedded in the form definitions or separately in its own table, and side-by-side with the BPMN in the deployment request?
