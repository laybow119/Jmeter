## Folder and File Structure Explanation
 - demo-service: the microservice that simulate store user behavior data
    - skill : flask + uwsgi + ngingx + influxdb + RESTful API design
 - locust-load-testing: run for performance test
    - skill : ab + locust + influx + grafana + telegraf
 - testcase: implement test script to test demo-service incluse functional and integration
    - skill : pytest
 - docker-compose.yml: deploy demo-service as docker container
    - skill : docker + docker-compose
 - jenkins_pipeline.groovy: CI process for deploy and test
    - skill : jenkins pipeline + groovy

## Task 1 Brief Description
Build up jenkins pipeline to deploy and test for microservice

### Demo-Service Spec
 - User Story 1: user can send behavior data to servie such as user love/like/angy/cry for specific article in specific time
 - User Story 2: user can query behavior data in service, for example: it can show how many articles that user likes it. 

### Prepared Test Script
Here does not eloborate all spec for test cases, only demo how to write test script
 - Functional: 
    - store behavior test
    - query behavior test (input invalid data payload)
 - Integration:
    - user store its behavior and can be retrived
    - user store different behavior several times at different timing and it can be queried with all records based on specific status

### Task Scenario
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

## Task 2 Brief Description
Evaluation for the microservice capacity which is RESTful API based service

### What the purpose is for performance test (load testing)?
 1. Evaluate the service capacity that how many RPS can be taken (Maximum RPS for single microservice with resource limitataion)
 2. Evaluate the service stablity under high RPS condition for specific long periods (should be more than 3 hours)
    - In my experience, sometimes service will get memory leak issue due to code does not handle memory GC or DB connection well under high RPS condition.
 3. Provide the test result with maximum RPS can benefit teams that OPS and PM/PO make some  decision or strategy with strong evidence
    - Operation team will know how many containers should be launched at the sametime on production environment
    - Project Manager/Product owner wil know it can be fulfill the requirement (real online user traffic)

### Load Testing Tool
Why choose Apache Benchmark (ab) and Python Locust to do the performance test?
 1. Apache Benchmark Testing (https://httpd.apache.org/docs/2.4/programs/ab.html)
    - It is a tool for benchmarking your RESTful service
    - It can quickly hit the api with High RPS and check service status
        - Only for quickly test single api with high RPS
 2. Python Locust (https://locust.io/)
    - It can simulate thousands of concurrent users on a single machine. (cost concern)
    - It write very expressive scenarios as user behavior to access your service
    - Support distributed mode which is master and slave architect. It can easy setup load generator with high RPS without extra effort
    - Support multi protocol load testing such as RESTful and gRPC (https://grpc.io/) protocol

### Task Scenario
 - Assume target RPS is 500 and need to complete in 10 mins
    - In demo-service, assume the api ratio is 1 for store user behavoir but 2 is for query non-exist data (just for demo)
 - Load testing for check service capacity and stablity
 - System status monitor such as CPU / Memory / Disk

### Test Script
Check "locustfile.py" file in locust-load-testing folder and path is /perf_script/mendix-demo
 - Apply locust framework
    - Impelement classe for HttpLocust, TaskSet, task, Locust
 - Simply to write the test scenario as user behavior 
    - Store user behavior data
    - Query user behavior data

### Setup performance environment 
 - Locust distributed mode
    - Deploy one master and ten slave node to simulate total count request count is 500 
        - Check deploy.sh and README.md in locust-load-testing folder for more detail 
    - Open locust web console to check Performance test reult such as RPS / response time ..etc
 - System status
    - Install telegraf + influxdb then it will auto output system information into fluxdb and can be present by grafana dashboard

In my experience, web UI console is timeseries based which means you will missing some data if run load testing for serveral hours due to it does not make sense for keeping browser for long period.
 - I did the survey and come out the solution was that create another worker to flush statistic result into timeseries db (influxdb) and use grafana dashboard to query and show it.

 In this task, I did not really use distributed mode as testing frmework since the RPS is quite low. I only use local for test but I will give some screenshot as evidence for above desicription
 - The screenshot has been taken in my recent project that load testing againist RESTful and gRPC micro service and also wrote the wiki for guide other SDET member how to use it

 BTW, if you are very interested in how to setup for this part I can show you in the interview if time is possible.

## Screenshot of test result for demo project
 - You can reference folder "result_screenshot_example" for CI test result
    - build_success.png: it output with pipeline script with success tesult and of course that I have to comment out some code snippets only left stage description for concept and flow description. 
    - build_fail_with_log.png: it shows if pipeline build fail and you can check the log from jenkins console and as well as can be received some message from pipeline script

## Result for load testing (Using local to do peformance test)
 AB test
 - Max RPS is 178 (only for query api)
 - Hit total 500 request and finish in 2.8 sec, max response time would be 160 ms)
 ```bash
10:04:00 laybow_kuo ~ $ ab -n 500 -c 20 http://127.0.0.1:8000/mendix-demo/v1/test123/query
This is ApacheBench, Version 2.3 <$Revision: 1826891 $>
Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
Licensed to The Apache Software Foundation, http://www.apache.org/

Benchmarking 127.0.0.1 (be patient)
Completed 100 requests
Completed 200 requests
Completed 300 requests
Completed 400 requests
Completed 500 requests
Finished 500 requests

Server Software:        Werkzeug/0.16.0
Server Hostname:        127.0.0.1
Server Port:            8000

Document Path:          /mendix-demo/v1/test123/query
Document Length:        122 bytes

Concurrency Level:      20
Time taken for tests:   2.804 seconds
Complete requests:      500
Failed requests:        0
Total transferred:      134500 bytes
HTML transferred:       61000 bytes
Requests per second:    178.34 [#/sec] (mean)
Time per request:       112.148 [ms] (mean)
Time per request:       5.607 [ms] (mean, across all concurrent requests)
Transfer rate:          46.85 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.1      0       1
Processing:     8  108  14.2    109     160
Waiting:        8  107  14.0    107     154
Total:          9  108  14.1    109     160

Percentage of the requests served within a certain time (ms)
  50%    109
  66%    113
  75%    116
  80%    118
  90%    122
  95%    126
  98%    130
  99%    133
 100%    160 (longest request)
 ```

Locust Test 
- Max RPS is 97 (For both store and query api)
    - Store user behavior data
    - Query user behavior data
- Simulate 100 users and hatching rate in 0.1, total request is 5245 in 1 mins
```bash
10:52:47 laybow_kuo ~/Documents/laybow_mendix/locust-load-testing/perf_script/mendix-demo $ locust -f locustfile.py -c 100 -r 0.1 -t 1m --no-web
[2019-11-03 10:52:48,978] T-01002306ML-2.local/INFO/locust.main: Run time limit set to 60 seconds
[2019-11-03 10:52:48,978] T-01002306ML-2.local/INFO/locust.main: Starting Locust 0.11.0
[2019-11-03 10:52:48,978] T-01002306ML-2.local/INFO/locust.runners: Hatching and swarming 100 clients at the rate 0.1 clients/s...

....

[2019-11-03 10:53:48,833] T-01002306ML-2.local/INFO/locust.main: Time limit reached. Stopping Locust.
[2019-11-03 10:53:48,834] T-01002306ML-2.local/INFO/locust.main: Shutting down (exit code 0), bye.
[2019-11-03 10:53:48,834] T-01002306ML-2.local/INFO/locust.main: Cleaning up runner...
[2019-11-03 10:53:48,839] T-01002306ML-2.local/INFO/locust.main: Running teardowns...
 Name                                                          # reqs      # fails     Avg     Min     Max  |  Median   req/s
--------------------------------------------------------------------------------------------------------------------------------------------
 GET /mendix-demo/v1/a5166aab-7764-433a-8e83-2035b065a45b/query    3426     0(0.00%)      44      14     159  |      41   61.50
 POST /mendix-demo/v1/user/status                                1819     0(0.00%)      29       8     121  |      26   35.50
--------------------------------------------------------------------------------------------------------------------------------------------
 Total                                                           5245     0(0.00%)                                      97.00

Percentage of the requests completed within given times
 Name                                                           # reqs    50%    66%    75%    80%    90%    95%    98%    99%   100%
--------------------------------------------------------------------------------------------------------------------------------------------
 GET /mendix-demo/v1/a5166aab-7764-433a-8e83-2035b065a45b/query     3426     41     50     57     61     73     86     98    110    160
 POST /mendix-demo/v1/user/status                                 1819     26     33     38     42     52     60     72     78    120
--------------------------------------------------------------------------------------------------------------------------------------------
 Total                                                            5245     35     45     52     56     67     78     92    100    160
```
