# Debezium Partition Key Guide

## What is Debezium?

Debezium is a **translator + forwarder**. It reads your database's internal transaction log (WAL) and pushes structured events to a destination — nothing more.

```
WAL → Read → Translate → Push to destination
```

---

## How It Works

Since Debezium runs on top of **Kafka Connect**, you get two modes:

**Source Connector** (what Debezium is)

```
WAL → Debezium → Kafka Topic
```

**Sink Connector** (other Kafka Connect plugins)

```
Kafka Topic → Sink Connector → Elasticsearch / S3 / Postgres / Redis / etc.
```

You can chain them:

```
WAL → Debezium → Kafka → Sink Connector → Elasticsearch
                       → Your Consumer Service
                       → Another DB
```

> Debezium's only responsibility is WAL → Kafka. Everything downstream is your responsibility.

---

## Partition Key Strategy

### The Golden Rule

**Use the Primary Key of the row** as the partition key.

```
partition_key = table_name + primary_key_value
```

### Why Primary Key?

**Ordering guarantee per row** — this is the most critical requirement.

Imagine updates to `user_id=123`:

```
INSERT  balance=100
UPDATE  balance=150
UPDATE  balance=200
DELETE
```

If these go to **different partitions** → consumers process them **out of order** → data is corrupted.

Same partition key = same partition = **guaranteed order for that row**.

### What Debezium Does By Default

Debezium automatically uses the **row's primary key as the Kafka message key**:

```json
{
  "key":   { "id": 123 },
  "value": { "before": {...}, "after": {...}, "op": "u" }
}
```

Kafka then routes same keys to the same partition automatically.

---

## Partition Key Reference

|Scenario|Partition Key|
|---|---|
|Default (most cases)|Primary Key ✅|
|Multi-tenant app|`tenant_id`|
|Time-series data|`device_id` or `sensor_id`|
|Max parallelism, order doesn't matter|Random / round-robin|

> ⚠️ **Never** partition by timestamp or random value for CDC — you'll lose row-level ordering.

---

## Customizing the Partition Key

### Option 1: One Topic Per Table (Recommended)

The simplest and cleanest approach — Debezium does this by default:

```
postgres.public.users    → partition key = user_id
postgres.public.orders   → partition key = order_id
postgres.public.payments → partition key = payment_id
```

Table name is **implicit from the topic itself**. No custom config needed.

**Benefits:**

- Cleaner architecture
- Easier permission management
- Scale consumers independently per table
- Same row-level ordering guarantee

### Option 2: Custom Key via SMT (Single Message Transform)

If you need to merge multiple tables into one topic with a custom key:

```json
{
  "name": "my-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "transforms": "addTableName",
    "transforms.addTableName.type": "org.apache.kafka.connect.transforms.ReplaceField$Key",
    "transforms.addTableName.renames": "id:user_id"
  }
}
```

### Option 3: Write a Custom SMT

For full control over the key structure:

```java
public class TablePrefixKeyTransform implements Transformation<SourceRecord> {

    @Override
    public SourceRecord apply(SourceRecord record) {
        String table = (String) record.sourceOffset().get("table");
        Struct oldKey = (Struct) record.key();

        SchemaBuilder newKeySchema = SchemaBuilder.struct()
            .field("table", Schema.STRING_SCHEMA)
            .field("id", Schema.INT64_SCHEMA);

        Struct newKey = new Struct(newKeySchema.build())
            .put("table", table)
            .put("id", oldKey.getInt64("id"));

        return record.newRecord(
            record.topic(), record.kafkaPartition(),
            newKeySchema.build(), newKey,   // ← new key
            record.valueSchema(), record.value(),
            record.timestamp()
        );
    }
}
```

---

## Summary

|Approach|When to Use|
|---|---|
|**Default (Primary Key)**|Most cases — simple and correct|
|**One topic per table**|Best practice for clean isolation|
|**Custom SMT**|Only when merging multiple tables into one topic|

> **Order per row = correctness. Everything else is secondary.**