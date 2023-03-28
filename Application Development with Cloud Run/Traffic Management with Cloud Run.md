# Traffic Management with Cloud Run [APPRUN]

In this lab, you learn to:

- Enable the Cloud Run API.

- Deploy a Cloud Run Service.

- Assign tags to a revision.

- Migrate traffic to a revision.

- Roll back a revision.

## Overview
Cloud Run allow you to specify which revisions should receive traffic and to specify the percentage of traffic to be received by a revision. This feature allows you to rollback to a previous revision, gradually roll out a revision, and split traffic between multiple revisions.
Three high-level processes are involved in resolving a technical problem.
  1. situational overview
  2. requirements gathering
  3. developing a minimal viable product

### Situational overview
In this lab, you will help the development team at Critter Junction investigate Cloud Run revisions. The team would like to explore how revision management can be incorporated into their existing development workflow.

### Requirements gathering
The operations team at Critter Junction would like to create a status page for their services without introducing additional complexity to their existing systems. Experimentation with Serverless products has led them to select Cloud Run due to the wealth of features supported.

**Defining the service priorities**

The operations team at Critter Junction are keen to define a solution that can be implemented quickly. After discussions with the engineering team, they discovered that serverless may be a good option. To move forward, a series of meetings are held with stakeholders to ascertain the key priorities. The results of which are shown below:


|Ref | User Story |
| ---| ---        |  
| 1  | As a product lead, I want to ensure the website remains responsive, so customers face minimal wait times.|
| 2  | As a developer lead, I want to increase the velocity of service deployments. |
| 3  | As an ops lead, I want to ensure system stability is observed, so the system performance is not degraded through the deployment of new revisions. |


The team leads follow up the meetings and agree the following high level tasks would indicate the project requirements have been met:

| Ref| Definition of Done |
| ---| --- |
| 1  | User base are not impacted by the rollout of new features|
| 2  | Revision management enabled for service deployments |
| 3  | New revision retains existing level of operational stability by deployment to a reduced user base. |


In consideration of the requirements, the development team decide to look in to the following:
  - Traffic Migration
  - Revision Tags

**Traffic migration versus revision tags**
| Feature | Description |
| --- | --- |
| Revision Tags | "Appropriate for use cases where a task producer needs to defer or control the execution timing of a specific webhook or remote procedure call." |
| Traffic Migration | "Cloud Run allows you to specify which revisions should receive traffic and to specify traffic percentages that are received by a revision.." |

**MVP architecture**

**Developing a minimal viable product (MVP)**

Critter Junction has a backend product status service that they would like integrated with Cloud Run. To build an MVP the following activities are required:

- Configure the environment
- Test Revision Tags
- Test Traffic Migration
- Deploy a public service

### Task 1. Configure the environment
1.  Enable Cloud Run API  
`gcloud services enable run.googleapis.com`
2. Set compute region  
`gcloud config set compute/region us-central1`
3. Create LOCATION environment variable  
`LOCATION="us-central1"`

[gcloud run services update-traffic](https://cloud.google.com/sdk/gcloud/reference/run/services/update-traffic)