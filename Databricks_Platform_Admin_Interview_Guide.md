# Databricks Platform Administrator & Architect — Interview Master Guide

> 38 technical questions with deep, architecture-grade explanations.
> Diagrams · Code samples · Best practices · Real-world scenarios.

---

## Table of Contents

1. [Databricks Architecture & Workspace Fundamentals](#1-databricks-architecture--workspace-fundamentals)
2. [Unity Catalog & Data Governance](#2-unity-catalog--data-governance)
3. [Identity, Access Management & SSO](#3-identity-access-management--sso)
4. [Compute, Clusters & SQL Warehouses](#4-compute-clusters--sql-warehouses)
5. [Networking & Security Architecture](#5-networking--security-architecture)
6. [Secrets, Encryption & Compliance](#6-secrets-encryption--compliance)
7. [Delta Lake, Storage & Performance](#7-delta-lake-storage--performance)
8. [Jobs, Workflows & Delta Live Tables](#8-jobs-workflows--delta-live-tables)
9. [Monitoring, Logging & FinOps](#9-monitoring-logging--finops)
10. [CI/CD, DevOps & Infrastructure-as-Code](#10-cicd-devops--infrastructure-as-code)
11. [Migration, DR & High Availability](#11-migration-dr--high-availability)
12. [Scenario-Based & Troubleshooting Questions](#12-scenario-based--troubleshooting-questions)
13. [Quick-Fire Conceptual Questions](#13-quick-fire-conceptual-questions)
14. [Cheat-Sheet: CLI, API & Terraform](#14-cheat-sheet-cli-api--useful-sql)
15. [Interview Strategy & Tips](#15-interview-strategy--tips)

---

## 1. Databricks Architecture & Workspace Fundamentals

### Architecture Diagram (textual)

```
┌─────────────────────────────────┐        ┌──────────────────────────────────────┐
│   CONTROL PLANE                 │        │   DATA / COMPUTE PLANE               │
│   (Databricks Cloud Account)    │ <----> │   (Customer Cloud: AWS/Azure/GCP)    │
│                                 │  mTLS  │                                      │
│   • Web UI / REST API           │        │   • Spark Clusters                   │
│   • Notebooks Service           │        │     (Driver + Executors)             │
│   • Job Scheduler               │        │     inside customer VPC/VNet         │
│   • Cluster Manager             │        │   • SQL Warehouses (Photon)          │
│   • Unity Catalog Metastore     │        │   • Cloud Storage (S3/ADLS/GCS)      │
│   • Audit Logs / Usage / Lineage│        │   • VPC, Private Endpoints, KMS      │
└─────────────────────────────────┘        └──────────────────────────────────────┘
```

### Q1. Explain the Databricks high-level architecture (Control Plane vs Data Plane)

**Two-plane model:**
- **Control Plane** runs in Databricks-managed cloud. Hosts the web app, REST API endpoints, the cluster manager, the job scheduler, the notebook command results store, and the Unity Catalog metastore. Also contains the workspace database (encrypted at rest), the secret store, and audit log generators.
- **Data / Compute Plane** runs in *your* cloud subscription. Contains the actual Spark driver and executor VMs, SQL warehouse compute, and your cloud storage (S3/ADLS/GCS). Your data **never** leaves this plane (except for ephemeral notebook display rows, which can be disabled).

**Two flavours of the data plane:**
1. **Classic Compute Plane** — VMs launched in the customer's VPC/VNet using customer's cloud credentials. Full network control.
2. **Serverless Compute Plane** — VMs launched in Databricks' multi-tenant fleet but with strong isolation (per-customer firewalls, ephemeral compute, mTLS). Faster start-up, lower idle cost.

**Communication:** Mutual TLS between control and data plane. With *Secure Cluster Connectivity (SCC)*, all communication initiated *from* the data plane outbound — no inbound port opened, no public IP needed on workers.

> **Architect tip:** Emphasise the "data sovereignty" angle — many regulated customers choose Databricks precisely for this separation. Mention that *metadata* in the control plane (notebook code, cell results) can be encrypted with **Customer Managed Keys (CMK)**.

### Q2. What is a Databricks Account vs a Workspace? How do they relate to Unity Catalog?

- **Account** — top-level container. One per cloud customer (`accounts.cloud.databricks.com`). Owns identities, **metastores**, billing, audit log delivery, network configurations, credentials.
- **Workspace** — regional deployment containing compute (clusters), notebooks, jobs, DLT pipelines, dashboards, repos, and workspace-local file system (DBFS legacy).
- **Unity Catalog metastore** — created at the *account* level and attached to one or more workspaces in the same region. Lets multiple workspaces share governed data.

**Typical organisation:**
- 1 Account → multiple metastores (one per region)
- 1 Metastore → multiple workspaces (Dev/QA/Prod, or BU-A/BU-B)
- 1 Workspace → one or more catalogs visible via the metastore

### Q3. Multi-workspace design strategy for 20+ teams

**Hub-and-spoke pattern:**
1. **Environment isolation:** separate workspaces for *Dev*, *UAT*, *Prod*. Different cloud subscriptions if possible.
2. **Domain isolation:** one workspace per data domain (Finance, Risk, Marketing) — data mesh pattern.
3. **Shared governance:** a single Unity Catalog metastore per region; all workspaces federate identity from the account.
4. **Cost separation:** tag clusters with `cost-center`, `env`, `owner`; enforce via cluster policies; use `system.billing.usage` for chargeback.
5. **Identity:** SCIM groups defined in IdP (Azure AD/Okta) sync into the account.
6. **Code/CI:** central Git org; Databricks Asset Bundles deploy to each workspace.

> **Why not one giant workspace?** Workspace-level quotas, blast radius of a bad init script, separate notebook namespaces, RBAC clarity, per-environment Databricks Runtime upgrades.

---

## 2. Unity Catalog & Data Governance

### Unity Catalog Hierarchy

```
METASTORE (account / region)
    │
    ├── CATALOG (prod_finance)
    │   └── SCHEMA (gl)
    │       ├── TABLE  · VIEW  · VOLUME  · FUNCTION  · MODEL  · STREAM
    │       └── External Location · Storage Credential
    │
    ├── CATALOG (prod_risk)
    │   └── SCHEMA (market)
    │
    └── CATALOG (dev_sandbox)
        └── SCHEMA (experiments)

Three-level namespace:  catalog.schema.object
```

### Q4. How is Unity Catalog architected and what objects does it govern?

**Architecture:** UC is a global, account-level service running in the control plane. Stores metadata in Databricks' managed PostgreSQL and exposes a permissions engine that intercepts every data access from compute.

**Securable objects (govern via GRANT/REVOKE):**
- **Metastore**, **Catalog**, **Schema**
- **Table** (managed/external), **View**, **Materialized View**, **Streaming Table**
- **Volume** (governed file storage replacing DBFS mounts)
- **Function** (Python/SQL UDF), **Registered Model** (MLflow)
- **External Location**, **Storage Credential**, **Connection** (Lakehouse Federation)
- **Share**, **Recipient**, **Provider** (Delta Sharing)
- **Clean Room** (privacy-preserving collaboration)

**Permissions model:**
```sql
GRANT SELECT      ON TABLE   prod_finance.gl.journal TO `fin-analysts`;
GRANT USE CATALOG ON CATALOG prod_finance            TO `fin-analysts`;
GRANT USE SCHEMA  ON SCHEMA  prod_finance.gl         TO `fin-analysts`;
```

A user needs **USE CATALOG** + **USE SCHEMA** + the specific privilege (SELECT/MODIFY/etc.) on the object. Ownership grants all privileges and is transferable.

**Enforced everywhere:** Notebooks, jobs, DLT, SQL warehouses, JDBC/ODBC, Delta Sharing.

### Q5. External Location vs Storage Credential vs Volume

| Object | Definition | Use case |
|---|---|---|
| **Storage Credential** | UC-managed reference to a cloud IAM principal (IAM role on AWS, Managed Identity on Azure, Service Account on GCP) | Created once per env; foundation for all external access |
| **External Location** | A URL (e.g. `abfss://...`) bound to a storage credential, with its own GRANTs (READ FILES, WRITE FILES, CREATE TABLE) | Defines *where* external tables and volumes can be created |
| **External Table** | Table whose data lives at an external location but metadata in UC | Existing data lakes, sharing with non-Databricks engines |
| **Volume** | Governed file location for unstructured/semi-structured files (PDFs, images, JSON, ML artefacts) | Replaces DBFS mounts; accessible via `/Volumes/cat/schema/vol/...` |

```sql
CREATE STORAGE CREDENTIAL prod_cred WITH AZURE_MANAGED_IDENTITY '/subscriptions/.../uc-mi';

CREATE EXTERNAL LOCATION prod_loc URL 'abfss://prod@dlsprod.dfs.core.windows.net/'
       WITH (STORAGE CREDENTIAL prod_cred);

GRANT READ FILES, WRITE FILES ON EXTERNAL LOCATION prod_loc TO `de-engineers`;

CREATE VOLUME prod_finance.gl.docs LOCATION 'abfss://prod@dlsprod.../docs';
```

### Q6. Row-level and column-level security in Unity Catalog

**Two approaches:**

**1. Dynamic Views** (works on legacy HMS too):
```sql
CREATE VIEW finance.gl.journal_secure AS
SELECT account_id,
       CASE WHEN is_account_group_member('finance-pii') THEN ssn
            ELSE '***' END AS ssn,
       amount
FROM   finance.gl.journal
WHERE  region IN (SELECT region FROM finance.sec.user_region
                  WHERE user = current_user());
```

**2. Row Filters & Column Masks (UC native, GA):**
```sql
CREATE FUNCTION mask_ssn(s STRING) RETURNS STRING
RETURN CASE WHEN is_account_group_member('pii') THEN s ELSE '***-**-****' END;

ALTER TABLE finance.gl.journal
  ALTER COLUMN ssn SET MASK mask_ssn;

CREATE FUNCTION region_filter(region STRING) RETURNS BOOLEAN
RETURN region = current_user_region();

ALTER TABLE finance.gl.journal SET ROW FILTER region_filter ON (region);
```

> Row filters / column masks are reusable functions — easier to audit than views and they propagate to all queries (BI, JDBC, Sharing).

### Q7. Lakehouse Federation — when to use it

Lakehouse Federation lets UC *query external databases* (Snowflake, Redshift, Synapse, MySQL, Postgres, SQL Server, BigQuery, Salesforce) as foreign catalogs — no data movement.

```sql
CREATE CONNECTION snow_prod TYPE snowflake
  OPTIONS (host '...', user secret(...), password secret(...));

CREATE FOREIGN CATALOG snow_fin USING CONNECTION snow_prod
  OPTIONS (database 'FIN_DB');
```

**Use cases:** migration phase, read-only BI access to ODS systems, joining lakehouse data with operational stores, federated governance.

---

## 3. Identity, Access Management & SSO

### Q8. End-to-end identity in Databricks

**Three principal types:** Users, Groups, Service Principals — all defined at the *account* level and pushed to workspaces (identity federation).

**Provisioning paths:**
1. **SCIM 2.0** — push provisioning from Azure AD / Okta / OneLogin / AWS IAM Identity Center → account-level. Recommended for production.
2. **Manual** via account console (small orgs).
3. **Terraform** (`databricks_user`, `databricks_group`, `databricks_service_principal`) — auditable, idempotent.

**SSO:** SAML 2.0 (most common) or OIDC. Configured once at account; workspaces inherit.
**MFA:** Always enforce at the IdP layer.

**Tokens:**
- **OAuth M2M (recommended)** — service principal generates short-lived OAuth tokens via client-id/secret.
- **OAuth U2M** — user delegates browser auth (CLI/REST).
- **Personal Access Tokens (PAT)** — long-lived; restrict creation via cluster/workspace settings.

**Group hierarchy:** Account groups → Workspace assignments → UC GRANTs. Always grant to *groups*, never to users directly.

### Q9. Service Principal vs Managed Identity vs PAT

| Type | Best for | Auth | Notes |
|---|---|---|---|
| Databricks Service Principal | CI/CD pipelines, jobs run-as identity | OAuth M2M | Strongly recommended; supports federated identity |
| Azure Managed Identity | Azure-hosted jobs / Functions calling Databricks | Federated to Databricks SP via Workload Identity Federation | No secrets to manage |
| Personal Access Token (PAT) | Quick scripts only | Static bearer token | Long-lived; rotate; avoid in CI/CD |
| GitHub OIDC / AWS IAM Role | GitHub Actions, AWS Lambda | Token exchange | Best-in-class — no static secrets |

> **Anti-pattern:** Storing PATs in Git secrets. Use OIDC federation from GitHub Actions to a Databricks service principal.

### Q10. Workspace-level ACLs vs UC permissions

**Workspace ACLs** govern compute and code:
- Clusters & cluster policies (CAN_USE / CAN_MANAGE / CAN_RESTART)
- Jobs & pipelines (CAN_VIEW / CAN_MANAGE_RUN / IS_OWNER)
- Notebooks, folders, repos (Workspace Object ACLs)
- SQL Warehouses, alerts, dashboards
- Token usage, IP access list management

**Unity Catalog** governs data:
- Catalog/Schema/Table/View/Volume/Function/Model — SELECT, MODIFY, USE, etc.
- External locations, storage credentials, connections, shares.

**Two systems, two policies** — both must allow the action.

---

## 4. Compute, Clusters & SQL Warehouses

### Compute Types at a Glance

| Type | Use Case | Cost Profile |
|---|---|---|
| **All-Purpose Cluster** | Interactive, multi-user, long-lived. Use for exploration, dashboards | ~2× DBU rate vs job cluster |
| **Job Cluster** | Ephemeral, per-run, cheap. Use for production workflows | Lowest DBU rate; auto-terminates |
| **SQL Warehouse (Classic/Pro/Serverless)** | BI, ad-hoc SQL. Photon-on-by-default | Per-second |
| **Serverless Compute** | Notebooks/jobs/DLT, 5-10s startup, Databricks-managed | Pay per second; no cluster mgmt |

### Q11. Cluster sizing methodology

1. **Characterise workload:**
   - ETL / batch → memory-optimised (Standard_E or r6gd)
   - SQL / interactive → Photon + compute-optimised
   - ML training → GPU (Standard_NC, p3)
   - Streaming → balance memory + steady throughput
2. **Estimate working-set size:** data volume × inflation factor (1.5–3× for shuffles). Rule of thumb: cluster RAM ≈ 1.5× shuffled data.
3. **Pick worker count:** start with 4–8 workers; enable autoscaling with `min=2`, `max=expected peak × 1.5`.
4. **Driver sizing:** normally same SKU as workers; bigger if you do `collect()`, broadcast joins, or run Pandas-on-Spark with large local conversions.
5. **Validate with Spark UI:**
   - Stage timeline: are tasks evenly distributed?
   - Spill to disk → increase memory or partitions
   - GC overhead > 10% → bigger memory / smaller partitions
   - Idle executors → reduce min workers
6. **Enable Photon** for SQL + Delta ETL — usually 2–3× speedup, 30–50% TCO reduction.

> Target average CPU utilisation of 60–80% across the run.

### Q12. Cluster policies — how they work

JSON document with rules per attribute. Each rule has `type` (fixed/allowlist/regex/range/forbidden), value, and optional `hidden`/`defaultValue`.

```json
{
  "spark_version": { "type": "regex", "pattern": "^(14|15)\\..*-scala2.12$" },
  "node_type_id": { "type": "allowlist", "values": ["Standard_D8ds_v5", "Standard_D16ds_v5"] },
  "autotermination_minutes": { "type": "range", "minValue": 10, "maxValue": 60, "defaultValue": 30 },
  "custom_tags.cost-center": { "type": "fixed", "value": "FIN-1234" },
  "spark_conf.spark.databricks.cluster.profile": { "type": "forbidden" },
  "init_scripts.*": { "type": "forbidden" },
  "runtime_engine": { "type": "fixed", "value": "PHOTON" }
}
```

**Common policy patterns:**
- *Personal compute* — single-user, small node, 30 min autoterm.
- *Shared analytics* — Photon, autoscaling 2–8, no init scripts.
- *Job-only* — used in production jobs, no user can attach.
- *ML GPU* — restricted to ML group, GPU node types.

### Q13. Photon — technical detail

Photon is a **C++ vectorised query engine** that replaces parts of the Spark execution plan (filter, project, aggregate, hash-join) with native code on columnar batches using SIMD.

**Speeds up:** SQL workloads, Delta MERGE/UPDATE/DELETE, large aggregations, joins, COPY INTO, CONVERT TO DELTA.
**Does *not* help much for:** Python UDFs (not vectorised), Scala UDFs, ML training, heavy `foreachPartition`, RDD APIs.

**Cost:** Photon DBU rate is ~2× non-Photon — but if it makes the job >2× faster (commonly 3–5×) total cost drops. **Always benchmark.**

**Check if it's actually used:** Spark UI → SQL plan, nodes are prefixed `Photon` (e.g. `PhotonScan`). If you see `Row` operators wrapping a UDF, Photon falls back to Spark.

### Q14. Instance Pools

Pre-acquired, idle cloud VMs kept warm. When a cluster requests workers, it pulls them from the pool (boot already done) — startup time drops from 3–5 min to 30–60 s.

**Costs:** You pay cloud cost for pool VMs (*not* DBUs) while idle, plus DBU only when a cluster uses them.

**Best for:** Many short jobs, sequential job runs, interactive workloads with spiky usage.

**Best practices:**
- One pool per node type.
- Set `min_idle_instances` to your typical concurrency.
- Set `idle_instance_autotermination_minutes` to 15–30.
- Tag pool to attribute cost.
- Pin Databricks Runtime version to avoid re-downloads.

### Q15. SQL Warehouses — Classic vs Pro vs Serverless

| Feature | Classic | Pro | Serverless |
|---|---|---|---|
| Startup time | ~4 min | ~4 min | ~5–10 sec |
| Photon | Yes | Yes | Yes |
| Predictive I/O | No | Yes | Yes |
| Intelligent caching | Limited | Yes | Yes |
| Multi-tasking / cluster scaling | Basic | Better | Best (instant) |
| Compute location | Customer cloud | Customer cloud | Databricks cloud |
| Cost model | VM + DBU | VM + DBU | DBU only (premium rate) |
| Best for | Legacy | Cost-balanced BI | BI w/ low concurrency variance |

**Recommendation:** Use **Serverless** for production BI to maximise concurrency and minimise wait. Use Pro if you have strict data-residency that forbids serverless.

---

## 5. Networking & Security Architecture

### Hardened Topology

```
┌────────────────────────────────────────────────────────────────────────┐
│ Customer Virtual Network (VNet/VPC)                                    │
├────────────────────────────────────────────────────────────────────────┤
│ Public Subnet (empty if SCC enabled)   │  Private Subnets (Workers)    │
│  NAT GW / Egress Firewall              │  Driver + Executors (no PIP)  │
├────────────────────────────────────────────────────────────────────────┤
│ Private Endpoints → Databricks Control Plane (SCC Relay)               │
│ Private Endpoints → Storage (ADLS / S3) · Key Vault · Event Hubs       │
├────────────────────────────────────────────────────────────────────────┤
│ NSG / NACL: Deny all inbound, allow only required egress FQDNs         │
│ Egress firewall (Azure FW / AWS NFW) with allow-list                   │
└────────────────────────────────────────────────────────────────────────┘
```

### Q16. Securing a workspace at network layer

1. **VNet/VPC Injection** — deploy workspace into a customer-controlled VNet so you own subnets, route tables, NSGs.
2. **Secure Cluster Connectivity (SCC / NPIP)** — no public IPs on workers; outbound-only relay to control plane.
3. **Private Link to control plane** — replaces public endpoints. Two private endpoints typically: front-end (UI/API for users) and back-end (SCC relay + REST from clusters).
4. **Private endpoints to storage** (ADLS / S3 Gateway / GCS), Key Vault, Event Hubs / Kinesis, Snowflake, etc.
5. **Egress firewall** (Azure Firewall / AWS Network Firewall) with FQDN allowlist for Databricks domains, PyPI mirror, Maven, telemetry.
6. **IP Access Lists** at workspace level for UI/REST access.
7. **Customer-Managed VPC/VNet peering** for hub-spoke routing.
8. **Disable public network access** on the workspace.

> For PCI/HIPAA, also enable: **Compliance Security Profile (CSP)**, enhanced security monitoring, FedRAMP region, audit log delivery.

### Q17. Compliance Security Profile (CSP)

A workspace-level setting (Enterprise tier) that turns on hardened controls required for HIPAA, PCI-DSS, FedRAMP Moderate/High, IRAP.

**What it changes:**
- Forces enhanced security monitoring (eBPF-based file integrity, AV)
- Restricts DBR versions to a curated, hardened list
- Enforces SCC + private link
- Disables certain insecure APIs (e.g., DBFS root mounts)
- Mandatory automated patching cadence

Cannot be undone once enabled for a workspace.

---

## 6. Secrets, Encryption & Compliance

### Q18. Encryption layers and where to apply CMK

**Five encryption layers:**
1. **Workspace storage** (DBFS root, notebooks, query results) — encrypted at rest by default. Add **CMK for managed services** (CMK-MS) to use your KMS/AKV key.
2. **Workspace files in control plane** — additional CMK on the Databricks control-plane database.
3. **EBS volumes / managed disks** on workers — CMK for workspace storage (CMK-WS).
4. **Customer data in cloud storage** (ADLS/S3) — you manage; UC enforces access.
5. **In-transit** — TLS 1.2+ everywhere; mTLS between data and control plane.

**Other:**
- Spark local disk encryption (LUKS/dm-crypt) — enable via cluster policy
- Secret store backed by Databricks-managed key (or AKV-backed scope)

### Q19. Secret scopes — types and best practices

| Type | Backing store | Best for |
|---|---|---|
| Databricks-backed | Databricks-managed encrypted KV | Quick start, dev |
| Azure Key Vault-backed | AKV | Azure Databricks production |
| AWS Secrets Manager (via IAM) | ASM | AWS Databricks production |

```python
# Access in notebook (redacted in logs)
pwd = dbutils.secrets.get("prod-akv", "sf-pwd")
spark.conf.set("fs.azure.account.key...", pwd)
```

**Best practices:**
- Never log or `print()` secrets — output auto-redacted only for display.
- Grant *READ* on scope to a service principal or AAD group, not individuals.
- Rotate via KV versioning; the scope always reads latest.
- Avoid storing secrets in cluster Spark config except via secret reference: `{{secrets/scope/key}}`.

### Q20. Audit logs

**Two paths:**
1. **System tables (recommended):**
```sql
SELECT event_time, user_identity.email, action_name, request_params
FROM   system.access.audit
WHERE  workspace_id = 1234567890
  AND  event_date >= current_date() - interval 7 days
  AND  action_name IN ('createCluster', 'deleteCluster');
```
2. **Audit Log Delivery** — configure account-level delivery to S3/ADLS as JSON; ingest into SIEM.

**Critical actions to alert on:**
- PAT creation / token usage from unusual IPs
- Cluster created with `init_scripts`
- UC GRANTs to broad groups (e.g. `account users`)
- Workspace admin promotion
- Failed login spikes

---

## 7. Delta Lake, Storage & Performance

### Medallion Architecture

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   BRONZE     │ → │   SILVER     │ → │    GOLD      │
│              │   │              │   │              │
│ Raw ingest   │   │ Cleansed     │   │ Business     │
│ Append-only  │   │ Deduped      │   │ aggregates   │
│ Schema-on-r  │   │ Typed        │   │ Star/snow    │
│ Auto Loader  │   │ CDC applied  │   │ Feature/ML   │
│ CDC, Kafka,  │   │ PII masked   │   │ Mat. views   │
│ files, JDBC  │   │ DQ enforced  │   │ BI/API ready │
└──────────────┘   └──────────────┘   └──────────────┘
```

### Q21. Delta Lake internals — ACID on object storage

A Delta table is a directory containing:
- `_delta_log/00000000000000000000.json`, `...01.json` — transaction log entries (one JSON per commit)
- Periodic `...10.checkpoint.parquet` snapshots compacting older commits
- Data files: `part-00000-xxxx.parquet`

**ACID mechanism:**
- **Atomic:** Commit writes a single JSON file via *atomic put-if-absent* (Azure has atomic rename; S3 uses DynamoDB-based commit coordinator or built-in conditional writes).
- **Consistent:** Each entry contains `add`/`remove` actions; readers materialise current snapshot by replaying log.
- **Isolation:** Optimistic concurrency — writers detect conflicts on overlapping file sets; serialisable for INSERT/MERGE, write-serialisable for UPDATE/DELETE.
- **Durable:** All files in cloud storage.

**Time Travel:** `SELECT * FROM t VERSION AS OF 42` reads the log up to commit 42.
**VACUUM:** removes files no longer referenced by any version newer than retention threshold (default 7 days).
**OPTIMIZE:** bin-packs small files into 1 GB targets; with `ZORDER BY` co-locates rows by chosen columns using space-filling curves.

### Q22. Liquid Clustering vs Z-Order

Liquid Clustering uses an **incremental tree-based clustering** (not a one-shot Z-Order curve) that:
- Re-clusters only changed data on each commit — no full-table rewrites
- Allows you to change clustering keys without rewriting the whole table
- Avoids the "partition trap": no skewed partitions, no small-file explosion
- Works with concurrent writers

```sql
CREATE TABLE sales.orders (order_id BIGINT, region STRING, ts TIMESTAMP, amt DECIMAL(18,2))
CLUSTER BY (region, ts);

-- Maintenance
OPTIMIZE sales.orders;
ALTER TABLE sales.orders CLUSTER BY (region, customer_id);  -- swap keys
```

**Pick clustering columns:** the columns you filter and join on most. Up to 4 columns recommended.

### Q23. Predictive Optimization

A managed service (UC-enabled) where Databricks automatically runs `OPTIMIZE`, `VACUUM` and statistics collection on managed tables at intelligent intervals.

```sql
ALTER CATALOG prod_finance ENABLE PREDICTIVE OPTIMIZATION;
```

Saves engineering effort and cost (it batches operations during low-cost windows).

### Q24. Auto Loader — how it scales to millions of files

`cloudFiles` source on Structured Streaming with two listing modes:
- **Directory Listing** (default) — periodically lists; fine for < 1M files.
- **File Notification** — Auto Loader sets up cloud-native notifications (S3 → SNS+SQS, Azure → Event Grid + Queue, GCS → Pub/Sub). Scales to billions of files; near-zero list cost.

```python
df = (spark.readStream.format("cloudFiles")
        .option("cloudFiles.format", "json")
        .option("cloudFiles.useNotifications", "true")
        .option("cloudFiles.schemaLocation", "/Volumes/.../_schema")
        .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
        .load("abfss://raw@.../events/"))
```

**Features:** schema inference + evolution, rescued data column, exactly-once with RocksDB-backed source progress, backfill mode.

---

## 8. Jobs, Workflows & Delta Live Tables

### Q25. Production-grade workflow design (nightly ETL)

**Architecture:**
1. **Trigger:** cron at 02:00, or file-arrival trigger on landing zone.
2. **Task 1 — Ingest** (Auto Loader to Bronze) on a small job cluster.
3. **Task 2 — Quality gate**: notebook running DQ checks; sets task value.
4. **Task 3 — DLT pipeline** for Silver → Gold transformations with expectations.
5. **Task 4 — Refresh** Materialised Views for BI.
6. **Task 5 — Notify** downstream systems (webhook) and post run summary to Slack.

**Reliability features:**
- Per-task `max_retries=3`, `retry_on_timeout=true`, `min_retry_interval_millis=60000`
- SLA: `timeout_seconds` + deadline alert
- Notifications: `on_failure`, `on_duration_warning`, `on_streaming_backlog`
- Run-as a Service Principal for deterministic identity
- Job parameters + dynamic value references (`{{job.start_time.iso_date}}`)
- Idempotency via *idempotency tokens* when triggered via API
- Lineage automatically captured in UC (column-level for DLT)

### Q26. Delta Live Tables (DLT) — internals

DLT is a **managed declarative ETL framework**. You define streaming tables and materialised views via Python decorators or SQL; the DLT runtime resolves the DAG, provisions compute, manages checkpoints, enforces expectations, and emits lineage.

```python
import dlt
from pyspark.sql.functions import *

@dlt.table(comment="Raw orders", table_properties={"quality": "bronze"})
def orders_bronze():
    return (spark.readStream.format("cloudFiles")
              .option("cloudFiles.format", "json")
              .load("/Volumes/raw/orders/"))

@dlt.table(comment="Cleaned orders")
@dlt.expect_or_drop("valid_amt", "amount > 0")
@dlt.expect_or_fail("valid_id", "order_id IS NOT NULL")
def orders_silver():
    return (dlt.read_stream("orders_bronze")
              .withColumn("amount", col("amount").cast("decimal(18,2)")))
```

**Modes:**
- **Triggered** — runs to completion and stops. Cheaper for batch.
- **Continuous** — always-on streaming. Lower latency.

**Event hooks:** the DLT event log records every commit, expectation result, dependency change.
**CDC:** `APPLY CHANGES INTO` handles SCD Type 1 and Type 2 declaratively.

### Q27. Streaming Tables vs Materialised Views

| Aspect | Streaming Table | Materialised View |
|---|---|---|
| Refresh | Incremental on append-only sources | Incremental when possible, else full recompute |
| Use case | Bronze/Silver ingest | Gold aggregations served to BI |
| State | Checkpoint-managed | Snapshot-managed |
| Defined in | SQL or Python DLT | SQL DLT or Databricks SQL |

---

## 9. Monitoring, Logging & FinOps

### Q28. Most-used system tables for a platform admin

| Table | Purpose |
|---|---|
| `system.billing.usage` | DBU consumption per workspace, sku, tags |
| `system.billing.list_prices` | Pricing for cost calculations |
| `system.access.audit` | All API/UI actions, identity, IP |
| `system.access.table_lineage` | Read/write lineage with column detail |
| `system.access.column_lineage` | Column-to-column lineage |
| `system.compute.clusters` | Cluster definitions + change history |
| `system.compute.node_timeline` | Per-node CPU/mem/network metrics |
| `system.lakeflow.jobs` / `job_run_timeline` | Job and task runs |
| `system.query.history` | SQL queries with stats |
| `system.information_schema` | All UC objects, grants, tags |

```sql
-- Top 10 most expensive jobs last 7 days
SELECT u.usage_metadata.job_id, SUM(u.usage_quantity * p.pricing.default) AS cost
FROM   system.billing.usage u
JOIN   system.billing.list_prices p
  ON   p.sku_name = u.sku_name
  AND  u.usage_date BETWEEN p.price_start_time AND COALESCE(p.price_end_time, current_date())
WHERE  u.usage_date >= current_date() - 7
  AND  u.usage_metadata.job_id IS NOT NULL
GROUP BY 1 ORDER BY cost DESC LIMIT 10;
```

### Q29. FinOps strategy for Databricks

1. **Tag everything:** mandate `cost-center`, `env`, `owner`, `project` via cluster policies; tags propagate to cloud invoice.
2. **Right compute for right work:** Job clusters for prod, SQL Warehouses for BI, Serverless for spiky workloads, Photon where it pays off.
3. **Autoscale + auto-terminate:** default 30-min idle; min workers low.
4. **Use spot/preemptible** for non-critical workers (`first_on_demand=1` for driver).
5. **Predictive Optimization** + Auto-compaction for storage cost reduction.
6. **Budget alerts** at account level + per cost-center dashboards.
7. **Reservations / commits** for steady DBU usage (Databricks committed-spend discount).
8. **Weekly review** of top-10 jobs by DBU; refactor offenders.
9. **Quotas** per workspace (max concurrent clusters, max workers).
10. **Lifecycle:** orphan workspace cleanup; archive cold Delta partitions to lower storage tier.

---

## 10. CI/CD, DevOps & Infrastructure-as-Code

### CI/CD Pipeline Flow

```
[Git Repo] → [CI: Lint+Tests] → [DAB Deploy DEV] → [Integration Tests] → [DAB Deploy PROD (approval)]
                                          │
                                          └── Terraform manages workspace, policies, UC, permissions
```

### Q30. CI/CD pipeline design

**Two streams of changes:**
1. **Infrastructure** (workspace, clusters, policies, UC catalogs, permissions, secrets, jobs) → **Terraform** (databricks provider). PRs reviewed; `plan` in CI; `apply` via manual approval.
2. **Code & jobs** (notebooks, wheels, DLT, asset definitions) → **Databricks Asset Bundles (DABs)**. `databricks bundle validate` + `deploy` per environment.

**Pipeline stages (GitHub Actions example):**
```yaml
- name: Validate bundle
  run: databricks bundle validate -t dev

- name: Run unit tests
  run: pytest tests/ --junitxml=results.xml

- name: Deploy to dev
  run: databricks bundle deploy -t dev

- name: Run integration job
  run: databricks bundle run integration_tests -t dev

- name: Deploy to prod (with approval)
  if: github.ref == 'refs/heads/main'
  run: databricks bundle deploy -t prod
```

**Branching:** trunk-based with short-lived feature branches; PR to `main`; main always deployable.
**Auth in CI:** OIDC federation → Databricks service principal. No PATs.

### Q31. Databricks Asset Bundle example (databricks.yml)

```yaml
bundle:
  name: customer_etl

include:
  - resources/*.yml

variables:
  catalog: { description: "UC catalog", default: "dev_finance" }

targets:
  dev:
    workspace: { host: https://adb-1234.azuredatabricks.net }
    variables: { catalog: "dev_finance" }
    run_as: { service_principal_name: sp-de-dev }
  prod:
    workspace: { host: https://adb-5678.azuredatabricks.net }
    variables: { catalog: "prod_finance" }
    run_as: { service_principal_name: sp-de-prod }
    mode: production

resources:
  jobs:
    nightly_etl:
      name: nightly-etl-${bundle.target}
      tasks:
        - task_key: ingest
          notebook_task: { notebook_path: ./notebooks/ingest.py }
          job_cluster_key: small
        - task_key: transform
          depends_on: [{ task_key: ingest }]
          pipeline_task: { pipeline_id: ${resources.pipelines.silver_pipe.id} }
      job_clusters:
        - job_cluster_key: small
          new_cluster:
            spark_version: 15.4.x-scala2.12
            node_type_id: Standard_D8ds_v5
            num_workers: 2
      schedule: { quartz_cron_expression: "0 0 2 * * ?", timezone_id: UTC }
```

### Q32. Terraform snippet — end-to-end workspace setup

```hcl
resource "databricks_catalog" "prod" {
  name    = "prod_finance"
  comment = "Production finance catalog"
  properties = { env = "prod" }
}

resource "databricks_schema" "gl" {
  catalog_name = databricks_catalog.prod.name
  name         = "gl"
}

resource "databricks_grants" "gl_grants" {
  schema = "${databricks_catalog.prod.name}.${databricks_schema.gl.name}"
  grant {
    principal  = "fin-analysts"
    privileges = ["USE_SCHEMA", "SELECT"]
  }
}

resource "databricks_cluster_policy" "shared" {
  name = "shared-analytics"
  definition = jsonencode({
    "spark_version"           = { type = "regex",     pattern = "^15\\..*" }
    "node_type_id"            = { type = "fixed",     value = "Standard_D8ds_v5" }
    "autotermination_minutes" = { type = "range",     minValue = 10, maxValue = 60 }
    "custom_tags.cost-center" = { type = "fixed",     value = "FIN-001" }
    "init_scripts.*"          = { type = "forbidden" }
  })
}
```

---

## 11. Migration, DR & High Availability

### Q33. DR plan with RTO=4h, RPO=15min

**Active-Passive cross-region (most common):**
1. **Workspace:** stand up identical secondary workspace in DR region; managed by Terraform.
2. **Identity:** account-level identities + SSO already global. No replication needed.
3. **UC metastore:** parallel metastore in DR region; sync catalog/schema definitions via Terraform.
4. **Data replication:**
   - S3 → Cross-Region Replication, or ADLS → GRS / object replication. RPO ~15 min.
   - For tables that demand stricter RPO: **Delta Deep Clone** on a scheduled job: `CREATE OR REPLACE TABLE dr.tbl DEEP CLONE prod.tbl;`
   - **Delta Sharing** is *not* for DR — read-only and cross-account, not cross-region replica.
5. **Code:** Git is source of truth; CD pipeline can deploy to either region.
6. **Secrets:** AKV with geo-redundant tier or paired-region replication.
7. **Workflows / DLT:** deployed via DABs to both regions; activated in DR only on failover.
8. **Runbook:** automated DNS cutover + DAB `deploy -t dr`.

**RTO 4h achievable** if secondary is warm (clusters defined but not running) and Terraform/DAB deploy is fully automated.

### Q34. Migrating Hive Metastore to Unity Catalog

**Use the UCX toolkit (Databricks Labs).** Phased migration:
1. **Assess:** UCX inventory — catalogs, tables, mounts, ACLs, jobs, init scripts, code references.
2. **Plan namespaces:** map HMS DBs → UC schemas; choose target catalog per business unit.
3. **Create UC artefacts:** storage credentials, external locations, catalogs, schemas.
4. **Migrate identity:** workspace-local groups → account-level groups (UCX scripts).
5. **Migrate tables:**
   - External tables → `SYNC SCHEMA ... TO uc_catalog.uc_schema`
   - Managed HMS tables → `CREATE TABLE uc.schema.t DEEP CLONE hive_metastore.db.t` then redirect writes.
6. **Update code:** switch to three-level names; remove DBFS mounts.
7. **Migrate permissions:** map ACL grants to UC GRANTs.
8. **Cutover & decommission:** read-only HMS for safety period; then disable.

---

## 12. Scenario-Based & Troubleshooting Questions

### Q35. Nightly job that ran in 30 min now takes 3 hours — diagnosis

1. **Compare with last good run** in Job UI (same input volume? same code?).
2. **Spark UI → SQL tab**: find stages with longest duration; check for skew.
3. **Stages tab:** task time distribution; max/median > 5 suggests skew.
4. **Executors:** spilled bytes? GC time? If high → memory pressure.
5. **Storage:** too many small files? `DESCRIBE DETAIL` on Delta tables; if `numFiles` exploded → run `OPTIMIZE`.
6. **Statistics:** ANALYZE TABLE if stats outdated.
7. **Code:** any new join? Check broadcast threshold. Use `EXPLAIN FORMATTED`.
8. **Cluster:** was the SKU changed? Autoscaling actually scaling? Spot reclaim?
9. **External system:** source DB slowdown? Storage account throttling (Azure 20k req/s)?
10. **Fix:** apply Z-Order/Liquid clustering on hot column; enable AQE skew join; salt; increase shuffle partitions; switch to Photon.

### Q36. New team needs Databricks access — onboarding walkthrough

1. **Identify use case** — ETL, ML, BI? Determines workspace and compute
2. **Provision identities** — sync AAD/Okta group via SCIM
3. **Assign to workspace** at account level
4. **Create UC catalog/schema** for the team; grant `USE CATALOG`, `CREATE SCHEMA`
5. **Apply cluster policy** matching their tier (e.g., "data-engineer-policy")
6. **Set up Git Folder / Repo** integration
7. **Provision secret scope** (KV-backed) for their credentials
8. **Add tags** for chargeback
9. **Provide onboarding doc + Databricks Academy links**
10. **Set budget alert** on their cost-center tag

---

## 13. Quick-Fire Conceptual Questions

| Question | Crisp answer |
|---|---|
| What is a DBU? | Databricks Unit — per-second compute billing unit. Rate varies by SKU (Jobs Compute < All-Purpose) and tier (Premium/Enterprise) and engine (Photon higher rate). |
| DBFS vs Volume? | DBFS = legacy workspace-local FS (not governed). Volume = UC-governed file location (recommended). |
| What replaces mounts in UC? | External Locations + Volumes. |
| REORG vs OPTIMIZE? | REORG applies physical changes from schema evolution (e.g., dropped columns); OPTIMIZE compacts files. |
| What is Photon? | C++ SIMD execution engine for columnar batches that replaces Spark Row execution where supported. |
| Disk cache vs Spark cache? | Disk cache (formerly Delta cache) is automatic, SSD-resident, decompressed Parquet data. Spark cache is RDD/DataFrame in-memory. |
| OPTIMIZE vs VACUUM? | OPTIMIZE compacts/clusters; VACUUM permanently deletes files older than retention threshold (default 7 days). |
| What is AQE? | Adaptive Query Execution — runtime re-optimisation of joins, partition coalesce, skew handling. |
| What is DBR for ML? | Databricks Runtime for ML — includes XGBoost, scikit-learn, MLflow, TensorFlow, PyTorch pre-installed. |
| Workflows pricing difference? | Jobs compute SKU is ~50% cheaper DBU rate than All-Purpose. |
| What is Delta Sharing? | Open protocol to share Delta tables across orgs without copying; recipients use any compatible engine. |
| What is a Lakehouse? | Data architecture unifying warehouse-style ACID + BI with lake-style scale + ML on open formats. |
| Init script types? | Cluster-scoped (preferred, via Volumes/Workspace), global (deprecated/restricted in UC). |
| What does SCC stand for? | Secure Cluster Connectivity — no public IPs, outbound-only relay. |
| Token vs OAuth? | PAT is static bearer token; OAuth (U2M/M2M) uses short-lived tokens, more secure. |
| Account console vs Workspace? | Account = global config (identity, billing, metastore, networks). Workspace = compute/notebooks/jobs. |
| What is DAB? | Databricks Asset Bundle — YAML IaC for jobs/pipelines/notebooks/dashboards. |
| Can UC span clouds? | Metastore is per region (cloud). Cross-cloud sharing via Delta Sharing. |
| MLflow Model Registry under UC? | Models are first-class UC objects with GRANTs, lineage, versions, aliases. |
| Workspace files vs Repos? | Both Git-aware; Workspace files now subsume Repos (single experience). |

---

## 14. Cheat-Sheet: CLI, API & Useful SQL

### Databricks CLI
```bash
databricks auth login --host https://adb-1234.azuredatabricks.net
databricks clusters list -o JSON | jq '.[] | {name, state, driver_node_type_id}'
databricks jobs run-now --job-id 12345 --notebook-params '{"date":"2026-05-12"}'
databricks bundle deploy -t prod
databricks fs cp -r ./local /Volumes/main/raw/landing
databricks secrets put-secret prod-akv sf-pwd
databricks tokens create --comment ci --lifetime-seconds 3600
```

### REST API key endpoints
```
POST /api/2.1/jobs/create
POST /api/2.1/jobs/run-now
GET  /api/2.0/clusters/get?cluster_id=...
POST /api/2.0/cluster-policies/create
GET  /api/2.1/unity-catalog/catalogs
POST /api/2.1/unity-catalog/grants/...
POST /api/2.0/permissions/jobs/<job_id>
```

### Useful UC SQL
```sql
SHOW CATALOGS;
SHOW GRANTS ON TABLE cat.sch.tbl;
DESCRIBE EXTENDED cat.sch.tbl;
DESCRIBE HISTORY  cat.sch.tbl LIMIT 20;
DESCRIBE DETAIL   cat.sch.tbl;
SELECT * FROM cat.sch.tbl VERSION AS OF 100;
OPTIMIZE cat.sch.tbl ZORDER BY (region, dt);
VACUUM   cat.sch.tbl RETAIN 168 HOURS;
ANALYZE  TABLE cat.sch.tbl COMPUTE STATISTICS FOR ALL COLUMNS;
```

### Diagnostic Spark configs
```properties
spark.databricks.delta.optimize.maxFileSize       = 1073741824   # 1 GB
spark.databricks.delta.autoCompact.enabled        = true
spark.databricks.delta.optimizeWrite.enabled      = true
spark.databricks.io.cache.enabled                 = true
spark.sql.adaptive.enabled                        = true
spark.sql.adaptive.skewJoin.enabled               = true
spark.sql.autoBroadcastJoinThreshold              = 31457280     # 30 MB
```

---

## 15. Interview Strategy & Tips

### Structure every answer (STAR-T)
1. **Situation** — one-line context: where does this apply?
2. **Technical depth** — explain the mechanism (control plane, log structure, optimistic concurrency, etc.).
3. **Action / Best practice** — what a senior architect would recommend.
4. **Result / Trade-off** — what you gain and what you give up.
5. **Tooling** — Terraform / DAB / CLI snippet to show you've done it.

### Top do's
- Draw the control-plane / data-plane diagram on whiteboard — it impresses every panel.
- Mention Unity Catalog at every governance/security touch-point.
- Always cite a metric: "reduced cluster cost by 38% by switching to job clusters + spot + Photon".
- Speak in terms of *policies*, not *permissions*: "I enforce X via cluster policy".
- Know the difference between **workspace ACLs** and **UC privileges** cold.
- Know one DR strategy end-to-end.

### Top don'ts
- Don't say "DBFS" as a recommendation. Always Volumes.
- Don't recommend PATs in CI — use OAuth + service principals.
- Don't confuse **OPTIMIZE** and **VACUUM**.
- Don't say Delta is a database — it's a storage format with a transaction log.
- Don't promise Photon makes everything faster — it doesn't help Python UDFs.

### Questions to ask the interviewer
- How is governance organised today — single metastore, multiple metastores, or still on Hive?
- Which cloud, what's the network posture (public, VNet-injected, PL)?
- What's the FinOps maturity — chargeback, budgets, predictive optimisation?
- How do you handle DR — active-passive, RPO target?
- Standard CI/CD pattern — Terraform + Bundles or something else?

---

*End of Guide — Databricks Platform Admin / Architect Interview Master Pack*
