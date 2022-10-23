## Orchestrating the Cloud with Kubernetes
Lab is to learn how to:
* provision complete Kubernetes cluster using Kubernetes Engine.
* deploy and manage Docker containers using kubectl
* break application into microservices using Kubernetes Deployments and Services

Video
https://drive.google.com/file/d/19y4zO9WCk7n7RF1nIBDj-QgapL9sVvup/view?usp=sharing

Example 12-Factor application. 
https://github.com/kelseyhightower/app
Monolith to be re-factored into microservices
    - monolith docker, includes auth and hello services https://hub.docker.com/r/kelseyhightower/monolith
    - auth microservice, docker, generates JWT tokens for authenticated users. https://hub.docker.com/r/kelseyhightower/auth
    - hello microservice, docker. https://hub.docker.com/r/kelseyhightower/hello
    - nginx, official docker image, frontend to auth and hello services. https://hub.docker.com/_/nginx

If starting GKE from Cloud Shell, need to set environment, start cluster, be authenticated to cluster
    `gcloud config set compute/zone [name of zone]`
    `gcloud container clusters create io`
    `gcloud container clusters get-credentials io` to re-authenticate if lose connection

Quick start of Kubernetes 
1. using container image
    kubectl create deployment nginx --image=nginx:1.10.0 
        - nginx is the [deployment name] 
        - --image=[docker image name]:[version]
        - deployments keep pods up even when nodes fail
2. using pod configuration yaml files
        ``apiVersion: v1
          kind: Pod
          metadata:
            name: [name of pod]
            labels:
                app: [app name]
          spec:
            containers:
                - name: [name of container]
                image: [image name]:[version]
                args:
                ports:
                resources:
        ``
3.  kubectl create -f pod/[pod name].yaml

4.  kubectl get pods
        - view running containers in default namespace
5.  kubectl get pods -l 'app=monolith'
        - same get pods query but with a 'app=monolith' label
6.  kubectl describe pods [pod name]
        - get details

7.  kubectl label pods [pod name] 'label name'
        - add 'label name' label to the pods named [pod name]
8.  kubectl get pods [pod name] --show-labels
        - to confirm labels applied to pods

9.  kubectl expose deployment nginx --port 80 --type LoadBalancer
        - nginx is the [deployment name]
        - --port flags port number of container
        - -- type flags Service type as LoadBalancer(with public IP). Other 2 types: ClusterIP(internal), NodePort(externally accessible IP)
10. kubectl get services
        - view running services

    
11. curl http://[external IP]:[port number]

12. kubectl port-forward [pod name] [local host port]:[pod port]
        - by default, pods are allocated private IP addresses can cannot be accessed from outside cluster.
        - use `kubectl port-forward` to map local port to port inside pod
        - to map localhost port to port inside pod

13. kubectl logs [pod name]
        - to view logs for the pod
14. kubectl logs -f [pod name]
        - -f to get a real-time stream of the logs

15. kubectl exec [pod name] --stdin --tty -c [pod name] -- /bin/sh
        - to run interactive shell inside pod
        - once in shell, you can ping external site from pod. `ping -c 3 [website name]` will ping xxx website 3 times.
        - `exit` when finished using pod shell

Pods
https://kubernetes.io/docs/concepts/workloads/pods/
collection of 1 or more containers.
generally, if have multiple containers with hard dependency on each other (tightly coupled), package as single pod.
example: a logical application
    - containers
        - (micro)service container, nginx web server container
    - volumes/storage 
        - data disks that live as long as pod lives. 
        - shared by containers in pod
        - https://kubernetes.io/docs/concepts/storage/volumes/
    - shared namespace
        - containers in a pod can communicate with each other, share resources and dependencies
    - one IP per pod
        - shared by containers in pod
    co-located, co-scheduled
    
Services
https://kubernetes.io/docs/concepts/services-networking/service/
3 types of Service
    - 3 levels of access
        1. Cluster IP (internal)
            - Service can only be accessed within cluster
        2. NodePort
            - gives each node in cluster an externally accessible IP
            - remember to config firewall to allow external traffic to access nodePort
                ```
                gcloud compute firewall-rules create allow-monolith-nodeport \
                    --allow=tcp:31000
                ```
            - use `gcloud compute instances list` to get info on instances, which will include EXTERNAL_IP which can be used to curl pods
        3. Load Balancer
            - adds load balancer from cloud provider, which forwards traffic from service to nodes within
an abstraction to expose an application running on pods as a network service
    - defines a logical set of Pods, and a policy to access them.
the set of pods provides one functionality, e.g. "backend", of an application
    - sometimes called, micro-services
Although (1) each pod has a fixed IP address. (2) there is a single DNS name for a set of pods (namespace).
    - pods are created and destroyed to match desired state of cluster (Kubernetes is declarative)
    - pods can also be stopped and re-created when they fail liveliness and readiness tests
    - the replicas within a micro-service are fungible
    - an example is the use of Deployment yaml file to define desired state
    - the IP address for this functionality changes once pods change (dynamic)
    - so use Service to abstract away problem, provide static endpoint
has a selector to target pod. 
    - use labels to determine which Pods to select. Selected pods will be exposed by the Service.
    - target pods created with labels defined in Deployment yaml
    - a controller scans for pods that match selector, and POSTs updates to Endpoint object
for service discovery in application, can query control plane API server for Endpoints, which will get updated whenever pods for Service change; or query load balancer / NodePort
enables decoupling
a REST object in Kubernetes
    - POST Service definition (yaml file) to API Server to create new instance
    - ``    apiVersion: v1
            kind: Service
            metadata:
                name: [service name]
            spec:
                selector:
                    app.kubernetes.io/name: MyApp label
            ports:
                - name: [name of service port]
                  protocol: TCP
                  port: 80
                  targetPort: 9376
                  targetPort: [port name]
      ``
    - Kubernetes will assign IP to Service ("cluster IP")
    - can name port in Pod object, and use [port name] in Service object.
    - Kubernetes support multiple port definitions in a Service object.
    - can expose different port numbers for versioning
    - default protocol is TCP, but can support others, e.g. HTTP/S
Can have Service without selector
    - abstract to other backends, e.g. external database cluster, Service in other Namespaces or cluster
    - add Endpoints object with IP and port/s manually (only created automatically with selector)
    - Kubernetes API Server does not proxy to endpoints that are not mapped to pods
Endpoints object capacity is 1000
    - EndpointSlices are more scalable
    - 100 endpoints per resource
Virtual IP and Service proxies
    - every node runs kube-proxy
    - kube-proxy implements virtual IP ("cluster IP") for Service
    -kube-proxy runs in iptables mode
        - install iptables rules. 
        - backend selected at random. If first Pod does not response, connection fails.
        - can use readiness probes (https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) to ensure backend Pods are healthy
    - kube-proxy runs in ipvs mode 
        - lower latency, higher throughput than iptables mode
        - uses hash table as underlying table structure, works in kernel space
To bind particular client to same Pod, use sessionAffinity: "client IP"
When Pod runs on Node, kubelet adds environment variables for active Service.
    - must create Service before Pod because must publish cluster IP and port first
Best practice to setup DNS for cluster using add-on (https://kubernetes.io/docs/concepts/cluster-administration/addons/)
    - pings Kubernetes API to detect new Services -> set up DNS
Headless Service -> no cluster IP
Avoiding collisions
    - each Service receives unique cluster IP
    - internal allocator atomically updates global allocation map in etcd b4 creating each Service
Useful commands
    `cat services.yaml` to read config file
    `kubectl create -f services.yaml` to create Service from config file
    `kubectl get services [service name]` for basic name/type/IPs/Port/Age info
    `kubectl describe services [service name]` to see details
    `kubectl describe services [service name] | grep Endpoints` to get Endpoint IP of Service


Volume
- a directory, perhaps with data on it, that is accessible to containers in a pod.
- files in a container are lost when container crash. kubelet restarts container with clean slate.
    - second problem is how to share files between containers in a pod.
- A Docker volume is a directory on disk or in another container.
    - there are volume drivers
    - concept is looser and less managed
- Kubernetes support many specific types of volumes
    - a Pod can use any number of volume types simultaneously
    - ephemeral volume type exists during lifetime of Pod only
    - Kubernetes does not destroy persistent volume types
    - data is preserved across container restarts
- to use
    1. volumes must exist
    2. declare in the Pod manifest .spec.volumes
    3. specify where to mount each volume the container uses in .spec.containers.volumeMounts
- see Kubernetes documentation for the different types of volumes


Deployments
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#what-is-a-deployment
- describe desired state for Pods and ReplicaSets
- Deployment Controller will change actual state to desired state at a controlled rate.
- keep Pods up and running even when nodes fail
- use cases:
    - new rollout of ReplicaSet/s. ReplicaSet creates Pods in the background.
    - scale up to handle more load
    - declare new state of the Pods. 
        - update PodTemplateSpec
        - new ReplicaSet created
        - Pods migrated from old to new ReplicaSet
        - everytime new ReplicaSet created, Deployment revision number updated
    - check if rollout stuck
        - via kubectl rollout status deployment/[deployment-name]
        - can apply actions, e.g. scale down, rollback, pause, based on status
    - pause
        - to apply fixes to PodTemplateSpec, then restart with new rollout
    - clean up old ReplicaSets not needed
        - consume resources in etcd, and crowd output of `kubectl get rs`
    - rollback.
        - if current state not stable
        - to earlier Deployment version

[Deployment Object](https://kubernetes.io/docs/reference/kubernetes-api/workload-resources/deployment-v1/)
- deployment-name.yaml
```
apiVersion: apps/v1             //required field
kind: Deployment                //required 
metadata:                       //required
  name: deployment-name         //name of this deployment
  labels:
    app: nginx  
spec:
  replicas: 3                   //number of replicated pods to create. Do not set if using HorizontalPodAutoscaler, allow Kubernetes 
                                //control plane to mange
  selector:                     //required field
    matchLabels:               //how Deployment identifies the Pods to manage. match selected label. other method is matchExpressions
      app: nginx                // label defined in Pod template. map of {key,value} pairs.
  template:                     //required field. .spec.template exactly same schema as Pod template.
    metadata:
      labels:                   //template labelling of Pods
        app: nginx  
    spec:                       //Pod template's specification
      containers:               // can see that the Pods run 1 container each
      - name: nginx             //name of container
        image: nginx:1.14.2     //Docker Hub image:version number
        ports:
        - containerPort: 80
  strategy:
    type: RollingUpdate         //Recreate or RollingUpdate. "Recreate" terminates old Pods before creation for upgrades.
    rollingUpdate:
    - maxSurge: 25%             //default. value is absolute number or % of desired pods.
      maxUnavailable:           //default. value is absolute number or % of desired pods.
  revisionHistoryLimit: 10      //default. specifies how many old ReplicaSets to retain, rest will be garbage collected. 
                                //0 means no history, cannot rollback. 
                                //Old ReplicaSets consume resources in etcd and crowd `kubectl get rs`.   
  progressDeadlineSeconds: 600  //default. How long in sec to wait before report as failed progressing.
```

Useful commands:
    - `kubectl apply -f [link-to-manifest.yaml, e.g. https://k8s.io/examples/controllers/nginx-deployment.yaml]`
        - to create deployment from manifest
    - `kubectl get deployments`
        - to get status of deployments
        - info on how many replicas with desired state ready, up-to-date, available to users
    - `kubectl rollout status [kind]/[name]`
        - e.g. `kubectl rollout status deployment/nginx-deployment`
    - `kubectl get rs`
        - to get status of the ReplicaSets
        - name of replica set is in format [deployment-name]-[hash]
        - name, desired, current, ready, age
        - to watch status in progress, `kubectl get rs -w`
    - `kubectl get pods --show-labels`
        - shows status of each Pod with label. According to selector and Pod template label in Deployment.yaml
        - Deployment Controller adds a pod-template-hash label to every Pod in the same ReplicaSet to uniquely identify them
    - `kubectl set image deployment/[deployment-name] [name of container]=[new image label]`
        - to set container image to diff version
    - `kubectl edit deployment/[deployment-name]`
        - to make changes to manifest
    - `kubectl describe deployment`
        - for details on Deployment
    - `kubectl rollout history deployment/[deployment-name]`
        - for revisions and cause of each new revision
        - for details on specific ? revision, use `kubectl rollout history deployment/[deployment-name] --revision=?`
    - `kubectl rollout undo deployment/[deployment-name]`
        - to undo current rollout, rollback to previous version
        - to rollback to specific version, use `kubectl rollout undo deployment/[deployment-name] --to-revision=?`
    - `kubectl scale deployment/[deployment-name] --replicas=10`
        - for manual scaling to xx ReplicaSets
    - `kubectl autoscale deployment/[deployment-name] --min=10 --max=15 --cpu-percent=80`
        - for horizontal Pod auto-scaling
    `kubectl rollout pause deployment/[deployment-name]`
        - to pause rollout


**NEW** rollout **only** if Deployment's Pod template changed, i.e. changes under `.spec.template`
    - e.g. label updated, container image changed
    - default:
        - max 25% unavailable; 75% of desired number of Pods are up
        - max 25% more than desired number of Pods are created
    - if container updated while rollout is in progress, Deployment Controller will create new ReplicaSet for latest desired state and scale up. Will scale down rollout in progress and add to old ReplicaSet list. Does not wait for rollout to be completed before applying new desired state.
    - change label selector
        - cannot change for `apiVersion: apps/v1`
        - adding labels to selector
            - need to match with same update to Pod template labels
                - if not -> validation error
            - will not select existing ReplicaSets and Pods -> become orphans 
            - will create new ReplicaSet
            - same effect if value of existing selector key changed
        - deleting existing label from selector
            - no need matching change in Pod template labels
            - no effect on existing ReplicaSets and Pods -> still labelled
            - no new ReplicaSet created
            - no orphans

**Rollback**
- to a previous revision
- when rollout stuck, image pull, containers crash
- a Deployment's revision is created only when there is a rollout, i.e. only when there is a change to `.spec.template`
- so when rollback, only Deployment's Pod template part is rolled back
- scaling, meaning changes to `.spec.replicas`, does not change Deployment's revision
    - for both manual- and auto-scaling
    - so rollback has no effect on scaling
- a faulty rollout will be stopped by Deployment Controller because of `.maxUnavailable`. New faulty Pod will not be ready and available.
- use `kubectl rollout status deployment/[deployment-name]` to check on status
- use `kubectl rollout history deployment/[deployment-name]` to check on previous revisions available
- to undo rollout, `kubectl rollout undo deployment/[deployment-name]`

**Scaling**
- manually change number of ReplicaSets
    - `kubectl scale deployment/[deployment-name] --replicas=10`
- horizontal auto-scaling
    - response to workload increase by increasing the number of Pods
    - https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale-walkthrough/
    - vertical scaling is to increase CPU and memory allocation per Pod. no change to Pod number.
    - `kubectl autoscale deployment/[deployment-name] --min=10 --max=15 --cpu-percent=80`
- proportional scaling
    - when scaling a rolling (in-progress or paused) update
    - there are more than 1 existing active ReplicaSets
    - Deployment controller distributes changes into existing ReplicaSets proportionally
    - obeys `.maxSurge=?` and `.maxUnavailable=?` constraints

**Pausing and Resuming rollout**
- rollouts can be paused to apply a few fixes, before resuming
    - to pause, `kubectl rollout pause deployment/[deployment-name]`
    - some fixes possible: 
        - change container image `kubectl set image deployment/[deployment-name] [container-name]=[container-image]`
        - update resources `kubectl set resources deployment/[deployment-name] -c=[container-name] --limits=cpu=200m --memory=512Mi`
    - to resume, `kubectl rollout resume deployment/[deployment-name]`

**States of Deployment**
1. progressing
    - when
        1. when creating new ReplicaSet/s
        2. scaling up latest ReplicaSet
        3. scaling down old ReplicaSet/s
        4. new Pods ready or available
    - Deployment controller adds attributes to `.status.conditions`:
        ``` 
        type: Progressing
        status: "True"
        reason: NewReplicaSetCreated | FoundNewReplicaSet | ReplicaSetUpdated
        ```
2. complete
    - all replicas updated to latest state and available
    - no old replicas running
    - Deployment controller adds attributes to `.status.conditions`:
        ```
        type: Progressing
        status: "True"
        reason: NewReplicaSetAvailable
        ```
3. failed to progress
    - when
        1. insufficient quota for containers
        2. readiness probe failures
        3. image pull errors
        4. insufficient permissions
        5. limit ranges
        6. application runtime misconfiguration
    - detect failed deployment by specifying a deadline for deployment to progress: `.spec.progressDeadlineSeconds`
    - once deadline exceeded, `.status.conditions` updates to
        ```
        type: Progressing
        status: "False"
        reason: ProgressDeadlineExceeded
        ```
    - Kubernetes takes no further action after reporting failure. Need an external orchestrator to rollback.
    - can `kubectl describe deployment [deployment-name]` to see reason for failure. For details, use `kubectl get deployment [deployment-name] -o yaml`



Replica Sets
https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
purpose: maintain stable set of identical Pods
use of higher-order Deployment is recommended

frontend.yaml   //ReplicaSet config manifest
```
apiVersion: apps/v1     //required
kind: ReplicaSet        //required
metadata:               //required
  name: frontend
  labels:               
    app: guestbook
    tier: frontend
spec:                   //required
  # modify replicas according to your case
  replicas: 3           // number of Pods to maintain
  selector:             // label selector
    matchLabels:         
      tier: frontend    // If Pod has no controller or .ownerReferences, and matches selector, it will be acquired by ReplicaSet
  template:             // pod template. must have labels
    metadata:
      labels:
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
```


frontend-b2zdv.yaml //Pod object
```
apiVersion: v1
kind: Pod
metadata:
  creationTimestamp: "2020-02-12T07:06:16Z"
  generateName: frontend-
  labels:
    tier: frontend
  name: frontend-b2zdv      //name of pod, e.g. `kubectl get pods`
  namespace: default
  ownerReferences:          //info on ReplicaSet that owns this Pod. Link between ReplicaSet and Pod.
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: frontend
    uid: f391f6db-bb9b-4c09-ae74-6a1f77f3d5cf
```

Some useful commands
1. `kubectl apply -f https://xxxx/frontend.yaml`
2. `kubectl get rs`
3. `kubectl describe rs/[ReplicaSet-name]`
4. `kubectl get pods`
5. `kubectl get pods [name-of-pod] -o yaml`

What happens to Pods not created by ReplicaSet template?
Pods with label that match selector of ReplicaSet. 
    - If ReplicaSet has already deployed its own replicas and fulfilled desired state requirements, the new Pods created alone will be acquired and terminated.
    - if Pods are created from yaml first, then create ReplicaSet, the ready Pods will be selected and acquired to form part of its replica.

Deleting ReplicaSets with or without its Pods
 - see kubernetes.io documentation
  - does not update Pods. To do so, use Deployment

Isolating Pods from a ReplicaSet
- remove Pods from service by changing their labels
- the removed Pods will be replaced
- e.g. for debugging, data recovery

Scaling Pods within a ReplicaSet
- scale by updating `.spec.replicas` field
- ReplicaSet controller ensures desired number of Pods with matching label selector
- when scaling down, the Pods are prioritized according to following algorithm:
    1. pending and unscheduled Pods
    2. if `controller.kubernetes.io//pod-deletion-cost` annotation is set, pod with lower value deleted first
    3. Pods on nodes with more replicas deleted first
    4. Pods created more recently

Pod deletion cost
- set `controller.kubernetes.io/pod-deletion-cost`
- Pods with lower cost are deleted first
- implicit value is '0', if not setted
- updating this annotation generates Pod updates on apiserver, so avoid doing so frequently
- example use case: different Pods of an application could have different utilization levels. One driver Pod controls scale down. The application updates once before scale down. E.g. Apache Spark

Scaling entire ReplicaSets using Horizontal Pod Autoscaler (HPA)
- Use `kubectl autoscale rs [replica-name] --max=10 --min=3 --cpu-percent=50`
- Or `kubectl apply -f https://xxxx/hpa-rs.yaml`

hpa-rs.yaml     //HPA manifest
```
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

Alternatives to ReplicaSets
1. Deployment (recommended)
2. Bare Pods
    - ReplicaSet recommended even when only 1 Pod required
    - ReplicaSet will ensure replacement of Pod in case Pod terminated or deleted, e.g. when node fails or under maintenance (kernel upgrade)
    - can supervise multiple Pods across multiple nodes
3. Job
    - for Pods expected to terminate on their own (batch job)
4. DaemonSet
    - for machine-level function, e.g. monitoring and logging
    - life-time tied to machine. Pod starts before others, terminates when machine shut-down.