author: Elizabeth Christensen
id: sync-data-from-postgres-to-snowflake-with-iceberg-and-pg-lake
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/platform
language: en
summary: Learn how to sync data from Postgres to Snowflake using a shared Apache Iceberg table and the pg_lake extension
environments: web
status: Published
feedback link: https://github.com/Snowflake-Labs/sfguides/issues

# Sync Data from Snowflake Postgres to Snowflake with Iceberg and pg_lake
<!-- ------------------------ -->
## Overview

Duration: 5

If you're running an application on Snowflake Postgres, you probably have operational data that would be valuable for analytics in Snowflake. The [**pg_lake**](https://docs.snowflake.com/en/user-guide/snowflake-postgres/postgres-pg_lake) extension makes this easy — it lets you create Iceberg tables directly from Postgres using standard SQL (`CREATE TABLE ... USING iceberg`), and Snowflake can read those tables natively through a catalog integration. No ETL pipelines, no data movement tooling, no extra infrastructure.

Because Postgres acts as the Iceberg catalog, creating an Iceberg table is as simple as creating any other Postgres table. You write data with normal `INSERT`/`UPDATE`/`DELETE` statements and full transaction semantics, and Snowflake picks up changes automatically.

In this quickstart, you will create an Iceberg table in Snowflake Postgres using pg_lake, connect Snowflake to it, and start querying — all in a few minutes.

### What You Will Build
- A Snowflake Postgres instance with the pg_lake extension
- An Iceberg table with sample IoT sensor data
- A Snowflake catalog integration connected to the Postgres Iceberg catalog
- A queryable Iceberg table in Snowflake sourced from Postgres

### What You Will Learn
- How to enable pg_lake in Snowflake Postgres
- How to create and populate Iceberg tables from Postgres
- How to connect Snowflake to Postgres-managed Iceberg tables
- How to query Iceberg data from Snowflake
- How to manually refresh and enable auto-refresh for Iceberg tables

### Prerequisites
- Access to a Snowflake account with Snowflake Postgres enabled
- A SQL client capable of connecting to Postgres (e.g., `psql`)

<!-- ------------------------ -->
## Create a Postgres Instance

Duration: 5

### Overview
Start by creating a new Snowflake Postgres instance. You will need the instance name in a later step when configuring the Snowflake catalog integration.

### Create the Instance
Create a new Snowflake Postgres instance from the Snowflake UI or SQL. If this is your first time, follow the [Getting Started with Snowflake Postgres](https://www.snowflake.com/en/developers/guides/getting-started-with-snowflake-postgres/) guide for detailed instructions. Copy and save the **instance name** — you will need it when setting up the catalog integration in Snowflake.

### Connect to the Instance
Connect to your Snowflake Postgres instance using `psql` or your preferred SQL client:

```bash
psql postgres://<user>:<password>@<instance-host>:5432/postgres
```

> aside positive
> 
> **Tip**: Keep your instance name handy — you will reference it in the Snowflake catalog integration step.

<!-- ------------------------ -->
## Set Up pg_lake and Create Iceberg Data

Duration: 10

### Overview
Enable the pg_lake extension and create a sample Iceberg table with IoT sensor data.

### Enable pg_lake
Install the pg_lake extension along with its dependencies:

```sql
CREATE EXTENSION pg_lake CASCADE;
```

### Create an Iceberg Table
For this example, we will create a table of sensor readings. The `USING iceberg` clause tells Postgres to store the table as an Iceberg table:

```sql
CREATE TABLE sensor_readings (
    sensor_id INT,
    device_type TEXT,
    reading_time TIMESTAMPTZ,
    temperature NUMERIC(5,2),
    humidity NUMERIC(5,2),
    battery_pct NUMERIC(4,1)
) USING iceberg;
```

### Seed Sample Data
Insert 5,000 rows of simulated sensor data spanning the last 90 days:

```sql
INSERT INTO sensor_readings (sensor_id, device_type, reading_time, temperature, humidity, battery_pct)
SELECT
    (random() * 50)::int AS sensor_id,
    (ARRAY['thermostat', 'weather_station', 'greenhouse', 'cold_storage', 'hvac'])[1 + (random() * 4)::int] AS device_type,
    now() - (random() * interval '90 days') AS reading_time,
    (random() * 40 + 5)::numeric(5,2) AS temperature,
    (random() * 60 + 20)::numeric(5,2) AS humidity,
    (random() * 80 + 20)::numeric(4,1) AS battery_pct
FROM generate_series(1, 5000);
```

<!-- ------------------------ -->
## Connect Snowflake to Iceberg

Duration: 10

### Overview
Now switch to Snowflake. You will create a catalog integration that connects Snowflake to the Iceberg metadata managed by Postgres, then create an Iceberg table in Snowflake that reads from it.

### Create a Database
Create a database to hold your Iceberg tables in Snowflake:

```sql
CREATE DATABASE pglake;
USE DATABASE pglake;
```

### Create the Catalog Integration
Create a catalog integration that points Snowflake at the Postgres Iceberg catalog. Replace `{instance_name}` with the Snowflake Postgres instance name you saved earlier:

```sql
CREATE OR REPLACE CATALOG INTEGRATION postgres_iceberg_integration
  CATALOG_SOURCE    = SNOWFLAKE_POSTGRES
  TABLE_FORMAT      = ICEBERG
  CATALOG_NAMESPACE = 'public'
  REST_CONFIG = (
    POSTGRES_INSTANCE      = '{instance_name}'
    CATALOG_NAME           = 'postgres'
    ACCESS_DELEGATION_MODE = VENDED_CREDENTIALS
  )
  ENABLED = TRUE;
```

### Create the Iceberg Table
Create an Iceberg table in Snowflake that references the `sensor_readings` table in Postgres:

```sql
CREATE OR REPLACE ICEBERG TABLE iot_sensors_from_postgres
    CATALOG = 'postgres_iceberg_integration'
    CATALOG_TABLE_NAME = 'sensor_readings';
```

<!-- ------------------------ -->
## Query Iceberg Data

Duration: 5

### Overview
With the Iceberg table created in Snowflake, you can now query data that lives in Postgres-managed Iceberg storage.

### Verify Row Count
Confirm data is visible in Snowflake:

```sql
SELECT COUNT(*) FROM iot_sensors_from_postgres;
```

### Analyze the Data
Run any typical Snowflake analytics. For example, with the sample data loaded above, you can see how many readings land on each day over the 90-day window:

```sql
SELECT
    READING_TIME::DATE AS reading_date,
    COUNT(*) AS record_count
FROM iot_sensors_from_postgres
GROUP BY reading_date
ORDER BY reading_date;
```

<!-- ------------------------ -->
## Refresh and Auto-Refresh

Duration: 10

### Overview
With pg_lake, Postgres is the Iceberg catalog and Snowflake reads the Iceberg metadata through the catalog integration. This makes the table "externally managed" from Snowflake's perspective — Snowflake can query and refresh the data, but Postgres owns the table lifecycle.

Key points:
- You can always run a **manual refresh** to pull the latest data
- **Auto-refresh** (continuous polling) is available but off by default — you must explicitly enable it per table
- The polling frequency is controlled on the catalog integration via `REFRESH_INTERVAL_SECONDS` (default 30s)

### Manual Refresh
If you have inserted new data in Postgres and want it visible in Snowflake immediately, run a manual refresh:

```sql
ALTER ICEBERG TABLE iot_sensors_from_postgres REFRESH;
```

Verify the new data is visible:

```sql
SELECT COUNT(*) FROM iot_sensors_from_postgres;
```

### Enable Auto-Refresh
Enable automatic refresh so Snowflake continuously polls for changes:

```sql
ALTER ICEBERG TABLE iot_sensors_from_postgres SET AUTO_REFRESH = TRUE;
```

### Configure Refresh Interval
The catalog integration controls how often Snowflake polls for changes. Check the current setting:

```sql
DESCRIBE CATALOG INTEGRATION postgres_iceberg_integration;
```

To change the refresh interval (for example, refresh every 60 seconds):

```sql
ALTER CATALOG INTEGRATION postgres_iceberg_integration SET REFRESH_INTERVAL_SECONDS = 60;
```

### Monitor Auto-Refresh
Check whether auto-refresh is running:

```sql
SELECT SYSTEM$AUTO_REFRESH_STATUS('iot_sensors_from_postgres');
```

An `executionState` of `RUNNING` confirms auto-refresh is working.

View the refresh history to see when snapshots were last picked up:

```sql
SELECT * FROM TABLE(INFORMATION_SCHEMA.ICEBERG_TABLE_SNAPSHOT_REFRESH_HISTORY(
  TABLE_NAME => 'iot_sensors_from_postgres'
)) ORDER BY REFRESHED_ON DESC LIMIT 10;
```

<!-- ------------------------ -->
## Conclusion and Resources

Duration: 2

### Congratulations!
You have successfully synced data from Postgres to Snowflake using Iceberg tables and the pg_lake extension — with no ETL pipeline required.

### What You Learned
- How to enable pg_lake in Snowflake Postgres and create Iceberg tables with standard SQL
- How to create a Snowflake catalog integration that connects to Postgres-managed Iceberg metadata
- How to query Iceberg data from Snowflake
- How to manually refresh Iceberg tables and enable auto-refresh for continuous sync
- How to tune refresh intervals for different workload requirements

### Going Further
This guide covered a simple insert-and-query workflow. In production, you will likely want to automate incremental data processing on the Postgres side so that new data flows into Iceberg continuously. A few tools worth exploring:

- **[pg_cron](https://github.com/citusdata/pg_cron)**: Schedule recurring SQL jobs directly inside Postgres — useful for periodic inserts, aggregations, or maintenance tasks that feed your Iceberg tables.
- **[pg_incremental](https://github.com/CrunchyData/pg_incremental)**: Built on top of pg_cron, this extension provides exactly-once incremental batch processing for append-only data streams (IoT, time series, events). Define a pipeline once and it processes only new data on each run.
- **[Table partitioning](https://www.snowflake.com/en/engineering-blog/postgres-time-series-iceberg/)**: Combine Postgres declarative partitioning with pg_lake to build automated archiving — keep recent data in local Postgres partitions for fast writes, then offload older partitions to Iceberg on S3 for analytics in Snowflake.


### Related Resources
- [Introducing pg_lake: Integrate Your Data Lakehouse with Postgres](https://www.snowflake.com/en/engineering-blog/pg-lake-postgres-lakehouse-integration/) (blog)
- [Snowflake Postgres: Unify Postgres and Analytics on One Platform](https://www.snowflake.com/en/blog/streamline-data-movement-snowflake-postgres/) (blog)
- [pg_lake: Configuring S3 Storage](https://docs.snowflake.com/en/user-guide/snowflake-postgres/postgres-pg_lake)
- [Snowflake Postgres Extensions](https://docs.snowflake.com/en/user-guide/snowflake-postgres/postgres-extensions)
- [pg_lake GitHub Repository](https://github.com/Snowflake-Labs/pg_lake)
- [pg_lake Iceberg Tables Documentation](https://github.com/Snowflake-Labs/pg_lake/blob/main/docs/iceberg-tables.md)
- [Apache Iceberg Documentation](https://iceberg.apache.org/docs/latest/)
- [Snowflake Documentation](https://docs.snowflake.com/)
