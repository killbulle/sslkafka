---
title: "Tuning and trouble Shooting"
author: Mgo
date: March 27, 2020
output: pdf_document
---

# System level tuning for kafka

* On your server in the etc/sysctl.conf file
- vm.swappiness=1 
- vm.dirty_ratio=80
- vm.dirty_background_ratio=5 
  >this one could be less on large server with a lot of ram 

 *Saving etc/sysctl.conf and with sysctl -p

 *TODO With more information we can tune the network 
https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/kafka_system_level_broker_tuning.html
 



### Additional reference for performance

* [Tuning Kafka for low latency guaranteed messaging -- Jiangjie (Becket) Qin (LinkedIn),](https://www.youtube.com/watch?v=oQe7PpDDdzA)

* [Let’s Load test, Kafka!] (https://medium.com/selectstarfromweb/lets-load-test-kafka-f90b71758afb)


#Performance Troobleshooting Tools
- Add  sysstat  package for your server
-- https://www.thegeekstuff.com/2011/03/sar-examples/
-- https://access.redhat.com/solutions/3511971
* I recomand to move the samplint around 30 second and leave retention around 10 days maximun


