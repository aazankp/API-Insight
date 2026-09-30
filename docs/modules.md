# System Modules & Subsystems

This document provides a breakdown of each core module within the **ApiMonitor** platform, outlining its architectural role, underlying services, data models, and primary interactions.

---

## 1. Tenancy & Landlord Management Module

### Purpose
Provides multi-tenant boundary enforcement and organization governance. It isolates data across different customer organizations while allowing platform administrators (landlords) to manage tenant accounts from a centralized control plane.

### Core Components
- **`TenantManager` Service**: Implements tenant resolution algorithms (inspecting domains, subdomains, session cookies, and pre-login organization codes) and enforces global tenant scoping.
- **`CentralTenantController`**: Handles administrative actions for super-admins, including provisioning organizations, managing check credit balances, and toggling tenant states.
- **`BelongsToOrganization` Trait**: Injects global Eloquent query scopes ensuring all database queries are constrained by `organization_id`.

---

## 2. Projects & Domain Management Module

### Purpose
Organizes API monitoring into distinct applications, domains, and environments (Production, Staging, QA).

### Core Components
- **`Project` Model**: Represents an application boundary owned by a tenant. Holds configuration for environments, custom domains, public status page toggles, and alert rules.
- **`ProjectController`**: Manages project lifecycle, triggers manual SSL audits, and binds monitors to specific projects.
- **`AlertSetting` Model**: 1:1 relationship with projects defining alert channels (Slack, Discord, Custom Webhooks, Email) and notification triggers.

---

## 3. API Monitors & Synthetic Probing Module

### Purpose
Represents individual synthetic API monitoring targets. Allows engineers to configure HTTP request parameters, intervals, latency expectations, and validation criteria.

### Core Components
- **`ApiMonitor` Model**: Stores endpoint URL, HTTP method, check cadence, warning/critical latency thresholds, expected status codes, mutation modes, and body payloads.
- **`ApiMonitorExpectation` Model**: Defines assertions evaluated against response payloads (e.g., JSONPath checks).
- **`ApiMonitorController`**: Handles monitor creation, editing, manual "Check Now" triggering, active state toggling, and AI Detective invocation.

---

## 4. Cloud Execution & Dispatch Engine

### Purpose
Executes asynchronous outbound HTTP health checks against public API endpoints at scale while enforcing security controls.

### Core Components
- **`DispatchDueMonitorsCommand`**: Scheduled Artisan command (`monitor:dispatch-due`) executed every minute to identify monitors due for checks across all active tenants.
- **`CheckCloudApiJob`**: Queued background job that executes non-blocking HTTP requests via Guzzle/Laravel HTTP client.
- **`SsrfProtectionService`**: Security filter that resolves DNS and validates IP addresses against private networks (RFC 1918), loopbacks (`127.0.0.1`), Carrier-Grade NAT, and cloud metadata endpoints (`169.254.169.254`) before dispatching outbound requests.

---

## 5. Private On-Premise Agent Daemon & Protocol

### Purpose
Facilitates monitoring of private microservices, VPC endpoints, and intranet APIs hosted behind corporate firewalls without requiring inbound firewall openings.

### Core Components
- **`api-monitor-agent.php`**: Lightweight, standalone PHP CLI script that runs autonomously on target infrastructure.
- **`AgentApiController`**: Ingress API (`/api/v1/agent/*`) handling agent connections, token handshakes, heartbeats, job polling, and telemetry ingestion.
- **`AgentAuthenticationService`**: Manages SHA-256 token hashing, setup token expiration, and secure session management.
- **`DetectOfflineAgentsJob`**: Scheduled cron job detecting agents with expired heartbeats (> 120s) and triggering alerting workflows.

---

## 6. Contract Guard & Schema Validation Module

### Purpose
Prevents silent backend contract breakages by evaluating live JSON payloads against inferred schemas and tracking structural drift.

### Core Components
- **`ContractGuardService`**: 
  - Infers schemas from successful response payloads up to 20 keys deep.
  - Enforces field existence, strict data types (`string`, `integer`, `boolean`, `array`, `object`), and nullability.
  - Generates structural **MD5 hashes** to detect silent schema modifications even when HTTP status is 200.
  - Detects payload size anomalies ($\ge 65\%$ sudden increase or decrease).

---

## 7. Rate Limit & Quota Intelligence Module

### Purpose
Extracts, normalizes, and monitors API rate limits and token quotas across modern AI and SaaS API providers.

### Core Components
- **`RateLimitParserService`**: Normalizes response headers across **Anthropic**, **OpenAI**, **OpenRouter**, and **Standard RFC** specifications.
- **Calculated Telemetry**: Tracks requests remaining, tokens remaining, percentage of quota consumed, and calculated timestamps to limit resets.

---

## 8. Health Evaluation & SLA Scoring Engine

### Purpose
Processes raw check telemetry to determine endpoint health status, compute latency percentiles, and evaluate composite SLA scores.

### Core Components
- **`MonitorHealthEvaluator`**: Compares status codes, latencies, and Contract Guard results to categorize check outcomes into `Healthy`, `Degraded`, or `Down`.
- **`HealthScoreService`**:
  - Computes weighted Composite Health Scores (0–100) based on Availability (40%), Performance (35%), and Reliability (25%).
  - Calculates true latency percentiles: **$p_{50}$**, **$p_{95}$**, and **$p_{99}$**.
  - Detects **Latency Regressions** when current $p_{95}$ exceeds historical baselines by $\ge 25\%$.
  - Tracks **Mean Time to Recovery (MTTR)** across incident histories.

---

## 9. Incident Management & AI Detective Module

### Purpose
Manages the lifecycle of outages and performance degradations, orchestrating automated root-cause investigations powered by Google Gemini AI.

### Core Components
- **`IncidentService`**: State machine managing incident opening, failure counter increments, recovery evaluations, and downtime duration calculation.
- **`Incident` Model**: Persists incident records, start/resolved timestamps, failure counts, error traces, and AI diagnostic results.
- **`AiIncidentAnalysisService`**: Formats telemetry context into structured prompts for **Google Gemini 1.5 Flash**, outputting structured JSON diagnoses, blast-radius evaluations, and remediation checklists. Includes an offline heuristic fallback engine.

---

## 10. SSL Certificate Auditing Module

### Purpose
Monitors TLS/SSL certificate health across all configured project domains, alerting before certificate expirations cause browser security warnings.

### Core Components
- **`SslMonitorService`**: Opens secure TLS socket connections to inspect certificate authority issuers, validity periods, and expiration dates.
- **`CheckSslCertificatesCommand`**: Scheduled Artisan command (`ssl:check`) executing twice daily (00:00 and 12:00) across all projects.
- **`SslCheck` Model**: Stores certificate issuer, validity dates, days remaining, and threshold alert flags (30, 14, 7 days, and expired).

---

## 11. Multi-Channel Alerting & Webhooks Module

### Purpose
Routes operational alerts across multiple delivery channels to ensure on-call teams are notified immediately during incidents.

### Core Components
- **`AlertService`**: Central notification coordinator managing threshold rules, recipient deduplication, and multi-channel dispatch.
- **Delivery Channels**:
  - **In-App Notification Center**: Polled notifications with read/unread status.
  - **Email**: Transactional HTML notification templates.
  - **Slack**: Webhook integration with formatted Block Kit attachments.
  - **Discord**: Webhook integration with rich embed cards and status colors.
  - **Custom Webhooks**: Outbound JSON HTTP POST payloads for external automation.

---

## 12. Public Status Pages Module

### Purpose
Provides transparent, public-facing uptime dashboards accessible without authentication, keeping external stakeholders informed during outages.

### Core Components
- **`PublicStatusController`**: Serves project status pages via unique public slugs (`/status/{slug}`).
- **`PublicStatus/Show.vue`**: Responsive status dashboard displaying system status banners (Operational, Degraded, Partial Outage, Major Outage), 30-day uptime figures, and recent incident logs.

---

## 13. Executive Reliability & SLA Reporting Module

### Purpose
Generates formal reliability audits and SLA reports for engineering management, clients, and compliance teams.

### Core Components
- **`ReportController`**: Generates reports for individual monitors (`/monitors/{id}/report`) or entire project portfolios (`/projects/{id}/report`).
- **`Reports/Show.vue`**: Print-ready, PDF-exportable layout presenting health scores, percentile latencies, incident logs, and automated engineering recommendations.

---

## 14. Authentication & User Management Module

### Purpose
Handles user onboarding, credential verification, session management, and enterprise authentication standards.

### Core Components
- **Laravel Fortify**: Headless authentication backend managing logins, password resets, and email verification.
- **WebAuthn / Passkeys**: Passwordless biometric authentication support via `@laravel/passkeys`.
- **Two-Factor Authentication (TOTP)**: Built-in authenticator app integration.
- **`UserController`**: Administrative interface for managing team members within an organization.

---

## 15. Role-Based Access Control (RBAC) Module

### Purpose
Enforces granular authorization boundaries across tenant workspaces.

### Core Components
- **Spatie Laravel Permission**: Enforces roles (`Super Admin`, `Admin`, `User`) and granular permissions (`monitors.create`, `projects.ssl`, `roles.edit`).
- **Team Scoping**: All role assignments and permission evaluations are scoped by `organization_id`, ensuring permissions granted in one organization do not bleed into another.

---

## 16. Activity & Audit Logging Module

### Purpose
Maintains an immutable audit trail of all administrative and operational actions for security compliance.

### Core Components
- **Spatie Activity Log**: Automatically logs model creation, updates, and deletions.
- **`SecretSanitizer`**: Intercepts logged properties to ensure passwords, auth tokens, and API keys are redacted before persistence.
- **`ActivityLogController`**: Administrative interface for inspecting audit trails with before-and-after property diffs.
