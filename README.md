# Elasticsearch, Logstash, Kibana (ELK) Dockerization

This guide is for ELK 8.17.0 using Docker Compose.

Logstash receives data, transmits it to Elasticsearch which can then be viewed in Kibana.

1. [Without X-Pack security enabled](#1-without-x-pack-security-enabled)
2. [With X-Pack security enabled](#2-with-x-pack-security-enabled)

> **Note:** Credentials used throughout this guide are examples only. Replace them with strong, unique values for any real deployment.

## 1. Without X-Pack security enabled

#### Plan of action

1. Pull ELK images
2. Write logstash config
3. Write docker-compose file
4. Execution and access

#### Pull ELK images

    docker pull docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    docker pull docker.elastic.co/logstash/logstash:8.17.0
    docker pull docker.elastic.co/kibana/kibana:8.17.0

Show pulled images by executing

    docker images

#### Write logstash config

Create a file named **logstash.conf**.

    input {
        tcp {
            port => 9600
            codec => json
        }
    }
    output {
        elasticsearch {
            hosts => ["elasticsearch:9200"]
        }
        stdout { codec => rubydebug }
    }

##### Explanation of logstash config

- `input` allows to receive incoming connection. Here, we're receiving **TCP** input connection on port **9600** having **codec json**.
- `output` allows logstash to output outgoing connection. In our case output is being transmitted to elasticsearch on its default address **elasticsearch:9200**. **rubydebug** outputs event data using the ruby "awesome_print" library. This is the default codec for stdout.

#### Write docker-compose file

Create a file named **docker-compose.yml** for setup your docker-compose stack. Here is our directory structure

> ELK Dockerization
>> docker-compose.yml
>> logstash.conf

##### Explanation of docker-compose file

- `elasticsearch`, `logstash` and `kibana` are service names, you can change the name whatever you want.
- `container_name` is a name for your container.
- `xpack.security.enabled=false` disables security for this simple setup. In Elasticsearch 8.x, security is enabled by default [1], so we explicitly disable it here.
- `volumes` to define a file/folder that you want to use for the container.
- `./elasticsearch-data:/usr/share/elasticsearch/data` means you want to set data on container persist on your local folder named **elasticsearch-data**. **/usr/share/elasticsearch/data** is a folder that is already created inside the elasticsearch container.
- `./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro` mounts the pipeline config into Logstash's pipeline directory [3].
- `ports` is to define which ports you want to expose and define, in this case we're using default ports **9200, 9600, 5601** and exposing it to host machine.
- `ntw-elk` is used as a bridge to connect all the three services in a network.
- `healthcheck` verifies Elasticsearch is ready before dependent services start.
- `depends_on` with `condition: service_healthy` ensures Logstash and Kibana wait for Elasticsearch to be healthy [7].

#### Execution and access

Now run the docker-compose file with `docker compose up` or `docker compose up -d` to run containers in the background [6].

Type `docker ps` to see our running containers.

Once the services are up and running, open your browser and open the url http://localhost:9200/. That's elasticsearch.
Also, if you navigate to http://localhost:5601/ you should see the Kibana console.
To make your application send data to logstash, use localhost:9600 TCP address.

## 2. With X-Pack security enabled

In Elasticsearch 8.x, security features (including TLS) are enabled by default [1]. We still need to generate certificates for transport-layer encryption and configure credentials for inter-service communication.

1. Pull ELK images
2. Create SSL certificate for Elasticsearch and enable SSL
3. Write elasticsearch settings
4. Write logstash config
5. Write kibana settings
6. Write docker-compose-elk-auth file
7. Installing the SSL certificate on Elasticsearch and enabling TLS in config
8. Generate passwords and configure the credentials
9. Update passwords in config files
10. Execution and access

#### Pull ELK images

    docker pull docker.elastic.co/elasticsearch/elasticsearch:8.17.0
    docker pull docker.elastic.co/logstash/logstash:8.17.0
    docker pull docker.elastic.co/kibana/kibana:8.17.0

Show pulled images by executing

    docker images

#### Create SSL certificate for Elasticsearch and enable SSL

If you already have an SSL certificate you can use that, if not, then follow the steps below to create it.

##### Write docker-compose-cert file

    services:

      elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:8.17.0
        container_name: elasticsearch
        volumes:
          - ./elasticsearch-data:/usr/share/elasticsearch/data

##### Execute **docker-compose-cert.yml** file

    docker compose -f docker-compose-cert.yml up

Now we will need to get inside our docker container running elasticsearch service.

    docker compose exec -it elasticsearch bash

Create certificate authority. Execute the below command in the bash of container to create **elastic-stack-ca.p12** file.

    bin/elasticsearch-certutil ca

You can set or ignore certificate authority filename and its password.

Create **elastic-certificates.p12** SSL certificate file.

    bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

It'll prompt for filename and password which you gave above.

Now we need to get **elastic-certificates.p12** file outside our container.

    docker cp elasticsearch:/usr/share/elasticsearch/elastic-certificates.p12 .

#### Write elasticsearch settings

Create a file **elasticsearch.yml**.

    discovery.type: single-node
    network.host: 0.0.0.0
    xpack.security.enabled: true
    xpack.security.transport.ssl.enabled: true
    xpack.security.transport.ssl.keystore.type: PKCS12
    xpack.security.transport.ssl.verification_mode: certificate
    xpack.security.transport.ssl.keystore.path: elastic-certificates.p12
    xpack.security.transport.ssl.truststore.path: elastic-certificates.p12
    xpack.security.transport.ssl.truststore.type: PKCS12

#### Write logstash config

Create a file **logstash-auth.conf**.

    input {
        tcp {
            port => 9600
            codec => json
        }
    }
    output {
        elasticsearch {
            hosts => ["elasticsearch:9200"]
            user => "elastic"
            password => "<ELASTIC_PASSWORD>"
        }
        stdout { codec => rubydebug }
    }

##### Explanation of logstash config

- `input` allows to receive incoming connection. Here, we're receiving **TCP** input connection on port **9600** having **codec json**.
- `output` allows logstash to output outgoing connection. In our case output is being transmitted to elasticsearch on its default address **elasticsearch:9200**. Because elasticsearch will be protected with a password, we need to set **user** and **password** for it. **rubydebug** outputs event data using the ruby "awesome_print" library. This is the default codec for stdout.

#### Write kibana settings

Create a file **kibana.yml**. Note: the `kibana` user is deprecated — use `kibana_system` instead [4].

    server.name: kibana
    server.host: 0.0.0.0
    server.port: 5601
    elasticsearch.hosts: ["http://elasticsearch:9200"]
    elasticsearch.username: kibana_system
    elasticsearch.password: <KIBANA_SYSTEM_PASSWORD>

#### Write docker-compose-elk-auth file

Create a **docker-compose-elk-auth.yml** file for setup your docker-compose stack. Here is our directory structure

> ELK Dockerization
>> docker-compose-elk-auth.yml
>> elasticsearch.yml
>> logstash-auth.conf
>> kibana.yml
>> elastic-certificates.p12

##### Explanation of docker-compose-elk-auth file

- `elasticsearch`, `logstash` and `kibana` are service names, you can change the name whatever you want.
- `container_name` is a name for your container.
- `ELASTIC_PASSWORD` environment variable sets the bootstrap password for the `elastic` superuser [1].
- `volumes` to define a file/folder that you want to use for the container.
- `./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro` mounts our custom elasticsearch settings as read only.
- `./elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12:ro` mounts the SSL certificate as read only.
- `./elasticsearch-data:/usr/share/elasticsearch/data` persists data to a local folder.
- `./logstash-auth.conf:/usr/share/logstash/pipeline/logstash.conf:ro` mounts the pipeline config into Logstash's pipeline directory [3].
- `./kibana.yml:/usr/share/kibana/config/kibana.yml:ro` mounts our custom kibana settings as read only.
- `ports` is to define which ports you want to expose and define, in this case we're using default ports **9200, 9600, 5601** and exposing it to host machine.
- `ntw-elk` is used as a bridge to connect all the three services in a network.
- `healthcheck` verifies Elasticsearch is ready (with authentication) before dependent services start.
- `depends_on` with `condition: service_healthy` ensures services start in the correct order [7].

#### Installing the SSL certificate on Elasticsearch and enabling TLS in config

Execute elasticsearch docker container by using command-

    docker compose -f docker-compose-elk-auth.yml up

#### Generate passwords and configure the credentials

Login into elasticsearch container-

    docker exec -it elasticsearch bash

Generate passwords for all the built-in users either automatically or interactively.

- automatically

        bin/elasticsearch-setup-passwords auto

- interactively

        bin/elasticsearch-setup-passwords interactive

Note them down and keep them somewhere safe. Exit the container by pressing `CTRL+D`. Bring the container down by `docker compose -f docker-compose-elk-auth.yml down`.

#### Update passwords in config files

- Update **elastic** user's password in **logstash-auth.conf** file.
- Update **kibana_system** user's password in **kibana.yml** file.

#### Execution and access

Now run the docker-compose file with `docker compose -f docker-compose-elk-auth.yml up` or `docker compose -f docker-compose-elk-auth.yml up -d` to run containers in the background [6].

Type `docker ps` to see our running containers.

Once the services are up and running, open your browser and open the url **http://localhost:9200/**. That's elasticsearch.
Also, if you navigate to **http://localhost:5601/** you should see the Kibana console. Login to it using **elastic** user and its password.
To make your application send data to logstash, use **localhost:9600** TCP address.

---

#### References

1. [Install Elasticsearch with Docker](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-with-docker)
2. [Configuring security in Elasticsearch](https://www.elastic.co/docs/deploy-manage/security)
3. [Configuring Logstash for Docker](https://www.elastic.co/docs/reference/logstash/docker-config)
4. [Built-in users — `kibana` user deprecated in favor of `kibana_system`](https://www.elastic.co/docs/deploy-manage/users-roles/cluster-or-deployment-auth/built-in-users)
5. [Running Kibana on Docker](https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-kibana-with-docker)
6. [Migrate to Docker Compose V2 (`docker compose` replaces `docker-compose`)](https://docs.docker.com/compose/releases/migrate/)
7. [Docker Compose Specification](https://docs.docker.com/reference/compose-file/)
