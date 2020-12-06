# 参考
版本：kafka-2.6.0

Confluent：[https://www.confluent.io/blog/category/apache-kafka/](https://www.confluent.io/blog/category/apache-kafka/)

# 术语
## Broker
Kafka节点

## Partition
单个Topic可拆分成多个Partition，并分散到不同的Broker中，以提高单Topic的吞吐量

## Replica
单个Partition可存在多个相同内容Replica，每个Replica分散在不同的Broker中，用以实现备份

## Partition Leader
每个Partition只有一个Replica接收客户端请求，其他Replica主动从Leader拉取数据

## ISR
与Leader数据同步的In-sync Replica

## Consumer Group
同一Consumer Group中的消费者共同消费一个Topic中的消息，实现相互间负载均衡、高可用

## Offset
由于Kafka不是传统类型的消息队列，所以需要存储单个Partition中Consumer Group的消费位置

## LSO
单个Replica上末尾已提交事务消息的偏移量（isolation.level=read_committed）

## HW
高水位，消费者可见的末尾消息的偏移量

## LEO
单个Replica上真实末尾消息的偏移量

# ZooKeeper数据结构
* /admin
    * /delete_topics
    * /preferred_replica_election
* /brokers
    * /ids
        * /\<broker.id>
    * /seqid
    * /topics
        * /\<topic>
          * /partitions/\<N>/state
* /cluster/id
* /config/*
* /consumers
* /controller
* /controller_epoch
* /isr_change_notification
* /latest_producer_id_block
* /log_dir_event_notification

## /brokers
### /brokers/ids/\<broker.id>
```json
{"listener_security_protocol_map":{"PLAINTEXT":"PLAINTEXT"},"endpoints":["PLAINTEXT://localhost:9092"],"jmx_port":-1,"port":9092,"host":"localhost","version":4,"timestamp":"1603608218902"}
```

### /brokers/seqid
`broker.id.generation.enable`启用时，用作分配`broker.id`

### /brokers/topics/\<topic>
```json
{"version":2,"partitions":{"0":[0]},"adding_replicas":{},"removing_replicas":{}}
```

### /brokers/topics/\<topic>/partitions/\<N>/state
```json
{"controller_epoch":6,"leader":0,"version":1,"leader_epoch":4,"isr":[0]}
```

## /consumers
用来存储consumer_offsets，已废弃，新版本存储在topic`__consumer_offsets`中

## /controller
```json
{"version":1,"brokerid":0,"timestamp":"1603608219071"}
```

## /isr_change_notification
ISR变动通知，KafkaController监听

## /latest_producer_id_block
用作分配producer id，以实现简单的`exactly once`语义

# 重要配置
参考：[http://kafka.apache.org/documentation/#configuration](http://kafka.apache.org/documentation/#configuration)

## Broker
auto.leader.rebalance.enable

unclean.leader.election.enable

min.insync.replicas

default.replication.factor

group.initial.rebalance.delay.ms

num.partitions

## Poducer
acks

buffer.memory

max.in.flight.requests.per.connection

batch.size

delivery.timeout.ms

linger.ms

max.block.ms

partitioner.class

enable.idempotence

client.id

transactional.id

## Consumer
group.id

group.instance.id

auto.offset.reset

enable.auto.commit

isolation.level

partition.assignment.strategy

auto.commit.interval.ms

max.poll.interval.ms

session.timeout.ms

client.id

# 消息传递语义
## At most once
Producer关闭重试机制

Consumer先commit后消费

## At least once
Producer打开重试机制

Consumer先消费后commit

## Exactly once
### Kafka层面
Producer开启`enable.idempotence`，开启后，每个Producer会被分配一个Producer ID，并且每个分区上的消息Batch都会分配一个sequenceNumber，Broker收到消息后，会和上一个保存消息的sequenceNumber进行比较，当sequenceNumber++时，接受此消息Batch

### Consumer层面
在Producer开启`enable.idempotence`的基础上，Consumer需要将offset和业务存储在一起，才能实现Exactly once语义，例如在一个数据库事务中同时处理业务和存储offset

# 事务
事务的引入需要使用到`TransactionCoordinator`，事务的状态存储在内部topic`__transaction_state`，成功事务的三步状态分别为Ongoing、PrepareCommit、CompleteCommit

Producer首先向`TransactionCoordinator`（Broker）发起事务请求，然后往对应Topic中发送消息，最后向`TransactionCoordinator`发起提交事务请求，`TransactionCoordinator`将事务状态写入`__transaction_state`，同时向各Topic写入已提交的标记

参考：[https://www.confluent.io/blog/transactions-apache-kafka/](https://www.confluent.io/blog/transactions-apache-kafka/)