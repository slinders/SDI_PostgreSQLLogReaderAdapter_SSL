# SDI PostgreSQLLogReaderAdapter with SSL cnfiguration

## Postgres database configuration
This example was tested on AWS RDS Postgres version 9.5.4 (minimum version that supports event triggers, which are needed for real-time replication), instance class db.t2.medium. 

To enforce SSL on the server, a new *parameter group* was created and assigned with parameter rds.force_ssl=1, then rebooted

## Register the adapter
Register the adapter either through the DP Agent client, or using the following statement in HANA:
CREATE ADAPTER "PostgreSQLLogReaderAdapter" AT LOCATION AGENT "<AGENT_NAME>";

## Create Remote Source on HANA
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
