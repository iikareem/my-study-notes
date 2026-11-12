---
type: moc
aliases:
  - OLAP MOC
  - OLAP Foundations
  - OLAP Index
tags:
  - concepts
  - database
  - olap
  - moc
---

# OLAP Foundations

> **Who this is for:** An experienced backend/Postgres engineer who is new to OLAP, columnar databases, Parquet, and analytical modeling. Every section contrasts with Postgres when it helps build the mental model. Commands are DuckDB SQL unless stated otherwise.

---

## Table of Contents

- [[01 Module 1 — OLTP vs OLAP Mental Model|Module 1 — OLTP vs OLAP Mental Model]]
    - [[01 Module 1 — OLTP vs OLAP Mental Model#Module 1 Summary|Module 1 Summary]]
    - [[01 Module 1 — OLTP vs OLAP Mental Model#1.1 What problem OLAP solves that Postgres struggles with|1.1 What problem OLAP solves that Postgres struggles with]]
    - [[01 Module 1 — OLTP vs OLAP Mental Model#1.2 Seek-heavy vs scan-heavy workloads|1.2 Seek-heavy vs scan-heavy workloads]]
    - [[01 Module 1 — OLTP vs OLAP Mental Model#1.3 When to keep analytics in Postgres vs offload|1.3 When to keep analytics in Postgres vs offload]]
    - [[01 Module 1 — OLTP vs OLAP Mental Model#1.4 Row store vs columnar store|1.4 Row store vs columnar store]]
    - [[01 Module 1 — OLTP vs OLAP Mental Model#1.6 Postgres → OLAP Translation Table|1.6 Postgres → OLAP Translation Table]]
- [[02 Module 2 — Columnar Execution Internals|Module 2 — Columnar Execution Internals]]
    - [[02 Module 2 — Columnar Execution Internals#Module 2 Summary|Module 2 Summary]]
    - [[02 Module 2 — Columnar Execution Internals#2.1 Vectorized execution vs volcano/tuple-at-a-time|2.1 Vectorized execution vs volcano/tuple-at-a-time]]
    - [[03 Volcano vs Vectorized Execution Models|Volcano vs Vectorized Execution Models]]
    - [[02 Module 2 — Columnar Execution Internals#2.2 Late materialization|2.2 Late materialization]]
    - [[02 Module 2 — Columnar Execution Internals#2.3 Predicate pushdown|2.3 Predicate pushdown]]
    - [[02 Module 2 — Columnar Execution Internals#2.4 Compression that works because of columns|2.4 Compression that works because of columns]]
    - [[02 Module 2 — Columnar Execution Internals#2.5 How to read an analytical EXPLAIN differently from Postgres EXPLAIN|2.5 How to read an analytical EXPLAIN]]
    - [[02 Module 2 — Columnar Execution Internals#2.7 Postgres → OLAP Translation Table|2.7 Postgres → OLAP Translation Table]]
- [[04 Module 3 — Parquet & File Formats|Module 3 — Parquet & File Formats]]
    - [[04 Module 3 — Parquet & File Formats#Module 3 Summary|Module 3 Summary]]
    - [[04 Module 3 — Parquet & File Formats#3.1 What Parquet is and why it exists|3.1 What Parquet is and why it exists]]
    - [[04 Module 3 — Parquet & File Formats#3.2 Row groups, column chunks, footers, statistics|3.2 Row groups, column chunks, footers, statistics]]
    - [[04 Module 3 — Parquet & File Formats#3.3 Why SELECT * hurts more in columnar formats|3.3 Why SELECT * hurts more in columnar formats]]
    - [[04 Module 3 — Parquet & File Formats#3.4 CSV vs Parquet trade-offs|3.4 CSV vs Parquet trade-offs]]
    - [[04 Module 3 — Parquet & File Formats#3.5 Schema evolution basics and common pitfalls|3.5 Schema evolution basics and common pitfalls]]
    - [[04 Module 3 — Parquet & File Formats#3.7 Postgres → Parquet Translation Table|3.7 Postgres → Parquet Translation Table]]
- [[05 Module 4 — Partitioning & Data Layout|Module 4 — Partitioning & Data Layout]]
    - [[05 Module 4 — Partitioning & Data Layout#Module 4 Summary|Module 4 Summary]]
    - [[05 Module 4 — Partitioning & Data Layout#4.1 Hive-style partitions|4.1 Hive-style partitions]]
    - [[05 Module 4 — Partitioning & Data Layout#4.2 Partition pruning vs Postgres indexes — same goal, different mechanism|4.2 Partition pruning vs Postgres indexes]]
    - [[05 Module 4 — Partitioning & Data Layout#4.3 Choosing partition keys|4.3 Choosing partition keys]]
    - [[05 Module 4 — Partitioning & Data Layout#4.4 The small-files problem|4.4 The small-files problem]]
    - [[05 Module 4 — Partitioning & Data Layout#4.5 Compaction and file sizing intuition|4.5 Compaction and file sizing intuition]]
    - [[05 Module 4 — Partitioning & Data Layout#4.6 How layout decisions affect cost more than clever SQL|4.6 How layout decisions affect cost more than clever SQL]]
    - [[05 Module 4 — Partitioning & Data Layout#4.8 Postgres → OLAP Translation Table|4.8 Postgres → OLAP Translation Table]]
- [[06 Module 5 — Star Schemas & Dimensional Design|Module 5 — Star Schemas & Dimensional Design]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#Module 5 Summary|Module 5 Summary]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.1 Why 3NF is wrong for analytics|5.1 Why 3NF is wrong for analytics]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.2 The star schema|5.2 The star schema]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.3 Fact tables|5.3 Fact tables]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.4 Dimension tables|5.4 Dimension tables]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.5 Grain — the most important design decision|5.5 Grain — the most important design decision]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.6 Slowly Changing Dimensions|5.6 Slowly Changing Dimensions]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.7 Snowflake schema vs star schema|5.7 Snowflake schema vs star schema]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.8 Common modeling mistakes|5.8 Common modeling mistakes]]
    - [[06 Module 5 — Star Schemas & Dimensional Design#5.9 Postgres → Dimensional Model Translation Table|5.9 Postgres → Dimensional Model Translation Table]]
- [[07 Module 6 — Ingestion Thinking|Module 6 — Ingestion Thinking]]
- [[08 Module 7 — DuckDB in Practice|Module 7 — DuckDB in Practice]]
- [[09 Module 8 - Query Patterns That Matter in OLAP|Module 8 — Query Patterns That Matter in OLAP]]

## Start / path

- [[OLAP Starter Path — Think OLAP First]]

## Pitwall case study

- [[Pitwall Project — Everything We Learned]]
- [[Pitwall ETL Lesson — Developer Takeaways]]
- [[Pitwall Star Schema Diagram]]

---

**Parent:** [[Concepts MOC]]
