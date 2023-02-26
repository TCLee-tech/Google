# Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub

Part of the [Google Cloud Serverless Workshop: Pet Theory Quest](https://www.cloudskillsboost.google/quests/152)

### Task 1. Architecture
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

