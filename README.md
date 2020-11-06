# SDI PostgreSQLLogReaderAdapter with SSL configuration on AWS RDS Postgres

## Postgres database configuration
This example was tested on AWS RDS Postgres version 9.5.4 (minimum version that supports event triggers, which are needed for real-time replication), instance class db.t2.medium. 

To enforce SSL on the server, a new *parameter group* was created and assigned with parameter rds.force_ssl=1, then rebooted

Also, Postgres prep instructions were followed according to https://help.sap.com/viewer/7952ef28a6914997abc01745fef1b607/2.0_SPS05/en-US/a2eed27beed940b09cef58cb686ea986.html
- To avoid remote access issues, in Amazon RDS ensure the database instance setting Publicly Acccessible has been enabled.
- Configure the PostgreSQL database for real-time replication by adding a parameter group in Amazon RDS as follows.
- Create a Parameter Group.
- Search for the parameter rds.logical_replication. Change its Values default to 1.
- Associate the parameter group to the database instance.
- Restart the database instance.

## Register the adapter
Register the adapter either through the DP Agent client, or using the following statement in HANA:
```
CREATE ADAPTER "PostgreSQLLogReaderAdapter" AT LOCATION AGENT "<AGENT_NAME>";
```

## Create user on Postgres that supports replication. The roles are AWS specific and the roles were taken from [AWS documention](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html)
create user hc_technical_user with password ****;
GRANT CONNECT ON DATABASE postgres TO hc_technical_user;
grant select on all tables in schema public to hc_technical_user;
grant rds_superuser to hc_technical_user;
grant rds_replication to hc_technical_user;

## Create Remote Source on HANA
```
DROP REMOTE SOURCE "RS_POSTGRES" CASCADE;
CREATE REMOTE SOURCE "RS_POSTGRES"
	ADAPTER "PostgreSQLLogReaderAdapter" AT LOCATION AGENT "<AGENT_NAME>"
		CONFIGURATION '<?xml version="1.0" encoding="UTF-8"?>
<ConnectionProperties name="configurations">
  <PropertyGroup name="database">
      <PropertyEntry name="pds_host_name"><HOSTNAME></PropertyEntry>
      <PropertyEntry name="pds_port_number">5432</PropertyEntry>
      <PropertyEntry name="pds_database_name">postgres?sslmode=require</PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="schema_alias_replacements">
      <PropertyEntry name="schema_alias"></PropertyEntry>
      <PropertyEntry name="schema_alias_replacement"></PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="data_type_mapping">
      <PropertyEntry name="timestamptz_mapping">VARCHAR</PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="logreader">
      <PropertyEntry name="skip_lr_errors">false</PropertyEntry>
      <PropertyEntry name="support_ddl_replication">true</PropertyEntry>
      <PropertyEntry name="lr_processor_parallelism">4</PropertyEntry>
      <PropertyEntry name="lr_max_op_queue_size">1000</PropertyEntry>
      <PropertyEntry name="lr_max_scan_queue_size">1000</PropertyEntry>
      <PropertyEntry name="lr_scan_batch_size">1000</PropertyEntry>
      <PropertyEntry name="lr_polling_timeout">-1</PropertyEntry>
      <PropertyEntry name="scan_sleep_increment">5</PropertyEntry>
      <PropertyEntry name="scan_sleep_max">60</PropertyEntry>
      <PropertyEntry name="sender_formatter_parallelism">4</PropertyEntry>
      <PropertyEntry name="sender_max_row_queue_size">1000</PropertyEntry>
      <PropertyEntry name="sender_batch_size">512</PropertyEntry>
      <PropertyEntry name="sender_batch_timeout">5</PropertyEntry>
      <PropertyEntry name="skip_format_errors">false</PropertyEntry>
      <PropertyEntry name="truncation_interval">10</PropertyEntry>
  </PropertyGroup>
  <PropertyGroup name="security">
      <PropertyEntry name="pds_use_agent_stored_credential">false</PropertyEntry>
  </PropertyGroup>
</ConnectionProperties>
'
WITH CREDENTIAL TYPE 'PASSWORD'
USING '<CredentialEntry name="credential"><user>postgres</user>
<password><PASS_WORD></password></CredentialEntry>';
```

## Test replication
Create a table in Postgres
```
--DROP TABLE public.t1;
CREATE TABLE public.t1 (
	id int4 NULL,
	"name" varchar(64) NULL,
	primary key (id)
);
INSERT INTO public.t1 VALUES (0, 'This is a test');
--might be needed of table was not created by hc_technical_user:
alter table public.t1 owner to hc_technical_user;
```

Create subscription
```
CREATE VIRTUAL TABLE "DBADMIN"."V_T1" at "RS_POSTGRES"."<NULL>"."<NULL>"."public.t1";
CREATE TABLE "DBADMIN"."T_T1" LIKE "DBADMIN"."V_T1";
CREATE REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" ON "DBADMIN"."V_T1" TARGET TABLE "DBADMIN"."T_T1" ;
ALTER REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" QUEUE;
ALTER REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" DISTRIBUTE;

ALTER REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" RESET;
DROP REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1";
DROP TABLE "DBADMIN"."V_T1";
DROP TABLE "DBADMIN"."T_T1";
```

## Troubleshooting log
```
create user hc_technical_user with password ****;
GRANT CONNECT ON DATABASE postgres TO hc_technical_user;
grant select on all tables in schema public to hc_technical_user;
```
Could not execute 'ALTER REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" QUEUE'
Error: (dberror) [256]: sql processing error: QUEUE: SUB_T1: Error occurred in start subscription call to adapter PostgreSQLLogReaderAdapter for remote subscription SUB_T1[id = 160782] in remote source POSTGRES1[id = 160769]. Error: exception 151077: CDC open session failed: RS[POSTGRES1]: CDC open() error: Error creating bean with name 'PostgreSQLLogReaderAdapter' defined in file [/usr/sap/dataprovagent4/LogReader/config/postgresql.xml]: Unsatisfied dependency expressed through constructor parameter 4; nested exception is org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'LogReader' defined in file [/usr/sap/dataprovagent4/LogReader/config/postgresql.xml]: Unsatisfied dependency expressed through constructor parameter 0; nested exception is org.springframework.beans.factory.BeanCreationException: Error creating bean with name 'OperationProcessor' defined in file [/usr/sap/dataprovagent4/LogReader/config/postgresql.xml]: Bean instantiation via constructor failed; nested exception is org.springframework.beans.BeanInstantiationException: Failed to instantiate [com.sap.hana.dp.postgresqllogreaderadapter.spring.AutowiredOperationProcessor]: Constructor threw exception; nested exception is java.lang.RuntimeException: com.sap.hana.dp.postgresqllogreaderadapter.common.la.init.InitializeException: org.postgresql.util.PSQLException: ERROR: permission denied to create event trigger "dpagent_postgres_replication_ddl_trigger" Hint: Must be superuser to create an event trigger. : line 4 col 0 (at pos 0)

after turning DDL replication off in remote source
```
<PropertyEntry name="support_ddl_replication">false</PropertyEntry>
```

Could not execute 'ALTER REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" QUEUE'
Error: (dberror) [256]: sql processing error: QUEUE: SUB_T1: Failed to add subscription for remote subscription SUB_T1[id = 160790] in remote source POSTGRES1[id = 160783]. Error: exception 151050: CDC add subscription failed: RS[POSTGRES1]: Failed to add the first subscription. Error: org.postgresql.util.PSQLException: ERROR: must be superuser, replication role, or rds_replication role to use logical replication slots : line 4 col 0 (at pos 0)

https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html 
To enable logical decoding for an Amazon RDS for PostgreSQL DB instance
The user account requires the rds_superuser role to enable logical replication. The user account also requires the rds_replication role to grant permissions to manage logical slots and to stream data using logical slots.

Set the rds.logical_replication static parameter to 1. As part of applying this parameter, we also set the parameters wal_level, max_wal_senders, max_replication_slots, and max_connections. These parameter changes can increase WAL generation, so you should only set the rds.logical_replication parameter when you are using logical slots.

Reboot the DB instance for the static rds.logical_replication parameter to take effect.

Create a logical replication slot as explained in the next section. This process requires that you specify a decoding plugin. Currently we support the test_decoding and wal2json output plugins that ship with PostgreSQL.
```
grant rds_superuser to hc_technical_user;
grant rds_replication to hc_technical_user;
```
Could not execute 'ALTER REMOTE SUBSCRIPTION "DBADMIN"."SUB_T1" QUEUE'
Error: (dberror) [256]: sql processing error: QUEUE: SUB_T1: Failed to add subscription for remote subscription SUB_T1[id = 160790] in remote source POSTGRES1[id = 160783]. Error: exception 151050: CDC add subscription failed: RS[POSTGRES1]: Failed to add the first subscription. Error: ERROR: must be owner of relation t1 : line 4 col 0 (at pos 0)
```
alter table public.t1 owner to hc_technical_user;
```
SUCCESS!

after turning DDL replication on in remote source
```
<PropertyEntry name="support_ddl_replication">false</PropertyEntry>
```
SUCCESS!
