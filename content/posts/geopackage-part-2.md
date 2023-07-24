---
title: "Geopackage and Python - Part 2"
date: 2023-04-04T21:35:41+01:00
draft: true
---

In the previous post, we took a deep dive into the geopackage format to get a better understanding, before demonstrating how you can create and manage a geopackage file with python using the `sqlite3` and `osgeo` libraries. In this post we will have a play around with triggers - utilising some of the advantages of the sqlite format to automate processing tasks and remove the need to use processing within QGIS.

Let's continue using the geopackage we created last time, but let's add a few fields in to the `points` layer - let's start by creating some auditing fields that track when features are created and updated. For this we can just use the `sqlite3` library from the command line (you could also do this using the DB Manager tool in QGIS):

```shell
$ sqlite3 sample.gpkg
sqlite> alter table points add update_date text;
```

Note that sqlite doesn't have a `DATETIME` data type like you would find in postgres, instead you must use either `TEXT`, `INT` or `REAL`. Also note that sqlite doesn't allow for the addition of new fields with default values that are not constant, else the following would be possible:
```shell
sqlite> alter table points add insert_date text default current_timestamp;
```
instead it will return an error, so we can instead set the default value using a trigger.

For those unfamiliar with SQL triggers - in simple terms they are database objects that are automatically run (i.e. triggered) when a table is altered using either an `UPDATE`, `INSERT` or `DELETE` statements. The syntax is as shown below:

```sql
create trigger [if not exists] trigger_name 
   [before|after|instead of] [insert|update|delete] 
   on table_name
   [when condition]
begin
 statements;
end;
```
so let's do this for our points table.

```shell
sqlite> create trigger datetime_on_insert
sqlite> before insert on points
sqlite> begin
sqlite> update points
sqlite> set insert_date = CURRENT_TIMESTAMP
sqlite> where fid = new.fid
sqlite> end;
```


