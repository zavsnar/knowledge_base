Confluent Kafka - Defenitive guide v2.
[E-book](https://assets.confluent.io/m/2849a76e39cda2bd/original/20201119-EB-Kafka_The_Definitive_Guide-Preview-Chapters_1_thru_6.pdf)

# The Raft Consensus Algorithm
Visualization https://raft.github.io/

# Overview
Kafka is a distributed commit log or distributing streaming platform.

Originally Kafka was created at LinkedIn for tracking user activities (clicks, views, etc.).


Message is an array of bytes and optionally can have metadata named _key_ (bute array either). _Key_ ofter used for partitioning messages. Messages can be gathered in batches  which can reduce the cost of network overhead while sending messages individually.
*Broker* is a server within Kafka *Cluster*. The broker can be elected as Controller (Leader) which is responsible for administrative operations, including assigning partitions to brokers and monitoring for broker failures.
A partition is owned by a single broker in the cluster, and that broker is called the _leader_ of the partition. A replicated partition is assigned to additional brokers, called _followers_ of the partition. 
All producers must connect to the leader in order to publish messages, but consumers may fetch from either the leader or one of the followers.

Originally Kafka can work only with a single cluster but for multi-cluster work has been created the solution *MirrorMaker*. MirrorMaker is presented as consumers and producers who transfer messages from one cluster to another.

# Producer
On producing messages to the cluster, a broker answers with an error or _RecordMetadata_ which contains the topic, partition, and the offset of the record within the partition.
## Configuration
_bootstrap.servers_ list of addresses of brokers in the cluster. It is not necessary to list all brokers since the client received all needed addresses on connection. But recommended to mention at least 2 servers in case one server is down on connection.
_key.serializer_ and _value.serializer_ are classes that help to serialize from object to byte array.

We can produce messages in blocking and non-blocking ways. We can manage this behavior by passing poll_timeout_sec and use_background_thread to our lib_kafka library.

_acks_ - defines how many brokers should receive a message. 0 - brokers do not send answers for messages; 1 - response is expected from the leader broker only; -1 (all) - all in-sync brokers should give successful answers. lib_kafka by default uses the most reliable variant.
But this parameter impacts only producer latency whereas end-to-end (producer - consumer) latency will be the same because the message becomes available to consume only when all in-sync replicas receive the message.

_deliver.timeout.ms_ - the max time from putting a message to a batch and a successful answer from the broker. If leader election time is 30 secs Confluent recommends setting the parameter to 120sec.
_linger.ms_ - the amount of time to wait for additional messages before sending the current batch. 

_compression.type_ - by default messages in batch are sent uncompressed but we can set types for example _snappy_ for low impact to CPU. It helps reduce network and storage utilization, which is often a bottleneck when sending messages to Kafka.

_max.in.flight.requests.per.connection_ - how many message batches the producer will send to the server without receiving responses. The default value is 5 but for 1 DC value of 2 shows the same performance.

_enable.idempotence_ - enable _exactly once_ semantics. The producer will add sequence number to messages and brokers will check that sequence and throw the Error DuplicateSequenceException on duplicate detection.
  Apache Kafka preserves the order of messages within a partition. This means that if messages are sent from the producer in a specific order, the broker will write them to a partition in that order and all consumers will read them in that order.
  Setting the _retries_ parameter to nonzero and the _max.in.flight.requests.per.connection_ to more than 1 means we can break the original message order.
  Since we want at least two in-flight requests for performance reasons, and a high number of retries for reliability reasons, the best solution is to set _enable.idempotence=true_. This guarantees message ordering and retries will not introduce duplicates. Enabling _enable.idempotence_ additionally requeres parameter _acks=all_. **Chapter 8**

## Partitioning
 The key is used for partitioning messages. The default partitioner (not RoundRobin) guarantees that messages with the same key will be put in the same partition. But this order may change if we add a new partition. Therefore Confluent recommends creating sufficient number of partitions on the start application.
 We can create our own Partitioner, for example, if we have uneven distribution we would direct messages from dominating keys to the last partition and other messages distribute for other partitions (Math.abs(Utils.murmur2(keyBytes)) % (numPartitions - 1);)

 ## Headers
 Headers provide additional meta-information that can be accepted before message deserialization.

 ## Interceptors
 You can create a class on Java that will do some work on send and on acknowledgment. And it is possible to enable this class by setting producer config with interceptor.classes=com.shapira.examples.interceptors.CountProducer

## Quotas and Throttling
 We can limit traffic from producing, consuming, and requesting messages. Produce and consume quotas limit the rate at which clients can send and receive data, measured in bytes per second. Request quotas limit
the percentage of time the broker spends processing client requests.
 Quotas can be applied to all-clients or a specific client. When the quota is reached broker starts throttle requests (or even mutes the communication channel). That leads to increasing queues on the client buffer.
And can be caused by _TimeoutException_.

# Consumer

## Rebalance
 Rebalance happens when a new consumer joins a group, on consumer timeout of proper leaving a group, and additionally, when consuming group settings are changed.
 Rebalance types:
 * **Eager rebalance**. All consumers stop consuming, give up their ownership of all partitions, rejoin the consumer group, and get a brand-new partition assignment. This leads to stopping consuming messages from all partitions for a while.
 * **Cooperative rebalance**. Reassignment of partitions happens by small sub-sets and in several phases which allow carry-on processing messages from other partitions.

 Consumers maintain ownership of the partitions assigned to them by sending (in a background thread) _heartbeats_ to a Kafka broker designated as the group coordinator (this broker can be different for different consumer groups). 
If a consumer stops sending _heartbeats_ for a period more than __session.timeout.ms__ that consumer is considered dead and the coordinator starts the rebalance. In addition, a consumer can notify the coordinator on its shutdown.
In addition to heartbeats, a consumer can be detected dead by delaying polls. If polling has not been done in _max.poll.interval.ms_ (by default 5 minutes) consumer is dead.

The first consumer to join the group becomes the group leader. The leader receives a list of all consumers in the group from the group coordinator and is responsible for assigning a subset of partitions to each consumer. After deciding
on the partition assignment, the consumer group leader sends the list of assignments to the _GroupCoordinator_, which sends this information to all the consumers.

Alongside with dynamic assignment of partitions to consumers, we can declare **static consumer** by defining __group.instance.id__ for it. In that case, partitions assigned to the static consumer won't be reassigned to another, and rebalance won't be started if the static consumer doesn't send heartbeats. But by achieving __session.timeout.ms__ consumer becomes identified dead and partitions will be reassigned to other consumers. Static consumers are handy for applications with heavy internal state which is expensive to construct and you don't want to start rebalance on consumer restart.

Consumers can subscribe to several topics using regex. When a new topic with a matching name is added the rebalance will happen almost immediately and consumers will start consuming from the new topic. But important to understand that the consumer periodically will request all topic list with all partitions and apply regex on them. Which can lead to significant overhead, and subscribing by regex is a very specific desire.

Consumers have to do _poll_ periodically, otherwise after _max.poll.interval.ms_ consumer will be considered dead. So, you should exclude long-blocking operations in the pool loop.

## Consumer Configuration

_fetch.min.bytes_ - the minimum amount of data that it wants to receive from the broker when fetching records, by default one byte. We can increase the value on low-rate topics to process batch messages instead of single ones. Of course, it results in increasing latency. 
_fetch.max.wait.ms_ -  how long to wait before fetching. In couple with _fetch.min.bytes_ messages will be sent whichever condition happens first. By default 500ms. You can decrease the value to improve latency.
_heartbeat.interval.ms_ must be lower than _session.timeout.ms_ and is usually set to one-third of the timeout value. These parameters directly imply on rebalances.

__partition.assignment.strategy__
 * Range - default strategy. Is the simplest but leads to different loads for consumers because distributed within a topic.
 * RoundRobin - is a more balanced variant
 * Sticky - as RoundRobin but try to stick partitions to consumers through rebalances which reduces some costs
 * Cooperative Sticky - joins previous strategies but additionally implements **Cooperative rebalance**





# Kafka Cluster Settings

## Retention
Defines the lifetime of messages in a topic. Messages are written on disk by batches (**segments**). And retention applied to segments not to single messages.
We can set _log.retention.ms_ or _log.retention.bytes_ what does meet first that will be applied. Retention can be set for a topic.

And retention applied after a segment was closed. When a segment is closing defined by _log.segment.bytes_ and _log.roll.ms_.
So messages cannot be removed until a segment hasn't been closed.

Topics can also be configured as log compacted, which means that Kafka will retain only the last message produced with a specific key. This can be useful for changelog-type data, where only the last update is interesting.




