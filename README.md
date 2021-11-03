# testing-logstash

### What is this
This repo includes a Dockerfile which creates CentOS based image and logstash testing tools.

### Where I can find the latest Docker image
`docker push mrceyhun/testing-logstash:latest`

### How I can deploy
You can deploy simple `logstash.yaml` to your kubernetes cluster.

### What is included in 'testing-logstash' image
- Logstash-7.x
- Python3
- nc (Netcat)
- nano
- vim
- `logstash` alias to `/usr/share/logstash/bin/logstash`
- WORKDIR is /data

## How I can test Logstash config
There are 2 scenarios included in this examples: `stdout` output filter and `http` output filter examples.

First of all deploy kubernetes deployment:
`kubectl apply -f logstash.yaml`

---
1. **Testing with `stdout` output filter**<br />
- Copy logstash config file and input logs file ("frontend.log" is used in logstash-ex2.conf's input filter) to k8s pod: 
  - `kubectl cp examples/logstash-ex2.conf logstash-b8d54cb84-j7fxx:/data -n default`
  - `kubectl cp frontend.log logstash-b8d54cb84-j7fxx:/data -n default`
- Get a shell in kubernetes pod:
  - `kubectl -n default exec --stdin --tty logstash-b8d54cb84-j7fxx -- /bin/bash`
- Run logstash with necessary flags (please see their meanings: logstash --help ):
  - `logstash -r -f logstash-ex2.conf`
- Both filtering output and Logstash info logs will be written to stdout, not to "/usr/share/logstash/logs".
- You can use `json` or `rubydebug` codec in output filter
- You don't need to start and stop logstash because it takes long time to start. `-t` flag allows to reread input files when there is a change in config file
- So, I suggest that keep running logstash, open new terminal tab and edit logstash-ex2.conf and save. These 2 configurations in input filter will allow to read input file from the beginning again  with the new configuration of course:
  - `start_position => "beginning"` 
  - `sincedb_path => "/dev/null"`

---

2. **Testing with `http` output filter**<br />
"http" output filter posts http requests to defined host:port. In order to catch these requests, you need a server which listens incoming requests. 

> Not: If you define "format => json" in "http" output filter, you will get exactly one json body for each line of input logs. Besides, if you define "format => json_batch" in "http" output filter, you will get a body like array of jsons which represents multiple line of input logs. We use "json_batch" in our config.

**Let's test**
- Copy logstash config file and input logs file ("frontend.log" is used in logstash-ex1.conf's input filter) to k8s pod:
  - `kubectl cp examples/logstash-ex1.conf logstash-b8d54cb84-j7fxx:/data -n default`
  - `kubectl cp frontend.log logstash-b8d54cb84-j7fxx:/data -n default`
- Get a shell in kubernetes pod:
  - `kubectl -n default exec --stdin --tty logstash-b8d54cb84-j7fxx -- /bin/bash`
- Change port number of http output filter in logstash-ex1.conf. I used 7777, see `http://127.0.0.1:7777` line. Use same port as argument to ./server.py 
- Run server.py either with
  - `./server.py <port>`                           # Prints raw data of incoming requests
  - or
  - `./server.py <port> 2>&1 | grep -F '[{' | jq`  # Prints body of the incoming requests in pretty json format
- Open another terminal tab and run logstash:
  - `logstash -r -f logstash-ex1.conf`
- You will see all fields you will send to your production server in the terminal tab you run "server.py".
- To continue testing without restarting logstash, you can open a 3rd shell in kubernetes pod and make changes in your configuration. After saving the config file, logstash will reread input data and send to defined host:port.
---


## References
- https://www.elastic.co/guide/en/logstash/current/input-plugins.html
- https://www.elastic.co/guide/en/logstash/current/output-plugins.html
- https://www.elastic.co/guide/en/logstash/current/filter-plugins.html
- https://www.elastic.co/guide/en/logstash/current/codec-plugins.html
- https://github.com/dmwm/CMSKubernetes/blob/master/kubernetes/cmsweb/monitoring/logstash.conf
- https://github.com/dmwm/CMSKubernetes/tree/master/kubernetes/cmsweb/monitoring
