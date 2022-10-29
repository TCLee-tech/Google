## Managing Deployments Using Kubernetes Engine
Heterogeneous deployments: 2 or more distinct infrastructure environments or regions
  - to address technical or operational need
  - includes
    1. multiple public cloud (multi-cloud)
    2. combination of on-premises and public cloud (hybrid or public-private)
    3. multiple regions in single cloud

Multiple deployment scenarios
1. continuous deployment
2. canary deployments
3. blue-green deployments

Limitations of single environment or region:
1. resources maxed out
    - compute, storage, networking resources cannot meet production needs, esp on-premises.
2. limited geographical reach
    - one deployment location only. Traffic from distant users need to travel around globe.
3. limited availability
    - if traffic is global, application may be less fault-tolerant and resilient.
4. vendor lock-in
    - platform and infrastructure abstractions may prevent porting applications
5. inflexible with resources
    - limited to compute, storage, service options offered by vendor

### To learn:
1. Deployment
2. Rolling updates - trigger, pause, resume, rollback
3. Canary deployment
4. Blue-green deployment    

### Task 1: Learn about deployment object
`kubectl explain [object or field name]` command helps explain what the object or field does. e.g. `kubectl explain deployment.metadata.name`  
`kubectl explain deployment --recursive` display all the fields for that object

### Task 2: Create deployment
1. write/update deployment config file:  
  `vi deployment/auth.yaml`, then `i` to insert.  
  ESC, then `:wq` to save.  
2. check content of deployment config file is correct:  
  `cat [deployment-file.yaml]`
3. create deployment object from config file:
  `kubectl create -f [deployment-file.yaml]`
4. verify:
  `kubectl get deployments`  
  `kubectl get replicasets`  
  `kubectl get pods`  
5. create service for deployment:  
`kubectl create -f [service-file.yaml]`
6. verify service:  
`kubectl get service [service-name]`

To scale deployment  
manual: `kubectl scale deployment [deployment name] --replica=?`  
verify correct number of pods: `kubectl get pods | grep [pod name]- | wc -l`  

### Task 3: Rolling updates  
Minimise resource requirment (e.g. no need 2 set of compute before switch over), downtime, performance impact, risk  

1.Trigger 
  - by editing deployment template image or label
    - `kubectl edit deployment [deployment-name.yaml]`
  - Kubernetes will begin new rollout once change saved
  - verify new ReplicaSet
    - `kubectl get replicaset`
    - will see old replicaSet desired/current/ready as 0/0/0 and new replicaSet's as 3/3/3
  - verify rollout history
    - `kubectl rollout history deployment/[deployment-name]`  

2.Pause
  - e.g. if detect issues with rollout
  - `kubectl rollout pause deployment/[deployment-name]`
  - to verify rollout status, `kubectl rollout status deployment/[deployment name]`

3.Resume
  - will have new pods mixed with old pods when paused
  - `kubectl rollout resume deployment/[deployment name]`
  - to verify rollout status, `kubectl rollout status deployment/[deployment name]`

4.Rollback
  - if have problems with new version
  - `kubectl rollout undo deployment/[deployment-name]`
  - to verify rollback in history, `kubectl rollout history deployment/[deployment name]`

  ### Task 4: Canary deployments
  - to test new deployment in production with small subset of users
  - create new deployment manifest **in addition** to current stable deployment
    - identify canary deployment with additional label (e.g. track:canary)
  - use same service for production and new canary deployment
    - service uses same selector (e.g. app:hello) for production and canary deployment
    - because canary deployment has fewer pods, it will be less visible
  - some steps:  
    `cat deployment/[canary deployment.yaml]`  
    `kubectl create -f deployment/[canary deployment.yaml]`  
    `kubectl get deployments`  
  - sessionAffinity  
    - create service with `.spec.sessionAffinity:ClientIP` to restrict user to same deployment version
    - all clients with same IP address get same version of application
    - example use case: UI change

### Task 4: Blue-green deployment
  - 2 seperate deployments: old "blue" version, new "green" version
    - both rolled out
  - 2 different service manifests. 
    - only difference: one for old application image version, one for new image version
  - `kubectl apply` service `.spec.type:LoadBalancer` for "green"
  - to check version of image applied
  ```
  curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
  ```
  - major disadvantage: need 2X resources for 2 deployments
  - to rollback `kubectl apply -f services/[old service.yaml]`

