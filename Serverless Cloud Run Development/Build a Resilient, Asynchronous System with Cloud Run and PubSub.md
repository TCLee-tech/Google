#Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub

Part of the [Google Cloud Serverless Workshop: Pet Theory Quest](https://www.cloudskillsboost.google/quests/152)

### Task 1. Architecture
- Pet Theory is a veterinary clinic chain
- They use an external company for lab tests
- When the test results are ready, they send a HTTP(S) POST to Pet Theory's web endpoint for such results.
- Pet Theory will 
  1. receive the HTTP POST request
  2. email test result to client
  3. SMS test result to client
- Want to do serverless migration. Isolated components of serverless architecture:
  1. Service handle HTTP request and response for test result(s).
  2. Service to send email
  3. Service to send SMS
  4. PubSub for inter-service communication
  Each service is a single function

![Build a Resilient, Asynchronous System with Cloud Run and Pub/Sub]()

