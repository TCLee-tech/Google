# Deploying Jobs on Google Kubernetes Engine

To learn:
1. How to define, deploy and clean up a GKE Job
2. How to define, deploy and clean up a GKE CronJob

### Task 1. Define and deploy a Job manifest
A Job is a controller object that represents a finite task.
  - manages a task until it runs to completion
  - does not manage a desired state, e.g. a certain number of running pods

#### Connect to the lab Google Kubernetes Engine cluster
1. Set environment variables  
`export my_zone=us-central1-a`  
`export my_cluster=standard-cluster-1`
2. Configure kubectl tab completion  
`source <(kubectl completion bash)`
3. Configure access to GKE cluster  
`gcloud container clusters get-credentials $my_cluster --zone $my_zone`
4. Clone the lab repository to Cloud Shell  
`git clone https://github.com/GoogleCloudPlatform/training-data-analyst`
5. Create soft link to working directory  
`ln -s ~/training-data-analyst/courses/ak8s/v1.1 ~/ak8s`
6. Change to directory containing (cron)jobs manifest yaml files  
`cd ~/ak8s/Jobs_CronJobs`

#### Create and run a Job
1. Sample Job manifest: `example-job.yaml`
    - computes Pi to 2,000 places.
```
apiVersion: batch/v1
kind: Job
metadata:
  # Unique key of the Job instance
  name: example-job
spec:
  template:
    metadata:
      name: example-job
    spec:
      containers:
      - name: pi
        image: perl:5.34
        command: ["perl"]
        args: ["-Mbignum=bpi", "-wle", "print bpi(2000)"]
      # Do not restart containers after they exit
      restartPolicy: Never
```
2. Create Job from manifest yaml file.  
`kubectl apply -f example-job.yaml`
3. To verify, view jobs.  
`kubectl get jobs`  
    - get names of jobs, #/total completions.
4. To get status of a job,  
`kubectl describe job [job name]`
    - status includes # Running, # Succeeded, # Failed
5. To view completed pods created by Job,  
`kubectl get pods`  
    - should see 0/total READY, Completed STATUS

#### Clean up and delete the Job
- when Job completes > it stops creating pods
  - Job API object remains, not deleted > can check status
  - pods complete and terminated, but not deleted > can retrieve logs and interact with them
1. To retrieve log file from the pod that ran the Job  
`kubectl log [pod name]`
2. Deleting a Job will clean up all pods it created. To delete a Job,  
`kubectl delete job [name of job]`
    - if try to query logs after deletion, fail as no pods found

### Task 2. Define and deploy a CronJob manifest
A CronJob performs a finite task a specific number of times e.g. run once, or repeatedly at intervals.

#### Create and run a CronJob
1. Sample CronJob manifest file `example-cronjob.yaml`
    - deploys new container every minute that prints the time, date and "Hello World".
```
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox
            args:
            - /bin/sh
            - -c
            - date; echo "Hello, World!"
          restartPolicy: OnFailure
```

  - `.spec.schedule` is a required field
    - accepts time in Unix standard `crontab` format
    - all times in UTC
    - minute (0-59) - hour (0-23) - day of month (1-31) - month (1-12) - day of week (0-6)
    - `*` and `?` accepted as wildcard values
    - combine `/` with ranges to specify task should be repeated at regular interval

2. Create CronJob,  
`kubectl apply -f example-cronjob.yaml`
3. To verify, get list of jobs in cluster   
`kubectl get jobs`
    - get names of jobs, #/total completions
    - by default, only last 3 successful and last failed jobs retained
4. To check status of a job,  
`kubectl describe job [job name]`
    - get # Running, # Succeeded, # Failed
5. To view output of Job by querying logs of Pod,  
`kubectl log [pod name]` 

#### Clean up and delete CronJob
1. Delete CronJob  
`kubectl delete cronjob [job name]`
2. To verify, get jobs  
`kubectl get jobs`
    - output should be "No resources found in default namespace".
