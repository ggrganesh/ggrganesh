# Databricks — Cost · Security · Governance Playbook

> 42 real-world scenarios across the **three pillars** that define a senior Databricks Platform Admin.
> Written from the perspective of a 10+ year platform admin.

---

## Three Pillars

| Pillar | Coverage |
|---|---|
| **A. COST & FinOps** | 14 scenarios — bill spikes, idle compute, untagged resources, Photon mis-use, predictive optimization, DLT economics, GPU runaway, cross-region egress |
| **B. SECURITY** | 14 scenarios — leaked PATs, init-script tampering, public IP exposure, over-broad SPs, SSO drift, data exfiltration via Sharing, CMK rotation, compliance scope |
| **C. GOVERNANCE** | 14 scenarios — HMS-to-UC migration, dark data, PII leakage, lineage gaps, schema drift, GDPR right-to-erasure, data certification, sprawl of grants |

Every scenario follows the same transparent structure:
**SYMPTOM → TRIAGE → ROOT CAUSE → DIAGNOSIS → TEMP FIX → PERMANENT FIX → PREVENTION**

---

## Table of Contents

### A. Cost & FinOps Scenarios
1. Monthly bill jumped 40% — incident root-cause
2. SQL Warehouse over-provisioned, dashboards rarely hit it
3. Production ETL running on All-Purpose clusters (2× cost)
4. Photon enabled but workload not Photon-friendly
5. Spot reclamation forces team back to on-demand (3× cost)
6. Idle SQL warehouses overnight burning DBUs
7. Dozens of Auto Loader streams running 24/7 each with tiny load
8. No chargeback — finance cannot attribute spend
9. Untagged clusters — FinOps reports return blanks
10. Predictive Optimization disabled — storage growing 20%/month
11. DLT continuous mode used where triggered would do
12. GPU clusters left running 24/7 by ML team
13. Cross-region storage egress on every job run
14. Reserved capacity / committed-spend not used effectively

### B. Security Scenarios
15. PAT leaked into a public Git repo
16. User clones an init script from the internet (malware risk)
17. Workspace accidentally exposes public IPs
18. Service principal granted Account Admin "for convenience"
19. Compromised user account — unusual access pattern
20. PII data found in dev catalog — not redacted
21. Secret hardcoded in cluster Spark config
22. SCIM stops syncing — orphan accounts retain access
23. Stale "all users" group with broad GRANTs across UC
24. Egress firewall bypass via DNS exfiltration
25. CMK rotation breaks workspace
26. Delta Sharing recipient extracts sensitive data they shouldn't have
27. Init script with sudo escalation discovered
28. Audit log delivery silently stopped 60 days ago

### C. Governance Scenarios
29. Hive Metastore still hosting 1,400 tables — migration blocker
30. "Dark data" — tables nobody owns, nobody documents
31. PII column discovered in a Gold table consumed by BI
32. Schema drift breaks downstream Power BI report
33. Same metric calculated three different ways in three catalogs
34. MLflow model in production with no lineage to training data
35. Dropping a table that backs 47 downstream views
36. Tag chaos — env, environment, ENV all in use
37. UC GRANT sprawl — impossible to answer "who has access?"
38. GDPR Right-to-be-Forgotten request — how to delete
39. Data product certification — no Bronze/Silver/Gold contract
40. Delta Sharing audit — recipient list out of date
41. Cross-cloud / cross-region data sovereignty breach
42. Materialised View invalidated silently — stale BI

### Appendices
- I. FinOps Dashboard — 8 copy-paste SQL queries
- II. Security Detection — 8 audit-log queries (SIEM-ready)
- III. Governance Quality — 6 UC information-schema queries
- IV. Tag Taxonomy — recommended enterprise tag standard
- V. Data Classification Framework
- VI. Compliance Mapping — PCI / HIPAA / SOC2 / GDPR controls to Databricks features
- VII. The Three Pillars — mental model for every architect answer

---

## A. Cost & FinOps Scenarios

### #1 — Monthly bill jumped 40% — incident root-cause — Critical

**SYMPTOM:**
Finance flags Databricks bill $180k vs $128k last month. No new business workloads on-boarded.

**TRIAGE:**
1. Pull `system.billing.usage` grouped by SKU/workspace/date — locate the day(s) where curve jumped.
2. Within those days, group by `cluster_id` / `job_id` / `warehouse_id` — find top contributors.
3. Compare current top contributors vs same window last month.

**ROOT CAUSE (in order of frequency):**
1. New All-Purpose cluster left running by an analyst (2× rate vs jobs).
2. An ML training job auto-scaled to max workers due to skew — 6× expected DBUs.
3. Photon enabled on a Python-UDF-heavy workload — ~2× DBU rate, no speedup.
4. SQL Warehouse min-clusters bumped from 1 to 4 in a config change.
5. Streaming pipeline switched from triggered to continuous.
6. DBR upgrade caused regression that doubled runtime of nightly ETL.
7. New Auto Loader pipelines using directory-listing instead of notification — listing storms.

**DIAGNOSIS — exact queries:**
```sql
-- Day-over-day cost trend
SELECT usage_date, sku_name, SUM(usage_quantity * lp.pricing.default) AS dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
WHERE  usage_date >= current_date() - 60
GROUP BY 1,2 ORDER BY 1,2;

-- Top deltas vs prior period
WITH cur AS (
  SELECT usage_metadata.job_id, SUM(usage_quantity * lp.pricing.default) c
  FROM   system.billing.usage u
  JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
  WHERE  usage_date BETWEEN current_date() - 30 AND current_date()
  GROUP BY 1),
prv AS (
  SELECT usage_metadata.job_id, SUM(usage_quantity * lp.pricing.default) p
  FROM   system.billing.usage u
  JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
  WHERE  usage_date BETWEEN current_date() - 60 AND current_date() - 30
  GROUP BY 1)
SELECT COALESCE(cur.job_id, prv.job_id) job_id,
       prv.p prior, cur.c current, cur.c - COALESCE(prv.p, 0) delta
FROM cur FULL OUTER JOIN prv ON cur.job_id = prv.job_id
ORDER BY delta DESC LIMIT 25;
```

**TEMPORARY FIX:**
1. Stop or downscale top three offenders.
2. Set SQL Warehouse min-cluster = 1 across the board until proven needed.
3. Disable Photon on workloads where benchmarks show no gain.

**PERMANENT FIX:**
1. Cluster policies enforce `autotermination_minutes <= 30`, max workers, allowed SKUs.
2. Forbid All-Purpose clusters in prod workspace (workspace ACL).
3. Budget alerts per cost-center tag at 50/75/90%.
4. Weekly FinOps standup with top-N delta report to engineering owners.
5. "Photon decision" gate — benchmark before enabling on a workload.

**PREVENTION:**
- Anomaly-detection alert: daily DBU per workspace vs 14-day moving average.
- Predictive Optimization on storage; chargeback dashboard live for owners.

---

### #2 — SQL Warehouse over-provisioned, dashboards rarely hit it — High

**SYMPTOM:**
A Large SQL Warehouse, min/max=2/8, but average concurrency observed = 1.2 queries. Burning ~$9k/month.

**ROOT CAUSE:**
1. Originally sized for "peak" that never materialised.
2. Dashboards refresh asynchronously, never concurrent.
3. Classic warehouse used where Serverless would auto-scale to zero.
4. Min-cluster > 1 because previous admin feared cold-start.

**DIAGNOSIS:**
```sql
SELECT warehouse_id,
       PERCENTILE(active_queries, 0.95) p95_concurrent,
       PERCENTILE(active_queries, 0.50) p50_concurrent,
       MAX(active_queries) max_concurrent
FROM   system.compute.warehouses_timeline
WHERE  start_time >= current_date() - 14
GROUP BY 1 ORDER BY max_concurrent;
```

**PERMANENT FIX:**
1. Resize to Small or X-Small; `min_num_clusters = 1`.
2. Switch to **Serverless SQL** — scales to zero in seconds, premium DBU rate but no idle cost.
3. Auto-stop = 10 min for non-business hours.
4. For dashboards: enable *query result cache* so repeat hits don't reach the warehouse.

**PREVENTION:**
- Right-sizing review every quarter using the percentile query above.

---

### #3 — Production ETL on All-Purpose clusters (~2× cost) — High

**SYMPTOM:**
Audit shows multiple nightly jobs scheduled against existing interactive clusters. DBU rate ~2× what Jobs Compute would charge.

**ROOT CAUSE:**
1. Teams used existing interactive clusters out of habit / familiarity.
2. Cluster pre-warming convenience — "we don't want startup delay".
3. Lack of Job Clusters template in Bundles / CI.
4. Cluster policies don't prevent it.

**PERMANENT FIX:**
1. Standard practice: every production job uses **job clusters** defined in DABs.
2. If startup latency matters → use **Instance Pools** shared by jobs.
3. Workspace policy: in prod, only Service Principals can run jobs; jobs cannot target interactive clusters (ACL).
4. Quarterly cost-review report: % of DBUs on Jobs Compute vs All-Purpose — target > 80% Jobs Compute.

---

### #4 — Photon enabled but workload not Photon-friendly — Medium

**SYMPTOM:**
Job runs identical duration to non-Photon run, but DBU rate is ~2×. Net result: ~2× cost, zero benefit.

**ROOT CAUSE:**
1. Workload dominated by Python UDFs (Photon falls back to Spark Row).
2. Scala UDFs that aren't vectorisable.
3. Heavy use of `foreachPartition` or RDD APIs.
4. Single-row, single-task driver-side work.
5. Custom data sources (non-Delta/non-Parquet/CSV) where Photon Scan can't engage.

**DIAGNOSIS:**
Spark UI → SQL plan: count nodes that say `Photon<Operator>` vs Spark `Row<Operator>`. If Photon < 50% of plan, you're not benefiting.

**PERMANENT FIX:**
1. Rewrite Python UDFs as **Pandas UDFs (vectorised)** or pure Spark SQL functions.
2. Disable Photon for known unfriendly jobs.
3. Always benchmark Photon ON vs OFF on a representative slice before enabling in prod.
4. Build a checklist for the "Photon decision" attached to cluster policy comments.

---

### #5 — Spot reclamation forces team back to on-demand (3× cost) — Medium

**SYMPTOM:**
Team disables all spot because a critical job failed twice. Costs jump 60%.

**PERMANENT FIX:**
1. Hybrid spot configuration: `first_on_demand = ceil(num_workers / 4)` — driver and a fraction of workers always on-demand, rest spot.
2. Job-level retries (2–3) absorb single-spot loss without disabling spot entirely.
3. Use **Instance Pools** with multiple SKUs in fallback list — reduces spot reclaim rate.
4. Run criticality classification: tier-0 = no spot, tier-1 = 25% spot, tier-2 = 100% spot for workers.
5. Track effective spot reclaim rate via `system.compute.node_timeline`.

---

### #6 — Idle SQL warehouses overnight burning DBUs — Medium

**SYMPTOM:**
Classic SQL warehouse with `auto_stop_mins = 60`; warehouse sees one stray query at 3 AM and keeps running until 4 AM.

**PERMANENT FIX:**
1. Switch to **Serverless SQL** — auto-stop is seconds, not minutes.
2. For Classic: `auto_stop_mins = 5–10` outside business hours.
3. Block ad-hoc queries from automation accounts that misfire (rate-limit at JDBC layer).
4. Schedule warehouse start/stop via API for predictable BI windows.

---

### #7 — Dozens of Auto Loader streams running 24/7 each with tiny load — High

**SYMPTOM:**
30+ streaming jobs continuously running — each on its own min-1 cluster. Most receive < 1 file/hour. Combined DBU is large.

**ROOT CAUSE:**
1. "Continuous" reflex — teams default to continuous because they read about streaming.
2. Latency requirement is actually minutes/hours, not sub-second.
3. No shared streaming runtime.

**PERMANENT FIX:**
1. Use `trigger(availableNow=True)` in jobs — reads *all new* files in a single run then exits. Run on schedule (every 15 min / hour).
2. For true continuous needs use **DLT continuous** with autoscaling.
3. Consolidate multiple small streams into a single DLT pipeline with multiple tables.

Decision table for the team:

| Latency required | Pattern | Cost |
|---|---|---|
| > 15 min | availableNow on schedule | Low |
| 1–15 min | DLT triggered (every N min) | Medium |
| < 1 min | DLT continuous | High |

---

### #8 — No chargeback — finance cannot attribute spend — High

**SYMPTOM:**
CFO asks "how much did Marketing spend on Databricks last quarter?" — no one can answer.

**ROOT CAUSE:**
1. No mandatory tags on compute.
2. Tags exist but inconsistent (`cost-center` vs `cost_center` vs `cc`).
3. Workspaces shared by multiple business units.
4. No mapping of users/SPs to business unit.

**PERMANENT FIX:**
1. Enforce tag schema via **cluster policy**: `cost_center`, `business_unit`, `env`, `owner`, `project` — all `"type":"fixed"` in some policies or `"allowlist"`.
2. Tag SQL Warehouses, DLT pipelines, jobs, instance pools.
3. Build Databricks SQL dashboard joining `system.billing.usage` with `custom_tags`.
4. Publish per-BU monthly invoice automatically.
5. Cross-charge unowned/untagged spend to a "Platform" cost center as a forcing function.

---

### #9 — Untagged clusters — FinOps reports return blanks — Medium

**ROOT CAUSE:**
Tags applied to *some* clusters only; legacy clusters created before policy enforcement remain.

**PERMANENT FIX:**
1. Run a Terraform reconciliation that re-tags every existing cluster monthly.
2. Cluster policy uses `"type":"fixed"` for the tag value or `"hidden":true` with default = SP-derived value.
3. Alert on untagged compute: query `system.compute.clusters` with no `cost_center` tag.

---

### #10 — Predictive Optimization disabled — storage growing 20%/month — Medium

**ROOT CAUSE:**
Without auto-optimize and predictive optimization, deleted/updated rows accumulate; `VACUUM` not running → storage explodes.

**PERMANENT FIX:**
1. `ALTER CATALOG <cat> ENABLE PREDICTIVE OPTIMIZATION;` at catalog level.
2. On individual tables: enable `delta.autoOptimize.optimizeWrite` + `delta.autoOptimize.autoCompact`.
3. For external tables / archival data → move to cool/archive storage tier.
4. Quarterly storage report by catalog/owner from `system.information_schema`.

---

### #11 — DLT continuous mode used where triggered would do — Medium

**ROOT CAUSE:**
Continuous mode keeps clusters running 24/7. Most pipelines have hour-scale latency requirements.

**PERMANENT FIX:**
1. Triggered DLT for batch → pipeline starts, completes, stops the cluster.
2. Continuous DLT only when SLA < 5 minutes is required.
3. Enable **Enhanced Autoscaling** (DLT-native) to right-size during runs.
4. Use **Serverless DLT** for cost-efficient batches.

---

### #12 — GPU clusters left running 24/7 by ML team — High

**SYMPTOM:**
Several A100 / V100 clusters running for weeks. $300–800/day each.

**PERMANENT FIX:**
1. GPU-specific cluster policy: `autotermination_minutes <= 30`, no exceptions.
2. Use **Model Serving** for inference (Databricks-managed, scales to zero).
3. Training jobs run as **job clusters**, not interactive.
4. Dedicated ML workspace with stricter quotas; spend dashboard reviewed weekly.
5. For experimentation, single-user GPU notebooks with 15-min idle terminate.

---

### #13 — Cross-region storage egress on every job run — Medium

**ROOT CAUSE:**
Workspace in region East-US-2 reads from a storage account in West-US. Every byte hits cross-region transfer pricing.

**PERMANENT FIX:**
1. Co-locate workspace and primary storage in same region.
2. For DR/secondary, replicate data to local region; don't fetch live from other region.
3. If unavoidable, batch-copy daily into local storage rather than per-job pull.
4. For Delta Sharing reads from outside region — document cost and chargeback accordingly.

---

### #14 — Reserved capacity / committed-spend discount not used effectively — Medium

**ROOT CAUSE:**
Steady baseline spend qualifies for committed-use discount but the team buys none. Or committed too much and now overprovisioned.

**PERMANENT FIX:**
1. Compute *baseline* from 90 days of `system.billing.usage` — commit to the 80th percentile.
2. Mix Databricks committed-use with cloud-provider reservations (compute VMs).
3. Re-evaluate quarterly. Don't over-commit.
4. Negotiate with Databricks: Photon discount, serverless discount, multi-year DBU commit.

---

## B. Security Scenarios

### #15 — PAT leaked into a public Git repo — Critical

**SYMPTOM:**
A security scanner alerts a Databricks PAT was pushed to a public GitHub repo 2 hours ago.
```
dapi9f3...c1a found in commit a17b3e9 (developer/repo, line 14 of config.py)
```

**TRIAGE — first 5 minutes:**
1. Revoke the token *immediately*: `POST /api/2.0/token-management/tokens/{id}`.
2. Identify the token's owner and scope of access.
3. Query audit log for any usage of this token since exposure window.

**DIAGNOSIS:**
```sql
SELECT event_time, user_identity.email, action_name,
       source_ip_address, user_agent, request_params
FROM   system.access.audit
WHERE  event_date >= current_date() - 3
  AND  user_identity.email = 'tok-owner@corp.com'
  AND  user_agent NOT LIKE '%databricks-ui%'
ORDER BY 1 DESC;
```
Look for: source IPs outside corporate range; unusual user-agents; high-volume read of tables containing sensitive data.

**TEMPORARY FIX:**
1. Revoke the leaked token.
2. Revoke *all* PATs of the same user as precaution.
3. Rotate any secrets the user could read.
4. Force user to re-authenticate via SSO + MFA.
5. Notify security incident response team; preserve audit logs for forensics.

**PERMANENT FIX:**
1. Disable PAT creation at workspace level for all users except a tiny break-glass group.
2. Mandate **OAuth M2M** for automation; **OIDC federation** from CI (no static secrets).
3. Enable IP Access Lists so even leaked tokens don't work outside corporate network.
4. Add git-pre-commit hook + repo secret scanning (GitHub Advanced Security / Trufflehog).
5. Annual access review — tokens older than 90 days auto-revoked.

**PREVENTION:**
- SIEM rule: alert on Databricks token usage from non-corporate IPs.
- Quarterly red-team exercise — plant a test secret and measure detection time.

---

### #16 — User clones an init script from the internet (malware risk) — Critical

**SYMPTOM:**
Audit log shows new cluster with `init_scripts: [{"file":{"destination":"https://gist.github.com/anon/...","sha256":null}}]`.

**ROOT CAUSE:**
1. User pasted an init script from Stack Overflow / blog without review.
2. Init script downloads a binary at boot and executes as root.
3. Cluster has egress to internet — script can exfiltrate.

**PERMANENT FIX:**
1. Cluster policy: forbid init scripts entirely, or allow only from a specific UC volume:
```json
{
  "init_scripts.*": { "type": "forbidden" }
}
// or allow only blessed location:
{
  "init_scripts.0.volumes.destination": {
    "type": "regex",
    "pattern": "^/Volumes/platform/init/.*"
  }
}
```
2. Global init scripts banned in UC.
3. All blessed init scripts code-reviewed; stored in UC volume with read-only ACL for users, write only for platform team.
4. Workers have no public internet egress; pull packages only from internal Artifactory via PE.
5. Enable **Enhanced Security Monitoring** (file integrity + AV) on Enterprise tier.

**PREVENTION:**
- Audit-log SIEM rule: alert on any init script not under the blessed path.
- Monthly review of all init scripts in use across workspaces.

---

### #17 — Workspace accidentally exposes public IPs — Critical

**SYMPTOM:**
Security scan finds Databricks workspace listed publicly (UI accessible from internet) and worker nodes have public IPs.

**ROOT CAUSE:**
1. Workspace created without VNet Injection / SCC.
2. SCC enabled but Public Network Access not disabled.
3. No IP Access List configured.
4. Front-end Private Link not configured.

**PERMANENT FIX:**
1. Re-deploy or migrate workspace with: VNet Injection + Secure Cluster Connectivity (SCC) + Public Network Access disabled.
2. Configure Private Endpoint for both back-end (SCC relay) and front-end (UI/API).
3. IP Access List allow-list = corporate egress IPs only.
4. Egress firewall (Azure FW / AWS NFW) with FQDN allowlist.
5. Use Compliance Security Profile for HIPAA/PCI workspaces.

---

### #18 — Service principal granted Account Admin "for convenience" — High

**SYMPTOM:**
A CI/CD SP has the `Account Admin` role; it touches everything in production.

**ROOT CAUSE:**
1. "Easiest path" during initial setup.
2. No granular role taxonomy designed.
3. Lack of Just-In-Time (JIT) elevation for admin tasks.

**PERMANENT FIX:**

Apply principle of least privilege:

| Role | Privileges | Used for |
|---|---|---|
| SP-deploy-dev | Workspace user + CAN_MANAGE on specific jobs | Dev CI/CD |
| SP-deploy-prod | Same but in prod workspace | Prod CI/CD — OIDC-only |
| SP-uc-grants | UC privilege on specific catalogs | Permissions reconciliation |
| break-glass-admin | Account Admin | Emergencies only, MFA + ticket required |

1. Use **Just-In-Time elevation** via PIM (Azure) / IAM Identity Center (AWS) for human admins.
2. Quarterly access review: list SPs and their effective privileges, owners attest.

---

### #19 — Compromised user account — unusual access pattern — Critical

**SYMPTOM:**
SIEM detects user logged in from a new country, then ran `SELECT * FROM finance.pii.customer_profiles LIMIT 1000000`.

**TEMPORARY FIX (first 10 minutes):**
1. Disable user at IdP (Entra ID/Okta) → cascades via SCIM.
2. Revoke all tokens for the user.
3. Kill any active sessions: `POST /api/2.0/token-management/...`.
4. Take snapshot of query history and audit log entries for forensics.
5. Notify incident response team.

**PERMANENT FIX:**
1. Enforce MFA + Conditional Access (block from non-corporate locations).
2. Row-level / column-level masking on PII data — even a compromised account sees masked.
3. Query history anomaly detection: alert on large reads of PII tables outside normal pattern.
4. UEBA (User & Entity Behaviour Analytics) feed from Databricks audit logs.
5. Compartmentalise PII catalogs — only specific groups, no "all-employees" GRANTs.

---

### #20 — PII found in dev catalog — not redacted — High

**ROOT CAUSE:**
Production data copied to dev without masking; developers see real SSN/credit cards.

**PERMANENT FIX:**
1. Prohibit copying prod → dev. Use synthetic data generators (Mockaroo, Faker, Databricks Labs **dbldatagen**).
2. Or: copy with column masking applied at extraction time using UC masks.
3. Apply **Lakehouse Monitoring** with PII detection scans on all catalogs.
4. UC tag PII columns with `pii=true`; dashboard surfaces any PII in non-prod catalogs.
5. Sensitive-data classifier in CI: scans table schemas/values weekly.

---

### #21 — Secret hardcoded in cluster Spark config — High

**SYMPTOM:**
Cluster JSON contains `spark.snowflake.password = "Sn0wflak3!"`. Visible to any user with CAN_VIEW on cluster.

**PERMANENT FIX:**
1. Use secret reference syntax in Spark config:
```properties
spark.snowflake.password = {{secrets/prod-akv/sf-pwd}}
```
Renders the value at runtime, never stored in cluster JSON.

2. Cluster policy: regex-forbid any value in `spark_conf.*` that doesn't match `^\{\{secrets/.*\}\}$` for known sensitive keys.
3. Migrate all secret scopes to Azure Key Vault-backed / AWS Secrets Manager.
4. Periodic scan: query `system.compute.clusters` for spark_conf values that look like secrets (length > 16, no spaces).

---

### #22 — SCIM stops syncing — orphan accounts retain access — High

**SYMPTOM:**
User `jdoe` left the company 45 days ago. Still has active groups in Databricks and can log in (SSO disabled by HR but Databricks not synced).

**ROOT CAUSE:**
1. SCIM connection broken (token expired, endpoint moved).
2. SCIM running but the user's group filter excluded them.
3. Manual users created locally bypass SCIM.
4. Just deactivation in IdP doesn't always deactivate in Databricks if SCIM isn't deprovisioning.

**DIAGNOSIS:**
```sql
-- Users active in Databricks but no recent SCIM sync
SELECT u.email, u.active, last_sync
FROM   account.users u
WHERE  u.active = true
  AND  last_sync < current_date() - 7;
```

**PERMANENT FIX:**
1. Monitor SCIM health endpoint; alert on missed sync.
2. Disable manual user creation in account console (Terraform-only).
3. Quarterly access review: HR-attested active user list cross-checked with Databricks.
4. Auto-deactivate users with no login > 60 days.
5. For SP rotation, set client-secret expiry < 1 year with KV alert.

---

### #23 — Stale "all users" group with broad GRANTs across UC — High

**SYMPTOM:**
Audit reveals `account users` has `SELECT` on multiple catalogs containing PII. Anyone in the org can read.

**PERMANENT FIX:**
1. Remove `account users` from any UC GRANT immediately.
2. Replace with specific functional groups (`fin-analysts`, `risk-readers`).
3. UC policy: deny GRANTs to `account users` via Terraform validation.
4. Quarterly grant review per catalog — owner attestation.

**DIAGNOSIS:**
```sql
-- Find broad grants
SELECT object_type, object_name, privilege_type, grantee
FROM   system.information_schema.privileges
WHERE  grantee IN ('account users', 'all_account_users');
```

---

### #24 — Egress firewall bypass via DNS exfiltration — Critical

**SYMPTOM:**
Network team observes high-volume DNS queries from Databricks worker subnet to a TXT-record domain — classic DNS-tunnel pattern.

**ROOT CAUSE:**
1. Egress firewall allows port 53 (DNS) to anywhere — abuser tunnels data out.
2. Init script or notebook contains exfiltration code.

**PERMANENT FIX:**
1. Force DNS through corporate DNS resolver only; block direct DNS to internet.
2. DNS resolver applies **Response Policy Zones (RPZ)** to block known-malicious / tunnel domains.
3. Egress firewall: allowlist-only FQDNs and protocols. Default-deny.
4. Enhanced Security Monitoring detects suspicious processes on workers.
5. Forbid arbitrary internet egress — use private endpoints for partners.

---

### #25 — CMK rotation breaks workspace — Critical

**SYMPTOM:**
Security team rotates the Customer-Managed Key (CMK) in Key Vault. Suddenly workspace UI fails; clusters can't start; existing managed disks fail to mount.
```
EncryptionKeyAccessDenied: Databricks managed identity cannot wrap key 'workspace-cmk' (version XYZ)
```

**ROOT CAUSE:**
1. Old key version disabled before Databricks rotated.
2. Databricks managed identity not granted permission on new key.
3. Key Vault soft-delete pending purge.

**TEMPORARY FIX:**
1. Re-enable old key version (do not purge soft-delete).
2. Add `Get / WrapKey / UnwrapKey` permission to Databricks identity on new key.
3. Update workspace CMK reference via API to new key URI.

**PERMANENT FIX:**
1. CMK rotation runbook: keep N-1 key version enabled for 14 days.
2. Use auto-rotation in Key Vault — Databricks supports auto-detection of new versions.
3. Disable purge protection only after verified successful rotation.
4. Periodic drill: rotate CMK in dev workspace quarterly.

---

### #26 — Delta Sharing recipient extracts sensitive data they shouldn't have — High

**SYMPTOM:**
Partner organisation downloads full table dump; you notice PII columns were not masked in the share.

**ROOT CAUSE:**
1. Share added the raw table instead of a masked view.
2. Row filters / column masks not applied to share.
3. Recipient credentials shared more widely than agreed.

**PERMANENT FIX:**
1. Share *views* with applied masks, never raw tables containing PII.
2. UC row filters / column masks **do not** flow through Delta Sharing for raw tables — share through a masked view.
3. Use Delta Sharing recipient tokens with short expiry; rotate quarterly.
4. Audit access via `system.access.audit` for action `deltaSharingQueriedTable`.
5. For sensitive scenarios use **Clean Rooms** instead of Sharing.

---

### #27 — Init script with sudo escalation discovered — Critical

**SYMPTOM:**
Init script does `chmod 4755 /usr/local/bin/custom_bin` + drops SUID binary — classic privilege escalation marker.

**PERMANENT FIX:**
1. Init scripts banned except from blessed UC volume with code review.
2. Enhanced Security Monitoring detects SUID changes / unexpected binaries.
3. Compliance Security Profile (CSP) hardens DBR — many escalation paths blocked.
4. Custom Container Service image built in CI is preferred over init scripts.

---

### #28 — Audit log delivery silently stopped 60 days ago — Critical

**SYMPTOM:**
Auditor asks for audit logs covering an incident 45 days ago. SIEM has no Databricks events for the period.

**ROOT CAUSE:**
1. Storage account where logs were delivered hit quota / went read-only.
2. IAM role of the log-delivery channel lost write permission after rotation.
3. Delivery channel deleted accidentally during workspace cleanup.

**PERMANENT FIX:**
1. Use `system.access.audit` as primary source — always present at account level.
2. Monitor audit log delivery channel health (account console API).
3. SIEM heartbeat — alert if no Databricks events > 1 hour during business hours.
4. Cross-account replication of log storage.
5. Retention 1–7 years per regulatory need; lock storage immutable (Object Lock / Immutable Blob).

---

## C. Governance Scenarios

### #29 — HMS still hosting 1,400 tables — migration blocker — High

**SYMPTOM:**
Hive Metastore (HMS) workspace has 1,400+ tables with mixed ownership and lineage; UC adoption stalled.

**PERMANENT FIX (phased):**
1. Run **UCX assessment** — auto-inventory: catalogs, mounts, tables, jobs, ACLs, code references.
2. Map HMS DBs → UC schemas based on data domains.
3. Create storage credentials + external locations.
4. Migrate identity (workspace-local groups → account-level groups).
5. Tables:
   - External tables → `SYNC` to UC schema.
   - Managed HMS tables → `CREATE TABLE uc.cat.sch.t DEEP CLONE hive_metastore.db.t`.
   - Update code: 3-level namespace; drop DBFS mounts.
6. Cutover: parallel writes for safety period; deprecate HMS.
7. Set `hive_metastore` to read-only via workspace setting.

---

### #30 — "Dark data" — tables nobody owns, nobody documents — Medium

**SYMPTOM:**
Hundreds of tables in `analytics_legacy` catalog. Owners no longer in the company. No descriptions, no tags, last write 800 days ago.

**PERMANENT FIX:**
1. Tag every table with `data_steward`, `data_owner`, `certified_level` (bronze/silver/gold).
2. Mandatory `COMMENT` on table and on every column — enforce via CI.
3. Run a quarterly "garbage collection" sweep:
```sql
SELECT table_catalog, table_schema, table_name, last_altered, last_read
FROM   system.information_schema.tables
WHERE  last_read < current_date() - 365
   OR  last_read IS NULL;
```
4. Owners attest "keep / archive / delete" within 30 days, otherwise auto-archive.
5. Mandate **data product certification** for any table consumed downstream.

---

### #31 — PII column discovered in a Gold table consumed by BI — Critical

**SYMPTOM:**
Lakehouse Monitoring scan flags `email_address` column in `gold.marketing.dim_customer` consumed by 12 Power BI reports.

**TEMPORARY FIX:**
1. Apply column mask immediately:
```sql
ALTER TABLE gold.marketing.dim_customer
  ALTER COLUMN email_address SET MASK default_mask;
```
2. Restrict SELECT to `pii_readers` group.
3. Notify downstream report owners that masked value is now returned.

**PERMANENT FIX:**
1. Mandate PII tagging at ingestion: `ALTER TABLE ... SET TAGS ('pii'='true')`; UC enforces grant rules.
2. Lakehouse Monitoring continuous scan + alert on any new PII column appearing in non-PII catalogs.
3. Reusable mask functions per data type (email, phone, SSN, credit card).
4. Data contract validation in CI: schema cannot introduce PII column without explicit annotation + reviewer approval.

---

### #32 — Schema drift breaks downstream Power BI report — High

**SYMPTOM:**
Power BI dataset fails to refresh: column `customer_segment` renamed to `cust_segment` upstream.

**PERMANENT FIX:**
1. Treat downstream-consumed tables as **data products** with versioned contracts.
2. Breaking changes require a deprecation cycle:
   - v2 created alongside v1.
   - Email all consumers (UC lineage gives the list).
   - v1 marked deprecated; column comment "deprecated, see v2".
   - v1 dropped 90 days later.
3. Use lineage:
```sql
SELECT DISTINCT target_table_full_name, target_type
FROM   system.access.table_lineage
WHERE  source_table_full_name = 'silver.crm.customer'
  AND  event_time >= current_date() - 30;
```
4. For BI: expose a **view** with stable column names; views absorb upstream drift.

---

### #33 — Same metric calculated three different ways in three catalogs — High

**ROOT CAUSE:**
No single semantic layer; each BI team duplicates KPIs (revenue, active user, churn).

**PERMANENT FIX:**
1. Establish **governed Gold layer** as the only source of truth for KPIs.
2. Use **Databricks Metric View** / certified **materialised views** with formal definitions.
3. Apply `certified=true` tag on Gold tables; documentation auto-discovered.
4. Data product certification process: peer review, sample value comparison, owner sign-off.
5. Decommission duplicate calculations; redirect via views.

---

### #34 — MLflow model in production with no lineage to training data — High

**SYMPTOM:**
Model serving endpoint live for 6 months. Auditor asks: "What data was it trained on?" Nobody can answer.

**PERMANENT FIX:**
1. Register models in **Unity Catalog Model Registry** — first-class governed object with lineage.
2. Log training data references in MLflow:
```python
mlflow.log_input(mlflow.data.from_delta(table_name="silver.train.fact", version="42"))
```
3. Lineage links automatically appear in UC: dataset → experiment → model.
4. Block production deployment if lineage missing (gate in CI).
5. Tag models with `training_date`, `feature_set_version`, `owner`.

---

### #35 — Dropping a table that backs 47 downstream views — Critical

**SYMPTOM:**
Engineer drops `silver.crm.account_history`. Within an hour, dozens of dashboards and DLT pipelines fail.

**DIAGNOSIS (should have run before drop):**
```sql
SELECT DISTINCT target_table_full_name, target_type
FROM   system.access.table_lineage
WHERE  source_table_full_name = 'silver.crm.account_history'
  AND  event_time >= current_date() - 90;
```

**PERMANENT FIX:**
1. Drop policy: any table drop requires a lineage check + 30-day deprecation notice.
2. Automate via CI/CD: when a Terraform `destroy` targets a UC table, plan output includes downstream impact.
3. Use `DROP TABLE ... PURGE` only with explicit approval flag.
4. Soft-delete pattern: rename to `_deprecated_` for 30 days before drop.

---

### #36 — Tag chaos — env, environment, ENV all in use — Medium

**PERMANENT FIX:**
1. Publish enterprise tag taxonomy (see Appendix IV).
2. Enforce via cluster policy / Terraform module — reject non-conforming tags.
3. Bulk-rewrite legacy tags one-time using Terraform with idempotent updates.
4. UC tags (object-level) follow same taxonomy as compute tags — consistency across all assets.

---

### #37 — UC GRANT sprawl — impossible to answer "who has access?" — High

**SYMPTOM:**
Auditor asks for list of all users who can read `finance.gl.journal`. Manual investigation takes 3 days.

**PERMANENT FIX:**
1. GRANT only to groups; users inherit via group membership.
2. Use `system.information_schema.privileges` for instant access answer:
```sql
SELECT grantee, privilege_type, inherited_from
FROM   system.information_schema.privileges
WHERE  object_type = 'TABLE'
  AND  object_name = 'finance.gl.journal';
```
3. Periodic UC access report dashboard.
4. Terraform-only grants; manual GRANTs raise alert.
5. Owner attestation quarterly: each catalog owner certifies the grants.

---

### #38 — GDPR Right-to-be-Forgotten request — how to delete — Critical

**SYMPTOM:**
Customer requests deletion of all personal data — must be implemented within 30 days.

**PERMANENT FIX (process):**
1. **Identify**: tag every table with PII; maintain a PII inventory dashboard.
2. **Locate**: query lineage to find all places the subject's identifier propagated.
3. **Delete**: `DELETE FROM ... WHERE subject_id = ?` on all tables.
4. **VACUUM**: with retention shortened in a controlled window after final deletion to remove physical files:
```sql
SET spark.databricks.delta.retentionDurationCheck.enabled = false;  -- only for GDPR
VACUUM cat.sch.tbl RETAIN 0 HOURS;
SET spark.databricks.delta.retentionDurationCheck.enabled = true;
```
5. **Backups**: schedule retention so backups expire within compliance window OR run scrub job on backups.
6. **Audit**: store proof of deletion in immutable evidence vault.
7. Build a **"forget"** reusable job parameterised by subject id; same job runs in DR region.

---

### #39 — Data product certification — no Bronze/Silver/Gold contract — Medium

**ROOT CAUSE:**
Teams call random tables "Gold" without quality contract; consumers unable to trust data.

**PERMANENT FIX:**

Publish certification levels:

| Level | Quality contract | SLA |
|---|---|---|
| Bronze | Raw, no contract | Best effort |
| Silver | Cleansed, typed, deduped, expectations enforced | Refresh interval defined |
| Gold | Business-conformed, owner, lineage, monitored, docs | SLA + on-call |
| Platinum | Gold + DR + 99.9% availability + immutable retention | Strict SLA, audited |

1. UC tag every table with `certification_level`; dashboard distinguishes them.
2. Only Gold/Platinum tables may be shared externally (Delta Sharing).

---

### #40 — Delta Sharing audit — recipient list out of date — Medium

**SYMPTOM:**
Quarterly audit reveals 11 Delta Sharing recipients still active; only 4 are current customers.

**PERMANENT FIX:**
1. Recipient tokens with 90-day expiry; rotate quarterly.
2. Owner attestation on each recipient quarterly — otherwise auto-revoke.
3. Audit log of all `deltaSharingQueriedTable` events shipped to SIEM.
4. Use UC tags `recipient_owner`, `contract_end_date` — alert near expiry.

---

### #41 — Cross-cloud / cross-region data sovereignty breach — Critical

**SYMPTOM:**
EU customer data discovered in US-region workspace via a copy job.

**PERMANENT FIX:**
1. UC metastore per geographic boundary; cross-metastore movement forbidden.
2. External location policies restrict storage URIs to in-region buckets only.
3. Data classification tag `data_residency=EU`; deny operations that target US storage.
4. Audit log monitor for cross-region cloud egress.
5. For controlled sharing across regions, use **Delta Sharing with explicit governance**.

---

### #42 — Materialised View invalidated silently — stale BI — Medium

**ROOT CAUSE:**
Source table had schema change; MV refresh fell back to full-recompute, taking 8 hours, ran nightly → never finished within window → data 2 days stale.

**PERMANENT FIX:**
1. Refresh monitoring — alert when last successful refresh older than SLA.
2. Incremental refresh design (DLT helps).
3. Test schema changes against MV definitions in CI — block PR if invalidation forces full recompute beyond SLA.
4. Document data freshness SLA per Gold table.

---

## Appendix I — FinOps Dashboard SQL

### 1) Daily DBU cost trend (last 90 days)
```sql
SELECT usage_date,
       SUM(usage_quantity * lp.pricing.default) AS dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
WHERE  usage_date >= current_date() - 90
GROUP BY 1 ORDER BY 1;
```

### 2) Cost by cost-center tag (last 30 days)
```sql
SELECT custom_tags.cost_center,
       SUM(usage_quantity * lp.pricing.default) AS dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
WHERE  usage_date >= current_date() - 30
GROUP BY 1 ORDER BY dollars DESC;
```

### 3) Jobs Compute vs All-Purpose split
```sql
SELECT billing_origin_product,
       SUM(usage_quantity * lp.pricing.default) dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
WHERE  usage_date >= current_date() - 30
GROUP BY 1;
```

### 4) Top 25 most expensive jobs (last 30 days)
```sql
SELECT u.usage_metadata.job_id, j.name,
       SUM(u.usage_quantity * lp.pricing.default) dollars
FROM   system.billing.usage u
JOIN   system.billing.list_prices lp ON lp.sku_name = u.sku_name
LEFT JOIN system.lakeflow.jobs j ON j.job_id = u.usage_metadata.job_id
WHERE  u.usage_date >= current_date() - 30
  AND  u.usage_metadata.job_id IS NOT NULL
GROUP BY 1,2 ORDER BY dollars DESC LIMIT 25;
```

### 5) Orphan / idle interactive clusters
```sql
SELECT c.cluster_id, c.cluster_name, c.owned_by,
       MAX(qh.start_time) last_query
FROM   system.compute.clusters c
LEFT JOIN system.query.history qh ON qh.cluster_id = c.cluster_id
WHERE  c.delete_time IS NULL
  AND  c.cluster_source = 'UI'
GROUP BY 1,2,3
HAVING COALESCE(MAX(qh.start_time), '1900-01-01') < current_date() - 7;
```

### 6) Untagged clusters in last 30 days
```sql
SELECT u.usage_metadata.cluster_id, SUM(usage_quantity) dbus
FROM   system.billing.usage u
WHERE  usage_date >= current_date() - 30
  AND  (custom_tags.cost_center IS NULL
       OR custom_tags.business_unit IS NULL)
GROUP BY 1 ORDER BY dbus DESC;
```

### 7) Storage growth per catalog (last 60 days)
```sql
SELECT table_catalog,
       SUM(total_size_bytes)/1e12 AS tb
FROM   system.information_schema.tables
GROUP BY 1 ORDER BY tb DESC;
```

### 8) Photon adoption %
```sql
SELECT
  SUM(CASE WHEN sku_name LIKE '%PHOTON%' THEN usage_quantity * lp.pricing.default END) /
  SUM(usage_quantity * lp.pricing.default) photon_pct
FROM system.billing.usage u
JOIN system.billing.list_prices lp ON lp.sku_name = u.sku_name
WHERE usage_date >= current_date() - 30;
```

---

## Appendix II — Security Audit Queries (SIEM-ready)

### 1) Token creation events
```sql
SELECT event_time, user_identity.email, action_name,
       request_params, source_ip_address, user_agent
FROM   system.access.audit
WHERE  action_name IN ('createToken', 'generateDbToken',
                       'createOauthSecret');
```

### 2) Failed login spikes
```sql
SELECT user_identity.email,
       COUNT(*) failures
FROM   system.access.audit
WHERE  event_date >= current_date() - 1
  AND  action_name = 'login'
  AND  response.status_code >= 400
GROUP BY 1 HAVING failures > 10;
```

### 3) Cluster created with init script
```sql
SELECT event_time, user_identity.email, request_params:cluster_name,
       request_params:init_scripts
FROM   system.access.audit
WHERE  action_name IN ('createCluster', 'editCluster')
  AND  request_params:init_scripts IS NOT NULL;
```

### 4) UC grants to "everyone"
```sql
SELECT event_time, user_identity.email, request_params
FROM   system.access.audit
WHERE  action_name LIKE '%Grant%'
  AND  request_params:principal IN ('account users', 'all_account_users');
```

### 5) Access from outside corporate IP ranges
```sql
SELECT event_time, user_identity.email, source_ip_address, action_name
FROM   system.access.audit
WHERE  event_date >= current_date() - 7
  AND  source_ip_address NOT REGEXP '^(10\\.|172\\.|192\\.168\\.)'
  AND  user_identity.email NOT LIKE '%@partners.corp.com';
```

### 6) Permission escalation events
```sql
SELECT event_time, user_identity.email, action_name, request_params
FROM   system.access.audit
WHERE  action_name IN ('updateAccountPermissions',
                       'updateGroup',
                       'createGroup',
                       'addUserToGroup');
```

### 7) Delta Sharing access
```sql
SELECT event_time, request_params:recipient_name,
       request_params:share_name, request_params:table_name,
       source_ip_address
FROM   system.access.audit
WHERE  action_name = 'deltaSharingQueriedTable';
```

### 8) Service principal usage anomalies
```sql
SELECT service_principal_id,
       COUNT(*) calls,
       COUNT(DISTINCT source_ip_address) distinct_ips
FROM   system.access.audit
WHERE  event_date >= current_date() - 1
  AND  service_principal_id IS NOT NULL
GROUP BY 1
HAVING distinct_ips > 3 OR calls > 50000;
```

---

## Appendix III — Governance Quality Queries

### 1) Tables without owner or comment
```sql
SELECT table_catalog, table_schema, table_name, table_owner,
       comment
FROM   system.information_schema.tables
WHERE  comment IS NULL OR table_owner IS NULL;
```

### 2) Columns with no comment in Gold catalogs
```sql
SELECT c.table_catalog, c.table_schema, c.table_name, c.column_name
FROM   system.information_schema.columns c
WHERE  c.table_catalog LIKE 'gold%'
  AND  c.comment IS NULL;
```

### 3) Untagged tables
```sql
SELECT t.table_catalog, t.table_schema, t.table_name
FROM   system.information_schema.tables t
LEFT JOIN system.information_schema.table_tags tt
       ON tt.catalog_name = t.table_catalog
      AND tt.schema_name  = t.table_schema
      AND tt.table_name   = t.table_name
WHERE  tt.tag_name IS NULL;
```

### 4) Lineage gaps (no downstream consumers)
```sql
SELECT t.table_catalog, t.table_schema, t.table_name
FROM   system.information_schema.tables t
LEFT JOIN system.access.table_lineage l
       ON l.source_table_full_name =
          CONCAT_WS('.', t.table_catalog, t.table_schema, t.table_name)
WHERE  l.target_table_full_name IS NULL
  AND  t.table_catalog IN ('silver_prod', 'gold_prod');
```

### 5) PII tagged columns — current state
```sql
SELECT catalog_name, schema_name, table_name, column_name
FROM   system.information_schema.column_tags
WHERE  tag_name = 'pii' AND tag_value = 'true';
```

### 6) Effective grants per principal
```sql
SELECT grantee, object_type, object_name, privilege_type, inherited_from
FROM   system.information_schema.privileges
WHERE  grantee = 'fin-analysts'
ORDER BY 2,3,4;
```

---

## Appendix IV — Recommended Enterprise Tag Taxonomy

| Tag key | Required? | Allowed values | Notes |
|---|---|---|---|
| `environment` | Yes | dev / qa / prod | Lowercase, fixed |
| `cost_center` | Yes | 4-digit code | Allowlist |
| `business_unit` | Yes | finance / risk / marketing / ... | Allowlist |
| `owner` | Yes | email | Active employee |
| `project` | Yes | free text | Lowercase, no spaces |
| `data_classification` | Yes (UC objects) | public / internal / confidential / restricted | For governance |
| `data_residency` | Yes (UC objects) | us / eu / apac / global | For sovereignty |
| `pii` | Conditional | true / false | Drives masking |
| `certification_level` | UC tables | bronze / silver / gold / platinum | SLA contract |
| `data_steward` | Yes (UC objects) | email | Governance contact |
| `retention_days` | UC tables | integer | For lifecycle automation |

---

## Appendix V — Data Classification Framework

| Class | Examples | Controls required |
|---|---|---|
| **Public** | Marketing site copy, published reports | None beyond UC defaults |
| **Internal** | Most operational data | Workspace SSO + UC grants |
| **Confidential** | Customer non-PII, financials | MFA, restricted groups, audit alerts |
| **Restricted** | PII, PHI, payment card | Column masking, row filters, CMK, IP allow-list, Compliance Security Profile, no Delta Sharing without legal sign-off |

---

## Appendix VI — Compliance Mapping

| Control | PCI-DSS | HIPAA | SOC 2 | GDPR | Databricks mechanism |
|---|---|---|---|---|---|
| Encryption at rest | 3.5 | 164.312(a) | CC6.1 | Art 32 | CMK on managed services + workspace |
| Encryption in transit | 4.1 | 164.312(e) | CC6.7 | Art 32 | TLS 1.2+ everywhere, mTLS plane-to-plane |
| Access control | 7 | 164.308(a)(4) | CC6.3 | Art 32 | UC + IAM + SSO + MFA |
| Audit logging | 10 | 164.312(b) | CC7.2 | Art 30 | system.access.audit + delivery to immutable store |
| Network segmentation | 1.2 | n/a | CC6.6 | Art 32 | VNet injection, Private Link, SCC |
| Vulnerability mgmt | 11 | 164.308(a)(1) | CC7.1 | Art 32 | DBR auto-patching, CSP, Enhanced Sec Monitoring |
| Data minimisation | 3.4 | 164.312(c) | CC6.1 | Art 5 | Column masks, row filters, tokenisation |
| Right to erasure | n/a | n/a | n/a | Art 17 | DELETE + VACUUM + backup scrub job |
| Data residency | contextual | contextual | contextual | Art 44–49 | Per-region UC metastore + external location policies |

---

## Appendix VII — The Three Pillars Mental Model

When asked any architectural question, walk the interviewer through these three pillars to demonstrate seniority. Every design decision can be justified from at least one of these vectors.

| Pillar | Key questions to ask | Tools / patterns to reach for |
|---|---|---|
| **Cost** | What's the steady-state spend? What's the spike? Who owns the cost? Is it Photon-friendly? Are we right-sized? | Cluster policies, tags, job clusters, Serverless, Predictive Optimization, system.billing.* |
| **Security** | Where does the data live? How is it encrypted? Who can access? Where does egress go? How do we know if compromised? | UC, SSO + MFA, SCIM, Private Link, SCC, CMK, IP allow-list, audit log to SIEM, CSP |
| **Governance** | Who owns this data? What's its lineage? What's its quality contract? Can we delete it on request? | UC tags, lineage, expectations, Lakehouse Monitoring, certified data products, GDPR runbooks |

**Closing principle:** A senior platform admin makes *cost, security, and governance* first-class concerns from day one — not retrofits. Every cluster policy, every UC GRANT, every job spec should answer all three.

---

*End of Playbook — Databricks Cost, Security & Governance Incident Playbook*
*Use alongside the Interview Master Guide and Troubleshooting Runbook.*
