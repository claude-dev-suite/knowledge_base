# Data Platform Architecture: Warehouse, Lake, Lakehouse

## Overview

A data platform serves two very different workloads — transactional (OLTP) and
analytical (OLAP) — and the architecture starts by separating them and then
choosing where analytical data lives and how it is refreshed.

## OLTP vs OLAP

- **OLTP:** many small, low-latency, row-oriented transactions on a normalized
  schema.
- **OLAP:** few large, throughput-oriented, column-oriented scans and
  aggregations.

Do not run heavy analytics on the OLTP store. Move data into an analytical store
via ETL/ELT or, increasingly, **change-data-capture (CDC)** that streams row
changes out of the OLTP database (e.g. Debezium) without dual writes.

## Storage architecture

| Option | Idea | Fits |
|--------|------|------|
| Warehouse (Snowflake, BigQuery, Redshift) | Managed columnar SQL store | Structured BI, governance, SQL-first teams |
| Data lake (files on object storage) | Cheap, schema-on-read, any format | Raw and varied data, ML feature sources |
| Lakehouse (Delta/Iceberg/Hudi on object storage) | Lake economics + ACID tables, schema, time travel | Unified BI + ML on open formats, avoids lock-in |

The **lakehouse** with open table formats is the common default in 2026 when you
want one copy of data serving both BI and ML without warehouse lock-in.

## Batch vs streaming

- **Batch** is periodic and simple but high-latency; **streaming** (Kafka +
  Flink) is continuous and low-latency but adds exactly-once, watermark, and
  state complexity.
- **Lambda** runs a batch layer (accurate) and a speed layer (fresh) and merges
  them — two codebases. **Kappa** is stream-only and reprocesses from the log —
  simpler; prefer it unless a separate batch layer is truly required.

## Modeling and ownership

- **Medallion** (bronze → silver → gold) layers raw, cleaned, and curated data.
- **Data mesh** decentralizes ownership into domain data products with federated
  governance — an organizational pattern for large enterprises, usually overkill
  for small teams that should start centralized.

## Guidance

Structured BI with a SQL team → warehouse (or lakehouse with a SQL engine).
Mixed BI + ML on open formats at scale → lakehouse (Iceberg/Delta) with Kappa
streaming and CDC ingestion. Real-time decisions → streaming with exactly-once
where correctness demands it.
