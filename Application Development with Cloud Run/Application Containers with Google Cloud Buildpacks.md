# Application Containers with Google Cloud Buildpacks [APPRUN]

To learn:
1. Build a container image from source code using Cloud Native Buildpacks pack CLI.
2. Use Docker to deploy the container image and run the container locally
3. Deploy container from source code using Code Run

Basic Linux commands
- Command --> Action
1. mkdir (make directory)	create a new folder
2. cd (change directory)	change location to another folder
3. ls (list )	list files and folders in the directory
4. cat (concatenate)	read contents of a file without using an editor
5. apt-get update	update package manager library
6. ping	signal to test reachability of a host
7. mv (move ) moves a file.	
8. cp (copy) makes a file copy
9. pwd (present working directory)	returns your current location	
10. sudo (super user do)	gives higher administration privileges
11. to edit existing files, use vi, nano, emac, or Cloud Shell editor.

<hr>

### Task 1. Enable the Cloud Run API and configure your Shell environment
1. Enable Cloud Run API
`gcloud services enable run.googleapis.com`
or, console > **APIs & Services** section
2. Set compute region
`gcloud config set compute/region us-central1`

### Task2: List the pack CLI commands
[pack](https://buildpacks.io/docs/tools/pack/) is tool maintained by the Cloud Native Buildpacks project.
pack CLI has been pre-installed.
Enter `pack` in Cloud Shell to list commands.

### Task 3. Clone the Buildpack sample repository
Google provides a Buildpack Repository containing sample application codes.
  - written in Javascript, python, Java, Go, .NET (C#)
  - for you to practice building container images, and deploying applications.
1. Clone repository of sample application codes
`git clone https://github.com/GoogleCloudPlatform/buildpack-samples.git`

# Build container image using pack CLI
```
cd buildpack-samples/sample-node
pack build --builder=gcr.io/buildpacks/builder sample-node
```
  - example for Javascript node application

# Run newly built container in [Docker](https://docs.docker.com/engine/reference/commandline/run/)
`docker run -it -e PORT=8080 -p 8080:8080 sample-node`
  - `-it` opens shell inside container. `-i` is to keep STDIN open even if not attached. `-t` is to allocate a pseudo-TTY.
  - `-e` exposes port 8080
  - `-p 8080:8080` is -p <host_port>:<container_port>. It publishes container port to host.
  - `sample-node` is image name  
To verify, in Cloud Shell > **Web preview** > **Preview on port 8080**.
`CTRL + C` to stop node application.

### Task 4. Build and run your sample app on Cloud Run
Cloud Run can deploy from source code using a single command.
- it builds container image  > push to Artifact Registry > deploy to Cloud Run
`gcloud beta run deploy --source .`
  - when prompted, 
    - confirm service name by pressing **Enter**
    - choose `us-central1` region
    - if asked to allow unauthenticated invocations, type `Y` and press `**Enter**
Click on the service URL to verify.


Some references:

[Building containers](https://cloud.google.com/run/docs/building/containers)

[Cloud Run](https://cloud.google.com/run)

[Quick start guides to build and deploy on Cloud Run for Go, Node.js, Python, Java, C#, C++, PHP, Ruby, Shell](https://cloud.google.com/run/docs/quickstarts)

[Developing Cloud Run services](https://cloud.google.com/run/docs/developing)

[Kubernetes / K8S](https://cloud.google.com/learn/what-is-kubernetes)
- containers orchestration
- Docker is open industry standard for packaging applications in containers.

Another serverless product:
[Cloud Functions pro tips: Building idempotent functions](https://cloud.google.com/blog/products/serverless/cloud-functions-pro-tips-building-idempotent-functions)