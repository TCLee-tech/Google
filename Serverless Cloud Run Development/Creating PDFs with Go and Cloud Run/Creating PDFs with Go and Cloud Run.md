# Creating PDFs with Go and Cloud Run
Aim: To build a web application running on serverless Cloud Run. The app automatically converts files uploaded into a Google Cloud Storage (GCS)folder into PDFs and store them in a different GCS folder.

To learn:
1. Convert Go application into a container.
2. Build containers using Cloud Build
3. Deploy a Cloud Run service using built container
4. Create Service Accounts and add permissions
5. Use event processing with Google Cloud Storage

### Architecture
![overall](https://github.com/TCLee-tech/Google/blob/0af88a9e890c110fecafb9083188160e40a9657a/Serverless%20Cloud%20Run%20Development/Creating%20PDFs%20with%20Go%20and%20Cloud%20Run/overall%20architecture.jpg)

### Using Googleapis
APIs that should be enabled:
| Name  | API   |
| ---   | ---   |
| Cloud Build | cloudbuild.googleapis.com |
| Cloud Storage | storage-component.googleapis.com |
| Cloud Run | run.googleapis.com |
<hr>

### Task 1: Get the source code
1. Activate lab account  
`gcloud auth list --filter=status:ACTIVE --format="value(account)"`  
2. Clone Pet Theory repository  
`git clone https://github.com/Deleplace/pet-theory.git`  
3. Change to directory for lab  
`cd pet-theory/lab03`  
<hr>

### Task 2: Create the application
Add the codes for the application written in Go to `server.go`  
To edit `server.go`, you can click on `Open Editor` in Cloud Shell. Then click on `Open in a new window`.  
Navigate to `pet-theory` > `lab03` > `server.go`  
```
package main
import (
      "fmt"
      "io/ioutil"
      "log"
      "net/http"
      "os"
      "os/exec"
      "regexp"
      "strings"
)
func main() {
      http.HandleFunc("/", process)
      port := os.Getenv("PORT")
      if port == "" {
              port = "8080"
              log.Printf("Defaulting to port %s", port)
      }
      log.Printf("Listening on port %s", port)
      err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
      log.Fatal(err)
}
func process(w http.ResponseWriter, r *http.Request) {
      log.Println("Serving request")
      if r.Method == "GET" {
              fmt.Fprintln(w, "Ready to process POST requests from Cloud Storage trigger")
              return
      }

      //
      // Read request body containing Cloud Storage object metadata
      //
      gcsInputFile, err1 := readBody(r)
      if err1 != nil {
              log.Printf("Error reading POST data: %v", err1)
              w.WriteHeader(http.StatusBadRequest)
              fmt.Fprintf(w, "Problem with POST data: %v \n", err1)
              return
      }

      //
      // Working directory (concurrency-safe)
      //
      localDir, errDir := ioutil.TempDir("", "")
      if errDir != nil {
              log.Printf("Error creating local temp dir: %v", errDir)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Could not create a temp directory on server. \n")
              return
      }
      defer os.RemoveAll(localDir)

      //
      // Download input file from Cloud Storage
      //
      localInputFile, err2 := download(gcsInputFile, localDir)
      if err2 != nil {
              log.Printf("Error downloading Cloud Storage file [%s] from bucket [%s]: %v",
gcsInputFile.Name, gcsInputFile.Bucket, err2)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]",
gcsInputFile.Name, gcsInputFile.Bucket)
              return
      }

      //
      // Use LibreOffice to convert local input file to local PDF file.
      //
      localPDFFilePath, err3 := convertToPDF(localInputFile.Name(), localDir)
      if err3 != nil {
              log.Printf("Error converting to PDF: %v", err3)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error converting to PDF.")
              return
      }

      //
      // Upload the freshly generated PDF to Cloud Storage
      //
      targetBucket := os.Getenv("PDF_BUCKET")
      err4 := upload(localPDFFilePath, targetBucket)
      if err4 != nil {
              log.Printf("Error uploading PDF file to bucket [%s]: %v", targetBucket, err4)
              w.WriteHeader(http.StatusInternalServerError)
              fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]",
gcsInputFile.Name, gcsInputFile.Bucket)
              return
      }

      //
      // Delete the original input file from Cloud Storage.
      //
      err5 := deleteGCSFile(gcsInputFile.Bucket, gcsInputFile.Name)
      if err5 != nil {
              log.Printf("Error deleting file [%s] from bucket [%s]: %v", gcsInputFile.Name,
gcsInputFile.Bucket, err5)
         // This is not a blocking error.
         // The PDF was successfully generated and uploaded.
      }
      log.Println("Successfully produced PDF")
      fmt.Fprintln(w, "Successfully produced PDF")
}
func convertToPDF(localFilePath string, localDir string) (resultFilePath string, err error) {
      log.Printf("Converting [%s] to PDF", localFilePath)
      cmd := exec.Command("libreoffice", "--headless", "--convert-to", "pdf",
              "--outdir", localDir,
              localFilePath)
      cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
      log.Println(cmd)
      err = cmd.Run()
      if err != nil {
              return "", err
      }
      pdfFilePath := regexp.MustCompile(`\.\w+$`).ReplaceAllString(localFilePath, ".pdf")
      if !strings.HasSuffix(pdfFilePath, ".pdf") {
              pdfFilePath += ".pdf"
      }
      log.Printf("Converted %s to %s", localFilePath, pdfFilePath)
      return pdfFilePath, nil
}
```
To build the Go application:   
`go build -o server`
<hr>

### Task 3: Create a PDF conversion service
Use open-source LibreOffice to convert .docx documents into PDFs.  
Add LibreOffice package into container with Go application.
![LibreOffice in container with Go application](https://github.com/TCLee-tech/Google/blob/0af88a9e890c110fecafb9083188160e40a9657a/Serverless%20Cloud%20Run%20Development/Creating%20PDFs%20with%20Go%20and%20Cloud%20Run/Create%20PDFs%20with%20Go%20and%20Cloud%20Run%20Task%203%20Image.jpg)
1. Edit `Dockerfile` manifest to include LibreOffice package.  
  In Cloud Shell, click on `Open editor` and browse to Dockerfile. Or `nano Dockerfile`.  
  Update `Dockerfile` to below:
```
FROM DEBIAN:BUSTER
RUN apt-get update -y \
  && apt-get install -y libreoffice \
  && apt-get clean
WORKDIR /usr/src/app
COPT server .
CMD [ "./server"]
```  
`File` > `Save` for Cloud Shell Editor. `CTRL+X` then `Y` for nano editor.  

2. Build `pdf-converter` container image using Cloud Build:  
`gcloud builds submit --tag gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter`

3. Deploy image as Cloud Run Service named `pdf-converter`: 
```
gcloud run deploy pdf-converter \
  --image gcr.io/$GOOGLE_CLOUD_PROJECT/pdf-converter \
  --platform managed \
  --region us-east1 \
  --memory=2Gi \
  --no-allow-unauthenticated \
  --set-env-vars PDF_BUCKET=$GOOGLE_CLOUD_PROJECT-processed \
  --max-instances=3
```
<hr>

### Task 4: Create the Service Account
1. Integrate Cloud Storage with PubSub. Create notifications to PubSub topic "new-doc" on event (file uploaded to Google Cloud Storage bucket named "upload").  
`gsutil notification create -t new-doc -f json -e OBJECT_FINALIZE gs://$GOOGLE_CLOUD_PROJECT-upload`
    - [gsutil notification command](https://cloud.google.com/storage/docs/gsutil/commands/notification)  
    - `-t` flag specifies PubSub topic  
    - '-f' flag specifies payload format of notification message  
    - '-e' flag filters for event type  
    - `OBJECT_FINALIZE` is when new object is created (event type)  
    - `gs://xxxx` is the source GCS bucket      

Notice that 2 GCS buckets were pre-created.  
  - Cloud console > **Navigation menu** > **Cloud Storage**.   
    - PROJECT_ID-processed  
    - PROJECT_ID-upload  

2. Create a Service Account to access Cloud Run  
A [Service Account](https://cloud.google.com/iam/docs/service-account-overview) is a special type of account used by an application or a compute workload. It can be granted permission to access certain resources based on IAM policy/roles. If a Service Account is attached to a resource (e.g. Compute Engine) running an application, the application can authenticate as the service account and make authorized API calls.  
`gcloud iam service-accounts create pubsub-cloud-run-invoker --display-name "PubSub Cloud Run Invoker"`  
    - Service Account is named `pubsub-cloud-run-invoker`

3. Bind IAM policy to service account invoking the Cloud Run service
    ```
    gcloud run services add-iam-policy-binding pdf-converter \
        --member=serviceAccount:pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
        --roles=roles/run.invoker \
        --region us-east1 \
        --platform managed
    ```
      - name of Cloud Run service: `pdf-converter`
      - `--member=PRINCIPAL` 
          -  Format is `user|group|serviceAccount:email` or `domain:domain`. 
          -  Identifies the Service Account named `pubsub-cloud-run-invoker`.
      - for the role of `run.invoker`
      - [gcloud run services add-iam-policy-binding](https://cloud.google.com/sdk/gcloud/reference/run/services/add-iam-policy-binding)
  3. Enable project to create PubSub authentication tokens
      `PROJECT_NUMBER=$(gcloud projects list --format="value(PROJECT_NUMBER)" --filter="$GOOGLE_CLOUD_PROJECT")`  
      ```
      gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
        --member=serviceAccount:service-$PROJECT_NUMBER@gcp-sa-pubsub.iam.gserviceaccount.com \
        --role=roles/iam.serviceAccountTokenCreator
      ```  

### Task 5: Testing the Cloud Run Service
Test if URL of deployed Cloud Run Service is protected by authentication.  
1. Save URL into an environmental variable **$SERVICE_URL**:
  `SERVICE_URL=$(gcloud run services describe pdf-converter --platform managed --regions us-east1 --format "value(status.url)")
2. Verify if URL saved: `echo $SERVICE_URL`
3. GET without authentication:
   `curl -X GET $SSERVICE_URL`
   GET request will fail with reponse `Your client does not have permission to get URL`
4. Try GET again, with authentication:  
   `curl -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" $SERVICE_URL` 
   Success! You will get response "Ready to process POST requests from Cloud Storage trigger`

### Task 6: Subscribe to PubSub topic
Event (upload to GCS) > notification message from GCS to PubSub topic queue > message pushed to subscribed Cloud Run service
```
gcloud pubsub subscriptions create pdf-conv-sub \       <= pdf-conv-pub is name of subscription
  --topic new-doc \                                     <= new-doc is name of topic
  --push-endpoint=$SERVICE_URL \                        <= URL for Cloud Run service
  --push-auth-service-account=pubsub-cloud-run-invoker@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com <= [name of service account]@[email.com]
```
### Task 7: Testing Cloud Storage Notification
Upload some files to `gs://$GOOGLE_CLOUD_PROJECT-upload` and check if you get PDF files in `gs://$GOOGLE_CLOUD_PROJECT-processed` 
`gsutil -m cp -r gs://spls/gsp762/* gs://$GOOGLE_CLOUD_PROJECT-upload`  
[gsutil command](https://cloud.google.com/storage/docs/gsutil/commands/cp) syntax: gsutil cp [flags] [source_url] [destination_url]
  - `cp` for copy
  - `-r` to copy entire dir directory
  - `-m` for multi-thread parallel processing
In Cloud console, go to **Cloud Storage** > **Buckets**
  - in bucket with name ending in `-upload`, **Refresh** and see the files deleted as they are converted to PDFs
  - in bucket with name ending in `-processed`, **Refresh** and see PDF version of all files. They should open properly.
Check log of Cloud Run for file conversions.
  - **Navigation menu** > **Cloud Run** > **pdf-converter** > **LOGS** tab > filter for "Converting"
  - can tally with files in `-upload` and `-processed` buckets
