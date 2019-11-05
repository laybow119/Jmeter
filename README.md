## Folder and File Structure Explanation
 - demo-service: the microservice that simulate store realtime of user behavior data
    - skill : flask + uwsgi + ngingx + influxdb + RESTful API design
 - locust-load-testing: run for performance test
    - skill : locust + influx + grafana + telegraf
 - testcase: implement test script to test demo-service include of functional and integration
    - skill : pytest
 - docker-compose.yml: deploy demo-service as docker container
    - skill : docker + docker-compose
 - jenkins_pipeline.groovy: CI process for deploy and test
    - skill : jenkins pipeline concept + groovy

## Task 1 Brief Description
Build up jenkins pipeline to deploy and test for microservice

### Demo-Service Spec
 - User Story 1: user can send behavior data to servie such as user love/like/angy/cry for specific article in specific time
 - User Story 2: user can query behavior data in service, for example: it can show how many articles that user likes it. 

### Prepared Test Script
Here does not eloborate all aspect of test cases, only demo how to write test script
 - Functional: 
    - store behavior test
    - query behavior test (input invalid data payload)
 - Integration:
    - user store its realtime behavior and can be retrieved
    - user store different behavior several times at different timing and it can be queried with all records based on specific status

### Task Scenario
 - Assume if code submitted/merged by developer this CI/CD flow will be automaticially trigger
 - Assume there is "Dockerfile" for build up microservices which implement by developer or sdet
 - Assume there is test script for testing service quality implement by sdet
 - Build up auto CI/CD for release microservice include of deploying and testing
    - it will auto publish docker image to somewhere if build and test pass in CI/CD flow

### CI Steps Explanation:
There are several steps in CI/CD progress for constructing pipeline.
 1. Git sync from code repository
 2. Build up microservice based on Dockerfile
 3. Deploy service to QA/Local environment
 4. Run test parallelly (This project only has master node, not multiple nodes)
 5. Push image to private docker registry
 6. Clean up temporary docker image file to avoid issue of no space left on disk 
 7. Archive build/test reulst for debug if any 
 8. Send build result whcih to somewhere 

## Task 2 Brief Description
Evaluation for the microservice capacity which is RESTful API based service

### What the purpose stands for performance test (load testing)?
 1. Evaluate the service capacity that how many Request Per Seconnd (RPS) can be taken (Maximum RPS for single container under resource limitataion when deployed)
 2. Evaluate the service stablity under high RPS condition (stress test) for specific long periods (should be more than 3 hours)
    - in my experience, sometimes service will get memory leak issue due to code does not handle memory GC or DB connection well under high RPS condition.
 3. Provide the test result with maximum RPS can benefit teams that OPS and PM/PO can make some decision or strategy with strong evidence
    - operation team will know how many containers should be launched at the sametime on production environment against the target RPS
    - project manager or product owner will know it can be fulfill the requirement or not(real online user traffic)

### Load Testing Tool

Why select both Apache Benchmark (ab) and Python Locust to do the performance test?
 1. Apache Benchmark Testing (https://httpd.apache.org/docs/2.4/programs/ab.html)
    - it is a tool for benchmarking your RESTful service
    - it can quickly hit the api with High RPS and check service status
        - only for quickly test single api with high RPS
    - opensource
 2. Python Locust (https://locust.io/)
    - it can simulate thousands of concurrent users on a single machine. (cost concern)
    - it write very expressive scenarios as user behavior to access your service
    - support distributed mode which is master and slave architect. It can easy setup load generator with high RPS without extra effort
    - support multi protocol load testing such as RESTful and gRPC (https://grpc.io/) protocol
    - opensource

### Task Scenario
 - Assume target RPS is 500 and need to complete in 10 mins
    - in demo-service, assume the api ratio that sotre is 50 and query is 1(just for demo, simulate store action is more than query)
 - Load testing for check service capacity

### Test Script
Check "locustfile.py" file in locust-load-testing folder and path is /perf_script/mendix-demo
 - Apply locust framework
    - impelement and inheritance of serveral class ex: HttpLocust, TaskSet, task, Locust
 - Simply to write the test scenario as user behavior includes of store and query action

### Setup performance environment 
 - Locust distributed mode
    - deploy one master and ten slave node to simulate total count request count is 500 
        - check deploy.sh and README.md in locust-load-testing folder for more detail 
    - open locust web console to check Performance test reult such as RPS / response time ..etc
    - I Dockerize python locust that only focus on write the test script and implement deploy script (can integrate to CI/CD flow that automated setup performace environment)

```bash
$ sh ./deploy.sh -a redeploy -f ./perf_script/mendix-demo/locustfile.py --web_port 8888 --master_ip 10.222.131.36 --slave_number 5

Creating network "locust-load-testing_default" with the default driver
Creating locust_master                      ... done
Creating locust-load-testing_locust-slave_1 ... done
Creating locust-load-testing_locust-slave_2 ... done
Creating locust-load-testing_locust-slave_3 ... done
Creating locust-load-testing_locust-slave_4 ... done
Creating locust-load-testing_locust-slave_5 ... done

```
 - System status
    - install telegraf + influxdb then it will auto output system information into fluxdb and can be present by grafana dashboard

In my experience, web UI console is timeseries based which means you will missing some data if run load testing for serveral hours due to it does not make sense for keeping browser for long period.
 - I did the survey and come out the solution was that create another worker to flush statistic result into timeseries db (influxdb) and use grafana dashboard to query and show it.

 In this task, I only run test on QA machine (one node only) and --no_web mode but I can give some screenshot as evidence for above desicription, include web ui, system information and statistic result show by gragana
 - The screenshot has been taken in my recent project that load testing against RESTful and gRPC micro service and also wrote the wiki for guide other SDET member how to use it

## Screenshot of test result for demo project
 - You can reference folder "result-screenshot-example" for CI and Peformance test result
    - build_success.png: it output with pipeline script with success tesult and of course that I have to comment out some code snippets only left stage description for concept and flow description. 
    - build_fail_with_log.png: it shows if pipeline build fail and you can check the log from jenkins console and as well as can be received some message from pipeline script


## Result for load testing (Using local to do peformance test)
RPS could be got more high with deploy it on AWS instance instead of on local machine.
 AB test
 - The Max RPS of store and query api are 155 and 40, respectively.
 - AB test only can test one api at the same time (you still can launch two process to hit both api)
 ```bash
------------------------------ Store API ------------------------------
15:34:03 laybow_kuo ~/Downloads/event-publisher $ ab -n 500 -c 20 -T 'application/json' -p test.json http://127.0.0.1:8000/mendix-demo/v1/user/status

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

Document Path:          /mendix-demo/v1/user/status
Document Length:        86 bytes

Concurrency Level:      20
Time taken for tests:   3.225 seconds
Complete requests:      500
Failed requests:        0
Total transferred:      116000 bytes
Total body sent:        112500
HTML transferred:       43000 bytes
Requests per second:    155.04 [#/sec] (mean)
Time per request:       129.000 [ms] (mean)
Time per request:       6.450 [ms] (mean, across all concurrent requests)
Transfer rate:          35.13 [Kbytes/sec] received
                        34.07 kb/s sent
                        69.19 kb/s total

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.6      0       5
Processing:    20  126  19.5    125     174
Waiting:       14  124  18.8    123     172
Total:         20  126  19.2    125     174
 
------------------------------ Query API ------------------------------
15:35:42 laybow_kuo ~/Downloads/event-publisher $ ab -n 500 -c 20 http://127.0.0.1:8000/mendix-demo/v1/test123/query

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
Time taken for tests:   12.615 seconds
Complete requests:      500
Failed requests:        0
Total transferred:      134500 bytes
HTML transferred:       61000 bytes
Requests per second:    39.63 [#/sec] (mean)
Time per request:       504.607 [ms] (mean)
Time per request:       25.230 [ms] (mean, across all concurrent requests)
Transfer rate:          10.41 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    0   0.4      0       5
Processing:    50  498 193.1    468    1178
Waiting:       43  495 193.1    466    1177
Total:         50  498 193.1    468    1178
 ```

Locust Test 
- Max RPS is 153 for store api but is 110 for both store and query api
    - since query api will decline RPS since I do not setup db index in infuxdb
- Simulate 50 users and hatching rate in 0.1, total request is over 5245 in 1 mins and max response time would be 610 ms

```bash
[root@ip-10-222-131-36 locust-load-testing]# sh ./deploy.sh -a down -f ./perf_script/mendix-demo/locustfile.py --web_port 8888 --master_ip 10.222.131.36 --slave_number 5

------------------------------ Store API ------------------------------
 Name                                                          # reqs      # fails     Avg     Min     Max  |  Median   req/s
--------------------------------------------------------------------------------------------------------------------------------------------
 POST /mendix-demo/v1/user/status                                9333     0(0.00%)      37      10     132  |      32  153.40
--------------------------------------------------------------------------------------------------------------------------------------------
 Aggregated                                                      9333     0(0.00%)                                     153.40

Percentage of the requests completed within given times
 Name                                                           # reqs    50%    66%    75%    80%    90%    95%    98%    99%   100%
--------------------------------------------------------------------------------------------------------------------------------------------
 POST /mendix-demo/v1/user/status                                 9333     32     37     41     46     61     68     77     82    130
--------------------------------------------------------------------------------------------------------------------------------------------
 Aggregated                                                       9333     32     37     41     46     61     68     77     82    130

------------------------------ Query and Store API, ratio is 10:1 ------------------------------
 Name                                                          # reqs      # fails     Avg     Min     Max  |  Median   req/s
--------------------------------------------------------------------------------------------------------------------------------------------
 GET /mendix-demo/v1/a5166aab-7764-433a-8e83-2035b065a45b/query     143     0(0.00%)     390     246     608  |     390    2.40
 POST /mendix-demo/v1/user/status                                6583     0(0.00%)      36      11     179  |      32  107.70
--------------------------------------------------------------------------------------------------------------------------------------------
 Aggregated                                                      6726     0(0.00%)                                     110.10

Percentage of the requests completed within given times
 Name                                                           # reqs    50%    66%    75%    80%    90%    95%    98%    99%   100%
--------------------------------------------------------------------------------------------------------------------------------------------
 GET /mendix-demo/v1/a5166aab-7764-433a-8e83-2035b065a45b/query      143    390    410    430    440    470    510    560    590    610
 POST /mendix-demo/v1/user/status                                 6583     32     37     41     44     53     65     87    100    180
--------------------------------------------------------------------------------------------------------------------------------------------
 Aggregated                                                       6726     33     38     42     45     57     76    280    390    610

```

## Conclusion for performance test
You will find the max RPS is not the same between ab and locust tool
 - ab only test get API while locust test both store and query
 - ifra implement and design mindset is different
    - ab : hit total request you desired and set concurrency number
    - locust: hit in specific periods and concurrency user count you desired with hatch rate (rps should be incremented while user count increase)
        - if use distributed mode then can get more high RPS

In generally, ab is only for quickly check service performance while locust is more suite for load testing. 

ps. Jmeter will consume too many CPU/Memory IO when did the load generator and you have to set up more instances or container if you desired to hit high RPS (I was also familiar with Jmeter)
