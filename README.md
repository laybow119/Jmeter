### Requirement and Support Feature:
  - Python3 + Locust + influxdb

### Build image
`docker build -t mendix_demo_locust .`

### Run as Service
`sh ./deploy.sh`
  - -a : redeploy / deploy / down 
  - -m : master / slave / standalone
  - -f : path for your locust file, ex: perf_script/$project_name/locustfile.py
  - -web_port : web console port
  - -master_ip : master ip where slaves will to connect with
  - -slave_number : define that how many slave you want to launch

### deploy command example :
  - standalone : `sh ./deploy.sh -a deploy -m standalone -f ./perf_script/mendix-demo/locustfile.py`
  - master : `sh ./deploy.sh -a deploy -m master -f ./perf_script/mendix-demo/locustfile.py --web_port 40000`
  - slave : `sh ./deploy.sh -a deploy -m slave -f ./perf_script/mendix-demo/locustfile.py --master_ip localhost --slave_number 5`

### master.yml environments
  - LOCUST_MODE : master / slave / standalone
  - LOCUST_MASTER_WEB_PORT : port on which to run web host
  - LOCUST_MASTER_HEARTBEAT_LIVENESS : set number of seconds before failed heartbeat from slave.
  - LOCUST_MASTER_HEARTBEAT_INTERVAL : set number of seconds delay between slave heartbeats to master.
  - LOCUST_MASTER_NO_WEB : Disable the web interface, and instead start running the test immediately.
  - LOCUST_MASTER_TESTING_PERIOD : Stop after the specified amount of time, e.g. (300s, 20m, 3h, 1h30m, etc.). only used together with $LOCUST_MASTER_NO_WEB is "true"
  - LOCUST_MASTER_NUM_CLIENTS : Number of concurrent Locust users. Only used together with $LOCUST_MASTER_NO_WEB is "true"
  - LOCUST_MASTER_HATCH_RATE : The rate per second in which clients are spawned. only used together with with $LOCUST_MASTER_NO_WEB is "true"
  - LOCUST_MASTER_EXPECT_SLAVE_NUM : How many slaves master should expect to connect before starting the test. only used together with $LOCUST_MASTER_NO_WEB is "true"

### slave.yml environments
  - LOCUST_MODE : master / slave / standalone
  - LOCUST_MASTER_HOST : Host or IP address of locust master for distributed for loading test.
