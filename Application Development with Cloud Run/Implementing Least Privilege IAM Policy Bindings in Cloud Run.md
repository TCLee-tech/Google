# Implementing Least Privilege IAM Policy Bindings in Cloud Run [APPRUN]

Good guide: [Serverless Expeditions video series](https://www.youtube.com/playlist?list=PLIivdWyY5sqJwq_pgOxcHzusWjXDVCEiX)

**Situation**  
Quickway Parking has a Cloud Run billing service that they would like to be made more secure. The component is called over the internet using an assigned URL, but will need eventually to be able to be called from a scheduling service.

**To learn:**
1. Configure your environment, including enabling the Cloud Run API
2. Create and deploy a public un-authenticated Cloud Run service
3. Re-deploy Cloud Run service to authenticate service requests.
4. Invoke a non-public Cloud Run service from another service account.

### Task 1. Configure the environment
1. Enable Cloud Run API:  
`gcloud services enable run.googleapis.com`
2. Set Cloud Run region:  
`gcloud config set run/region us-central1`
3. Create LOCATION environment variable:  
`LOCATION="us-central1"`

<hr>

### Task 2. Create and deploy a public Cloud Run service
Deploy Cloud Run billing service.
The Quickway DevOps team already has a billing image available on Google Cloud.

1. Open Cloud Shell and deploy this Cloud Artifact Registry image:
```
gcloud run deploy quickway-parking-billing-v1 \
    --image gcr.io/qwiklabs-resources/gsp723-parking-service \
    --region $LOCATION \
    --allow-unauthenticated
```

2. Assign the URL for the new service to an environment variable:
`QUICKWAY_SERVICE=$(gcloud run services list --format='value(URL)' --filter="quickway")`

3. Cloud Console > **Cloud Run** > select name of service just deployed.  
Take a closer look at the authentication applied and the role permissions assigned. At the moment, both the Cloud Scheduler and anyone on the internet can access this resource. This resource is public.  
Google Cloud promotes the grant of permissions with the least privilege needed.  
When the billing service was originally deployed, it used the `--allow-unauthenticated` permission, which makes the service publicly available.  

![Implementing Least Privilege IAM Policy Bindings in Cloud Run](https://github.com/TCLee-tech/Google/blob/8080e65e3bcd54296ec1b5520bc6b10f933bbc69/Application%20Development%20with%20Cloud%20Run/Implementing%20Least%20Privilege%20IAM%20Policy%20Bindings%20in%20Cloud%20Run%20Task%202%20pic1.jpeg)

| Type | Permission | Description |
| ---  |   ---      | ---         |
| URL access| `--allow-unauthenticated` | Make the service **public** (i.e. unauthenticated users can access it) |
| Invoke permission | allUsers | Allow the service to be invoked by anyone. |

By removing the `--allow-unauthenticated` permission, you can use option flag to secure the service.
| Type | Permission | Description |
| ---  |   ---      | ---         |
| URL access| `--no-allow-unauthenticated` | make the service **private** (i.e. only autneticated users can access it) |
| Invoke permission | none | Nobody can invoke the service. |

Updated application design:
![Implementing Least Privilege IAM Policy Bindings in Cloud Run Task 2 Image 2](https://github.com/TCLee-tech/Google/blob/bd02654e555275e6c7abf75726ea8f1ceb2a1e49/Application%20Development%20with%20Cloud%20Run/Implementing%20Least%20Privilege%20IAM%20Policy%20Bindings%20in%20Cloud%20Run%20Task%202%20pic2.jpeg)

Main changes are:
1. Remove unauthenticated access and invoke permission associated with Billing Service.
2. Create a new Service Account with the permissions to invoke the private Billing Service.

<hr>

### Task 3. Authenticating service requests
1. Delete the existing deployed Billing Service:  
`gcloud run services delete quickway-parking-billing-v1`  
2. Redeploy Billing Service with `--no-allow-unauthenticated`  
```
gcloud run deploy quickway-parking-billing-v2 \
    --image gcr.io/qwiklabs-resources/gsp723-parking-service 
    --region $LOCATION 
    --no-allow-unauthenticated
```
Redeploying the billing service means it no longer allows unauthenticated access via its URL. In addition, access permission to invoke the service via the URL has been removed.

### Defining a Service Account
What if you need to invoke a non-public Cloud Run service from another service account?
In that case, you will need to add permissions to invoke Billing Service application from the new service, using **IAM policy bindings**.
This can be done via the Console, or by using the command line interface. We will use the Console in this lab to set up the new policy binding for the Billing service.

1. In the Cloud Console, select **IAM & Admin** > **Service Accounts**.
2. Create a new Service Account that will provide authenticated access by clicking on **Create Service Account** near the top of the menu.
3. Name the new Service Account `Billing Initiator`.
![]()
4. Click the **Create and Continue** button to create the account and advance to the **Grant Access** step.
5. Select the Role dropdown, scroll the left side to *Cloud Run*, and select the role *Cloud Run Invoker* to give the Billing Initiator permissions to invoke the Billing Service.
6. Select **Continue** and then **Done** to complete the setup of the Service Account. You will see the new service account at the top of your list of service accounts in the Console window.
![]()
The service account "Billing Initiator" has been created. It has authorization to invoke `quickway-parking-billing-v2` Cloud Run instance, using an IAM policy binding.
