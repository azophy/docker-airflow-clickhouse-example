End-to-end log processing pipeline using Minio-Airflow-Clickhouse
=================================================================

personal experiment with Apache Airflow. based on:
- [Running Airflow in Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)

## Overview

```mermaid
flowchart LR
    A[NestJS
    worker] -->|Fluentd collect
    docker log| B(Minio)
    B --> |Airflow DAG| D[Clickhouse]
    D --> E[Grafana]
```

This is my experiment on implementing end-to-end log processing pipeline. Outline of the pipeline:
- the log is generated by services and sent via stdout to be collected by docker
- the log collected by docker would be further sent into Minio object storage with the help of fluentd
- periodically, and Airflow DAG would push the logs in minio into Clickhouse

## Key Technologies
- docker compose : orchestration
- [NestJs](https://nestjs.com) : web service
- [FluentBit](https://fluentbit.io) : log file collector
- [Minio](https://min.io) : object storage for collection log files
- [Apache Airflow](https://airflow.apache.org) : data processing pipeline
- [Clickhouse](https://clickhouse.com) : final analytic database / data warehouse
- [Grafana](https://grafana.com) : Visualization

## Install
1. clone repo
2. copy `.env.example` to `.env`
3. setup airflow user:
  ```bash
  echo -e "AIRFLOW_UID=$(id -u)" >> .env
  ```
4. docker compose up -d

## Using this repo
### Minio
check localhost:9001 . the default user & password is 'minio-root'

### Airflow
availabel on localhost:8080 . the default username & password is 'airflow'

### Clickhouse
the default username is 'default' with password 'clickhouse-root'

### Grafana
check localhost:3002

### Generating load
Use docker compose exec to run wrk with the provided script

```bash
docker compose exec tester wrk -c 100 \
                               -t 5 \
                               --timeout 1s \
                               --latency \
                               -d "30s" \
                               -s ./script.lua \
                               http://nestjs:3000/dummy
```

## Pipeline Explanation
### FluentBit
This acts as log collector, tail-ing all docker logs in the host, adding several metadata like
container id & name, and forwarding it to Minio. If you have played with FluentBit before, you may
realize that FluentBit is pretty powerful. Its even possible to directly pipe all docker logs into
clickhouse without needing any intermediary steps like Minio & Airflow. However in the next
section I will explain some advantages we try to gains by using the longer method.

On the other hand, I faced some challenges when formatting & parsing the docker log files. Because
we only used the docker log files, we dont get many docker related by default (including container
name). To solve that, we use some lua script to parse container_id from the FluentBit tag, and
retrieve container config file in docker folder to finally obtain container name.

### Minio
Act as a "data lake". By introducing Minio as log storage instead of directly piping the log into
clickhouse, we gain several advantages:
- allow excess load handling incase there are peak traffic
- act as backup and allow better data retention policy

In production we may replace Minio with cloud offerings such as Amazon's S3 or Cloudflare's R2.
But for our project here, using Minio is sufficient.

The issue that I just found out about Minio, which actually just tried to match with S3's API
spec, is regarding sorting the filename. S3 actually dont provide any way to sort the files based
on time. To handle this, I decided to add timestamp into our filename so we would be able to sort
the file.

### Airflow
here, we build a simple DAG script to periodically check our minio instance, import into
clickhouse, and keep track which log file we already imported. To keep track of the log file, we
create a 'migration' table to store all log files & its import status
(new/processing/success/error)

### Clickhouse
Our main analytical engine and "data warehouse". Clickhouse is a very powerful OLAP database with
lots of features & integrations

### Grafana
Our visualization tool. We use this to access Clickhouse and provide basic dashboard. In this repo
we have setup clickhouse plugin so our Grafana instance is ready to connect with our clickhouse
server. Just add a new connection in "Home > Connections > Data sources > Add data source". Search
for "clickhouse" and use this settings:
- server address: clickhouse
- port: 9000
- protocol: native
- username: default
- password: clickhouse-root

Click "save & test" to make sure our inputs are correct. If you notification that data source is
working, you could head to "Home > Explore".


