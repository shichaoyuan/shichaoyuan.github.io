---
layout: post
title: Kafka Protocol & KIP
date: 2017-08-23 10:00:00 +08:00
tags: kafka
---

# Kafka Protocol & KIP

## Links

[KIP](https://cwiki.apache.org/confluence/display/KAFKA/Kafka+Improvement+Proposals)

[Kafka protocol guide](https://kafka.apache.org/protocol)

[A Guide To The Kafka Protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)


## ApiVersion

[KIP-35 - Retrieving protocol version](https://cwiki.apache.org/confluence/display/KAFKA/KIP-35+-+Retrieving+protocol+version)

[KAFKA-3304 KIP-35 - Retrieving protocol version](https://issues.apache.org/jira/browse/KAFKA-3304)

Fix Version/s: 0.10.2.0

[KAFKA-3307 Add ApiVersion request/response and server side handling.](https://issues.apache.org/jira/browse/KAFKA-3307)

Fix Version/s: 0.10.0.0

```
ApiVersionRequest => ApiKeys
  // empty

ApiVersionResponse => ApiVersions
  ErrorCode = INT16
  ApiVersions = [ApiVersion]
    ApiVersion = ApiKey MinVersion MaxVersion
      ApiKey = INT16
      MinVersion = INT16
      MaxVersion = INT16
```

## Metadata

### v1

[KIP-4 - Metadata Protocol Changes](https://cwiki.apache.org/confluence/display/KAFKA/KIP-4+-+Command+line+and+centralized+administrative+operations)

Released: 0.10.0.0

```
MetadataRequest => [topics]
Stays the same as version 0 however behavior changes.
In version 0 there was no way to request no topics, and and empty list signified all topics.
In version 1 a null topics list (size -1 on the wire) will indicate that a user wants ALL topic metadata. Compared to an empty list (size 0) which indicates metadata for NO topics should be returned.

MetadataResponse => [brokers] controllerId [topic_metadata]
  brokers => node_id host port rack
    node_id => INT32
    host => STRING
    port => INT32
    rack => NULLABLE_STRING
  controllerId => INT32
  topic_metadata => topic_error_code topic is_internal [partition_metadata]
    topic_error_code => INT16
    topic => STRING
    is_internal => BOOLEAN
    partition_metadata => partition_error_code partition_id leader [replicas] [isr]
      partition_error_code => INT16
      partition_id => INT32
      leader => INT32
      replicas => INT32
      isr => INT32

Adds rack, controller_id, and is_internal to the version 0 response.
The behavior of the replicas and isr arrays will be changed in order to support the admin tools, and better represent the state of the cluster:
In version 0, if a broker is down the replicas and isr array will omit the brokers entry and add a REPLICA_NOT_AVAILABLE error code.
In version 1, no error code will be set and a the broker id will be included in the replicas and isr array.
Note: A user can still detect if the replica is not available, by checking if the broker is in the returned broker list.
```

### v2

[KIP-78: Cluster Id](https://cwiki.apache.org/confluence/display/KAFKA/KIP-78%3A+Cluster+Id)

Released: 0.10.1.0

```
Metadata Request (Version: 2) => [topics]
  topics => STRING

Metadata Response (Version: 2) => cluster_id [brokers] controller_id [topic_metadata]
  cluster_id => STRING
  brokers => node_id host port rack
    node_id => INT32
    host => STRING
    port => INT32
    rack => NULLABLE_STRING
  controller_id => INT32
  topic_metadata => topic_error_code topic is_internal [partition_metadata]
    topic_error_code => INT16
    topic => STRING
    is_internal => BOOLEAN
    partition_metadata => partition_error_code partition_id leader [replicas] [isr]
      partition_error_code => INT16
      partition_id => INT32
      leader => INT32
      replicas => INT32
      isr => INT32
```

## CreateTopics/DeleteTopic

[KIP-4 - Command line and centralized administrative operations](https://cwiki.apache.org/confluence/display/KAFKA/KIP-4+-+Command+line+and+centralized+administrative+operations)

[KAFKA-2945 CreateTopic - protocol and server side implementation](https://issues.apache.org/jira/browse/KAFKA-2945)

Fix Version/s: 0.10.1.0

```
CreateTopics Request (Version: 0) => [create_topic_requests] timeout
  create_topic_requests => topic num_partitions replication_factor [replica_assignment] [configs]
    topic => STRING
    num_partitions => INT32
    replication_factor => INT16
    replica_assignment => partition_id [replicas]
      partition_id => INT32
      replicas => INT32
    configs => config_key config_value
      config_key => STRING
      config_value => STRING
  timeout => INT32

CreateTopics Response (Version: 0) => [topic_error_codes]
  topic_error_codes => topic error_code
    topic => STRING
    error_code => INT16
```

[KAFKA-2946 DeleteTopic - protocol and server side implementation](https://issues.apache.org/jira/browse/KAFKA-2946)

Fix Version/s: 0.10.1.0

```
DeleteTopics Request (Version: 0) => [topics] timeout
  topics => STRING
  timeout => INT32

DeleteTopics Response (Version: 0) => [topic_error_codes]
  topic_error_codes => topic error_code
    topic => STRING
    error_code => INT16

```

## DescribeConfigs/AlterConfigs

[KIP-133: Describe and Alter Configs Admin APIs](https://cwiki.apache.org/confluence/display/KAFKA/KIP-133%3A+Describe+and+Alter+Configs+Admin+APIs)

Released: 0.11.0.0

```
DescribeConfigs Request (Version: 0) => [resource [config_name]]
  resource => resource_type resource_name
    resource_type => INT8
    resource_name => STRING
  config_name => STRING

DescribeConfigs Response (Version: 0) => error_code [entities]
  entities => error_code resource_type resource_name [configs]
    error_code => INT16
    resource_type => INT8
    resource_name => STRING
    configs =>
      config_name => STRING
      config_value => STRING
      read_only => BOOLEAN
      is_default => BOOLEAN
      is_sensitive => BOOLEAN
```

```
AlterConfigs Request (Version: 0) => [resources] validate_only
  validate_only => BOOLEAN
  resources => resource_type resource_name [configs]
    resource_type => INT8
    resource_name => STRING
    configs =>
      config_name => STRING
      config_value => STRING

AlterConfigs Response (Version: 0) => [responses]
  responses => resource_type resource_name error_code error_message
    resource_type => INT8
    resource_name => STRING
    error_code => INT16
    error_message => STRING
```

## ListGroups/DescribeGroup

[KIP-40: ListGroups and DescribeGroup](https://cwiki.apache.org/confluence/display/KAFKA/KIP-40%3A+ListGroups+and+DescribeGroup)

Released: 0.9.0.0

```
ListGroupsRequest =>

ListGroupsResponse => [GroupId ProtocolType]
  GroupId => string
  ProtocolType => string
```

```
DescribeGroupRequest => GroupId
  GroupId => string

DescribeGroupResponse => ErrorCode State ProtocolType Protocol Leader Members
  ErrorCode => string
  State => string
  ProtocolType => string
  Protocol => string
  Leader => string
  Members => [MemberId Host ClientId MemberMetadata MemberAssignment]
    MemberId => string
    Host => string
    ClientId => string
    MemberMetadata => bytes
    MemberAssignment => bytes
```
