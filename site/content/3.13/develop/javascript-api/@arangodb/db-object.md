---
title: The `db` object of the JavaScript API
menuTitle: '`db` object'
weight: 5
description: >-
  The database object represents the currently selected database and provides
  access to information and methods for executing operations in the context of
  this database
# Undocumented on purpose:
#   db._path() // MMFiles legacy  
---
The `db` object of the JavaScript API is available in [arangosh](../../../components/tools/arangodb-shell/_index.md)
by default, and can also be imported and used in Foxx services and other
server-side JavaScript contexts from the `@arangodb` module.

## Property access

- `db.<collection-name>` and `db["<collection-name>"]` return a
  [_collection_ object](collection-object.md) for the
  specified collection if it exists.

- `db.<view-name>` and `db["<view-name>"]` return a
  [_view_ object](view-object.md) for the
  specified View if it exists.

## Databases

### `db._createDatabase(name [, options [, users]])`

Creates a new database with the specified name.
There are restrictions for database names
(see [Database names](../../../concepts/data-structure/databases.md#database-names)).

{{< info >}}
You can only create new databases from within the `_system` database.
{{< /info >}}

Note that even if the database is created successfully, there will be no
change into the current database to the new database. Changing the current
database must explicitly be requested by using the
`db._useDatabase()` method.

The `options` attribute can be used to set defaults for collections that will
be created in the new database (_cluster only_):

- `sharding`: The sharding method to use. Valid values are: `""` or `"single"`.
  Setting this option to `"single"` enables the OneShard feature.
- `replicationFactor`: Default replication factor. Special values include
  `"satellite"`, which will replicate the collection to every DB-Server, and
  `1`, which disables replication.
- `writeConcern`: how many copies of each shard are required to be in sync on
  the different DB-Servers. If there are less then these many copies in the
  cluster a shard will refuse to write. The value of `writeConcern` cannot be
  greater than `replicationFactor`.

The optional `users` attribute can be used to create initial users for
the new database. If specified, it must be a list of user objects. Each user
object can contain the following attributes:

- `username`: the user name as a string. This attribute is mandatory.
- `passwd`: the user password as a string. If not specified, then it defaults
  to an empty string.
- `active`: a boolean flag indicating whether the user account should be
  active or not. The default value is `true`.
- `extra`: an optional JSON object with extra user information. The data
  contained in `extra` will be stored for the user but not be interpreted
  further by ArangoDB.

If no initial users are specified, a default user `root` will be created
with an empty string password. This ensures that the new database will be
accessible via HTTP after it is created.

You can create users in a database if no initial user is specified. Switch
into the new database (username and password must be identical to the current
session) and add or modify users with the following commands.

```js
require("@arangodb/users").save(username, password, true);
require("@arangodb/users").update(username, password, true);
require("@arangodb/users").remove(username);
```
Alternatively, you can specify user data directly. For example:

```js
db._createDatabase("newDB", {}, [{ username: "newUser", passwd: "123456", active: true}])
```

### `db._useDatabase(name)`

Changes the current database to the specified database.
Note that the database specified by `name` must already exist.

Changing the database might be disallowed in some contexts, for example,
in server-side actions (including Foxx).

When performing this command from arangosh, the current credentials (username
and password) will be re-used. These credentials might not be valid to
connect to the database specified by `name`. Additionally, the database
only be accessed from certain endpoints only. In this case, switching the
database might not work, and the connection / session should be closed and
restarted with different username and password credentials and/or
endpoint data.

### `db._databases()`

Returns the list of all existing databases.

{{< info >}}
The databases can only be listed from within the `_system` database.
{{< /info >}}

### `db._dropDatabase(name)`

Drops the specified database. The database specified by `name` must exist.

{{< info >}}
Databases can only be dropped from within the `_system` database.
The `_system` database itself cannot be dropped.
{{< /info >}}

Databases are dropped asynchronously, and will be physically removed if
all clients have disconnected and references have been garbage-collected.

### `db._name()`

Returns the name of the current database as a string.

**Examples**

```js
---
name: dbName
description: ''
---
require("@arangodb").db._name();
```

### `db._id()`

Returns the identifier of the current database as a string.

**Examples**

```js
---
name: dbId
description: ''
---
require("@arangodb").db._id();
```

### `db._isSystem()`

Returns whether the currently used database is the `_system` database.

The system database has some special privileges and properties, for example,
database management operations such as creating or dropping databases can only
be executed from within the `_system` database. The `_system` database itself
cannot be dropped.

### `db._properties()`

Returns the properties of the current database as an object with the following
attributes:

- `id`: the database identifier
- `name`: the database name
- `isSystem`: the database type
- `path`: the path to the database files (not used anymore, always `""`)
- `sharding`: the sharding method to use for new collections _(cluster only)_
- `replicationFactor`: default replication factor for new collections
  _(cluster only)_
- `writeConcern`: a shard will refuse to write if less than this amount
  of copies are in sync _(cluster only)_

**Examples**

```js
---
name: dbProperties
description: ''
type: cluster
---
require("@arangodb").db._properties();
```

## Collections

### `db._create(collection-name [, properties [, type] [, options]])`

Create a new document collection or edge collection.

`db._create(collection-name)`

Creates a new document collection named `collection-name` with the default
settings and returns a [_collection_ object](collection-object.md).

If a collection or View with this name exists already, or if the name format is invalid, an
error is thrown. For information about the naming constraints for collections, see
[Collection names](../../../concepts/data-structure/collections.md#collection-names).

---

`db._create(collection-name, properties)`

`properties` must be an object with the following attributes:

- `waitForSync` (boolean, _optional_, default `false`): If `true`, creating,
  changing, or removing a document waits until the data is synchronized to disk.

- `keyOptions` (object, _optional_): The options for key generation. If
  specified, then `keyOptions` should be an object containing the
  following attributes:
  - `type` (string): specifies the type of the key generator.
    The available generators are `"traditional"` (default), `"autoincrement"`,
    `"uuid"` and `"padded"`.
    - The `traditional` key generator generates numerical keys in ascending order.
      The sequence of keys is not guaranteed to be gap-free.
    - The `autoincrement` key generator generates numerical keys in ascending order, 
      the initial offset and the spacing can be configured (**note**: 
      `autoincrement` is only supported for non-sharded or 
      single-sharded collections). 
      The sequence of generated keys is not guaranteed to be gap-free, because a new key
      is generated on every document insert attempt, not just for successful
      inserts.
    - The `padded` key generator generates keys of a fixed length (16 bytes) in
      ascending lexicographical sort order. This is ideal for the RocksDB storage engine,
      which slightly benefits keys that are inserted in lexicographically
      ascending order. The key generator can be used in a single-server or cluster.
      The sequence of generated keys is not guaranteed to be gap-free.
    - The `uuid` key generator generates universally unique 128 bit keys, which 
      are stored in hexadecimal human-readable format. This key generator can be used
      in a single-server or cluster to generate "seemingly random" keys. The keys 
      produced by this key generator are not lexicographically sorted.

    Please note that keys are only guaranteed to be truly ascending in single
    server deployments and for collections that only have a single shard (that includes
    collections in a OneShard database).
    The reason is that for collections with more than a single shard, document keys
    are generated on Coordinator(s). For collections with a single shard, the document
    keys are generated on the leader DB-Server, which has full control over the key
    sequence.

  - `allowUserKeys` (boolean, _optional_): If set to `true`, then you are allowed
    to supply own key values in the `_key` attribute of documents. If set to
    `false`, then the key generator is solely responsible for generating keys and
    an error is raised if you supply own key values in the `_key` attribute
    of documents.

    {{< warning >}}
    You should not use both user-specified and automatically generated document keys
    in the same collection in cluster deployments for collections with more than a
    single shard. Mixing the two can lead to conflicts because Coordinators that
    auto-generate keys in this case are not aware of all keys which are already used.
    {{< /warning >}}
  - `increment`: The increment value for the `autoincrement` key generator.
    Not allowed for other key generator types.
  - `offset`: The initial offset value for the `autoincrement` key generator.
    Not allowed for other key generator types.
  - `lastValue`: the offset value for the `autoincrement` or `padded`
    key generator. This is an internal property for restoring dumps properly.

- `schema` (object\|null, _optional_, default: `null`): 
  An object that specifies the collection-level document schema for documents.
  The attribute keys `rule`, `level` and `message` must follow the rules
  documented in [Document Schema Validation](../../../concepts/data-structure/documents/schema-validation.md)

- `computedValues` (array\|null, _optional_, default: `null`): An array of objects,
  each representing a [Computed Value](../../../concepts/data-structure/documents/computed-values.md).

- `cacheEnabled` (boolean): Whether the in-memory hash cache for documents should be
  enabled for this collection (default: `false`). Can be controlled globally
  with the `--cache.size` startup option. The cache can speed up repeated reads
  of the same documents via their document keys. If the same documents are not
  fetched often or are modified frequently, then you may disable the cache to
  avoid the maintenance costs.

- `isSystem` (boolean, _optional_, default: `false`): If `true`, create a
  system collection. In this case, the collection name should start with
  an underscore. End-users should normally create non-system collections
  only. API implementors may be required to create system collections in
  very special occasions, but normally a regular collection is sufficient.

- `syncByRevision` (boolean, _optional_, default: `true`):
  Whether the newer revision-based replication protocol
  is enabled for this collection. This is an internal property.

- `numberOfShards` (number, _optional_, default `1`): In a cluster, this value
  determines the number of shards to create for the collection. In a single
  server setup, this option is meaningless.

- `shardKeys` (array, _optional_, default: `["_key"]`): In a cluster, this
  attribute determines which document attributes are used to determine the
  target shard for documents. Documents are sent to shards based on the
  values they have in their shard key attributes. The values of all shard
  key attributes in a document are hashed, and the hash value is used to
  determine the target shard. Note that values of shard key attributes cannot
  be changed once set.
  This option is meaningless in a single server setup.

  When choosing the shard keys, you must be aware of the following
  rules and limitations: In a sharded collection with more than
  one shard it is not possible to set up a unique constraint on
  an attribute that is not the one and only shard key given in
  `shardKeys`. This is because enforcing a unique constraint
  would otherwise make a global index necessary or need extensive
  communication for every single write operation. Furthermore, if
  `_key` is not the one and only shard key, then it is not possible
  to set the `_key` attribute when inserting a document, provided
  the collection has more than one shard. Again, this is because
  the database has to enforce the unique constraint on the `_key`
  attribute and this can only be done efficiently if this is the
  only shard key by delegating to the individual shards.

- `replicationFactor` (number\|string, _optional_, default `1`): In a cluster, this
  attribute determines how many copies of each shard are kept on 
  different DB-Servers. The value 1 means that only one copy (no
  synchronous replication) is kept. A value of k means that
  k-1 replicas are kept. Any two copies reside on different DB-Servers.
  Replication between them is synchronous, that is, every write operation
  to the "leader" copy is replicated to all "follower" replicas,
  before the write operation is reported successful.

  If a server fails, this is detected automatically and one of the
  servers holding copies take over, usually without an error being
  reported.

  The `replicationFactor`
  may be set to `"satellite"`, making the collection locally joinable
  on every DB-Server. This reduces the number of network hops
  dramatically when using joins in AQL at the costs of reduced write
  performance on these collections.

- `writeConcern` (number, _optional_, default `1`): In a cluster, this
  attribute determines how many copies of each shard are required
  to be in sync on the different DB-Servers. If there are less then these
  many copies in the cluster, a shard refuses to write. The value of
  `writeConcern` cannot be greater than `replicationFactor`.
  Please note: during server failures this might lead to writes
  not being possible until the failover is sorted out and might cause
  write slow downs in trade for data durability.

- `shardingStrategy` (optional): specifies the name of the sharding
  strategy to use for the collection. There are
  different sharding strategies to select from when creating a new 
  collection. The selected `shardingStrategy` value remains
  fixed for the collection and cannot be changed afterwards. This is
  important to make the collection keep its sharding settings and
  always find documents already distributed to shards using the same
  initial sharding algorithm.

  The available sharding strategies are:
  - `"community-compat"`: default sharding used by ArangoDB
    Community Edition before version 3.4
  - `"enterprise-compat"`: default sharding used by ArangoDB
    Enterprise Edition before version 3.4
  - `"enterprise-smart-edge-compat"`: default sharding used by smart edge
    collections in ArangoDB Enterprise Edition before version 3.4
  - `"hash"`: default sharding used for new collections starting from version 3.4
    (excluding smart edge collections)
  - `"enterprise-hash-smart-edge"`: default sharding used for new
    smart edge collections starting from version 3.4
  - `enterprise-hex-smart-vertex`: sharding used for vertex collections of
    EnterpriseGraphs

  If no sharding strategy is specified, the default is `hash` for
  all normal collections, `enterprise-hash-smart-edge` for all smart edge
  collections, and `enterprise-hex-smart-vertex` for EnterpriseGraph
  vertex collections.
  Manually overriding the sharding strategy does not yet provide a 
  benefit, but it may later in case other sharding strategies are added.
  
  In single-server mode, the `shardingStrategy` attribute is meaningless 
  and is ignored.

- `distributeShardsLike` (string, _optional_, default: `""`):
  The name of another collection. If this property is set in a cluster, the
  collection copies the `replicationFactor`, `numberOfShards` and `shardingStrategy`
  properties from the specified collection (referred to as the _prototype collection_)
  and distributes the shards of this collection in the same way as the shards of
  the other collection. This data co-location is utilized to optimize queries.

  You need to use the same number of `shardKeys` as the prototype collection, but
  you can use different attributes.

  {{< info >}}
  Using this parameter has consequences for the prototype collection.
  It can no longer be dropped unless the sharding-imitating collections are
  dropped beforehand. Equally, backups and restores of imitating collections
  alone result in errors about missing sharding prototypes.
  {{< /info >}}

- `isSmart` (boolean): Whether the collection is for a SmartGraph or
  EnterpriseGraph. This is an internal property.

- `isDisjoint` (boolean): Whether the collection is for a Disjoint SmartGraph.
  This is an internal property.

- `smartGraphAttribute` (string, _optional_):
  The attribute that is used for sharding: vertices with the same value of
  this attribute are placed in the same shard. All vertices are required to
  have this attribute set and it has to be a string. Edges derive the
  attribute from their connected vertices.

- `smartJoinAttribute` (string, _optional_): In a cluster, this attribute 
  determines an attribute of the collection that must contain the shard key value 
  of the referred-to SmartJoin collection. Additionally, the sharding key 
  for a document in this collection must contain the value of this attribute, 
  followed by a colon, followed by the actual primary key of the document.

  This feature requires the
  `distributeShardsLike` attribute of the collection to be set to the name
  of another collection. It also requires the `shardKeys` attribute of the
  collection to be set to a single shard key attribute, with an additional `:`
  at the end.
  A further restriction is that whenever documents are stored or updated in the 
  collection, the value stored in the `smartJoinAttribute` must be a string.

---

`db._create(collection-name, properties, type)`

Specifies the optional `type` of the collection, it can either be `document` 
or `edge`. On default it is document. Instead of giving a type you can also use 
`db._createEdgeCollection()` or `db._createDocumentCollection()`.

---

`db._create(collection-name, properties [, type], options)`

As an optional third parameter (if the `type` string is omitted) or as fourth
parameter, you can specify an optional options map that controls how the
cluster creates the collection. These options are only relevant at
creation time and are not persisted:

- `waitForSyncReplication` (default: `true`)
  If enabled, the server only reports success back to the client
  if all replicas have created the collection. Set to `false` if you want faster
  server responses and don't care about full replication.

- `enforceReplicationFactor` (default: `true`)
  If enabled, the server checks if there are enough replicas
  available at creation time and bails out otherwise. Set to `false` to disable
  this extra check.

**Examples**

With defaults:

```js
---
name: collectionDatabaseCreateSuccess
description: ''
---
var coll = db._create("users");
coll.properties();
~db._drop("users");
```

With properties:

```js
---
name: collectionDatabaseCreateProperties
description: ''
---
var coll = db._create("users", { waitForSync: true });
coll.properties();
~db._drop("users");
```

With a key generator:

```js
---
name: collectionDatabaseCreateKey
description: ''
---
var coll = db._create("users", { keyOptions: { type: "autoincrement", offset: 10, increment: 5 } });
db.users.save({ name: "user 1" });
db.users.save({ name: "user 2" });
db.users.save({ name: "user 3" });
~db._drop("users");
```

With a special key option:

```js
---
name: collectionDatabaseCreateSpecialKey
description: ''
---
var coll = db._create("users", { keyOptions: { allowUserKeys: false } });
db.users.save({ name: "user 1" });
db.users.save({ name: "user 2", _key: "myuser" }); // xpError(ERROR_ARANGO_DOCUMENT_KEY_UNEXPECTED)
db.users.save({ name: "user 3" });
~db._drop("users");
```

### `db._createDocumentCollection(collection-name [, properties])`

See [`db._create(collection-name [, properties])`](#db_createcollection-name--properties--type--options).

### `db._createEdgeCollection(collection-name [, properties])`

`db._createEdgeCollection(collection-name)`

Creates a new edge collection named `collection-name` with the default settings
and returns a [_collection_ object](collection-object.md).

If a collection or View with this name exists already, an error is thrown.

---

`db._createEdgeCollection(collection-name, properties)`

Creates a new edge collection with the specified properties.
See [`db._create(collection-name, properties)`](#db_createcollection-name--properties--type--options)
for the available properties.

### `db._collections()`

Returns all collections of the current database.
Each array element is a [_collection_ object](collection-object.md).

**Examples**

```js
---
name: collectionsDatabaseName
description: ''
---
~db._create("example");
db._collections();
~db._drop("example");
```

### `db._collection(collection)`

`db._collection(collection-name)`

Returns the collection with the specified name as a [_collection_ object](collection-object.md),
or `null` if no such collection exists.

---

`db._collection(collection-identifier)`

Returns the collection with the given identifier as a [_collection_ object](collection-object.md),
or `null` if no such collection exists.

{{< warning >}}
Accessing collections by identifier is discouraged for end-users.
Access collections using the collection name instead.
{{< /warning >}}

**Examples**

Get a collection by name:

```js
---
name: collectionDatabaseNameKnown
description: ''
---
db._collection("demo");
```

Get a collection by identifier:

```
arangosh> db._collection(123456);
[ArangoCollection 123456, "demo" (type document, status loaded)]
```

Unknown collection:

```js
---
name: collectionDatabaseNameUnknown
description: ''
---
db._collection("unknown");
```

### `db._truncate(collection)`

Truncates a `collection`, removing all documents but keeping all its
index definitions and other settings.

---

`db._truncate(collection-name)`

Truncates a collection named `collection-name`. No error is thrown if
there is no such collection.

---

`db._truncate(collection-identifier)`

Truncates a collection identified by `collection-identified`. No error is
thrown if there is no such collection.

**Examples**

Truncates a collection:

```js
---
name: collectionDatabaseTruncateByObject
description: ''
---
~db._create("example");
var coll = db._collection("example");
var doc = coll.save({ "Hello" : "World" });
coll.count();
db._truncate(coll);
coll.count();
~db._drop("example");
```

Truncates a collection identified by name:

```js
---
name: collectionDatabaseTruncateName
description: ''
---
~db._create("example");
var coll = db._collection("example");
var doc = coll.save({ "Hello" : "World" });
coll.count();
db._truncate("example");
coll.count();
~db._drop("example");
```

### `db._drop(collection [, options])`

Drops a `collection` and all its indexes and data.

`db._drop(collection-name)`

Drops a collection named `collection-name` and all its indexes. No error
is thrown if there is no such collection.

---

`db._drop(collection-identifier)`

Drops a collection identified by `collection-identifier` with all its
indexes and data. No error is thrown if there is no such collection.

---

`db._drop(collection-name, options)`

In order to drop a system collection, you must specify an `options` object
with attribute `isSystem` set to `true`. Otherwise, it is not possible to
drop system collections.

{{< info >}}
Collections in a cluster deployment that are prototypes for collections
with `distributeShardsLike` parameter  cannot be dropped.
{{< /info >}}

**Examples**

Drops a collection:

```js
---
name: collectionDatabaseDropByObject
description: ''
---
~db._create("example");
var coll = db._collection("example");
db._drop(coll);
~db._drop("example");
```

Drops a collection identified by name:

```js
---
name: collectionDatabaseDropName
description: ''
---
~db._create("example");
coll = db._collection("example");
db._drop("example");
coll;
```

Drops a system collection

```js
---
name: collectionDatabaseDropSystem
description: ''
---
~db._create("_example", { isSystem: true });
var coll = db._example;
db._drop("_example", { isSystem: true });
```

## Documents

### `db._exists(document)`

`db._exists(object)`

Checks whether a document exists using an object containing an `_id` attribute.

An error is thrown if a `_rev` attribute is specified but the found document
has a different revision.

Instead of returning the found document or an error, this method
only returns an object with the attributes `_id`, `_key` and `_rev`, or
`false` if no document with the given `_id` or `_key` exists. It can
thus be used for easy existence checks.

This method throws an error if used improperly, e.g. if called
with a string that isn't a document identifier, or an object with an invalid or
missing `_id` attribute.

---

`db._exists(document-identifier)`

Checks whether a document exists using a document identifier.

### `db._update(document, data [, options])`

`db._update(object, data)`

Updates an existing document described by the `object`, which must
be an object containing the `_id` attribute. There must be
a document with that `_id` in the current database. This
document is then patched with the `data` given as second argument.
Any attribute `_id`, `_key` or `_rev` in `data` is ignored.

The method returns a document with the attributes `_id`, `_key`, `_rev`
and `_oldRev`. The attribute `_id` contains the document identifier of the
updated document, the attribute `_rev` contains the document revision of
the updated document, the attribute `_oldRev` contains the revision of
the old (now updated) document.

If the object contains a `_rev` attribute, the method first checks
that the specified revision is the current revision of that document.
If not, there is a conflict, and an error is thrown.

---

`db._update(object, data, options)`

Updates an existing document with additional boolean `options` passed via an object:

  - `waitForSync`: One can force
    synchronization of the document creation operation to disk even in
    case that the `waitForSync` flag is been disabled for the entire
    collection. Thus, the `waitForSync` option can be used to force
    synchronization of just specific operations. To use this, set the
    `waitForSync` parameter to `true`. If the `waitForSync` parameter
    is not specified or set to `false`, then the collection's default
    `waitForSync` behavior is applied. The `waitForSync` parameter
    cannot be used to disable synchronization for collections that have
    a default `waitForSync` value of `true`.
  - `overwrite`: If this flag is set to `true`, a `_rev` attribute in
    the selector is ignored.
  - `returnNew`: If this flag is set to `true`, the complete new document
    is returned in the output under the attribute `new`.
  - `returnOld`: If this flag is set to `true`, the complete previous
    revision of the document is returned in the output under the 
    attribute `old`.
  - `silent`: If this flag is set to `true`, no output is returned.
  - `keepNull`: The optional `keepNull` parameter can be used to modify 
    the behavior when handling `null` values. Normally, `null` values
    are stored in the database. By setting the `keepNull` parameter to
    `false`, this behavior can be changed so that top-level attributes and
    sub-attributes in `data` with `null` values are removed from the target
    document (but not attributes of objects that are nested inside of arrays).
  - `mergeObjects`: Controls whether objects (not arrays) will be 
    merged if present in both the existing and the patch document. If
    set to `false`, the value in the patch document will overwrite the
    existing document's value. If set to `true`, objects will be merged.
    The default is `true`.

---

`db._update(document-identifier, data)`

`db._update(document-identifier, data, options)`

Updates an existing document described by a document identifier, optionally
with additional boolean options (see above).

No revision check is performed.

**Examples**

Create and update a document:

```js
---
name: documentDocumentUpdate
description: ''
---
~db._create("example");
a1 = db.example.insert({ a : 1 });
a2 = db._update(a1, { b : 2 });
a3 = db._update(a1, { c : 3 }); // xpError(ERROR_ARANGO_CONFLICT);
~db._drop("example");
```

Ignore a revision mismatch when updating the document:

```js
---
name: documentDocumentUpdateOverwrite
description: ''
---
~db._create("example");
a1 = db.example.insert({ a : 1 });
a2 = db._update(a1, { b : 2 });
a3 = db._update(a1, { c : 3 }, { overwrite: true });
~db._drop("example");
```

### `db._replace(document, data)`

`db._replace(object, data)`

Replaces an existing document described by the `object`, which must
be an object containing the `_id` attribute. There must be
a document with that `_id` in the current database. This
document is then replaced with the `data` given as second argument.
Any attribute `_id`, `_key` or `_rev` in `data` is ignored.

The method returns a document with the attributes `_id`, `_key`, `_rev`
and `_oldRev`. The attribute `_id` contains the document identifier of the
updated document, the attribute `_rev` contains the document revision of
the updated document, the attribute `_oldRev` contains the revision of
the old (now replaced) document.

If the object contains a `_rev` attribute, the method first checks
that the specified revision is the current revision of that document.
If not, there is a conflict, and an error is thrown.

---

`collection.replace(object, data, options)`

Replaces an existing document, with additional boolean `options` passed via an object:

  - `waitForSync`: One can force
    synchronization of the document creation operation to disk even in
    case that the `waitForSync` flag is been disabled for the entire
    collection. Thus, the `waitForSync` option can be used to force
    synchronization of just specific operations. To use this, set the
    `waitForSync` parameter to `true`. If the `waitForSync` parameter
    is not specified or set to `false`, then the collection's default
    `waitForSync` behavior is applied. The `waitForSync` parameter
    cannot be used to disable synchronization for collections that have
    a default `waitForSync` value of `true`.
  - `overwrite`: If this flag is set to `true`, a `_rev` attribute in
    the selector is ignored.
  - `returnNew`: If this flag is set to `true`, the complete new document
    is returned in the output under the attribute `new`.
  - `returnOld`: If this flag is set to `true`, the complete previous
    revision of the document is returned in the output under the 
    attribute `old`.
  - `silent`: If this flag is set to `true`, no output is returned.

---

`db._replace(document-identifier, data)`

`db._replace(document-identifier, data, options)`

Replaces an existing document described by a document identifier, optionally
with boolean options (see above).

No revision check is performed.

**Examples**

Create and replace a document:

```js
---
name: documentsDocumentReplace
description: ''
---
~db._create("example");
a1 = db.example.insert({ a : 1 });
a2 = db._replace(a1, { a : 2 });
a3 = db._replace(a1, { a : 3 });  // xpError(ERROR_ARANGO_CONFLICT);
~db._drop("example");
```

Ignore a revision mismatch when replacing the document:

```js
---
name: documentsDocumentReplaceOverwrite
description: ''
---
~db._create("example");
a1 = db.example.insert({ a : 1 });
a2 = db._replace(a1, { a : 2 });
a3 = db._replace(a1, { a : 3 }, { overwrite: true });
~db._drop("example");
```

### `db._remove(document)`

`db._remove(object)`

Removes a document described by the `object`, which must be an object
containing the `_id` attribute. There must be a document with
that `_id` in the current database. This document is then
removed.

The method returns a document with the attributes `_id`, `_key` and `_rev`.
The attribute `_id` contains the document identifier of the
removed document, the attribute `_rev` contains the document revision of
the removed document.

If the object contains a `_rev` attribute, the method first checks
that the specified revision is the current revision of that document.
If not, there is a conflict, and an error is thrown.

---

`db._remove(object, options)`

Removes a document, with additional boolean `options` passed via an object:

  - `waitForSync`: One can force
    synchronization of the document creation operation to disk even in
    case that the `waitForSync` flag is been disabled for the entire
    collection. Thus, the `waitForSync` option can be used to force
    synchronization of just specific operations. To use this, set the
    `waitForSync` parameter to `true`. If the `waitForSync` parameter
    is not specified or set to `false`, then the collection's default
    `waitForSync` behavior is applied. The `waitForSync` parameter
    cannot be used to disable synchronization for collections that have
    a default `waitForSync` value of `true`.
  - `overwrite`: If this flag is set to `true`, a `_rev` attribute in
    the selector is ignored.
  - `returnOld`: If this flag is set to `true`, the complete previous
    revision of the document is returned in the output under the 
    attribute `old`.
  - `silent`: If this flag is set to `true`, no output is returned.

---

`db._remove(document-identifier)`

`db._remove(document-identifier, options)`

Removes an existing document described by a document identifier, optionally
with additional boolean options (see above).

No revision check is performed.

**Examples**

Remove a document:

```js
---
name: documentsCollectionRemoveSuccess
description: ''
---
~db._create("example");
a1 = db.example.insert({ a : 1 });
db._remove(a1);
db._remove(a1); // xpError(ERROR_ARANGO_DOCUMENT_NOT_FOUND);
db._remove(a1, {overwrite: true}); // xpError(ERROR_ARANGO_DOCUMENT_NOT_FOUND);
~db._drop("example");
```

Remove the document in the revision `a1` with a conflict:

```js
---
name: documentsCollectionRemoveConflict
description: ''
---
~db._create("example");
a1 = db.example.insert({ a : 1 });
a2 = db._replace(a1, { a : 2 });
db._remove(a1); // xpError(ERROR_ARANGO_CONFLICT)
db._remove(a1, {overwrite: true});
db._document(a1); // xpError(ERROR_ARANGO_DOCUMENT_NOT_FOUND)
~db._drop("example");
```

Remove a document using a document identifier:

```js
---
name: documentsCollectionRemoveIdentifier
description: ''
---
~db._create("example");
db.example.insert({ _key: "123456", a: 1 } );
db.example.remove("example/123456");
~db._drop("example");
```

## Views

### `db._createView(name, type [, properties])`

Creates a new View and returns a [_view_ object](view-object.md).

`name` is a string and the name of the View. No View or collection with the
same name may already exist in the current database. For information about the
naming constraints for Views, see [View names](../../../concepts/data-structure/views.md#view-names).

`type` must be the string `"arangosearch"`, as it is currently the only
supported View type.

`properties` is an optional object containing View configuration specific
to each View-type.
- [`arangosearch` View definition](../../../index-and-search/arangosearch/arangosearch-views-reference.md#view-definitionmodification)
- [`search-alias` View definition](../../../index-and-search/arangosearch/search-alias-views-reference.md#view-definition)

**Examples**

```js
---
name: viewDatabaseCreate
description: ''
---
var view = db._createView("example", "arangosearch");
view.properties()
db._dropView("example")
```

### `db._views()`

Returns all Views of the current database.

Each element of the returned array is a [_view_ object](view-object.md).

**Examples**

List all Views:

```js
---
name: viewDatabaseList
description: ''
---
~db._createView("exampleView", "arangosearch");
db._views();
~db._dropView("exampleView");
```

### `db._view(view)`

`db._view(view-name)`

Returns the View with the given name as a [_view_ object](view-object.md),
or `null` if no such View exists.

```js
---
name: viewDatabaseGet
description: ''
---
~db._createView("example", "arangosearch", {});
var view = db._view("example");
// or, alternatively
var view = db["example"];
~db._dropView("example");
```

---

`db._view(view-identifier)`

Returns the View with the given identifier as a [_view_ object](view-object.md),
or `null` if no such View exists.

{{< warning >}}
Accessing Views by identifier is discouraged for end-users.
Access Views using the View name instead.
{{< /warning >}}

**Examples**

Get a View by name:

```js
---
name: viewDatabaseNameKnown
description: ''
---
db._view("demoView");
```

Unknown View:

```js
---
name: viewDatabaseNameUnknown
description: ''
---
db._view("unknown");
```

### `db._dropView(view)`

`db._dropView(name)`

Drops a view named `name` and all its data.

No error is thrown if there is no such View.

---

`db._dropView(view-identifier)`

Drops a View identified by `view-identifier` with all its data.
No error is thrown if there is no such View.

**Examples**

Drop a view:

```js
---
name: viewDatabaseDrop
description: ''
---
var view = db._createView("exampleView", "arangosearch");
db._dropView("exampleView");
db._view("exampleView");
```

## AQL

### `db._createStatement(queryString)`

See [`db._createStatement()`](../../../aql/how-to-invoke-aql/with-arangosh.md#with-db_createstatement-arangostatement).

### `db._query(queryString [, bindVars [, mainOptions] [, subOptions]])`

See [`db._query()`](../../../aql/how-to-invoke-aql/with-arangosh.md#with-db_query).

### `db._explain(queryString)`

See [`db._explain()`](../../../aql/execution-and-performance/explaining-queries.md).

### `db._parse(queryString)`

See [`db._parse()`](../../../aql/how-to-invoke-aql/with-arangosh.md#query-validation-with-db_parse).

### `db._profileQuery(queryString [, bindVars [, options])`

See [`db._profileQuery()`](../../../aql/execution-and-performance/query-profiling.md).

## Indexes

### `db._index(index)`

Fetches an index by identifier.

See [`db._index()`](../../../index-and-search/indexing/working-with-indexes/_index.md#fetching-an-index-by-identifier).

### `db._dropIndex(index)`

Drops an index by identifier.

See [`db._dropIndex()`](../../../index-and-search/indexing/working-with-indexes/_index.md#dropping-an-index-via-a-database-object).

## Transactions

### `db._createTransaction()`

{{< tag "arangosh" >}}

Starts a Stream Transaction.

See [`db._createTransaction()`](../../transactions/stream-transactions.md#create-transaction).

### `db._executeTransaction()`

Executes a JavaScript Transaction.

See [`db._executeTransaction()`](../../transactions/javascript-transactions.md#execute-transaction).

## Global

### `db._compact([options])`

Compacts the entire data, for all databases.

This command can be used to reclaim disk space after substantial data deletions
have taken place. It requires superuser access.

The optional `options` attribute can be used to get more control over the 
compaction. The following attributes can be used in it:

- `changeLevel`: whether or not compacted data should be moved to the minimum
  possible level. The default value is `false`.
- `compactBottomMostLevel`: whether or not to compact the bottommost level of
  data. The default value is `false`.

{{< warning >}}
This command can cause a full rewrite of all data in all databases, which may
take very long for large databases. It should thus only be used with care
and only when additional I/O load can be tolerated for a prolonged time.
{{< /warning >}}

### `db._engine()`

Returns the name of the storage engine used by the server (`rocksdb`), as well
as a list of supported features such as types of indexes.

### `db._engineStats()`

Returns statistics related to the storage engine activity, including figures
about data size, cache usage, etc.

### `db._version()`

Returns the server version string.

Note that this is different to the version of the database.

**Examples**

```js
---
name: dbVersion
description: ''
---
require("@arangodb").db._version();
```

## License

### `db._getLicense()`

{{< tag "arangosh" >}}

Returns the current license.

See [`db._getLicense()`](../../../operations/administration/license-management.md#check-the-license).

### `db._setLicense(licenseString)`

{{< tag "arangosh" >}}

Sets a license.

See [`db._setLicense()`](../../../operations/administration/license-management.md#apply-a-license).
