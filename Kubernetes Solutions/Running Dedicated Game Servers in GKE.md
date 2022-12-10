# Running Dedicated Game Servers in Google Kubernetes Engine
Package server applications as containers for game companies
  - optimize VM utilization
  - isolate run-time

Infrastructure components:  
1. Container images of Dedicated Game Server (DGS) based on Linux
    - base Linux image + binaries + necessary libraries
    - create using docker
2. Kubernetes cluster
3. Persistent disk volume for other game assets
    - read-only
    - mount to containers at run-time
4. Scaling Manager that changes the number of VMs (i.e. start, stop) based on current DGS load
5. Other frontend and backend components for production games. 
    - e.g. analytics stack, database of records, match-making service [Open Match](https://open-match.dev/site/), chat, leaderboard

[Overview of Cloud Game Infrastructure](https://cloud.google.com/architecture/cloud-game-infrastructure#high-level_components)

#### Task 1. Containerizing the Dedicated Game Server (DGS)
DGS pattern:
1. Server binary compiled from same code base as client.
2. The only data in server binary are those needed to run simulation.
3. Server container image: base OS image + binaries + minimum libraries needed to run server process.
4. Assets mounted from separate persistent volume.

#### Task 2. Creating a Dedicated Game Server container image
1. Create a VM
    - From Cloud Console, **Compute Engine** -> **VM Instances** -> **Create Instance**
    - **Identity and API access** -> **Allow full access to all Cloud APIs** -> **Create**
    - **SSH** into instance
2. Install kubectl and docker on VM
    - `sudo apt-get update`
    - `sudo apt-get -y install kubectl google-cloud-sdk-gke-gcloud-auth-plugin`
    - install docker dependencies
    ```
    sudo apt-get -y install \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common
     ```
    - install official GPG keys (to validate integrity of Linux package to be downloaded)
    ```
    curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    ```
    - add stable Docker repo
    ```
    echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list
    ```
    - update package cache   
      `sudo apt-get update`
    - install Docker-ce   
      `sudo apt-get -y install docker-ce docker-ce-cli containerd.io`
    - run as non-root user   
      `sudo usermod -aG docker $USER`
3. Generate container image
    - clone server files. `gsutil -m cp -r gs://spls/gsp133/gke-dedicated-game-server .`
    - select `gcr.io` regon closest to Kubernetes cluster. [Pushing and pulling images documentation](https://cloud.google.com/container-registry/docs/pushing-and-pulling#choosing_a_registry_name)
    - prepare environment. `export GCR_REGION=<REGION> PROJECT_ID=<PROJECT_ID> printf "$GCR_REGION \n$PROJECT_ID\n"`
    - write [Dockerfile](https://docs.docker.com/engine/reference/builder/) `cd gke-dedicated-game-server/openarena`
    - docker build. `docker build -t ${GCR_REGION}.gcr.io/${PROJECT_ID}/openarena:0.8.8 .`
    - docker push. `gcloud docker -- push ${GCR_REGION}.gcr.io/${PROJECT_ID}/openarena:0.8.8`   

#### Task 3. Create asset disk
Game assets put on a read-only [persistent disk](https://cloud.google.com/compute/docs/disks/#pdspecs), attached to multiple VM instances running DGS containers.
1. Store a zone ID in an environment variable
```
region=us-east1
zone_1=${region}-b
gcloud config set compute/region ${region}
```
2. Create asset-builder VM instance using gcloud
```
gcloud compute instances create openarena-asset-builder \
   --machine-type f1-micro \
   --image-family debian-11 \
   --image-project debian-cloud \
   --zone ${zone_1}
```
3. Create persistent disk.
    - seperate from boot disk.
    - configure to remain undeleted when VM removed.
    - Kubernetes [persistentVolume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) works best with single `ext4` file system without partition table. [Reference](https://cloud.google.com/compute/docs/disks/add-persistent-disk#formatting).
    - some disk types perform better at larger sizes. [Reference](https://cloud.google.com/compute/docs/disks/performance).
  ```
  gcloud compute disks create openarena-assets \
   --size=50GB --type=pd-ssd\
   --description="OpenArena data disk. Mount read-only at
/usr/share/games/openarena/baseoa/" \
   --zone ${zone_1}
```
4. Attach persistent disk
```
gcloud compute instances attach-disk openarena-asset-builder \
   --disk openarena-assets --zone ${zone_1}
```
5. SSH into the asset-builder VM instance
6. Format the persistent disk with `mkfs.ext4` [command](https://manpages.ubuntu.com/manpages/xenial/man8/mkfs.8.html)
    - new disks are unformated
    - `sudo lsblk` to view attached disks and their partitions
    - confirm device ID (sdb) for openarena-assets disk
    - `sudo mkfs.ext4 -m 0 -F -E lazy_itable_init=0,lazy_journal_init=0,discard /dev/sdb`  
7. mkdir the asset directory `/usr/share/games/openarena/baseoa/` in the VM instance, mount the persistent disk to this directory. When game server is installed to the VM, the asset archives get copied to persistent disk.
```
sudo mkdir -p /usr/share/games/openarena/baseoa/
sudo mount -o discard,defaults /dev/sdb \
    /usr/share/games/openarena/baseoa/
sudo apt-get update
sudo apt-get -y install openarena-server
sudo gsutil cp gs://qwiklabs-assets/single-match.cfg /usr/share/games/openarena/baseoa/single-match.cfg
```
8. Unmount persistent volume, shut down instance.
```
sudo umount -f -l /usr/share/games/openarena/baseoa/
sudo shutdown -h now
```
9. Delete asset-builder VM instance. In main lab VM instance SSH console,  
  `echo $zone_1`  
  `region=us-east1 zone_1=${region}-b`  
  `gcloud compute instances delete openarena-asset-builder --zone ${zone_1}`  

Advice:
  - put all asset files in a suitable directory structure
  - use script running gcloud commands to automate persistent disk creation
  - create multiple copies of PD, and attach to VMs in balanced manner to mitigate failure risk

#### Task 4. Setting up a Kubernetes cluster
It is important to choose the correct [machine type](https://cloud.google.com/compute/docs/machine-resource) for the cluster.
Two factors to consider:
  * The largest number of concurrent DGS pods you plan to run. [Kubernetes has a max number of nodes per cluster](https://kubernetes.io/docs/setup/best-practices/cluster-large/). 1 node is 1 physical or virtual machine. Each machine can have multi-core (vCPUs). Each vCPU can run one or more DGS. A multi-core VM can support more DGS than a single-core VM.
  * VM instance gradularity. The simplest way for resources to be added/removed is in increments of a single VM of the type chosen during cluster creation.

1. In **Cloud Shell**, create network and firewall rules for Kubernetes cluster.
```
gcloud compute networks create game
gcloud compute firewall-rules create openarena-dgs --network game \
    --allow udp:27961-28061  
```
2. In **SSH console of main VM instance**, create a 3-node cluster.
```
gcloud container clusters create openarena-cluster \
   --num-nodes 3 \
   --network game \
   --machine-type=n1-standard-2 \
   --zone=${zone_1}
```
With managed instance group, the default Container Engine image template includes Kubernetes components and automatically registers node with master on startup.  

3. Set up local Cloud Shell with [authentication credentials](https://cloud.google.com/sdk/gcloud/reference/container/clusters/get-credentials) to access Kubernetes cluster.  
`gcloud container clusters get-credentials openarena-cluster --zone ${zone_1}`

#### Configuring the assets disk within Kubernetes
Typical DGS pod do not need write access to game assets
 - read-only persistent disk can mount to multiple DGS pods
1. Create a `persistentVolume` resource to detects the persistent disk.
  Create a [persistentVolumeClaim](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistentvolumeclaims) resource for pod to claim persistentVolume resource
 ```
kubectl apply -f k8s/asset-volume.yaml
kubectl apply -f k8s/asset-volumeclaim.yaml
```
2. Verify that PV and PVC are in `Bound` status.  
`kubectl get persistentVolume`  
`kubectl get persistentVolumeClaim`  

#### Configuring the DGS pods
- all functionalities needed for DGSes to run as containers should be included directly in binaries.
- there is negligible startup and shutdown time for instance, i.e. on demand
- pod networking and scheduling handled by Kubernetes
- each DGS run instance only lasts length of a single game match.
- each game match is configured with a defined time limit
- container exits when match completed. New container for new game.
- this design simplifies pod lifecycle.
- there is a `openarena/k8s/oarena_pod.yaml` to describe pod resource

#### Matchmaker / Scaling Manager
  - needed to handle client reconnections when there are network issues or server crashes
  - should not communicate with backend DGS servers directly
  - should [query Kubernetes API](https://kubernetes.io/docs/tasks/access-application-cluster/access-cluster/#directly-accessing-the-rest-api) for state of DGS pods, to start pods.
  - no need for parallel concurrent running of pods (Kubernetes functionality) as games are independent sessions
  - no need for auto restart of pods. If DGS crash, state lost, player just start afresh, new DGS pod.
  - no need for [Kubernetes Job](https://kubernetes.io/docs/concepts/workloads/controllers/job/)

#### Task 5. Setting up the scaling manager
Scaling manager
  - scales the number of VMs used as nodes based on DGS load
  - VMs as [managed instance groups](https://cloud.google.com/compute/docs/instance-groups/)
  - based on scripts (.sh file) that run forever
  - inspects number of DGS pods running and requested, then resize node pool
  - scripts packaged in docker container images with with libraries and [Cloud SDK](https://cloud.google.com/sdk/docs/)
  - scripts written to restart in the event of failure
  
In practice,
1. Docker files in `scaling_manager/` directory
2. Configure environment variables   
    `export GCR_REGION=us/eu`    
    `export PROJECT_ID=[PROJECT_ID]`  
3. Run build-and-push shell script to build docker image and push to gcr.io
```
cd ../scaling-manager
chmod +x build-and-push.sh
source ./build-and-push.sh
```
4. YAML deployment template for scaling manager in `scaling_manager/k8s/openarena-scaling-manager-deployment.yaml` directory.
5. Customize file with cluster's environment variables
    - get name of base instance 
      `gcloud compute instance-groups managed list`
    - set environment variables  
      `export GKE_BASE_INSTANCE_NAME=[BASE_INSTANCE_NAME]`  
      `export GCP_ZONE=[ZONE]`  
      `printf "$GCR_REGION \n$PROJECT_ID \n$GKE_BASE_INSTANCE_NAME \n$GCP_ZONE \n"`  
   - apply variables to yaml deployment template  
      `sed -i "s/\[GCR_REGION\]/$GCR_REGION/g" k8s/openarena-scaling-manager-deployment.yaml`  
      `sed -i "s/\[PROJECT_ID\]/$PROJECT_ID/g" k8s/openarena-scaling-manager-deployment.yaml`  
      `sed -i "s/\[ZONE\]/$GCP_ZONE/g" k8s/openarena-scaling-manager-deployment.yaml`  
      `sed -i "s/\gke-openarena-cluster-default-pool-\[REPLACE_ME\]/$GKE_BASE_INSTANCE_NAME/g" k8s/openarena-scaling-manager-deployment.yaml`  
6. Kubectl apply deployment.yaml  
  `kubectl apply -f k8s/openarena-scaling-manager-deployment.yaml`
7. Verify if deployment successful  
  `kubectl get pods` 

#### Scaling nodes
Scaling Manager calls Kubernetes API to check node usage and resize [managed instance group](https://cloud.google.com/compute/docs/instance-groups/) running the cluster's VMs. Common issues:
1. CPU and memory usage metrics often not representative of DGS usage. Use number of DGSes, number of VMs and network traffic too.
2. Need to maintain buffer of available, under-utilized nodes. Scheduling an optimized container on a ready node takes seconds. Starting a new node takes minutes. Unacceptable latency for players.
3. Many autoscalers cannot handle pod shutdowns properly. All pods need to terminate before node removal for scale down. Ensure no premature termination of game in play.
Use such DGS performance characteristics as metrics to determine when to add/remove VMs.  

Scale up nodes
  - by increasing number of nodes in Managed Instance Group
  - use [resource requests and limits for pods](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#resource-requests-and-limits-of-pod-and-container) to specify resource (i.e. CPU and memory) for a DGS pod. Example: 1 vCPU and 500MB memory.
  - compare to vCPU and memory of machine type when setting up VM. Example: 1vCPU and 600MB for `n1-highcpu` processor.
  - in this example, will know that 1 DGS pod equals 1 vCPU.
  - so if compare number of DGS pod in cluster to number of vCPUs in all nodes in cluster, will get % of resources used.
  - can add more nodes if % falls below a threshold.
  - many production games have different game types, specific maps, number of player slots etc that result in different usage profiles. May help to have multiple pod configurations for the different usage scenarios.   

Scale down nodes
  - multi-step process
  - `scaling_manger.sh` script used in this example
  - first node returned by Kubernetes API marked as 'unavailable' to Kubernetes scheduler using [cordon command](https://kubernetes.io/docs/concepts/architecture/nodes/#manual-node-administration). Prevents new pods from being scheduled on node, but no effect on existing pods. `kubectl cordon $NODENAME` 
  - once no more running pods on node tainted as 'unavailable', node is removed from Managed Instance Group using [abandon-instances command](https://cloud.google.com/sdk/gcloud/reference/compute/instance-groups/managed/abandon-instances).
  - a `node_stopper.sh` monitors abandoned and unscheduleable nodes for absence of DGS pods. Once all pods exit, shuts down [node](https://kubernetes.io/docs/tutorials/kubernetes-basics/explore/explore-intro/).
Scale down DGS pods
  - DGS pods are configured to exit when game complete (single match, time constraint etc). No extra script needed.
  - matchmaker system only adds DGS pods for new matches.

#### Task 6. Testing the setup
1. Test pod deployment  
   - update the template `openarena/k8s/openarena-pod.yaml` with [GCR_REGION] and [PROJECT_ID] environment variables.  
     ```
     cd ..
     sed -i "s/\[GCR_REGION\]/$GCR_REGION/g" openarena/k8s/openarena-pod.yaml
     sed -i "s/\[PROJECT_ID\]/$PROJECT_ID/g" openarena/k8s/openarena-pod.yaml
     ```  
   - Kubectl apply for pods.  
     `kubectl apply -f openarena/k8s/openarena-pod.yaml`  
   - Verify pod deployment.  
     `kubectl get pods`  
  
2. Test connection to game server from downloaded local client.  
   - Identify game server IP address.  
   ```
   export NODE_NAME=$(kubectl get pod openarena.dgs -o jsonpath="{.spec.nodeName}")
   export DGS_IP=$(gcloud compute instances list \
     --filter="name=( ${NODE_NAME} )" \
     --format='table[no-heading](EXTERNAL_IP)')
   printf "Node Name: $NODE_NAME \nNode IP : $DGS_IP \nPort : 27961\n"
   printf " launch client with: \nopenarena +connect $DGS_IP +set net_port 27961\n"
   ```  
   - **Multiplayer** -> **Specify** -> add IP and port -> **Connect**.  

3. Test scaling manager
   - Use a script that requests a number of pods per unit time
   - `bash ./scaling-manager/tests/test-loader.sh` requests for 4 DGS pods per minute for 5 minutes
   - In Cloud Console -> **Kubernetes** -> **Workloads**, will see sequence of workloads startup. 
   - Initially, test containers will show as error state of 'unavailable'.
   - However,  scaling manager kicks in to add VM instances and later test containers will start successfully.
