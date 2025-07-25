---
title: Deprecated and removed features
menuTitle: Deprecated and removed features
weight: 100
description: >-
  Features listed in this section should no longer be used, because they are considered obsolete and may get removed in a future release
aliases:
  - ../operations/upgrading-manual-deployments/upgrading-an-active-failover-deployment
  - ../operations/upgrading/manual-deployments/active-failover
  - ../deploy/active-failover/using-the-arangodb-starter
  - ../deploy/active-failover/manual-start
  - ../deploy/active-failover/administration
  - ../deploy/active-failover
  - ../deploy/arangosync
  - ../deploy/arangosync/deployment
  - ../deploy/arangosync/deployment/arangodb-cluster
  - ../deploy/arangosync/deployment/arangosync-master
  - ../deploy/arangosync/deployment/arangosync-workers
  - ../deploy/arangosync/deployment/prometheus-and-grafana
  - ../deploy/arangosync/administration
  - ../deploy/arangosync/operations-and-maintenance
  - ../deploy/arangosync/security
  - ../deploy/arangosync/monitoring
  - ../deploy/arangosync/troubleshooting
  - ../components/arangodb-server/ldap
  - ../arangograph/migrate-to-the-cloud
  - ../data-science/pregel
  - ../data-science/pregel/algorithms
  - ../develop/http-api/pregel
  - ../operations/installation/macos
  - ../operations/installation/windows
  - ../operations/installation/compiling/compile-on-windows
  - ../operations/upgrading/os-specific-information/macos
  - ../operations/upgrading/os-specific-information/windows
---
Features listed on this page should no longer be used because they have been
deprecated and may get removed in a future release, or have been removed already
and are thus no longer available.

Deprecated features are still available for backward compatibility, but you should
update your applications to prepare for upgrades of ArangoDB that may remove the
features. There are usually alternatives to replace the old features with.

{{< info >}}
This page only lists significant obsolete features but not minor API changes.
See the [Release notes](_index.md) of the respective versions for
detailed information about breaking changes before upgrading.
{{< /info >}}

- **Native Windows and macOS support**:
  Starting with v3.12, the native platform support for the Windows and macOS
  operating systems has been removed and ArangoDB packages for Windows and macOS
  are not provided anymore. You can use the official
  [Docker images](https://hub.docker.com/_/arangodb/) instead, to run ArangoDB
  in Linux containers.

- **Active Failover deployment mode**:
  Running a single server with asynchronous replication to one or more passive
  single servers for automatic failover is no longer supported from v3.12.0 onward.
  You can use [cluster deployments](../deploy/cluster/_index.md) instead, which
  offer better resilience and synchronous replication.

- **Datacenter-to-Datacenter Replication (DC2DC)**:
  The Datacenter-to-Datacenter Replication for cluster deployments including the
  _arangosync_ tool is no longer supported from v3.12 onward.

- **LDAP authentication**:
  ArangoDB user authentication with an LDAP server in the Enterprise Edition is
  no longer available starting with v3.12.0.

- **VelocyStream protocol**:
  ArangoDB's own bi-directional asynchronous binary protocol VelocyStream is no
  longer supported. VelocyPack remains as ArangoDB's binary storage format and
  you can continue to use it in transport over the HTTP protocol, as well as use
  JSON over the HTTP protocol.

- **Standalone Agency and Agency HTTP API**:
  The Standalone Agency deployment mode and the corresponding Agency HTTP API
  are no longer available starting with v3.12.0. 

- **Little-endian on-disk key format for the RocksDB storage engine**:

  The little-endian on-disk key format for the RocksDB storage engine is
  deprecated and support is removed in v3.12.0.

  Only deployments that were set up with the RocksDB storage engine using
  ArangoDB v3.2 or v3.3 and that have been upgraded since then are affected.

  See [Incompatible changes in ArangoDB 3.12](version-3.12/incompatible-changes-in-3-12.md#little-endian-on-disk-key-format-for-the-rocksdb-storage-engine-unsupported)
  for details.

- **Pregel**:

  The distributed iterative graph processing (Pregel) system is no longer supported
  from v3.12 onward. All Pregel graph algorithms, the Pregel JavaScript API and
  HTTP API, and everything else related to Pregel has been removed.
  All other graph features including AQL graph traversals and path finding
  algorithms are unaffected.

- **Cloud Migration Tool**:
  The `arangosync-migration` tool to move from on-premises to the cloud is not
  available anymore.

- **Leader/Follower Deployment Mode**:
  The Leader/Follower deployment mode is deprecated and already removed from
  documentation. OneShard databases in clusters are a better alternative.

- **Skiplist and hash indexes**:
  Skiplist and hash indexes have been deprecated in 3.9 and will be removed in a 
  future version of ArangoDB. Currently, they are an alias for a
  [persistent index](../index-and-search/indexing/basics.md#persistent-index).

- **Bundled NPM modules**:
  The bundled NPM modules `aqb`, `chai`, `dedent`, `error-stack-parser`,
  `graphql-sync`, `highlight.js`, `i` (inflect), `iconv-lite`, `joi`,
  `js-yaml`, `lodash`, `minimatch`, `qs`, `semver`, `sinon`, and `timezone`
  have been deprecated in 3.9 and will be removed in a future version of ArangoDB.
  If you want to use NPM modules in your Foxx service, please refer to the
  [Foxx guide](../develop/foxx-microservices/guides/using-node-modules.md).

- **Batch Requests API**:
  The [batch request REST API](../develop/http-api/batch-requests.md) was deprecated
  in v3.8.0 and has been removed in v3.12.3. Instead of using this API, please use the
  [HTTP interface for documents](../develop/http-api/documents.md#multiple-document-operations)
  that can insert, update, replace or remove arrays of documents.

- **PUT method in Cursor API**:
  The HTTP endpoint `PUT /_api/cursor/<cursor-id>` in the
  [Cursor REST API](../develop/http-api/queries/aql-queries.md) is deprecated and will be
  removed in a future version. Please use the drop-in replacement
  `POST /_api/cursor/<cursor-id>` instead. The POST endpoint is functionally
  equivalent to the PUT endpoint, but does not violate idempotency requirements
  prescribed by the [HTTP specification](https://tools.ietf.org/html/rfc7231#section-4.2).

- **Fulltext indexes**:
  The fulltext index type is deprecated from version 3.10 onwards.
  It is recommended to use [ArangoSearch](../index-and-search/arangosearch/_index.md) for advanced full-text search capabilities.

- **Simple Queries**: Idiomatic interface in arangosh to perform trivial queries.
  They are superseded by [AQL queries](../aql/_index.md), which can also
  be run in arangosh. AQL is a language on its own and way more powerful than
  *Simple Queries* could ever be. In fact, the (still supported) *Simple Queries*
  are translated internally to AQL, then the AQL query is optimized and run
  against the database in recent versions, because of better performance and
  reduced maintenance complexity.

- **Accessing collections by ID instead of by name**:
  Accessing collections by their internal ID instead of accessing them by name
  is deprecated and highly discouraged. This functionality may be removed in
  future versions of ArangoDB.

- **Old metrics REST API**:
  The old metrics API under `/_admin/metrics` is deprecated and replaced by
  a new one under `/_admin/metrics/v2` from version 3.8.0 on. This step was
  necessary because the old API did not follow quite a few Prometheus
  guidelines for metrics.

- **Statistics REST API**:
  The endpoints `/_admin/statistics` and `/_admin/statistics-description`
  are deprecated in favor of the new metrics API under `/_admin/metrics/v2`.
  The metrics API provides a lot more information than the statistics API, so
  it is much more useful.

- **Database target version REST API**:
  The `GET /_admin/database/target-version` endpoint is deprecated in favor of the
  more general version API with the endpoint `GET /_api/version`. The endpoint may be
  removed in a future version of ArangoDB.

- **Replication logger-follow REST API**:
  The endpoint `/_api/replication/logger-follow` is deprecated since 3.4.0 and
  may be removed in a future version. Client applications should use the REST 
  API endpoint `/_api/wal/tail` instead, which is available since ArangoDB 3.3.

- **Loading and unloading of collections**:
  The JavaScript functions for explicitly loading and unloading collections,
  `db.<collection-name>.load()` and `db.<collection-name>.unload()` and their
  REST API endpoints `PUT /_api/collection/<collection-name>/load` and
  `PUT /_api/collection/<collection-name>/unload` are deprecated in 3.8.
  There should be no need to explicitly load or unload a collection with the
  RocksDB storage engine. The load/unload functionality was useful only with
  the MMFiles storage engine, which is not available anymore since 3.7.

- **Actions**: Snippets of JavaScript code on the server-side for minimal
  custom endpoints. Since the Foxx revamp in 3.0, it became really easy to
  write [Foxx Microservices](../develop/foxx-microservices/_index.md), which allow you to define
  custom endpoints even with complex business logic.

  From v3.5.0 on, the system collections `_routing` and `_modules` are not
  created anymore when the `_system` database is first created (blank new data
  folder). They are not actively removed, they remain on upgrade or backup
  restoration from previous versions.

- **Outdated AQL functions**: The following AQL functions are deprecated and
  their usage is discouraged:
  - `IS_IN_POLYGON`
  - `NEAR`
  - `WITHIN`
  - `WITHIN_RECTANGLE`

  See [Geo functions](../aql/functions/geo.md) for substitutes.

- **`bfs` option** in AQL graph traversal: Using the *bfs* attribute inside
  traversal options is deprecated since v3.8.0. The preferred way to start a
  breadth-first traversal is by using the new `order` attribute, and setting it
  to a value of `bfs`.

- **`overwrite` option**: The `overwrite` option for insert operations (either
  single document operations or AQL `INSERT` operations) is deprecated in favor
  of the `overwriteMode` option, which provides more flexibility.

- **`minReplicationFactor` collection option**: The `minReplicationFactor`
  option for collections has been renamed to `writeConcern`. If
  `minReplicationFactor` is specified and no `writeConcern` is set, the
  `minReplicationFactor` value will still be picked up and used as
  `writeConcern` value. However, this compatibility mode will be removed
  eventually, so changing applications from using `minReplicationFactor` to
  `writeConcern` is advised.

- **Outdated startup options**

  The following _arangod_ startup options are deprecated and will be removed
  in a future version:
  - `--database.old-system-collections` (no need to use it anymore)
  - `--server.jwt-secret` (use `--server.jwt-secret-keyfile`) 
  - `--arangosearch.threads` / `--arangosearch.threads-limit`
    (use the following options instead):
    - `--arangosearch.commit-threads`
    - `--arangosearch.commit-threads-idle`
    - `--arangosearch.consolidation-threads`
    - `--arangosearch.consolidation-threads-idle`
  - `--rocksdb.exclusive-writes` (was intended only as a stopgap measure to
    make porting applications from MMFiles to RocksDB easier)
  - `--network.protocol`: network protocol to use for cluster-internal 
    communication. The protocol will be auto-decided from version 3.9 onwards.
  - `--query.allow-collections-in-expressions`: allow full collections to be 
    used in AQL expressions. This option defaults to `false` from version 3.9
    onwards and will be removed in a future version. It is only useful to 
    enable it when migrating from older versions.

  The following options are deprecated for _arangorestore_:
  - `--default-number-of-shards` (use `--number-of-shards` instead)
  - `--default-replication-factor` (use `--replication-factor` instead)

  The following startup options are deprecated in _arangod_ and all client tools:
  - `--log` (use `--log.level` instead)
  - `--log.use-local-time` (use `--log.time-format` instead)
  - `--log.use-microtime` (use `--log.time-format` instead)
  - `--log.performance` (use `--log.level` instead)

- **Obsoleted startup options**: Any startup options marked as obsolete can be
  removed in any future version of ArangoDB, so their usage is highly
  discouraged. Their functionality is already removed, but they still exist to
  prevent unknown startup option errors.

- **arangoimp** executable: ArangoDB release packages install an executable named
  _arangoimp_ as an alias for the _arangoimport_ executable. This is done to 
  provide compatibility with older releases, in which _arangoimport_ did not
  yet exist and was named _arangoimp_. The renaming was actually carried out in
  the codebase in December 2017. Using the _arangoimp_ executable is deprecated,
  and it is always favorable to use _arangoimport_ instead. 
  While the _arangoimport_ executable will remain, the _arangoimp_ alias will be 
  removed in a future version of ArangoDB.

- **HTTP and JavaScript traversal APIs**: The HTTP traversal API as well as the
  `@arangodb/graph/traversal` JavaScript traversal module were deprecated since
  version 3.4.0 and have been removed in version 3.12.0. You can
  [traverse graphs with AQL](../aql/graphs/traversals.md) instead.

- **Specialized index creation methods in JavaScript API**:
  The following JavaScript methods for creating indexes from the ArangoShell
  (_arangosh_) or from within Foxx are deprecated:
  - `collection.ensureHashIndex(...)`
  - `collection.ensureUniqueConstraint(...)`
  - `collection.ensureSkiplist(...)`
  - `collection.ensureUniqueSkiplist(...)`
  - `collection.ensureFulltextIndex(...)`
  - `collection.ensureGeoIndex(...)`
  - `collection.ensureGeoConstraint(...)`

  Instead of using these methods, you should use the generic
  `collection.ensureIndex(...)` method, which provides a superset of all the
  deprecated methods. Also see
  [Creating an index](../index-and-search/indexing/working-with-indexes/_index.md#creating-an-index).
