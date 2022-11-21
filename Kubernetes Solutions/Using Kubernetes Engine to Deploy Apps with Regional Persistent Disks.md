## [Using Kubernetes Engine to Deploy Apps with Regional Persistent Disks](https://www.cloudskillsboost.google/focuses/1050?parent=catalog)

Deploying an application (WordPress as example) using a regional Kubernetes Engine cluster and regional persistent disk.

### What to learn
1. Create a regional Kubernetes Engine Cluster
2. Create a Kubernetes StorageClass resource
3. Create PersistentVolumeClaims for the StorageClass
4. Deploy application (WordPress) chart (Helm packaging format; to deploy single instance of Kubernetes application) using [Helm](https://helm.sh/) (Kubernetes package manager)
5. Simulate zone failure by deleting instance group

### [Regional GKE cluster](https://cloud.google.com/kubernetes-engine/docs/concepts/regional-clusters)
- a **zonal cluster** has a single control plane and nodes in a single zone only.
- a **multi-zonal cluster** has a single control plane, and nodes running in multiple zones.
- a **regional cluster** has multiple replicas of the control plane in different zones of a region. Nodes are in single or multiple zones, depending on config.
  - default is to replicate node pool (control plane + nodes) across 3 zones in same region.
- advantages
  1. Resilience from single zone failure. If one zone down, cluster's control plane remains available as long as other 2 default control plane replicas remain available.
  2. during cluster maintenance, only one control plane replica unavailable. Kubernetes API still available. Cluster still operational. Reduced downtime.
- rebalance manually or use **cluster autoscaler**
  - can over-provision scaling limits, to ensure minimum availability.
  - e.g. over-provision three-zone cluster to 150% (50% excess capacity), 100% traffic routed to available zones if one zone's capacity lost.
- limitations
  1. 9 nodes (3 per zone) consume 9 IP addresses. Can reduce to one node per zone.
    - newly created Cloud Billing accounts granted only 8 IP addresses per region, so need to request for quota increase.
  2. to run GPUs in regional cluster, must have GPUs available in all 3 zones.
  3. node-node traffic across zone incurs cost.
- GKE assigns persistent disk to single, random zone.


