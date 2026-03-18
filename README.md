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
git checkout 8.1.0-post
```

To run cp-demo the first time with defaults, run the following command. The very first run downloads all the required Docker images (~15 minutes) and sets up the environment (~5 minutes).

```bash
./scripts/start.sh
```

Verify the status of the Docker containers show Up state.

```bash
docker compose ps
```



Go to http://localhost:9021/clusters using as user superUser and password superUser, and on the topic users edit the configuration settings changing  confluent_value_schema_validation to false.

Execute:

```bash
docker-compose exec connect /bin/bash
```

Disable broker side schema id validation:

```shell
kafka-configs \
  --bootstrap-server kafka1:9091 \
  --alter \
  --topic users \
  --add-config 'confluent.value.schema.validation=false' \
  --command-config /etc/kafka/secrets/client_sasl_plain.config
```

After that:

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