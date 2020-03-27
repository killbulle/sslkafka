---
title: "Tuning and trouble Shooting"
author: Mgo
date: March 27, 2020
output: pdf_document
---

##System level tuning for kafka
* On your server in the etc/sysctl.conf file
◦ vm.swappiness=1 (Default: 60)
◦ vm.dirty_ratio=80 (Default: 20)
◦ vm.dirty_background_ratio=5 (Default: 10)

*TODO With more information we can tune the network 
https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/kafka_system_level_broker_tuning.html
 
 Saving etc/sysctl.conf with sysctl -p


### Additional reference for performance

* [Tuning Kafka for low latency guaranteed messaging -- Jiangjie (Becket) Qin (LinkedIn),](https://www.youtube.com/watch?v=oQe7PpDDdzA)

* [Let’s Load test, Kafka!] (https://medium.com/selectstarfromweb/lets-load-test-kafka-f90b71758afb)


#Performance Troobleshooting Tools
- Add  sysstat  package for your server
-- https://www.thegeekstuff.com/2011/03/sar-examples/
-- https://access.redhat.com/solutions/3511971
* I recomand to move the samplint around 30 second and leave retention around 10 days mad



