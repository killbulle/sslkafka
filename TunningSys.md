#System level tuning for kafka
* On your server in the etc/sysctl.conf file
◦ vm.swappiness=1 (Default: 60)
◦ vm.dirty_ratio=80 (Default: 20)
◦ vm.dirty_background_ratio=5 (Default: 10)

TODO With more information we can tune the networ 
https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/kafka_system_level_broker_tuning.html
 
 Saving etc/sysctl.conf with sysctl -p


### Additional reference for performance

* [Tuning Kafka for low latency guaranteed messaging -- Jiangjie (Becket) Qin (LinkedIn),](https://www.youtube.com/watch?v=oQe7PpDDdzA)

* [Let’s Load test, Kafka!] (https://medium.com/selectstarfromweb/lets-load-test-kafka-f90b71758afb)
