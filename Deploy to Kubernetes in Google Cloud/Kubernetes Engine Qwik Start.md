## Kubernetes Engine: Qwik Start
References:
- GKE provides managed environment to deploy, manage and scale containers.
https://cloud.google.com/kubernetes-engine/
- Environment is made up of Compute Engine instances forming a container cluster. Cluster architecture:
https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture
- Open-source Kubernetes
https://kubernetes.io/

- Cluster management features (additional benefits) of GKE:
1. load balancing and scaling
https://cloud.google.com/compute/docs/load-balancing-and-autoscaling
2. subset of nodes with same configuration (node pools)
https://cloud.google.com/kubernetes-engine/docs/concepts/node-pools
3. cluster autoscaler (auto resize number of nodes in a node pool)
https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-autoscaler
4. auto-upgrade software
https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-upgrades
5. auto-repair to maintain node health and availability
https://cloud.google.com/kubernetes-engine/docs/how-to/node-auto-repair
6. logging and monitoring for visibility
https://cloud.google.com/stackdriver/docs/solutions/gke

7. Kubernetes Deployment, a REST Object, for stateless applications, e.g. web server
https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
8. Services - an abstraction that defines a logical set of Pods and a policy to access them. A REST Object.
https://kubernetes.io/docs/concepts/services-networking/service/
9. "kubectl create" command - e.g. kubectl create deployment [name] --image=gcr.io/location
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#create
10. Google's container registry
https://cloud.google.com/container-registry/docs
11. "kubectl expose" command - expose a resource (deployment, replica set, pod, service, replication controller) as a new Kubernetes Service. e.g. kubectl expose deployment [name] --type=LoadBalancer --port 8080
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#expose
12. "kubectl get" command - a table of the most important info on resources. e.g. kubectl get service
https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get
13. deleting a cluster
https://cloud.google.com/kubernetes-engine/docs/how-to/deleting-a-cluster