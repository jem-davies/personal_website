---
author: "Jem Davies"
title: "Kafka with TLS/SSL in Docker"
date: "2026-01-13"
description: "Setting up Kafka with TLS/SSL in Docker for integration testing"
tags: ["Kafka", "Docker", "tls", "Bento"]
ShowToc: true
ShowBreadCrumbs: false
---

---

I am trying to work in more Test-Driven-Development style - which effectively 
means writing the tests for some functionality before writing the implementation.

When the functionality requires connecting to another service, I think that it 
makes sense to start with an integration test. Docker is a great tool for spinning
up an ephemeral service to use in the test. 

Recently I found myself wanting to implement Kafka <--> Bento (stream processor) 
connection using OAUTHBEARER. To use OAUTHBEARER effectively you should also
set-up that connection with tls/ssl.

Anyway, this is more of a brain-dump on how to setup set-up a kafka instance in 
a docker container that will communicate with the host using tls/ssl.

---

### Use `openssl` to generate CA, Certs & Keys

Using `openssl` we can create: 

 - A self-signed certificate (CA)
 - A server-key & a server-cert signed by our CA
 - A client-key & a client-cert signed by our CA

<p></p>

Create new Certificate Authority (`ca-cert.pem` & `ca-key.pem`)

```sh
openssl req -x509 -newkey rsa:4096 -days 365 -nodes -keyout ca-key.pem \
-out ca-cert.pem -subj "/CN=Local Dev CA"
```

<p></p>

Create a certificate signing request (CSR) & private server key (`server-key.pem` & `server-req.pem`)

```sh
openssl req -newkey rsa:4096 -nodes -keyout server-key.pem -out server-req.pem \
-subj "/CN=localhost" -addext "subjectAltName = DNS:localhost,IP:127.0.0.1"
```

<p></p>

Use our CA to sign our CSR and create our server's signed cert (`server-cert.pem`)

```sh
openssl x509 -req -in server-req.pem -CA ca-cert.pem -CAkey ca-key.pem \
-CAcreateserial -out server-cert.pem -days 365 -copy_extensions copy
```

<p></p>

Do the same to create the client's key & cert (`client-key.pem` & `client-cert.pem`)

```sh
openssl req -newkey rsa:4096 -nodes -keyout client-key.pem -out client-req.pem \
-subj "/CN=localhost" -addext "subjectAltName = DNS:localhost,IP:127.0.0.1"
```

```sh
openssl x509 -req -in client-req.pem -CA ca-cert.pem -CAkey ca-key.pem \
-CAcreateserial -out client-cert.pem -days 365 -copy_extensions copy
```

### Use `keytool` to create a Java 'truststore' & 'keystore'

So in the Java world they have somethings called truststore & keystore, which I 
understand is Java's way of storing things like ssl certs & keys. Originally 
Kafka only supported this way of providing certs & keys - but it does now support
pem files... Although I struggled to find sufficient documentation on how to do 
that.

To create the truststore & keystore you will need `keytool`, this is 
installed with Java but if you don't have it & don't want it - you could use the 
below docker image in an interactive shell which does have it installed.

First we need to export into an intermediary format (`server.p12`):

```sh
# You will be asked to provide a password 'password' will do for an int. test!
openssl pkcs12 -export -inkey server-key.pem -in server-cert.pem -certfile \
ca-cert.pem -out server.p12 -name kafka-server
```

Create the keystore (`kafka.keystore.jks`):

```sh
keytool -importkeystore -deststorepass changeit -destkeypass changeit \
-destkeystore kafka.keystore.jks -srckeystore server.p12 -srcstoretype PKCS12 \
-srcstorepass password -alias kafka-server
```

Create the truststore (`kafka.truststore.jks`):

```sh
keytool -import -trustcacerts -alias dev-ca -file ca-cert.pem -keystore \
kafka.truststore.jks -storepass changeit -noprompt
```

### Create a docker-compose 

Create a new directory 'certs' with the keystore & truststore - which will be 
mounted into the container: 

```sh
mkdir ./certs
cp ./kafka.truststore.jks ./kafka.keystore.jks ./certs
```

<p></p>

Copy the below yaml into a `docker-compose.yaml` file:

```yaml
services:
  kafka:
    image: bitnami/kafka:latest
    container_name: kafka
    ports:
      - 9092:9092
      - 9093:9093
    volumes:
      - ./certs:/bitnami/kafka/config/certs:ro
    environment:
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: broker,controller
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,SSL://:9093,CONTROLLER://:9094
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092,SSL://localhost:9093
      KAFKA_CFG_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 1@localhost:9094
      KAFKA_CFG_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CFG_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_CFG_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_CFG_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_CFG_NUM_PARTITIONS: 3
      KAFKA_CFG_SSL_KEYSTORE_LOCATION: /bitnami/kafka/config/certs/kafka.keystore.jks
      KAFKA_CFG_SSL_KEYSTORE_PASSWORD: changeit
      KAFKA_CFG_SSL_KEY_PASSWORD: changeit
      KAFKA_CFG_SSL_TRUSTSTORE_LOCATION: /bitnami/kafka/config/certs/kafka.truststore.jks
      KAFKA_CFG_SSL_TRUSTSTORE_PASSWORD: changeit
      KAFKA_CFG_SSL_CLIENT_AUTH: required
```

Start the container with: 

```sh
docker compose up -d 
```

### Create a test-topic & create an event

Let's open up an interactive shell to that container, create a topic & write an
event to it:

```sh
docker exec -it kafka bin/bash
```

<p></p>

kafka-console-producer.sh will take input from stdin and send it to the topic:

```sh
cd ./opt/bitnami/kafka/bin/
./kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092
./kafka-console-producer.sh --topic test-topic --bootstrap-server localhost:9092
>> Hello from Kafka with tls/ssl!
```

### Test with Bento 

Create a [Bento](https://warpstreamlabs.github.io/bento/) config to connect to 
kafka on the SSL advertised listener with mTLS:

```yaml
input:
  kafka:
    addresses: 
      - localhost:9093
    topics: 
      - test-topic
    consumer_group: bento
    tls:
      enabled: true
      root_cas_file: "./ca-cert.pem"
      client_certs:
        - cert_file: ./client-cert.pem
          key_file: ./client-key.pem

output:
  stdout: {}
```

```sh
bento -c ./config.yaml
```

You should see the events from the test-topic in stdout!

### Conclusion 

Again this was more of a brain-dump for myself because I had quite the time of 
setting this up... Originally, I was wanting to stick with `.pem` format for the 
keys & certs but struggled to find documentation on how to do that.

I switched to using the `bitnami/kafka` image because the kafka config, didn't
work with `apache/kafka`. I was getting a weird bash interpolation error in the 
docker logs... 

It seems that the README for the bitnami image has more on how to set this up
with pem files: https://hub.docker.com/r/bitnami/kafka#security, which I didn't 
see before deciding to give up and switch to Java's truststore & keystore.

I mentioned in the beginning that this was for an integration test, the next step
would be to convert the docker-compose yaml into some code using a docker SDK like: 

 - https://github.com/ory/dockertest (Go)
 - https://github.com/testcontainers/testcontainers-python (Python)

And automating the creation of the topic & test events.

This set-up wouldn't be suitable in prod for at least a few reasons:

 - The private key files & certs aren't encrypted, this was disabled with the 
 `nodes` option in `openssl`
 - The kafka image is still using 'PLAINTEXT' for some of the listeners 
 - The passwords are defaults 'changeit' & 'password'

I think that looking ahead at getting a full setup with a SASL enabled kafka 
broker with OAUTHBEARER will be even more difficult, however it seems that there
is a project: [strimzi](https://strimzi.io/) that might help.