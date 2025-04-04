---
title: Kafka Connect ArangoDB Sink Connector
menuTitle: Kafka Connector
weight: 20
description: >-
  The Kafka connector allows you to export data from Apache Kafka to ArangoDB
  by writing data from one or more topics in Kafka to a collection in ArangoDB
---
{{< info >}}
Check out the [connector demo](https://github.com/arangodb/kafka-connect-arangodb/tree/main/demo)
to learn more about the connector.
{{< /info >}}

## Supported versions

This connector is compatible with:

- Kafka 3.4 and higher versions
- JDK 8 and higher versions
- ArangoDB 3.11.1 and higher versions

## Installation

Download the Jar file from [Maven Central](https://repo1.maven.org/maven2/com/arangodb/kafka-connect-arangodb)
and copy it into one of the directories that are listed in the Kafka Connect
worker's `plugin.path` configuration property. This must be done on each of the
installations where Kafka Connect will be run.

Once installed, you can then create a connector configuration file with the
connector's settings, and deploy that to a Connect worker.
See the [configuration](configuration.md) documentation for
the available options.

For more detailed plugin installation instructions, see the
[Confluent Documentation](https://docs.confluent.io/platform/current/connect/userguide.html#connect-installing-plugins).

## Connection handling

Task connections to the database are evenly distributed across all available
ArangoDB Coordinators. If a connectivity error occur, a connection is
re-established with a different Coordinator.

Available Coordinators can be periodically monitored by setting
`connection.acquireHostList.enabled` to `true` and optionally configuring the
monitoring interval.

Over time, due to connection failovers, the distribution of task connections
across the Coordinators may become uneven. To address this, connections are
periodically re-balanced across the available Coordinators. The rebalancing
interval can be configured via the `connection.rebalance.interval.ms` property.
The default is 30 minutes.

## Delivery guarantees

This connector guarantees that each record in the Kafka topic is delivered at
least once. For example, the same record could be delivered multiple times in
the following scenarios:

- Transient errors in the communication between the connector and the
  database system, leading to [retries](#retries)
- Errors in the communication between the connector and Kafka, preventing to
  commit offsets of already written records
- Abrupt termination of connector task

When restarted, the connector resumes reading from the Kafka topic at an offset
prior to where it stopped. As a result, at least in the cases mentioned above,
some records might get written to ArangoDB more than once. Even if configured
for idempotent writes (e.g. with `insert.overwriteMode=replace`), writing the
same record multiple times still updates the `_rev` field of the document.

Note that in case of retries, [Ordering Guarantees](#ordering-guarantees)
are still provided.

To improve the likelihood that every write survives even in case of a DB-Server
failover, consider configuring the configuration property `insert.waitForSync`
(default `false`), which determines whether the write operations are synced
to disk before returning.

## Error handling

The connector categorizes all the possible errors into two types,
data errors and transient errors.

### Data errors

Data errors are unrecoverable and caused by the data being processed.
For example:

- Conversion errors:
    - Illegal key type
    - Illegal value type
- Server validation errors:
    - Illegal `_key`, `_from`, `_to` values
    - JSON schema validation errors
- Server constraints violations
    - Unique index violations
    - Key conflicts (in case of `insert.overwriteMode=conflict`)

The configuration property `data.errors.tolerance` allows you to configure the
behavior for tolerating data errors:

- `none`: data errors result in an immediate connector task failure (default)
- `all`: changes the behavior to skip over records generating data errors.
  If DLQ is configured, then the record is reported
  (see [Dead Letter Queue](#dead-letter-queue)).

Data errors detection can be further customized via the `extra.data.error.nums`
configuration property. In addition to the cases listed above, the server errors
reporting `errorNums` listed by this configuration property are considered
data errors.

### Transient errors

Transient errors are recoverable and may succeed if retried with some delay
(see [Retries](#retries)). If all retries fail, then the connector task fails.

All errors that are not data errors are considered transient errors.

## Retries

In case of transient errors, the [`max.retries` configuration property](configuration.md#maxretries)
determines how many times the connector retries.

The [`retry.backoff.ms` configuration property](configuration.md#retrybackoffms)
allows you to set the time in milliseconds to wait following an error before a
retry attempt is made.

Data errors are never retried.

## Dead Letter Queue

This connector supports the Dead Letter Queue (DLQ) functionality.
For information about accessing and using the DLQ,
see [Confluent Platform Dead Letter Queue](https://docs.confluent.io/platform/current/connect/concepts.html#dead-letter-queue).

Only data errors can be reported to the DLQ. Transient errors, after potential
retries, always make the task fail.

You can enable DLQ support for data errors by setting `data.errors.tolerance=all`
and `errors.deadletterqueue.topic.name`.

## Multiple tasks

The connector can scale out by running multiple tasks in parallel as specified by the
[`tasks.max`](https://docs.confluent.io/platform/current/installation/configuration/connect/sink-connect-configs.html#tasks-max)
configuration parameter. In this case, each connector task handles the records
from different Kafka topics partitions.

## Data mapping

The sink connector optionally supports schemas. For example, the Avro converter
that comes with Schema Registry, the JSON converter with schemas enabled, or the
Protobuf converter.

Kafka record keys and Kafka record value field `_key`, if present, must be a
primitive type of either:

- `string`
- `integral numeric` (integer)

The record value must be either:

- `struct` (Kafka Connect structured record)
- `map`
- `null` (tombstone record)

Record values are converted to JSON objects using Kafka Connect `JsonConverter`,
ensuring compatibility with Kafka Connect data types and support to schemas.
If the data in the topic is not of a compatible format, applying an
[SMT](https://docs.confluent.io/platform/current/connect/transforms/overview.html)
or implementing a custom converter may be necessary.

## Key handling

The `_key` of the documents inserted into ArangoDB is derived in the following way:

1. Use the Kafka record value field `_key` if present and not null, else
2. Use the Kafka record key if not null, else
3. Use the Kafka record coordinates (`topic-partition-offset`)

## Delete mode

The connector can delete documents in a database collection when it consumes a
tombstone record, which is a Kafka record that has a non-null key and a null value.
This behavior is disabled by default, meaning that any tombstone records results
in a failure of the connector.

You can enable deletes with `delete.enabled=true`.

Delete operations are idempotent and do not throw errors if the target document
does not exist.

Enabling delete mode does not affect the `insert.overwriteMode`.

## Write modes

The `insert.overwriteMode` configuration parameter allow you to set the write
behavior in case a document with the same `_key` already exists:

- `conflict`: the new document value is not written and an exception is thrown (default)
- `ignore`: the new document value is not written
- `replace`: the existing document is overwritten with the new document value
- `update`: the existing document is patched (partially updated) with the new document value

## Idempotent writes

All the write modes supported are idempotent, with the exception that the
document revision field (`_rev`) changes every time a document is written. See
[Document revisions](../../../concepts/data-structure/documents/_index.md#document-revisions)
for more details.

If there are failures, the Kafka offset used for recovery may not be up-to-date
with what was committed as of the time of the failure, which can lead to
re-processing during recovery. In case of `insert.overwriteMode=conflict` (default),
this can lead to constraint violations errors if records need to be re-processed.

## Ordering guarantees

Kafka records in the same Kafka topic partition that are mapped to documents with the
same `_key` (see [Key handling](#key-handling)) are written to ArangoDB in the
same order as they are in the Kafka topic partition.

The order of the writes for records in the same Kafka partition that are mapped
to documents with different `_key` is not guaranteed.

The order between writes for records in different Kafka partitions is not guaranteed.

To guarantee documents in ArangoDB are eventually consistent with the records in
the Kafka topic, it is recommended deriving the document `_key` from Kafka
record keys and using a key-based partitioner that assigns the same partition to
records with the same key (i.e. Kafka default partitioner).

Otherwise, in case the document `_key` is assigned from Kafka record value field
`_key`, the same could be achieved using a field partitioner on `_key`.

When restarted, the connector may resume reading from the Kafka topic at an offset
prior to where it stopped. This can lead to reprocessing of batches containing
multiple Kafka records that are mapped to documents with the same `_key`.
In such case, it is possible to observe the related document in the database
being temporarily updated to older versions and eventually to newer ones.

## Monitoring

The Kafka Connect framework exposes basic status information over a REST interface.
Fine-grained metrics, including the number of processed messages and the rate of
processing, are available via JMX. For more information, see
[Monitoring Kafka Connect and Connectors](https://docs.confluent.io/current/connect/managing/monitoring.html)
(published by Confluent, but equally applies to a standard Apache Kafka distribution).

## SSL

To connect to ArangoDB using an SSL connection, you must set the `ssl.enabled`
configuration property to `true`.

### Certificate from file

The connector can load the trust store to be used from file. The following
configuration properties can be used:

- `ssl.truststore.location`: the location of the trust store file
- `ssl.truststore.password`: the password for the trust store file

Note that the trust store file path needs to be accessible at the same given
location from all Kafka Connect workers.

### Certificate from configuration property value

The connector can accept the SSL certificate value from configuration property
encoded as Base64 string. The following configuration properties can be used:

- `ssl.cert.value`: Base64-encoded SSL certificate

See [SSL configuration](configuration.md#ssl) for further options.

## Limitations

- Record values are required to be object-like structures (DE-644)
- Auto-creation of ArangoDB collection is not supported (DE-653)
- Batch writes are not guaranteed to be executed serially (FRB-300)
- Batch writes may succeed for some documents while failing for others (FRB-300)
  This has two important consequences:
  - Transient errors might be retried and succeed at a later point
  - Data errors might be asynchronously reported to the DLQ
