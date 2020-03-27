
#Création config SSL + SCRAM

## Various Links 
[Doc apache](https://kafka.apache.org/090/documentation.html#security_overview)
[Doc Confluent](https://docs.confluent.io/current/security/security_tutorial.html#security-tutorial)
[Doc d'install vertica pour la mise en place SSL](https://www.vertica.com/docs/9.2.x/HTML/Content/Authoring/KafkaIntegrationGuide/TLS-SSL/KafkaTLS-SSLExamplePart3ConfigureKafka.htm)
[Documentation procedure SASL](https://medium.com/egen/securing-kafka-cluster-using-sasl-acl-and-ssl-dec15b439f9d)
[Firewall constraint](https://stackoverflow.com/questions/38531054/kafka-and-firewall-rules)


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
```
keytool -keystore kafka01.keystore.jks -alias localhost  -validity 365 -genkey -keyalg RSA -ext SAN=DNS:kafka01.mycompany.com
```
> Without user prompts, pass command line arguments
keytool -keystore kafka.server.keystore.jks -alias localhost -keyalg RSA -validity {validity} -genkey -storepass {keystore-pass} -keypass {key-pass} -dname {distinguished-name} -ext SAN=DNS:{hostname}


#### Step 4 : Export  broker certificate for signing with CA ROOT
```
keytool -keystore kafka01.keystore.jks -alias localhost  -certreq -file kafka01.unsigned.crt
```
#### Step 5 : Sign the broker certificate
```
openssl x509 -req -CA root.crt -CAkey root.key -in kafka01.unsigned.crt -out kafka01.signed.crt -days 365 -CAcreateserial
```
#### Step 6 :import the root cetificate in the broker keystore
```
$ keytool -keystore kafka01.keystore.jks -alias CARoot -import -file root.crt
```
#### Step 7 import signed cerficate CAroot in the keystore
```
keytool -keystore kafka01.keystore.jks -alias localhost   -import -file kafka01.signed.crt
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

#### Step 12: Kafka server restart

Testing tls connection
```
openssl s_client -debug -connect broker_host_name:9093 -tls1
```
* A tools for rolling restart
[Yelp tools for rolling update](https://github.com/Yelp/kafka-utils) 

#### Step 13 - A

* Configuration without 2-ways authentification
```
keytool -keystore kafka.client.truststore.jks -alias CARoot -import -file  -file root.crt
```
* And on the client part we  create a config file

```
security.protocol = SSL
ssl.truststore.location = <pathtostore>/kafka.client.truststore.jks
ssl.truststore.password = <password>
```

Configuration client avec authent forte


WARNING il faut alors activer le ssl.client.auth=required





security.protocol=SSL
ssl.truststore.location=/<pathtostore>/kafka.truststore.jks
ssl.truststore.password=<trustore_password>
ssl.keystore.location=/<pathtostore>/client.keystore.jks
ssl.keystore.password=<keystore_password>
ssl.key.password=<key_password>
ssl.enabled.protocols=TLSv1.2,TLSv1.1,TLSv1
ssl.client.auth=required




#### Etape 14 test client
~# cd /opt/kafka


/opt/kafka# bin/kafka-console-producer.sh --broker-list kafka01.mycompany.com:9093  \                                       --topic test --producer.config config/client.properties


/opt/kafka# bin/kafka-console-consumer.sh --bootstrap-server kafaka01.mycompany.com:9093  --topic test \
                                          --consumer.config config/client.properties --from-beginning

