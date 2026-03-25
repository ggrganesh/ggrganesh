Databricks Dashboard — Complete SQL Queries
Dashboard Name: Platform_Usage_Compute_Dashboard

You need 4 datasets in the Databricks SQL Dashboard. Each query becomes a widget/visualization.

Dataset 1: Total DBX Job Runs (Counter Widget)
sql

-- Dataset: total_job_runs

-- Visualization: Counter

SELECT

    
COUNT
(
DISTINCT
 run_id
)
 
AS
 total_job_runs
FROM
 system
.
lakeflow
.
job_run_timeline
WHERE
 
DATE
(
period_start_time
)
 
=
 
CURRENT_DATE
(
)
Dataset 2: DBU (Counter Widget)
sql

-- Dataset: total_dbu

-- Visualization: Counter

SELECT

    
ROUND
(
SUM
(
usage_quantity
)
,
 
2
)
 
AS
 total_dbu
FROM
 system
.
billing
.
usage

WHERE
 usage_date 
=
 
CURRENT_DATE
(
)

  
AND
 usage_unit 
=
 
'DBU'
Dataset 3: Top 10 Expensive Jobs (Table Widget)
sql

-- Dataset: top_10_expensive_jobs

-- Visualization: Table

WITH
 job_costs 
AS
 
(

    
SELECT

        u
.
usage_metadata
.
job_id 
AS
 job_id
,

        
ROUND
(

            
SUM
(
u
.
usage_quantity 
*
 
COALESCE
(
lp
.
pricing
.
default
,
 
0
)
)
,

            
2

        
)
 
AS
 cost
,

        
ROUND
(
SUM
(
u
.
usage_quantity
)
,
 
2
)
 
AS
 dbu
    
FROM
 system
.
billing
.
usage
 u
    
LEFT
 
JOIN
 system
.
billing
.
list_prices lp
        
ON
 u
.
sku_name 
=
 lp
.
sku_name
        
AND
 u
.
usage_date 
>=
 lp
.
price_start_time
        
AND
 
(

            lp
.
price_end_time 
IS
 
NULL

            
OR
 u
.
usage_date 
<
 lp
.
price_end_time
        
)

    
WHERE
 u
.
usage_date 
=
 
CURRENT_DATE
(
)

      
AND
 u
.
usage_metadata
.
job_id 
IS
 
NOT
 
NULL

    
GROUP
 
BY
 u
.
usage_metadata
.
job_id
)

SELECT

    
COALESCE
(
j
.
name
,
 CAST
(
jc
.
job_id 
AS
 STRING
)
)
 
AS
 
`
Job Name
`
,

    
COALESCE
(
j
.
tags
[
'app'
]
,
 
'Unknown'
)
 
AS
 
`
APP
`
,

    jc
.
cost 
AS
 
`
COST
`
,

    jc
.
dbu 
AS
 
`
DBU
`

FROM
 job_costs jc
LEFT
 
JOIN
 system
.
lakeflow
.
jobs j
    
ON
 CAST
(
jc
.
job_id 
AS
 STRING
)
 
=
 CAST
(
j
.
job_id 
AS
 STRING
)

ORDER
 
BY
 jc
.
cost 
DESC

LIMIT
 
10
Dataset 4: Platform Metrics (Table Widget)
sql

-- Dataset: platform_metrics

-- Visualization: Table

WITH
 catalogs 
AS
 
(

    
SELECT
 
COUNT
(
*
)
 
AS
 val
    
FROM
 system
.
information_schema
.
catalogs
    
WHERE
 catalog_name 
NOT
 
IN
 
(
'system'
,
 
'__databricks_internal'
)

)
,

workspaces 
AS
 
(

    
SELECT
 
COUNT
(
DISTINCT
 workspace_id
)
 
AS
 val
    
FROM
 system
.
billing
.
usage

    
WHERE
 usage_date 
=
 
CURRENT_DATE
(
)

)
,

sql_warehouses 
AS
 
(

    
SELECT
 
COUNT
(
DISTINCT
 warehouse_id
)
 
AS
 val
    
FROM
 system
.
compute
.
warehouses
    
WHERE
 delete_time 
IS
 
NULL

)
,

clusters 
AS
 
(

    
SELECT
 
COUNT
(
DISTINCT
 cluster_id
)
 
AS
 val
    
FROM
 system
.
compute
.
clusters
    
WHERE
 delete_time 
IS
 
NULL

      
AND
 cluster_source 
IN
 
(
'UI'
,
 
'API'
)

)
,

schemas 
AS
 
(

    
SELECT
 
COUNT
(
*
)
 
AS
 val
    
FROM
 system
.
information_schema
.
schemata
    
WHERE
 catalog_name 
NOT
 
IN
 
(
'system'
,
 
'__databricks_internal'
)

      
AND
 schema_name 
!=
 
'information_schema'

)
,

job_runs 
AS
 
(

    
SELECT
 
COUNT
(
DISTINCT
 run_id
)
 
AS
 val
    
FROM
 system
.
lakeflow
.
job_run_timeline
    
WHERE
 
DATE
(
period_start_time
)
 
=
 
CURRENT_DATE
(
)

)
,

dbu 
AS
 
(

    
SELECT
 
ROUND
(
SUM
(
usage_quantity
)
,
 
2
)
 
AS
 val
    
FROM
 system
.
billing
.
usage

    
WHERE
 usage_date 
=
 
CURRENT_DATE
(
)

      
AND
 usage_unit 
=
 
'DBU'

)
,

tables_count 
AS
 
(

    
SELECT
 
COUNT
(
*
)
 
AS
 val
    
FROM
 system
.
information_schema
.
tables

    
WHERE
 table_catalog 
NOT
 
IN
 
(
'system'
,
 
'__databricks_internal'
)

      
AND
 table_schema 
!=
 
'information_schema'

)

SELECT
 
'Catalogs'
 
AS
 Metric
,
 val 
FROM
 catalogs
UNION
 
ALL
 
SELECT
 
'DBX Workspaces'
,
 val 
FROM
 workspaces
UNION
 
ALL
 
SELECT
 
'DBX - SQL Warehouse'
,
 val 
FROM
 sql_warehouses
UNION
 
ALL
 
SELECT
 
'DBX - Interactive Clusters'
,
 val 
FROM
 clusters
UNION
 
ALL
 
SELECT
 
'Schemas'
,
 val 
FROM
 schemas
UNION
 
ALL
 
SELECT
 
'Databricks Job Runs'
,
 val 
FROM
 job_runs
UNION
 
ALL
 
SELECT
 
'DBU'
,
 val 
FROM
 dbu
UNION
 
ALL
 
SELECT
 
'Tables'
,
 val 
FROM
 tables_count
How to Create the Dashboard in Databricks
Step-by-Step
Go to Databricks → Left sidebar → SQL → Dashboards
Click Create Dashboard → Name it: Platform_Usage_Compute_Dashboard
Click Add → Dataset → Paste each SQL query above
Add visualizations:
