---
aliases:
- /releases/4.0/node-operations-database-schema-setup.html
date: '2020-01-08T09:59:25Z'
menu:
- corda-enterprise-4-0
tags:
- node
- operations
- database
- schema
- setup
title: Database schema setup
---


# Database schema setup

The page describes how to create or update database schema content for a Corda node.
Ensure database permissions are set as described in [Node database](node-database.md) before proceeding.

Database schema (database objects like tables, indexes) can be created and updated in various ways
depending on your development processes:


* Automatically by a Corda node upon startup
* Automatically by a database administrator using [Corda Database Management Tool](database-management.md#database-management-tool-ref)
* Manually by a database administrator with DDL scripts generated by [Corda Database Management Tool](database-management.md#database-management-tool-ref)

You may not want a Corda node to alter the schema directly e.g. this is a database administrator responsibility.
[Corda Database Management Tool](database-management.md#database-management-tool-ref) can be used to generate the DDL
that a database administrator would them apply following their own operational processes.

Corda database schema management applies to both internal node tables and custom tables required by CorDapps.
The only exception is an upgrade of Cordapps running on a node with a H2 database,
in which case, any alteration to existing columns needs to be run manually.

Corda Enterprise requires all Cordapp to contain data migration scripts for their custom tables.
Because of that the best is to setup a database in one go for a Corda node and for custom tables required by Cordapps.

The three different Corda database management strategies are described below.



## Database schema management upon node startup

A Corda node can create and update the database schema upon startup.
A Corda node requires *administrative* permissions to execute this strategy.
To allow the node to auto create/upgrade schema add `runMigration` option in `node.conf`:

> 
> ```groovy
> database {
>     runMigration = true
>     # ...
> }
> ```
> 

Each time a Corda node is upgraded to a new release, it updates the schema upon startup automatically.
For new or upgraded CorDapps, the node needs to be restarted to apply any schema changes related to a Cordapps custom tables.



## Database Management Tool applies schema changes directly

[Corda Database Management Tool](database-management.md#database-management-tool-ref) can connect directly to a database to create/update the schema.

> 
> {{< note >}}
> Database Management Tool creates/upgrades a single database schema only.
> The setup process needs to be repeated for each node with respective database credentials and node config.
> 
> {{< /note >}}


The easiest way to you use the tool is to provide access to a Corda node base directory, so it can reuse
the database URL/credentials from a Corda node `node.conf` file and CorDapps.
You can also supply your own configuration file e.g. when the tool is run from within a different machine
and there’s no access to a Corda node installation directory.
The file needs to have the following configuration entries, any surplus options are ignored:

> 
> ```none
> myLegalName = <Node legal name>
> dataSourceProperties = {
>     dataSourceClassName = <JDBC Data Source class name>
>     dataSource.url = <JDBC database URL>
>     dataSource.user = <Database user>
>     dataSource.password = <Database password>
> }
> database = {
>     transactionIsolationLevel = <Transaction isolation level>
>     schema = <Database schema name>
> }
> ```
> 

Refer to [Node Configuration](node-database.md#db-setup-vendors-ref) for your database specific options.

> 
> 
> {{< warning >}}
> Applying schema changes for some Corda releases (e.g. Corda 4.0) requires data rows migration.
> For such migration `myLegalName` must exactly match the node name which is using the given database schema.
> Any misconfiguration may cause data rows migration to be wrongly applied without any error.
> 
> {{< /warning >}}
> 
> 

The JDBC driver needs to be added to the *drivers* subdirectory.
If you are providing separate directory for the tool ensure to
copy CorDapps to the *cordapps* subdirectory, this is required to collect
and run any database migrations scripts for Cordapps.
A given Corda release may require to put any used CorDapps *cordapps* subdirectory in case for data rows updates.

Ensure the Corda node using the schema to be upgraded is shut down before running the tool.

> 
> ```shell
> java -jar tools-database-manager-RELEASE-VERSION.jar execute-migration -b .
> ```
> 

The option `-b` denotes the directory containing the *node.conf* file
and both subdirectories *drivers* and *cordapps*.
Refer to [the tool usage](database-management.md#database-management-tool-ref) for available command line options.

You need to repeat the process when the Corda node is upgraded
or the new/upgraded Cordapp (having custom table changes) is deployed in to the node.



## Database Management Tool generates DDL script to be run manually

This is the most controlled way to create a database schema and allows to audit the DDL scripts
([Data Definition Language](https://en.wikipedia.org/wiki/Data_definition_language) scripts)
before executing it on a database.

> 
> {{< note >}}
> Database tool can create/upgrade a single database schema only.
> The setup process needs to be repeated for each node with respective
> database credentials and node config.
> 
> {{< /note >}}


### 1. Essential preparation before the first installation

This step may be required when creating a new schema only.
Skip the step if you are upgrading the schema which already was in use by a Corda node.
Corda Database Management Tool interacts with a running database and needs two control tables
*DATABASECHANGELOG* and *DATABASECHANGELOGLOCK* to be present.
Those tables are created automatically if they are missing and the tool connects to a database
with *administrative* permissions (e.g. it can create tables).
Alternatively, the database administrator needs to create these 2 tables manually.

Create the following tables only if the database is empty and database management tool connects
with *restrictive* database permissions.
Replace schema namespace *my_schema* with the schema used by a Corda node.

Script for Azure SQL and SQL Server:

> 
> ```sql
> CREATE TABLE my_schema.DATABASECHANGELOG (
> ID nvarchar(255) NOT NULL,
> AUTHOR nvarchar(255) NOT NULL,
> FILENAME nvarchar(255) NOT NULL,
> DATEEXECUTED datetime2(3) NOT NULL,
> ORDEREXECUTED int NOT NULL,
> EXECTYPE nvarchar(10) NOT NULL,
> MD5SUM nvarchar(35) NULL,
> DESCRIPTION nvarchar(255) NULL,
> COMMENTS nvarchar(255) NULL,
> TAG nvarchar(255) NULL,
> LIQUIBASE nvarchar(20) NULL,
> CONTEXTS nvarchar(255) NULL,
> LABELS nvarchar(255) NULL,
> DEPLOYMENT_ID nvarchar(10) NULL
> );
> CREATE TABLE my_schema.DATABASECHANGELOGLOCK (
> ID int NOT NULL,
> LOCKED bit NOT NULL,
> LOCKGRANTED datetime2(3) NULL,
> LOCKEDBY nvarchar(255) NULL,
> CONSTRAINT PK_DATABASECHANGELOGLOCK PRIMARY KEY (ID)
> );
> ```
> 

Script for Oracle, change the *users* tablespace if necessary:

> 
> ```sql
> CREATE TABLE my_user."DATABASECHANGELOG" (
> "ID" VARCHAR2(255) NOT NULL ENABLE,
>     "AUTHOR" VARCHAR2(255) NOT NULL ENABLE,
>     "FILENAME" VARCHAR2(255) NOT NULL ENABLE,
>     "DATEEXECUTED" TIMESTAMP (6) NOT NULL ENABLE,
>     "ORDEREXECUTED" NUMBER(*,0) NOT NULL ENABLE,
>     "EXECTYPE" VARCHAR2(10) NOT NULL ENABLE,
>     "MD5SUM" VARCHAR2(35),
>     "DESCRIPTION" VARCHAR2(255),
>     "COMMENTS" VARCHAR2(255),
>     "TAG" VARCHAR2(255),
>     "LIQUIBASE" VARCHAR2(20),
>     "CONTEXTS" VARCHAR2(255),
>     "LABELS" VARCHAR2(255),
>     "DEPLOYMENT_ID" VARCHAR2(10)
> ) TABLESPACE users;
> CREATE TABLE my_user."DATABASECHANGELOGLOCK" (
> "ID" NUMBER(*,0) NOT NULL ENABLE,
>     "LOCKED" NUMBER(1,0) NOT NULL ENABLE,
>     "LOCKGRANTED" TIMESTAMP (6),
>     "LOCKEDBY" VARCHAR2(255),
>     CONSTRAINT "PK_DATABASECHANGELOGLOCK" PRIMARY KEY ("ID")
> ) TABLESPACE users;
> ```
> 

Script for PostgreSQL:

> 
> ```sql
> CREATE TABLE "my_schema".databasechangelog (
> id varchar(255) NOT NULL,
>     author varchar(255) NOT NULL,
>     filename varchar(255) NOT NULL,
>     dateexecuted timestamp NOT NULL,
>     orderexecuted int4 NOT NULL,
>     exectype varchar(10) NOT NULL,
>     md5sum varchar(35) NULL,
>     description varchar(255) NULL,
>     comments varchar(255) NULL,
>     tag varchar(255) NULL,
>     liquibase varchar(20) NULL,
>     contexts varchar(255) NULL,
>     labels varchar(255) NULL,
>     deployment_id varchar(10) NULL
> );
> CREATE TABLE "my_schema".databasechangeloglock (
>     id int4 NOT NULL,
>     locked bool NOT NULL,
>     lockgranted timestamp NULL,
>     lockedby varchar(255) NULL,
>     CONSTRAINT pk_databasechangeloglock PRIMARY KEY (id)
> );
> ```
> 


### 2. Extract DDL script using Database Management Tool

Corda is not released with a separate set of DDL scripts, instead you can use
[Corda Database Management Tool](database-management.md#database-management-tool-ref) to output the DDL scripts.
The DDL scripts contain the history of database evolution - a series of table alterations leading to the current state.
This follows the functionality of [Liquibase](http://www.liquibase.org) used by Corda to for the database schema management.
The scripts will not contain some data rows upgrades (which are run separately in the 3rd step).
The tool needs the access to a running database and takes the required database URL/credentials
from `node.conf` file. Follow the same configuration as described in
[Database Management Tool creates or upgrades database schema directly](#database-migration-tool-minimal-config-ref) section.
The only difference is that the `myLegalName = <Node legal name>` option is not needed for this step.


{{< warning >}}
It is recommended you do not use floating point types as fields on Persistent States because they might not store correctly on some database management systems (DBMS). We recommend using the “BigDecimal” data type instead.

{{< /warning >}}


After configuring the tool run:

> 
> ```shell
> java -jar tools-database-manager-RELEASE-VERSION.jar dry-run -b .
> ```
> 

The option `-b` points to the directory contains the `node.conf` file
and both subdirectories *drivers* and *cordapps*.

The command will not update any tables, except two control tables if they are not present (see the first step).
The generated scrip named *migration*.sql* will be present in the directory denoted by `-b` option.
This script contains all statements to create/modify data structures (e.g. tables/indexes)
and inserts control data into *DATABASECHANGELOG* table.


### 3. Apply DDL scripts on a database

The generated script should be applied by your database administrator using their tooling of choice.
The script needs to be executed by database user with *administrative* permissions.
Ensure the Corda node using the schema to be upgraded is shut down before executing the script.

The whole script needs to be run, run only selected parts of the script would cause
the database schema content to be in inconsistent version.


{{< warning >}}
The DDL scripts don’t contain any check preventing running them twice.
An accidental re-run of the scripts will fail (as the tables are already there)
but may left some orphan old tables.

{{< /warning >}}



### 4. Apply remaining data upgrades on a database

This step is only require if you are upgrading an existing node (an existing database needs to be upgraded)
and a Corda release require this step. This is documented in the specific release notes for each version
(e.g.  Corda 4.0 release notes mention that this step is obligatory).

The schema structure changes may require to propagate data to new tables/column based on the existing data
and specific node configuration (e.g. node legal name).
For some such migrations this cannot be expressed by DDL script
and it needs to be preformed by Database Management Tool (or a node).
There are to options how to apply the remaining data upgrade:


#### Apply remaining data upgrades using Database Management Tool

Database Management Tool can execute the remaining data upgrade.
As the schema structure is already created in the 3rd step, the tool can connect
with *restricted* database permissions.
The only activities will be inserting/upgrading data (rows) and schema structure is not altered.

Reuse the same `node.conf` and directory setup from the second step.
the only difference is that the config file has to contain `myLegalName` option.
This option is required for running this step.
Ensure that the correct value is set for correct database connection settings
(e.g. while upgrading database for the node *O=PartyA,L=London,C=GB*, assign the same value to *myLegalName*).

> 
> 
> {{< warning >}}
> Any `node.conf` misconfiguration may cause data rows migration to be wrongly applied without any error.
> Ensure `myLegalName` must exactly match the node name which is using the given database schema,
> especially if you use own supplied `node.conf` file.
> The safest way is to use the configuration file of a node.
> 
> {{< /warning >}}
> 
> 

The tool should have access to the same Cordapps which are used by the node.
If you are not reusing an existing node directory,
copy all Cordapps from a node to the *cordapss* subdirectory of the tool base directory (denoted by `-b` command line option).

Ensure the Corda node using the schema to be upgraded is shut down before running the script.
To run the remaining data migration execute command:

> 
> ```shell
> java -jar tools-database-manager-4.0-RC03.jar execute-migration -b .
> ```
> 


#### Allow a node to apply the remaining data upgrade

As the schema structure is already created in the 3rd step,
the additional data migration only consists of data manipulation only (e.g. row inserts/updates).
This can be performed by a node even if it’s connecting to a database with *restricted* permissions.
All is needed for this configure a Corda node in the same ways as for
[Corda node creates or updates database schema upon startup](#db-setup-auto-upgrade-ref)
- add `runMigration = true` option in `node.conf` file:

> 
> ```none
> database {
>     runMigration = true
>     # ...
> }
> ```
> 

Upon the node startup the node will perform the remaining data upgrades.
