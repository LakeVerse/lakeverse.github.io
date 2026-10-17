---
title: Iceberg Table Health & Compaction Advisor
status: shipped
order: 1
summary: Keeps Iceberg tables fast and cheap without anyone babysitting them — it finds the partitions that are hurting, compacts just those, and stays out of the way of the jobs writing to them.
stack: [Apache Iceberg, Spark Operator, Kubernetes, AWS Glue, Postgres, Trino, Kestra]
repo: ""   # e.g. https://github.com/lakeverse/optimicer
---

## Why it exists

Every busy Iceberg table rots the same way. Frequent dbt runs and streaming commits leave behind
thousands of small files, manifests pile up, and snapshots never expire. Queries slow down, planning
gets expensive, and storage bills climb — quietly, until someone notices.

The usual fix doesn't scale. A whole-table `OPTIMIZE` rewrites far more data than it needs to,
runs for hours on large tables, and collides with the very pipelines producing the data. So teams
either schedule it blindly at 2 a.m. and hope, or they don't do it at all.

**The advisor replaces "compact everything on a timer" with "compact exactly what needs it, when
it's safe."**

## What it does

**Measures health, per partition.** Lightweight, read-only probes written in plain Trino SQL track
file counts and size distribution, manifest growth and snapshot age for every partition. Because the
probes are SQL-only, they run against any catalog without installing anything on the platform — a
safe first step even on a lakehouse with hundreds of tables nobody has documented.

**Turns metrics into a plan.** Instead of a yes/no "this table needs compaction," the advisor ranks
partitions (or groups of partitions) by how much they're actually hurting and produces a scoped
rewrite plan — plus manifest rewrites and snapshot expiry where they're due.

**Compacts without getting in the way.** The advisor reads commit history from snapshot summaries
(Trino query IDs) to learn when each table's writers are active, and schedules work in the quiet
windows between them. Rewrites use Spark's `rewrite_data_files` with partial progress, so a large
job commits as it goes instead of all-or-nothing — and readers never see a thing.

**Runs on demand, not on standby.** Compaction runs as ephemeral Spark jobs on Kubernetes through
the Spark Operator: a cluster spins up for the plan, does the work, and disappears. No always-on
Spark cluster to pay for.

**Remembers what it did.** Every probe result, plan and run is recorded in a state store — Postgres,
or Iceberg tables if you'd rather not run a database — so you can see a table's health trend and
prove the compaction paid off.

## Three editions, one codebase

| Edition | For | What you get |
|---|---|---|
| **Solo** | Trying it on one table | A stateless library: point it at a table, get a health report and a plan. No tables or database needed. |
| **Fleet** | Running a lakehouse | State, history and orchestrated compaction across every opted-in table. |
| **Plus** | Hands-off operations | Learned merge windows and approval workflows, so compaction can be fully automatic — or gated on a human sign-off. |

## Results

<!-- Fill in from your benchmark runs before publishing. -->
- Small files reduced from **[X]** to **[Y]** on a **[N]**-partition test table
- Query planning time down **[X]%**, scan time down **[Y]%**
- Compaction work cut by **[X]%** versus a whole-table `OPTIMIZE`, because only the unhealthy partitions were rewritten

## How it's built

- **Catalog:** AWS Glue Data Catalog, developed first against an emulated AWS stack, then real AWS
- **Compute:** Spark on Kubernetes (EKS) via the Spark Operator, launched per plan
- **Probes and checks:** Trino SQL
- **Orchestration:** fits alongside Kestra, which runs the jobs that write to the tables
- **State:** pluggable — Postgres or Iceberg, chosen by a config property

## Design choices worth calling out

- **Standalone, not a platform takeover.** Studied Iceberg compaction libraries and kept the optimicer deliberately small, could work with any orchestrator and compute the platform might already have. 
- **Read-only first.** The probe layer is useful on its own and needs no write access, which makes it
  an easy yes for a platform team to try.
- **Writer-aware timing over fixed schedules.** Using commit history instead of a data column or a cron guess means compaction adapts as pipelines change.
