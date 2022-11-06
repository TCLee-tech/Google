## Continuous Delivery with Jenkins in Kubernetes Engine
### What is Kubernetes Engine?
- Google Cloud's hosted version of open-source *Kubernetes* - a cluster manager and orchestration platform for containers.
- Containers are lightweight applications bundled with all needed dependencies and libraries
  - benefits: highly available, secure, quick to deploy.
  - one virtual host can run jobs based on different operating systems.
- GKE provides `ephemeral build executors` - these are only used when builds are actively running, otherwise leaving resources for other cluster tasks. They are fast - launched within seconds.
- GKE is pre-equipped with Google's load balancer - , provide global IP address, access to Google's backbone network, can automate routing of web traffic to instance(s), handle SSL termination.

### What is [Jenkins](https://www.jenkins.io/)?
- Open-source automation server to orchestrate build, test and deployment pipelines.
- Allows agile iteration cycles since many processes in continuous delivery can be automated
- Extensible with hundreds of plugins along toolchain.
  - [Jenkins on Kubernetes](https://cloud.google.com/architecture/jenkins-on-kubernetes-engine)

### Learning Tasks
1. Provision a Jenkins application into a Kubernetes Engine Cluster
2. Set up your Jenkins application using Helm Package Manager
3. Explore features of a Jenkins application
4. Create and exercise a Jenkins pipeline

### Task 2. Provisioning Jenkins
#### Creating a Kubernetes cluster
1. [Create a cluster](https://cloud.google.com/sdk/gcloud/reference/container/clusters/create) for running containers
`gcloud container clusters create [name] [flags]`
2. Verify that cluster is running
`gcloud container clusters list`
3. Fetch cluster endpoint and authentication data to update kubeconfig file
`gcloud container clusters get-credentials [cluster name]`
4. Verify kubectl tool can connect
`kubectl cluster-info`

### Task 3: Setup Helm
- use Helm to [install Jenkins from the Charts repository](https://www.jenkins.io/doc/book/installing/kubernetes/#configure-helm)
- Helm is package manager that makes it easy to config and deploy Kubernetes.
- Its package format is called **chart**.
  - chart is a collection of files (mostly yaml) at known locations.
   - metadata (chart.yaml), default config (values.yaml), manifest templates (templates/deployment.yaml, templates/service.yaml etc)
`helm repo add jenkins https://charts.jenkins.io`
- Update repo `helm repo update`

### Task 4: Configure and install Jenkins
There are necessary plugins for Jenkins to connect to cluster and GCP. Use a [values.yaml](https://raw.githubusercontent.com/jenkinsci/helm-charts/main/charts/jenkins/values.yaml) template to automate configuration.
Plugins for 
 - [Kubernetes](https://plugins.jenkins.io/kubernetes/#plugin-content-configuration-on-google-container-engine)
   - creates a Kubernetes pod for each Jenkins agent started, and stops pod after each build
   - automates scaling of Jenkins agents
   - requests by Jenkins master pod
   - the Jenkins web UI and agent pods use Cluster IP to communicate, not accessible from outside cluster
   - [steps to run Jenkins and agents in cluster](https://plugins.jenkins.io/kubernetes/#plugin-content-configuration-on-google-container-engine)
 - workflow-multibranch
 - git
 - configuration-as-code
 - Google-oauth-plugin
 - Google-source-plugin
 - Google-storage-plugin

1. Use Helm to install
`helm install cd jenkins/jenkins -f jenkins/values.yaml --wait`
2. Verify that Jenkins pod status is `Running` and is in READY state.
`kubectl get pods`
3. Configure Jenkins service account to be able to deploy to cluster
`kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins`
4. Set up port forwarding from Cloud Shell to Jenkins UI
```
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```
5. Verify Jenkins Service created
`kubectl get svc`

### Task 5: Jenkins login
1. Jenkins chart will auto-generate **admin** username and password to login to Jenkins.
2. To retrieve password, `printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo`

### Task 7: Deploy application
Kubectl commands relevant to application deployment:
1. Change to application's directory 
`cd [path/application]`
2. Create Kubernetes namespace to logically isolate deployment
`kubectl create ns [namespace name]`
3. `kubectl apply` deployments and services to namespace
`kubectl apply -f [name of file] -n [namespace name]`
4. To manually scale up running replicas
`kubectl scale deployment [deployment name] -n [namespace name] --replicas ?`
5. To verify running pods
`kubectl get pods -n [namespace name] -l [label=xxx]`
6. To retrieve external IP for service
`kubectl get service [service name] -n [namespace name]`
7. To preview application
`curl http://external-IP:port`

### Task 8: Creating the Jenkins pipeline  
Create repo and push codes to repository.
1. Create repo in [Cloud Source Repository](https://cloud.google.com/source-repositories/docs/) to host source code
`gcloud source repo create [repo name]`
2. Initialize local directory for Git
`git init`
3. Configure [git credential helper](https://git-scm.com/book/en/v2/Git-Tools-Credential-Storage)
`git config credential.helper gcloud.sh`
4. Add remote repo
`git remote add origin https://source.developers.google.com/Project-ID/[repo name]`
5. Set username and email for Git commits.
`git config --global user.email [abc@yahoo.com]`
`git config --global user.name [username]`
6. Add, commit and push
`git add .`
`git commit -m "messsage"`  //-m flag for message
`git push origin master`    // push [remote to repo] [local from repo]
7. If need to create Git branch
`git checkout -b [branch name]`

Configure Jenkins to use cluster's service account credentials to access code repository.
Step 1: In the Jenkins user interface, click **Manage Jenkins** in the left navigation then click **Manage Credentials**.
Step 2: Click **Jenkins**
Step 3: Click **Global credentials (unrestricted)**.
Step 4: Click **Add Credentials** in the left navigation.
Step 5: Select **Google Service Account from metadata** from the **Kind** drop-down and click **OK**

Configure Jenkins to use Kubernetes Cloud
Step 1: In the Jenkins user interface, select **Manage Jenkins > Manage nodes and clouds**.
Step 2: Click **Configure Clouds** in the left navigation pane.
Step 3: Click **Add a new cloud** and select **Kubernetes**.
Step 4: Click **Kubernetes Cloud Details**.
Step 5: In the **Jenkins URL** field, enter the following value: http://[name of jenkins master pod]:[port]
Step 6: In the **Jenkins tunnel** field, enter the following value: [name of jenkins agent pod]:[port]
Step 7: Click Save.

Create multibranch pipeline in Jenkins
Step 1: Click **Dashboard > New Item** in the left panel.
Step 2: Name the project [sample-app], then choose the **Multibranch Pipeline** option and click **OK**.
Step 3: On the next page, in the **Branch Sources** section, click **Add Source** and select **git**.
Step 4: Paste the **HTTPS clone URL** of your sample-app repo in Cloud Source Repositories into the **Project Repository** field. https://source.developers.google.com/Project-ID/repo
Step 5: From the **Credentials** drop-down, select the name of the credentials you created when adding your service account previously.
Step 6: Under **Scan Multibranch Pipeline Triggers** section, check the **Periodically if not otherwise run** box and set the **Interval** value to 1 minute.
Step 7: Click **Save** leaving all other options with their defaults.

The Jenkinsfile defines the entire pipeline. It is written in [Grovvy syntax](https://www.jenkins.io/doc/book/pipeline/syntax/).