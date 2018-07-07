Monitoring kafka brokers inside kubernetes cluster with Burrow and Prometheus Operator
=====

Build Kafka docker image with jmx-exporter:


```Dockerfile
FROM confluentinc/cp-kafka:4.1.0

COPY prometheus-exporter/jmx_prometheus_javaagent-0.3.0.jar /usr/share/java/kafka/jmx_prometheus_javaagent-0.3.0.jar
COPY prometheus-expoerter/jmx-exporter.yml /usr/share/java/kafka/jmx-exporter.yml
```

![alt text](https://raw.githubusercontent.com/ignatev/burrow-kafka-dashboard/master/image.png)
![alt text](https://github.com/ignatev/burrow-kafka-dashboard/blob/master/Screenshot%20from%202018-04-07%2022-37-14.png?raw=true)

Kafka as stateful set inside kubernetes cluster
Put jmx_exporter.jar inside kafka container
Add jmx_exporter KAFKA_OPTS
Run burrow and burrow-exporter
Setup Prometheus targets
