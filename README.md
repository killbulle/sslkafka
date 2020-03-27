---
title: SSL configuration
author: Mgo
date: March 27, 2020
fontsize: 11pt
geometry: margin=1in
documentclass: report
output:
   pdf_document:
   toc: true 
   toc_depth: 2      
---

#Création config SSL + SCRAM

## Various Links 
* [Doc apache](https://kafka.apache.org/090/documentation.html#security_overview)

* [Doc Confluent](https://docs.confluent.io/current/security/security_tutorial.html#security-tutorial)

* Vastly Inspired form [Doc d'install vertica pour la mise en place SSL](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/KafkaIntegrationGuide/TLS-SSL/KafkaTLS-SSLExamplePart3ConfigureKafka.htm)

* [Documentation procedure SASL](https://medium.com/egen/securing-kafka-cluster-using-sasl-acl-and-ssl-dec15b439f9d)


* [Firewall constraint](https://stackoverflow.com/questions/38531054/kafka-and-firewall-rules)


## SSL root Certificate création

####  Step 1 : Root Certificate création

```	
openssl req -new -x509 -keyout ca-key -out root.crt -days 365
```	
	    
#### Step  2 : Créate a __truststore__ for one spécific broker
```
keytool -keystore kafka.truststore.jks -alias CARoot -import -file root.crt
```

> _A similar procédure will be need for the client in another step_

#### Step 3 : Create a keystore with the good SAN(Subject alternative name)
> Here we add a SAN(Subject alternative name) to allow SSL endpoint identification
This identification can be disable in the broker server.properties
__ssl.endpoint.identification.algorithm=__ 
_Note here the value for desactivating is blank_

* For Kafka broker named kafka01
  
  >manual mode
```
keytool -keystore kafka01.keystore.jks -alias localhost  -validity 365 \n
-genkey -keyalg RSA -ext SAN=DNS:kafka01.mycompany.com
```
> Without user prompts, pass command line arguments (do not use if you do the manual mode)
```
keytool -keystore kafka.server.keystore.jks -alias localhost -keyalg RSA \
-validity {validity}  -genkey -storepass {keystore-pass} -keypass {key-pass}\
 -dname {distinguished-name}  -ext SAN=DNS:{hostname}
```

#### Step 4 : Export  broker certificate for signing with CA ROOT
```
keytool -keystore kafka01.keystore.jks -alias localhost \
  -certreq -file kafka01.unsigned.crt
```
#### Step 5 : Sign the broker certificate
```
openssl x509 -req -CA root.crt -CAkey root.key -in kafka01.unsigned.crt \
-out kafka01.signed.crt -days 365 -CAcreateserial
```
#### Step 6 :import the root cetificate in the broker keystore
```
$ keytool -keystore kafka01.keystore.jks -alias CARoot -import -file root.crt
```
#### Step 7 import signed cerficate CAroot in the keystore
```
keytool -keystore kafka01.keystore.jks -alias localhost  \
 -import -file kafka01.signed.crt
```
#### Step 8: Copy truststore et keystore vers les serveurs kafka
>use your favorite tools scp or another

#### Step 9:  Move the file in the good kafka config directory
>for example
```
cd /opt/kafka/config/
/opt/kafka/config# cp /root/kafka01.keystore.jks /root/kafka.truststore.jks .
```
#### Step 10: Check the right of the file
	>Check the files trustore and keystore acces right

#### Step 11: Configuration du server kafka

In the  server.properties file
```
security.protocol=SSL
listeners=SSL://:9093,SASL_SSL://:9094
security.inter.broker.protocol=SSL
ssl.truststore.location=/opt/kafka/config/kafka.truststore.jks
ssl.truststore.password=trustore_password
ssl.keystore.location=/opt/kafka/config/client.keystore.jks
ssl.keystore.password=keystore_password
ssl.key.password=key_password
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
#ssl.client.auth=required  
```
>__Note that ssl.client.auth=require__ is commented
If enabled it require client use a certificate for a two way auth
Note also that if client use SCRAM it will ignore it

## Repeat  step 2 - 11 For each broker



#### Step 12: Kafka server restart

Testing tls connection
```
openssl s_client -debug -connect broker_host_name:9093 -tls1
```
* A tools for rolling restart
[Yelp tools for rolling update](https://github.com/Yelp/kafka-utils) 

#### Step 13 A Configuration without 2-ways authentification
* import the root certificate in a client truststore
```
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file  -file root.crt
```

* And on the client part we  create a config file

```
security.protocol = SSL
ssl.truststore.location = <pathtostore>/kafka.client.truststore.jks
ssl.truststore.password = <password>
```

#### Step 13 B (Optional and replace Step 13 A) Configuration with 2-ways authentification
__WARNING__  __ssl.client.auth=required is required in the server properties__

* For 2-ways authentification we need a keystore for the client

1. Create the client keystore:
```
keytool -keystore client.keystore.jks -alias localhost -validity 365 -genkey -keyalg RSA \
 -ext SAN=DNS:fqdn_of_client_system
```
"first and last name?" is the FQDN of the system you will use to run the producer and/or consumer. 

2. Export the client certificate so it can be signed: 
```
keytool -keystore client.keystore.jks -alias localhost -certreq -file client.unsigned.cert
```

3. Sign the client certificate with the root CA:
```
openssl x509 -req -CA root.crt -CAkey root.key -in client.unsigned.cert \ 
-out client.signed.cert    -days 365 -CAcreateserial 
```

4. Add the root CA to keystore:
```
keytool -keystore client.keystore.jks -alias CARoot -import -file root.crt
```
5. Add the signed client certificate to the keystore:
```
keytool -keystore client.keystore.jks -alias localhost -import -file client.signed.cert
```
6. Copy the keystore to a location where you will use it.

 For example, you could choose to copy it to the same directory where you copied the keystore
 for the Kafka broker. If you choose to copy it to some other location, or intend to use some 
 other user to run the command-line clients, be sure to add a copy of the truststore file 
 you created for the brokers. Clients can reuse this truststore file for authenticating 
 the Kafka brokers because the same CA is used to sign all of the certificates.
  Also set the file's ownership and permissions accordingly.


* Create a config file
```
security.protocol=SSL
ssl.truststore.location=/<pathtostore>/kafka.truststore.jks
ssl.truststore.password=<trustore_password>
ssl.keystore.location=/<pathtostore>/client.keystore.jks
ssl.keystore.password=<keystore_password>
ssl.key.password=<key_password>
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
ssl.client.auth=required
```



#### Step 14 testing client
```
~# cd /opt/kafka

/opt/kafka# bin/kafka-console-producer.sh --broker-list kafka01.mycompany.com:9093 \
 --topic test --producer.config config/client.properties
```
>test
>toto
```
/opt/kafka# bin/kafka-console-consumer.sh --bootstrap-server kafaka01.mycompany.com:9093  \
--topic test  --consumer.config config/client.properties --from-beginning
```
>test
>toto




# SCRAM configuration for ACL

[SASL config](https://medium.com/egen/securing-kafka-cluster-using-sasl-acl-and-ssl-dec15b439f9d)

### Step 1 : Create the super user
```
./bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password='admin-secret']'\n
 --entity-type users --entity-name admin
```

#### Step2 : Create the  kafka_server_jaas.conf
```
KafkaServer {
org.apache.kafka.common.security.scram.ScramLoginModule required
username="admin"
password="admin-secret";
};
```
#### Step 3 : Create the kafka config client properties  in /config folder command)
```
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="demo-user" password="secret";
ssl.truststore.location=
<kafka-binary-dir>/config/truststore/kafka.truststore.jks
ssl.truststore.password=password
```

#### Step 4: Configure the server properties
```
listeners=PLAINTEXT://localhost:9092,SASL_PLAINTEXT://localhost:9093,SASL_SSL://localhost:9094
advertised.listeners=PLAINTEXT://localhost:9092,SASL_PLAINTEXT://localhost:9093,SASL_SSL://localhost:9094
security.inter.broker.protocol=SASL_SSL
#ssl.endpoint.identification.algorithm=
ssl.client.auth=required
sasl.mechanism.inter.broker.protocol=SCRAM-SHA-512
sasl.enabled.mechanisms=SCRAM-SHA-512
# Broker security settings
ssl.truststore.location=
<kafka-binary-dir>/config/truststore/kafka.truststore.jks
ssl.truststore.password=password
ssl.keystore.location=
<kafka-binary-dir>/config/keystore/kafka.keystore.jks
ssl.keystore.password=password
ssl.key.password=password
# ACLs
authorizer.class.name=kafka.security.auth.SimpleAclAuthorizer
super.users=User:admin
#zookeeper SASL
zookeeper.set.acl=false
########### SECURITY using SCRAM-SHA-512 and SSL ###################
```

#### Step 5 start the kafka with jaasconfig
```
export KAFKA_OPTS=-Djava.security.auth.login.config=<kafka-binary-dir>/config/kafka_server_jaas.conf
./bin/kafka-server-start.sh ./config/server.properties
```



#### Step 6 : Create standard user
```
./bin/kafka-configs.sh --zookeeper localhost:2181 --alter --add-config 'SCRAM-SHA-512=[password='secret']' --entity-type users --entity-name demouser
```


#### Step 7: Creating Topic without Create Permissions:
```
./bin/kafka-topics.sh --create --bootstrap-server localhost:9094 --command-config ./config/ssl-user-config.properties --replication-factor 1 --partitions 1 --topic demo-topic
```
 > FAIL

#### Step 8: Create permission for topic creation
```
./bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:demouser --operation Create --operation Describe  --topic demo-topic

```
#### Step 9 : Create the topic
```
./bin/kafka-topics.sh --create --bootstrap-server localhost:9094 --command-config ./config/ssl-user-config.properties --replication-factor 1 --partitions 1 --topic demo-topic

```

#### Step 10: Create ssl-producer.properties in config folder
```
bootstrap.servers=localhost:9094
compression.type=none
### SECURITY ######
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="demouser" password="secret";
ssl.truststore.location=
<kafka-binary-dir>/config/truststore/kafka.truststore.jks
ssl.truststore.password=password

```
#### Step 11: Produce without right
```
./bin/kafka-console-producer.sh --broker-list localhost:9094 --topic demo-topic --producer.config config/ssl-producer.properties
```
>FAIL

#### Step 12: Assign produce ACL
```
./bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:demouser --producer --topic demo-topic
```

#### Step 13 create a consumer group configuration
```
bootstrap.servers=localhost:9094
# consumer group id
group.id=demo-consumer-group
### SECURITY ######
security.protocol=SASL_SSL
sasl.mechanism=SCRAM-SHA-512
sasl.jaas.config=org.apache.kafka.common.security.scram.ScramLoginModule required username="demouser" password="secret";
ssl.truststore.location=
<kafka-binary-dir>/config/truststore/kafka.truststore.jks
ssl.truststore.password=password
```


#### Step 14: Fail to consume without ACL
```
./bin/kafka-console-consumer.sh --bootstrap-server localhost:9094 --topic demo-topic --from-beginning --consumer.config config/ssl-consumer.properties
```

#### Step 15: Add ACL consume configuration
```
./bin/kafka-acls.sh --authorizer-properties zookeeper.connect=localhost:2181 --add --allow-principal User:demouser --consumer --topic demo-topic --group demo-consumer-group
```