# End-to-End System Workflows

This document details the primary operational and data workflows within the **ApiMonitor** platform, illustrated with step-by-step descriptions and Mermaid sequence and state diagrams.

---

## 1. Workflow 1: Tenant Provisioning & Workspace Setup

This workflow describes how the platform initializes a new organization workspace, assigns administrative permissions, and establishes tenant boundaries.

```mermaid
sequenceDiagram
    autonumber
    participant SuperAdmin as Super Admin (Landlord)
    participant CentralUI as Central Console (/central/tenants)
    participant TenantMgr as TenantManager
    participant DB as MySQL Database
    participant Seeder as RBAC Initializer

    SuperAdmin->>CentralUI: Submit New Organization Form (Name, Code, Admin Credentials, Credit Limit)
    CentralUI->>TenantMgr: Validate & Create Tenant Record
    TenantMgr->>DB: INSERT INTO tenants (name, code, slug, is_active, credit_limit)
    TenantMgr->>DB: INSERT INTO users (organization_id, name, email, password)
    TenantMgr->>Seeder: Initialize Tenant RBAC Roles & Permissions
    Seeder->>DB: Seed roles (Admin, User) scoped to organization_id
    Seeder->>DB: Assign 'Admin' role to new tenant user
    CentralUI-->>SuperAdmin: Display confirmation with login portal URL
```

### Steps:
1. **Landlord Submission**: Super-admin opens `/central/tenants` and submits organization parameters (organization name, unique uppercase code, initial admin email, password, and monthly check credit balance).
2. **Entity Generation**: Platform persists the tenant record, generating unique slugs and codes.
3. **Admin User Creation**: Creates the organization's primary administrator account associated with the new `organization_id`.
4. **RBAC Seeding**: Seeds tenant-scoped roles (`Admin`, `User`) and associates initial permissions using Spatie's team-scoping architecture.
5. **Credit Allocation**: Initializes the check credit budget, allowing the tenant to begin creating projects and monitors.

---

## 2. Workflow 2: Endpoint Registration & Synthetic Configuration

This workflow illustrates how an engineer registers a new API monitor, configures authentication profiles, and sets validation expectations.

```mermaid
sequenceDiagram
    autonumber
    participant Engineer as DevOps / Engineer
    participant UI as Monitor Form (/monitors/create)
    participant Controller as ApiMonitorController
    participant AuthServ as AuthProfileService
    participant DB as MySQL Database

    Engineer->>UI: Select Project & Configure Endpoint (URL, Method, Interval, Latency Thresholds)
    opt Reusable Auth Required
        Engineer->>UI: Select existing Auth Profile (Bearer / Basic / API Key)
    end
    opt Mutation Mode
        Engineer->>UI: Set Mutation Mode (Read-Only / Synthetic / Actual) & Body Payload
    end
    Engineer->>Controller: POST /monitors
    Controller->>AuthServ: Verify and associate AuthProfile ID
    Controller->>DB: INSERT INTO api_monitors
    opt Has Assertions
        Controller->>DB: INSERT INTO api_monitor_expectations (JSONPath, status rules)
    end
    Controller-->>Engineer: Redirect to Monitor Overview with initial check scheduled
```

### Steps:
1. **Endpoint Specification**: Engineer specifies the target API URL, HTTP method, check interval (1m–60m), and timeout threshold.
2. **Latency Expectations**: Configures warning latency (e.g., 500ms) and critical latency (e.g., 1000ms) boundaries.
3. **Authentication Binding**: Attaches an existing `AuthProfile` (or creates a new profile referencing environment secrets).
4. **Payload Setup**: Sets the mutation safety mode and defines request headers, query parameters, or request bodies (`raw_json`, `form_data`, `x_www_form_urlencoded`).
5. **Schedule Initialization**: Calculates `next_check_at = now()`, immediately queueing the monitor for initial execution.

---

## 3. Workflow 3: Automated Cloud Polling & Ingestion Cycle

This workflow details the automated minute-by-minute execution of public API health checks.

```mermaid
sequenceDiagram
    autonumber
    participant Cron as System Cron
    participant Command as DispatchDueMonitorsCommand
    participant Queue as Redis / DB Queue
    participant Worker as CheckCloudApiJob
    participant SSRF as SsrfProtectionService
    participant Target as Target Public API
    participant Evaluator as Health Evaluator & Ingestion
    participant DB as MySQL Database

    Cron->>Command: Trigger every minute (monitor:dispatch-due)
    Command->>DB: Query active monitors where next_check_at <= now()
    loop For each due monitor
        Command->>Queue: Dispatch CheckCloudApiJob(monitor_id)
    end

    Queue->>Worker: Process CheckCloudApiJob
    Worker->>SSRF: Validate URL against private IPs & metadata addresses
    SSRF-->>Worker: Approved (Public IP confirmed)
    Worker->>Target: Outbound HTTP Request (Monotonic timer started)
    Target-->>Worker: HTTP Response (Status, Body, Headers, Timing)
    
    Worker->>Evaluator: Parse Telemetry (Status, Latency, Rate Limit Headers)
    Evaluator->>DB: Record ApiCheck entry
    Evaluator->>DB: Update ApiMonitor (current_status, uptime_percentage, next_check_at)
```

### Steps:
1. **Cron Trigger**: The system scheduler executes `php artisan monitor:dispatch-due` every minute without overlapping.
2. **Due Monitor Query**: Evaluates active monitors across all tenants whose `next_check_at` timestamp has elapsed.
3. **Asynchronous Dispatch**: Pushes individual `CheckCloudApiJob` instances to the queue worker pool.
4. **SSRF Screening**: `SsrfProtectionService` performs DNS resolution and verifies that resolved IP addresses do not map to loopback (`127.0.0.1`), RFC 1918 subnets, or cloud metadata endpoints (`169.254.169.254`).
5. **HTTP Execution**: Issues the outbound request, tracking latency with microsecond accuracy via `hrtime`.
6. **Telemetry Parsing**: Parses HTTP status code, response time, payload size, and rate-limit headers.
7. **Persistence**: Saves an immutable `ApiCheck` record and updates monitor aggregates (uptime percentage, current latency, next check timestamp).

---

## 4. Workflow 4: Private On-Premise Agent Registration & Execution

This workflow shows how private VPC agents register, maintain heartbeats, pull intranet jobs, and return metrics.

```mermaid
sequenceDiagram
    autonumber
    participant Admin as DevOps Engineer
    participant AgentDaemon as api-monitor-agent.php
    participant Gateway as AgentApiController (/api/v1/agent/*)
    participant Auth as AgentAuthenticationService
    participant DB as MySQL Database
    participant PrivateAPI as Intranet Microservice

    Admin->>Gateway: POST /agents/setup-token (Generate 1-time setup token)
    Gateway-->>Admin: Returns setup token (valid 1 hour)
    Admin->>AgentDaemon: Execute: php api-monitor-agent.php connect <TOKEN>
    AgentDaemon->>Gateway: POST /api/v1/agent/connect (setup_token, agent_name, version)
    Gateway->>Auth: Validate SHA-256 token hash & issue live Agent Token
    Gateway-->>AgentDaemon: Return live Agent Bearer Token
    
    loop Every 30 seconds (Heartbeat)
        AgentDaemon->>Gateway: POST /api/v1/agent/heartbeat
        Gateway->>DB: UPDATE agents SET last_seen_at = now(), status = 'online'
    end

    loop Polling Loop
        AgentDaemon->>Gateway: GET /api/v1/agent/jobs
        Gateway-->>AgentDaemon: Return batch of due intranet monitors
        AgentDaemon->>PrivateAPI: Execute local HTTP check
        PrivateAPI-->>AgentDaemon: Local HTTP response
        AgentDaemon->>Gateway: POST /api/v1/agent/results (Job metrics & headers)
        Gateway->>DB: Record ApiCheck & update monitor state
    end
```

### Steps:
1. **Token Generation**: In the web UI, an administrator generates a setup token for a specific project.
2. **Agent Connect Handshake**: The standalone agent script runs `php api-monitor-agent.php connect <TOKEN>`. The gateway verifies the SHA-256 token hash, invalidates the setup token, and returns a permanent, hashed live agent bearer token.
3. **Heartbeat Maintenance**: The agent transmits periodic heartbeats every 30 seconds. A scheduled task flags agents offline if no heartbeat is received for 120 seconds.
4. **Reverse Job Polling**: The agent queries `/api/v1/agent/jobs` to retrieve due checks assigned to its project.
5. **Local Execution**: The agent executes HTTP checks inside the private network (intranet APIs, VPC databases).
6. **Result Ingestion**: Results (status, response time, headers, error traces) are submitted via `/api/v1/agent/results` with cache-based deduplication.

---

## 5. Workflow 5: Incident Lifecycle & AI Root-Cause Investigation

This state diagram and sequence illustrate how incidents open, trigger AI investigations, notify stakeholders, and resolve.

### Incident State Machine

```mermaid
stateDiagram-v2
    [*] --> Healthy: Checks Passing (Status 200, Valid Schema)
    Healthy --> Degraded: High Latency / Schema Drift
    Degraded --> Healthy: Latency Normalizes / Schema Re-verified
    Healthy --> Failing: Check Fails (5xx / Timeout / Network Error)
    Degraded --> Failing: Check Fails

    Failing --> OpenIncident: Consecutive Failures >= Threshold (Default: 1)
    OpenIncident --> OpenIncident: Subsequent Failures (Update last_error, increment failure_count)
    OpenIncident --> AI_Investigating: Trigger AI Detective (Gemini 1.5 Flash)
    AI_Investigating --> OpenIncident: Attach Root Cause & Remediation Steps

    OpenIncident --> ResolvedIncident: Consecutive Successes >= Recovery Threshold
    ResolvedIncident --> Healthy: Calculate MTTR & Dispatch Recovery Alert
```

### Detailed Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Check as Ingested Check (Failed)
    participant IncidentSvc as IncidentService
    participant AISvc as AiIncidentAnalysisService
    participant Gemini as Google Gemini 1.5 Flash API
    participant AlertSvc as AlertService
    participant Webhooks as Slack / Discord / Email
    participant DB as MySQL Database

    Check->>IncidentSvc: Ingest Failed Check (Down Status)
    IncidentSvc->>DB: Increment consecutive_failures
    alt Consecutive Failures >= Threshold AND No Open Incident
        IncidentSvc->>DB: INSERT INTO incidents (status='open', started_at=now())
        IncidentSvc->>AISvc: Request Automated Investigation
        
        alt Gemini API Key Configured
            AISvc->>Gemini: POST generateContent (Sanitized Telemetry Context)
            Gemini-->>AISvc: Return Structured JSON SRE Diagnosis
        else Key Missing or Offline
            AISvc->>AISvc: Generate Deterministic Heuristic Diagnosis
        end

        AISvc->>DB: UPDATE incidents SET ai_analysis, likely_cause, impact_summary
        IncidentSvc->>AlertSvc: notifyApiDown(monitor, incident)
        AlertSvc->>Webhooks: Dispatch formatted alert (Slack Block Kit / Discord embed)
    else Incident Already Open
        IncidentSvc->>DB: UPDATE incidents SET last_error, failure_count = failure_count + 1
    end
```

---

## 6. Workflow 6: Contract Guard Drift Detection & Alerting

This workflow shows how silent JSON schema breakages are detected and alerted without needing HTTP 5xx errors.

```mermaid
sequenceDiagram
    autonumber
    participant Worker as Cloud Worker / Agent
    participant Target as Target API
    participant Guard as ContractGuardService
    participant DB as MySQL Database
    participant Alert as AlertService

    Worker->>Target: HTTP GET /api/v1/orders
    Target-->>Worker: HTTP 200 OK with modified JSON payload
    Worker->>Guard: validate(monitor, responseJson, responseSize)
    
    Guard->>Guard: Check payload size vs last_response_size (Flag if change >= 65%)
    Guard->>Guard: Extract structural keys and compute MD5 hash
    Guard->>Guard: Evaluate field assertions (existence, types, nullability)
    
    alt Schema Hash Changed OR Required Key Missing OR Type Mismatch
        Guard-->>Worker: Contract Validation Failed (Violations List, Drift Detected)
        Worker->>DB: Record ApiCheck (health_status = 'degraded', error = 'Contract violation')
        Worker->>DB: UPDATE api_monitors SET last_schema_hash, last_response_size
        Worker->>Alert: Dispatch Contract Change Notification
    else Contract Matches
        Guard-->>Worker: Contract Passed
    end
```

---

## 7. Workflow 7: Executive SLA Report Generation & Status Page Publishing

This workflow illustrates how operational telemetry is aggregated into customer-facing dashboards and executive reports.

```mermaid
sequenceDiagram
    autonumber
    participant Viewer as Executive / Stakeholder
    participant Router as Web Routing
    participant Controller as ReportController / PublicStatusController
    participant HealthSvc as HealthScoreService
    participant DB as MySQL Database

    alt Public Status Page Request
        Viewer->>Router: GET /status/{slug}
        Router->>Controller: PublicStatusController::show(slug)
        Controller->>DB: Query Project & Monitors (without auth check)
        Controller->>Controller: Calculate overall system health (Operational / Outage)
        Controller-->>Viewer: Render PublicStatus/Show.vue (30-day timeline)
    else Executive SLA Report Request
        Viewer->>Router: GET /monitors/{id}/report?timeframe=30d
        Router->>Controller: ReportController::monitorReport(id, timeframe)
        Controller->>HealthSvc: getMonitorMetrics(monitor, '30d')
        HealthSvc->>DB: Query checks & incidents for 30d window
        HealthSvc->>HealthSvc: Calculate Composite Score, p50/p95/p99, MTTR, Regression %
        Controller-->>Viewer: Render Reports/Show.vue (Print / PDF Ready)
    end
```
