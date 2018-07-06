Monitoring kafka brokers inside kubernetes cluster with Burrow and Prometheus Operator
=====

How to setup Prometheus Operator to scrape and store metrics exposed by burrow and jmx
-----

* Download jmx-exporter: https://github.com/prometheus/jmx_exporter
* Create jmx-exporter.yml configfile, here are couple links:
https://github.com/prometheus/jmx_exporter/blob/master/example_configs/kafka-0-8-2.yml
https://github.com/Yolean/kubernetes-kafka/blob/master/prometheus/10-metrics-config.yml

* Build Kafka docker image with jmx-exporter:

```Dockerfile
FROM confluentinc/cp-kafka:4.1.0

COPY prometheus-exporter/jmx_prometheus_javaagent-0.3.0.jar /usr/share/java/kafka/jmx_prometheus_javaagent-0.3.0.jar
COPY prometheus-expoerter/jmx-exporter.yml /usr/share/java/kafka/jmx-exporter.yml
```

Kafka kubernetes manifests
-----
Enable jmx-exporter java agent, add this options into *env* part of Kafka statefulset manifest:

```yaml
        - name: KAFKA_OPTS
          value: >
            -javaagent:/usr/share/java/kafka/jmx_prometheus_javaagent-0.3.0.jar=5555:/usr/share/java/kafka/jmx-exporter.yml
```

Burrow
-----

Burrow is recomended tool for monitoring consumer lag -- one of the most important metric for Kafka cluster.
It has comprehensive documentation, please find it here: https://github.com/linkedin/Burrow/wiki
There is only one problem with Burrow, it doesn't expose metrics in Prometheus format, and this is the reason why we should use burrow-exporter docker container for converting metrics.

Create Burrow config.toml:

```toml
kind: ConfigMap
metadata:
  name: burrow-config
apiVersion: v1
data:
  burrow.toml: |-
    [general]
    access-control-allow-origin="*"
    
    [logging]
    level="info"
    
    [zookeeper]
    servers=["zk-0.zk-hs:2181","zk-1.zk-hs:2181","zk-2.zk-hs:2181", "zk-3.zk-hs:2181", "zk-4.zk-hs:2181"]
    
    [client-profile.kafka-profile]
    kafka-version="0.10.2.0"
    client-id="burrow-client"
    
    [cluster.gw]
    class-name="kafka"
    client-profile="kafka-profile"
    servers=["kafka-0.kafka-hs:9092","kafka-1.kafka-hs:9092","kafka-2.kafka-hs:9092","kafka-3.kafka-hs:9092","kafka-4.kafka-hs:9092"]
    topic-refresh=120
    offset-refresh=10
    
    [consumer.consumer_kafka]
    class-name="kafka"
    cluster="gw"
    servers=["kafka-0.kafka-hs:9092","kafka-1.kafka-hs:9092","kafka-2.kafka-hs:9092","kafka-3.kafka-hs:9092","kafka-4.kafka-hs:9092"]
    client-profile="kafka-profile"
    start-latest=true
    offsets-topic="__consumer_offsets"
    group-whitelist=".*"
    group-blacklist="^(console-consumer-|python-kafka-consumer-).*$"
    
    [httpserver.default]
    address=":8000"

```

![alt text](https://raw.githubusercontent.com/ignatev/burrow-kafka-dashboard/master/image.png)
![alt text](https://github.com/ignatev/burrow-kafka-dashboard/blob/master/Screenshot%20from%202018-04-07%2022-37-14.png?raw=true)

Kafka as stateful set inside kubernetes cluster
Put jmx_exporter.jar inside kafka container
Add jmx_exporter KAFKA_OPTS
Run burrow and burrow-exporter
Setup Prometheus targets
