# Build a Serverless App with Cloud Run that Creates PDF Files

##### Fictitious Business Scenario
- This lab is part of the [Google Cloud Serverless Workshop: Pet Theory Quest](https://www.cloudskillsboost.google/quests/98)
- You are given a fictitious business scenario of a Pet Theory chain of veterinary clinics.
- The customer wants to do serverless migration.
- The clinic currently sends invoices in .docx format to customers. However, many customers complain that they cannot open .docx files.
- Pet Theory wants to build a PDF converter application on Cloud Run. It will automatically convert .docx files uploaded into a storage bucket on Google Cloud Storage into pdf files and stores them in another bucket.

##### Why [Cloud Run](https://cloud.google.com/run/) ?
1. Serverless. Abstracts away infrastructure maintenance.
2. Scales to zero. No cost when not in use.
3. Create/build your own custom binary packages based on containers.
    - means you can build consistent isolated artifacts.
    - means you can write in any programming language (Java, Javascript, Go, Python)

##### Architecture
![overall architecture](https://github.com/TCLee-tech/Google/blob/fe15a9966954213246bb002dd60d057ecceca227/Serverless%20Cloud%20Run%20Development/Build%20a%20Serverless%20App%20with%20Cloud%20Run%20that%20creates%20PDF%20files/Build%20a%20Serverless%20App%20with%20Cloud%20Run%20that%20creates%20PDF%20files%20overall%20architecture.jpeg)

To learn:
1. Create container from a node.js application
2. Build containers using Cloud Build
3. Deploy containers for Cloud Run Services
4. Use event notification/processing with Google Cloud Storage

<Hr>

### Task 2: Enable the Cloud Run API
Run [LibreOffice](https://www.libreoffice.org/) in serverless environment using [Cloud Run](https://www.youtube.com/watch?v=16vANkKxoAU&t=1317s)  
Cloud console > **Navigation menu** > **API & Services** > **Library** > Seach for Cloud Run > **Enable** if not enabled.  

<Hr>

### Task 3: Deploy a simple Cloud Run
1. Clone Pet Theory repository  
`git clone https://github.com/rosera/pet-theory.git`  
2. Change to working directory for lab  
`cd pet-theory/lab03`  
3. Add `"start": "node index.js"` to `"scripts"` section in **package.json** file.  
Click on **Open editor** in Cloud Shell and navigate to `package.json` file. Or, use `nano package.json`.  
```
"scripts": {
    "start": "node index.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
```  
`CYRL + X` then `Y` to save.  

4. Install packages for node.js
```
npm install express
npm install body-parser
npm install child_process
npm install @google-cloud/storage
```
5. Review the codes in **lab03/index.js**.  
This is the main application file. It will be deployed as a Cloud Run service that accepts HTTP POST requests. If the incoming request is a PubSub notification, the app will write the  details to the log. If not, it simply returns 'OK'.
```
const express    = require('express');
const app        = express();
const bodyParser = require('body-parser');

app.use(bodyParser.json());

const port = process.env.PORT || 8080;

app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {
  console.log('OK');
  try {
    const file = decodeBase64Json(req.body.message.data);
    console.log(`file: ${JSON.stringify(file)}`);
  }
  catch (ex) {
    console.log(ex);
  }
  res.set('Content-Type', 'text/plain');
  res.send('\n\nOK\n\n');
})

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}
```
6. Review the **Dockerfile**
```
FROM node:12                        <= informs the base image to use as template
WORKDIR /usr/src/app
COPY package.json package*.json ./
RUN npm install --only=production
COPY . .
CMD [ "npm", "start" ]              <= command to execute: npm start. Runs the command specified in "start" property of "script" object of package.json.
```
7. Build the container and push it to Google Container Registry (GCR)  
`gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter`  
8. To verify that container image is successfully stored in GCR, **Container Registry** > **Images**  
![GCR image](https://github.com/TCLee-tech/Google/blob/b796999e37f12fb166654d491aada626cdec0f66/Serverless%20Cloud%20Run%20Development/Build%20a%20Serverless%20App%20with%20Cloud%20Run%20that%20creates%20PDF%20files/Build%20a%20Serverless%20App%20with%20Cloud%20Run%20that%20creates%20PDF%20files%20Task%203.jpeg)  
9. Deploy container to Cloud Run  
```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --no-allow-unauthenticated \    <= specifies that incoming HTTP request must be authorized
  --max-instances=1
```
  - You will get service URL once deployment complete. Syntax: https://pdf-converter-[hash].a.run.app  

10. Extract service URL to an environment variable **SERVICE_URL**.  
`SERVICE_URL=$(gcloud beta run services describe pdf-converter --platform managed --region us-east1 --format="value(status.url)")`  
11. To check that above is successful, `echo $SERVICE_URL`  
12. Test with annoymous unauthorized POST request to Cloud Run Service URL endpoint  
`curl -X $SERVICE_URL`  
You will get an error message "Your client does not have permission to get the URL".  
13. Test as authorized user  
`curl -X POST -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL`  
You should get an "OK" response if successful.  

<Hr>
    
### Task 4: Trigger your Cloud Run service when a new file is uploaded
##### Create the Cloud Storage Buckets
1. Create a GCS bucket to upload .docx files  
`gsutil mb gs://$GOOGLE_CLOUD_PROJECT-upload`  
    - gsutil is a Python application for interacting with Google Storage  
    -  `mb` for make bucket  
    - syntax: `gs://BUCKET_NAME/OBJECT_NAME`  

2. Create a GCS bucket for the processed PDFs  
`gsutil mb gs://$GOOGLE_CLOUD_PROJECT-processed`  

3. To verify: Cloud console > **Navigation menu** > **Cloud storage** >look for the 2 buckets created  

##### Set up Cloud Storage to send notifications to PubSub on event  
`gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload`  
  - `t` specifies topic "new-doc"  
  - `-f` specifies format "json"  
  - `-e` specifies event: when object finalized  
  - `gs://bucket-name` is the source bucket  

##### Service Account for PubSub to forward notifications and invoke Cloud Run
1. Create Service Account  
  `gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"`  
    - name of service account: pubsub-cloud-run-invoker  

2. Bind IAM permission for Cloud Run invoker role to Service Account  
  ```
  gcloud beta run services add-iam-policy-binding pdf-converter \
    --member=serviceAccount:pubsub-cloud-run-invoker@GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --role=roles/run.invoker \
    --platform managed \
    --region us-east1
  ```  
  -> pdf-converter: name of Cloud Run Service  
  -> `member=serviceAccount:[service account name]@email.com`  

3. Enable project to create PubSub authentication tokens  
    - get project number  
      `gcloud projects list`  
      copy PROJECT_NUMBER for PROJECT_ID starting with "qwiklabs-gcp-"  
    - set project number into an environment variable  
      `PROJECT_NUMBER=[project number]`  
    - bind IAM permission for project to create PubSub authentication tokens  
  ```
    gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
      --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com \
      --role=roles/iam.serviceAccountTokenCreator
  ```
      -> target: $GOOGLE_CLOUD_PROJECT, unique project ID  
      -> `member=serviceAccount:[service account name]@email.com`  

4. Create PubSub subscription    
  ```    
  gcloud beta pubsub subscriptions create pdf-conv-sub \
    --topic new-doc \
    --push-endpoint=$SERVICE_URL \
    --push-auth-service-account=pubsub-cloud-run=invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com
  ```  
  -> name of subscription: pdf-conv-sub  
  -> name of topic: new-doc  
  -> URL of Cloud Run service: in $SERVICE_URL  
  -> push authentication service account: [service account name, i.e. pubsubs-cloud-run-invoker]@email.com  

<Hr>

### Task 5: See if Cloud Run service is triggered when files are uploaded to Cloud Storage
Upload some test files to Cloud Storage and check Cloud Logging.  
1. Upload test files to Cloud Storage  
`gsutil -m cp gs://spls/gsp644/* gs://$GOOGLE_CLOUD_PROJECT-upload`  
    - `cp` means copy
    - `-m` specifies multi-thread parallel uploading
    - syntax: `gsutil cp [flags] [source_url] [destination_url]`

2. Check logs
Cloud console > **Navigation menu** > **Logging** > filter results to **Cloud Run Revision** and click **Apply** > **Run Query**.
    - under **Query results**, look for log entries that starts with `file:`. Each is a dump of file data that PubSub sent to Cloud Run service when a new file is uploaded.
    - note: the containerized application running on Cloud Run does not include functions that use LibreOffice to convert .docx to pdf yet.

3. To delete the files in the `-upload` folder,  
`gsutil rm -m gs://$GOOGLE_CLOUD_PROJECT-upload/*`  

<Hr>

### Task 6: Docker containers
##### Updating Dockerfile
Container requires node Express components and LibreOffice package
  - LibreOffice package was not included in container previously.
![Task 6 image]()

In Dockerfile, add the commands used to install LibreOffice as a `RUN` command.
```
FROM node:12                            <= base image
RUN apt-get update -y \                 <= commands to install LibreOffice
  && apt-get install -y libreoffice \
  && apt-get clean
WORKDIR /usr/src/app
COPY package.json package*.json ./      <= copy from local into container root
RUN npm install --only=production       <= command to install npm
COPY . .
CMD [ "npm", "start" ]                  <= "npm start" runs the command specified by "start" property of "scripts" object in package.json
```
##### Edit index.js to include pdf conversion service
```
const {promisify} = require('util');
const {Storage}   = require('@google-cloud/storage');
const exec        = promisify(require('child_process').exec);
const storage     = new Storage();
const express     = require('express');
const bodyParser  = require('body-parser');
const app         = express();

app.use(bodyParser.json());
const port = process.env.PORT || 8080;

app.listen(port, () => {
  console.log('Listening on port', port);
});

app.post('/', async (req, res) => {                           // main logic of index.js
  try {
    const file = decodeBase64Json(req.body.message.data);     // extract file details from PubSub notification
    await downloadFile(file.bucket, file.name);               //download file from Cloud Storage to temporary virtual memory that behaves like local disk
    const pdfFileName = await convertFile(file.name);         // convert file to PDF
    await uploadFile(process.env.PDF_BUCKET, pdfFileName);    // upload converted PDFs to Cloud Storage. process.env.PDF_BUCKET environment variable contains name of GCS bucket for processed PDFs.
    await deleteFile(file.bucket, file.name);                 // delete the original file from Cloud Storage
  }
  catch (ex) {
    console.log(`Error: ${ex}`);
  }
  res.set('Content-Type', 'text/plain');
  res.send('\n\nOK\n\n');
})

function decodeBase64Json(data) {
  return JSON.parse(Buffer.from(data, 'base64').toString());
}

async function downloadFile(bucketName, fileName) {
  const options = {destination: `/tmp/${fileName}`};
  await storage.bucket(bucketName).file(fileName).download(options);
}

async function convertFile(fileName) {
  const cmd = 'libreoffice --headless --convert-to pdf --outdir /tmp ' + `"/tmp/${fileName}"`;
  console.log(cmd);
  const { stdout, stderr } = await exec(cmd);
  if (stderr) {
    throw stderr;
  }
  console.log(stdout);
  pdfFileName = fileName.replace(/\.\w+$/, '.pdf');
  return pdfFileName;
}

async function deleteFile(bucketName, fileName) {
  await storage.bucket(bucketName).file(fileName).delete();
}

async function uploadFile(bucketName, fileName) {
  await storage.bucket(bucketName).upload(`/tmp/${fileName}`);
}
```
##### Build container and deploy to Cloud Run
1. Build container containing LibreOffice package.
`gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter`
2. Deploy to Cloud Run
```
gcloud run deploy pdf-converter \                       //name of cloud run service: pdf-converter
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --max-instances=1 \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed //set environment variable PDF_BUCKET to Cloud Storage bucket for processed files
```

### Task 7: Testing the pdf-conversion service
1. Test with HTTP POST using [cURL](https://curl.se/docs/manpage.html)
`curl -X POST -H "Authhorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL`
  - `-X` specifies the request method to use
  - `-H` refers to extra header info to send. It is added to the regular request headers.
You should get an "OK" response if Cloud Run deployment was successful.
2. Test with files.
  - LibreOffice can convert many file types into PDFs: docx, xlsx, jpg, png, gif, etc
  - upload some sample files into "-upload" Cloud Storage bucket
  - `gsutil -m cp gs://spls/gsp644/* gs://$GOOGLE_CLOUD_PROJECT-upload`
  - To verify, Cloud console > **Navigation menu** > **Cloud Storage** > open `-upload` bucket > click **Refresh** button a few times to see uploaded files deleted as they are converted into PDFs. Open `-processed` bucket to check if it contains PDF versions of all files. Open a few PDF files to check if they are converted properly.

