---
title: Backup and Restore
menuTitle: Backup & Restore
weight: 215
description: >-
  Physical backups, logical backups with _arangodump_ and _arangorestore_,
  hot backups with _arangobackup_
---
ArangoDB supports three backup methods:

1. Physical (raw or "cold") backups
2. Logical backups
3. Hot backups

These backup methods save the data which is in the database system. In addition,
make sure to backup things like configuration files, startup scripts, Foxx
services, access tokens, secrets, certificates etc. and store them in a
different location securely.

Performing frequent backups is important and a recommended best practices that
can allow you to recover your data in case unexpected problems occur.
Hardware failures, system crashes, or users mistakenly deleting data can always
happen. Furthermore, while a big effort is put into the development and testing
of ArangoDB (in all its deployment modes), ArangoDB, as any other software
product, might include bugs or errors and data loss could occur.
It is therefore important to regularly backup your data to be able to recover
and get up and running again in case of serious problems.

Creating backups of your data before an ArangoDB upgrade is also a best practice.

{{< warning >}}
Making use of a high availability deployment modes of ArangoDB
does not remove the need of
taking frequent backups, which are recommended also when using such deployment modes.
{{< /warning >}}

## Physical backups

Physical (raw or "cold") backups can be done when the ArangoDB Server is not running
by making a raw copy of the ArangoDB data directory.

Such backups are extremely fast as they only involve file copying.

If ArangoDB runs in Cluster mode, it is necessary to copy the data directories
of all the involved processes (_Agents_, _Coordinators_ and _DB-Servers_).

{{< warning >}}
It is extremely important that physical backups are taken only after all the ArangoDB
processes have been shut down and the processes are not running anymore.
Otherwise files might still be written to, likely resulting in a corrupt and incomplete backup.
{{< /warning >}}

It is not always possible to take a physical backup as this method requires a shutdown
of the ArangoDB processes. However in some occasions such backups are useful, often
in conjunction to the backup coming from another backup method.

## Logical Backups

Logical backups can be created and restored with the tools
[_arangodump_](../components/tools/arangodump/_index.md) and
[_arangorestore_](../components/tools/arangorestore/_index.md).

## Hot Backups

Hot backup and restore associated operations can be performed with the
[_arangobackup_](../components/tools/arangobackup/_index.md) client tool and the
[Hot Backup HTTP API](../develop/http-api/hot-backups.md).

Many operations cannot afford downtimes and thus require administrators and
operators to create consistent freezes of the data during normal operation.
Such use cases imply that near instantaneous hot backups must be
obtained in sync across say a cluster's deployment. For this purpose the
hot backup mechanism was created.

The process of creating hot backups is ideally an instantaneous event during
normal operations, that consists of a few subsequent steps behind the scenes:

- Stop all write accesses to the entire installation using a write transaction lock.
- Create a new local directory under `<data-dir>/backups/<timestamp>_<backup-label>`.
- Create hard links to the active database files in `<data-dir>` in the newly
  created backup directory.
- Release the write transaction lock to resume normal operation.
- Report success of the operation.

The above quite precisely describes the tasks in a single instance installation
and could technically finish in under a millisecond. The unknown factor above is
of course, when the hot backup process is able to obtain the write transaction lock.

When considering the ArangoDB cluster two more steps need to integrate while
others just become slightly more exciting. On the Coordinator tasked with the
hot backup the following is done:

- Using the Agency, make sure that no two hot backups collide.
- Obtain a dump of the Agency's `Plan` key.
- Stop all write access to the **entire cluster** installation using a
  global write transaction lock, this amounts to get each local write
  transaction lock on each DB-Server, all at the same time.
- Getting all the locks on the DB-Servers is tried using subsequently growing
  time periods, and if not all local locks can be acquired during a period,
  all locks are released again to allow writes to continue. If it is not
  possible to acquire all local locks in the same period, and this continues
  for an extended, configurable amount of time, the Coordinator gives
  up. With the `allowInconsistent` option set to `true`, it proceeds instead
  to create a potentially non-consistent hot backup.
- **On each DB-Server** create a new local directory under
  `<data-dir>/backups/<timestamp>_<backup-label>`.
- **On each DB-Server** create hard links to the active database files
  in `<data-dir>` in the newly created backup directory.
- **On each DB-Server** store a redundant copy of the above Agency dump.
- Release the global write transaction lock to resume normal operation.
- Report success of the operation.

Again under good conditions, a complete hot backup could be obtained from a
cluster with many DB-Servers within a very short time in the range
of that of the single server installation.

### Technical Details

- **The Global Write Transaction Lock**

  To create a consistent snapshot of an ArangoDB single server or
  cluster deployment, all transactions need to be suspended in order for the
  state of a deployment to be consistent. However, there is no way for ArangoDB
  to know by its own when this time comes. This is why a hot backup needs to
  acquire a global write transaction lock in order to create the backup in a
  consistent state.

  On a single server instance, this lock is eventually obtained and the hot
  backup is then created within a very short amount of time.

  However, in a cluster, this process is more complex. One Coordinator tries to
  obtain the global write transaction lock on all _DB-Servers_ simultaneously.
  Depending on the activity in the cluster, it can take some time for the
  Coordinator to acquire all the locks the cluster needs. Grabbing all the
  necessary locks at once might not always be successful, leading to times 
  when it seems like the cluster's write operations are suspended.

  This process can happen multiple times until all locks are obtained.
  The system administrator has control over the length of the time during which
  the lock is tried to be obtained each time, prolonging the last wait time by
  10% (which gives more time for the global write transaction lock to resolve).

- **Agency Lock**

  Less of a variable, however equally important is to obtain a freeze on the
  cluster's structure itself. This is done through the creation of a simple key
  lock in the cluster's configuration to stop all ongoing background tasks,
  which are there to handle fail overs, shard movings, server removals etc.
  Its role is also to prevent multiple simultaneous hot backup operations.
  The acquisition of this key is predictably done within a matter of a few seconds.

- **Operation's Time Scope**

  Once the global write transaction lock is obtained, everything goes very quickly.
  A new backup directory is created, the write ahead lock is flushed and
  hard links are made on file system level to all persistent files.
  The duration is not affected by the amount of data in ArangoDB and is near
  instantaneous.

- **Point in Time Recovery**

  One of the great advantages of the method is the consistent snapshot nature.
  It gives the operator of the database the ability to persist a true and
  complete time freeze at near zero impact on the ongoing operation.
  The recovery is easy and restores the entire ArangoDB installation to a
  desired snapshot.

  Apart from the ability of creating such snapshots it offers a great and easy
  to use opportunity to experiment with ArangoDB with a means to protect
  against data loss or corruption.

- **Remote Upload and Download**

  We have fully integrated the
  [rclone](https://rclone.org/) sync for cloud storage. Rclone is a very
  versatile inter site sync facility, which opens up a vast field of transport
  protocols and remote syncing APIs from Amazon's S3 over Dropbox, WebDAV,
  all the way to the local file system and network storage.

  One can use the upload and download functionalities to migrate entire cluster
  installations in this way, copy cluster and single server snapshots all
  over the world, create an intuitive and easy to use quick access safety
  backbone of the data operation. 

  Rclone is open source and available under the MIT license, is battle tested
  and has garnered close to 15k stars on GitHub professing to the confidence
  of lots of users.

### Hot Backup Limitations

ArangoDB hot backups impose limitations with respect to storage engine,
storage usage, upgrades, deployment scheme, etc. Please review the below
list of limitations closely to conclude which operations it might or might
not be suited for.

- **Global Scope**

  In order to be able to create hot backups instantaneously, they are created
  on the file system level and thus well below any structural entity related to
  databases, collections, indexes, users, etc.

  As a consequence, a hot backup is a backup of the entire ArangoDB single server
  or cluster. In other words, one cannot restore to an older hot backup of a
  single collection or database. With every restore, one restores the entire
  deployment including of course the `_system` database.

  Note that this applies in particular in the case that a certain user
  might have admin access for the `_system` database, but explicitly has
  no access to certain collections. The backup still extends across
  **all** collections!

  {{< danger >}}
  A restore to an earlier hot backup snapshot also reverts users, graphs,
  Foxx apps - everything - back to that at the time of the hot backup!
  {{< /danger >}}

- **Cluster's Special Limitations**

  Creating hot backups can only be done while the internal structure of the
  cluster remains unaltered. The background of this limitation lies in the
  distributed nature and the asynchronicity of creation, alteration and
  dropping of cluster databases, collections and indexes.

  It must be ensured that for the hot backup no such changes are made to the
  cluster's inventory, as this could lead to inconsistent hot backups.

- **Restoring from a different version**

  Hot backups share the same limitations with respect to different versions
  as ArangoDB itself. This means that a hot backup created with some version
  `a.b.c` can without any limitations be restored on any version `a.b.d` with
  `d` not equal to `c`, that is, the patch level can be changed arbitrarily.
  With respect to minor versions (second number, `b`), one can only upgrade
  and **not downgrade**. That is, a hot backup created with a version `a.b.c`
  can be restored on a version `a.d.e` for `d` greater than `b` but not for `d`
  less than `b`. At this stage, we do not guarantee any compatibility between
  versions with a different major version number (first number).

- **Identical Topology**

  Unlike dumps created with [_arangodump_](../components/tools/arangodump/_index.md) and restored
  with [_arangorestore_](../components/tools/arangorestore/_index.md),
  hot backups can only be restored to the same type and structure of deployment.
  This means that one cannot restore a 3-node ArangoDB cluster's hot backup to
  any other deployment than another 3-node ArangoDB cluster of the same version.

- **Storage Space**

  Without the creation of hot backups, RocksDB keeps compacting the file system
  level files as the operation continues. Compacted files are subsequently
  deleted automatically. Every hot backup needs to hold on to the
  files as they were at the moment of the hot backup creation, thus preventing
  the deletions and consequently growing the storage space of the ArangoDB
  data directory. That growth of course depends on the amount of write operations
  per time.

  This is a crucial factor for sustained operation and might require
  significantly higher storage reservation for ArangoDB instances involved and
  a much more fine grained monitoring of storage usage than before.

  Also note that in a cluster, each RocksDB instance is backed up
  individually and hence the overall storage space is the sum of all
  RocksDB instances (i.e., data which is replicated between instances is
  not de-duplicated for performance reasons).

- **Global Transaction Lock**

  In order to be able to create consistent hot backups, it is mandatory to get
  a very brief global transaction lock across the entire installation.
  In single server deployments, constant invocation of very long running
  transactions could prevent that from ever happening during a timeout period.
  The same holds true for clusters, where this lock must now be obtained on all
  DB-Servers at the same time.

  Especially in the cluster, the result of these successively longer tries to
  obtain the global transaction lock might become visible in periods of apparent
  dead time. Locks might be obtained on some machines and and not on others, so
  that the process has to be retried over and over. Every unsuccessful try would
  then lead to the release of all partial locks.

  {{< info >}}
  The _arangobackup_ tool provides a `--force` option
  that can be used to abort ongoing write transactions and thus to more quickly
  obtain the global transaction lock.
  {{< /info >}}

  At this stage, index creation constitutes a write transactions, which means
  that during index creation one cannot create a hot backup. We intend to lift
  this limitation in a future version.
  
- **Services on Single Server**

  On a single server, the installed Foxx microservices are not backed up and are
  therefore also not restored. This is because in single server mode
  the service installation is done locally in the file system and does not
  track the information in the `_apps` collection.

  In a cluster, the Coordinators eventually restore the state of the
  services from the `_apps` and `_appbundles` collections after a backup is
  restored.

- **Encryption at Rest**

  The hot backup simply takes a snapshot of the database files.
  If you use encryption at rest, then the backed up files are
  encrypted, with the encryption key that has been used in the
  instance which created the backup.

  Such an encrypted backup can only be restored to an instance using the
  same encryption key.

- **Known Issues**

  See the list of [Known Issues](../release-notes/version-3.12/known-issues-in-3-12.md#hot-backup).
