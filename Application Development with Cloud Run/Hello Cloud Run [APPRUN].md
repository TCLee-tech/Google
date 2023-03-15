# Hello Cloud Run [APPRUN]

[Cloud Run](https://cloud.google.com/run)
  - lets you run stateless containers
    - invoked by HTTP requests
  - is serverless
    - abstracts away infrastructure management
  - built from [Knative](https://cloud.google.com/knative/)
  - containers can run fully managed on Cloud Run
    - or in GKE cluster

Cloud Run is built from [Knative](https://cloud.google.com/knative/). 
Knative is for building and running serverless applications on Kubernetes.
- containers
- cached artifacts for faster deployment
- infrastructure abstraction
- cloud-native or on prem
- scale-to-zero
- auto-scaling
- pub-sub
- loosely coupled event-driven framework
- pluggable for CICD, logging and monitoring

To learn:
1. Engable Cloud Run API
2. Create simple node.js application that can be deployed as a serverless and stateless container.
3. Containerize application and upload to Artifact Registry
4. Deploy container image to Cloud Run
5. Delete unneeded images to avoid extra storage charges

<Hr>

### Task 1: Enable the Cloud Run API and configure Cloud Shell environment
1. Either
    - `gcloud services enable run.googleapis.com` or 
    - Cloud console > **APIs & Services**  
2. Set compute region  
`gcloud config set compute/region us-central1`  
3. Create LOCATION environment variable  
`LOCATION="us-central1"`  

<Hr>
  
### Task 2: Write sample application
Create an Express node.js application that responds to HTTPS requests  
1. Create a `helloworld` directory  
`mkdir helloworld && cd helloworld`
2. Create `package.json` file.
To edit file, use `vi`, `emac`, `nano` or Cloud Shell editor by clicking **Open Editor** in Cloud Shell.  
`nano package.json`

```
{
"name": "helloworld",
"description": "Simple hello world sample in Node",
"version": "1.0.0",
"main": "index.js",

"scripts": {
    "start": "node index.js",
},

"author": "Google LLC",
"license": "Apache-2.0",

"dependencies": {
    "express": "^4.17.1"    //dependency on Express web application framework
}

}
```
`CTRL + X` then `Y` to save `package.json`.

3. Create `index.js` file.  
`nano index.js`

```
const express = require('express');
const app = express():
const port = process.env.PORT || 8080;
app.get('/', (req, res) => {
    const name = process.env.NAME || 'World';
    res.send('Hello ${name}!');
});
app.listen(port, () => {
    console.log('helloworld: listening on port ${port}');
});
```
`CTRL + X` then `Y` to save `index.js`.

<Hr>
  
### Task 3: Containerize your app and upload it to Artifact Registry
1. Create `Dockerfile`  
`nano Dockerfile`

```
# Use the official lightweight Node.js 12 image
# https://hub.docker.com/_/node
FROM node:12-slim

# Create and change to the app directory
WORKDIR /usr/src/app

# Copy application dependency manifests to the container image.
# A wildcard is used to ensure copying both package.json AND package-lock.json (when available).
# Copying this first prevents re-running npm install on every code change.
COPY package*.json ./

# Install production dependencies
# If you add a package-lock.json, speed your build by switching to 'npm.ci'.
# RUN npm ci --only=production
RUN npm install --only=production

# Copy local code to the container image
COPY . ./

# Run the web service on container startup.
CMD [ "npm", "start" ]
```
`CTRL + X` then `Y` to save `Dockerfile`

2. Build container image using Cloud Build. Run from directory containing `Dockerfile`  
`gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld`  
    - container building happens in a series of steps
3. To verify, can list all container images associated with current project.  
`gcloud container images list`
4. To run and test the application locally from Cloud Shell, use `docker run`  
`docker run -d -p 8080:8080 gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld`
    - `-d` option is to run docker container in background in "detached" mode. Container exits when root process used to run container exits.
    - `-p` option is to PUBLISH hostPort: containerPort
    - [Docker run reference](https://docs.docker.com/engine/reference/run/)  
To view output, `curl localhost:8080` or in Cloud Shell > **Web preview** > **Preview on port 8080**  

Note: Run `gcloud auth configure-docker` if docker run cannot pull remote container image.

<Hr>
  
### Task 4: Deploy to Cloud Run
1. Deploy container image to Cloud Run  
`gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld --allow-unauthenticated --region=$REGION`
    - the `allow-unauthenticated` flag allows unauthenticated public access to service
    - Cloud Run will automatically and horizontally scale containers to handle received requests. Can scale down to zero when no demand.
    - you pay for **CPU, memory and networking** when handling requests
2. Verify.  
    - access deployed app at service URL `https://helloworld-hash.run.app`
    - Check from Cloud console: **Navigation menu** > **Cloud Run** > look for `helloworld` service.

### Task 5: Clean Up
You will not be charged if Cloud Run service is not in use.
However, you will be charged as long as you store container image in Artifact Repository.
1. To delete container image,  
`gcloud container images delete gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld`
2. To delete `helloworld` Cloud Run service,  
`gcloud run services delete helloworld --region=us-central1`






