### Distributed Load Testing using Kubernetes

To learn: How to use Kubernetes Engine to deploy a distributed load testing framework.
  - use multiple containers to create high-volume scalable traffic for load testing
  - can be use to test
    - REST-based API website / service
    - gaming
    - Internet-of-Things (IoT) applications  

[Google Cloud Architecture Center Reference](https://cloud.google.com/architecture/distributed-load-testing-using-gke)  
[Github](https://github.com/GoogleCloudPlatform/distributed-load-testing-using-kubernetes)

#### The sample application tested in this lab
  - an application with REST-based endpoints to receive HTTP requests.
  - this app is modeled after backend service component for many IoT deployments.
    - devices will register with service -> then report sensor readings or metrics, periodically re-register with service.

#### Load testing tool
  - [Locust](https://locust.io/)
    - open-source, Python-based
    - distributed, scalable - simulates concurrent users 
    - can swarm multiple target paths, e.g. /login, /metrics
    - workload modeled as set of @Tasks
    - can be event-driven by .add_listener
  - container-based computing
    - Locust software packaged as Docker image
    - a container cluster consists of a master plane and worker nodes. Orchestrated by Kubernetes.
    - a pod is the smallest compute unit that can be defined, deployed and managed.
      - can contain 1 or more Locust containers.
    - a Deployment Controller provides declarative updates for Pods and ReplicaSets.
      - this lab has 2 deployments: locust-master and locust-worker.
    - a Service is an interface abstraction, with a defined set of logical pods and a policy for accessing them.  
      - a consistent IP address for service.
      - if pods are destroyed and recreated during node failures, updates or maintenance
          -> pod IP address will be different (dynamic).

#### Task 2. Get the sample code and build a Docker image for the application
`gsutil -m cp -r gs://spls/gsp182/distributed-load-testing-using-kubernetes .`
  - syntax: `gsutil cp [options] src_url dst_url`
  - [cp](https://cloud.google.com/storage/docs/gsutil/commands/cp) : copy data
  - **-m**: multi-threaded / multi-processing parallel copying
  - **-r**: to copy entire directory tree
  - **.** refers to present directory  

`gcloud builds submit --tag gcr.io/$PROJECT/locust-tasks:latest docker-image/.`
  - syntax: `gcloud builds GROUP|COMMAND [flags]`
  - submit a build using Google Cloud Build
  - When the builds/use_kaniko property is True, builds submitted with --tag will use [Kaniko](https://github.com/GoogleContainerTools/kaniko) to execute builds
    - Kaniko does not need Docker daemon
    - executes each command inside Dockerfile
    - builds image
    - pushes to an image registry

[Skaffold](https://skaffold.dev/) for continuous deployment of containers
  - watch source code repo -> build artifacts -> test -> tag policies -> push -> deploy -> monitor -> clean-up

#### Task 3. Deploy web application
`gcloud app deploy sample-webapp/app.yaml`
  - [gcloud app deploy](https://cloud.google.com/sdk/gcloud/reference/app/deploy) [deployable yaml/xml/jar file]
  - deploy code and configuration of an application to App Engine

#### Task 4. Deploy Kubernetes cluster
```
gcloud container clusters create $CLUSTER \
  --zone $ZONE \
  --num-nodes=5
```

#### Task 6. Deploy locust-master
1. Replace [TARGET_HOST] and [PROJECT_ID] in locust-master-controller.yaml and locust-worker-controller.yaml with the deployed endpoint and project-id respectively.  
  `sed -i -e "s/\[TARGET_HOST\]/$TARGET/g" kubernetes-config/locust-master-controller.yaml`
    - [sed stands for stream editor in UNIX](https://www.educba.com/sed-command-in-unix/)
    - most common use: find and replace (substitution), insert, delete
    - **-i**: to edit source file. By default, source file not edited. 
    - **-e**: execute the commands
    - **s**: substitute
    - syntax: /pattern/action
      - pattern: regular expression
      - action: command to perform
      - **/**: delimiter
    - \ : escapes [ and ] special characters
    - **/g**: global replacement. Replace all occurences of [TARGET_HOST] with $TARGET 

2. Deploy locust-master  
`kubectl apply -f kubernetes-config/locust-master-controller.yaml`  

3. To verify, `kubectl get pods -l app=locust-master`  
    - -l: "label" flag
    - app=locust-master is the label on pods  
4. Deploy locust-master-service  
`kubectl apply -f kubernetes-config/locust-master-service.yaml`
    - service ensures exposed ports available to other pods via hostname:port within cluster, and can be referred via descriptive port name.
    -  The use of a service allows the Locust workers to easily discover and reliably communicate with the master, even if the master fails and is replaced with a new pod by the deployment. 
    - __type: LoadBalancer__ in yaml file will direct GKE to create Compute Engine forwarding rule from publicly available IP to the locust-master pod, hence accessing cluster resources.
5. To view external_IP  
`kubectl get svc locust-master`

#### Task 7. Deploy locust-workers
Single deployment, multiple pods across entire cluster.  
1. Deploy locust-worker-controller  
`kubectl apply -f kubernetes-config/locust-worker-controller.yaml`  
2. To verify, `kubectl get pods -l app=locust-worker`
    - -l app=locust-worker is the flag to "label" [label to match on pod]
3. Manually scale up number of pods to simulate more users without redeployment  
`kubectl scale deployment/locust-worker --replicas=20`
4. To verify, `kubectl get pods -l app=locust-worker`

#### Task 8. Execute tests
1. Get external IP address
```
EXTERNAL_IP=$(kubectl get svc locust-master -o yaml | grep ip | awk -F": " '{print $NF}')
echo http://$EXTERNAL_IP:8089
```
- [kubectl get svc](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get) 
  - **-o yaml**: output format, as yaml file.
- [grep](https://www.man7.org/linux/man-pages/man1/grep.1.html) searches for pattern and prints each line that matches pattern
- [awk](https://www.howtogeek.com/562941/how-to-use-the-awk-command-on-linux/) is an awkward old language for text processing
  - for extracting and transforming text, from files or as part of pipeline.
  - entire awk program is enclosed in single quotes(')
  - **-F**: separator string. To use `: ` to separate fields in lines of text. Default separator is whitespace.
  - a `pattern {action}` statement.
  - Fields are identified by "$" sign. `NF` stands for Number of Fields. `$NF` represents the last field.
  
