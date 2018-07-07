Monitoring Kafka with Burrow and Prometheus Operator
=====

![alt text](https://raw.githubusercontent.com/ignatev/burrow-kafka-dashboard/master/image.png)

How to setup Prometheus Operator to scrape and store metrics exposed by burrow and jmx
-----

This how-to is
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
It has comprehensive documentation, please find it here: https://github.com/linkedin/Burrow/wiki \
There is only one problem with Burrow, it doesn't expose metrics in Prometheus format, and this is the reason why we should use burrow-exporter docker container for converting metrics.

Create burrow docker image:
We will mount config inside working container using Kubernetes configmap feature.

Create start.sh script and Dockerfile:

```bash
#!/bin/sh
KAFKA_VERSION=${KAFKA_VERSION:-0.10.1.0}
echo "start"
cat $CONFIG_FILE
exec $GOPATH/bin/Burrow -config-dir /etc/burrow/config/

```

```Dockerfile
FROM golang:alpine
RUN apk add --update bash curl git && apk add ca-certificates wget && update-ca-certificates && rm -rf /var/cache/apk/*
RUN go get github.com/linkedin/Burrow \
    github.com/golang/dep/cmd/dep
WORKDIR $GOPATH/src/github.com/linkedin/Burrow
RUN dep ensure && go install && mkdir -p /etc/burrow/
ADD ./ /etc/burrow/
WORKDIR /etc/burrow/
CMD ["/etc/burrow/start.sh"]

```


Prometheus Operator
-----

Prometheus Operator uses k8s services and service monitors CRD for generating Prometheus config.

Apply this manifest inside namespace where Prometheus Operator works:
Burrow Service, ConfigMap and Deployment:
```yaml
kind: Service
apiVersion: v1
metadata:
  name: burrow
  labels:
    k8s-app: burrow
spec:
  selector:
    k8s-app: burrow
  ports:
  - name: metrics
    port: 8080
---
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
---
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: burrow
  labels:
    k8s-app: burrow
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-app: burrow
  template:
    metadata:
      labels:
        k8s-app: burrow
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
    spec:
      containers:
      - name: burrow
        image: aksregistryprod.azurecr.io/burrow:v8
        ports:
        - name: api
          containerPort: 8000
        readinessProbe:
          httpGet:
            path: /burrow/admin
            port: 8000
        livenessProbe:
          httpGet:
            path: /burrow/admin
            port: 8000
        volumeMounts:
        - name: config
          mountPath: /etc/burrow/config
      - name: burrow-exporter
        image: aksregistryprod.azurecr.io/burrow-exporter
        ports:
        - name: metrics
          containerPort: 8080
        env:
        - name: BURROW_ADDR
          value: http://localhost:8000
        - name: METRICS_ADDR
          value: 0.0.0.0:8080
        - name: INTERVAL
          value: "15"
      volumes:
      - name: config
        configMap:
          name: burrow-config

```

Modify Kafka service manifest -- add port on which jmx metrics exposed and add label for Prometheus Operator
Kafka Service:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: kafka-hs
  labels:
    k8s-app: kafka-metrics
spec:
  clusterIP: None
  selector:
    app: kafka
  ports:
  - port: 9092
    name: internal
  - port: 9093
    name: client
  - port: 5555
    name: metrics
```

ServiceMonitor
----
Apply these manifests inside the namespace where your Prometheus Operator works, e.g. monitoring

Kafka ServiceMonitor:
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: kafka-metrics
  labels:
    k8s-app: kafka-metrics
spec:
  selector:
    matchLabels:
      k8s-app: kafka-metrics
  namespaceSelector:
    matchNames:
    - kafka-namespace
  endpoints:
  - port: metrics
    interval: 10s
```

Burrow ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: burrow
  labels:
    k8s-app: burrow
spec:
  selector:
    matchLabels:
      k8s-app: burrow
  namespaceSelector:
    matchNames:
    - monitoring
  endpoints:
  - port: metrics
    interval: 10s

```
