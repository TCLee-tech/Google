# Serverless Firebase Development: Challenge Lab

### Challenge scenario
In this lab you will create a frontend solution using a Rest API and Firestore database. Cloud Firestore is a NoSQL document database that is part of the Firebase platform where you can store, sync, and query data for your mobile and web apps at scale. Lab content is based on resolving a real world scenario through the use of Google Cloud serverless infrastructure.

You will build the following architecture:
![overall architectire](https://github.com/TCLee-tech/Google/blob/4aa0f37a518552df5ddad4589db4ca7e12a539e7/Serverless%20Firebase%20Development/Image%201.png)

#### Provision the environment
1. Link to the project:  
`gcloud config set project $(gcloud projects list --format='value(PROJECT_ID)' --filter='qwiklabs-gcp')`
2. Clone the repo:  
`git clone https://github.com/rosera/pet-theory.git`

### Task 1. Create a Firestore database
In this scenario, you create a Firestore Database in Google Cloud. The high level architecture diagram below summarizes the general architecture.
![Task 1 architecture](https://github.com/TCLee-tech/Google/blob/cc5e6c7f37e5feaee90a2e6ce742e7d61b2120c4/Serverless%20Firebase%20Development/Image%202.png)
Requirements:

|Field |	Value |
| ---  |     ---  |
|Cloud Firestore |	Native Mode |
|Location	| Nam5 (United States) |

To complete this section successfully, you are required to implement the following:

- Cloud Firestore Database
- Use Firestore Native Mode
- Add location Nam5 (United States)

### = Task 1 Solution =
[Getting Started with Cloud Firestore](https://firebase.google.com/docs/firestore/quickstart)  
1. In the Cloud Console, go to **Navigation menu** and select **Firestore**.
2.  Click the **Select Native Mode** button.
3. In the **Select a location** dropdown, choose **Nam5** and then click **Create Database**.  

OR  [Reference](https://cloud.google.com/sdk/gcloud/reference/firestore/databases/create)  
In Cloud Shell CLI, enter `gcloud firestore databases create --region=nam5`  

### Task 2. Populate the Database
In this scenario, populate the database using test data.

A high level architecture diagram below summarizes the general architecture.
![Task 2 Image 3](https://github.com/TCLee-tech/Google/blob/d348c801d996e8e2092acf9f41ec974850117bc7/Serverless%20Firebase%20Development/Task%202%20Image%203.png)

Example Firestore schema:

|Collection |	Document |	Field |
| ---       | ---        | ---    |
|data |	70234439 |	[dataset] |

The [Netflix Shows Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows) includes the following information:

|Field |	Description |
| ---  | ---            |
|show_id: |	Unique ID for every Movie / Tv Show |
|type: |	Identifier - A Movie or TV Show |
|title:	| Title of the Movie / Tv Show |
|director: |	Director of the Movie |
|cast:	| Actors involved in the movie / show |
|country:	| Country where the movie / show was produced |
|date_added: |	Date it was added on Netflix |
|release_year: |	Actual Release year of the move / show |
|rating: |	TV Rating of the movie / show |
|duration:	| Total Duration - in minutes or number of seasons |

To complete this section successfully, you are required to implement the following tasks:

1. Use the sample code from `pet-theory/lab06/firebase-import-csv/solution`:  
`npm install`  
2. To import CSV use the node `pet-theory/lab06/firebase-import-csv/solution/index.js`:  
`node index.js netflix_titles_original.csv`  

Note: Verify the Firestore Database has been updated by viewing the data in the Firestore UI.  
