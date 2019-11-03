# In this demo project, there are several folder as below
 - demo-service: the microservice that simulate store user behavior data
    - skill : flask + uwsgi + ngingx + influxdb 
 - locust-load-testing: run for performance test
    - skill : locust + influx + grafana
 - testcase: implement test script to test demo-service incluse functional and integration
    - skill : pytest
 - docker-compose.yml: deploy demo-service as docker container
    - skill : docker + docker-compose
 - jenkins_pipeline.groovy: CI process for deploy and test
    - skill : jenkins pipeline + groovy

# Task 1 Scenario and Explanation (Build up jenkins pipeline to deploy and test for microservice)

### Demo-Service Spec
 - User Story 1: user can send behavior data to servie such as user love/like/angy/cry for specific article in specific time
 - User Story 2: user can query behavior data in service, for example: it can show how many articles that user likes it. 

### Prepared Test Script (Not eloborate all spec for test cases, only demo how to write test script)
 - Functional: 
    - store behavior test
    - query behavior test (input invalid data payload)
 - Integration:
    - user store its behavior and can be retrived
    - user store different behavior several times at different timing and it can be queried with all records based on specific status

### Task Scenario: 
 - Assume if code submitted/merged by developer this CI/CD flow will be automaticially trigger.
 - Assume there is Dockerfile in microservice which implement by developer or sdet
 - Assume test is test script for testing service quality implement by sdet
 - Build up auto CI/CD for release microservice include deploying and testing
    - It will auto publish docker image to somewhere if build and test pass in CI/CD flow

### CI Steps Explanation:
There are several steps in CI/CD progress for constructing pipeline.
 1. Git sync from code repository
 2. Build microservice based on Dockerfile
 3. Deploy service to QA/Local environment
 4. Run test parallelly (This project only has master node, not multiple nodes)
 5. Push image to private docker registry
 6. Clean up temporary docker image file to avoid issue of no space left on disk 
 7. Archive build/test reulst for debug if any 
 8. Send build result to somewhere 

# Task 2 Scenario and Explanation (Evaluation of the microservice RPS, RESTful API based service)

### What the purpose is for performance test (load testing)?
 1. Evaluate the service capacity that how many RPS can be taken (Maximum RPS for single microservice with resource limitataion)
 2. Evaluate the service stablity under high RPS condition for specific long periods (should be more than 3 hours)
    - In my experience, sometimes service will get memory leak issue due to code does not handle memory GC or DB connection well under high RPS condition.
 3. Provide the test result with maximum RPS can benefit teams that OPS and PM/PO make some  decision or strategy with strong evidence
    - Operation team will know how many containers should be launched at the sametime on production environment
    - Project Manager/Product owner wil know it can be fulfill the requirement (real online user traffic)

### For this task, why I choose Apache Benchmark (ab) and Python Locust to do the performance test?
 1. Apache Benchmark Testing (https://httpd.apache.org/docs/2.4/programs/ab.html)
    - It is a tool for benchmarking your RESTful service
    - It can quickly attack the api with High RPS and check service status
 - Simulate thousands of concurrent users on a single machine. (cost concern)
 - 
