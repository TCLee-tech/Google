# Using a Global Load Balancer with Cloud Run [APPRUN]

### Overview

![overview](https://github.com/TCLee-tech/Google/blob/15a7a633ba235470a235a23d703c86e40d8a01ea/Application%20Development%20with%20Cloud%20Run/Using%20Global%20Load%20Balancer%20with%20Cloud%20Run%20overview.jpeg)

Serverless Network Endpoint Groups (NEGs) allow you to use Google Cloud serverless services with external HTTP(S) Load Balancing. After you have configured a load balancer with the serverless NEG backend, requests to the load balancer are routed to the serverless app backend.

In this lab you will learn how to set up and use an HTTP global load balancer with Cloud Run.

**To learn:**
1. Enable the Cloud Run API.
2. Create and deploy a sample application in Cloud Run.
3. Configure a Serverless Network Engpoint Group (NEG).
4. Create a global load balancer with a serverless NEG backend.
5. Route requests for the sample application through the global load balancer.

**Basic Linux Commands**

![linux commands](https://github.com/TCLee-tech/Google/blob/16b08cb2b28816331ac8a8575b8fd010bfe250d0/Application%20Development%20with%20Cloud%20Run/Using%20Global%20Load%20Balancer%20with%20Cloud%20Run%20linux%20commands.jpeg)

<hr>

### Task 1. Configure the environment
 We will enable the Cloud Run API and set the compute region.
 1. Enable Cloud Run API:  
 `gcloud services enable googleapis.com`  
 or use **API & Services** section of console to enable Cloud Run API.  
 2. Set compute and Cloud Run regions.  
 `gcloud config set compute/region us-central1`  
 `gcloud config set run/region us-central1`  
 3. Create a LOCATION environment variable.  
 `LOCATION="us-central1"`  

 <hr>

 ### Task 2. Write a simple application
For this lab, we will create a simple Cloud Run Python "Hello, World" app and deploy it as a Cloud Run service in the us-central1 region. This simple app will become the target for an external HTTP load balancer which will use a serverless NEG backend to route requests to that service.

1. In Cloud Shell create a new directory named `helloworld`, then move your view into that directory:  
`mkdir helloworld && cd helloworld`  
2. Use `vi, emac, nano` or Cloud Shell Editor to create a `main.py` file.  
Click on the "helloworld" folder if you are using Cloud Shell Editor.  
`nano main.py` to create file and add the following content:  
```
import os
from flask import Flask
app = Flask(__name__)
@app.route("/")
def hello_world():
    name = os.environ.get("NAME", "World")
    return "Hello {}!".format(name)
if __name__ == "__main__":
    app.run(debug=True, host="0.0.0.0", port=int(os.environ.get("PORT", 8080)))
```
`CTRL+X` then `Y` to save and exit.  
This code responds to requests on port 8080 with a "Hello World" greeting. The app is now ready to be containerized and uploaded to Artifact Registry.  

<hr>

### Task 3. Containerize your app and upload it to Container Registry (Artifact Registry)
1. Create `Dockerfile` in same directory as `main.py` file and add the following content:
```
# Use the official lightweight Python image.
# https://hub.docker.com/_/python
FROM python:3.9-slim
# Allow statements and log messages to immediately appear in the Knative logs
    ENV PYTHONUNBUFFERED True
# Copy local code to the container image.
ENV APP_HOME /app
WORKDIR $APP_HOME
COPY . ./
# Install production dependencies.
RUN pip install Flask gunicorn
# Run the web service on container startup. Here we use the gunicorn webserver, with one worker process and 8 threads.
# For environments with multiple CPU cores, increase the number of workers to be equal to the cores available.
# Timeout is set to 0 to disable the timeouts of the workers to allow Cloud Run to handle instance scaling.  
CMD exec gunicorn --bind :$PORT --workers 1 --threads 8 --timeout 0 main:app  
```
This Python Dockerfile starts a Gunicorn web server which listens on the port defined by the PORT environmental variable set in the main.py file (port 8080).    
2. Now, build container image using Cloud Build, by running following command **from the directory containing the Dockerfile**:  
`gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld`  

<hr>

### Task 4. Deploy your container to Cloud Run

1. Deploy container image to Cloud Run.
`gcloud run deploy --image gcr.io/$GOOGLE_CLOUD_PROJECT/helloworld`  
You can view your project ID by running the command `gcloud config get-value project`.  
2. The API should already be enabled, but if you are prompted to enable the API, reply `y`.
3. You will then be prompted for the service name: press **Enter** to accept the default name, `helloworld`.
4. If prompted for region: in this case, select us-central1.  
5. You will be prompted to allow unauthenticated invocations: respond `y`.  
6. Wait a few moments until the deployment is complete. Upon success, the command line displays the service URL.  
7. Visit your deployed container by opening the service URL in a web browser.  

<hr>

### Task 5. Reserve an external IP address
Now that your services are up and running, you need to set up a global static external IP address that your customers use to reach your load balancer.

A static external IP address provides a single address to reach your serverless app. Reserving an IP address would also be essential if you were using a custom domain for your serverless app.

1. Use the following command to reserve your static IP address:  
`gcloud compute addresses create example-ip --ip-version=IPV4 --global`  
2. To verify by displaying IP address reserved:  
`gcloud compute addresses describe example-ip --format="get(address)" --global`  

<hr>

### Task 6: Create the external HTTP load balancer
Load balancers use a serverless Network Endpoint Group (NEG) backend to direct requests to a serverless Cloud Run service.
So, let's first create our severless NEG for your serverless Python app from Task 2.
1. To create a serverless NEG with a Cloud Run service:
```
gcloud compute network-endpoint-groups create myneg \   <=NEG named myneg
    --region=$LOCATION \
    --network-endpoint-type=serverless \
    --cloud-run-service=helloworld              <= refering to the Cloud Run service
```
2. Create a backend service for HTTP load balancer:  
`gcloud compute backend-services create mybackendservice --global`  
3. Add the serverless NEG as a backend to the backend service:  
```
gcloud compute backend-services add-backend mybackendservice \  <= name of backend is "mybackendservice"
    --global \
    --network-endpoint-group=myneg \                <= name of serverless NEG to add
    --network-endpoint-group-region=$LOCATION
```
4. Create URL map to route incoming requests to the backend service:
```
gcloud compute url-maps create myurlmap \
    --default-service mybackendservice      <= name of backend service
```

If you had more than one backend service, you could use host rules to direct requests to different services based on host name, or you could set up path matchers to direct requests to different services based on the requested path. For this lab, there is only one backend service, so none of these is necessary.

5. Create target HTTP(S) proxy [part of frontend] to route requests to your URL map:
```
gcloud compute target-http-proxies create mytargetproxy \
    ---url-map=myurlmap
```
6. Within frontend, create a global forwarding rule to route incoming requests to the proxy:
```
gcloud compute forwarding-rules create myforwardingrule \
    --address=example-ip \      <= example-ip is the static IP address created in Task 5
    --target-http-proxy=mytargetproxy \
    --global \
    --ports=80
```

<hr>

### Task 7. Test your external HTTP load balancer
You can test your HTTP load balancer by going to http://IP_ADDRESS, where IP_ADDRESS is the load balancer's static IP address you reserved in Task 5 in this lab. When you open this URL, you should see the helloworld service homepage.

[Using Python on Google Cloud with Cloud Run](https://www.youtube.com/watch?v=s2TIWIzCftM&list=PLIivdWyY5sqJwq_pgOxcHzusWjXDVCEiX)
