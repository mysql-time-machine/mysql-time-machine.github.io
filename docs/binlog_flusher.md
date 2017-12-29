---
layout: post
title: binlog_flusher
date: 2017-12-29
---

### Overview
Binlog Flusher is a python script which flushes MySQL database tables to the binlog in order to have the initial snapshot of the database in the binlog.

### Flushing Database to the Binlog
***WARNING: you should NEVER run binlog-flusher on the MySQL master. Binlog flusher renames all tables during the blackhole copy and if the program does not finish successfully, the database can be left in an inconsistent state and in worst case you will need to reclone the database. The probability of this is low, but still, if it happens, you do NOT want this to happen on MySQL master.***

Assuming you adhere to the above warning, you can flush the contents of a database to the binlog with:

````bash
python data-flusher.py --host $host
                       [--mycnf $mycnf] \
                       [--db $db] [--table $table] \
                       [--stop-slave/--no-stop-slave] [--start-slave/--no-start-slave] \
                       [--skip $skip]

````

Where parameters are:

  * `--host`: host name of the MySQL slave which databases need to be flushed to the binlog
  * `--mycnf`: filename that contains the admin privileges used for the blackhole_copy of initial snapshot. Defaults to ~/my.cnf
  * `--db`: comma separated list of databases to copy. Leave blank for all databases
  * `--table`: comma separated list of tables to copy. Leave blank for all tables
  * `--stop-slave/--no-stop-slave`: stop the replication thread whilst flushing the to the binlog. default=True
  * `--start-slave/--no-start-slave`: restart the replication thread after finishing the flushing to the binlog. default=False
  * `--skip`: separated list of schemas to skip (not to flush in the binlog)

and the configuration file with admin credentials (`--mycnf` parameter above) has the following structure:
````
[client]
user=admin
password=admin
````

After the data has been flushed to the binlog, the replication is stopped by default (see default values for parameters above). You can start the replication with:

````
mysql> start slave;
````

In case binlog flusher didn't exit gracefully and the database has been left in an inconsistent state, you can run db-recovery.py to recover the database.

````bash
python db-recovery.py --host $host \
                      --hashfile $hashfile \
                      [--mycnf .my.cnf] \
                      [--db $db] [--table $table] \
                      [--stop-slave/--no-stop-slave] \
                      [--start-slave/--no-start-slave] \
                      [--skip $skip]
````

Where `--hashfile` parameter contains the name of a rollback file, generated during the flush procedure, that contains the mappings from the backup table names to original table names.

````
_BKTB_1, $tablename1$
_BKTB_2, $tablename2$
....
````
