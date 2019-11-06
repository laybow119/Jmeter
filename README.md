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
 - User Story 1: user can send behavior data to servie such as user love/like/angy/cry for specific article in a specific time
 - User Story 2: user can query behavior data from service, for example: it can show how many articles that user likes it. 

### Prepared Test Script
Here does not eloborate all aspect of test cases, only demo how to write test script
 - Functional: 
    - store behavior test
    - query behavior test (input invalid data payload)
 - Integration:
    - user store its realtime behavior and can be retrieved
    - user store different behavior several times at different timing and it can be queried with all records based on specific status

### Task Assumptions
 - Assume this CI/CD flow will be automaticially trigger if code submitted/merged by developer 
 - Assume there is "Dockerfile" for build up microservices which implement by developer or sdet
 - Assume there is test script for testing service quality implement by sdet
 - Assume there is output folder where store build result or debug log

### CI Steps Explanation:
There are several steps in CI/CD progress for constructing pipeline.
 1. Git sync from code repository
 2. Build up microservice based on Dockerfile
 3. Deploy service to QA/Local environment
 4. Run test parallelly (Ideally to run parallelly but for demo only has master node)
 5. Push image to private docker registry
 6. Clean up temporary docker image file (avoid disk issue of no space left) 
 7. Archive build/test reulst to artifacts on jenkins 
 8. Send notification of build result to somewhere such as collaboration channel of team

### Conclusion for CI Pipleline Construction
 - You have to fingure out the whole picture of pipeline purpose
   - take care for both deployment and test process
   - send notification to collaboration channel no matter what build result is succee or failure
 - The main issues for constructing pipeline 
   - infrastructure design
   - script error handling
   - debug message is hard to understand

## Task 2 Brief Description
Evaluation for the microservice capacity

### Task Spec
Check capacity is expected the hit total request count to 500 and completing 400 in 10 mins (RPS should be less than 1) 

### What purpose stands for performance test (load testing)?
 1. Evaluate the service capacity that how many Request Per Seconnd (RPS) can be taken against microservice 
   - maximum RPS for single container with resource limitataion when deployed
 2. Check the response time for a request is acceptable (user acceptance)
   - the response time shoule be less than 10 seconds under high RPS
 3. Evaluate the service stablity under high RPS condition for specific long periods (should be more than 3 hours)
    - in my experience, sometimes service will get memory leak issue due to code does not handle memory GC or DB connection well under high RPS condition.
 4. Provide the test result of maximum RPS can benefit team that OPS and PM/PO can make some decision or strategy with strong evidence
    - operation team will know how many containers should be launched at the sametime on production environment against the target RPS
    - project manager or product owner will know it can be fulfill the requirement or not(real online user traffic)

### Load Testing Tool
Why select both Apache Benchmark (ab) and Python Locust to do the performance test?
 1. Apache Benchmark Testing (https://httpd.apache.org/docs/2.4/programs/ab.html)
    - it is a tool for benchmarking your RESTful service
    - it can quickly hit the api with high RPS and check service status
        - only for quickly test single api
    - opensource project
 2. Python Locust (https://locust.io/)
    - it can simulate thousands of concurrent users on a single machine. (cost concern)
    - it write very expressive scenarios as user behavior to access your service
    - support distributed mode which is master and slave architect
      - it can easy setup load generator with high RPS without extra effort
      - framework will controal the comunication well between master and slave nodes
    - support multi protocol load testing such as RESTful and gRPC (https://grpc.io/) protocol
    - opensource project

### Task Requirement
 - Target RPS is higher than 1 for total request count 500 and need to complete 400 in 10 mins
    - in demo-service, assume the api ratio that sotre is 50 and query is 1(just for demo, simulate store action is more frequently than query)
 - The response time for a request should be less than 10 seconds

### Test Script
Check "locustfile.py" file in locust-load-testing folder and path is /perf_script/mendix-demo
 - Apply locust framework
    - impelement and inheritance of serveral class ex: HttpLocust, TaskSet, task, Locust
 - Simply to write the test scenario as user behavior includes of store and query action

### Setup performance environment
 - Apache benchmarking
   - install ab command : sudo yum install -y httpd-tools
   - command : ab -n $total_count -c $concurrency_count $url
 - Python Locust (Dockerize solution which design by myself)
    - deploy one master and five slave nodes in local machine 
        - command : sh ./deploy.sh -a $action -f $locust_file --web_port $web_port --master_ip $master_ip --slave_number $slave_number 
        - check the locust_master_salve.yaml file and deploy.sh in locust-load-testing folder for more deployment setting
    - in this demo is quickly using --no_web config that only come out the statistic result in standard output
    - deployment for launch master and slave nodes as below 

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
    - in this demo it does not demo this part (but will give you sceenshot of previous project)
    - install command
```bash
wget https://dl.influxdata.com/telegraf/releases/telegraf-1.12.0-1.x86_64.rpm
yum localinstall telegraf-1.12.0-1.x86_64.rpm
vim /etc/telegraf/telegraf.conf // modify target url that output data to influxdb
```

### Result for load testing
 AB test
 - The Max RPS of store and query api are 155 and 40, respectively. And response time is less than 10 secs for both api
 - AB test only can test one api at the same time (you still can launch multiple process to hit mutli api)
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
    - query api will decline RPS while user data incremented since I do not fine tune the query performance for accessing influxdb 
- Simulate 50 users and hatching rate in 0.1, total request is over 5245 in 1 mins and max response time would be 610 ms

```bash
[root@ip-10-222-131-36 locust-load-testing] sh ./deploy.sh -a deploy -f ./perf_script/mendix-demo/locustfile.py --web_port 8888 --master_ip 10.222.131.36 --slave_number 5

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

### Conclusion for performance test
You will find the max RPS is not the same between ab and locust tool
 - ab only can test one api for one process while locust test both store and query api
 - infrastructure implement and design model is different
    - ab : hit total request you desired and set concurrency number
    - locust: hit in specific periods and concurrency user count you desired with hatch rate (rps should be incremented while user count increase)

In generally, ab is only for quickly check service performance while locust is more suite for load testing of multiple api and different protocol. 

In my experience, locust web UI console is time series based which means you will missing some data if run load testing for serveral hours due to it does not make sense for keeping browser open for long period.
 - I did the survey and come out the solution was that create another worker to flush statistic result into influxdb and use grafana dashboard to query and show it.

 - in this task, I only run test on local machine (one node only) and --no_web mode but I can give some screenshot for above desicription, include web ui, system information and statistic result show by gragana dashboard
 - the screenshot has been taken in my recent project that load testing against RESTful and gRPC microservice and also wrote the wiki for guide other SDET member how to use it

ps. Why do not select Jmeter due to it consume too many CPU/Memory IO when launch the load generator and you have to set up more instances or container if you desired to hit high RPS (I was also familiar with Jmeter script)
