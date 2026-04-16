author: Elizabeth Christensen
id: postgres-migrations-using-logical-replication
categories: snowflake-site:taxonomy/solution-center/certification/quickstart, snowflake-site:taxonomy/product/platform
language: en
summary: Learn how to lift and shift existing PostgreSQL workloads into Snowflake Postgres using logical replication for minimal downtime environments
environments: web
status: Published
feedback link: https://github.com/Snowflake-Labs/sfguides/issues

# Postgres Migrations Using Logical Replication
<!-- ------------------------ -->
## Overview

Duration: 5

Moving a Postgres database isn't a small task—typically it's one of the biggest projects you'll undertake. If you're migrating to an entirely new platform, you have several options:

- **pg_dump and pg_restore**: A reliable way to collect an entire database and restore it to a new location. Great for smaller databases (50-150GB) where a brief downtime window is acceptable.

- **WAL-based migration**: For larger databases with WAL backup systems like pgBackRest or WAL-G/E, you can stream WAL to your new host for an exact copy with minimal downtime.

- **Logical replication**: When your database is too large for dump/restore downtime but you don't have direct WAL access (common with managed services like RDS), logical replication provides a path forward.

This quickstart focuses on the **logical replication** approach for migrating to Snowflake Postgres. Your existing database becomes the `publisher`, and Snowflake Postgres becomes the `subscriber`. During initial load, all data is copied from publisher to subscriber. After that, any transactions on the publisher are continuously sent to the subscriber, and then a final migration cutover is initiated.

### What You Will Build
- A sample remote database
- Snowflake Postgres instance
- A logical replication pipeline from a remote PostgreSQL database to Snowflake Postgres
- A completed data migration from one Postgres instance to the other

### What You Will Learn
- How to migrate schema to Snowflake Postgres
- How to configure your source database as a publisher
- How to set up Snowflake Postgres as a replication subscriber
- How to monitor replication progress
- How to perform a minimal-downtime cutover
- How to fix sequences after migration

### Prerequisites
- A source Postgres database, for this guide we will use a local database. 
- Access to a Snowflake account with Snowflake Postgres enabled
- Network connectivity between source and target databases

<!-- ------------------------ -->
## Migrate Schema

Duration: 10

### Overview
Logical replication only replicates data changes (`INSERT`, `UPDATE`, `DELETE`), so you must ensure that the target database has the correct schema beforehand.

### Export Schema from Source
To get a schema-only dump of your source database and apply it to Snowflake Postgres, run:

```bash
pg_dump -Fc -s $SOURCE_DB_URI | pg_restore --no-acl --no-owner -d $TARGET_DB_URI
```

Where:
- `$SOURCE_DB_URI` is the connection string for your current database
- `$TARGET_DB_URI` is the connection string for your Snowflake Postgres database

> aside positive
> 
> **Important**: If your migration process proceeds while application development continues, you must keep the receiving database's schema in sync with any schema changes made on your source database.

<!-- ------------------------ -->
## Configure Publisher

Duration: 15

### Overview
Configure your source database to act as a publisher for logical replication.

### Enable Logical Replication
Logical replication is enabled via the `wal_level` setting. Set this on your source database:

```
wal_level = logical
```

> aside negative
> 
> Some managed Postgres services (like Amazon RDS) may have a different method to enable this setting. Check your provider's documentation.

### Review Replication Slot Settings
For large replication projects, you may need to adjust these settings from their defaults:

- `max_replication_slots`
- `max_wal_senders`
- `max_logical_replication_workers`
- `max_worker_processes`
- `max_sync_workers_per_subscription`

See the PostgreSQL documentation on [logical replication configuration](https://www.postgresql.org/docs/current/logical-replication-config.html) for guidance on these parameters.

### Create a Replication User
Create a dedicated user for replication with the `REPLICATION` role attribute and read access to tables being replicated:

```sql
CREATE ROLE replication_user WITH REPLICATION LOGIN PASSWORD 'your_secure_password';
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

### Handle Tables Without Primary Keys
Postgres needs a way to uniquely identify rows for `UPDATE` and `DELETE` operations. First, identify tables without primary keys:

```sql
SELECT tab.table_schema,
       tab.table_name
FROM information_schema.tables tab
LEFT JOIN information_schema.table_constraints tco
          ON tab.table_schema = tco.table_schema
          AND tab.table_name = tco.table_name
          AND tco.constraint_type = 'PRIMARY KEY'
WHERE tab.table_type = 'BASE TABLE'
      AND tab.table_schema NOT IN ('pg_catalog', 'information_schema')
      AND tco.constraint_name IS NULL
ORDER BY table_schema,
         table_name;
```

For tables without primary keys, you have two options:

**Option 1**: Use an existing `UNIQUE` index:
```sql
ALTER TABLE tablename REPLICA IDENTITY USING INDEX idx_some_unique_index;
```

**Option 2**: If no unique index exists, use `REPLICA IDENTITY FULL` (treats the entire row as the key):
```sql
ALTER TABLE tablename REPLICA IDENTITY FULL;
```

### Create the Publication
Create a publication that includes all tables you want to replicate:

```sql
CREATE PUBLICATION snowflake_migration FOR ALL TABLES;
```

Verify your tables are included in the publication:

```sql
SELECT * FROM pg_publication_tables;
```

All tables you intend to migrate should appear in this list.

<!-- ------------------------ -->
## Configure Subscriber

Duration: 10

### Overview
Set up Snowflake Postgres to subscribe to the publication and begin receiving data.

### Create the Subscription
On your Snowflake Postgres database, create a subscription using the connection details for your source database:

```sql
CREATE SUBSCRIPTION snowflake_migration 
CONNECTION 'host={source_host} port=5432 dbname={database} user={replication_user} password={password}' 
PUBLICATION snowflake_migration;
```

Replace the placeholders:
- `{source_host}` - hostname or IP of your source database
- `{database}` - name of the source database
- `{replication_user}` - the replication user created earlier
- `{password}` - the password for the replication user

Creating the subscription will:
1. Create a replication slot on the publisher
2. Begin copying data from tables in the publication
3. Create temporary slots for each table during initial synchronization

> aside positive
> 
> You can limit how many tables are synchronized at once with the `max_sync_workers_per_subscription` setting on the subscriber.

<!-- ------------------------ -->
## Monitor Replication

Duration: 5

### Overview
Monitor the initial data copy and ongoing replication to ensure everything is proceeding correctly.

### Check Subscription Status
On the subscriber (Snowflake Postgres), query `pg_stat_subscription` to see replication status:

```sql
SELECT * FROM pg_stat_subscription;
```

Example output:
```
-[ RECORD 1 ]---------+------------------------------
subid                 | 27183
subname               | snowflake_migration
worker_type           | table synchronization
pid                   | 1197139
leader_pid            |
relid                 | 26721
received_lsn          |
last_msg_send_time    | 2025-09-26 15:54:45.095215+00
last_msg_receipt_time | 2025-09-26 15:54:45.095215+00
latest_end_lsn        |
latest_end_time       | 2025-09-26 15:54:45.095215+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 27183
subname               | snowflake_migration
worker_type           | apply
pid                   | 47075
leader_pid            |
relid                 |
received_lsn          | 4E32/7092F6F8
last_msg_send_time    | 2025-09-26 15:55:11.020012+00
last_msg_receipt_time | 2025-09-26 15:55:11.021989+00
latest_end_lsn        | 4E32/7092F3E0
latest_end_time       | 2025-09-26 15:55:10.843251+00
```

### Check Per-Table Sync State
View the synchronization state of each table:

```sql
SELECT * FROM pg_subscription_rel;
```

The `srsubstate` column indicates the state:
| Code | State |
|------|-------|
| `d` | Data is being copied |
| `f` | Finished table copy |
| `s` | Synchronized |
| `r` | Ready (normal replication) |

### Verify Row Counts
Because of table bloat and internal statistics differences, you cannot reliably compare table sizes. Instead, compare row counts:

```sql
SELECT count(*) FROM your_table;
```

Run this on both source and target to verify data completeness.

<!-- ------------------------ -->
## Perform Cutover

Duration: 10

### Overview
Once replication is caught up and you've validated the data, perform the cutover to complete your migration.

### Cutover Steps

1. **Stop application writes** to the source database
2. **Verify replication is caught up** - ensure all tables show state `r` (ready) in `pg_subscription_rel`
3. **Fix sequences** (see next section)
4. **Update application connection strings** to point to Snowflake Postgres
5. **Resume application operations**

> aside negative
> 
> Plan your cutover window carefully. While logical replication minimizes downtime, you still need a brief window where writes are stopped to ensure consistency.

<!-- ------------------------ -->
## Fix Sequences

Duration: 5

### Overview
Logical replication copies data but doesn't update sequence values. You must synchronize sequences before resuming production operations.

### Generate Sequence Update Commands
Run this query on your **source** database to generate `setval` commands for all sequences:

```sql
SELECT
    'SELECT setval(' || quote_literal(quote_ident(n.nspname) || '.' || quote_ident(c.relname)) || ', ' || s.last_value || ');'
FROM
    pg_class c
    JOIN pg_namespace n ON n.oid = c.relnamespace
    JOIN pg_sequences s ON s.schemaname = n.nspname
        AND s.sequencename = c.relname
WHERE
    c.relkind = 'S';
```

### Apply to Target
Execute the generated commands on your Snowflake Postgres database to synchronize all sequence values.

> aside positive
> 
> **Tip**: Save the output to a file and review before executing to ensure all sequences are captured correctly.

<!-- ------------------------ -->
## Clean Up

Duration: 5

### Overview
After a successful migration, clean up the replication configuration on both sides.

### On the Subscriber (Snowflake Postgres)
Drop the subscription:

```sql
DROP SUBSCRIPTION snowflake_migration;
```

### On the Publisher (Source Database)
Drop the publication:

```sql
DROP PUBLICATION snowflake_migration;
```

Optionally, remove the replication user if no longer needed:

```sql
DROP ROLE replication_user;
```

<!-- ------------------------ -->
## Conclusion and Resources

Duration: 2

### Congratulations!
You've successfully migrated your Postgres database to Snowflake Postgres using logical replication!

### What You Learned
- How to export and apply schema using `pg_dump` and `pg_restore`
- How to configure a source database as a publisher with proper replication settings
- How to handle tables without primary keys using replica identity
- How to create subscriptions on Snowflake Postgres
- How to monitor replication progress and verify data consistency
- How to perform a minimal-downtime cutover
- How to synchronize sequences after migration

### Key Takeaways
Logical replication is a safe and effective migration strategy. Data consistency for replicated tables is ensured as long as:
- The subscriber's schema is identical to the publisher's
- Replication is one-way with no conflicting writes on the subscriber

### Related Resources
- [PostgreSQL Logical Replication Documentation](https://www.postgresql.org/docs/current/logical-replication.html)
- [PostgreSQL Logical Replication Configuration](https://www.postgresql.org/docs/current/logical-replication-config.html)
- [Amazon RDS Logical Replication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_PostgreSQL.html#PostgreSQL.Concepts.General.FeatureSupport.LogicalReplication)
- [Snowflake Documentation](https://docs.snowflake.com/)
