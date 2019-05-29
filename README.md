# docker-compose-prometheus-elasticsearch-mongo
Project to setup Prometheus Monitoring and Grafana to monitor Elasticsearch and MongoDB

## What is this project for?
The goal of the project is to show how to set up all the components needed to monitor Elasticsearch and MongoDB. I'm using Prometheus for monitoring and Grafana for visualization. The project is not going to explain how to use or what is Prometheus and Grafana. 

> Let's go over the list of components that we need for the project.

## List of components:
1. Prometheus
2. Grafana
3. MongoDB
4. Elasticsearch
5. MongoDB Prometheus Exporter
6. Elasticsearch Prometheus Exporter

## What is the exporter
Learn more about exporters in Prometheus [here](https://prometheus.io/docs/instrumenting/exporters/). 

## What is the deployment process?
I'm using docker-compose to deploy all the components in one-go.

## What is in the `mongo-exporter` folder?
The mongo-exporter doesn't yet have an official Container Image built. So the Dockerfile inside the `mongo-exporter` folder downloads the latest code from the GitHub repository, build & makes the binary, bundles the final image to use.

## Let's take a look at what is in the `docker-compose.yml` file
```
version: '3.2'
services:
# elastic search component. exposing 9200 port
  elastic_search:
    restart: 'always'
    ports:
    - '9200:9200'
    image: elasticsearch
    container_name: elasticsearch-container
# setting the environment variable discovery.type=single-node to bring up a single node elasticsearch
    environment:
    - 'discovery.type=single-node'
# provide the network alias name to use it in the exporter
    networks:
      infranet:
        aliases:
        - 'elasticsearch-service'
# elasticsearch prometheus exporter to monitor the elasticsearch service. expose port 9108.
# We will use this information to configure the prometheus to listen.
  es_exporter:
    restart: 'always'
    ports:
    - '9108:9108'
    image: justwatch/elasticsearch_exporter
    container_name: es-exporter-container
# pass the elasticsearch url as an input to the exporter
    command:
    - '-es.uri=http://elasticsearch-service:9200'
# specify the depends on the elasticsearch so the docker brings up this service only after the elasticsearch is up.
# depends_on is not a full safe approach. read more about it here. 
# https://docs.docker.com/compose/startup-order/
    depends_on:
    - elastic_search
    networks:
      infranet:
        aliases:
        - elasticsearch-exporter-service
# mongodb service
  mongo:
    restart: 'always'
    ports:
    - '27017:27017'
    image: mongo
    container_name: mongo-container
    networks:
      infranet:
        aliases:
        - 'mongodb-service'
# setting up the mongodb exporter service. exposing the port 9001
  mongo_exporter:
    restart: 'always'
    ports:
    - '9001:9001'
# as you might have noticed, I'm using my personal repository. because as you know, the mongodb exporter is not 
# officially available as a container image. so i locally build it with the below name.
    image: gpuliyar/mongo-exporter
    container_name: mongo-exporter-container
# provide the mongodb service information for the exporter to work
    environment:
    - 'MONGO_SERVICE=mongodb-service'
    - 'MONGO_PORT=27017'
    depends_on:
    - mongo
    networks:
      infranet:
        aliases:
        - 'mongodb-exporter-service'
# start the grafana service
  grafana:
    restart: 'always'
    ports:
    - '3000:3000'
    image: grafana/grafana
    container_name: grafana-container
# start the prometheus service
  prometheus:
    restart: 'always'
    ports:
    - '9090:9090'
    image: prom/prometheus
    container_name: prometheus-container
    command:
    - '--config.file=/prometheus/config/prometheus.yaml'
    - '--storage.tsdb.path=/data'
    volumes:
    - ./config:/prometheus/config
    - ./data:/data
    depends_on:
    - activemq
    - es_exporter
    - mongo_exporter
    networks:
      infranet:
        aliases:
        - 'prometheus-service'
networks:
  infranet:
```

## Let's try to understand what is in the `config/promethues.yaml` file
Read [here](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) to know more about Prometheus Configuration.

```
# global configuration 
global:
  scrape_interval: 15s
  evaluation_interval: 15s
# gloabl external labels for mapping
  external_labels:
    monitor: rnd
    environment: poc-environment
    service: prometheus
    region: local
    dc: local
# scrape configuration starts here
scrape_configs:
# configuration to monitor the prometheus service itself.
- job_name: prometheus
  scrape_interval: 30s
  scrape_timeout: 30s
  honor_labels: true
  static_configs:
  - targets: ['prometheus-service:9090']
# configuration to monitor the elasticsearch service by listening to elasticsearch exporter
- job_name: elasticsearch
  scrape_interval: 30s
  scrape_timeout: 30s
  honor_labels: true
  static_configs:
  - targets: ['elasticsearch-exporter-service:9108']
# configuration to monitor the mongodb service by listening to mongodb exporter
- job_name: mongodb
  scrape_interval: 30s
  scrape_timeout: 30s
  honor_labels: true
  static_configs:
  - targets: ['mongodb-exporter-service:9001']
```

## Important note:
`docker-compose` command sometimes fails to bring up the `prometheus` service due to access permission to `data` folder. In that case, run the below command as a workaround.

```
chmod 777 data
```

## That's it, job done!
