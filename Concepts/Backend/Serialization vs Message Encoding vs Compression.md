
---
tags:
  - backend
  - serialization
  - kafka
  - redis
  - protobuf
  - avro
---

# Serialization vs Message Encoding vs Compression

## The Three Concepts

|Concept|What it does|Examples|
|---|---|---|
|**Serialization**|Turns an in-memory object into bytes to send or store (deserialization reverses it)|The whole job|
|**Message encoding**|The _format_ used inside serialization|JSON, protobuf, Avro, MessagePack|
|**Compression**|Makes already-produced bytes smaller; independent of format|gzip, zstd, snappy, lz4|

**Pipeline:** object → serialize (using an encoding) → compress → network → decompress → deserialize

## Serialization Is the Big Box

Message encoding sits _inside_ serialization. Serialization usually includes:

- **Schema**: structure and types (`.proto` file, Avro schema). JSON has no required schema.
- **Schema evolution / compatibility**: backward and forward compatibility rules so producers and consumers can change independently.
- **Type mapping**: how language types become wire types (int → varint, dates, null handling).
- **Message encoding**: the actual byte format.
- **Framing**: how one message is separated from the next in a stream (length prefix, delimiter).
- **Character encoding**: for text formats, characters → bytes (UTF-8).

Compression and encryption are **not** part of serialization. They are separate layers applied afterward.

## JSON Is Also Serialization

- JSON is a **text** encoding. Flow: object → JSON text → UTF-8 bytes.
- Protobuf is a **binary** encoding. Flow: object → protobuf bytes (no UTF-8 step).
- The real difference is text vs binary, not "serialized vs not serialized."

## Why JSON Is Costly on the Network

- Repeated keys: `"customerId"` is written 10,000 times in an array of 10,000 objects.
- Delimiters, quotes, spaces, and newlines.
- Numbers and booleans written as text (`1000000` = 7 bytes vs 4 bytes as int32; `true` = 4 characters).
- Slower to parse than binary formats.

## Binary Encodings (protobuf, Avro)

- The schema lives **outside** the message, so messages carry only values.
    - Protobuf: small field numbers, schema compiled from `.proto` into both services.
    - Avro: schema usually shared through a **Schema Registry**; messages carry just a schema ID. The reader needs the schema to decode.
- Both sides must agree on the schema and follow compatibility rules (e.g., add optional fields, never reuse protobuf field numbers).
- Still needs serialize/deserialize, but it is **cheaper**: less CPU, less bandwidth, less storage.

## Compression on Top

- Compression works well on JSON because JSON is repetitive, so the gap with binary shrinks after compression.
- But compression costs CPU, and binary still parses much faster than JSON.
- Best setup is usually **binary encoding + batch compression**.
- In Kafka, compression is applied per **batch**, so repeated content across messages compresses very well.
- Codec trade-off: lz4/snappy for speed, zstd for ratio.

## Kafka and Redis Store Opaque Bytes

- Neither deserializes your payload (for plain Kafka topics and plain Redis strings).
- The CPU cost of serialize/deserialize is paid by **your producer, consumer, and app code**.
- GUI tools only decode the bytes for display; the stored data stays raw.
- Savings **on Kafka itself**: less disk, less network (producer → broker, replication, broker → consumer), longer retention.
- Savings **on Redis itself**: less memory (all data is in RAM), less network I/O. Redis runs commands on a single thread, so big values make each command take longer.
- Exceptions to "opaque":
    - Kafka may recompress a batch if topic compression differs from the producer's; Schema Registry validation can be enabled broker-side.
    - Redis parses data if you use RedisJSON or Lua scripts that decode it.
- Big JSON blobs in Redis also hurt because every read pulls the whole blob even when you need one field.

## When JSON Is Still the Right Choice

- Public APIs and browser clients (debuggability, easy integration).
- Low-volume internal traffic.
- Config and logs that people read directly.

Binary pays off on high-throughput service-to-service traffic and event streams. A sensible split: JSON toward the frontend, protobuf/gRPC between internal services if the volume justifies it.

## Quick Recap

- **Serialization** = the whole job (object → bytes).
- **Message encoding** = the chosen byte format inside it (JSON = text, protobuf/Avro = binary).
- **Compression** = a separate step on the resulting bytes.
- CPU saving happens in **your services**; memory, disk, and network savings happen in **Kafka and Redis**.

## Self-Test

- Why does Avro need the schema at read time while JSON doesn't?
- Why does compressing JSON narrow, but not close, the gap with protobuf?
- Why does a large value in Redis hurt more than in Kafka?