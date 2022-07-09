# Google Cloud Fundamentals: Core Infrastructure

## Storage
Types of data: Structured, Unstructured, transactional, relational<br>
Cloud Storage, Cloud SQL, Cloud Spanner, Firestore, Cloud Bigtable.

#### Cloud Storage:
- Object storage, aka Binary Large-Object (BLOB) Storage.
- Not file storage, or block storage (chunks on disk).
- Object storage format consists of (1) binary form of the actual data, (2) relevant associated meta-data, (3) globally unique identifier (in the form of URLs).
- Examples: video, audio, pictures, photos.
- Organised into buckets.
- Each bucket needs unique identifier, specific geographical location (minimize latency).
- Objects stored are immutable. Means original object not edited. New version created with each change. 
	- New version can replace/overwrite original; or not (versioning control).
	- With versioning, will have archive, history trail, can restore.
- Objects may contain personal identifiable info. Control access with Identity Access Management (IAM) or Access Control List (ACL). 
    - Users only given permissions to do their job.
	- IAM inherited from project -> bucket -> object.
	- ACL for finer control. 
		- ACL consists of scope and permission.
		- scope is the who that can access and perform permission.
		- permission is the do-what, e.g. read, write or delete.
- Lifecycle policies
	- e.g. delete objects older than 365 days
	- e.g. delete objects created before DD/MM/YY
	- e.g. keep 3 most recent versions
	- e.g. move objects older than 1 year into coldline storage

---
## Storage classes and data transfer
1. Standard storage
	- frequently accessed ("hot") data
2. Nearline storage
	- data accessed 1/month or less
	- backups, archives
3. Coldline storage
	- data accessed once per quarter (3-months)
4. Archive storage
	- data accessed 1/year
	- higher cost for access read/write

Characteristics
 - unlimited storage (no minimum object size)
 - worldwide access & location
 - low latency and high durability
 - geo-redundancy if store in multi regions. Load balancing.
 - consistent experience (security, tools, APIs)
 - pay only for what you use
 - no prior provisioning of capacity
 - encrypts data on server side
 - HTTPS/TLS from customer to google endpoint

Methods to bring data into Google storage:
1. Online transfer. 
	- drag and drop on console.
	- gsutil command on Cloud SDK.
2. Storage Transfer Service.
	- schedule and manage batch transfer.
	- from another cloud provider, another cloud storage region, https endpoint.
3. Transfer Appliance.
	- rack lease from Google Cloud.
	- connect to on prem network, load data, ship to Google data centre, upload to Google Cloud.
	- up to 1 pentabyte on 1 appliance.
4. From other Google Cloud services.
	- e.g. tables from Cloud SQL, Big Query.
	- logs
	- objects used by App Engines, Firestore backups
	- VM images, instance startup scripts

---
## Cloud SQL
Relational databases as a service
 - MySQL
 - PostgreSQL
 - SQL Server

Google cloud takes care of mundane time-consuming tasks
 - apply patches/updates
 - manage backups
 - configuring replications

Characteristics
 - doesn't need software installation or maintenance
 - can scale up to 64 cores, 400+ GB of RAM, 30TB of storage
 - supports automatic replication, e.g. from primary internal/external instance
 - supports managed backups. An instance covers 7 backups.
 - encrypts customer data when on Google's internet networks and when stored in database tables, files and backups.
 - includes network firewall, control access to each database instance

Cloud SQL instances can be accessed by other Google Cloud and external services using standard MySQL drivers
 - e.g. MySQL Connector/J for Java, MySQLdb for Python
 - MySQL Workbench, TOAD for MySQL

---
## Cloud Spanner
Relational database for
 - database sizes > 2 TB
 - scales horizontally, with joins and secondary indexes
 - strongly consistent, globally
 - high availability
 - high (>10,000) input/output operations per second

 ---
 ## Firestore
 - NoSQL
- horizontally scalable
- flexible
- suitable when there is mobile, web and server development/usage

Data stored in Documents -> organized into Collection
 - documents can contain complex nested objects
 - documents can contain sub-collections

Queries
 - can combine filtering and sorting options
 - can include multiple chained filters
 - can retrieve individual documents
 - can retrieve all documents in a Collection
 - indexed by default, proportional to size of result set, or size of database

Characteristics
- data synchronised to update data on all connected devices
- cache active data so that app can write/read/listen/query even when device offline. synchronise local changes to database once device online.
- auto multi-region data replication
- strong consistency
- atomic batch operations
- realtime transaction support

Charges
- each read/write/delete documents
- query per document read
- amount of database storage
- certain network bandwidth
- ingress/egress above certain free quota

---
## Cloud Bigtable
- NoSQL Big data database
- for massive workloads
- consistent low latency
- high throughput
- for: operational & analytical applications. E.g. loT user analytics and financial data analysis

Choose Bigtable if:
- NoSQL data
- \> 1TB of semi-structured or structured data
- big data: asynchronous batch, or synchronous real time processing
- high throughput, or rapidly changing data
- time series data, or with natural semantic ordering
- ML algos

Connections to other Google Cloud services and 3rd party clients
- use APIs to connect to data service layers, e.g. managed VMs, HBase REST Server, Java Server using HBase client
	- usually for dashboards
- real-time stream. Framework examples: Dataflow, Apache Spark, Data Storm
- batch processing. Hadoop MapReduce, Dataflow, Apache Spark
- calculated/summarized data can be written back into Bigtable or a downstream database

---
## Comparing Storage options
BigQuery is for database storage and database analysis; "half half .."<br>
See table for main storage options
![Google Storage Classes](https://github.com/TCLee-tech/Google/blob/e8ce3e2764900ca24a4241196bfbc822d8466041/Google%20Cloud%20Fundamentals%20-%20Core%20Infrastructure/Storage%20comparison.jpg)

---
## Intro to Containers

IaaS: Share compute resources, virtual hardware.
- custom configurable software, e.g. runtime, web server, database, middleware
- custom configurable system resources, e.g. disk space, disk I/O, networking
- large, slow (need hardware and OS to boot), costly.
- smallest unit is 1 VM.

PaaS: App Engine
- access to programming services
- write app codes with dependent libraries only
- platform will scale workload and infrastructure independently
- scales rapidly
- but cannot fine-tune architecture to save cost
 
Containers: 
- independently scale workloads
- invisible box around code and dependencies
- abstract away hardware, OS and file system
- only need OS kernel (the essential OS core, main layer between OS and hardware) that supports containers and container runtime (codes/instructions, not written by developer, that are executed while program is running)
- application programs make system calls to create and start process (instance of program). These are program interfaces between the app and kernel. System calls to request for kernel services.
- can use same container for development, staging, production.
- can move same container from laptop, to server, to cloud.
- can scale to thousands in seconds on same host
- microservices: 1 function per container. App made up of many functional containers. 
	- connect containers with network. 
	- modular. 
	- easily deployable. 
	- start/stop containers for particular app functions.
	- scale up and down, even to zero.
	- scale across group of hosts. Can be geographically distributed for redundancy. 
	- replace containers on damaged hosts

---
## Kubernetes
- orchestration tool - manage and scale containers.
	- across many hosts
	- microservices
	- deploy rollouts and rollbacks ... scale, roll-out new version of app
- set of APIs to deploy containers across set of nodes 
- cluster divided into control plane (set of primary components) and set of nodes
- each node is a compute instance (not exactly a VM)
- declarative -> config file
- open-source

Pod
- imaginery wrapper around 1 or more containers
- smallest unit in Kubernetes you can create or deploy
- represents a running process: the entire app, or an app component
- generally, 1 container per pod
- but can have multiple containers with hard dependency on each other within a single pod, sharing networking (virtual ethernet) and storage.
- every pod has unique network IP address, and set of ports
- every pod has configurable options
- command: "kubectl run" starts deployment of container in pod
- a deployment is a group of replicas of same pod
	-> the deployment can be component of an app, or entire app
	-> kubernetes maintains deployment even if a node fails
- command: "kubectl get pods" => list of running pods
- kubernetes create service with fixed IP for group of pods
	- attach to external network load balancer with public IP
	- interact with traffic outside cluster. 
	- any traffic that reach that public IP will be routed to a pod behind the service.
	- what is service? An abstraction (i.e. codes) which defines a logical set of pods and the policies to access them.
	- pods assigned own internal IP address when created, but it is not stable over time. The service group's fixed IP address is the stable endpoint for them.
	- e.g. 2 groups of pods - "frontend" and "backend" service. A pod in "backend" service can be destroyed and replaced. To frontend, backend service is unchanged with same IP.

command: "kubectl scale"
- autoscaling of deployment
- can be configured with parameters, e.g. to scale up when CPU utilization > 80%

kubernetes' biggest advantage is that it can be declarative 
- describe desired state	
- using deployment config file
- run "kubectl get pods" to get file
- unlike imperative commands, e.g. "kubectl run" and "kubectl scale". These are ok for testing.  

To check if number of replicas deployed correctly, use "kubectl get deployments" or "kubectl describe deployments". <br>

To change number of container replicas, update number in deployment config file, then run "kubectl apply". <br>

"kubectl get services" will return service endpoints. <br>

To roll out new version of app, <br>
 - new codes, new containers
 - use "kubectl rollout" 
 - or change the number of replicas in deployment config file, then run "kubectl apply".
 - kubernetes wait for new pods to roll-out before destroying old ones. => less risky if app in production use. 

---
## Google Kubernetes Engine (GKE)

Kubernetes with some extra Google features
- load balancing for Compute Engine instances
- node pools to designate subset of nodes within a cluster for flexibility
- automatic scaling of node instance count
- auto upgrade for software running on nodes
- auto-repair to maintain node health and availability
- logging and monitoring with Cloud's operations suite for visibility on cluster

manage via console or gcloud command (SDK)
- deploy and manage applications
- perform administration tasks
- set policies
- monitor workload health
customise 
- machine type
- number of nodes
- machine setting

node - a Compute Engine instance <br>

command to startup GKE "gcloud container clusters create k1".

---
## Hybrid and multi-cloud architecture

On-prem 
- scale by spreading computing workload over 2 or more networked servers. 
- expand server hardware, network. many steps.
- takes time. months to years.
- expensive. useful life of hardware 3-5 years.

cloud 
- use containers to break down workloads into microservices
- immediate on demand availability of hardware
- fault tolerance, geo-redundancy
- reduce latency to client, central party
- use specialised products and services, only available in cloud

hybrid / multi-cloud
- move only some workloads to cloud
- migrate at own pace
- take advantage of cloud's flexibility, scalability, lower computing costs
- use specialised service, e.g. ML, AI, content caching, data analysis, long-term storage, loT toolkit

---
## Anthos
- hybrid and multi-cloud solution
- for distributed systems
- for entire software delivery life-cycle (SDLC)
- fully integrated central control plane
- framework rests on Kubernetes and GKE on-prem
	- use same configuration for containers in-cloud and on-prem
	- replicate containers anywhere
- Kubernetes on-cloud
	- high availability and SLA
	- runs certified Kubernetes for portability across cloud and on-prem
	- auto-scaling, auto-repair, auto-upgrade
	- multi-regional clusters for high availability. multiple Kubernetes control planes.
	- node storage replication across multiple regions and zones
- GKE on-prem
	- best-practice latest release version validated and tested by Google
	- production grade
	- access to container-based Google services, e.g. cloud build, container registry, cloud audit logs, cloud marketplace.
	- integrates with Istio (https://istio.io/ , service mesh for distributed or microservices architecture), kNative(https://knative.dev/docs/ , serverless containers).

- policy-based tools for consistent monitoring and maintenance, both on-prem and off-prem
	- Istio open-source service meash to manage microservice traffic, security, observability
	- fully managed logging, metrics collection, monitoring, alerting solutions
	- Anthos Configuration Management is single-source configuration management
		- policy kept in a git repository, either on-prem or in-cloud.	
		- can be enforced locally in all environments
		- can deploy code changes with single repo commit
		- can implement configuration inheritance using namespaces, preventing naming and permissions conflict.

---
## App Engine
- about developing applications
- fully managed, serverless platform for developing and hosting web applications at scale
- choose languages, libraries, frameworks to develop apps
- tools: Eclipse, IntelliJ, Maven, Git, Jenkins, PyCharm
- Google automatically provision and scale app instances based on demand
- no infra to provision and maintain
- built-in services and APIs: NoSQL datastores, memcache, load balancing, health checks, application logging, user authentication API
- has SDKs to help deploy and manage apps on local machine
	- each sdk includes
		- all APIs and libraries
		- secure sandbox environment to emulate APP Engine in cloud
		- deplyment tools to upload to cloud and manage versions
- Google Cloud Console manages apps in production
	- create new apps
	- change which version is live
	- config domain names
	- examine access and error logs
Web Security Scanner
	- automatically scan and detect app vulnerabilities
	- https://cloud.google.com/security-command-center/docs/concepts-web-security-scanner-overview
	- https://cloud.google.com/architecture/framework/security

---
## App Engine Environments
#### Standard
- container instances
- preconfigured with runtime and libraries that support app engine standard APIs
- support standardised list of languages and versions
	* Java, Python, node.js, Go, PHP, Ruby
- to use Standard environment, need to
	* use a supported langauge version
	* conform to sandbox constraints that are dependent on runtime
- features:
	* persistent storage with queries, sorting and transactions
	* automatic scaling and load balancing
	* asynchronous task queues for performing work outside scope of a request
	* scheduled tasks for triggering events at specified times or regular intervals
	* integrate with other Google Cloud services and APIs
	- applications run in secure sandbox environment
		* irregardless of hardware, OS and geographical location
		* Google distributes containers and scales servers to meet traffic demand
- workflow
	1. develop web app locally
	2. deploy to App Engine with SDK
	3. App Engine scales and services the app

#### Flexible
- docker containers in Compute Engine VMs
- App Engine manages the VMs
- features:
	* VM instances health checked, healed and co-located - for optimal performance.
	* updates automatically applied to OS
	* VM instances automatically located in geographical region according to project settings
	* VM instances restarted weekly. OS and security updates applied during restart.
- supports:
	* microservices
	* authorization
	* SQL and NoSQL databases
	* traffic splitting
	* logging
	* search
	* versioning
	* security scanning
	* memcache
	* content delivery networks
- possible customization:
	* custom configurations and libraries, while focusing on writing code
	* custom runtime and OS
	* custom language versions, beyond what's supported by Standard environment
	* provide custom Docker image or a Dockerfile from open-source community

|--- | Standard environment | Flexible environment |
|--- | ---                  | ---                  |
|Instance startup | seconds | minutes|
| SSH access | no | yes (not default) |
|write to local disk | no (some runtimes have read + write access to /tmp directory) | yes (ephemeral disk} |
| support for 3rd party binaries | yes, for some languages | yes |
| network access |  via App Engine services | yes |
|pricing model | after free tier, pay per instance, with auto shutdown | pay for resource allocation per hr, no auto shutdown |

App Engine vs GKE
- App Engine Standard environment - for owners who want auto deployment and scaling
- GKE gives full flexibility of Kubernetes

---
## Google Cloud API management tools
2 tools: Cloud Endpoints and Apigee Edge <br>
API
- a clean, well-defined, documented interface. 
- hides the complexity and changes of server-side implementation.
- can be versioned. add/deprecate features

#### Cloud Endpoints
 - a distributed API management system
 - a distributed, extensible service proxy 
 - proxy runs in its own docker container
 - for low latency, high performance connections
 - provides API console, hosting, logging, monitoring and other services
 - helps developer to create, share, maintain, secure APIs
 - supports APIs that meet OpenAPI specification
 - supports applications that run on App Engine, GKE, Compute Engine
 - clients include Android, iOS, Javascript

#### Apigee Edge
 - specific focus on business problems
	- e.g. having quota on API calls, rate limiting, analytics
 - many Apigee Edge users provide software service to clients
 - backend services do not need to be on Google Cloud
	- can be used for functional decomposition of legacy applications
	- presents a consistent interface to clients
	- peels away services/functions into microservices
	- until entire legacy application can be retired

---
## Cloud Run
 - managed compute platform
 - serverless stateless containers
 - no infrastructure. focus on developing applications		
 - built on Knative, an open API and runtime environment based on Kubernetes
 - Docker container nodes (workloads) can move across platform and environment easily.
 - auto-scaling. can scale to zero. 
 - fast, almost instantaneous
 - pay only for resources used
	- during Startup + Handling requests + Shutdown
	- calculated to within 100ms
	- not billed if no requests handled
	- extra small fee per million requests handled
	- container with more vCPU and memory is more expensive
	- max is 4 vCPU and 8GB memory
	- not paying for idle server capacity (VMs)
 - can run for any binaries compiled for Linux 64-bit:
	- source code
	- system packages
	- library dependencies
	- runtime
 - acceptable programming languages:
	- Java
	- Python
	- node.js
	- PHP
	- Go
	- C++
	- Cobol, Haskell, Perl

#### Developer workflow
1. write source code
	- should start a server that listens for web requests
2. build and package application codes into container image
3. push container image to artifact registry, or deploy with Cloud Run
	- Cloud Run can only deploy applications stored in container/artifact registry
	- can build, push and deploy from local machine if have permissions
4. Cloud Run 
	- returns unique HTTPS URL for client connection
	- starts containers on demand to handle web requests or Pub/Sub events
	- ensures all incoming requests handled, by dynamically adding/removing containers
	- handles HTTPS connection: generating valid SSL certificate, configuring SSL termination correctly, handling incoming requests, decrypting, forwarding to application.

Flow:	client --HTTPS--> https://***.run.app or https://your.domain --> Cloud Run Proxy --HTTP--> Container

#### Two types of workflow
1. Container-based workflow
	- transparency
	- flexibility
	- developer determines what files go inside container image
2. Source-based workflow
	- write source code -> deploy codes to Cloud Run
	- Cloud Run uses Buildpacks (https://buildpacks.io/) to package codes into container image -> web app
	- ensures container image is correctly configured, built in consistent way, secure.

---
## Developing and Deploying in the Cloud
### Development in the Cloud
|Development in the cloud | Deployment |
|---|---|
| - Cloud source repositories|  - Infrastructure as a code |
| - Cloud functions| |
| - Terraform | |

Many developers use git repositories to
 - store
 - manage
 - version
 - may run own Git instances if total control required
 - may use hosted-Git provider if total control not required

Cloud-source repositories
 - repo hosted on Google Cloud
 - codes are private to your cloud projects
 - IAM for access control
 - supports collaborative development of applications and services on Google Cloud
 - unlimited number of private git repositories
 - Google's diagnostic tools, e.g. debugger and error reporting, can be applied on code base
 - migration support for source codes already in Git or Bitbucket

Cloud Function
 - lightweight and small.
 - aynchronous single-purpose event-based that respond to Cloud Storage, web or pub/sub events
 - synchronous execution with HTTP invocation
 - compute solution with no need to manage server or runtime environment
 - can use to construct applications from bite-sized business logic, then connect and extend to cloud services
 - billed to nearest 100ms, only when codes running
 - write in Javascript (node.js), python or Go. manage in node.js environment on Google Cloud.
 - example: upload of image to Cloud Storage triggers cloud function to resize image, convert to standard format and store in specific directories.

---
### Deployment: Infrastructure as Code
Terraform
 - write a template to specify components of infrastructure
 - deploy to scale and create as many identical application environments as needed
 - use HashiCorp Configuration Language (HCL)
 - if need to update infra environment, update the template
 - store and version-control Terraform in Cloud Source Repositories
 - more efficient
 - labour intensive if need to manually configure hardware via console or command-line interface

---
## Logging and Monitoring in the Cloud
### The importance of monitoring
Monitoring is the foundation of product reliability
 - reveals what needs urgent attention
 - shows trends in application usage patterns
 - helps capacity planning
 - helps improve application experience

In Google Site Reliability Engineering book
 - landing.google.com/sre/books
 - monitoring is about collecting, processing, aggregating, displaying real-time quantitative data on a system
	-e.g. query counts and types
	- error counts and types
	- processing times
	- server lifetimes
Use
 - ensure continued system operations, improvement with data
 - uncover trend analyses over time
 - build dashboards for DevOps
 - alert key personnel when systems violate predefined SLOs. 
 	- automated system for filtering alerts to the most urgent/important ones.
 - compare systems and system changes, debug
 - provide data for improved incident response

Pyramid of monitoring
 - Product client sees
 - developing
 - capacity planning / deploy 	      
 - testing / CICD 		      
 - postmortems / root cause analyses 	
 - incident response / failure / security breach
 - monitoring

---
## Measuring performance and reliability
4 golden signals
- latency
- traffic
- saturation
- errors

Latency
 - measure of how fast to return result
 - directly affects user experience
 - change in latency could indicate emerging issues
 - values tied to capacity demand
 - can be used to measure system improvements

How to measure latency
 - page load
 - number of requests waiting for a thread
 - query duration
 - service response time
 - transaction duration
 - time to first response
 - time to complete data return	

Traffic
 - measures how many signals reaching system
 - important because
	- indicator of current system demand
	- historical trends are used for capacity planning
	- core measure when calculating infrastructure spend
 - measures
	- \# HTTP requests per second
	- \# requests for static vs dynamic content
	- network I/O
	- \# concurrent sessions
	- \# transactions per second
	- \# of retrievals per second
	- \# of active requests
	- \# of read operations
	- \# of write operations
	- \# of active connections

Saturation
 - how close to capacity a system is
 - important because saturation:
	- is an indicator of how full the system is
	- focuses on the most constrained resources
	- frequently tied to degrading performances as capacity is reached
 - some capacity metrics
	- % memory utilisation
	- % thread pool utilisation
	- % cache utilisation
	- % disk utilisation
	- % CPU utilisation
	- disk quota
	- memory quota
	- \# of available connections
	- \# users on the system

Errors
 - events that measure system failures or other issues
 - fault causing incorrect/unexpected results or behave in unintended ways
 - important because may indicate
	- something is failing
	- configuration or capacity issues
	- service level objective violations
	- time to send out an alert
 - metrics
	- wrong answers or incorrect content
	- \# 400/500 HTTP codes
	- \# failed requests
	- \# exceptions
	- \#stack traces
	- servers that fail liveness checks
	- \# dropped connections

---
## Understanding SLIs, SLOs, and SLAs
- measures of user experience
- use as primary driver for decision making

### Service Level Indicator
 - carefully selected monitoring metrics
 - monitor service reliability
 - number of good events / count of all valid events
 - correlation with user experience

### Service Level Objective
 - combines service level indicator with a target reliability
 - usually just short of 100%, e.g. 99.5%, 99.9%
 - Specific - quantitative
 - Measurable - a number or delta. can be equation.
 - Achievable - 100% cannot be maintained over time
 - Relevant - to user. will it help achieve application goals?
 - Time-bound - 99.9% .. per month or per year? Include weekends in a week? rolling period?
 - concrete actions must be taken if not meeting SLOs, e.g.
   	- slow down rate of change
	- engineering effort towards eliminating risks
	- improve reliability

#### Service Level Agreement
 - commitments to customers
 - that systems and applications will only have certain amount of down time (outage)
 - minimum service level
 - compensation to customers if SLA breached
 - protect reputation
 - alerting thresholds (e.g. 99.9%) set above SLA minimum service level (e.g. 99.5%)

---
## Integrated observability tools
 1 monitoring
 2 logging
 3 error reporting
 4 debugging
- can physically inspect hardware on-prem, not in cloud

Process
 1. Capture signals
	- from hardware and software
	- metrics from apps, services, platform, microservices
	- logs from apps, services, platform
	- trace from apps
 2. Visualize and analyze signals
	- dashboards
	- metrics explorer
	- logs explorer
	- service monitoring - of SLOs and track error budgets
	- health checks of uptimes and latency for sites and services
	- debugger
	- profiler
 3. Manage incidents
	- alert to key personnels - can be automated
	- error reporting - spot, count and analyze errors
	- SLO - to be adhered to
 4. Troubleshoot

---
## Monitoring
 - signal data -> measurements -> maths calculation -> align data over time
	- e.g. raw CPU usage -> average -> value per minute
 - \> 1000 streams of metric data available by default on Google Cloud, for incorporation into dashboards etc
 - some examples of data points
 	- run massive scalable queries in BigQuery
		- want to know queries in flight, scanned bytes billed, data slots used
	- run containerized applications in Cloud Run
		- want to know CPU utilization, billable time, memory utilization
	- own applications
		- can use OpenTelemetry (https://opentelemetry.io/) for custom metrics
	- compute instances
		- CPU and memory utilization, uptime, disk throughput
 - provides visibility into the performance, uptime and overall health of cloud apps
 - collects metrics, events, metadata, logs, services, systems, agents, custom code
 - from application components, e.g. Cassandra, Nginx, Apache Web Server, ElasticSearch
 - ingests data and generate insights to dashboards, metrics explorer, charts, automated alerts
	
---
## Logging tools
Cloud logging allows
- collect
- store
- search
- analyze
- monitor
- alert
of log entries and events.

Auto logging is enabled for Google Cloud products, e.g. App Engine, Cloud Run, Compute Engine VMs running logging agent, GKE.

Log analysis
 - uses Google Cloud's integrated Logs Explorer
 - entries can be exported for alternative analysis or further analysis
 - pub/sub messages can be analysed in near real-time using custom code or stream processing, e.g. dataflow
 - examine logging data with SQL queries using BigQuery
 - archived log files in Cloud Storage can be analyzed

Log export
 - as files to Cloud Storage
 - as messages through Pub/Sub
 - into BigQuery tables
 - log-based metrics can be created and integrated into dashboards, alerts and service SLOs.

Log retention
 - different log type has different default duration
 - data access logs retained for default 30 days, max 3,650 days
 - admin logs retained for default 400 days
 - logs can be exported to Cloud Storage or BigQuery to extend retention

Type of log visible in Cloud Logging depends on Cloud product used. 

4 key types of logs:
1. Cloud audit logs
	- who did what, where and when?
	- admin activity logs track config changes
	- data access logs track calls that read the config or metadata of resources
	- data access logs track user-driven calls that create, modify or read user-provisioned	resource data
	- system event - non-human Cloud admin actions that change config of resources
	- access transparency - log actions google staff take when accessing your content
2. Agent logs
	- Fluentd agent (https://www.fluentd.org/). 
		- Cloud Logging version customized and packaged by Google.
		- can be installed on any Google Cloud or AWS VM/container to ingest data from Google Cloud instances
3. Network logs
	- provide network security telemetry
		- for network and security operations
		- VPC flow logs record network flow
		- can be use for network monitoring, forensics, real-time security analysis, expense optimization.
	- firewall rules
		- logs allow one to audit, verify and analyze effects of firewall rules
	- NAT gateway
		- info on network connections and errors
4. Service logs
	-Standard out/error
	- e.g. container in node.js -> deploy to Cloud Run -> any output to standard out/error will be logged

---
## Error reporting and debugging tools
Error reporting
 - counts, analyzes, aggregates the exceptions (crashes) in running Cloud services
 - management interface displays results with sorting and filtering capabilities
 - dedicated view for error details: time chart, occurrences, affected user count, first- and last-seen dates, cleaned exception stack trace.
 - create alerts to receive notifications on new errors

Debugger
 - debug while application running in production to examine code's function and performance
 - collaborate with other team members by sharing debug sessions with a Console URL
 - application's state in production can be captured at a specific line location with snapshots
 - integrate into existing developer workflows
 - integrates with version control systems, e.g. Cloud Source Repositories, GitHub, Bitbucket, GitLab.

Cloud Trace
 - collects latency data from distributed Google apps
 - captures traces from applications deployed on App Engine, VMs, Kubernetes containers.
 - performance insights are in near real-time
 - auto analyze all applications' traces to generate detailed latency reports. Can detect performance degradations.
 - continuously gather trace data to auto identify recent changes to application's performance

Cloud Profiler
 - uses statistical techniques and extremely low-impact instrumentation that runs across all production app instances. provides complete CPU and heap picture of app.
 - allows developers to analyze apps running anywhere, e.g. Google Cloud, other Cloud platforms, on-prem. Supports Java, python, Go, node.js
 - presents call hierachy and resource consumption for relevant function in an interactive flame graph. see what is consuming resources. know the different ways the function is called.
