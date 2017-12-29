---
layout: post
title: failover_options
date: 2017-12-29
---

### Overview
There are two types of failover that are suported:

- Replicator failover
- MySQL failover

Both types are based on pseudo GTIDs.

### Pseudo GTIDs
Pseudo GTIDs are implemented as a `NoOp` statements that happen every 5 seconds on MySQL master.
Each pseudo GTID statement goes to the binlog and contains a unique id. The statements are of the form:

````sql
drop view if exists `pgtid_meta`.`_pseudo_gtid_hint__asc:5B01A7FA:ECBC11E7B0CC218F:A8312C70`
````

In addition, these unique ids have a property of being ascending so given two Pseudo GTIDs we can always know which one originated before the other.

The script to create these events is given at [pseudo_gtid](https://github.com/mysql-time-machine/mysql-scripts/blob/master/pseudo_gtid.sql)

### Safe Checkpoints
Replicator will keep track of pseudo GTIDs and as soon as all rows that precede a given pseudoGTID are successfully committed, replicator will
store that pGTID as a safe checkpoint in zookeeper.

### Replicator Failover
There are two ways to setup replicator HA.

1. Deploy the replicator as a single container in an orchestrated environments such as Marathon or Kubernetes. These orchestrators can restart it if it crushes.
2. Run two instances of the replicator at the same time, where one is the leader and the other can become the leader if the first one dies.
3. Combine (1) and (2) for maximum high availability.

Once the replicator is started/restarted it will make sure to acquire leadership lock in zookeeper and to continue replication from the last safe check point. In case multiple replicators are started only one will acquire the leadership lock and the other instances will wait until the leader dies (in which case one of them will become the new leader and continue the replication from the last safe check point)

### MySQL failover with pGTID
In case MySQL dies, replicator will die too, and then the standard replicator failover applies. Since the new MySQL slave will be different (with different binlog file names and positions corresponding to the same events) pGTID is used to find
the right binlog filename and position to continue from.

### Known Limitations
In case of MySQL failover, Kafka applier can end up with up to 5 seconds of duplicate rows. Hbase does not have this limitation since the initial batch of duplicate rows will override the same versions in HBase.
