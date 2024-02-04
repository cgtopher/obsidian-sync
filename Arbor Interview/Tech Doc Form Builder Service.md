# Tech Doc: Form Builder Service

## Overview:
This document provides an overview of the Form Builder service, which is used to create new forms claimants use to submit claims. It is a Spring Boot service that has an API for creating, updating, and deploying forms to the Workflow Service. 

## Schema:
In order to provide data needed by [React Flow](https://reactflow.dev/), the main library used by the frontend, Form Builder service operates on a graph representation of forms. The model is built to be able to be easily passed to Camunda's [Model API](https://docs.camunda.org/manual/latest/user-guide/model-api/bpmn-model-api/create-a-model/) so it can generate the BPMN notation supported by the workflow engine (explored more farther down).

```SQL
-- Audit columns have been excluded  
CREATE TABLE forms(  
  id UUID PRIMARY KEY,  
  name CHARACTER VARYING NOT NULL,  
  bpmn CHARACTER VARYING NOT NULL -- XML representation  
);  
  
CREATE TABLE tasks(  
    id UUID PRIMARY KEY,  
    form_id UUID NOT NULL,  
    task_type CHARACTER VARYING NOT NULL, -- Enum handled in application layer  
    x_axis BIGINT NOT NULL,  
    y_axis BIGINT NOT NULL,  
    FOREIGN KEY (form_id) REFERENCES forms (id)  
);  
  
CREATE TABLE pages(  
    id UUID PRIMARY KEY,  
    form_id UUID NOT NULL,  
    task_id UUID NOT NULL,  
    definition JSONB NOT NULL,  
    FOREIGN KEY (form_id) REFERENCES forms (id),  
    FOREIGN KEY (task_id) REFERENCES tasks (id)  
);  
  
CREATE TABLE services(  
    id UUID PRIMARY KEY,  
    form_id UUID NOT NULL,  
    task_id UUID NOT NULL,  
    service_type CHARACTER VARYING NOT NULL, -- Enum  
    definition JSONB NOT NULL,  
    FOREIGN KEY (form_id) REFERENCES forms (id),  
    FOREIGN KEY (task_id) REFERENCES tasks (id)  
);  
  
CREATE TABLE sequences(  
    id UUID PRIMARY KEY,  
    form_id UUID NOT NULL,  
    from_task UUID NOT NULL,  
    to_task UUID NOT NULL,  
    FOREIGN KEY (from_task) REFERENCES tasks (id),  
    FOREIGN KEY (to_task) REFERENCES tasks (id)  
);
```


## Implementation:

On creation a row in `forms` is added with an initial BPMN, representing a process with no tasks. The rest of the resources are behind respective REST endpoints under `/forms`, which are called whenever a change happens to the form on the frontend, ie. dropping a node on the grid or creating/updating a page. The reasoning behind treating all of the components as separate resources is that it allows the frontend to handle all of the components individually. This makes it easier to have separate screens for building the flow, creating a page in the page builder, and defining an integration point in a service task,  and makes it the application as a whole more extensible. 


### Graph Endpoints
All endpoints for forms, tasks, and sequences return a `FormResponse` which describes the current state of the form, so the frontend can re-render and always be up-to-date with the server, particularly in the case a request to add a resource fails.

![[claimformcreator-Form Designer.drawio.png]]

**Form Response:**
```JSON
{  
  "id": "uuid",  
  "name": "string",  
  "tasks": [  
    {  
      "id": "uuid",  
      "taskType": "enum(USER|SERVICE|GATEWAY)",  
      "xAxis": "number",  
      "yAxis": "number"  
    }  
  ],  
  "sequences": [  
    {  
      "id": "uuid",  
      "fromTask": "uuid",  
      "toTask": "uuid"  
    }  
  ]  
}
```

##### `POST /forms`:

**Request**
```JSON
{  
  "name": "string"  
}
```

Creates new form

##### `POST /forms/<formId>/tasks`

**Request**
```JSON
{
  "taskType": "enum(USER|SERVICE|GATEWAY)",
  "xAxis": "number",  
  "yAxis": "number"  
}
```
 ``
Adds a new task of the specified type to the form, no data is associated with it initially, but there will be validation that something is attached to the task before its sent to the Workflow service.

##### `POST /forms/<formId>/sequences`

**Request**
```JSON
{  
  "fromTask": "uuid",  
  "toTask": "uuid"  
}
```

Adds an "edge" between two tasks. Uses "sequence" verbiage as this is [what's used by Camunda](https://docs.camunda.org/manual/7.20/reference/bpmn20/gateways/sequence-flow/).


### Task Resource Endpoints
These endpoints allow the user to attach resources to tasks such as the page that should be rendered for a user task, the integration that should be run for a service task, or conditions for a gateway.

##### `POST /forms/<formId>/tasks/<taskId>/page`

**Request**
```JSON
{
	"definition": {
		// Json Schema Page Data
	}
}
```

Adds page rendering data to a task, it accepts a JSON schema that describes the layout of a page. The Workflow Service uses [JSONForms](https://jsonforms.io/) to render the pages, but that is out of scope for this doc.


##### `POST /forms/<formId>/tasks/<taskId>/service`

**Request**
```JSON
{  
  "serviceType": "enum(HTTP|LAMBDA)",  
  "variables": {  
    "<variableName>": "location.of.data"  
  },
  "returns": {
	  "<variableName>": "dataType(string|number|boolean)"
  }
}
```

Creates a service execution for a task. Service types are tied to [Java Delegates](https://docs.camunda.org/manual/7.20/user-guide/process-engine/delegation-code/#java-delegate) that are executed when the task is reached. The `variables` and `returns` fields describe the shape of I/O for the integration point. Claims Form Service has the concept of `Data Tags` which are essentially pointers to locations of data in a form. This is out of the scope of this doc, since it really needs one of it's own.