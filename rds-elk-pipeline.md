# RDS to ELK Pipeline — Comprehensive Documentation

## Table of Contents
1. [Requirement](#1-requirement)
2. [Architecture Plan](#2-architecture-plan)
3. [AWS Infrastructure Setup](#3-aws-infrastructure-setup)
4. [Elasticsearch Index Setup](#4-elasticsearch-index-setup)
5. [Lambda Implementation](#5-lambda-implementation)
6. [IAM & Security Setup](#6-iam--security-setup)
7. [EventBridge Schedule](#7-eventbridge-schedule)
8. [Testing Guide](#8-testing-guide)
9. [Debugging Guide](#9-debugging-guide)
10. [Kibana Dashboard Guide](#10-kibana-dashboard-guide)

---

## 1. Requirement

### Business Need
Daily ingestion of workflow and project data from an AWS RDS PostgreSQL database (Aurora) into an Elasticsearch (ELK) stack for operational dashboarding and historical trend analysis.

### Goals
- Query all workflows regardless of status from the portal database daily
- Forward results to Elasticsearch with a daily snapshot date
- Enable Kibana dashboards to filter by status, agency, zone, and date
- Track historical state changes (Active → Inactive → Deleted) over time
- Zero operational overhead — fully serverless and automated

### Data Source
- **Database**: AWS Aurora PostgreSQL (`portal` database)
- **Tables**: `workflow`, `project`, `lookup`, `application`
- **Cluster ARN**: `arn:aws:rds:ap-southeast-1:058619734885:cluster:dbs-aurora-v2-cft2-prdmzdb-enginedb`

### Data Destination
- **Elasticsearch**: `https://cftstackopsprd1.es.vpce.ap-southeast-1.aws.elastic-cloud.com:443`
- **Index**: `workflows-daily-snapshot`
- **Version**: ELK 8.19.7

---

## 2. Architecture Plan

```
EventBridge (cron: 0 1 * * ? *)
    │
    ▼ triggers daily at 01:00 UTC (09:00 SGT)
Lambda Function (Python 3.12)
    │
    ├── Secrets Manager ──► portal DB secret (password)
    ├── Secrets Manager ──► ES API key secret
    │
    ├── RDS Data API ──► Aurora PostgreSQL (portal DB)
    │       └── paginated query (LIMIT 200 OFFSET n)
    │
    └── Elasticsearch Bulk API ──► workflows-daily-snapshot index
            └── VPCE (private, stays within AWS network)
```

### Key Design Decisions

| Decision | Choice | Reason |
|---|---|---|
| Compute | AWS Lambda | Serverless, zero ops, free at daily scale |
| DB connectivity | RDS Data API | No VPC complexity, no psycopg2 layer needed |
| Scheduling | EventBridge cron | Native AWS, reliable, no EC2 needed |
| Credentials | Secrets Manager | Secure, no plaintext in code or env vars |
| ES connectivity | VPCE | Private network, no internet exposure |
| Pagination | LIMIT/OFFSET 200 | Avoids RDS Data API 1MB response limit |
| Deduplication | Composite `_id` = `workflow_id_YYYY-MM-DD` | Same run can repeat safely |
| Data scope | All workflow statuses | Enables full historical and status analysis |

---

## 3. AWS Infrastructure Setup

### 3.1 Lambda Function

**Console**: Lambda → Create function

| Setting | Value |
|---|---|
| Function name | `rds-elk-workflows` |
| Runtime | Python 3.12 |
| Architecture | x86_64 |
| Memory | 128 MB |
| Timeout | 5 minutes |

### 3.2 VPC Configuration

**Console**: Lambda → Configuration → VPC

| Setting | Value |
|---|---|
| VPC | `vpc-060bcf283b3fc796f` (vpc-cft2-prod-prdez-monitoring) |
| Subnets | `subnet-0075b657584e282bc`, `subnet-03a9ac1b0d2894cfe` |
| Security Group | `sg-03949cb6942248603` (sgrp-cft2-prd-ez-asa-canary) |

### 3.3 Lambda Layer — Elasticsearch Client

Build on AWS CloudShell:

```bash
# Clean and rebuild
rm -rf elastic-layer elastic-layer.zip
mkdir -p elastic-layer/python

# Install elasticsearch v8 (must match ELK server version)
pip install "elasticsearch==8.17.2" -t elastic-layer/python/

# Package
cd elastic-layer
zip -r ../elastic-layer.zip python/
```

Download from CloudShell:
- **Actions → Download file** → `/home/cloudshell-user/elastic-layer.zip`

Upload as Lambda Layer:
- **Lambda → Layers → Create layer**
- Name: `elasticsearch-python-v8`
- Runtime: `Python 3.12`

Attach to Lambda:
- **Lambda → your function → Layers → Add a layer → Custom layers**

### 3.4 Secrets Manager

**Secret 1 — Already exists (portal DB password)**
- Name: `rds-db-credentials/cluster-CILSHCE5TCEAGJ2E4UWTGNUFTI/portaluser/1696488964620-oVVO3A`
- Used as-is, no modification needed

**Secret 2 — ES API key**
- Name: `cft2-active-workflow-ingestion-stackops`
- Type: Other type of secret
- Rotation: Disabled
- Value format:
```json
{
  "ES_API_KEY": "your-base64-encoded-api-key"
}
```

---

## 4. Elasticsearch Index Setup

### 4.1 Create the Index

Run in **Kibana Dev Console** (Menu → Dev Tools):

```
PUT /workflows-daily-snapshot
{
  "settings": {
    "number_of_shards": 1,
    "number_of_replicas": 1
  },
  "mappings": {
    "dynamic": false,
    "properties": {
      "@timestamp":                { "type": "date" },
      "snapshot_date":             { "type": "date", "format": "yyyy-MM-dd" },
      "agency_name":               { "type": "keyword" },
      "project_name":              { "type": "keyword" },
      "project_id":                { "type": "keyword" },
      "project_created_at":        { "type": "date" },
      "project_status":            { "type": "keyword" },
      "workflow_name":             { "type": "keyword" },
      "workflow_id":               { "type": "keyword" },
      "workflow_created_at":       { "type": "date" },
      "workflow_status":           { "type": "keyword" },
      "sender_application_name":   { "type": "keyword" },
      "sender_zone":               { "type": "keyword" },
      "receiver_application_name": { "type": "keyword" },
      "receiver_zone":             { "type": "keyword" }
    }
  }
}
```

**Expected response:**
```json
{
  "acknowledged": true,
  "shards_acknowledged": true,
  "index": "workflows-daily-snapshot"
}
```

### 4.2 Verify the Mapping

```
GET /workflows-daily-snapshot/_mapping
```

### 4.3 Create the Elasticsearch API Key

```
POST /_security/api_key
{
  "name": "lambda-workflows-indexer-v2",
  "role_descriptors": {
    "workflows_writer": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["workflows-active", "workflows-daily-snapshot"],
          "privileges": ["create_index", "write", "monitor"]
        }
      ]
    }
  }
}
```

> **Important**: Copy the `encoded` value from the response immediately. Store it in Secrets Manager as `ES_API_KEY`. It will not be shown again.

### 4.4 Field Type Reference

| Field | Type | Purpose |
|---|---|---|
| `@timestamp` | date | Ingest timestamp (UTC) |
| `snapshot_date` | date (yyyy-MM-dd) | SGT date of snapshot for dashboard filtering |
| `agency_name` | keyword | Filter/group by agency |
| `project_name` | keyword | Filter/group by project |
| `project_id` | keyword | Exact match, join key |
| `project_created_at` | date | Project creation timeline |
| `project_status` | keyword | Active / Inactive / Deleted |
| `workflow_name` | keyword | Filter/group by workflow |
| `workflow_id` | keyword | Unique identifier, used in composite `_id` |
| `workflow_created_at` | date | Workflow creation timeline |
| `workflow_status` | keyword | Active / Inactive / Deleted / Denied / Pending |
| `sender_application_name` | keyword | Source application filter |
| `sender_zone` | keyword | Source zone (Intranet / Internet / etc.) |
| `receiver_application_name` | keyword | Destination application filter |
| `receiver_zone` | keyword | Destination zone |

---

## 5. Lambda Implementation

### 5.1 Complete Lambda Function Code

```python
import json
import datetime
import boto3
from elasticsearch import Elasticsearch, helpers

# ---------- CONFIGURATION ----------
session = boto3.Session()
client  = session.client('rds-data')
sm      = session.client('secretsmanager')

db_cluster_arn = 'arn:aws:rds:ap-southeast-1:058619734885:cluster:dbs-aurora-v2-cft2-prdmzdb-enginedb'
secret_arn     = 'arn:aws:secretsmanager:ap-southeast-1:058619734885:secret:rds-db-credentials/cluster-CILSHCE5TCEAGJ2E4UWTGNUFTI/portaluser/1696488964620-oVVO3A'
es_secret_name = 'cft2-active-workflow-ingestion-stackops'
database_name  = 'portal'
es_index       = 'workflows-daily-snapshot'
es_url         = 'https://cftstackopsprd1.es.vpce.ap-southeast-1.aws.elastic-cloud.com:443'
BATCH_SIZE     = 200

def get_es_api_key():
    secret = sm.get_secret_value(SecretId=es_secret_name)
    return json.loads(secret['SecretString'])['ES_API_KEY']

def execute_query(sql):
    response = client.execute_statement(
        resourceArn=db_cluster_arn,
        secretArn=secret_arn,
        database=database_name,
        sql=sql,
        includeResultMetadata=True
    )
    columns = [col['name'] for col in response['columnMetadata']]
    rows = []
    for record in response['records']:
        row = {}
        for col, field in zip(columns, record):
            val = list(field.values())[0] if field else None
            if val == 'isNull':
                val = None
            row[col] = val
        rows.append(row)
    return rows

def parse_date(value):
    if not value:
        return None
    value = str(value).strip()
    value = value.replace(' ', 'T')
    if value.endswith('+00'):
        value = value + ':00'
    elif value.endswith('-00'):
        value = value + ':00'
    return value

def fetch_all_rows():
    all_rows = []
    offset = 0

    while True:
        print(f"Fetching batch at offset {offset}...")
        batch = execute_query(f"""
            SELECT
                agency.item_display_name         AS agency_name,
                p.display_name                   AS project_name,
                p.id::TEXT                       AS project_id,
                p.created_at::TEXT               AS project_created_at,
                project_status.item_display_name AS project_status,
                w.display_name                   AS workflow_name,
                w.id::TEXT                       AS workflow_id,
                w.created_at::TEXT               AS workflow_created_at,
                workflow_status.item_display_name AS workflow_status,
                sender_app.display_name          AS sender_application_name,
                sender_zone.item_display_name    AS sender_zone,
                receiver_app.display_name        AS receiver_application_name,
                receiver_zone.item_display_name  AS receiver_zone
            FROM public.workflow w
            INNER JOIN public.project p              ON w.project_id = p.id
            INNER JOIN public.lookup agency          ON p.agency_id = agency.id
            INNER JOIN public.lookup project_status  ON p.status_id = project_status.id
            INNER JOIN public.lookup workflow_status ON w.status_id = workflow_status.id
            INNER JOIN public.application sender_app ON w.sender_application_id = sender_app.id
            INNER JOIN public.application receiver_app ON w.receiver_application_id = receiver_app.id
            INNER JOIN public.lookup sender_zone     ON sender_app.zone_id = sender_zone.id
            INNER JOIN public.lookup receiver_zone   ON receiver_app.zone_id = receiver_zone.id
            ORDER BY agency.item_display_name, p.display_name, w.display_name
            LIMIT {BATCH_SIZE} OFFSET {offset}
        """)

        print(f"Batch returned {len(batch)} rows")

        if not batch:
            break

        all_rows.extend(batch)
        offset += BATCH_SIZE

        if len(batch) < BATCH_SIZE:
            break

    return all_rows

def lambda_handler(event, context):
    print("Fetching all workflows from portal DB...")

    rows = fetch_all_rows()
    print(f"Total rows fetched: {len(rows)}")

    now       = datetime.datetime.utcnow().isoformat()
    sgt_today = (datetime.datetime.utcnow() + datetime.timedelta(hours=8)).strftime('%Y-%m-%d')

    print(f"Snapshot date: {sgt_today}")

    actions = []
    for row in rows:
        row["project_created_at"]  = parse_date(row.get("project_created_at"))
        row["workflow_created_at"] = parse_date(row.get("workflow_created_at"))
        row["@timestamp"]          = now
        row["snapshot_date"]       = sgt_today

        # Composite ID: same workflow on same day = overwrite (idempotent)
        # different day = new document (historical snapshot)
        doc_id = f"{row['workflow_id']}_{sgt_today}"

        actions.append({
            "_index": es_index,
            "_id":    doc_id,
            "_source": row
        })

    es_api_key = get_es_api_key()
    es = Elasticsearch(
        es_url,
        api_key=es_api_key
    )

    success, errors = helpers.bulk(es, actions, raise_on_error=False)
    print(f"Indexed: {success}, Errors: {len(errors)}")
    if errors:
        print("Error details:", errors[:3])

    return {"indexed": success, "errors": len(errors)}
```

### 5.2 Key Implementation Notes

- **RDS Data API** is used instead of direct psycopg2 connection — no VPC port access needed, no DB driver layer required
- **`includeResultMetadata=True`** returns column names automatically — no hardcoded column positions
- **Pagination** at BATCH_SIZE=200 stays well under RDS Data API's 1MB response limit
- **`parse_date()`** converts PostgreSQL timestamp format `2024-03-08 04:23:51+00` to ES-compatible ISO 8601 `2024-03-08T04:23:51+00:00`
- **Composite `_id`** = `workflow_id_YYYY-MM-DD` makes each run idempotent — safe to re-run on the same day

---

## 6. IAM & Security Setup

### 6.1 Inline Policy — `rds-elk-workflows-policy`

**Console**: IAM → Lambda role → Add permissions → Create inline policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RDSDataAccess",
      "Effect": "Allow",
      "Action": [
        "rds-data:ExecuteStatement"
      ],
      "Resource": "arn:aws:rds:ap-southeast-1:058619734885:cluster:dbs-aurora-v2-cft2-prdmzdb-enginedb"
    },
    {
      "Sid": "PortalSecretAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:058619734885:secret:rds-db-credentials/cluster-CILSHCE5TCEAGJ2E4UWTGNUFTI/portaluser/1696488964620-oVVO3A"
    },
    {
      "Sid": "ESSecretAccess",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:ap-southeast-1:058619734885:secret:cft2-active-workflow-ingestion-stackops*"
    }
  ]
}
```

### 6.2 Inline Policy — `lambda-vpc-network-policy`

Required for Lambda to attach to VPC:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "LambdaVPCAccess",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateNetworkInterface",
        "ec2:DescribeNetworkInterfaces",
        "ec2:DeleteNetworkInterface",
        "ec2:AssignPrivateIpAddresses",
        "ec2:UnassignPrivateIpAddresses"
      ],
      "Resource": "*"
    }
  ]
}
```

### 6.3 Security Summary

| Component | Method | Notes |
|---|---|---|
| RDS password | Secrets Manager | Existing secret, reused |
| ES API key | Secrets Manager | Scoped to specific indices only |
| ES connectivity | VPCE | Private — no internet exposure |
| RDS connectivity | RDS Data API | IAM-authenticated, no open ports |
| Lambda env vars | None used for secrets | All secrets in Secrets Manager |

---

## 7. EventBridge Schedule

**Console**: EventBridge → Schedules → Create schedule

| Field | Value |
|---|---|
| Name | `cft2-active-workflow-daily-ingestion` |
| Schedule type | Cron-based |
| Cron expression | `0 1 * * ? *` |
| Timezone | UTC (= 09:00 SGT) |
| Target | Lambda function |
| Payload | `{}` |
| Retry attempts | 2 |
| Max event age | 1 hour |

---

## 8. Testing Guide

### 8.1 Manual Lambda Test

**Console**: Lambda → Test tab

```json
{}
```

**Expected successful output:**
```json
{
  "indexed": 5815,
  "errors": 0
}
```

**Expected CloudWatch log pattern:**
```
Fetching all workflows from portal DB...
Fetching batch at offset 0...
Batch returned 200 rows
Fetching batch at offset 200...
...
Total rows fetched: 5815
Snapshot date: 2026-05-31
Indexed: 5815, Errors: 0
```

### 8.2 Elasticsearch Verification Queries

**Check document count:**
```
GET /workflows-daily-snapshot/_count
```

**Preview a sample document:**
```
GET /workflows-daily-snapshot/_search
{
  "size": 1
}
```

**Check status breakdown:**
```
GET /workflows-daily-snapshot/_search
{
  "size": 0,
  "aggs": {
    "by_workflow_status": {
      "terms": { "field": "workflow_status" }
    },
    "by_project_status": {
      "terms": { "field": "project_status" }
    }
  }
}
```

**Check a specific snapshot date:**
```
GET /workflows-daily-snapshot/_search
{
  "query": {
    "term": { "snapshot_date": "2026-05-31" }
  },
  "size": 0,
  "aggs": {
    "by_status": {
      "terms": { "field": "workflow_status" }
    }
  }
}
```

**Check active workflows for a specific agency:**
```
GET /workflows-daily-snapshot/_search
{
  "query": {
    "bool": {
      "must": [
        { "term": { "workflow_status": "Active" } },
        { "term": { "agency_name": "AGC" } }
      ]
    }
  }
}
```

**Verify mapping:**
```
GET /workflows-daily-snapshot/_mapping
```

---

## 9. Debugging Guide

### 9.1 Common Errors and Fixes

#### Error: `UnsupportedResultException — result exceeds size limit 1MB`
**Cause**: RDS Data API caps response at 1MB. Query returning too many rows at once.
**Fix**: Pagination is already implemented with `LIMIT 200 OFFSET n`. If still occurring, reduce `BATCH_SIZE` to 100.

```python
BATCH_SIZE = 100  # reduce if still hitting limit
```

---

#### Error: `media_type_header_exception — Accept version must be 8 or 7, but found 9`
**Cause**: `elasticsearch` Python package version does not match server version. Package v9 sending v9 headers to v8 server.
**Fix**: Rebuild Lambda layer with exact v8 package.

```bash
pip install "elasticsearch==8.17.2" -t elastic-layer/python/
```

---

#### Error: `security_exception — action unauthorized for API key on indices [index-name]`
**Cause**: API key was scoped to a different index name.
**Fix**: Create a new API key with the correct index name in privileges.

```
POST /_security/api_key
{
  "name": "lambda-workflows-indexer-v2",
  "role_descriptors": {
    "workflows_writer": {
      "cluster": ["monitor"],
      "indices": [
        {
          "names": ["workflows-daily-snapshot"],
          "privileges": ["create_index", "write", "monitor"]
        }
      ]
    }
  }
}
```

---

#### Error: `document_parsing_exception — failed to parse date field [2024-03-08 04:23:51+00]`
**Cause**: PostgreSQL returns timestamps with space separator and short timezone `+00`. ES expects ISO 8601 with `T` separator and full offset `+00:00`.
**Fix**: `parse_date()` function handles this — ensure it is applied to both date fields.

```python
row["project_created_at"]  = parse_date(row.get("project_created_at"))
row["workflow_created_at"] = parse_date(row.get("workflow_created_at"))
```

---

#### Error: `The provided execution role does not have permissions to call CreateNetworkInterface on EC2`
**Cause**: Lambda execution role missing VPC network policy.
**Fix**: Attach `lambda-vpc-network-policy` inline policy (see Section 6.2).

---

#### Error: Lambda timeout
**Cause**: 5815 rows × 200 batch size = ~30 API calls. Should complete in ~15-20 seconds. If timing out, check VPC/VPCE connectivity.
**Fix**: Verify Lambda is in the same VPC as the VPCE endpoint. Check security group allows outbound to port 443.

---

### 9.2 CloudWatch Log Queries

**View recent Lambda executions:**

```
AWS Console → CloudWatch → Log groups → /aws/lambda/rds-elk-workflows
```

**Filter for errors only:**
```
fields @timestamp, @message
| filter @message like /Error/
| sort @timestamp desc
| limit 20
```

**Check indexed counts over time:**
```
fields @timestamp, @message
| filter @message like /Indexed/
| sort @timestamp desc
| limit 30
```

### 9.3 Re-run Safety

The pipeline is fully idempotent. Re-running on the same day:
- Same `workflow_id_YYYY-MM-DD` document ID → overwrites existing document
- No duplicates created
- Safe to trigger manually anytime via Lambda Test tab

---

## 10. Kibana Dashboard Guide

### 10.1 Create Data View

**Kibana → Stack Management → Data Views → Create data view**

| Field | Value |
|---|---|
| Name | `Workflows Daily Snapshot` |
| Index pattern | `workflows-daily-snapshot` |
| Timestamp field | `snapshot_date` |

### 10.2 Useful Dashboard Queries

**Active workflows today:**
```
workflow_status: "Active" AND snapshot_date: "2026-05-31"
```

**All workflows for a specific agency:**
```
agency_name: "AGC" AND snapshot_date: "2026-05-31"
```

**Workflows by zone pair:**
```
sender_zone: "Intranet" AND receiver_zone: "Internet"
```

**Workflows created this year:**
```
workflow_created_at >= "2026-01-01"
```

### 10.3 Recommended Visualisations

| Visualisation | Type | Fields |
|---|---|---|
| Active workflow count trend | Line chart | `snapshot_date` (x), count (y), filter `workflow_status: Active` |
| Workflows by agency | Horizontal bar | `agency_name` terms aggregation |
| Status breakdown | Pie chart | `workflow_status` terms aggregation |
| Zone traffic heatmap | Data table | `sender_zone` + `receiver_zone` |
| New workflows this month | Metric | filter `workflow_created_at` last 30 days |
| Status changes | Data table | Same `workflow_id` across two `snapshot_date` values |

---

## Appendix — Resource Reference

| Resource | Value |
|---|---|
| Lambda function | `rds-elk-workflows` |
| Lambda layer | `elasticsearch-python-v8` (elasticsearch==8.17.2) |
| Lambda runtime | Python 3.12 |
| VPC | `vpc-060bcf283b3fc796f` |
| Subnets | `subnet-0075b657584e282bc`, `subnet-03a9ac1b0d2894cfe` |
| Security group | `sg-03949cb6942248603` |
| RDS cluster ARN | `arn:aws:rds:ap-southeast-1:058619734885:cluster:dbs-aurora-v2-cft2-prdmzdb-enginedb` |
| Portal DB secret ARN | `arn:aws:secretsmanager:ap-southeast-1:058619734885:secret:rds-db-credentials/cluster-.../portaluser/...` |
| ES API key secret | `cft2-active-workflow-ingestion-stackops` |
| ES endpoint | `https://cftstackopsprd1.es.vpce.ap-southeast-1.aws.elastic-cloud.com:443` |
| ES index | `workflows-daily-snapshot` |
| EventBridge rule | `cft2-active-workflow-daily-ingestion` |
| Schedule | `cron(0 1 * * ? *)` = 09:00 SGT daily |
| Daily document volume | ~5,815 (grows over time) |
| Estimated annual index size | ~1.2 GB |
