# Running a MongoDB Database in Kubernetes with StatefulSets
Using persistent storage with Kubernetes  
Example: Run a stateful application (MongoDB database) on a stateless service (container)

Learning points:
1. Deploy a Kubernetes cluster. 
2. Instantiate a Headless Service, a StatefulSet, a StorageClass. Git clone MongoDB replica set/sidecar.
3. Connect Kubernetes cluster to MongoDB replica set.
4. Scale MongoDB replica set instances up and down.
5. Clean up StatefulSet environment and shut down services.

## Task 1. Set a compute zone
- Set a compute zone so that all VMs in cluster are created in same region.
- Use [gcloud command line tool](https://cloud.google.com/sdk/gcloud/)
- `gcloud config set compute/zone us-central1-f`

## Task 2. Create a new cluster
 - [gcloud container clusters create](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create)
 - `gcloud container clusters create hello-world --num-nodes=2`  

## Task 3. Setting up / Task 4. Deploying the Headless Service and StatefulSet
- Will be creating a [MongoDB replica set](https://www.mongodb.com/docs/manual/replication/)
  - a MongoDB replica set is a group of Mongod instances/processes that maintain the same data set.
  - group consists of several data-bearing nodes, and optionally, an arbiter node.
  - one primary data-bearing node. the rest are secondary nodes.
  - only the primary node receives all write operations. All changes to its data are updated in its operation log, i.e. oplog.
  - secondary nodes will replicate primary's oplog and apply operations to their own data set.
  - replica sets provide redundancy and high availability -> suitable for production deployment.
- Git clone and download [MongoDB replica set/sidecar](https://github.com/thesandlord/mongo-k8s-sidecar)
- `kubectl apply` a [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- `kubectl apply` a [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services)
- `kubectl apply` a [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)

- clone MongoDB replica set from Github repo.   
  `gsutil -m cp -r gs://spls/gsp022/mongo-k8s-sidecar .`  
  `cd ./mongo-k8s-sidecar/example/StatefulSet/`
- set the [type of storage](https://cloud.google.com/compute/docs/disks/) to use for database nodes.  
  - Instantiate StorageClass.
  `kubectl apply -f googlecloud_ssd.yaml`  
  - StorageClass has provisioner that specifies what disk will be used to provide persistenceVolume (PV).
- deploy headless service and StatefulSet.  
  `kubectl apply -f mongo-statefulset.yaml`
  - headless service overview
    - first section of `mongo-statefulset.yaml` refers to a `headless service`.
    - service describes policies or rules for accessing a specific set of pods
    - a headless service does not prescribe load balancing
      - ensure `.spec.clusterIP: None`
    - StatefulSet part of configuration provides unique DNS to access each pod, hence connection to individual MongoDB pods.
  - StatefulSet overview
    - metadata section: name of StatefulSet, label, number of replicas
    - pod spec section: `terminationGracePeriodSeconds`, configuration for 2 containers,
      - `terminationGracePeriodSeconds` is the grace period before pod shutdown when number of replicas scaled down.
      - 1st container runs MongoDB, has command line flags to configure replica set name, path to mount persistenceVolume.
      - 2nd container runs sidecar (a helper container that helps main container to run its job)
    - volumeClaimTemplates section: instantiates PVC, specifies accessMode and storage size.

- sample StatefulSet manifest
```
apiVersion: v1   <-----------   Headless Service configuration
kind: Service
metadata:
  name: mongo
  labels:
    name: mongo
spec:
  ports:
  - port: 27017
    targetPort: 27017
  clusterIP: None
  selector:
    role: mongo
---
apiVersion: apps/v1    <------- StatefulSet configuration
kind: StatefulSet
metadata:
  name: mongo
spec:
  serviceName: "mongo"
  replicas: 3
  selector:
    matchLabels:
      role: mongo
  template:
    metadata:
      labels:
        role: mongo
        environment: test
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: mongo
          image: mongo
          command:
            - mongod
            - "--replSet"
            - rs0
            - "--smallfiles"
            - "--noprealloc"
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-persistent-storage
              mountPath: /data/db
        - name: mongo-sidecar
          image: cvallance/mongo-k8s-sidecar
          env:
            - name: MONGO_SIDECAR_POD_LABELS
              value: "role=mongo,environment=test"
  volumeClaimTemplates:
  - metadata:
      name: mongo-persistent-storage
      annotations:
        volume.beta.kubernetes.io/storage-class: "fast"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 50Gi
```


## Task 5. Connect to the MongoDB Replica set
- Pods deployed sequentially until entire relicaset ready and running. 
  - Each pod is deployed completely (mongoDB container, sidecar, storage volume) before the next starts deploying.
  - to verify replicaSet deployment, `kubectl get statefulset`
- each pod is a MongoDB node
  `kubectl get pods`
- to start a shell session in a specific MongoDB pod in the Kubernetes cluster
  - `kubectl exec` [command](https://www.containiq.com/post/using-kubectl-exec-shell-commands-examples)
  - to debug or extract data
  - a `REPL` environment connected to MongoDB
    - REPL: Read Eval Print Loop
    - a computer environment like a console where a command is entered and system responds with an output in an interactive mode.
  - `kubectl exec -ti mongo-0 -- mongosh`
    - kubectl connects to specific "mongo-0" pod (no matter which node) in Kubernetes cluster and executes "mongosh" command.
    - `--` seperates command to pass to container from kubectl arguments.
    - `ti` refers to `--stdin(-i)` and `--tty(-t)` flags. `-i` instructs kubectl to route input stream to container. `-t` sets up an interactive session where input is supplied to processes running in container. Without `it`, get read-only output stream.
    - once in, instantiate MongoDB replicaset with default configuration
      - `rs.initiate()`
    - to print replicaset configuration, `rs.conf()
  - to see info for other replicaset pods, do likewise. Or, expose replicasets through [nodepost](https://kubernetes.io/docs/concepts/services-networking/service/#type-nodeport) or [loadbalancer](https://cloud.google.com/load-balancing/docs/https)

## Task 6. Scaling the MongoDB replica set
- manually scale MongoDB replicasets from 3 to 5
  - use [kubectl scale](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#scale)
  - `kubectl scale --replicas=5 statefulset mongo`
- to verify, `kubectl get pods`

## Task 7. Using the MongoDB replica set
- DNS of pod in a StatefulSet linked to a Headless Service is `<pod name>.<service name>`, e.g. mongo-0.mongo
- this is used in [connection string URI](https://www.mongodb.com/docs/manual/reference/connection-string/) used to connect applications to mongoDB instances.
  - syntax for replicaset: [mongodb://][host name: port for all mongod instances listed in replicaset config][/dbName]
  - e.g. `mongodb://mongo-0.mongo,mongo-1.mongo:27017/dbName_?`

## Task 8. Clean up
- need to delete StatefulSet, Headless Service, provisioned volumes (PVC), and the Kubernetes cluster.  
`kubectl delete statefulset mongo`  
`kubectl delete svc mongo`  
`kubectl delete pvc -l role=mongo`  //-l for label  
`gcloud container clusters delete 'hello-world"`  

<hr>

## StatefulSets
- API object used to manage stateful applications
- manages deployment and scaling of Pods
- guaranteees the uniqueness and ordering of Pods
- like Deployment, StatefulSet creates identical Pods from the same spec
  - but each Pod in a StatefulSet has a persistent identifier that is maintained across rescheduling
  - Pods not interchangeable
  - individual Pods can fail, but persistent Pod identifiers can match existing volumes to replacement Pods

#### When to use StatefulSets
For applications that need:
- stable, persistent storage.   stable: persistence across Pod(s) rescheduling
- stable, unique network identifiers
- ordered deployment, scaling or deletion
- ordered automated rolling updates
For stateless replicas, use [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) or [ReplicaSet](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)

#### Limitations of StatefulSets
1. the storage must be provisioned by administrator, or dynamic [PersistentVolume Provisioner](https://github.com/kubernetes/examples/blob/master/staging/persistent-volume-provisioning/README.md) based on `storage class`
2. need [Headless Service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services) for networking of Pods.
3. If StatefulSet deleted or scaled, the volumes unchanged. For data safety reason.
4. When StatefulSet deleted, no guarantee Pods terminated. Recommend to scale down to 0 Pods prior to deletion.
5. When use Rolling Updates with default Pod management Policy, may get into broken state that needs manual intervention to repair.

#### Sample
```
apiVersion: v1
kind: Service       <= Headless Service named "nginx"
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet       <= StatefulSet named "web" with 3 replicas of nginx containers in unique pods
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  minReadySeconds: 10 # by default is 0 <= min second newly created pod must be running w/o crash, for it to be considered available.
  template:                                 impt in Rolling Update strategy.
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: registry.k8s.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:     <=  each pod receives PVC that binds PersistentVolume dynamically provisioned by StorageClass.
  - metadata:                   
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: "my-storage-class"
      resources:
        requests:
          storage: 1Gi
```
#### Pod identity
- StatefulSet Pods' unique identity consists of an ordinal, a stable network identity, and a stable storage.
- default ordering starts from 0.
  - use `.spec.ordinals` to config
- pod has stable network ID
  - pod hostname syntax is `$(statefulset name)-$(ordinal)`, e.g. web-0, web-1.
  - domain managed by Headless Service has syntax `$(service name).$(namespace).svc.cluster.local`
    - sub-domain for pod `$(podname).$(governing service domain)`
  - to discover Pods promptly after creation, (1) query Kubernetes API; (2) decrease cache time in Kubernetes DNS Provider (edit config map for CoreDNS).
- stable storage
    - volumeClaimTemplate will create PVC that binds PV dynamically generated by StorageClass.
    - if pod is (re)generated, its `volumeMounts` mount PV associated with PVC.
    - if StatefulSet or pod deleted, must delete PVC and PV seperately.
- pod name label
  - all pods created by StatefulSet controller has a label `statefulset.kubernetes.io/pod-name`. This allows Service to be attached to a specific pod for StatefulSet.


#### Scaling and Deployment
- when there are N replicas, pods are created sequentially, in order from {0,1,2....N-1}
- pods are terminated in reverse order {N-1, N-2 .... 0}
- before scaling of next pod, all predecessors must be Running and Ready
- before termination of a pod, all previous pods must be completely shutdown
- above ordering guarantees for scaling can be altered via `.spec.podManagementPolicy` field. Can set as `parallel` instead of `OrderedReady`. No effect on updates.

#### Update Strategies
- `.spec.updateStrategy` field allows you to configure and disable automated rolling updates for containers, labels, resource request/limits, and annotations for pods.
- possible values:
  - **OnDelete**. StatefulSet controller will only create new pods that reflect modifications made to `.spec.template` when user manually delete pods.
  - **RollingUpdate** default. Automated, rolling update strategy. Same sequence as pod termination (from largest ordinal to smallest), 1 pod at a time.
- there is option to do a partitioned rolling update, e.g. a staged update, a phrased roll out, a canary roll out. 
  - specify `.spec.updateStrategy.rollingUpdate.partition` number.
  - if partition specified, only pods with ordinal equal or larger than partition will be updated when `.spec.template` updated.
  - if `.spec.updateStrategy.rollingUpdate.partition` > `.spec.replicas`, no updates propagated when `.spec.template` changed.

#### Maximum unavailable Pods
- Can control by specifying the `.spec.updateStrategy.rollingUpdate.maxUnavailable` field.
- can be absolute number or %.
- default is 1. cannot be 0.

#### Forced rollback
- If change pod template -> Rolling Updates with default Pod Management Policy fails (e.g. application-level config error, bad binary) -> broken pod does not become Running and Ready -> StatefulSet will stop rollout and wait forever.
- need to (1) correct error, (2) revert pod template to good configuration, (3) delete broken pod.
- StatefulSet will then begin to recreate pods.

#### PersistentVolumeClaim retention
- controls if and when PVCs deleted during StatefulSet lifecycle
- optional
- must enable `StatefulSetAutoDeletePVC` [feature gate](https://kubernetes.io/docs/reference/command-line-tools-reference/feature-gates/) on API server and controller manager.
- once enabled, two policies can be condigured:
  - **whenDeleted** for what happens when StatefulSet deleted
    - __delete__ when pods deleted, PVC deleted
    - __retain__ PVC retained (default)
  - **whenScaled** for what happens when replica count scaled down
    - __delete__  when pod replicas scaled down -> pods deleted -> PVC deleted
    - __retain__ PVC retained (default)
- only apply when StatefulSet deleted or scaled down
- does not apply when pods removed for other reasons, e.g. node failure. No change to PVC.
```
apiVersion: apps/v1
kind: StatefulSet
...
spec:
  persistentVolumeClaimRetentionPolicy:
    whenDeleted: Retain
    whenScaled: Delete
...
```
- the mechanism (how this PVC retention policy works): the StatefulSet controller adds an [owner reference](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/#owner-references-in-object-specifications) to the StatefulSet on its PVC. After pods are terminated, the marked PVCs are deleted by garbage collector.
- if controller crashes and restarts, no pod will be deleted before owner reference is updated according to policy.

#### Replicas
- `.spec.replicas` specifies desired number of pods. Optional.
- can scale manually. `kubectl scale statefulset xxxx --replicas=X`
- or scale by applying manifest `kubectl apply -f statefulset.yaml`
- if using [HorizontalPodAutoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/), do not set `.spec/replicas`. Kubernetes control plane will manage automatically.

