# Kafka Connect Mirror

The purpose of this repo is to setup a local Kafka instance that mirrors/replicates from a Heroku Kafka cluster using Kafka Connect. It uses the [MirrorSourceConnector](https://github.com/apache/kafka/blob/trunk/connect/mirror/src/main/java/org/apache/kafka/connect/mirror/MirrorSourceConnector.java) provided by Kafka Connect.

Most of this work was adapted from the following blog post: https://medium.com/@ankochem/tutorial-mirror-maker-2-0-basic-configuration-d0d1cbca695d

## Pre-Reqs
* [Docker](https://www.docker.com/products/docker-desktop/) (including Docker Compose)
* [Java JDK](https://www.oracle.com/java/technologies/downloads/)
* [OpenSSL](https://www.openssl.org/)

## Docker Containers

Using Docker Compose, the following containers are setup in the `docker-compose.yml`:

* Apache Kafka
* Zookeeper
* Kafka Connect

## Setup

### Certificates
Kafka Connect requires certificates in the JKS format but Heroku provides them in PEM format, so you will first need to convert them.

1. Retrieve the necessary certs from Heroku config:
    ```
    export APP_NAME=<YOUR_APP_NAME>
    heroku config:get KAFKA_CLIENT_CERT -a $APP_NAME > certs/cert.pem
    heroku config:get KAFKA_CLIENT_CERT_KEY -a $APP_NAME > certs/cert.pem
    heroku config:get KAFKA_TRUSTED_CERT -a $APP_NAME > certs/trusted_cert.pem
    ```

2. Generate a pkcs12 file from the cert.pem and key.pem. Below commands assume you are in the `certs` directory.

    ```
    openssl pkcs12 -export -out cert.pkcs12 -in cert.pem -inkey key.pem
    ```
    Update the property `source.cluster.ssl.key.password` in the `connectors.json` file with the password you just entered.

3. Create a keystore in the JKS format:
    ```
    keytool -v -importkeystore -srckeystore cert.pkcs12 -srcstoretype PKCS12 -destkeystore keystore.jks -deststoretype JKS
    ```
    You will need to enter the password from the previous step when prompted for the source keystore password. You will also enter a new password for the destination keystore. This new password should be used for the `source.cluster.ssl.keystore.password` property in the `connectors.json`.

4. Convert the CA cert into DER format:
    ```
    openssl x509 -in trusted_cert.pem -out cert.der -outform der
    ```

5. Import the CA cert into a truststore:
    ```
    keytool -importcert -alias herokukafka -file cert.der -keystore truststore.jks
    ```

    Enter a password when prompted and also type 'yes' when asked "Trusted this certificate?". The password should be used for the `source.cluster.ssl.truststore.password` property in the `connectors.json`.

The only files you really need now are `truststore.jks` and `keystore.jks`, so if you choose you can now remove the other files in the `certs` directory.

### Connectors.json

1. Update the `topics` property with the name of the topic(s) you wish to sync from your Heroku Kafka cluster.

2. Update the `source.cluster.bootstrap.servers` property with the output of the below command:
    ```
    heroku config:get KAFKA_URL -a $APP_NAME | sed -r 's/kafka\+ssl:\/\///g'
    ```

## Running the sync

1. Start the Docker containers.
   ```
    docker-compose up -d
    ```
2. Once the containers have fully started, you can run the following to create the connector using the Kafka Connect REST API:
    ```
    cat connectors.json | curl -X POST -H 'Content-Type: application/json' localhost:28082/connectors --data-binary @-
    ```
3. Verify the connector was successfully created.
    ```
    curl localhost:28082/connectors
    ```
    If there is an error you can delete and recreate the connector. To delete a connector run:
    ```
    curl -X DELETE localhost:28082/connectors/test_mirror
    ```

4. Verify the topic(s) have been created in your local Kafka instance.
    ```
    docker exec -it broker /bin/bash
    kafka-topics --list --bootstrap-server broker:29092
    ```
    The names of the topics will be prefixed with "source". This can be controlled by updating the attribute `source.cluster.alias` in the `connectors.json`.
5. In a new terminal, start sending some messages to your Heroku Kafka cluster.
    ```
    export SOURCE_TOPIC_NAME=<NAME OF YOUR TOPIC IN HEROKU KAFKA>
    while [[ true ]]; do heroku kafka:topics:write -a $APP_NAME $SOURCE_TOPIC_NAME $RANDOM; sleep 5; done
    ```
6. Back in the connect container bash, start listening on your local Kafka topic:
    ```
    export DEST_TOPIC_NAME=<NAME OF YOUR TOPIC IN YOUR LOCAL KAFKA>
    kafka-console-consumer --topic $DEST_TOPIC_NAME --bootstrap-server broker:29092
    ```
    You should start seeing random numbers appear as the topic mirrors the Heroku Kafka topic.

    Note: your local topic will be prepended with the value in the `source.cluster.alias` property.

## Tearing Down

To stop all the Docker containers run:

```
docker-compose down
```




