# Kafka Monitoring

Broker, Producer(java client), Consumer(java client)

- JVM/java 相关指标

- 各自组件的指标，只采集Broker JMX，并不能覆盖完整的指标采集。

## JMX domains

### Common

```
JMImplementation
com.sun.management
java.lang
java.nio
java.util.logging
jdk.management.jfr
```

### Broker

```
kafka
kafka.cluster
kafka.controller
kafka.coordinator.group
kafka.coordinator.transaction
kafka.log
kafka.network
kafka.server
kafka.utils
```

### Producer

```
kafka.producer
```

### Consumer

```
kafka.consumer
```



