# Cloud SQL with Cloud Run [APPRUN]

### Overview
Cloud SQL automatically ensures that your databases are reliable, secure and scalable, so that your business continues to run without disruption. Cloud SQL automates all your backups, replication, encryption patches, and capacity increases - while ensuring greater than 99.95% availability, anywhere in the world.

Lab content is based on resolving a customer use case through the use of serverless architecture. The three high level sections to resolve a technical problem:
- Situational Overview
- Requirements Gathering
- Developing a minimal viable product

### Prerequisites
Must be familiar with
- Cloud Build
- Cloud Tasks
- Cloud Run
- Cloud SQL

### Objectives
To learn:
- configure the environment and enable Cloud Run API
- create a Cloud SQL instance
- populate a Cloud SQL instance
- use environment variables for Cloud Run service
- deploy a public Cloud Run service

<hr>

### Situational overview
In this lab, you will help the development team at Critter Junction try Cloud SQL. The company runs its infrastructure on Google Cloud and is very interested in experimenting with severless. The dev team will like to explore how to use Cloud SQL and Cloud Run.

### Requirements gathering
The team at Critter Junction has a web application that requires a data store. Cloud SQL appears to be a good solution, but the team does not have any experience with this product.

The team would also like a solution that does not introduce additional complexity into their systems. They used PostgreSQL so would like to use that if it is an option. Now that you know a bit more about Critter Junction and the issues they face, try and **prioritise the key criteria for a solution**.

### Defining Critter Junction priorities
The team at Critter Junction is keen to define a solution that can be implemented quickly. 

A series of meetings with stakeholders is held to ascertain the key priorities. The results of which are shown below:

| Ref | User Story |
| --- | --- |
| 1 | As a development lead, I want to focus on frontend development, so my team can maintain velocity. |
| 2 | As a data lead, I want to focus on existing skill sets, so that developers minimize learning new products. |
| 3 | As an ops lead, I want to ensure service accounts are used, so product authentication is handled internally. |

From a discussion with the team leads, the following high level tasks are defined:

| Ref | Definition of Done |
| --- | --- |
| 1 | Deply the website using Cloud Run |
| 2 | Use Cloud SQL with Postgres |
| 3 | Use a Service Account with minimal IAM permissions to call Cloud SQL |

The following high level architecture diagram summarises the minimal viable product they wish to investigate. In the proposed solution, Cloud SQL will be used to handle the data tier.

![Cloud SQL with Cloud Run](https://github.com/TCLee-tech/Google/blob/6b4908a63b4b75ab43d1fe56285b42f3025b506c/Application%20Development%20with%20Cloud%20Run/Cloud%20SQL%20with%20Cloud%20Run%20lab.jpeg)

### Developing a Minimal Viable Product (MVP)
Integrate a website hosted by Cloud Run with Cloud SQL. To build a MVP, the following activities are required:
- configure the environment
- use environment variables for Cloud Run service
- deploy a public Cloud Run service
- create a Cloud SQL instance

<hr>

### Task 1: Configure the environment
Set up some environment variables to make provisioning process easier.  
1. Enable Cloud Run API.  
`gcloud services enable run.googleapis.com`
2. Set the compute region.  
`gcloud config set compute/region us-central1`
3. Create a LOCATION environment variable.  
`LOCATION="us-central1"`

<hr>

### Cloud SQL overview
Google Cloud SQL is a fully managed relational database service for **MySQL, PostgreSQL and SQL Server**.
Cloud SQL key features:
1. **Fully managed**  
Cloud SQL automatically ensures that your databases are reliable, secure and scalable. Your business can run without disruptions. Cloud SQL automates all backups, replication, encryption patches and capacity increases - while ensuring greater than 99.95% availability, anywhere in the world.
2. **Integrated**  
Access Cloud SQL instances from just about any application. Easily connect from App Engine, Compute Engine, Google Kubernetes Engine, and your workstation. Open up analytics possibilities by using [BigQuery to directly query your Cloud SQL databases](https://cloud.google.com/bigquery/docs/cloud-sql-federated-queries).
3. **Reliable**  
Easily configure replication and backups to protect your data. Enable automatic failover to make your database highly available. Your data is automatically encrypted, and Cloud SQL is SSAE 16, ISO 27001, and PCI DSS compliant and supports HIPAA compliance.
4. **Easy migration to Cloud SQL**  
Database Migration Service (DMS) makes it easy to migrate your production databases to Cloud SQL with minimal downtime. This severless offering eliminates the hassle of manual provisioning, managing and monitoring migration-specific resources. DMS leverages the native replication capabilities of MySQL and PostgreSQL to maximise the fidelity and reliability of your migration. It is available at no additional charge for native like-to-like migrations to Cloud SQL.

### Task 2: Create a Cloud SQL instance
Clodu SQL has a number of configuration options. Learn how to configure a PostgreSQL database in the following section.
1. Create a PostgreSQL instance with the following values:  

| Field | Value |
| --- | --- |
| Instance ID | poll-database |
| Password | secretpassword |
| Database version | PostgreSQL 13 |
| Region | us-central1 |
| Zone | Single Zone |

2. Click **Create Instance** > **Choose PostgreSQL** 

![Create PostgreSQL in Cloud SQL](https://github.com/TCLee-tech/Google/blob/9e96bce2d4fe9f9fb89985d738a035096a375588/Application%20Development%20with%20Cloud%20Run/Cloud%20SQL%20with%20Cloud%20Run%20lab%202.jpg)  

Note: Cloud SQL uses a connection name which is composed of the project ID + region + database name. A service account is also created to be used to access the database created.

3. Note the instance connection settings:
- Public IP address
- Outgoing IP address
- Connection name
- Service Account

<hr>

### Task 3: Populate the Cloud SQL instance
Populate the Cloud SQL database with table and data. 
1. Connect to the Cloud SQL instance (allowlist will be created for Cloud Shell IP).  
`gcloud sql connect poll-database --user=postgres` //instance ID: poll-database
2. Enter the Cloud SQL password when requested (i.e. "secretpassword")  
3. Connect to the database.  
`\connect postgres;`
4. Create the `votes` table:  
```
CREATE TABLE IF NOT EXISTS votes
( vote_id SERIAL NOT NULL, time_cast timestamp NOT NULL, candidate VARCHAR(6) NOT NULL, PRIMARY KEY (vote_id));
```
5. Create the `totals` table:  
```
CREATE TABLE IF NOT EXISTS totals
( total_id SERIAL NOT NULL, candidate VARCHAR(6) NOT NULL, num_votes INT DEFAULT 0, PRIMARY KEY (total_id));
```
6. Initialise the data for "TABS":  
`INSERT INTO totals (candidate, num_votes) VALUES ('TABS', 0);`
7. Initialise the data for "SPACES":  
`INSERT INTO totals (candidate, num_votes) VALUES ('SPACES', 0);`

<hr>

### Task 4: Deploy a public service
To provision the poll-service, the application expects some environment variables to be set.
| Environment Value | Description |
| --- | --- |
| DB_USER | Name of the database user |
| DB_PASS | Password for the database user |
| DB_NAME | Name of the database |
| CLOUD_SQL_CONNECTION_NAME | Name given to the Cloud SQL instance |

1. Set Cloud SQL Connection Name as an environment variable.  
`CLOUD_SQL_CONNECTION_NAME=$(gcloud sql instances describe poll-database --format='value(connectionName)')`

2. Deploy poll service on Cloud Run.  
```
gcloud run deploy poll-service \
--image gcr.io/qwiklabs-resources/gsp737-tabspaces \
--region $LOCATION \
--allow-unauthenticated \
--add-cloudsql-instances=$CLOUD_SQL_CONNECTION_NAME \
--set-env-vars "DB_USER=postgres" \
--set-env-vars "DB_PASS=secretpassword" \
--set-env-vars "DB_NAME=postgres" \
--set-env-vars "CLOUD_SQL_CONNECTION_NAME=$CLOUD_SQL_CONNECTION_NAME"
```
Note: Cloud Run uses key value pairs to define environment variables. [More info on Environment Variables in Cloud Run](https://cloud.google.com/run/docs/configuring/environment-variables#command-line)   
Any configuration change leads to the creation of a new revision.  

3. Cloud Run service endpoint can be accessed as per below:  
`POLL_APP_URL=$(gcloud run services describe poll-service --platform managed --region us-central1 --format="value(status.address.url)")`

<hr>

### Task 5: Testing the application
To test the application, browse to the Cloud Run endpoint and click to enter some data  

[Severless Expeditions video series](https://www.youtube.com/playlist?list=PLIivdWyY5sqJwq_pgOxcHzusWjXDVCEiX)




