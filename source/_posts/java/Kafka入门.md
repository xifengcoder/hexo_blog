---
title: Kafka入门
urlname: Kafka_quickstart
date: 2023-12-23 13:59:06
tags:
categories:
description:
---

### Kafka入门

启动ZooKeeper

```shell
# Start the ZooKeeper service
$ bin/zookeeper-server-start.sh config/zookeeper.properties
```

打开另一个控制台，启动Kafka

```shell
# Start the Kafka broker service
$ bin/kafka-server-start.sh config/server.properties
```

创建Topic

```shell
$ bin/kafka-topics.sh --bootstrap-server localhost:9092 --create --topic topic-create --partitions 4 --replication-factor 1

# output
Created topic topic-create.
```

查看Topic信息：

```shell
$bin/kafka-topics.sh --describe --topic topic-create --bootstrap-server localhost:9092
Topic: topic-create	TopicId: pLImIYImQ92Ii4KPg4-YxQ	PartitionCount: 4	ReplicationFactor: 1	Configs: 
	Topic: topic-create	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
	Topic: topic-create	Partition: 1	Leader: 0	Replicas: 0	Isr: 0
	Topic: topic-create	Partition: 2	Leader: 0	Replicas: 0	Isr: 0
	Topic: topic-create	Partition: 3	Leader: 0	Replicas: 0	Isr: 0
```

往Kafka指定的Topic写数据：

```shell
$ bin/kafka-console-producer.sh --topic topic-create --bootstrap-server localhost:9092
>hello, This is message 1
>This is message2
>
```

从Kafka中读取消息：

```shell
$ bin/kafka-console-consumer.sh --topic topic-create --from-beginning --bootstrap-server localhost:9092
hello, This is message 1
This is message2
```

