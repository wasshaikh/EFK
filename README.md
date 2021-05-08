# EFK

**Docker Compose**
This article explains how to collect Docker logs and propagate them to EFK (Elasticsearch + Fluentd + Kibana) stack. The example uses Docker Compose for setting up multiple containers..


**Kibana**
​Elasticsearch is an open-source search engine known for its ease of use. Kibana is an open-source Web UI that makes Elasticsearch user-friendly for marketers, engineers and data scientists alike.

By combining these three tools EFK (Elasticsearch + Fluentd + Kibana) we get a scalable, flexible, easy to use log collection and analytics pipeline. In this article, we will set up four (4) containers, each includes:

​Apache HTTP Server​

​Fluentd​

​Elasticsearch​

​Kibana​

All the logs of httpd will be ingested into Elasticsearch + Kibana, via Fluentd.

Prerequisites: Docker
Please download and install Docker / Docker Compose. Well, that's it :)

​**Docker Installation​**

**Step 0: Create docker-compose.yml**
Create docker-compose.yml for Docker Compose. Docker Compose is a tool for defining and running multi-container Docker applications.

With the YAML file below, you can create and start all the services (in this case, Apache, Fluentd, Elasticsearch, Kibana) by one command:

version: '3'
services:
  web:
    image: httpd
    ports:
      - "80:80"
    links:
      - fluentd
    logging:
      driver: "fluentd"
      options:
        fluentd-address: localhost:24224
        tag: httpd.access
​
  fluentd:
    build: ./fluentd
    volumes:
      - ./fluentd/conf:/fluentd/etc
    links:
      - "elasticsearch"
    ports:
      - "24224:24224"
      - "24224:24224/udp"
​
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.10.2
    environment:
      - "discovery.type=single-node"
    expose:
      - "9200"
    ports:
      - "9200:9200"
​
  kibana:
    image: kibana:7.10.1
    links:
      - "elasticsearch"
    ports:
      - "5601:5601"
The logging section (check Docker Compose documentation) of web container specifies Docker Fluentd Logging Driver as a default container logging driver. All the logs from the web container will automatically be forwarded to host:port specified by fluentd-address.

Step 1: Create Fluentd Image with your Config + Plugin
Create fluentd/Dockerfile with the following content using the Fluentd official Docker image; and then, install the Elasticsearch plugin:

# fluentd/Dockerfile
​
FROM fluent/fluentd:v1.12.0-debian-1.0
USER root
RUN ["gem", "install", "fluent-plugin-elasticsearch", "--no-document", "--version", "4.3.3"]
USER fluent
Then, create the Fluentd configuration file fluentd/conf/fluent.conf. The forward input plugin receives logs from the Docker logging driver and elasticsearch output plugin forwards these logs to Elasticsearch.

# fluentd/conf/fluent.conf
​
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>
​
<match *.**>
  @type copy
​
  <store>
    @type elasticsearch
    host elasticsearch
    port 9200
    logstash_format true
    logstash_prefix fluentd
    logstash_dateformat %Y%m%d
    include_tag_key true
    type_name access_log
    tag_key @log_name
    flush_interval 1s
  </store>
​
  <store>
    @type stdout
  </store>
</match>
NOTE: The detail of used parameters for @type elasticsearch, see Elasticsearch parameters section and fluent-plugin-elasticsearch furthermore.

**Step 2: Start the Containers**
**Let's start the containers:**

$ docker-compose up
Use docker ps command to verify that the four (4) containers are up and running:

$ docker ps
CONTAINER ID        IMAGE                                                 COMMAND                  CREATED             STATUS              PORTS                              NAMES
558fd18fa2d4        httpd                                                 "httpd-foreground"       17 seconds ago      Up 16 seconds       0.0.0.0:80->80/tcp                 docker_web_1
bc5bcaedb282        kibana:7.10.1                                         "/usr/local/bin/kiba…"   18 seconds ago      Up 17 seconds       0.0.0.0:5601->5601/tcp             docker_kibana_1
9fe2d02cff41        docker.elastic.co/elasticsearch/elasticsearch:7.10.2  "/usr/local/bin/dock…"   20 seconds ago      Up 18 seconds       0.0.0.0:9200->9200/tcp, 9300/tcp   docker_elasticsearch_1

**Step 3: Generate httpd Access Logs**

Use curl command to generate some access logs like this:

$ curl http://localhost:80/[1-10]
<html><body><h1>It works!</h1></body></html>
<html><body><h1>It works!</h1></body></html>
<html><body><h1>It works!</h1></body></html>
<html><body><h1>It works!</h1></body></html>

**Step 4**: Confirm Logs from Kibana
Browse to http://localhost:5601/app/management/kibana/indexPatterns and set up the index name pattern for Kibana. Specify fluentd-* to Index name or pattern and click Create.


​Kibana Index Kibana Timestamp​

Then, go to Discover tab to check the logs. As you can see, logs are properly collected into the Elasticsearch + Kibana, via Fluentd.
