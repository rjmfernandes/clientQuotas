# CP Client Quotas

Considering for our tests we will be using users to set client quotas we will use CP-Demo.

Check: https://docs.confluent.io/platform/current/tutorials/cp-demo/docs/on-prem.html#module-1-deploy-cp-demo-environment-using-script

Run 

```bash
git clone https://github.com/confluentinc/cp-demo
```

Navigate to the cp-demo directory and switch to the Confluent Platform release branch:

```bash
cd cp-demo
git checkout 7.5.3-post
```

To run cp-demo the first time with defaults, run the following command. The very first run downloads all the required Docker images (~15 minutes) and sets up the environment (~5 minutes).

```bash
./scripts/start.sh
```

Verify the status of the Docker containers show Up state.

```bash
docker-compose ps
```

Your output should resemble:

```
           Name                          Command                  State                                           Ports
------------------------------------------------------------------------------------------------------------------------------------------------------------
connect                       bash -c sleep 10 && cp /us ...   Up             0.0.0.0:8083->8083/tcp, 9092/tcp
control-center                /etc/confluent/docker/run        Up (healthy)   0.0.0.0:9021->9021/tcp, 0.0.0.0:9022->9022/tcp
elasticsearch                 /bin/bash bin/es-docker          Up             0.0.0.0:9200->9200/tcp, 0.0.0.0:9300->9300/tcp
kafka1                        bash -c if [ ! -f /etc/kaf ...   Up (healthy)   0.0.0.0:10091->10091/tcp, 0.0.0.0:11091->11091/tcp, 0.0.0.0:12091->12091/tcp,
                                                                              0.0.0.0:8091->8091/tcp, 0.0.0.0:9091->9091/tcp, 9092/tcp
kafka2                        bash -c if [ ! -f /etc/kaf ...   Up (healthy)   0.0.0.0:10092->10092/tcp, 0.0.0.0:11092->11092/tcp, 0.0.0.0:12092->12092/tcp,
                                                                              0.0.0.0:8092->8092/tcp, 0.0.0.0:9092->9092/tcp
kibana                        /bin/sh -c /usr/local/bin/ ...   Up             0.0.0.0:5601->5601/tcp
ksqldb-cli                    /bin/sh                          Up
ksqldb-server                 /etc/confluent/docker/run        Up (healthy)   0.0.0.0:8088->8088/tcp
openldap                      /container/tool/run --copy ...   Up             0.0.0.0:389->389/tcp, 636/tcp
restproxy                     /etc/confluent/docker/run        Up             8082/tcp, 0.0.0.0:8086->8086/tcp
schemaregistry                /etc/confluent/docker/run        Up             8081/tcp, 0.0.0.0:8085->8085/tcp
streams-demo                  /app/start.sh                    Up             9092/tcp
tools                         /bin/bash                        Up
zookeeper                     /etc/confluent/docker/run        Up (healthy)   0.0.0.0:2181->2181/tcp, 2888/tcp, 3888/tcp
```

Go to http://localhost:9021/clusters using as user superUser and password superUser, and on the topic users edit the configuration settings changing  confluent_value_schema_validation to false.

Execute:

```bash
docker-compose exec connect /bin/bash
```


```bash
tee -a ./config2.properties <<EOF
security.protocol=SSL
ssl.truststore.location=/etc/kafka/secrets/kafka.appSA.truststore.jks
ssl.truststore.password=confluent
ssl.keystore.location=/etc/kafka/secrets/kafka.appSA.keystore.jks
ssl.keystore.password=confluent
ssl.key.password=confluent
EOF
```

And after:

```bash
kafka-producer-perf-test      --throughput 100000 --num-records 1000000 --topic users  --record-size 1000 --print-metrics      --producer-props bootstrap.servers=kafka1:11091 acks=all client.id=test-1     --producer.config config2.properties
```

Check the output and you should see something like this:

```
26881 records sent, 5271.8 records/sec (5.03 MB/sec), 1714.5 ms avg latency, 4145.0 ms max latency.
34832 records sent, 6958.1 records/sec (6.64 MB/sec), 6007.6 ms avg latency, 6720.0 ms max latency.
51520 records sent, 10086.1 records/sec (9.62 MB/sec), 3000.9 ms avg latency, 4854.0 ms max latency.
20400 records sent, 4068.6 records/sec (3.88 MB/sec), 5168.0 ms avg latency, 6839.0 ms max latency.
53520 records sent, 10684.8 records/sec (10.19 MB/sec), 4626.9 ms avg latency, 7356.0 ms max latency.
```

If you now execute:

```bash
kafka-configs --bootstrap-server kafka1:12091 --alter --add-config 'producer_byte_rate=10000' --entity-type users --entity-name appSA
```

And then run again:

```bash
kafka-producer-perf-test      --throughput 100000 --num-records 650 --topic users  --record-size 1000 --print-metrics      --producer-props bootstrap.servers=kafka1:11091 acks=all client.id=test-1     --producer.config config2.properties
```

You can compare results and check the throttling is happening:

```
241 records sent, 42.6 records/sec (0.04 MB/sec), 843.5 ms avg latency, 4955.0 ms max latency.
112 records sent, 21.7 records/sec (0.02 MB/sec), 7278.0 ms avg latency, 10199.0 ms max latency.
112 records sent, 18.3 records/sec (0.02 MB/sec), 12910.6 ms avg latency, 16330.0 ms max latency.
112 records sent, 21.4 records/sec (0.02 MB/sec), 18620.7 ms avg latency, 21564.0 ms max latency.
112 records sent, 18.3 records/sec (0.02 MB/sec), 24270.8 ms avg latency, 27686.0 ms max latency.
```

If you compare the ending metrics you should also see first for throtle-time:

```
producer-metrics:produce-throttle-time-avg:{client-id=test-1}                                      : 0.000
producer-metrics:produce-throttle-time-max:{client-id=test-1}                                      : 0.000
```

And after:

```
producer-metrics:produce-throttle-time-avg:{client-id=test-1}                                      : 3023.636
producer-metrics:produce-throttle-time-max:{client-id=test-1}                                      : 3267.000
```

And the same for byte-rate first:

```
producer-topic-metrics:byte-rate:{client-id=test-1, topic=users}                                   : 19102101.903
```

And after:

```
producer-topic-metrics:byte-rate:{client-id=test-1, topic=users}                                   : 1919.926
```

Exit the connect container and stop the Docker environment, destroy all components and clear all Docker volumes.

```bash
exit
```

```bash
./scripts/stop.sh
```