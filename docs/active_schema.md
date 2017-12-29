---
layout: post
title: active_schema
date: 2017-12-29
---

### Overview:

Active schema is an empty copy of the database we want to replicate. It contains all the tables from the original database but the tables are empty. The point of this is to have a schema copy and to be able to update this copy independent of the main database because we need to have access to the schema that corresponds to a given position in the binlog (which can be different than the current schema of the database we are replicating).

### Initializing the Active Schema
Create new schema named 'testdb_active_schema'.

````SQL
CREATE DATABASE testdb_active_schema;
````

dump the schema:

````
mysqldump --host=localhost --user=test_user --password=test_pass --no-data --single-transaction testdb > schema_dump.sql
````

replace 'testdb.' with 'testdb_active_schema.' in schema_dump.sql, for example in vim:

````VIM
%s/testdb\./testdb_active_schema\./g
````

then import the schema to testdb_active_schema:

````
mysql --host=localhost --user=meta_user --password=meta_pass < schema_dump.sql
````
