CYBERXRAY 

Overview
This document defines the API specifications for the XRay Backend API integrating the ReconX API, which is responsible for:

Orchestrating ReconX runs for asset discovery.

Managing run lifecycles, tool statuses, and discovered outputs.

Generating summaries via the ReconIQ system.

These APIs follow Richardson’s Maturity Model (Level 2) using RESTful principles with clearly defined routes, HTTP methods, status codes, and resource hierarchies.

 

Narratives
Enable analysts to trigger recon runs and view results.

Rerun failed tools selectively or as a batch.

View discovered assets with filters and metadata.

Export all recon data for offline or reporting use.

Support agent-driven summary generation.

Functional Requirements
Must allow initiation and monitoring of recon runs by asset.

Support reruns on failure (manual or automated).

Integrate with DynamoDB and S3.

Ensure pagination, sorting, and filters for discovered assets.

Generate AI-powered recon summaries.

Export entire recon run into a zip (stdout/stderr + summary).

Store all metadata and actions per tenant.

Use Cases
Actor

Use Case

Description

Release

Analyst

Start Recon Run

Trigger ReconX scan against selected asset

MVP

Analyst

View Run Status

Track per-tool results and timing

MVP

Analyst

Rerun Tools

Re-execute failed/all tools

MVP

Analyst

Export Run

Download JSON/ZIP of results

MVP+1

Analyst

Generate Summary

AI summary of recon results

MVP+1

Analyst

Comment Run

Collaboration on findings

MVP+1

Admin

Audit Runs

Retrieve metadata and tool behavior

MVP+1

Database Specifications
 DynamoDB Table: recon_run 
NEW
 

Entry is added from the CyberXray Backend API

id: UUID for this recon run (e.g., 5d12-...)

tenant_id: Owning tenant ID (e.g., 2b44-...)

target: Target asset (e.g., hypertrends.com, 192.168.0.0/24)

toolset: JSON list of tools used (e.g., ["amass", "nmap"])

retry_count: Times this run was retried (e.g., 1)

status: Run status (PENDING, RUNNING, FAILED, COMPLETED)

started_at: Start time (e.g., 2025-09-16T12:30:00Z)

completed_at: Completion time (e.g., 2025-09-16T13:00:00Z)

notes: Free-text notes by analyst (e.g., Excluded port 25)

record_metadata: JSON for internal logs/config (e.g., { source: "API", profile: "full" })

created_at: Auto timestamp (e.g., 2025-09-16T12:29:00Z)

created_by: UUID of creator (e.g., user123)

updated_at: Last updated timestamp

updated_by: UUID of updater

 

DynamoDB Table: cyberxray-tools-tracking 
IN PROGRESS

Entry is added from the ReconX API

id: UUID for this tool run (e.g., a123-...)

type: Fixed as "reconx"

run_id: Recon run ID this tool belongs to (e.g., fc23-...)

tool_name: Name of the tool (e.g., amass, nmap)

status: Execution status (SUCCESS, FAILURE, TIMEOUT, etc.)

start_time: ISO string timestamp (e.g., 2025-09-16T12:30:00Z)

end_time: ISO string timestamp (e.g., 2025-09-16T12:35:22Z)

duration_seconds: Tool execution duration (e.g., 132.8)

s3_output_path: S3 path to stored artifacts (e.g., s3://.../amass/output.json)

error: Error message if any (e.g., Timeout after 60s)

cmd: Full command run (e.g., amass enum -d 
Example Domain )

retry_count: Number of retries for this tool (e.g., 2)

 

S3 Structure: 
IN PROGRESS

Entry is added from the ReconX API

artifacts/{run_id}/reconx/{tool_name}/{run_id}

artifacts/{run_id}/agentiq/{agent_name}

API Specifications - Xray Backend API
1. Trigger Recon Run
ITEMS

DESCRIPTION

STATUS

MVP
 

HTTP METHOD

POST

ROUTE

/api/recon_runs

REQUEST BODY



{
  "tenant_id": "2b44f6c2-1234-4321-aaaa-fd88e81d3123",
  "asset_id": "9e98d390-87b9-4a42-9ec4-0571adf51e2e",
  "target": "hypertrends.com",
  "notes": "Exclude port 25, passive scan only",
  "record_metadata": {
    "key": "value pairs",
    "source": "API",
    "profile": "passive-only",
    "triggered_by": "schedule"
  },
  "user": "user-6e8217ba-c40f-4983-9e8e-d703f9ac2e07"
}
REQUIRED vs OPTIONAL FIELDS

tenant_id (UUID) – Required – Owning tenant identifier

asset_id (UUID) – Required – ID of the asset being scanned

target (string) – Required – Target domain/IP/CIDR (e.g., “The AI Revenue-Growth Company ”)

user (UUID) – Optional – ID of the user/system that triggered the run

notes (string) – Optional – Free-text notes by analyst (e.g., “Exclude port 25”)

record_metadata (JSON) – Optional – Internal metadata like source/profile/triggered_by

RESPONSE BODY



{
  "id": "5d12e3ab-9c99-4e2a-a20e-fc32a1c41234",
  "tenant_id": "2b44f6c2-1234-4321-aaaa-fd88e81d3123",
  "asset_id": "3a74b9e0-12de-4aa3-9b21-fdaae55f4444",
  "target": "hypertrends.com",
  "retry_count": 0,
  "status": "PENDING",
  "started_at": null,
  "completed_at": null,
  "notes": "Exclude port 25, passive scan only",
  "record_metadata": {
    "source": "API",
    "profile": "passive-only",
    "triggered_by": "schedule"
  },
  "audit_info": {
    "created_at": "2025-09-16T12:32:00Z",
    "created_by": "user-6e8217ba-c40f-4983-9e8e-d703f9ac2e07",
    "updated_at": "2025-09-16T12:32:00Z",
    "updated_by": "user-6e8217ba-c40f-4983-9e8e-d703f9ac2e07"
  }
}
RESPONSE CODES

202 Accepted, 400 Bad Request, 403 Forbidden

ERROR CODES

ASSET_NOT_ELIGIBLE_FOR_RECON - Asset is marked as excluded, archived, or not allowed for recon.

ASSET_NOT_FOUND - The provided asset_id does not exist in the database.

RECON_RUN_ALREADY_IN_PROGRESS - A recon run is already active for this asset; duplicates are not allowed.

INVALID_TOOLSET_CONFIGURATION - One or more tools specified are unsupported or invalid.

NOTES

Triggers a new ReconX run. Validates tenant and asset ID internally.

2. Get Recon Run Status
ITEMS

DESCRIPTION

STATUS

MVP

HTTP METHOD

GET

ROUTE

/api/recon_runs/{run_id}

RESPONSE BODY



{
  "tenant_id": "2b44f6c2-1234-4321-aaaa-fd88e81d3123",
  "asset_id": "9e98d390-87b9-4a42-9ec4-0571adf51e2e",
  "target": "hypertrends.com",
  "notes": "Exclude port 25, passive scan only",
  "record_metadata": {
    "key": "value pairs",
    "source": "API",
    "profile": "passive-only",
    "triggered_by": "schedule"
  },
  "user": "user-6e8217ba-c40f-4983-9e8e-d703f9ac2e07"
}
RESPONSE CODES

200 OK, 404 Not Found

ERROR CODES

RUN_NOT_FOUND

NOTES

Used to poll recon status in frontend dashboard.

3. Rerun Tools
ITEMS

DESCRIPTION

STATUS

MVP

HTTP METHOD

POST

ROUTE

/api/recon_runs/{run_id}/rerun

REQUEST BODY



{
  "asset_id": "9e98d390-87b9-4a42-9ec4-0571adf51e2e",
  "mode": "failed"
}
OR



{
  "asset_id": "9e98d390-87b9-4a42-9ec4-0571adf51e2e",
  "mode": "all"
}
RESPONSE BODY

{ "message": "Rerun triggered" }

RESPONSE CODES

200 OK, 400 Bad Request

ERROR CODES

RUN_NOT_FOUND - The specified run_id does not exist in the system.

ASSET_ID_REQUIRED - asset_id query parameter is missing or empty.

ASSET_NOT_FOUND - The specified asset_id is invalid or not registered under the tenant.

ASSET_NOT_ELIGIBLE_FOR_RERUN - Asset is marked as DISABLED, ARCHIVED, or excluded from reruns.

RUN_ALREADY_COMPLETED - The original run has already been completed and is not eligible for rerun.

RETRY_LIMIT_EXCEEDED - Retry attempts for one or more tools have already reached the maximum allowed (usually 3).

INVALID_MODE - The provided mode is not one of the supported values: "failed", "all", or "custom".

RUN_ALREADY_IN_PROGRESS - A rerun is already in progress for this run_id; duplicate executions are not allowed.

NOTES

Supports rerun of failed or all tools. Retry cap = 3.

4. Pause Recon Job (Soft Remove)
ITEMS

DESCRIPTION

STATUS

V2
 

HTTP METHOD

POST

ROUTE

/api/recon_runs/{run_id}/pause

REQUEST BODY



{
  "asset_id": "9e98d390-87b9-4a42-9ec4-0571adf51e2e",
  "reason": "Analyst requested pause due to incorrect configuration"
}
RESPONSE BODY



{
  "message": "Recon run has been paused",
  "run_id": "a472e395-bd3f-4d3a-8e3c-4104c9f8f9f3",
  "status": "PAUSED",
  "audit_info": {
    "updated_by": "user-6e8217ba-c40f-4983-9e8e-d703f9ac2e07",
    "updated_at": "2025-09-16T14:42:01.127Z"
  }
}
RESPONSE CODES

202 Accepted, 404 Not Found

ERROR CODES

RUN_NOT_FOUND – Provided run_id does not exist

ASSET_NOT_FOUND – Asset with given asset_id not found

RUN_NOT_ACTIVE – Only active runs can be paused

ALREADY_PAUSED – The run is already in a paused state

NOTES

MVP+1 feature. Used to stop noisy or incorrect jobs.

5. Discovered Assets Listing
ITEMS

DESCRIPTION

STATUS

MVP

PURPOSE

Fetch summary of tool executions (status, duration, logs) for a given pentest run, including optional filtering by tool name or status.

HTTP METHOD

GET

ROUTE

/api/recon_runs/{run_id}/discovered_assets

QUERY PARAM

tenant_id

asset_id (required)

skip

take

type

RESPONSE BODY



{
  "items": [
    {
      "id": "64931a74-a88b-4c19-a39c-e51535089609",
      "tenant_id": "bad7f61e-101e-4688-b7bc-423cadf90f1f",
      "asset_type": "DOMAIN",
      "value_raw": "dev1.Example.COM",
      "value_norm": "dev1.example.com",
      "status": "DISCOVERED",
      "source": "amass",
      "ip_address": "104.21.39.28",
      "service_info": {
        "port": 443,
        "protocol": "https",
        "banner": "nginx 1.18.0",
        "service_name": "nginx"
      },
      "confidence_score": 0.92,
      "discovered_at": "2025-09-16T12:34:29Z",
      "source_asset": "example.com",
      "tags": [
        "prod",
        "external"
      ],
      "asset_metadata": {
        "cloud": "aws",
        "region": "us-west-1"
      },
      "risk_score": 75,
      "audit_info": {
        "created_at": "2025-09-16T16:19:42.528Z",
        "created_by": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "updated_at": "2025-09-16T16:19:42.528Z",
        "updated_by": "3fa85f64-5717-4562-b3fc-2c963f66afa6"
      }
    }
  ],
  "currentPageNumber": 0,
  "recordsPerPage": 50,
  "totalRecords": 1,
  "totalPages": 1
}
RESPONSE CODES

200 OK

ERROR CODES

TENANT_ID_REQUIRED – Missing required tenant_id in request

RUN_NOT_FOUND – Specified run_id does not exist or is inaccessible

NO_RESULTS_FOUND – No discovered assets match the given filters

NOTES

Pagination is mandatory. Support search & filter on asset type.

6. List Tool Results
ITEMS

DESCRIPTION

STATUS

MVP

HTTP METHOD

GET

ROUTE

/api/recon_runs/{run_id}/tool_results

QUERY PARAM

asset_id (required)

tool_name (optional)

skip

take

order_by

status

run_type 

recon or vul_scan or exploit

RESPONSE BODY



{
  "items": [
    {
      "id": "f4919dcb-792c-4506-9fd4-dc09f03e779a",
      "run_id": "bad7f61e-101e-4688-b7bc-423cadf90f1f",
      "tool_name": "telnet",
      "status": "PARTIAL_SUCCESS",
      "type": "reconx",
      "cmd": "sh -lc telnet hypertrends.com 8080",
      "s3_output_path": "s3://cyberbenchmark-artifacts-dev/artifacts/bad7f61e-101e-4688-b7bc-423cadf90f1f/reconx/telnet/bad7f61e-101e-4688-b7bc-423cadf90f1f",
      "start_time": "2025-09-16T08:05:29.592682+00:00",
      "end_time": "2025-09-16T08:07:37.375486+00:00",
      "duration_seconds": 127.782804,
      "retry_count": 0,
      "error": null
    },
    {
      "id": "5506a692-8419-4977-81b0-9ab2a188a9f1",
      "run_id": "bad7f61e-101e-4688-b7bc-423cadf90f1f",
      "tool_name": "theharvester",
      "status": "SUCCESS",
      "type": "reconx",
      "cmd": "theHarvester -b all -l 200 -d hypertrends.com",
      "s3_output_path": "s3://cyberbenchmark-artifacts-dev/artifacts/bad7f61e-101e-4688-b7bc-423cadf90f1f/reconx/theharvester/bad7f61e-101e-4688-b7bc-423cadf90f1f",
      "start_time": "2025-09-16T08:05:29.557165+00:00",
      "end_time": "2025-09-16T08:07:23.581089+00:00",
      "duration_seconds": 114.023924,
      "retry_count": 0,
      "error": null
    }
  ],
  "currentPageNumber": 0,
  "recordsPerPage": 10,
  "totalRecords": 2,
  "totalPages": 1
}
RESPONSE CODES

200 OK

ERROR CODES

TENANT_ID_REQUIRED - Missing required tenant_id in request or query parameters.

RUN_NOT_FOUND - Specified run_id does not exist, is archived, or is inaccessible to the tenant.

INVALID_TOOL_NAME - The specified tool_name is not a supported or recognized recon tool.

INVALID_STATUS_FILTER - The provided status filter is invalid (expected: SUCCESS, FAILURE, TIMEOUT, etc.).

DYNAMODB_QUERY_FAILED - Failed to fetch data from DynamoDB due to query error, timeout, or capacity issue.

S3_ACCESS_FAILED - Error while accessing associated artifacts from S3 (missing/forbidden/corrupted).

NOTES

Use this in UI tool cards per recon run.

7. List Tool Result Statistics API
ITEMS

DESCRIPTION

STATUS

v2

HTTP METHOD

GET

ROUTE

/api/recon_runs/{run_id}/statistics

QUERY PARAM

asset_id (required)

RESPONSE BODY



{
  "run_id": "bad7f61e-101e-4688-b7bc-423cadf90f1f",
  "asset_id": "6c731f62-b177-412b-b228-62d9c0dfe03d",
  "status": "COMPLETED",
  "tools_used": 8,
  "tool_names": [
    "nmap",
    "telnet",
    "theharvester",
    "amass",
    "dnsrecon",
    "whois",
    "nikto",
    "subfinder"
  ],
  "tools_by_type": {
    "reconx": [
      "nmap",
      "subfinder",
      "dnsrecon",
      "theharvester",
      "amass",
      "whois"
    ],
    "vulnx": ["nikto"],
    "exploitx": []
  },
  "counts": {
    "success_count": 6,
    "failure_count": 1,
    "partial_success_count": 1,
    "timeout_count": 0,
    "completion_ratio": 0.875
  },
  "durations": {
    "total_seconds": 628.42,
    "duration_human": "10 minutes 28 seconds",
    "earliest_start_time": "2025-09-16T08:05:29.557165+00:00",
    "latest_end_time": "2025-09-16T08:15:57.981089+00:00"
  },
  "timeline": [
    {
      "tool": "nmap",
      "start_time": "2025-09-16T08:05:29.557165+00:00",
      "end_time": "2025-09-16T08:07:15.000000+00:00",
      "duration_seconds": 105.44
    },
    {
      "gap_between_tools_sec": 5.2
    },
    {
      "tool": "amass",
      "start_time": "2025-09-16T08:07:20.200000+00:00",
      "end_time": "2025-09-16T08:09:45.000000+00:00",
      "duration_seconds": 144.8
    }
  ],
  "errors": [
    {
      "tool_name": "telnet",
      "status": "PARTIAL_SUCCESS",
      "error": "Some ports unreachable"
    },
    {
      "tool_name": "amass",
      "status": "FAILURE",
      "error": "Amass scan timed out"
    }
  ],
  "performance": {
    "slow_agents": ["amass"],
    "retry_heavy_agents": ["telnet"],
    "longest_duration_agent": "amass",
    "fastest_agent": "dnsrecon"
  }
}
RESPONSE CODES

200 OK

ERROR CODES

TENANT_ID_REQUIRED - Missing required tenant_id in request or query parameters.

RUN_NOT_FOUND - Specified run_id does not exist, is archived, or is inaccessible to the tenant.

INVALID_TOOL_NAME - The specified tool_name is not a supported or recognized recon tool.

INVALID_STATUS_FILTER - The provided status filter is invalid (expected: SUCCESS, FAILURE, TIMEOUT, etc.).

DYNAMODB_QUERY_FAILED - Failed to fetch data from DynamoDB due to query error, timeout, or capacity issue.

S3_ACCESS_FAILED - Error while accessing associated artifacts from S3 (missing/forbidden/corrupted).

NOTES

Use this in UI tool cards per recon run.

8. List Tool Result Metrics API
ITEMS

DESCRIPTION

STATUS

v2

PURPOSE

Provides high-level operational and security metrics for the dashboard view, including:

Trends in asset activity, vulnerabilities, and test frequency

Security risk score tracking

Summary of the most recent completed test run

This endpoint powers dashboards for security teams, admins, and tenants to track cyber risk posture over time

HTTP METHOD

GET

ROUTE

/api/recon_runs/metrics

QUERY PARAM

asset_id (required)

RESPONSE BODY



{
  "metrics": {
    "active_assets": {
      "value": 847,
      "change_percentage": 12,
      "trend": "up"
    },
    "vulnerabilities_found": {
      "value": 23,
      "change_percentage": -8,
      "trend": "down"
    },
    "tests_completed": {
      "value": 156,
      "change_percentage": 24,
      "trend": "up"
    },
    "risk_score": {
      "value": 7.2,
      "scale": "out_of_10"
    }
  },
  "latest_test": {
    "status": "COMPLETED",
    "duration": {
      "hours": 1,
      "minutes": 15
    },
    "assets_found": 12,
    "tools_used": 4
  }
}
RESPONSE CODES

200 OK

ERROR CODES

TENANT_ID_REQUIRED - Missing required tenant_id in request or query parameters.

RUN_NOT_FOUND - Specified run_id does not exist, is archived, or is inaccessible to the tenant.

INVALID_TOOL_NAME - The specified tool_name is not a supported or recognized recon tool.

INVALID_STATUS_FILTER - The provided status filter is invalid (expected: SUCCESS, FAILURE, TIMEOUT, etc.).

DYNAMODB_QUERY_FAILED - Failed to fetch data from DynamoDB due to query error, timeout, or capacity issue.

S3_ACCESS_FAILED - Error while accessing associated artifacts from S3 (missing/forbidden/corrupted).

NOTES

Use this in UI tool cards per recon run.

9. Add Analyst Comment
ITEMS

DESCRIPTION

STATUS

MVP

PURPOSE

Allows analysts to attach structured comments or observations to a specific pentest run and asset.

Supports internal collaboration, auditability, and evidence trails inside the XRay Portal.

HTTP METHOD

POST

ROUTE

/api/recon_runs/{run_id}/comments

QUERY PARAM

asset_id (required)

REQUEST BODY



{
  "user": "uuid",
  "comment": "Found duplicate subdomain.",
  "tags": ["false-positive", "recheck"],
  "type": "observation"
}
RESPONSE BODY

 

RESPONSE CODES

201 Created

ERROR CODES

RUN_NOT_FOUND – No pentest run found for given run_id

ASSET_ID_REQUIRED – Missing required asset_id in query params

COMMENT_TOO_LARGE – Comment exceeds maximum allowed length (e.g., 2000 characters)

INVALID_TYPE – If comment type is not one of the allowed enum values

TAG_LIMIT_EXCEEDED – Too many tags provided (e.g., max 10)

NOTES

Used for collaboration inside XRay portal.

10. Export Recon Run as ZIP/JSON
ITEMS

DESCRIPTION

STATUS

v2

HTTP METHOD

POST

ROUTE

/api/recon_runs/{run_id}/export

QUERY PARAM

asset_id (required)

REQUEST BODY

{ "format": "zip" } or { "format": "json" }

RESPONSE BODY

{ "download_url": "https://s3/..." }

RESPONSE CODES

200 OK

ERROR CODES

RUN_NOT_FOUND, EXPORT_FAILED

NOTES

Initiates S3 export job and returns pre-signed URL.

11. Generate Recon Summary (ReconIQ)
ITEMS

DESCRIPTION

STATUS

MVP

HTTP METHOD

POST

ROUTE

/api/recon_runs/{run_id}/summary

QUERY PARAM

asset_id (required)

RESPONSE BODY

Markdown/HTML-based summary generated by ReconIQ agent

RESPONSE CODES

200 OK, 202 Processing, 404 Not Found

ERROR CODES

SUMMARY_NOT_READY, RUN_NOT_FOUND

NOTES

Supports AgentIQ output streaming. Used in UI post-recon.

12. Fetch Recon Summary (ReconIQ)
ITEMS

DESCRIPTION

ITEMS

DESCRIPTION

STATUS

MVP

HTTP METHOD

GET

ROUTE

/api/recon_runs/{run_id}/summary

QUERY PARAM

asset_id (required)

RESPONSE BODY

Markdown/HTML-based summary generated by ReconIQ agent

RESPONSE CODES

200 OK, 202 Processing, 404 Not Found

ERROR CODES

SUMMARY_NOT_READY, RUN_NOT_FOUND

NOTES

Supports AgentIQ output streaming. Used in UI post-recon.


Current XRAY Backend API

Current ReconX API