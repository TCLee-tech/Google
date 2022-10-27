## Hello Node Kubernetes
Kubernetes can run on many different environments: laptop to high-availability multi-node clusters; public clouds to on-premise deployments, virtual machines to bare metal.

Learning points:
1. create a node.js server
2. create a Docker container image
3. create a container cluster
4. create a Kubernetes pod
5. scale up your services

Linux text editors: `vim`, `emac`, `nano`

###Task 1: Create node.js application
1.`vi server.js`
2. Start vim editor `i`
3. write application code
```
var http = require('http');
var handleRequest = function(request, response) {
    response.writeHead(200);
    response.end("Hello World!");
}
var www = http.createServer(handleRequest);
www.listen(8080);
```
4. Save: ESC then `:wq`
5. Google Cloud Shell has node executable installed. So, to start node server, `node server.js`
5. Can preview on port 8080

### Task 2: Create Docker container image
1. You need to write a Dockerfile to describe image.
Docker container images can be extended from existing images.
Create Dockerfile `vi Dockerfile`
2. Start vim editor `i`
3. Write content of Dockerfile.
```
FROM node:6.9.2         // node image from Docker Hub to base on
EXPOSE 8080             // port to expose
COPY server.js .        // copy your application to image
CMD node server.js      // start the node server
```
4. Save Dockerfile: ESC, `:wq`
5. Build container image.
`docker build -t gcr.io/PROJECT_ID/hello-node:v1`
-t flag: to destination
syntax: [container registry]/[project]/[name of image]:[tag/version]
6. Test run a container from the newly-built image
syntax `docker run [OPTIONS] IMAGE[:TAG\@DIGEST][COMMAND][ARG...]`
https://docs.docker.com/engine/reference/run/#general-form
`docker run -d -p 8080:8080 gcr.io/PROJECT_ID/hello-node:v1`
-d flag: Start in detached mode. Run as daemon. Exit when root process used to start container exits, unless `rm` option also specified.
-p flag: to expose container's internal port to the host interfaces. Exposed port/s on host accessible to any client that can reach the host. Format: hostPort:containerPort.
7. Preview using `curl http://localhost:8080`
8. List Docker containers to find container ID
`docker ps`
9. Stop container
`docker stop [container ID]`
10. Push container to registry, e.g. [Google Container Registry](https://cloud.google.com/container-registry)
`docker push gcr.io/PROJECT_ID/hello-node:v1`
syntax: `docker push [container registry]/[folder]/[image name][:tag/version]  

### Task 3: Create Kubernetes cluster
1. Set gcloud config to project ID
`gcloud config set project PROJECT_ID`
2. Create cluster with 2 n1-standard-1 nodes
```gcloud container clusters create hello-world \
        --num-nodes 2 \
        --machine-type n1-standard-1 \
        --zone us-central1-a
```  
Recommended to create cluster in same zone as storage bucket used by container registry

### Task 4: Create pod
1. Create a deployment object.
```
kubectl create deployment hello-node \
    --image=gcr.io/PROJECT_ID/hello-node:v1`
```
2. Check deployment
`kubectl get deployments`
View pod
`kubectl get pods`
`kubectl cluster-info`
`kubectl config view`
For trouble-shooting,
`kubectl get events`
`kubectl logs [pod-name]`
[Command line tool kubectl](https://kubernetes.io/docs/reference/kubectl/)

### Task 5: Allow external traffic
By default, pod is only accessible by its internal IP within cluster.
1. Need to expose new Kubernetes **service**
Use `kubectl expose` operation with `--type="LoadBalancer"` flag to create externally accessible IP.
Exposing deployment, not pod. Service will load balance across all pods managed by deployment.
Kubernetes will create load balancer, network forwarding rules, firewall rules,  target pools.
`kubectl expose deployment hello-node --type="LoadBalancer" --port=8080`
2. To get the externally-accessible IP, get Kubernetes to list services `kubectl get services`

### Task 6: Scale up service
1. Kubernetes can scale horizontally. Handle increased/decreased load.
Declarative approach: declare target state.
Manual: `kubectl scale deployment hello-node --replicas=4`
Autoscale: `kubectl autoscale [TYPE] [NAME] --min=? --max=? --cpu-percent=?`
2. To check deployment, `kubectl get deployment`
To list pods, `kubectl get pods`

### Task 7: Update to application
1. Changes to application, e.g. bug fixes, new features
`vi server.js` ... `i` ... xxx Esc,`:wq`
2. Build new container image
`docker build -t gcr.io/PROJECT_ID/hello-node:v2 .`
3. Push to container registry  
`docker push gcr.io/PROJECT_ID/hello-node:v2`
This will push v2 of application to container registry. v1 still exists in registry.
4. Need to edit deployment yaml manifest to change image from v1 to v2.
`kubectl edit deployment hello-node`, `i` to insert
Save and close yaml file after editing. ESC, `:wq`
5. Check deployment
`kubectl get deployments`
https://kubernetes.io/docs/tutorials/kubernetes-basics/update/update-intro/