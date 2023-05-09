# Using Cloud PubSub with Cloud Run [APPRUN]

### Overview
Cloud PubSub enables applications to take advantage of efficient message queues. The service is compatible with a range of services and in this lab, learn how to use it with Cloud Run.

Lab content is based on resolving a customer use case through the use of serverless infrastructure. The lab features three high level sections that resolve a technical problem:

- Situational Overview
- Requirements Gathering
- Developing a Minimal Viable Product

### Objectives
Learn to:
- enable the Cloud Run API
- deploy a Cloud Run service
- deploy a PubSub service
- create a PubSub subscription

### Situational overview
In this lab, you will help the development team at Critter Junction investigate Cloud PubSub. The dev team would like to explore how to perform efficient queue processing within their applications.

#### Requirements gathering
The team at Critter Junction has a public web application and several microservices built on Google Cloud. Communication between the microservices is critical and needs a resilient form of messaging to be established between each application component.

The dev team's previous attempts were unsuccessful due to the microservices needing to know a lot about each other (high coupling). In addition, if a service was temporarily unavailable, messages would be lost.

The team needs a solution that includes a level of resilience without introducing additional service dependencies (i.e. low coupling) into their systems. Now that you know a bit more about Critter Junction and the issues they face, try and prioritise the key criteria for a  solution.

#### Defining Critter Junction priorities
Initial discussions with the Critter Junction stakeholders to ascertain the key priorities are held. The results of which are shown below:
| Ref | User Story |
| --- | --- |
| 1 | As a developer lead, I want to ensure messaging is resilient, so service operations will be restored without needing manual intervention. |
| 2 | As a program manager, I want services to be capable of scaling seamlessly, so additional transactional load does not lead to system instability. |
| 3 | As an ops lead, I want services to be managed, so I don't need to reassign staff from important maintenance work. |

From a discussion with the team leads, the following high level tasks are defined:
| Ref | Definition of Done |
| --- | --- |
| 1 | Establish an asynchronous component for inter-service communication |
| 2 | Implement proven scalability solution |
| 3 | Services must run unsupervised |

The team at Critter Junction are keen to define a solution that can be implemented quickly. The dev team narrowed down to two options:
- Cloud PubSub
- Cloud Tasks  

[PubSub vs Tasks](https://cloud.google.com/pubsub/docs/choosing-pubsub-or-cloud-tasks)

| Product | Use case | Choice |
| --- | --- | ---|
| PubSub | Optimal for general event data ingestion and distribution patterns where some degree of control over execution can be sacrificed. | <== |
| Cloud Tasks | Suitable for use cases where a task producer needs to defer or control the execution timing of a specific webhook or remote procedure call. | X |

PubSub choosen because they only require a push-based distribution pattern. Following high-level architecture diagram summarizes the Minimal Viable Product (MVP) they wish to investigate. PubSub is used to handle asynchronous messages between services.
![PubSub on Cloud Run](https://github.com/TCLee-tech/Google/blob/00b05dbcbfe34eeaaac1a0c741b628d9fe2bba7c/Application%20Development%20with%20Cloud%20Run/Using%20PubSub%20with%20Cloud%20Run%20lab%201.jpeg)

<hr>

### Task 1. Ensure that the Cloud PubSub API is successfully enabled
Restart connection to the Cloud PubSub API.
1. In the Cloud console, enter "Cloud PubSub API" in the top search bar.
2. Click on the result for **Cloud Pub/Sub API**
3. Click **Manage** > **Disable API**
4. If asked to confirm, click **Disable**
5. Again, when prompted `Do you want to disable Cloud Pub/Sub API and its dependent APIs?`, Click **Disable**.
6. Click **Enable**.

<hr>

### Task 2. Developing a Minimal Viable Product (MVP)
Critter Junction has multiple Cloud Run services that they would like integrated with Cloud Pub/Sub. To build a MVP, the following tasks are required:
- deploy a producer service
- deploy a consumer service
- create a service account
- create a PubSub queue

#### Deploy a producer service
Critter Junction specifies that the external facing Store Service should be configured as a public endpoint, meaning:
| Type | Permission | Description |
| --- | --- | --- |
| URL access | --allow-unauthenticated | Make the service PUBLIC (i.e. unauthenticated user can access). |
| invoke permission | allUsers | Allow the service to be invoked/triggered by anyone. |

The producer service accepts all the internet-based connections for orders. To do this, the service is defined as not requiring authentication and can be trggered by anyone.
Information collected by this service will be passed to the backend consumer services.

1. Enable Cloud Run API and configure your Shell environment.  
`gcloud services enable run.googleapis.com`  
2. Set compute region.  
`gcloud config set compute/region us-central1`  
3. Create a LOCATION environment variable.  
`LOCATION="us-central1"`  
4. Deploy the Store Service.  
```
gcloud run deploy store-service \
    --image gcr.io/qwiklabs-resources/gsp724-store-service \
    --region $LOCATION \
    --allow-unauthenticated
```
Once "store-service" is deployed, it is publicly accessible via its default URL.  

#### Deploy a consumer service
The dev team also need to configure Order Service as a private endpoint. Unlike the frontend Store Service, the Order Service is not meant to be publicly accessible over the internet and should be provisioned with least privilege permissions. For Cloud Run services, this can be achieved by using the following settings:

| Type | Permission | Description |
| --- | --- | --- |
| URL access | --no-allow-unauthenticated | Make the service PRIVATE (i.e. only authenticated users can access). |
| Invoke permission | none | Do not allow the service to be invoked/triggered by anyone. |

1. Configure and deploy `order-service`:
```
gcloud run deploy order-service \
    --image gcr.io/qwiklabs-resources/gsp724-order-service \
    --region $LOCATION \
    --no-allow-unauthenticated
```
Upon deployment, only authenticated accounts can access this service. In addition, `order-service` can only be used by an account with **Cloud Run Invoker** role.

<hr>

### Task 3. Deploy Cloud Pub/Sub
**overview**
Google Cloud Pub/Sub is an asynchronous messaging service that decouples services that produce events from services that process events.

Cloud Pub/Sub core concepts:
- topic
- subscription
- message
- message attribute

| Field | Description |
| --- | --- |
| Topic | A named resource to which messages are sent by publishers. |
| Subscription | A named resource representing the stream of messages from a single specific topic to be delivered to the subscribing application. For more details on subscriptions and message delivery sematics, see the [Subscriber Guide](https://cloud.google.com/pubsub/docs/subscriber) |
| Message | The combination of data and (optional) attributes that a publisher sends to a topic and is eventually delivered to subscribers. |
| Message attribute | A key-value pair that a publisher can define for a message. For example, key **iana.org/language_tag** and value **en** could be added to messages to mark them as readable by an English-speaking subscriber. |

Cloud Pub/Sub can be used in a wide variety of use cases. The most common are listed below:
| Use Cases | Example |
| --- | --- |
|Balancing workloads in network clusters | For example, a large queue of tasks can be efficiently distributed among multiple workers, such as Google Compute Engine instances. |
| Implementing asynchronous workflows | For example, an order processing application can place an order on a topic, from which it can be processed by one or more workers. |
| Dsitributing event notifications | For example, a service that accepts user signups can send notifications whenever a new user registers, and downstream services can subscribe to receive notifications of the event. |
| Refreshing distributed caches | For example, an application can publish invalidation events to update the IDs of objects that have changed. |
| Logging to multiple systems | For example, a Google Compute Engine instance can write logs to the monitoring system, to a database for later querying etc. |
| Data streaming from various processes or devices | For example, a residential sensor can stream data to backend servers hosted in the cloud. |
| Reliability improvement | For example, a single-zone Compute Engine service can operate in additional zones by subscribing to a common topic, to recover from failures in a zone or region. |

Now that the producer (`store-service`) and consumer (`order-service`) services have been successfully deployed, focus on PubSub.  
PubSub requires 2 activities:
- creating a topic
- creating a subscription

**Creating a Topic**  
When an asynchronous (push) event is created, the application will be able to process the associated messages. [Push event processing with Pub/Sub](https://cloud.google.com/run/docs/triggering/pubsub-push) provides a scalable way to handle messaging on Google Cloud.   
With a PubSub topic, messages are now independently stored in a resilient manner.  

Create a new PubSub topic with the following values:
| Field | Value |
| --- | --- |
| Name | ORDER_PLACED |
| Encryption | Google-managed key |

NOTE: Messages sent via PubSub are encrypted as base64 on transmission and need to be decoded on receipt.

`gcloud pubsub topics create ORDER_PLACED`

<hr>

### Task 4. Create a Service Account.
When an event is triggered, a Service Account is needed to invoke backend Cloud Run "order-service". To enable the Service Account to perform this task, the following activities are required:
- create Service Account
- bind Invoker role with its permissions
- create PubSub subscription

**Service Account creation** 
1. Create a Service Account `pubsub-cloud-run-invoker` that will provide authenticated access.  
`gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "Order Initiator"`  
2. Confirm that the Service Account has been created.  
`gcloud iam service-accounts list --filter="Order Initiator"`  
 
 
**Bind role that has required permissions**  
Need to explicitly bind role with required permissions to invoke Cloud Run. To bind role/permissions to an identity on Cloud Run, you need to know:  
| Category | Description |
| --- | --- |
| Service Name | Name of deployed Cloud Run service to access. |
| Member | The service account bound to the role. |
| Region | The region in which the service is deployed. |
| Platform | Type of platform, e.g. Cloud Run managed, Cloud Run on Anthos, Cloud Run for VMWare. |

1. Bind the Service Account with `Cloud Run Invoker` role, attach to Cloud Run "order-service":
```
gcloud run services add-iam-policy-binding order-service \
    --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --role=roles/run.invoker
    --platform managed
```

Syntax:
>gcloud run services add-iam-policy-binding SERVICE \
>   --member=serviceAccount:SERVICE_ACCOUNT_NAME@PROJECT_ID.iam.gserviceaccount.com \
>   --role=roles/run.invoker
>   --platform managed

2. Add the project number to an environment variable:
```
PROJECT_NUMBER=$(gcloud projects list \
    --filter="qwiklabs-gcp" \
    --format='value(PROJECT_NUMBER)')
```
3. Grant Service Account access to the project so that it has permission to complete actions on resources in project:
```
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --role=roles/run.invoker
```

<hr>

### Task 5. Create a PubSub subscription.
1. First, add the backend Cloud Run "order-service" API endpoint to an environment variable:
```
ORDER_SERVICE_URL=$(gcloud run services describe order-service \
    --region $LOCATION \
    --format="value(status.address.url)")
```
2. Then, create the subscription:
```
gcloud pubsub subscriptions create order-service-sub \
    --topic ORDER_PLACED \
    --push-endpoint=$ORDER_SERVICE_URL \
    --push-auth-service-account=pubsub-cloud-run-invoker@GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
```

<hr>

### Task 6. Testing the application
To test the application, send a sample schema to the frontend store service.
1. Create a file `nano test.json`
2. Add the following content:
```
{
  "billing_address": {
    "name": "Kylie Scull",
    "address": "6471 Front Street",
    "city": "Mountain View",
    "state_province": "CA",
    "postal_code": "94043",
    "country": "US"
  },
  "shipping_address": {
    "name": "Kylie Scull",
    "address": "9902 Cambridge Grove",
    "city": "Martinville",
    "state_province": "BC",
    "postal_code": "V1A",
    "country": "Canada"
  },
  "items": [
    {
      "id": "RW134",
      "quantity": 1,
      "sub-total": 12.95
    },
    {
      "id": "IB541",
      "quantity": 2,
      "sub-total": 24.5
    }
 ]
}
```
**CTRL+X** then **Y** to save and exit nano editor.

2. Add the frontend "store-service" endpoint to an environment variable:
```
STORE_SERVICE_URL=$(gcloud run services describe store-service \
    --region $LOCATION \
    --format="value(status.address.url)")
```

3. Post a message to the frontend store-service:  
`curl -X POST -H "Content-Type: application/json" -d @test.json $STORE_SERVICE_URL`  

4. To verify, check for `ORDER ID` in log of frontend store-service (public endpoint) and backend order-service (private endpoint):    
Go to **Navigation menu** > **Cloud Run** > select **store-service** > **LOGS** > add the log filter `ORDER ID` to see the ID generated by the frontend store-service.  

![store-service](https://github.com/TCLee-tech/Google/blob/721b73c5a693ca7cd1015f9183af3f5ff10dfed5/Application%20Development%20with%20Cloud%20Run/Using%20PubSub%20with%20Cloud%20Run%20lab%202.jpeg)

The frontend store-service will publish a message to Cloud Pub/Sub. Message will be delivered to backend order-service. To read the message, the payload needs to be decoded from base64.  

Go to **Navigation menu** > **Cloud Run** > select **order-service** > **LOGS** > add the log filter `Order Placed` to see the ID passed from the frontend store-service.

![order-service](https://github.com/TCLee-tech/Google/blob/6ec012801704f1625f19eccb4547e5820f930634/Application%20Development%20with%20Cloud%20Run/Using%20PubSub%20with%20Cloud%20Run%20lab%203.jpeg)  

<hr>

### Task 7. Revised architecture
The high-level architecture using PubSub between frontend and backend Cloud Run services:
![revised architecture](https://github.com/TCLee-tech/Google/blob/7c64743fe89dd69d6547823970bf6ddde8258bf0/Application%20Development%20with%20Cloud%20Run/Using%20PubSub%20with%20Cloud%20Run%20lab%204.jpeg)

[Severless Expeditions](https://www.youtube.com/playlist?list=PLIivdWyY5sqJwq_pgOxcHzusWjXDVCEiX)



