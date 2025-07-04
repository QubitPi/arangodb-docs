---
title: Upgrading _Starter_ Deployments
menuTitle: Starter Deployments
weight: 15
description: >-
  Procedure to perform a rolling upgrade with the `arangodb` executable
aliases:
  - upgrading-starter-deployments
---
The ArangoDB [_Starter_](../../components/tools/arangodb-starter/_index.md) supports an automated procedure
to perform upgrades, including rolling upgrades
of a [Cluster](../../deploy/cluster/_index.md) setup.

The upgrade procedure of the _Starter_ described in this section can be used to
upgrade to a new hotfix, or to perform an upgrade to a new minor version of ArangoDB.
Please refer to the [Upgrade paths](_index.md#upgrade-paths) section
for detailed information.

## Upgrade Scenarios

The following four cases are possible:

1. You have installed via an installation package (e.g. a `.deb` or `.rpm` package)
   and you will upgrade this installation using again an installation package
   (e.g. a `.deb` or `.rpm`).
2. You have installed via the `.tar.gz` distribution and you will upgrade this
   installation using again a `.tar.gz` distribution.
3. You have installed via an installation package (e.g. a `.deb` or `.rpm` package)
   and you will upgrade this installation using a `.tar.gz` distribution.
4. You have installed via the `.tar.gz` distribution and you will upgrade this
   installation using an installation package (e.g. a `.deb` or `.rpm` package).

Cases 1. and 2. are more common, though cases 3. and 4. are also possible.

## Upgrade Procedure

The following procedure has to be executed on every ArangoDB _Starter_ instance.
It is assumed that a _Starter_ deployment with mode `single` or `cluster` is running.

{{< warning >}}
It is highly recommended to take a backup of your data before upgrading ArangoDB
using [_arangodump_](../../components/tools/arangodump/_index.md).
{{< /warning >}}

### Install the new ArangoDB version binary

Installing the new ArangoDB version binary also includes the latest ArangoDB _Starter_
binary, which is necessary to perform the rolling upgrade.

The first step is to install the new ArangoDB package.

**Note:** you do not have to stop the _Starter_ processes before upgrading it.

For example, if you want to upgrade to `3.9.2` on Debian or Ubuntu, either call

```bash
apt install arangodb=3.9.2
```

(`apt-get` on older versions) if you have added the ArangoDB repository. Or
install a specific package using

```bash
dpkg -i arangodb3-3.9.2-1_amd64.deb
```

after you have downloaded the corresponding file from
[www.arangodb.com/download/](https://www.arangodb.com/download/).

If you are using the `.tar.gz` distribution,
you can simply extract the new archive in a different
location and keep the old installation where it is. Note that
this does not launch a standalone instance, so the following section can
be skipped in this case.

#### Stop the Standalone Instance

As the package will automatically start the standalone instance, you might want to
stop it now, as otherwise this standalone instance that is started on your machine
can create some confusion later. As you are using the _Starter_ you do not need
this standalone instance, and you can hence stop it:

```bash
service arangodb3 stop
```

Also, you might want to remove the standalone instance from the default
_runlevels_ to prevent it to start on the next reboot of your machine. How this
is done depends on your distribution and _init_ system. For example, on older Debian
and Ubuntu systems using a SystemV-compatible _init_, you can use:

```bash
update-rc.d -f arangodb3 remove
```

### Stop the *Starter* without stopping the ArangoDB Server processes

Now all the _Starter_ (_arangodb_) processes have to be stopped.
Please note that **no** _arangod_ processes should be stopped!

In order to stop the _arangodb_ processes, leaving the _arangod_ processes they
have started up and running (as we want for a rolling upgrade), we will need to
use a command like `kill -9`.

{{< tip >}}
When using _SystemD_ as supervisor, make sure that the
[unit file](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)
contains `KillMode=process` (see
[systemd.kill documentation](https://www.freedesktop.org/software/systemd/man/systemd.kill.html#KillMode=)).
Otherwise `kill -9` will not just kill the respective _arangodb_ starter process,
but also the _arangod_ server processes it started because of the
default setting `KillMode=control-group`.
{{< /tip >}}

```bash
kill -9 <pid-of-starter>
```

The _PID_ associated to your _Starter_ can be checked using a command like `ps`:

```bash
ps -C arangodb -fww
```

The output of the command above does not only show the PIDs of all _arangodb_
processes but also the used commands, which can be useful for the following
restart of all _arangodb_ processes.

The output below is from a test machine where three instances of a _Starter_ are
running locally. In a more production-like scenario, you will find only one instance
of _arangodb_ running:

```bash
ps -C arangodb -fww
UID        PID  PPID  C STIME TTY          TIME CMD
max      29419  3684  0 11:46 pts/1    00:00:00 arangodb --starter.data-dir=./db1
max      29504  3695  0 11:46 pts/2    00:00:00 arangodb --starter.data-dir=./db2 --starter.join 127.0.0.1
max      29513  3898  0 11:46 pts/4    00:00:00 arangodb --starter.data-dir=./db3 --starter.join 127.0.0.1
```

You can use `pstree` to inspect the _arangod_ server instances launched by one
of these starters:

```bash
pstree -Tp 29419 
arangodb(29419)─┬─arangod(30201)
                ├─arangod(30202)
                └─arangod(30217)
```

### Restart the *Starter*

When using a supervisor like _SystemD_, this will happen automatically. In case
the _Starter_ was initiated manually, the _arangodb_ processes have to be restarted
manually with the same command that has been used before.

You can inspect which processes belong to a starter instance using the `pstree`
command (see above).

If you are using the `.tar.gz` distribution,
your new version of the executable might be located in a
different directory. Make sure that you now start the new _Starter_
executable (`bin/arangodb`) in the new installation place. If you are
using a supervisor like _SystemD_, you might have to adjust the path to
the executable in the service description to the new location.
Do this before you `kill -9` the _Starter_ or else the old version will be
restarted in this case. If you forgot, simply do the `kill -9` again.

After you stopped the _Starter_ make sure the `arangod` processes it spawned
are still running; they should be re-parented to the systemd or init process:

```bash
ps -f -p 30201,30202,30217
UID          PID    PPID  C STIME TTY          TIME CMD
root     30201       1  0 13:02 pts/45   00:01:09 usr/sbin/arangod ...
root     30202       1  0 13:02 pts/45   00:00:55 usr/sbin/arangod ...
root     30217       1  0 13:02 pts/45   00:01:11 usr/sbin/arangod ...
```

If not, rollback to the old version and restart that _Starter_, or the
subsequent upgrade procedure will fail.

After you have successfully restarted the _Starter_ you will find yourself in
the following situation:

- The _Starter_ is up and running, and it is on the new version
- The ArangoDB Server processes are up and running, and they are still on the
  old version

### Start the upgrade process of all *arangod* servers

Once you have carried out the above steps on all servers of the cluster, the
actual upgrade procedure can be started.

Run the following command on any of the cluster nodes for any of the starter 
endpoints (e.g. `http://localhost:8528`) to upgrade the entire cluster:

```bash
arangodb upgrade --starter.endpoint=<endpoint-of-a-starter>
```

If the upgrade command fails, please try again. If the upgrade command continues
to fail, please contact the ArangoDB support.

#### Deployment mode `single`

For deployment mode `single`, the `arangodb upgrade` command will:

- Restart the single server with an additional `--database.auto-upgrade=true` argument.
  The server will perform the auto-upgrade and then stop.
  After that the _Starter_ will automatically restart it with its normal arguments.

The `arangodb upgrade` command will complete right away.
Inspect the log of the _Starter_ to know when the upgrade has finished.

#### Deployment mode `cluster`

The _Starters_ will now perform an initial check that upgrading is possible
and when that all succeeds, create an upgrade _plan_. This _plan_ is then 
executed by every _Starter_.

The `arangodb upgrade` command will show the progress of the upgrade
and stop when the upgrade has either finished successfully or finished
with an error.

### Uninstall old package

{{< info >}}
This step is required in the cases 2., 3. and 4. only. It is not required
in case 1., see [Upgrade Scenarios](#upgrade-scenarios) above.
{{< /info >}}

After verifying your upgraded ArangoDB system is working, you can remove
the old package. This can be done in different ways, depending on the case
you are:

- Cases 2. and 4.: just remove the old directory created by the `.tar.gz`
  (assumes your `--starter.data-dir` is located outside of this 
  directory - which is a recommended approach).
- Case 3.: just remove the old package by running the corresponding
  uninstallation command (the exact command depends on whether you are
  using a `.deb` or `.rmp` package and it is assumed that your 
  `--starter.data-dir` is located outside of the standard directories
  created by the installation package - which is a recommended approach).

## Retrying a failed upgrade

When an upgrade _plan_ (in deployment mode `cluster`)
has failed, it can be retried.

To retry, run:

```bash
arangodb retry upgrade --starter.endpoint=<endpoint-of-a-starter>
```

The `--starter.endpoint` option can be set to the endpoint of any
of the starters, e.g. `http://localhost:8528`.

## Aborting an upgrade

When an upgrade _plan_ (in deployment mode `cluster`)
is in progress or has failed, it can be aborted.

To abort, run:

```bash
arangodb abort upgrade --starter.endpoint=<endpoint-of-a-starter>
```

The `--starter.endpoint` option can be set to the endpoint of any
of the starters, e.g. `http://localhost:8528`.

Note that an abort does not stop all upgrade processes immediately.
If an _arangod_ server is being upgraded when the abort
was issued, this upgrade will be finished. Remaining servers will not be
upgraded.

## Going back to the previous version in case of failure

In case the upgrade cannot be carried out (i.e. the upgrade command continues 
to fail with an error message) please contact the ArangoDB support.

Until then, the not yet upgraded instances will still be running the previous
version of ArangoDB. When instances are restarted, they will be starting
with the newly installed version.

In order to avoid that instances use the new version upon the
next restart, it is possible to install the previous version again.
The steps for that are the same as for installing the new version (see above)
and making sure that the service for the standalone instance is disabled 
(also see above).
