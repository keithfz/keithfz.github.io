+++
title = "Olly deux"
date = "2025-07-09"
author = "keithfz"
description = "ClickHose and Kafka on K8s for observability 2.0"
+++
Charity Majors talks about observability 2.0. This refers to arbitrarily wide events that comprise all of your observability data points, rather than having seperate pillars such as metrics, logs, and traces.

I come from the Grafana stack, having run most of their products for monitoring some of my applications -- Loki for logs, Tempo for traces, Mimir for metrics, and surfaced via the Grafana UI. It's all backed by object storage, so it's been pretty cost-effective. Sure, sometimes things like text queries can be slow, you have to worry about things like cardinality -- but it's easy to spin up on Kubernetes, realatively a good performance vs cost tradeoff, and it offers all of the features I need.

Observability 2.0 is still pretty compelling to me for some reason. Thinking about why -- I think the biggest pain I've faced has always been the actual instrumentation and mechanisms for getting data out of my applications and into the system. Being able to easily dump out arbitrarily wide logs, semi-structured offers a single simple way to emit *everything*. For example, at my current work, I deal with a blend of Prometheus metrics that are scraped, OpenTelemetry based instrumentation that's pushed, some logging agents run as side-cars, others are DaemonSets that scrape stdout. There's still even statsd in a few places.

With all of this floating around inside my brain, I also just wanted to have a bit of fun and play around with analytics databases. Seemed like the right use case. 

First step is to get ClickHouse up and running. I just chose ClickHouse since it's a pretty common OLAP database. 

Where there's a nail, I see the K8s hammer, so naturally I'm running everything on K8s in a local Kind cluster. I'm using the Altinity operator for this demo which can be installed to the `clickhouse` namespace in your cluster as so:
```bash
curl -s https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/operator-web-installer/clickhouse-operator-install.sh | OPERATOR_NAMESPACE=clickhouse bash
```


In a serious ClickHouse installation you probably want Zookeeper (slash ClickHouseKeeper). Good news! There's a CRD for that. Although this is contrived, it will install the ClickHouseKeeper instance and the ClickHouse instance for you:
```yaml
---
apiVersion: clickhouse-keeper.altinity.com/v1
kind: ClickHouseKeeperInstallation
metadata:
  name: chk
  namespace: clickhouse
spec:
  configuration:
    clusters:
      - name: default
---
apiVersion: clickhouse.altinity.com/v1
kind: ClickHouseInstallation
metadata:
  name: olly-ch
  namespace: clickhouse
spec:
  configuration:
    zookeeper:
      nodes:
        - host: keeper-chk
          port: 2181
    clusters:
      - name: default
        layout:
          replicasCount: 1
```

Next problem is getting data into our system. ClickHouse is honestly not very good at getting many small writes. It excels in batching. And I don't feel like dealing with it at a client side. Kafka seems like a good solution for this, and also helps handle any spiky load. ClickHouse can natively consume from Kafka as well. Truly a match made in heaven for lazy tech blogs.

Let's grab the Strimzi Kafka operator and install it in the `kafka` namespace:
```bash
kubectl create ns kafka
helm install kafka-operator oci://quay.io/strimzi-helm/strimzi-kafka-operator -n kafka
```

We'll now need to deploy the Kafka CRDs. There's a Kafka cluster definition, the KafkaNodePool to define the controllers and brokers, as well as a KafkaTopic where we can set up our `events` topic to handle our events.
```yaml
apiVersion: kafka.strimzi.io/v1beta2
kind: KafkaNodePool
metadata:
  name: nodepool
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  replicas: 1
  roles:
    - controller
    - broker
  storage:
    type: jbod
    volumes:
      - id: 0
        type: persistent-claim
        size: 1Gi
        deleteClaim: true
        kraftMetadata: shared
---
apiVersion: kafka.strimzi.io/v1beta2
kind: Kafka
metadata:
  name: kafka-cluster
  annotations:
    strimzi.io/node-pools: enabled
    strimzi.io/kraft: enabled
spec:
  kafka:
    version: 4.0.0
    metadataVersion: 4.0-IV3
    listeners:
      - name: plain
        port: 9092
        type: internal
        tls: false
      - name: tls
        port: 9093
        type: internal
        tls: true
    config:
      offsets.topic.replication.factor: 1
      transaction.state.log.replication.factor: 1
      transaction.state.log.min.isr: 1
      default.replication.factor: 1
      min.insync.replicas: 1
  entityOperator:
    topicOperator: {}
    userOperator: {}
---
apiVersion: kafka.strimzi.io/v1beta1
kind: KafkaTopic
metadata:
  name: events
  labels:
    strimzi.io/cluster: kafka-cluster
spec:
  partitions: 1
  replicas: 1
```

Now if you've followed this correctly (and I've explained things correctly), then you should end up with the following:
```bash
➜  olly-deux kubectl get pods -n clickhouse
NAME                                   READY   STATUS    RESTARTS   AGE
chi-olly-ch-default-0-0-0              1/1     Running   0          16m
chk-chk-default-0-0-0                  1/1     Running   0          16m
clickhouse-operator-6f655c97bd-txvln   2/2     Running   0          17m
➜  olly-deux kubectl get pods -n kafka
NAME                                            READY   STATUS    RESTARTS   AGE
kafka-cluster-entity-operator-66b78476b-245tr   2/2     Running   0          94s
kafka-cluster-nodepool-0                        1/1     Running   0          2m22s
strimzi-cluster-operator-d5f66d84c-mp8n7        1/1     Running   0          10m
➜  olly-deux kubectl get kafkatopic -n kafka
NAME     CLUSTER         PARTITIONS   REPLICATION FACTOR   READY
events   kafka-cluster   1            1                    True
```

Now that we have the infrastructure pieces set up, we need to actually set up ClickHouse.

Exec into the container used by the ClickHouse StatefulSet:
```bash
kubectl exec -it chi-olly-ch-default-0-0-0 -n clickhouse -- bash
```

We can access ClickHouse using the following command in the container.
```
clickhouse-client
```

We'll need to create the database, and create 3 tables. The first table we'll call `ingest`. This uses ClickHouse's Kafka engine to consume from the topic. The ugly `kafka-cluster-nodepool-0.kafka.....` DNS name is just the K8s internal name of the Kafka broker we created earlier. We'll want to listen to the `events` topic, the consumer name will be `olly`, and we'll use `JSONEachRow` as our parsing strategy. When we receive messages, we'll try to parse them and extract them into the fields we defined in the `ingest`. I just threw together some random fields we might want to consider for an observability use case. For the general purposes of Kafka -> ClickHouse, this can really be whatever you want.
```sql
CREATE DATABASE olly

CREATE TABLE ingest
(
  timestamp UInt64,
  msg String,
  requestId String,
  host LowCardinality(String),
  tags JSON
)
ENGINE = Kafka(
  'kafka-cluster-nodepool-0.kafka-cluster-kafka-brokers.kafka.svc.cluster.local:9092',
  'events',
  'olly',
  'JSONEachRow'
);
```

Once we have the Kafka engine set up, we actually need to persist these in some sort of `*MergeTree` engine. The Kafka engine will not actually persist your data. This is pretty simple, we can just create a table called `events` that uses the same fields. One additional thing I did here of note, is that I changed the timestamp format from `UInt64` to `DateTime` to make it more readable when analyzing telemetry data.
```sql
CREATE TABLE events (
  timestamp DateTime, 
  msg String,
  requestId String,
  host LowCardinality(String),
  tags JSON
)
ENGINE = MergeTree
ORDER BY (host, timestamp);
```

*But wait* we need one more table to provide the mapping from the `ingest` table to the `events` table. We can create a materialized view, that selcts all non-empty messages from our `ingest` table, and stores them into the `events` table. We select all of the fields we want, and this also performs the conversion of timestamp to DateTime.
```sql
CREATE MATERIALIZED VIEW events_mv TO events
AS
  SELECT
    toDateTime(timestamp) AS timestamp,
    msg,
    requestId,
    host,
    tags
FROM ingest
WHERE msg <> '';
```

Ok finally. Now we just need to produce some stuff to Kafka and we should have the general structure of our pipeline. I created a silly Golang client that just publishes to the Kafka topic. Any Kafka client should work, as long as the data format matches the definitions that the consumer is expecting.

Exercise to the reader is to now query and play around with your data ;)

Later on I'd like to build out what some of the client side might actually look like in an application, and refining our ClickHouse schema to match. We'll also have to consider local collection of the metrics (we don't want our service to have to push all of our metrics over the internet, best to send it somewhere locally and get forwarded to our backend). For this, we can probably work something out with an OpenTelemetry collector. Another enhancement will be setting up Grafana to visualize some of our data.
