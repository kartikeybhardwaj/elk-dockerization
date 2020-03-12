# Elasticsearch, Logstash, Kibana (ELK) Dockerization

This guide is made for ELK v7.6.0.

Logstash receives data, transmits it to Elasticsearch which can then be viewed in Kibana.

1. [Without X-Pack security enabled](https://github.com/kartikeybhardwaj/elk-dockerization#1-without-x-pack-security-enabled)
2. [With X-Pack security enabled](https://github.com/kartikeybhardwaj/elk-dockerization#2-with-x-pack-security-enabled)

## 1. Without X-Pack security enabled

#### Plan of action

1. Pull ELK images
2. Write logstash config
3. Write docker-compose file
4. Execution and access

#### Pull ELK images

    docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.0
    docker pull docker.elastic.co/logstash/logstash:7.6.0
    docker pull docker.elastic.co/kibana/kibana:7.6.0

Show pulled image by executing

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

- `version` is a version of docker-compose file format, you can change to the latest version.
- `elasticsearch`, `logstash` and `kibana` are service names, you can change the name whatever you want.
- `container_name` is a name for your container.
- `volumes` to define a file/folder that you want to use for the container.
- `./elasticsearch-data:/usr/share/elasticsearch/data` means you want to set data on container persist on you local folder named **elasticsearch-data**. **/usr/share/elasticsearch/data** is a folder that is already created inside the elasticsearch container.
- `./logstash.conf:/etc/logstash/conf.d/logstash.conf:ro` means you want to copy `logstash.conf` to `/etc/logstash/conf.d/` as a read only file. **/etc/logstash/conf.d** is a folder that is already created inside the logstash container used for setting up its configuration, so we copy our config file to that folder.
- `command: logstash -f /etc/logstash/conf.d/logstash.conf` means logstash will use config file which we placed in.
- `ports` is to define which ports you want to expose and define, in this case we're using default ports **9200, 9600, 9601, 5601** and exposing it to host machine.
- `ntw-elk` is used as a bridge to connect all the three services in a network.

#### Execution and access

Now run the docker-compose file with `docker-compose up` or `docker-compose up -d` to run containers in the background.

Type `docker ps` to see our running containers.

Once the services are up and running, open your browser and open the url http://localhost:9200/. That's elasticsearch.
Also, if you navigate to http://localhost:5601/ you should see the Kibana console.
To make your application send data to logstash, use localhost:9600 TCP address.

## 2. With X-Pack security enabled

In order to enable **X-Pack security**, we will need to customize our elasticsearch, logstash and kibana services. Elasticsearch settings can be customized via **elasticsearch.yml** file, Logstash settings can be customized via **logstash.yml** file and Kibana settings can be customized via **kibana.yml** file.

1. Pull ELK images
2. Create SSL certificate for Elasticsearch and enable SSL
3. Write elasticseach settings
4. Write logstash config
5. Write logstash settings
6. Write kibana settings
7. Write docker-compose-elk-auth file
8. Installing the SSL certificate on Elasticsearch and enabling TLS in config
9. Generate passwords and configure the credentials
10. Update passwords config and setting files
11. Execution and access

#### Pull ELK images

    docker pull docker.elastic.co/elasticsearch/elasticsearch:7.6.0
    docker pull docker.elastic.co/logstash/logstash:7.6.0
    docker pull docker.elastic.co/kibana/kibana:7.6.0

Show pulled image by executing

    docker images

#### Create SSL certificate for Elasticsearch and enable SSL

If you already have an SSL certificate you can use that, if not, then follow the steps below to create it.

##### Write docker-compose-cert file

    version: "3.7"

    services:

    elasticsearch:
        image: docker.elastic.co/elasticsearch/elasticsearch:7.6.0
        container_name: elasticsearch
        volumes:
        - ./elasticsearch-data:/usr/share/elasticsearch/data

##### Explanation of docker-compose-cert file

- `version` is a version of docker-compose file format, you can change to the latest version.
- `elasticsearch` is a service name, you can change the name whatever you want.
- `container_name` is a name for your container.
- `volumes` to define a file/folder that you want to use for the container.
- `./elasticsearch-data:/usr/share/elasticsearch/data` means you want to set data on container persist on you local folder named **elasticsearch-data**. **/usr/share/elasticsearch/data** is a folder that is already created inside the elasticsearch container.

##### Execute **docker-compose-cert.yml** file

    docker-compose -f docker-compose-cert.yml up

Now we will need to get inside our docker container running elasticsearch service.

    docker-compose exec -it elasticsearch bash

Create certifiacte authority. Execute the below command in the bash of container to create **elastic-stack-ca.p12** file.

    bin/elasticsearch-certutil ca

You can set or ignore certifiacte authority filename and its password.

Create **elastic-certificates.p12** SSL certificate file.

    bin/elasticsearch-certutil cert --ca elastic-stack-ca.p12

It'll prompt for filename and password which you gave above.

Now we need to get **elastic-certificates.p12** file outide our container.

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

Create a file **logstash.conf**.

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
            password => "<elastic_USERS_PASSWORD_WILL_COME_HERE>"
        }
        stdout { codec => rubydebug }
    }

##### Explanation of logstash config

- `input` allows to receive incoming connection. Here, we're receiving **TCP** input connection on port **9600** having **codec json**.
- `output` allows logstash to output outgoing connection. In our case output is being transmitted to elasticsearch on its default address **elasticsearch:9200**. Because elasticsearch will be protected with a password, we need to set **user** and **password** for it. **rubydebug** outputs event data using the ruby "awesome_print" library. This is the default codec for stdout.

#### Write logstash settings

Create a file **logstash.yml**.

    xpack.monitoring.elasticsearch.username: logstash_system
    xpack.monitoring.elasticsearch.password: <logstash_system_USERS_PASSWORD_WILL_COME_HERE>

#### Write kibana settings

Create a file **kibana.yml**.

    server.name: kibana
    server.host: 0.0.0.0
    server.port: 5601
    elasticsearch.hosts: ["http://elasticsearch:9200"]
    elasticsearch.username: kibana
    elasticsearch.password: <kibana_USERS_PASSWORD_WILL_COME_HERE>

#### Write docker-compose-elk-auth file

Create a **docker-compose-elk-auth.yml** file for setup your docker-compose stack. Here is our directory structure

> ELK Dockerization  
>> docker-compose-elk-auth.yml  
>> elasticsearch.yml  
>> logstash.conf  
>> logstash.yml  
>> kibana.yml  
>> elastic-certificates.p12

##### Explanation of docker-compose-elk-auth file

- `version` is a version of docker-compose file format, you can change to the latest version.
- `elasticsearch`, `logstash` and `kibana` are service names, you can change the name whatever you want.
- `container_name` is a name for your container.
- `volumes` to define a file/folder that you want to use for the container.
- `./elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro` means you want to copy **elasticsearch.yml** to **/usr/share/elasticsearch/config/** as a **read only** file. **/usr/share/elasticsearch/config** is a folder that is already created inside the elasticsearch container used for as its settings, so we copy our settings file to it.
- `./elastic-certificates.p12:/usr/share/elasticsearch/config/elastic-certificates.p12:ro` means you want to copy **elastic-certificates.p12** to **/usr/share/elasticsearch/config/** as a **read only** file. **/usr/share/elasticsearch/config** is a folder that is already created inside the elasticsearch container used for as its settings, so we copy our SSL certificate file to it.
- `./elasticsearch-data:/usr/share/elasticsearch/data` means you want to set data on container persist on you local folder named **elasticsearch-data**. **/usr/share/elasticsearch/data** is a folder that is already created inside the elasticsearch container.
- `./logstash.conf:/etc/logstash/conf.d/logstash.conf:ro` means you want to copy `logstash.conf` to `/etc/logstash/conf.d/` as a read only file. **/etc/logstash/conf.d** is a folder that is already created inside the logstash container used for setting up its configuration, so we copy our config file to that folder.
- `./logstash.yml:/usr/share/logstash/config/logstash.yml:ro` means you want to copy `logstash.yml` to `/usr/share/logstash/config/` as a read only file. **/usr/share/logstash/config** is a folder that is already created inside the logstash container used for its settings, so we copy our settings file to it.
- `./kibana.yml:/usr/share/kibana/config/kibana.yml:ro` means you want to copy `kibana.yml` to `/usr/share/kibana/config/` as a read only file. **/usr/share/kibana/config** is a folder that is already created inside the kibana container used for its settings, so we copy our settings file to it.
- `command: logstash -f /etc/logstash/conf.d/logstash.conf` means logstash will use config file which we placed in.
- `ports` is to define which ports you want to expose and define, in this case we're using default ports **9200, 9600, 9601, 5601** and exposing it to host machine.
- `ntw-elk` is used as a bridge to connect all the three services in a network.

#### Installing the SSL certificate on Elasticsearch and enabling TLS in config

Execute elasticsearch docker container by using command-

    docker-compose -f docker-compose-elk-auth.yml up

#### Generate passwords and configure the credentials

Login into elasticsearch container-

    docker exec -it elasticsearch bash

Generate passwords for all the built-in users either automatically or interactively.

- automatically

    bin/elasticsearch-setup-passwords auto

- interactively

    bin/elasticsearch-setup-passwords interactive

Note them down and keep them somewhere safe. Exit the container by pressing `CTRL+D`. Bring the container down by `docker-compose -f docker-compose-elk-auth.yml down`.

#### Update passwords config and setting files

- Update **elastic** user's password in **logstash.conf** file.
- Update **logstash_system** user's password in **logstash.yml** file.
- Update **kibana** user's password in **kibana.yml** file.

#### Execution and access

Now run the docker-compose file with `docker-compose -f docker-compose-elk-auth up` or `docker-compose -f docker-compose-elk-auth up -d` to run containers in the background.

Type `docker ps` to see our running containers.

Once the services are up and running, open your browser and open the url **http://localhost:9200/**. That's elasticsearch.
Also, if you navigate to **http://localhost:5601/** you should see the Kibana console. Login to it using **elastic** user and its password.
To make your application send data to logstash, use **localhost:9600** TCP address.
