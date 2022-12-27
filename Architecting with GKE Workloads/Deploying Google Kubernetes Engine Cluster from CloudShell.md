# Deploying Google Kubernetes Engine Clusters from Cloud Shell
### Learn to
1. Use gcloud to deploy and modify GKE clusters
2. Use kubectl to inspect kubeconfig files and clusters
3. Deploy pods to GKE cluster with manifest files
4. Copy static html file into nginx server
5. Exercute commands directly in container using bash shell

### Task 1. Deploy GKE clusters
1. Before deploying GKE clusters, set environment variables for zone and cluster name.
`export $my_zone=us-central1-a`
`export $my_cluster-standard-cluster-1`
2. Create Kubernetes cluster using [gcloud container clusters create](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create)  
`gcloud container clusters create $my_cluster --num-nodes 3 --zone $my_zone --enable-ip-alias`
3. To verify, check Google Cloud Console > **Kubernetes Engine** > **Clusters**  

### Task 2. Modify GKE clusters
1. To change number of nodes, `gcloud container clusters resize $my_cluster --zone $my_zone --num-nodes=4`  
  - When issuing cluster commands, typically need to specify cluster name and cluster location (region or zone).
2. To verify, check Google Cloud Console > **Kubernetes Engine** > **Clusters**. The number of nodes has changed from 3 to 4.

### Task 3. Connect to a GKE cluster
- to connect to a cluster, need to have a kubectl configuration file with (a) credentials for authentication, (b) endpoint details of cluster.
- authentication:
  - for communications from external client > kube-APIserver on master node > container
  - for communications between containers within cluster
  - can take several forms
  - typically use OAuth2 tokens
  - for users, manage through Cloud Identity and Access Management for entire project, or cluster-specific role-based access control.
  - for cluster containers, use service accounts to authenticate and access external resources (e.g. Load Balancer)
  - `gcloud container clusters get-credentials $my_cluster --zone $my-zone`
  - only run command to get info for cluster created by another user, or in another environment. No need to run for cluster created yourself in same context.
  - can also use command to switch context to different cluster.
- kubeconfig file:
  - by default, `config` file is in home directory, `.kube` sub-directory.  
  - to open the kubeconfig file, `nano ~/.kube/config`. `CTRL+X` to exit.  
  - file can contain info on many clusters, users and contexts.
  - the active context (the cluster that kubectl commands manipulate) is indicated by `current-context` field.

### Task 4. Use kubectl to inspect a GKE cluster
- kubectl commands to cluster usually trigger REST API calls to kube-APIserver on master node.
- [kubectl config](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/) and [kubectl config](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_config/)
- to print content of kubeconfig file, `kubectl config view`
  - security certificate data is replaced by "DATA+OMITTED"
- to print info on active context cluster, `kubectl cluster-info`
- to print active context, `kubectl config current-context`
  - same info as `current-context` field in kubeconfig file
  - syntax: gke_[project ID]_[zone]_[display name]
- to print info on all contexts in kubeconfig file, `kubectl config get-contexts`
- to switch active context cluster, `kubectl config use-context [context you want to change to]`
  - kubeconfig file must have credentials and configuration for clusters you are switching between
- to view resource usage / metrics for nodes, `kubectl top nodes`
  - for pods, `kubectl top pods`
- to enable bash auto-complete for kubectl, `source <(kubectl completion bash)`
  - then `kubectl`+`SPACE`+`TAB`+`TAB` for list of all commands
  - `kubectl co`+`TAB`+`TAB` for all commands that start with "co", e.g. config, cordon, completion.

### Task 5. Deploy Pods to GKE clusters
- pod is an abstraction to deploy one or more containers as a single entity
  - schedule, create, destroy as a unit on same node
  - can have 1 container pulling 1 container image
  - or multiple containers from many container images
- to deploy nginx as a Pod named nginx-1, `kubectl create deployment --image=nginx nginx-1`
  - default behavior is to locate container image locally. If not present, then pull from Docker public registry.
  - to verify deployed pod, `kubectl get pods`
- to minimize error when typing long names, enter pod name into environment variable:
  `export my_nginx_pod=[pod name]`
  - to verify, `echo $my_nginx_pod`
- to get details on pod resource, `kubectl describe pod $my_nginx_pod`
  - [kubectl describe](https://jamesdefabia.github.io/docs/user-guide/kubectl/kubectl_describe/)
- to push file into container,
  1. create file `nano ~/test.html`
  2. add content to empty file, e.g.
  ```
  <html> <header><title>This is title</title></header>
  <body> Hello world </body>
  </html>
  ```
  3. `CTRL+X` > `Y` to save and exit nano editor.
  4. copy file into right location of nginx container for it to be served as static html
    `kubectl cp ~/test.html $my_nginx_pod:/usr/share/nginx/html/test.html`
    - default copies file to 1st container in pod
    - to copy to other containers, use `-c` option, followed by names of other containers.
- to expose a pod to clients outside the cluster, need a service.
  `kubectl expose pod $my_nginx_pod --port=80 --type=LoadBalancer`
  - to verify, `kubectl get services` then copy EXTERNAL-IP for `curl http://[EXTERNAL-IP]/test.html`

### Task 6. Introspect GKE Pods
- access and execute commands in pod 
- for debug, experiment
- no change to container image, so any changes not present in replicas
1. pod deployment
  - preferred method is to use manifest/config files
    - yaml syntax
    - hierachical structuring of objects and properties
  - `git clone https://github.com/GoogleCloudPlatform/training-data-analyst`
  - soft link to working directory `ln -s ~/training-data-analyst/courses/ak8s/v1.1 ~/ak8s`
  - change directory `cd ~/ak8s/GKE_Shell/`
  - kubectl apply manifest yaml file `kubectl apply -f ./new-nginx-pod.yaml`
  - verify, `kubectl get pods`
2. redirect shell to connect to pod
  - some container images include shell environment that can be launched.
  - e.g. bash shell with nginx container
  - to start new interactive bash shell in "new-nginx" container `kubectl exec -it new-nginx /bin/bash`
    - `exec`: execute
    - `-i` flag: stdin
    - `-t` flag: tty, send stdin to 'bash' in container, and stdout/stderr from 'bash' back to client
    - `-c` flag can be used to specify particular container by name
    - [kubectl exec](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#exec)
  - when in nginx bash shell, install nano editor
    `apt-get update`  
    `apt-get install nano`  
  - then create `test.html` and place in directory serving static files
    `cd /usr/share/nginx/html`  
    `nano test.html`  
    Paste into test.html
    ```
    <html> <header><title>This is title</title></header>
    <body> Hello world </body>
    </html>
    ```
    `CTRL+X` then `Y` to save and exit nano editor
  - `exit` nginx bash shell
3. port-forward from Google Cloud shell (port 10081) to nginx bash shell (port 80) 
  `kubectl port-forward new-nginx 10081:80`  
  - to test, open new Cloud Shell session and curl(transfer data) from `test.html`   
  `curl http://127.0.0.1:10081/test.html`
4. view logs of a pod
  - open another new Cloud Shell session
  - to display logs to date and stream new logs, `kubectl logs new-nginx -f --timestamps`
    - `--timestamps` flag is to include timestamps
