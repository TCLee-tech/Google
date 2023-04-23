# Configuring Egress from a Static Outbound IP Address [APPRUN]

### Overview
By default, a Cloud Run service will connect to external endpoints from the internet using a dynamic IP address pool. 
Some external endpoints require connections originating from a static IP address. Examples: database, API using an IP address-based firewall. Need to configure Cloud Run service to route requests through a static IP address. The default dynamic address pool is not suitable.

This lab teaches the process of configuring Cloud Run service to send requests using a static IP address.

### Prerequisites
These labs are based on intermediate knowledge of Google Cloud. While the steps required are covered in the content, it would be helpful to have familiarity with any of the following products:

- Static IP Addresses

- Cloud Run

### Objectives
To learn:
1. Build an image and run a Cloud Run service using sample code from the Buildpack samples repository
2. Create a subnetwork and a VPC Access Connector
3. Configure network address translation (NAT)
4. Route Cloud Run traffic through VPC network

<hr>

### Task 1. Enable the Cloud Run API and configure your Shell environment for flexibiilty
1. Enable Cloud Run API:  
`gcloud services enable run.googleapis.com`  
or, Cloud console > **APIs & Services** > enable  
2. Set compute region  
`gcloud config set compute/region us-central1`  
3. Create LOCATION environment variable  
`LOCATION="us-central1"`  

<hr>

### Task 2. Create a service
**Clone buildpack-samples repository to local directory**  
`git clone https://github.com/GoogleCloudPlatform/buildpack-samples.git`  

**Build a sample application**  
Build a sample Go application.   
For reference, the Github repo is https://github.com/GoogleCloudPlatform/buildpack-samples/tree/master/sample-go
```
cd buildpack-samples/sample-go
pack build --builder=gcr.io/buildpacks/builder sample-go  
```
- builder is an image that contains all the components needed to execute a build
- there are many builders from Google, Heroku, Packeto. Above is Ubuntu 18 base image with buildpacks for .NET, Go, Java, Node.js, and Python 
- output is a runnable application image available on local Docker daemon

**Run your new build container in Docker**
1. Docker run  
`docker run -it -e PORT=8080 -p 8080:8080 sample-go`  
    - options:  
      `-i` : keep STDIN open even if not attached  
      `-t` : allocate a pseudo-TTY  
      `-e` : set environment variable. In this case, set PORT=8080  
      `-p` : publish container's port:host's port  
      [Reference on docker run](https://docs-stage.docker.com/engine/reference/commandline/run/#options)  
    - "sample-go" is name of container image  

2. Cloud Shell > **Web Preview** > **Preview on port 8080** to see "hello, world" message if successful.
3. Back in Cloud Shell, `CTRL + C` to stop running container.
4. Publish container image so that you can use later.
    - First, set Buildpack as your default Builder, for convenience:  
    `pack set-default-builder gcr.io/buildpacks/builder:v1`  
    - Then, publish image:  
    `pack build --publish gcr.io/$GOOGLE_CLOUD_PROJECT/sample-go`  

<hr>
  
### Task 3. Create a subnetwork
Create a subnetwork for the Serverless VPC Access Connector to reside in. This ensures that other compute resources in the VPC, such as Compute Engine VMs or Google Kubernetes Engine clusters, do not accidentally use the static IP you have configured to access the internet.  

1. First, you will need to find the name of your VPC network by calling up a list of the networks associated with your GCP account:  
`gcloud compute networks list`  
For this lab, the only network available is the `default` network.  
Get the following response -  
> NAME: default  
> SUBNET_MODE: AUTO  
> BGP_ROUTING_MODE: REGIONAL  
> IPV4_RANGE:  
> GATEWAY_IPV4:  

2. Create a subnetwork called `mysubnet` with a CIDR range of `192.168.0.0/28` in the region `us-central1`. The subnets used for VPC access connectors must have a netmask of 28, or you will get an error later in the process.
```
gcloud compute networks subnets create mysubnet \
    --range=192.168.0.0/28 --network=default --region=$LOCATION
```

<hr>

### Task 4. Create a serverless VPC Access Connector
To route your Cloud Run service's outbound traffic to a VPC network, you need to set up a Serverless VPC Access Connector. Name the VPC Access Connector `myconnector` and create it in the subnetwork `mysubnet`:
```
gcloud compute networks vpc-access connectors create myconnector \
  --region=$LOCATION \
  --subnet-project=$GOOGLE_CLOUD_PROJECT \
  --subnet=mysubnet
```
You may be asked to enable vpcaccess.googleapis.com on your project. If so, enter `y` and wait for the process to complete before proceeding.

<hr>

### Task 5. Configure Network Address Translation (NAT)
To route outbound requests to external endpoints through a static IP (which is the main goal of this lab), you must first configure a Cloud NAT gateway.
1. Create a new Cloud Router named `myrouter` to program your NAT gateway:
```
gcloud compute routers create myrouter \
    --network=default \
    --region=$LOCATION
```
2. Next, reserve a static IP address. `myoriginip` is the name being assigned to your IP address resource.  
A reserved IP address resource retains the underlying IP address when the resource it is associated with is deleted and re-created. Using the same region as your Cloud NAT router will help to minimize latency and network costs.  
`gcloud compute addresses create myoriginip --region=$LOCATION`

3. Create the Cloud NAT gateway named `mynat`, by configuring the router `myrouter` to route traffic originating from the VPC network:
```
gcloud compute routers nats create mynat \
    --router=myrouter \
    --region=$LOCATION \
    --nat-custom-subnet-ip-ranges=mysubnet \
    --nat-external-ip-pool=myoriginip
```

<hr>

### Task 6. Route Cloud Run traffic through the VPC network
1. After NAT has been configured, you will deploy your Cloud Run service with the Serverless VPC Access Connector and set the VPC egress to route all traffic through the VPC network:
```
gcloud run deploy sample-go \
    --image=gcr.io/$GOOGLE_CLOUD_PROJECT/sample-go \
    --vpc-connector=myconnector \
    --vpc-egress=all-traffic
```
You may be asked to enable run.googleapis.com for your project. If so, respond `y` and wait for the API to enable.  

2. When asked for a region, choose `us-central1` as this location was used in the environment variable. Choose `y` to allow unauthenticated invocations. It will take a few minutes for the process to complete.  

3. Click on the service URL, and you will see the same "hello, world" message that you saw displayed in the web preview pane in Task 1.  

<hr>

You will have completed
- set up Cloud NAT on your VPC network with a predefined static IP address
- routed all of your Cloud Run service's outbound traffic into your VPC network
- confirmed that 100% of your traffic for the sample-go service is now flowing through your new VPC connector

[Serverless Expeditions video series](https://www.youtube.com/playlist?list=PLIivdWyY5sqJwq_pgOxcHzusWjXDVCEiX)




