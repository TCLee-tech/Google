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
  1. Receive lab report HTTPS POST request and response back.
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
