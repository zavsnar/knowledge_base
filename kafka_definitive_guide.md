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



# Kafka Cluster Settings

## Retention
Defines the lifetime of messages in a topic. Messages are written on disk by batches (**segments**). And retention applied to segments not to single messages.
We can set _log.retention.ms_ or _log.retention.bytes_ what does meet first that will be applied. Retention can be set for a topic.

And retention applied after a segment was closed. When a segment is closing defined by _log.segment.bytes_ and _log.roll.ms_.
So messages cannot be removed until a segment hasn't been closed.

Topics can also be configured as log compacted, which means that Kafka will retain only the last message produced with a specific key. This can be useful for changelog-type data, where only the last update is interesting.




