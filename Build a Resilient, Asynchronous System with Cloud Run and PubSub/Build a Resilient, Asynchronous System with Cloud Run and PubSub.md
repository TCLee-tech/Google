# Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub

Part of the [Google Cloud Serverless Workshop: Pet Theory Quest](https://www.cloudskillsboost.google/quests/152)

### Architecture
- Pet Theory is a veterinary clinic chain
- They use an external company for lab tests
- When the test results are ready, the lab sends a HTTP(S) POST to Pet Theory's web endpoint for test results.
- Pet Theory will 
  1. receive the HTTP POST request
  2. email the test result to client
  3. SMS the test result to client
- Pet Theory wants to do serverless migration. The isolated components of serverless architecture needed:
  1. Service to handle HTTP request and response for test result(s)
  2. Service to send email
  3. Service to send SMS
  4. PubSub for inter-service communication  
- Each service is a single function

![Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub](https://github.com/TCLee-tech/Google/blob/071734a39a74f249369dd950aa5c7ed7290456a9/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%201%20Image%201.jpg)

### Task 1: Create a PubSub topic
Name of PubSub topic: `new-lab-report`  
![Async w Cloud Run and PubSub Task 1 Image 2](https://github.com/TCLee-tech/Google/blob/52de20e08feb538368068d84e2696a3736678ed1/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%201%20Image%202.jpg)  
Whenever a Cloud Run service publishes a PubSub message, the message **must** be tagged with a **topic**.  
Lab Report Service consumes each HTTPS POST request and publishes a PubSub message for each successful POST.  
To create a PubSub topic,   
  `gcloud pubsub topics create new-lab-report`  
To enable Cloud Run,   
  `gcloud services enable run.googleapis.com`  
Any service (e.g. Email Service and SMS Service in diagram) subscribed to topic will be able to consume notification message from Lab Report Service.  


### Task 2: Build the Lab Report Service
![Async w Cloud Run and PubSub Task 2](https://github.com/TCLee-tech/Google/blob/5180bcea5d9c85e4dd247125c64c3753b478f6ba/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%202.jpg)  

This Cloud Run Service has 2 functions:
  1. Receive lab report HTTPS POST request and respond back.
  2. Publish message on PubSub.  

#### Steps to code Lab Report Service:
1. Git clone repository  
  `git clone https://github.com/rosera/pet-theory.git`  
2. Change to `lab-service` directory  
  `cd pet-theory/lab05/lab-service`  
3. The key files are:
    - package.json
    - index.js
    - Dockerfile
4. Install node packages/dependencies needed to parse incoming HTTPS requests and publish to PubSub. These commands will update `package.json`:
```
npm install express
npm install body-parser
npm install @google-cloud/pubsub
```

5. Update `package.json`. In the "scripts" section, add `"start": "node index.js"` as shown below:
```
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  ```  
  To open package.json with nano editor, `nano package.json`  
  To save, `CTRL+X` then `Y`  

6. Create new file `index.js` and add its code.  
  To create new file, `nano index.js`    
  Then paste the following code:  
```
const {PubSub} = require('@google-cloud/pubsub');
const pubsub = new PubSub();
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {                    //node Express listens to port and handles requests with zero fn that console logs
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {         //Express server handles requests to `/` endpoint with async function
  try {
    const labReport = req.body;             //** block-scoped constant (labReport) extracts HTTPS POST request body
    await publishPubSubMessage(labReport);  //** this expression awaits a function (publishPubSubMessage) that returns a promise. Execution of codes suspended (synchronous effect) until promise fulfilled. 
    res.status(204).send();                 //respond with 204 status once awaited promise fulfilled
  }
  catch (ex) {
    console.log(ex);
    res.status(500).send(ex);
  }
})
async function publishPubSubMessage(labReport) {          //awaited async function
  const buffer = Buffer.from(JSON.stringify(labReport));
  await pubsub.topic('new-lab-report').publish(buffer);
}
```
  `CTRL+X` then `Y` to save and exit `index.js`.
  
References:    
[node Express POST routing](https://expressjs.com/en/guide/routing.html)    
[Javascript const](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/const)  
[Javascript async](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)  

7. Create Dockerfile that defines how to package "Lab Report Service" Cloud Run Service into container.  
  To create new file, `nano Dockerfile`  
  Add the following code:  
```
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```
  `CTRL+X` then `Y` to save and exit.  

#### Deploy the Lab Report Service
1. Create a script named `deploy.sh`: `nano deploy.sh`
2. Paste below commands into it:
```
gcloud builds submit \                                      //Cloud Build - build container with tag
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service

gcloud run deploy lab-report-service \                      //Deploy container as Cloud Run Service with "lab-report-service" name
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/lab-report-service \
  --platform managed \
  --region us-east1 \
  --allow-unauthenticated \
  --max-instances=1
  ```
3. Save and exit: `CTRL+X` then `Y`.
4. Make the `deploy.sh` script executable.  
  `chmod u+x deploy.sh`
    - chmod command changes permission in the file
    - syntax: chmod [user flag] [-+=] [permissions] [file name]
    - `u` for file owner
    - `+` for add
    - `x` for execution permission
    - [Reference for chmod command in linux](https://linuxize.com/post/chmod-command-in-linux/)
5. Deploy script.  
  `./deploy.sh`

#### Testing the Lab Report Service
Simulate 3 HTTPS POSTs by the lab company.  
Each HTTP POST represents a lab report sent to Lab Report Service (Cloud Run).  
Because this is a test, each lab report will only contain an ID.  

Steps:
1. Put the Lab Report Service URL (https://xxx.run.app) in an environment variable to make it easier to work with.  
`export LAB_REPORT_SERVICE_URL=$(gcloud run services describe lab-report-service --platform managed --region us-east1 --format="value(status.address.url)")`  
2. Confirm URL captured in environment variable:  
  `echo $LAB_REPORT_SERVICE_URL`
3. Put the code for 3 HTTP POSTs into a script named `post-reports.sh`:
```
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 12}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 34}" \
  $LAB_REPORT_SERVICE_URL &
curl -X POST \
  -H "Content-Type: application/json" \
  -d "{\"id\": 56}" \
  $LAB_REPORT_SERVICE_URL &
```
Note -
  * cURL, short for client URL, is a tool for transferring data to and from a server. [What is cURL?](https://developer.ibm.com/articles/what-is-curl-command/)  
  * `-X` specifies custom request method to use when communicating with HTTP server. Default is GET.  
  * `-H` specifies header to be added to the info sent. Custom addition to the regular request headers. Will replace internal header if name is the same.  
  * `-d` specifies data to pass to the HTTP server.  
  * [Refer to curl.1 man page](https://curl.se/docs/manpage.html)  

4. Make the `post-reports.sh` script executable:  
`chmod u+x post-reports.sh`  
5. Run script:  
`./post-reports.sh`
6. Check Cloud Run log to verify.  
  In Cloud console > **Navigation menu** > **Cloud Run** > **Services** > **lab-report-service** > Click on the **LOGS** tab > check if there are three POST 204 entries.

<hr>

### Task 3. The Email Service
![Email Service](https://github.com/TCLee-tech/Google/blob/80e35c533e797f95717671da672f3c7a0868acf1/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%203%20Image%201.jpg)

This Cloud Run Service has 2 functions:
  1. Receive messages from subscribed PubSub topic and respond back.
  2. sendEmail() function.  

The key files are:
  - package.json
  - index.js
  - Dockerfile

#### Add codes for the Email Service
1. Change to Email Service directory  
  `cd ~/pet-theory/lab05/email-service`
2. Install packages/dependencies for node.js Express framework and to handle incoming HTTPS requests. These commands will update `package.json`:  
```
npm install express
npm install body-parser
```
3. Update `package.json`. In the "scripts" section, add `"start": "node index.js"` as shown below:  
```
  "scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  ```  
To open package.json with nano editor: `nano package.json`  
To save: `CTRL+X` then `Y`.  

4. Create `index.js`.  
  To create a new file: `nano index.js`  
  Then paste in the following code:  
```
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async (req, res) => {
  const labReport = decodeBase64Json(req.body.message.data);  //function (decodeBase64Json) that parse request body containing lab report data.
  try {
    console.log(`Email Service: Report ${labReport.id} trying...`);
    sendEmail();                                                      //function to send email. Add codes later.
    console.log(`Email Service: Report ${labReport.id} success :-)`);
    res.status(204).send();                                           //If PubSub message processed successfully, send HTTP 204 response back to PubSub.
  }
  catch (ex) {
    console.log(`Email Service: Report ${labReport.id} failure: ${ex}`);
    res.status(500).send();                                           //If PubSub message not processed (exception), return 500 status code. So that PubSub knows message not processed and can re-post later.
  }
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
function sendEmail() {
  console.log('Sending email');
}
```
To save and exit: `CTRL+X` then `Y`  

5. Create a `Dockerfile` and add the code below:  
```
FROM node:10
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

#### Deploy the Email Service
1. Create a new script `deploy.sh` and add the following codes:
```
gcloud builds submit \                                  //Build and tag container using Cloud Build
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/email-service

gcloud run deploy email-service \                       //Deploy Cloud Run Service using container image
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/email-service \
  --platform managed \
  --region us-east1 \
  --no-allow-unauthenticated \
  --max-instances=1
```
To create new `deploy.sh` with nano editor: `nano deploy.sh`    
To save: `CTRL+X` then `Y`  

2. Make `deploy.sh` executable:  
`chmod u+x deploy.sh`  
3. Deploy the script  
`./deploy.sh`

#### Configure PubSub to trigger the Email Service
![Async Cloud Run and PubSub Task 3 Image 2](https://github.com/TCLee-tech/Google/blob/62f855372d2a798091e50c78d2764e89f197dfdf/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%203%20Image%202.jpg)

To link PubSub message from PubSub topic to Cloud Run Service, need
1. service account that will invoke Cloud Run Service
2. iam policy for service account to invoke Cloud Run
3. iam policy for PubSub to create authentication tokens
4. create PubSub subscription for the Cloud Run Service  
<br>

So, 
1. Create service account for PubSub.  
`gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"`  
2. Add IAM policy for service account to invoke Cloud Run in response to PubSub messages.  
`gcloud run services add-iam-policy-binding email-service --member=serviceAccount:pubsub-cloud-run-invoker@GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-east1 --platofrm managed`  
3. Add IAM policy to enable project to create PubSub authentication tokens.  
    - `PROJECT_NUMBER=$(gcloud projects list --filter="qwiklabs-gcp" --format='value(PROJECT_NUMBER)')`  
    - `gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com --role=roles/iam.serviceAccountTokenCreator`  
4. Create PubSub subscription.  
    - put URL of Email Service in an environment variable  
      `EMAIL_SERVICE_URL=$(gcloud run services describe email-service --platform managed --region us-east1 --format="value(status.address.url)")`  
    - confirm EMAIL_SERVICE_URL  
      `echo $EMAIL_SERVICE_URL`  
    - PubSub subscription  
      `gcloud pubsub subscriptions create email-service-sub --topic new-lab-report --push-endpoint=$EMAIL_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker@GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com`  

#### Test Lab Report Service and Email Service together
Use the `post-reports.sh` script that simulates HTTP POSTs from the external lab company.  
`~/pet-theory/lab05/lab-service/post-reports.sh`  
In Cloud console **Nagivation Menu** > **Cloud Run**, will see **email-service** and **lab-report-service**. Click on **email-service** > **LOGS**. You will see the result of email-service being triggered by PubSub.  

<hr>

### Task 4: The SMS Service
![Async w Cloud Run and PubSub Task 4 Image 1](https://github.com/TCLee-tech/Google/blob/24cf3c9ff569bcc3f0dd476652f1e48a1faf4b11/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%204%20Image%201.jpg)

#### Add codes for the SMS Service
1. Change to directory for SMS service.  
`cd ~/pet-theory/lab05/sms-service`
2. Install pacakages/dependencies to handle incoming HTTPS requests.  
```
npm install express
npm install body-parser
```
3. Add `"start": "node index.js"` to "scripts" section in `package.json`.  
`nano package.json` to edit file.  
```
...
"scripts": {
  "start": "node index.js",                             <= //add
  "test": "echo \"Error: no test specified\" && exit 1"
},
...
```
4. Create `index.js` and add the following codes:
```
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
app.use(bodyParser.json());
const port = process.env.PORT || 8080;
app.listen(port, () => {
  console.log('Listening on port', port);
});
app.post('/', async(req, res) => {
  const labReport =  decodeBase64Json(req.body.message.data);
  try {
    console.log('SMS Service: Report ${labReport.id} trying...');
    sendSMS();
    console.log('SMS Service: Report ${labReport.id} success :-)');
    res.status(204).send();
  }
  catch (ex) {
    console.log('SMS Service: Report ${labReport.id} failure: ${ex}');
    res.status(500).send();
  }
})
function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
function sendSMS() {
  console.log('Sending SMS');
}
```
5. Create Dockerfile, adding codes below:
```
From node:10
WORKDIR /use/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]
```

#### Deploy the SMS Service
1. Create deployment script `deploy.sh` and add the following codes:
```
gcloud builds submit \
  --tag gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service
gcloud run deploy sms-service \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/sms-service \
  --platform managed \
  --region us-east1 \
  --no-allow-unauthenticated \
  --max-instances=1
```
2. Make `deploy.sh` executable.  
`chmod u+x deploy.sh`  
3. Deploy the Cloud Run service.  
`./deploy.sh`  

#### Configure Cloud PubSub to trigger SMS Service
![Async w Cloud Run and PubSub Task 4 Image 2](https://github.com/TCLee-tech/Google/blob/eefaf668f4a308a2c515cded29b768e01e337d4e/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Async%20w%20Cloud%20Run%20and%20PubSub%20Task%204%20Image%202.jpg)

To link PubSub message from PubSub topic to Cloud Run Service, need
1. service account that will invoke Cloud Run Service
2. iam policy for service account to invoke Cloud Run
3. iam policy for PubSub to create authentication tokens
4. create PubSub subscription for the Cloud Run Service

So,  
(1) was created in previous Email Service Task 3  
2. Enable IAM permission to invoke SMS Service  
`gcloud run services add-iam-policy-binding sms-service --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com --role=roles/run.invoker --region us-east1 --platform managed`  
(3) was created in precious Email Service Task 3  
4. Create PubSub subscription for SMS Service  
  `SMS_SERVICE_URL=$(gcloud run services describe sms-service --platform managed --region us-east1 --format="value(status.address.url)")`
  `echo $SMS_SERVICE_URL`   
  `gcloud pubsub subscriptions create sms-service-sub --topic new-lab-report --push-endpoint=$SMS_SERVICE_URL --push-auth-service-account=pubsub-cloud-run-invoker$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com`  

#### Test Lab Report Service and SMS Service together
Use the `post-reports.sh` script that simulates HTTP POSTs from the external lab company.  
`~/pet-theory/lab05/lab-service/post-reports.sh`  
In Cloud console **Nagivation Menu** > **Cloud Run**, will see **sms-service**, **email-service** and **lab-report-service**. Click on **sms-service** > **LOGS**. You will see the result of sms-service being triggered by PubSub.  

<hr>

### Task 5: Test the resiliency of the system
Simulate what happens when a service fails: deploy a bad version of Email Service.  
1. Go to `email-service` directory.  
`cd ~/pet-theory/lab05/email-service`  
2. Add `throw 'Email server is down'` to sendEmail() function in `index.js`. This will throw an exception.  
```
...
function sendEmail() {
  throw 'Email server is down';
  console.log('Sending email');
}
...
```
3. Deploy  
`./deploy.sh`  
4. HTTPS POST lab data to system again using `post-reports.sh` script.  
`~/pet-theory/lab05/lab-service/post-reports.sh`  
5. In Cloud console **Nagivation Menu** > **Cloud Run**, look at **email-service** > **LOGS**. You will see the service returning service code 500, and PubSub keep retrying calling this service. If you check logs for SMS service, you will see that it works fine.  
6. Remove `throw 'Email server is down'` in `index.js` to correct the error and let Email Service execute correctly.  
7. Deploy the fixed version of Email Service.  
`./deploy.sh`  
8. Check the logs for email-service. You will see Email Service returning status code 204, PubSub stopped invoking the service once it received a success response.  
    - Notice: PubSub kept retrying until successful.

![Cloud Run Email Service fail and success logs](https://github.com/TCLee-tech/Google/blob/e777fd11849e41e7d743c081b12d8591dac69e85/Build%20a%20Resilient,%20Asynchronous%20System%20with%20Cloud%20Run%20and%20PubSub/Cloud%20Run%20Email%20Service%20fail%20and%20success%20log.jpg)

### Take Aways
1. If micro-services communicate **asynchronously** via **PubSub**, the system can be more **resilient**.
2. The Cloud Run services are independent of each other, thanks to use of PubSub. For example, if customers want to receive lab results via another messaging service, it can be added without affecting others.
3. **PubSub** handles **retries**. The services don't have to. Services only return a status code: success or failure.
4. Because PubSub retries, if a service goes down, the system automatically "heals" itself when that service comes back online.
