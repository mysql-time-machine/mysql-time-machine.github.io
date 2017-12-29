---
layout: post
title: transitions
date: 2017-12-29
---

### MySQL Transactions Support
Replicator provides information about MySQL transaction ids to allow a reader to understand which events happened as part of the same transaction. A transaction in MySQL binlog looks like:
````
 - BEGIN
 - UPDATE/INSERT/DELETE/etc
 - COMMIT
````
MySQL will provide a xid (transaction id) and a proper timestamp only on a COMMIT event. Timestamps of previous events represent time on an actual query invocation during transaction. Since Replicator reads binary logs as a stream it needs to buffer binlog events until a COMMIT event will be found and then override timestamps of the events. Current implementation creates a new instance of CurrentTransaction.class with a BEGIN event and fills up a buffer with following data-events. When a COMMIT event received, Replicator does timestamps override, sets the xid for the events and forwards them to an Applier.class which writes them to a destination.

Big transactions are a special case. If there were a lot of changes in a single transaction, we can’t buffer all the changes in memory. To support the big changes Replicator implements “rewinding” of a transaction. It doesn’t know if a transaction is going to be big, so it’ll start with buffering as was described above. When a number of events in the transaction exceeds configured amount, Replicator will drop all of the buffered events and will read a binlog stream until a COMMIT event is reached and saved. Then it’ll stop a BinlogEventProducer.class instance and reset current position to the BEGIN event. Producer will be started for the position and all the following events will be forwarded to an applier right away without buffering, but with timestamps and xids which are stored in the saved COMMIT event. When the COMMIT event reached for the second time that means we’ve applied all of the transaction events and we’re able to switch rewinding-mode off.

There might be two different types of a COMMIT event. First type is XidEvent which contains a valid xid and happens when a transaction modifies one or more XA-capable storages (for ex. InnoDB). Another type is QueryEvent which contains an sql-command “COMMIT” and this type of event created when there were no XA-capable tables involved (ex. all the tables are MyISAM). For a QueryEvent event there’s no xid provided by MySQL. To make that kind of events trackable on a destination data-storage and through log files there’s transactionUUID provided by Replicator which is generated for each transaction during creation.

For the sake of internal simplicity Replicator uses the same pipeline for DDL-statements. For such events it will create a fake BEGIN event, apply an actual query and commit it with a fake COMMIT event.
