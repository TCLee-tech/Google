#### Qwiklabs: Confluent: Developing a Streaming Microservices Application
https://github.com/confluentinc/examples
https://github.com/confluentinc/kafka-streams-examples


#### Monolithic (3-tier, MVC) to modern architecture
##### modern:
 - **cloud-based**
	- off-prem monolithic
	- involves, application servers, web servers, databases, load balancers.
	- leverages on some cloud features, e.g. high availability, scalability, software-defined networking, auto-provisioning.
	- API Gateway for consistent API UI.
 - **cloud native**
	- is about how applications are created, not where.
	- follow 12-factors methodology (https://12factor.net/)
	- containers based
	- elements of this architecture:
		- REST-based messaging
		- API Gateway
		- containers registry
		- messaging middleware (pub/sub)
		- service mesh
		- containers orchestration
 - **microservice**
 	- sidecar, ambassador, adapter framework
	- containers, encapsulate dependencies
	- 1 container, 1 business logic/function/service
 	- containers orchestration: Kubernetes (Red Hat OpenShift, Azure AKS, GKE, AWS EKS/ECS/Fargate, IBM Kubernetes)
	- Istio - service mesh
 - **event-driven serverless**
	- architecture is event-centric. Services respond to events
	- decoupled services
	- services are serverless cloud functions
	- inter-service communication via event streams (and Kafka's Connect API)
	- rapid scaling, from zero, to zero.
	- stateless
	- Google Cloud Functions, Amazon Lambda, Azure Functions, IBM Cloud Functions, Apache OpenWhisk
	- types of serverless: Functions-as-a-Service(FaaS), Backend-as-a-Service(BaaS), Mobile-Backend-As-A-Service(MBaaS)
https://www.ibm.com/cloud/blog/four-architecture-choices-for-application-development

#### Command Query Responsibility Segregation (CQRS)
A software architectural pattern
 - seperates read tasks from write tasks
 - usually, more reads than writes
 - use multiple database replicas for reading only, distributed over geolocations, closer to users, lower latency, more efficient scaling.
 - fewer for writing
 - do not need same type of database for reading and writing
 - cons/risks: cost, need database expertise, data consistency, more points of failure
https://www.redhat.com/architect/pros-and-cons-cqrs
