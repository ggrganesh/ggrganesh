# Databricks Platform Admin — Troubleshooting Runbook

> 28 Real-World Scenarios with Root-Cause Analysis.
> Cluster failures · Spark performance · Delta corruption · Streaming · UC access · Networking · FinOps incidents.
> Written from the perspective of a 10+ year Databricks Platform Admin.

---

## Scenario Structure

Every scenario follows the same transparent structure:

| Section | What it answers |
|---|---|
| **SYMPTOM** | What the user reports / observable behaviour |
| **TRIAGE** | First 90-second checks |
| **RCA** | Ranked root-cause possibilities |
| **DIAGNOSIS** | Exact UI paths, queries, log files |
| **TEMP FIX** | Immediate mitigation |
| **PERMANENT FIX** | Proper engineering remediation |
| **PREVENTION** | So it never recurs |

---

## Table of Contents

**A. Cluster Issues**
1. Cluster fails to start — Cloud Provider Launch Failure
2. Cluster fails to start — Init script timeout
3. Driver unresponsive / cluster hangs
4. Executors keep getting lost (heartbeat timeout)
5. Autoscaling not scaling up under load
6. Cluster startup is unusually slow (10+ min)

**B. Spark Performance & Memory**
7. Data skew — one task takes 90% of stage time
8. Shuffle spill / OOM during shuffle stage
9. Driver OOM (GC overhead limit / heap)
10. Executor OOM mid-stage
11. Small files problem — query reads millions of files
12. Broadcast join failure — "broadcasted table too large"
13. AQE not optimising / wrong physical plan

**C. Delta Lake & Storage**
14. ConcurrentAppendException on Delta writes
15. VACUUM removed files still referenced by time-travel readers
16. Delta table corruption / cannot read commit log
17. ADLS / S3 throttling — 429/503 storms

**D. Streaming & Auto Loader**
18. Auto Loader not picking up new files
19. Structured Streaming backlog growing endlessly

**E. Unity Catalog & Access**
20. Sudden "PERMISSION_DENIED" on a table that worked yesterday
21. External table works in dev but fails in prod (storage credential)

**F. Jobs & Workflows**
22. Flaky job — fails intermittently with no code change
23. Job succeeds but produces wrong data (silent DQ failure)

**G. Cost & FinOps**
24. Unexpected DBU spike — bill anomaly
25. All-purpose cluster running 24/7 nobody owns

**H. Networking & Identity**
26. Cluster cannot reach external API — connection refused
27. PrivateLink failing — control plane unreachable
28. OAuth M2M token works locally, fails in cluster

**Appendices**
- I. The "First 5 Checks" mental model
- II. Spark UI — what to look at in each tab
- III. Critical logs & their locations
- IV. Diagnostic Spark configs with safe ranges
- V. System tables — copy-paste diagnostic queries

---

## A. Cluster Issues

### #1 — Cluster fails to start: "Cloud Provider Launch Failure" — Critical

**SYMPTOM:**
User clicks Start → after 3–8 minutes cluster shows `TERMINATED` with reason `CLOUD_PROVIDER_LAUNCH_FAILURE` or `BOOTSTRAP_TIMEOUT`. Sometimes only a subset of workers fails.
```
Cluster terminated. Reason: CLOUD_PROVIDER_LAUNCH_FAILURE
Message: AllocationFailed: Allocation failed. We do not have sufficient capacity
         for the requested VM size in this region.
```

**TRIAGE — first 90s:**
1. Open the *Event Log* tab → read the exact terminal event line.
2. Note the **node type**, **region**, **availability zone**, **VNet/subnet**.
3. Check Azure/AWS service health dashboard for the region.

**ROOT CAUSE (in order of frequency):**
1. **Capacity exhaustion** (e.g., `Standard_E32ds_v5` not available in zone).
2. **Subnet IP exhaustion** — cluster needs N×2 IPs; subnet /26 holds ~59 usable.
3. **NSG / NACL change** — egress to `443` blocked, can't reach control-plane relay.
4. **Spot reclamation** — entire pool of spot VMs revoked before bootstrap completes.
5. **IAM role / Managed Identity** lost permission to attach disk / pull image.
6. **Quota** exceeded on cores in subscription/account.
7. **Bootstrap script** downloading from blocked URL (PyPI, Maven) — see #2.

**DIAGNOSIS — exact paths:**
- UI: `Compute → <cluster> → Event Log`. Click each TERMINATED or DRIVER_HEALTHY event for `cause` JSON.
- Driver bootstrap log: `/databricks/init_scripts/*.log` (via Logs tab) or cluster log delivery path.
- Cloud side:
  - **Azure**: `az vm list-usage --location eastus2 --query "[?currentValue>=limit]"`
  - **AWS**: CloudTrail → `RunInstances` events with `errorCode`.
- System table:
```sql
SELECT event_time, event_type, event_details
FROM   system.compute.clusters
WHERE  cluster_id = '1234-567890-abc'
ORDER BY event_time DESC LIMIT 20;
```

**TEMPORARY FIX:**
1. Switch **availability zone** in cluster config (or remove zone pinning).
2. Fall back to an **alternate node type** in same family (e.g., `D8ds_v5 → D8ds_v4`).
3. Reduce **num_workers** and re-launch to confirm subnet/quota is the issue.
4. Disable spot temporarily (`spot_bid_max_price = -1` → on-demand).

**PERMANENT FIX:**
1. **Right-size subnets**: each Databricks workspace subnet should be at least `/22` for production. Re-IP if too small.
2. **Use Instance Pools** with pre-warmed capacity and allowlist of fallback SKUs.
3. **Request capacity reservation** in Azure or **Capacity Block** in AWS for steady prod workloads.
4. **Quota increase** through cloud support.
5. **Cluster policy**: pin only families with sufficient quota (regex on `node_type_id`).
6. **Hybrid spot**: `first_on_demand = 1` so driver is always on-demand; only workers can be spot.

**PREVENTION:**
- Alert on quota > 80% via cloud monitor.
- Daily dashboard: subnet IP usage, pool health, cluster start failure rate per workspace.
- Synthetic canary cluster that starts every hour; page on failure.

---

### #2 — Cluster fails to start: Init script timeout / failure — High

**SYMPTOM:**
Cluster reaches *Pending* then *Terminated* with reason `INIT_SCRIPT_FAILURE`. Or every cluster start adds 6–10 minutes versus normal.
```
Cluster terminated. Reason: INIT_SCRIPT_FAILURE
script: /Volumes/admin/init/install_libs.sh exited with code 124 (timeout)
```

**TRIAGE:**
1. Identify which init script failed (cluster Event Log → `script_path`).
2. Try cluster start **without** init scripts to isolate.
3. Read the actual script log: `/databricks/init_scripts/<ts>_<script>.stderr` in cluster log delivery folder.

**ROOT CAUSE:**
1. **PyPI / Maven egress blocked** by firewall; script hangs on DNS or TLS.
2. **Package install order** creating circular dep resolution that loops.
3. **Apt mirror unreachable** (custom Ubuntu repo dropped).
4. Script uses **workspace path** that's no longer accessible from the cluster identity.
5. Script writes > available disk and bootstrap fills tmpfs.
6. Newer DBR removed an OS tool the script relies on (e.g., `python2`).

**DIAGNOSIS:**
```bash
# If log delivery is configured
ls dbfs:/cluster-logs/<cluster-id>/init_scripts/

# Read directly from Volumes
cat /Volumes/admin/cluster_logs/<date>/<cluster-id>/init_scripts/*.stderr
```
Look for:
- `Connection timed out` → network egress issue.
- `Could not find a version that satisfies the requirement` → mirror/version pin.
- `Permission denied` → identity / SELinux.
- Last printed line > 240s before cutoff → script is genuinely slow.

**TEMPORARY FIX:**
1. Detach the init script from cluster config; install packages at notebook scope with `%pip install`.
2. Add `set -ex` + `timeout 300` guards so failures surface fast.
3. Use a pre-built **Databricks Container Services (DCS)** image with libs baked in.

**PERMANENT FIX:**
1. Move libraries to a **cluster library** (wheel/JAR) pulled from internal **Artifactory/Nexus** via private endpoint — no internet at boot time.
2. For OS-level needs, build a **custom DCS image** in CI; promotion through dev/prod registries.
3. Store init scripts in `/Volumes/<catalog>/<schema>/<vol>/` (UC-governed, immutable references).
4. Cluster policy: forbid `init_scripts.*` except from an approved location.

**PREVENTION:**
- CI test: spin up an ephemeral cluster with the script every PR; fail build on any non-zero exit.
- Lint scripts: `shellcheck` + check for unbounded `curl`/`pip`.
- Alert on init-script duration percentiles via `system.compute.clusters`.

---

### #3 — Driver unresponsive / cluster hangs — Critical

**SYMPTOM:**
Notebook cells stop returning. Cluster UI shows green but *Spark UI* is unreachable or commands queue forever. Eventually:
```
Driver is up but is not responsive, likely due to GC.
```

**TRIAGE:**
1. Open *Spark UI → Executors* tab. Look at **Driver** row: GC time vs Task time.
2. *Metrics* tab: driver heap utilisation. If > 90% sustained → driver memory pressure.
3. Check if a single user did `collect()`, `toPandas()`, or huge `display()`.

**ROOT CAUSE:**
1. **collect()/toPandas()** on a large DataFrame pulled millions of rows into JVM heap.
2. **Broadcast variable** too large (often unintended — implicit broadcast join).
3. **Schema inference** on a huge JSON/CSV file scanning all records in driver.
4. **Driver-side UDF** work (Pandas-on-Spark falling back to single node).
5. Too many cached/persisted DataFrames; metastore client memory leak.
6. **Notebook output buffer** too large (printing inside loops).
7. Shared cluster: dozens of concurrent users overwhelm driver thread pool.

**DIAGNOSIS:**
- Spark UI → *Executors*: Driver "Total GC Time" should be < 10% of "Total Task Time". If > 30% → memory issue.
- Spark UI → *SQL/DataFrame*: look at latest query plan for `Exchange BroadcastExchange` nodes from very large tables.
- Take a heap dump:
```bash
%sh
jcmd $(pgrep -f DriverDaemon) GC.heap_dump /tmp/driver.hprof
ls -lh /tmp/driver.hprof
```
Open with Eclipse MAT to find dominant objects.
- `spark.sparkContext.statusTracker.getJobIdsForGroup(None)` → see hanging jobs.

**TEMPORARY FIX:**
1. Restart driver only: *Cluster → Restart driver*. Cheaper than full restart.
2. Tell users to replace `display(df)` with `display(df.limit(1000))`.
3. Set `spark.driver.maxResultSize=4g` to fail-fast instead of hang.
4. Increase driver to bigger SKU temporarily.

**PERMANENT FIX:**
1. Move heavy work to **job clusters**; never run long-running ETL on shared all-purpose clusters.
2. Provide users a **cluster policy** with appropriate driver SKU and `spark.driver.maxResultSize=2g`.
3. Adopt `spark.databricks.driver.disableScalaOutput=true` for very chatty notebooks (legacy DBR only).
4. Lint notebooks for `collect()/toPandas()` in CI.
5. For Pandas users: enforce **pandas-on-Spark** (`import pyspark.pandas as ps`) not raw Pandas.

**PREVENTION:**
- Monitor driver heap and GC via `system.compute.node_timeline` with alert at > 85%.
- Workspace setting: enforce notebook output size limits.
- Educate: "driver is a single JVM, not a cluster".

---

### #4 — Executors keep getting lost (heartbeat timeout) — High

**SYMPTOM:**
Stage retries multiple times. Log full of:
```
ExecutorLostFailure (executor 7 exited caused by one of the running tasks)
Reason: Executor heartbeat timed out after 124953 ms
```

**TRIAGE:**
1. Spark UI → *Executors* → look at "Failed Tasks" and "Removed" executors.
2. Check if loss is on **spot** instances → spot reclamation.
3. Check executor logs for the dead one: GC overhead? OOM-killer? Disk full?

**ROOT CAUSE:**
1. **Spot reclamation** by cloud — most common.
2. **Container killed by OOM-killer**: kernel killed JVM after exceeding cgroup memory.
3. **Disk full**: shuffle spill exhausted local NVMe.
4. **Network blip**: SDN throttling, NSG drops; heartbeat misses 6× default 10s.
5. **Long GC pause** stalls heartbeat thread.
6. **Bad worker node** hardware (rare; cloud retires automatically).

**DIAGNOSIS:**
```bash
# Look at the executor stderr in cluster log delivery
grep -E "OutOfMemory|Killed|No space left|CompletionService" \
  /Volumes/admin/cluster_logs/<cluster-id>/executor/*/stderr
```
```sql
-- Check disk usage during the run
SELECT node_id, percentile(disk_used_percent, 0.95) p95
FROM   system.compute.node_timeline
WHERE  cluster_id = '...' AND start_time > current_timestamp() - interval 1 hour
GROUP BY 1 ORDER BY p95 DESC;
```

**TEMPORARY FIX:**
1. Set `spark.network.timeout=600s` and `spark.executor.heartbeatInterval=60s` to ride out brief stalls.
2. Switch to on-demand workers (disable spot) for the run.
3. Reduce concurrent tasks per executor (`spark.executor.cores` from 8 → 4) to relieve memory pressure.

**PERMANENT FIX:**
1. For disk-spill workloads, choose node types with **local NVMe** (e.g., `L` series on Azure, `i`/`m6id` on AWS).
2. Enable `spark.databricks.io.cache.enabled` + `spark.databricks.delta.optimizeWrite.enabled` to reduce shuffle.
3. Memory-bound jobs: change node family to memory-optimised (Standard_E / r6).
4. Use job clusters (no spot for driver, hybrid spot for workers with `first_on_demand = max(1, num_workers/4)`).

**PREVENTION:**
- Dashboard: executor loss rate per workload + spot reclaim rate.
- Cluster policy: forbid spot for production jobs above N workers.

---

### #5 — Autoscaling not scaling up under load — Medium

**SYMPTOM:**
Cluster at min_workers but tasks queued; Spark UI shows pending stages. No new workers appearing.

**TRIAGE:**
1. Confirm autoscaling enabled and `min < max`.
2. Check Event Log for `UPSIZE_REQUESTED` events; presence vs absence is the key clue.
3. Confirm cluster mode supports autoscaling (DLT modes have their own scaler).

**ROOT CAUSE:**
1. **Job is single-stage with few partitions** — Spark says: *I don't need more cores*. Autoscaler is healthy.
2. **Number of partitions < max workers × cores** — work is not parallelisable beyond min.
3. **Optimized autoscaling lag** — scale-up decision waits ~30s of saturation.
4. Spot allocation failing silently (see #1).
5. Cluster policy caps `num_workers`.
6. Pool empty + cloud capacity exhausted.

**DIAGNOSIS:**
```python
# How many partitions does the stage actually have?
spark.sparkContext.getConf().get("spark.sql.shuffle.partitions")   # default 200
df.rdd.getNumPartitions()
```
Look at *Stages* tab — if "Tasks: Succeeded/Total" is e.g. `4/4` with cluster at 8 cores you can't scale further.

**TEMPORARY FIX:**
1. `df.repartition(N)` to a number that matches your target cores.
2. Set `spark.sql.shuffle.partitions = 4 × total cores at max`.
3. Raise `min_workers` for the run.

**PERMANENT FIX:**
1. Switch from standard autoscaling to **Optimized autoscaling** (jobs only) — scales aggressively up and gradually down.
2. Use **Photon** + **AQE** (`spark.sql.adaptive.coalescePartitions.enabled=true`) so partition count auto-tunes.
3. Right-size partition strategy at write time: `df.repartition(...)` before `.write`.

**PREVENTION:**
- Code review checklist: every wide stage must justify partition count.
- Dashboards: cluster CPU vs worker count over time.

---

### #6 — Cluster startup unusually slow (10+ minutes) — Medium

**SYMPTOM:**
Normal start time was 3–5 min; suddenly all clusters take 10–15 min to reach RUNNING.

**ROOT CAUSE:**
1. Init script downloading from slow / overloaded source.
2. Image pull from a far-region container registry (cross-region traffic).
3. Pool empty — every start = cold cloud VM provisioning.
4. Custom Container Service image too large (> 4 GB).
5. NAT gateway port exhaustion → slow outbound TLS handshakes.

**DIAGNOSIS:**
- Event log → time between `STARTING` and `DRIVER_HEALTHY`: that's bootstrap.
- If gap between `NODES_LOST` ↔ `UPSIZE_COMPLETED` is high → cloud-side latency.
- NAT GW SNAT port exhaustion → check Azure Monitor metrics on NAT GW; mitigate with multiple public IPs or VNet integration.

**PERMANENT FIX:**
1. Use **Instance Pools** with `min_idle_instances` = expected concurrency.
2. Mirror Python/JVM dependencies to **internal Artifactory** with private endpoint.
3. Slim DCS image; multi-stage build, prune apt cache.
4. Add second NAT GW or use multiple public IPs (Azure SNAT port pooling) — or move to private endpoints for all egress.
5. Where appropriate, switch jobs to **Serverless** — 5–10s startup, no bootstrap to tune.

---

## B. Spark Performance & Memory

### #7 — Data skew: one task takes 90% of stage time — Critical

**SYMPTOM:**
Stage shows 199 tasks complete in 30s; 1 task running for 45 minutes. Sometimes that single task OOMs.
```
org.apache.spark.shuffle.FetchFailedException: ...
java.lang.OutOfMemoryError: Java heap space
```

**TRIAGE:**
1. Spark UI → *Stages* → click the slow stage → *Task duration* distribution. Max/median > 5 → skew.
2. "Shuffle Read Size" of slowest task — if 10 GB vs median 50 MB, you have a hot key.
3. Identify the join key / groupBy key on which it's skewed.

**ROOT CAUSE:**
1. Real-world data has one mega-key (e.g., NULL `customer_id`, "Unknown" country, top advertiser).
2. Join on a low-cardinality column.
3. Window function partitioned by skewed column.
4. Partitions on disk are skewed → unbalanced reads.

**DIAGNOSIS:**
```sql
-- Find the skewed key
SELECT join_key, count(*) c
FROM   silver.events
GROUP BY 1 ORDER BY c DESC LIMIT 20;
```
Confirm AQE skew handling is on:
```properties
spark.sql.adaptive.enabled                                  = true
spark.sql.adaptive.skewJoin.enabled                         = true
spark.sql.adaptive.skewJoin.skewedPartitionFactor           = 5
spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes = 256MB
```

**TEMPORARY FIX:**
1. Enable AQE skew join (defaults are good in DBR 13+).
2. Filter out the known hot key (NULL, "Unknown") and union separately.
3. Bump cluster memory + reduce executor cores to give the hot task more headroom.

**PERMANENT FIX:**
1. **Key salting** for the join (when AQE isn't enough):
```scala
val salted = events.withColumn("salt", (rand() * 50).cast("int"))
val dims   = dim.withColumn("salt", explode(array((0 until 50).map(lit): _*)))
salted.join(dims, Seq("key", "salt"))
```
2. Pre-aggregate on the skewed side before the join.
3. Use **broadcast hash join** if one side < 100 MB after filters: `broadcast(dim)`.
4. Liquid clustering on the join key in source table.
5. Replace `groupBy + first/last` with `reduceByKey`-style aggregations.

**PREVENTION:**
- Profile keys monthly; alert when top-1 key is > 5% of total.
- DLT expectation: `EXPECT count(distinct key) > threshold`.

---

### #8 — Shuffle spill / OOM during shuffle stage — High

**SYMPTOM:**
Stage shows huge "Shuffle Spill (Memory)" and "Shuffle Spill (Disk)" columns. Sometimes followed by:
```
org.apache.spark.shuffle.FetchFailedException
java.io.IOException: No space left on device
```

**TRIAGE:**
1. Spark UI → Stage → Summary Metrics → check "Spill" rows.
2. Executors tab → "Disk Used" column. Anything > 70% per node is a red flag.
3. If spill is small but task still fails → likely OOM, not disk.

**ROOT CAUSE:**
1. Too few shuffle partitions for the data volume.
2. Executors have too many cores per memory unit (high concurrency, low RAM/core).
3. Local NVMe too small or workload runs on non-NVMe SKU.
4. Cartesian / cross join exploded data.
5. Skew (see #7) concentrating data on one executor.

**TEMPORARY FIX:**
1. Increase `spark.sql.shuffle.partitions` to 4× total cores at max workers.
2. Reduce `spark.executor.cores` from 8 → 4 to double memory per task.
3. Switch to higher-memory node type for the run.

**PERMANENT FIX:**
1. Use NVMe SKUs (`L`, `i3`, `m6id`) where shuffle is heavy.
2. Enable `spark.databricks.io.cache.enabled = true` (Disk Cache) — reduces re-shuffle on subsequent reads.
3. Adopt **Photon** — much more efficient memory model on shuffle-heavy SQL.
4. Rewrite plan to avoid the shuffle: bucketed tables, pre-aggregation, or broadcast.
5. Liquid clustering by the join key removes the shuffle entirely on the read-side.

---

### #9 — Driver OOM (GC overhead limit / heap exhausted) — High

**SYMPTOM:**
```
java.lang.OutOfMemoryError: GC overhead limit exceeded
Driver crashed; cluster shows DRIVER_UNAVAILABLE → cluster restarted
```

**ROOT CAUSE:** Same root-causes as #3 plus:
1. `collect()` / `toPandas()` / `display(huge_df)` in notebook.
2. Many large broadcast variables not released.
3. Iterative algorithm that grows DataFrame lineage forever — driver lineage tree OOM.
4. Tens of thousands of small files; driver maintains metadata for each.
5. MLflow autolog with huge param/metric dictionaries.

**DIAGNOSIS:**
Enable JVM heap dump on driver:
```properties
spark.driver.extraJavaOptions "-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/driver-oom.hprof"
```
After crash, retrieve `/tmp/driver-oom.hprof` via Cluster Logs / log delivery; analyse with Eclipse MAT.

**PERMANENT FIX:**
1. Move heavy results to a Delta table; consume by paging from a downstream tool, never `collect()`.
2. For iterative ML: `df = df.localCheckpoint()` every N iterations to truncate lineage.
3. For "many small files" symptom: run `OPTIMIZE` or enable predictive optimisation.
4. Bigger driver SKU is the last resort, not the first.

---

### #10 — Executor OOM mid-stage — High

**SYMPTOM:**
```
ExecutorLostFailure ... Reason: Container killed by YARN/Kubernetes for exceeding memory limits.
java.lang.OutOfMemoryError: Java heap space
```

**ROOT CAUSE:**
1. Skew (#7).
2. Wide DataFrame with hundreds of columns and Python UDFs (each row serialised to Python via Arrow).
3. Window function with unbounded frame.
4. Cache of a large DataFrame in memory (`cache()` not paired with proper memory).
5. Aggregation that materialises group dicts of unbounded cardinality (high-cardinality groupBy without pre-aggregation).

**TEMPORARY FIX:**
- Reduce `spark.executor.cores` from 8 → 4 (half concurrency, double memory per task).
- Increase partitions: `df.repartition(2000)`.
- Drop `cache()` calls.

**PERMANENT FIX:**
1. Replace Python UDFs with native Spark SQL functions or **Pandas UDFs (vectorised)**.
2. Switch to **Photon** for SQL/Delta — much smaller memory footprint per row.
3. Reduce data shape: `select` only needed columns before joins.
4. For windowing: enforce bounded frames.

---

### #11 — Small files problem — query reads millions of files — High

**SYMPTOM:**
Query that used to take 30s now takes 10 minutes. `DESCRIBE DETAIL` shows `numFiles` = 2,300,000. List operations dominate.

**ROOT CAUSE:**
1. Streaming writer with trigger interval too small (10s) creating thousands of files/hour.
2. Partitioning on high-cardinality column (e.g., `partitioned by (user_id)`).
3. Hourly micro-batch jobs that never compact.
4. Auto-optimize / auto-compact disabled.

**DIAGNOSIS:**
```sql
DESCRIBE DETAIL cat.sch.tbl;            -- numFiles, sizeInBytes, partitionColumns
SELECT file_path, file_size_bytes
FROM   `cat`.information_schema.tables_history
WHERE  table_name = 'tbl' LIMIT 100;
```
Average file size should be 64 MB – 1 GB. Anything < 16 MB → small file problem.

**TEMPORARY FIX:**
```sql
OPTIMIZE cat.sch.tbl;
VACUUM cat.sch.tbl RETAIN 168 HOURS;     -- after a safe period
```

**PERMANENT FIX:**
1. Enable on table:
```sql
ALTER TABLE cat.sch.tbl SET TBLPROPERTIES (
    'delta.autoOptimize.optimizeWrite' = 'true',
    'delta.autoOptimize.autoCompact'   = 'true'
);
```
2. Drop low-value partition columns; switch to **Liquid Clustering**.
3. Streaming writer: `.trigger(processingTime='5 minutes')` or use **availability-mode trigger** for files.
4. Enable **Predictive Optimization** at catalog level so OPTIMIZE/VACUUM run automatically.

---

### #12 — Broadcast join failure: "broadcasted table is too large" — Medium

**SYMPTOM:**
```
org.apache.spark.SparkException: Could not execute broadcast in 300 secs.
Cannot broadcast the table that is larger than 8GB.
```

**ROOT CAUSE:**
1. Statistics outdated; planner thinks dim table is small.
2. Explicit `broadcast(df)` hint on a non-small table.
3. Auto-broadcast threshold (`spark.sql.autoBroadcastJoinThreshold`) set too high.
4. The "small" side has exploded due to a recent ingest.

**DIAGNOSIS:**
```python
spark.sql("EXPLAIN FORMATTED SELECT ...").show(truncate=False)
```
Look for `BroadcastHashJoin` nodes; check the build side row count.

**TEMPORARY FIX:**
- Disable: `spark.sql.autoBroadcastJoinThreshold = -1`.
- Remove explicit `broadcast()` hint.
- Force shuffle hash / sort merge join with `/*+ MERGE(t) */` SQL hint.

**PERMANENT FIX:**
1. `ANALYZE TABLE ... COMPUTE STATISTICS FOR ALL COLUMNS`.
2. Tune `spark.sql.autoBroadcastJoinThreshold` to 100 MB (default 10 MB is conservative).
3. For Delta tables, **predictive I/O + statistics** is automatic when Photon is enabled.

---

### #13 — AQE not optimising / wrong physical plan — Medium

**SYMPTOM:**
Plan picks SortMergeJoin where a broadcast would be ideal, or doesn't coalesce partitions; query slow.

**ROOT CAUSE:**
1. AQE disabled or partial: `spark.sql.adaptive.enabled` off.
2. Stats stale; planner cardinality estimate way off.
3. Plan uses DataFrame APIs in a way that defeats CBO (e.g., chained DataFrames without intermediate stats).
4. Subqueries via `spark.createDataFrame()` on Python objects → no stats.

**PERMANENT FIX:**
1. Confirm all AQE features:
```properties
spark.sql.adaptive.enabled                              = true
spark.sql.adaptive.coalescePartitions.enabled           = true
spark.sql.adaptive.skewJoin.enabled                     = true
spark.sql.adaptive.localShuffleReader.enabled           = true
spark.sql.adaptive.advisoryPartitionSizeInBytes         = 64MB
```
2. Refresh statistics on relevant tables nightly.
3. Use Delta tables (auto-collected statistics) instead of Parquet/CSV.

---

## C. Delta Lake & Storage

### #14 — ConcurrentAppendException on Delta writes — High

**SYMPTOM:**
```
io.delta.exceptions.ConcurrentAppendException: Files were added to partition [date=2026-05-12]
by a concurrent update. Please try the operation again.
```

**ROOT CAUSE:**
1. Two writers updating overlapping file sets.
2. MERGE without a partition filter — locks the entire table file set.
3. Streaming writer + batch writer to same table.
4. Multiple jobs scheduled at the same minute targeting the same partition.

**TEMPORARY FIX:**
- Add retry with backoff at application layer (3 retries, exponential).
- Stagger job schedules (e.g., job A 02:00, job B 02:15).

**PERMANENT FIX:**
1. Always include **partition predicates** in MERGE/UPDATE WHERE clause; Delta uses them to scope the conflict set.
2. Split a single huge merge into per-partition merges, run in parallel safely.
3. For high-frequency CDC use **APPLY CHANGES INTO** in DLT — handles concurrency for you.
4. Adopt **row-level concurrency** (Delta protocol >= reader 3 / writer 7) — reduces conflicts to per-file granularity.
5. Architecturally: one writer per table is the cleanest rule. Multiple readers OK.

---

### #15 — VACUUM removed files still in use by long readers — Critical

**SYMPTOM:**
Long-running queries fail mid-flight with:
```
FileNotFoundException: dbfs:/.../part-00007-xxxx.snappy.parquet
A file referenced in the transaction log cannot be found.
```

**ROOT CAUSE:**
1. Someone ran `VACUUM ... RETAIN 0 HOURS` to "save space".
2. Stream checkpoint pointing to an old snapshot > retention.
3. Time-travel query (`VERSION AS OF`) past retention window.

**TEMPORARY FIX:**
1. If cloud storage has **soft-delete / versioning**, restore the deleted files.
2. If not — recover from previous backup / DR copy, or rebuild table from source.
3. Restart streaming with a fresh checkpoint after re-bootstrapping the bronze table.

**PERMANENT FIX:**
1. Never reduce `delta.deletedFileRetentionDuration` below 7 days for production.
2. Set:
```sql
ALTER TABLE cat.sch.tbl SET TBLPROPERTIES (
  'delta.deletedFileRetentionDuration' = 'interval 30 days',
  'delta.logRetentionDuration'          = 'interval 30 days'
);
```
3. Enable **cloud storage versioning** (ADLS soft delete / S3 versioning) — single most important safety net.
4. Cluster policy: forbid `spark.databricks.delta.retentionDurationCheck.enabled = false`.
5. Use Predictive Optimization to manage VACUUM safely.

---

### #16 — Delta table corruption / cannot read commit log — Critical

**SYMPTOM:**
```
io.delta.exceptions.MetadataChangedException
Cannot read commit log: _delta_log/00000000000000123456.json
```

**ROOT CAUSE:**
1. Manual deletion of `_delta_log` files via storage UI.
2. A non-Delta tool (Hadoop FileSystem API, az copy) overwrote a JSON commit.
3. Cloud storage eventual consistency on legacy buckets (rare on modern ADLS/S3).
4. Two writers with different protocol versions clashed.
5. Checkpoint file written partially (process killed mid-write).

**DIAGNOSIS:**
```python
# List the log to see what's missing
dbutils.fs.ls("abfss://.../table/_delta_log/")
```
```sql
-- Get versions still healthy
DESCRIBE HISTORY cat.sch.tbl;
```

**TEMPORARY FIX:**
1. Restore the missing JSON from storage versioning.
2. If unrecoverable, rebuild table from last good checkpoint:
```sql
CREATE TABLE cat.sch.tbl_recovered DEEP CLONE
  cat.sch.tbl VERSION AS OF <last_good>;
```

**PERMANENT FIX:**
1. Enable cloud storage versioning (mandatory for prod tables).
2. Restrict storage account writes to the UC storage credential identity only; revoke human/SA write access at storage layer.
3. UC-managed tables are safer than external (UC + Predictive Optimization controls writes).
4. Set `delta.checkpoint.writeStatsAsStruct = true` to make checkpoints richer and more robust.

---

### #17 — Storage throttling — ADLS 429 / S3 503 SlowDown — High

**SYMPTOM:**
```
Operation failed: "This request is not authorized to perform this operation. Server busy."
azure: 429 SubscriptionRequestThrottling
or
S3 SlowDown: Please reduce your request rate.
```

**ROOT CAUSE:**
1. Job creates millions of LIST/HEAD requests on a single prefix (small files).
2. Auto Loader directory listing mode at scale.
3. Multiple jobs hammering same storage account; account-level request cap reached.
4. Hot partition (S3 partitions by prefix).

**TEMPORARY FIX:**
- Reduce parallel readers (lower executor count).
- Switch Auto Loader to **file notification mode**.
- Run `OPTIMIZE` to reduce file count.

**PERMANENT FIX:**
1. Spread tables across multiple storage accounts; one account per business domain.
2. Use **premium ADLS Gen2 with hierarchical namespace**; supports higher TPS.
3. On S3, scale prefixes (e.g., `year=2026/month=05/day=12/hour=09/`) so requests fan-out.
4. Use Auto Loader file notification, not list mode.
5. Enable **disk cache** on workers.

---

## D. Streaming & Auto Loader

### #18 — Auto Loader not picking up new files — High

**SYMPTOM:**
New files land in the source path but the stream never processes them. `numNewFiles` = 0 in progress logs.

**ROOT CAUSE:**
1. Notification mode set up incorrectly — event grid not bound, SQS subscription deleted.
2. File arrives outside the source path glob.
3. File extension filtered out by `cloudFiles.fetchParallelism` mismatch.
4. Listing checkpoint corrupted; stream thinks file already seen.
5. RocksDB state corrupted after a forced kill.
6. IAM/Managed identity lost permission to source path.

**DIAGNOSIS:**
```scala
spark.streams.active                              // list streams
val q = spark.streams.get("<query-id>")
q.lastProgress                                    // JSON with sources / metrics
```
```bash
# Inspect cloudFiles state
dbutils.fs.ls("<checkpoint>/sources/0/rocksdb/")

# For notification mode (Azure):
az eventgrid event-subscription list --source-resource-id /subscriptions/.../storageAccounts/X
```

**TEMPORARY FIX:**
1. Restart stream with `cloudFiles.includeExistingFiles = true` and a fresh checkpoint to re-read.
2. Switch temporarily to directory listing mode to confirm files exist.

**PERMANENT FIX:**
1. Manage Auto Loader subscriptions via Terraform; alert if event-grid subscription disappears.
2. Standardise on file notification mode for production streams.
3. Add file arrival monitoring — alert when no new files for X minutes.

---

### #19 — Structured Streaming backlog growing endlessly — High

**SYMPTOM:**
`inputRowsPerSecond` > `processedRowsPerSecond`. Lag grows hour over hour; eventually checkpoint storage explodes.

**ROOT CAUSE:**
1. Stage doing wide shuffles within the streaming micro-batch.
2. State store growing unbounded (stateful operations without watermark).
3. Sink (Delta MERGE) becoming a bottleneck under concurrent writes.
4. Skew on partition key.
5. Cluster too small for sustained throughput.

**DIAGNOSIS:**
```scala
q.recentProgress.last.batchDuration                  // target < trigger interval
q.recentProgress.last.stateOperators("numRowsTotal") // state row count
// Spark UI Streaming tab: input vs processing rate
```

**PERMANENT FIX:**
1. Add **watermark** on event time + drop late data: `.withWatermark("ts", "1 hour")`.
2. Use **RocksDB state store**: `spark.sql.streaming.stateStore.providerClass = org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider`.
3. For MERGE sinks: partition by event-time + use Liquid clustering on join key.
4. Switch to **DLT continuous mode** — DLT autoscales for backlog.
5. Right-size: target batchDuration ≤ 50% of trigger interval; scale workers until met.

---

## E. Unity Catalog & Access

### #20 — Sudden "PERMISSION_DENIED" on table that worked yesterday — High

**SYMPTOM:**
```
[INSUFFICIENT_PERMISSIONS] User does not have SELECT on TABLE finance.gl.journal.
Reason: Missing USE CATALOG.
```

**ROOT CAUSE:**
1. A SCIM sync removed user from the granting group.
2. Cluster mode changed to "No Isolation Shared" → UC not enforced (but legacy ACLs differ).
3. Owner of table changed; prior grants kept but a new row-filter requires extra privilege.
4. Catalog moved to a new metastore; old grants did not migrate.
5. Underlying external location's storage credential lost cloud-side permission.
6. User's account-level group disabled.

**DIAGNOSIS:**
```sql
SHOW GRANTS 'user@corp.com' ON TABLE finance.gl.journal;
SHOW GRANTS ON CATALOG finance;
SHOW GRANTS ON SCHEMA  finance.gl;

-- See last 50 grant changes
SELECT event_time, action_name, request_params
FROM   system.access.audit
WHERE  action_name LIKE '%Grant%'
  AND  event_time > current_timestamp() - interval 1 day
ORDER BY 1 DESC;
```

**TEMPORARY FIX:**
- Grant the user (or their group) the missing privilege explicitly.
- If access mode of cluster is wrong, restart on Shared/Assigned UC-compliant mode.

**PERMANENT FIX:**
1. Grants must always be to groups, never users. Manage groups in IdP.
2. SCIM source-of-truth documented; alert on group membership change for critical groups.
3. Terraform manages UC grants — any drift detected by daily plan run.
4. Storage credential rotation runbook: validate `CHECK STORAGE CREDENTIAL` after every IAM change.

---

### #21 — External table works in dev but fails in prod (storage credential) — Medium

**SYMPTOM:**
```
PERMISSION_DENIED: The current user does not have READ FILES on EXTERNAL LOCATION 'abfss://prod@...'.
or
Storage Credential validation failed: AuthorizationFailure (403)
```

**ROOT CAUSE:**
1. Storage credential's Managed Identity / IAM role lacks RBAC on the storage account.
2. Storage account firewall blocks Databricks' network.
3. External Location not granted to the running identity.
4. Path mismatch: registered external location is `abfss://prod@xxx/` but user reads `https://xxx.blob.core.windows.net/prod/...`.

**DIAGNOSIS:**
```sql
-- Validates IAM/MI + network + path
VALIDATE STORAGE CREDENTIAL prod_cred;
VALIDATE EXTERNAL LOCATION prod_loc;

-- See which paths a user can access
SHOW GRANTS ON EXTERNAL LOCATION prod_loc;
```

**PERMANENT FIX:**
1. Use Managed Identity per environment; assign *Storage Blob Data Contributor* on container.
2. Storage account firewall allows Databricks subnet or private endpoint only.
3. Grant READ/WRITE FILES on external location to the same groups in dev and prod.
4. Always use `abfss://` / `s3://` URIs through UC, never the underlying HTTPS form.

---

## F. Jobs & Workflows

### #22 — Flaky job — fails intermittently with no code change — High

**SYMPTOM:**
Same job code, same data; succeeds 70% of the time. Failures distribute across different tasks.

**ROOT CAUSE:**
1. Spot reclamation mid-stage (most common).
2. Race on shared Delta table (concurrent writer).
3. Upstream system flakiness (Kafka/Kinesis/JDBC source).
4. Cluster autoscaling adding workers from a misconfigured pool.
5. Pip install at runtime hitting transient PyPI errors.
6. Library version drift — `%pip install pkg` without pin.

**DIAGNOSIS:**
```sql
SELECT run_id, state.result_state, state.state_message,
       creator_user_name, period_start_time
FROM   system.lakeflow.job_run_timeline
WHERE  job_id = 12345
  AND  period_start_time > current_timestamp() - interval 7 days
ORDER BY 4 DESC;
```
Plot success/failure across times; correlate with spot reclaim windows in cloud monitor.

**PERMANENT FIX:**
1. For prod: `first_on_demand = num_workers` (no spot) until reliability improves.
2. Task-level retries: 2–3 with 60s backoff.
3. Pin all libraries (wheel + lock file via Bundles).
4. Make every task **idempotent** (use Delta MERGE on a natural key, or partition-overwrite).
5. Use `idempotency_token` on REST job submissions.

---

### #23 — Job succeeds but produces wrong data (silent DQ failure) — Critical

**SYMPTOM:**
Numbers in BI report look wrong by 12%. Source data unchanged.

**ROOT CAUSE:**
1. Schema evolution: source added a column; ETL ignored it or default-filled wrong.
2. Time-zone bug — date filter off by one day.
3. MERGE clause used a non-unique business key, causing duplicate matches.
4. Race condition: streaming + batch into the same partition.
5. Bad data in a join table from an upstream incident.
6. Auto Loader's `_rescued_data` column hiding malformed rows everyone ignores.

**DIAGNOSIS:**
```sql
-- Time travel to compare
SELECT count(*) FROM gold.fact_sales VERSION AS OF 42;
SELECT count(*) FROM gold.fact_sales VERSION AS OF 43;

-- Find rescued rows
SELECT _rescued_data, count(*) FROM silver.events
WHERE _rescued_data IS NOT NULL GROUP BY 1;
```

**PERMANENT FIX:**
1. DLT expectations on every silver/gold table — fail or quarantine.
2. Lakehouse Monitoring: row-count anomaly + drift detection.
3. MERGE keys must be unique — enforce by `EXCEPT` dedupe before merging.
4. Treat `_rescued_data` as a data-quality metric; alert when non-null.
5. Reconciliation job comparing source vs gold for the day.

---

## G. Cost & FinOps

### #24 — Unexpected DBU spike — bill anomaly — High

**SYMPTOM:**
Daily DBU consumption jumped 3× last week. Finance asks "what happened?".

**ROOT CAUSE:**
1. An all-purpose cluster left running by a user / dashboard.
2. Notebook with infinite loop or unbounded streaming query.
3. Photon enabled but workload not Photon-friendly (DBU rate higher, no speedup).
4. Job auto-scaled to max workers due to skew / shuffle.
5. SQL warehouse left at high cluster count, low utilisation.

**DIAGNOSIS:**
```sql
-- Top DBU consumers in last 7 days
SELECT
  usage_metadata.cluster_id,
  usage_metadata.job_id,
  custom_tags.cost_center,
  SUM(usage_quantity)                          AS dbus,
  SUM(usage_quantity * lp.pricing.default)     AS dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
WHERE  usage_date >= current_date() - 7
GROUP BY 1,2,3
ORDER BY dollars DESC LIMIT 25;
```

**TEMPORARY FIX:**
1. Stop offender clusters / jobs.
2. Shrink SQL warehouse to 1 cluster.
3. Disable Photon on the workload if not paying off.

**PERMANENT FIX:**
1. Mandatory cluster policies for all environments with `autotermination_minutes < 30`.
2. Budget alerts per cost-center tag.
3. Workspace setting: maximum cluster size + max concurrent clusters.
4. Streaming queries: `spark.sql.streaming.stopActiveRunOnRestart = true` + timeout guards.
5. Weekly FinOps review with top-N table to owners.

---

### #25 — All-purpose cluster running 24/7 nobody owns — Medium

**SYMPTOM:**
A cluster has been up for 14 days, zero command history for 12 days. Costing $200/day.

**ROOT CAUSE:**
1. Created without autotermination.
2. A dashboard or external app keeping a websocket alive (false activity).
3. JDBC/ODBC connection from BI tool keeping it warm.
4. Owner left the company; cluster orphaned.

**PERMANENT FIX:**
1. Cluster policy: `autotermination_minutes` mandatory 10–60.
2. Daily orphan-cluster scan (Terraform / API): clusters with no activity in 7 days → notify owner; terminate after 14.
3. For BI use SQL Warehouses, not all-purpose clusters.
4. Forbid creation of all-purpose clusters in prod workspace via policy permissions.

---

## H. Networking & Identity

### #26 — Cluster cannot reach external API — connection refused — High

**SYMPTOM:**
Notebook hangs or fails on `requests.get(...)` / JDBC connection.
```
ConnectionError: HTTPSConnectionPool(host='api.partner.com', port=443):
Max retries exceeded with url... [Errno 110] Connection timed out
```

**TRIAGE:**
1. From a notebook: `%sh nslookup api.partner.com; curl -v https://api.partner.com`
2. If DNS fails → private DNS / custom DNS issue.
3. If DNS works but TCP fails → NSG/firewall blocking egress.

**ROOT CAUSE:**
1. Egress firewall (Azure FW / AWS NFW) doesn't allow that FQDN.
2. Service Endpoint not enabled on subnet.
3. NAT GW SNAT port exhaustion (timeouts only at peak load).
4. Partner API IP-allowlist requires Databricks NAT GW public IP (changed).
5. Private DNS zone missing record for private endpoint.

**PERMANENT FIX:**
1. Use **Private Endpoint** to partner (if they support PrivateLink/PrivateConnect).
2. Maintain firewall FQDN allowlist as code (Terraform).
3. Use static NAT public IP; share with partners' allowlist.
4. Multiple NAT GWs / Azure Firewall multiple public IPs to relieve SNAT.

---

### #27 — Private Link failing — control plane unreachable — Critical

**SYMPTOM:**
Cluster bootstrap fails right at "Driver healthy" step. Or workspace login from corporate network fails.
```
Could not connect to Databricks control plane at adb-1234.azuredatabricks.net
```

**ROOT CAUSE:**
1. Private DNS zone for `privatelink.azuredatabricks.net` missing or wrong A record.
2. Private Endpoint approval still pending.
3. Subnet route table points to a firewall that drops the traffic.
4. SCC relay endpoint not provisioned (back-end PE).
5. Workspace deployed in a region where PL endpoint set is incomplete.

**DIAGNOSIS:**
```bash
# From a VM in the same subnet:
nslookup adb-1234.azuredatabricks.net
# Expected: PE IP (10.x.x.x), not public IP
tcptraceroute adb-1234.azuredatabricks.net 443
```

**PERMANENT FIX:**
1. Link the private DNS zone to all VNets where users/clusters live.
2. Approve PE on both front-end and back-end.
3. Bypass firewall for the PE IP (do not SNAT the private path).
4. Network architecture should be documented and automated via Terraform.

---

### #28 — OAuth M2M token works locally, fails in cluster — Medium

**SYMPTOM:**
```
PERMISSION_DENIED: authentication failed: token expired
or
401 Unauthorized when calling /api/2.1/jobs/run-now from job context
```

**ROOT CAUSE:**
1. Service principal not added to the workspace (account-level only).
2. Client secret expired.
3. Code reads PAT from env var that's only set on local machine.
4. Notebook uses `dbutils.notebook.context.apiToken` (legacy, ephemeral) instead of OAuth.
5. SP missing required workspace permission (e.g., CAN_USE on policy).

**PERMANENT FIX:**
1. Use Databricks-native OAuth: configure SP at account, assign to workspace, grant explicit ACLs.
2. Secrets stored in Key Vault scope; consumed via `dbutils.secrets.get`.
3. For CI: GitHub OIDC → Databricks SP federation, eliminating static secrets.
4. Rotate client secrets via Terraform + key vault expiry alerts.

---

## Appendix I — The "First 5 Checks" Mental Model

Senior admins run these five checks before drilling deeper on *any* ticket:

1. **"Did anything change?"** — DBR upgrade, library install, new job, schema change, network rule, IdP sync, IAM policy. Check audit log first 24 h before incident.
2. **"Is the workload self-similar?"** — Does the prior successful run differ in input size, distribution, or schedule? Compare via system tables.
3. **"Where is the time going?"** — Spark UI stages: read vs shuffle vs compute. Drives 90% of perf decisions.
4. **"Is it the platform or the code?"** — Run a canary cluster + canary notebook. If both fail → platform/cloud. If only this code fails → code/data.
5. **"What's the smallest reproduction?"** — Isolate variables (driver only, subset of data, single executor) before fixing.

---

## Appendix II — Spark UI: What to Look at in Each Tab

| Tab | Look for | Red flag |
|---|---|---|
| Jobs | Failed jobs, duration outliers | Retries > 0; gaps between jobs |
| Stages | Task time distribution; spill | Max/median > 5; spill > 1 GB |
| Storage | Cached RDDs/DataFrames | Many large caches; rare-hit caches |
| Environment | Effective configs | Photon flag, AQE flags, autoBroadcast |
| Executors | GC time, memory, disk | GC > 10% task time; disk > 70% |
| SQL / DataFrame | Plan tree, exchange nodes | BroadcastExchange > threshold; many small exchanges |
| Streaming | Input vs processing rate | Backlog growing; batch > trigger |
| JDBC/ODBC | SQL session details | Idle sessions hogging warehouse |

---

## Appendix III — Critical Logs & Their Locations

| What | Where |
|---|---|
| Cluster event log | UI → Compute → cluster → Event log |
| Driver log4j | `<log-delivery>/<cluster-id>/driver/log4j-active.log` |
| Driver stdout/stderr | `<log-delivery>/.../driver/{stdout,stderr}` |
| Executor logs | `<log-delivery>/.../executor/<exec-id>/{stdout,stderr,log4j}` |
| Init script logs | `<log-delivery>/.../init_scripts/<ts>_<file>.{stdout,stderr}` |
| Bootstrap output | `/databricks/init_scripts/` + `/databricks/spark/conf/` on node |
| Spark events (history server) | `<log-delivery>/.../eventlog/` |
| Audit (control plane) | `system.access.audit` |
| Usage / billing | `system.billing.usage` |
| Job runs | `system.lakeflow.job_run_timeline` |
| Query history | `system.query.history` |
| Lineage | `system.access.table_lineage`, `column_lineage` |

---

## Appendix IV — Diagnostic Spark Configs (with safe ranges)

```properties
# Adaptive Query Execution — keep ON for all SQL
spark.sql.adaptive.enabled                                   = true
spark.sql.adaptive.coalescePartitions.enabled                = true
spark.sql.adaptive.coalescePartitions.minPartitionSize       = 1MB
spark.sql.adaptive.advisoryPartitionSizeInBytes              = 64MB
spark.sql.adaptive.skewJoin.enabled                          = true
spark.sql.adaptive.skewJoin.skewedPartitionFactor            = 5
spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes  = 256MB

# Shuffle / parallelism
spark.sql.shuffle.partitions     = 4 * total_cores_at_max     # range 200..8000
spark.default.parallelism         = 2 * total_cores_at_max

# Broadcast
spark.sql.autoBroadcastJoinThreshold = 30MB..100MB

# Delta safety
spark.databricks.delta.retentionDurationCheck.enabled = true     # NEVER disable in prod
spark.databricks.delta.optimizeWrite.enabled          = true
spark.databricks.delta.autoCompact.enabled            = true

# Network / heartbeat tolerance
spark.network.timeout              = 600s
spark.executor.heartbeatInterval   = 60s

# Disk cache (formerly Delta cache)
spark.databricks.io.cache.enabled        = true
spark.databricks.io.cache.maxDiskUsage   = 50g
spark.databricks.io.cache.maxMetaDataCache = 1g

# Driver guard rails
spark.driver.maxResultSize         = 2g..4g     # fail-fast instead of hang

# Photon (DBR 11+)
spark.databricks.photon.enabled    = true
```

---

## Appendix V — System Tables: Copy-Paste Diagnostic Queries

### 1) Most expensive jobs last 30 days
```sql
SELECT u.usage_metadata.job_id, j.name,
       SUM(u.usage_quantity)                                  AS dbus,
       SUM(u.usage_quantity * lp.pricing.default)             AS dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
LEFT JOIN system.lakeflow.jobs j ON j.job_id = u.usage_metadata.job_id
WHERE  u.usage_date >= current_date() - 30
  AND  u.usage_metadata.job_id IS NOT NULL
GROUP BY 1,2 ORDER BY dollars DESC LIMIT 25;
```

### 2) Job reliability (success rate per job last 7 days)
```sql
SELECT job_id,
       COUNT(*)                                          AS runs,
       SUM(CASE WHEN result_state = 'SUCCESS' THEN 1 ELSE 0 END)/COUNT(*) AS success_rate
FROM   system.lakeflow.job_run_timeline
WHERE  period_start_time >= current_date() - 7
GROUP BY 1 HAVING success_rate < 0.95 ORDER BY success_rate;
```

### 3) Suspicious actions in audit log
```sql
SELECT event_time, user_identity.email, action_name,
       request_params, source_ip_address
FROM   system.access.audit
WHERE  event_date >= current_date() - 7
  AND  action_name IN (
        'createCluster', 'editCluster',
        'createToken', 'deleteToken',
        'createInitScript', 'updateUserPermissions',
        'updateAccountPermissions');
```

### 4) Tables with too many small files
```sql
WITH t AS (
  SELECT table_catalog, table_schema, table_name,
         num_files, total_size_bytes,
         total_size_bytes/num_files AS avg_file_size
  FROM   system.information_schema.tables
)
SELECT * FROM t
WHERE  num_files > 10000 AND avg_file_size < 32 * 1024 * 1024
ORDER BY num_files DESC LIMIT 25;
```

### 5) Orphan clusters (no activity in 7 days)
```sql
SELECT c.cluster_id, c.cluster_name, c.owned_by,
       MAX(qh.start_time) AS last_query
FROM   system.compute.clusters c
LEFT JOIN system.query.history qh ON qh.cluster_id = c.cluster_id
WHERE  c.delete_time IS NULL
GROUP BY 1,2,3
HAVING COALESCE(MAX(qh.start_time), '1900-01-01') < current_date() - 7;
```

### 6) Top contributors to shuffle (heavy queries)
```sql
SELECT statement_id, executed_by, query_source.notebook_id,
       total_duration_ms, read_bytes, shuffle_read_bytes, spilled_local_bytes
FROM   system.query.history
WHERE  start_time >= current_date() - 1
ORDER BY spilled_local_bytes DESC LIMIT 25;
```

---

*End of Runbook — Databricks Platform Admin Troubleshooting Playbook*
*Use alongside the main Interview Master Guide; cite specific scenario numbers in interview answers for instant credibility.*
