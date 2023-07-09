# AHYBRID-132: Receive Pub/Sub events with Eventarc and Cloud Run

#### Overview
In the first part of the lab, you deploy a Cloud Run service and use Eventarc to receive events from Pub/Sub. 
In the second part of the lab, you deploy Cloud Run for Anthos on an Anthos GKE cluster, and use Eventarc to receive events from Pub/Sub.

#### In this lab, you learn how to:
* Deploy a service that receives events to Cloud Run and Cloud Run for Anthos
* Create an event trigger with Eventarc
* Publish a message to a Pub/Sub topic to generate an event, and view it in the Cloud Run and Cloud Run for Anthos logs

<hr>

#### Task 1: Review your Anthos GKE cluster and install Cloud Run for Anthos
An Anthos GKE cluster has already been created for you in this lab. You will review it and install Cloud Run for Anthos. First, verify that the GKE cluster has been registered in an Anthos Fleet. Second, confirm that Anthos Service Mesh has been installed in the cluster. There are prerequisites to install Cloud Run for Anthos. Lastly, install Cloud Run for Anthos.   
   
1. In the Google Cloud console, on the **Navigation menu**, click **Kubernetes Engine > Clusters**. Notice that there is a GKE cluster.  
2. Click **Workloads** and verify that the GKE cluster is running the Anthos Service Mesh components **istio-ingressgateway** and **istiod-asm**.  
3. On the **Navigation menu**, click **Anthos > Clusters** and then verify that the cluster has been registered and appears in the list of **Anthos managed clusters**.  
4. Click **Activate Cloud Shell**. If prompted, click **Continue**.  
5. In Cloud Shell, set the Zone environment variable:    
`C1_ZONE="zone added at start of lab"`  
6. In Cloud Shell, initialize the environment variables:  
```  
export PROJECT_ID=$(gcloud config get-value project)
export C1_NAME="gke"
gcloud config set run/region us-central1
gcloud config set run/platform managed
gcloud config set eventarc/location us-central1
```    
7. Get the credentials for your **gke** GKE cluster:  
`gcloud container clusters get-credentials $C1_NAME --zone $C1_ZONE --project $PROJECT_ID`  
8. Enable Cloud Run for Anthos:   
`gcloud container fleet cloudrun enable --project=$PROJECT_ID`  
9. Enable Eventarc API:  
`gcloud services enable --project=$PROJECT_ID eventarc.googleapis.com`  
10. Install Cloud Run for Anthos on the cluster:    
`gcloud container fleet cloudrun apply --gke-cluster=$C1_ZONE/$C1_NAME`    
If this step fails, wait 30 seconds and try again.    

<hr>

#### Task 2: Deploy a Cloud Run application
1. In Cloud Shell, clone the git repository:
```
git clone https://github.com/GoogleCloudPlatform/nodejs-docs-samples.git
cd nodejs-docs-samples/eventarc/pubsub/
```
2. Build the container with Cloud Build:  
`gcloud builds submit --tag gcr.io/$(gcloud config get-value project)/events-pubsub`
3. Deploy the container image to Cloud Run:
```
gcloud run deploy helloworld-events-pubsub-tutorial \
--image gcr.io/$(gcloud config get-value project)/events-pubsub \
--allow-unauthenticated \
--max-instances=1
```
4. If asked to enable the API, type **y**.  
When the service URL is displayed, the deployment is complete.

<hr>

#### Task 3: Create an Eventarc trigger for Cloud Run
When a message is published to the Pub/Sub topic, an event trigger directs messages to the correct subscriber (receiver service deployed on Cloud Run).
1. In Cloud Shell, create an Eventarc trigger to listen for Pub/Sub messages:
```
gcloud eventarc triggers create events-pubsub-trigger \
--destination-run-service=helloworld-events-pubsub-tutorial \
--destination-run-region=us-central1 \
--event-filters="type=google.cloud.pubsub.topic.v1.messagePublished"
```
This creates a new Pub/Sub topic and a trigger for it called **events-pubsub-trigger**. The Pub/Sub subscription persists regardless of activity and does not expire.  
2. Confirm that the trigger was successfully created:  
`gcloud eventarc triggers list --location=us-central1`  
3. Set the Pub/Sub topic as an environment variable:  
`export RUN_TOPIC=$(gcloud eventarc triggers describe events-pubsub-trigger --format='value(transport.pubsub.topic)')`  
4. Send a message to the Pub/Sub topic to generate an event:   
`gcloud pubsub topics publish $RUN_TOPIC --messasge "Runner"`  
The event is sent to the Cloud Run service which logs the event message.  
5. To view the event message, in the Google Cloud Console, navigate to **Cloud Run**.  
6. Click on the **helloworld-events-pubsub-tutorial** service.  
7. Click on the **Logs** tab and look for "Hello, Runner!" message.   

Congratulations! You have deployed a Cloud Run application and used Eventarc to trigger events from Pub/Sub.  

<hr>

#### Task 4: Prepare the environment for Eventarc and Cloud Run for Anthos
1. Create a Service Account to use when creating triggers for Cloud Run for Anthos:
```
TRIGGER_SA=pubsub-to-anthos-trigger
gcloud iam service-accounts create $TRIGGER_SA
```
2. Grant appropriate roles to the new Service Account:
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:${TRIGGER_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
--role "roles/pubsub.subscriber"

gcloud projects add-iam-policy-binding $PROJECT_ID \
--member "serviceAccount:${TRIGGER_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
--role "roles/monitoring.metricWriter"
```
3. Enable GKE destinations for Eventarc:
`gcloud eventarc gke-destinations init`
4. At the prompt to bind the required roles, type **y**.
The follow roles are bound:
* roles/compute.viewer
* roles/container.developer
* roles/iam.serviceAccountAdmin

<hr>

#### Task 5: Deploy the Cloud Run for Anthos application
Deploy the container image to Cloud Run for Anthos:
```
gcloud run deploy subscriber-service \
--cluster $C1_NAME \
--cluster-location $C1_ZONE \
--platform gke \
--image gcr.io/$(gcloud config get-value project)/events-pubsub
```
If the command fails, the Anthos GKE cluster is not yet ready to accept Cloud Run for Anthos services. Wait a few minutes and try again.  

<hr>

#### Task 6: Create an Eventarc trigger for Cloud Run on Anthos
1. In Cloud Shell, create an Eventarc trigger that listens for Pub/Sub messages:
```
gcloud eventarc triggers create pubsub-trigger \
--location=us-central1 \
--destination-gke-cluster=$C1-NAME \
--destination-gke-location=$C1_ZONE \
--destination-gke-namespace=default \
--destination-gke-service=subscriber-service \
--destination-gke-path=/ \
--event-filters="type=google.cloud.pubsub.topic.v1.messagePublished" \
--service-account=${TRIGGER_SA}@${PROJECT_ID}.iam.gserviceaccount.com
```
This creates a new Pub/Sub topic and a trigger for it called **pubsub-trigger**. The Pub/Sub subscription persists regardless of activity and does not expire.  
2. Confirm that the trigger was successfully created:  
  `gcloud eventarc triggers list --location=us-central1`  
3. Set the Pub/Sub topic as an environment variable:  
```
export RUN_TOPIC=$(gcloud eventarc triggers describe pubsub-trigger \
--location=us-central1 \
--format='value(transport.pubsub.topic)')
```   
4. Send a message to the Pub/Sub topic to generate an event:    
`gcloud pubsub topics publish $RUN_TOPIC --message "Cloud Run on Anthos" `  
The event is sent to the Cloud Run for Anthos service, which logs the event message.    
5. To view the event messsage in the service logs, on the **Navigation menu**, click **Anthos > Cloud Run for Anthos**.
6. Click on the  **subscriber-service**.
7. Under the **Logs** tab, look for the "Hello, Cloud Run for Anthos!" message.  

Congratulations! You deployed a Cloud Run for Anthos application and used Eventarc to trigger events from Pub/Sub.




