# Comprehensive Feature Catalog

This document provides a detailed technical catalog of the core features implemented in **ApiMonitor**.

---

## 1. Dual Monitoring Engines (Cloud & Private On-Premise)

### High-Throughput Cloud Engine
- **Asynchronous Worker Execution**: Outbound synthetic checks are queued and executed via background queue workers (`CheckCloudApiJob`), preventing request bottlenecks.
- **Configurable HTTP Methods**: Full support for `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, and `HEAD` verbs.
- **Flexible Request Intervals**: Cadences configurable per monitor: 1 min, 5 min, 10 min, 15 min, 30 min, and 60 min.
- **Granular Timeout Controls**: Configurable HTTP timeouts (1–60 seconds) with millisecond-accurate timing using high-resolution monotonic clocks (`hrtime`).
- **Mutation & Safety Modes**:
  - `read_only_health_check`: Safe idempotent inspection (default).
  - `synthetic_test`: Non-destructive synthetic test scenarios.
  - `actual_request`: Full end-to-end payload execution.
- **Payload Types**: Supports `raw_json`, multipart `form_data`, and `x_www_form_urlencoded` with custom body specifications.
- **SSRF Defense**: Built-in validation blocks requests targeting loopback addresses, internal subnets (RFC 1918), and cloud metadata endpoints (`169.254.169.254`).

### On-Premise Private Agent Daemon
- **Zero-Dependency Standalone Script**: `api-monitor-agent.php` requires only standard PHP CLI (no Composer or framework dependencies needed on target hosts).
- **Reverse Polling Architecture**: Runs behind corporate firewalls and VPCs without requiring open inbound ports or public IP addresses.
- **Secure Token Handshake**: One-time setup tokens are hashed (`SHA-256`) and exchanged for permanent agent authentication tokens.
- **Autonomous Job Execution**: Agents pull batches of due monitoring jobs from `/api/v1/agent/jobs`, execute checks locally within private intranets, and report telemetry back via `/api/v1/agent/results`.
- **Heartbeat & Liveness Tracking**: Continuous 30-second heartbeats. A scheduled background job (`DetectOfflineAgentsJob`) flags agents offline if no heartbeat is received within 120 seconds.

---

## 2. AI Detective: SRE Incident Root-Cause Analysis

### Google Gemini 1.5 Flash Integration
When an endpoint experiences an outage or performance degradation, ApiMonitor invokes **Google Gemini 1.5 Flash** (`generativelanguage.googleapis.com`) to generate real-time Site Reliability Engineering (SRE) root-cause diagnostics.

- **Telemetry Context Assembly**: The engine feeds the model with monitor metadata, recent HTTP check logs (status codes, latency trends, error traces), duration of failure, consecutive failure counts, and payload diffs.
- **Structured JSON Output Schema**: Gemini is constrained to return strictly valid JSON without markdown wrapping:
  - `problem`: Precise description of what is failing (e.g., `GET /api/v1/users returned 502 Bad Gateway`).
  - `when_started`: Human-readable start timestamp and relative duration.
  - `previous_behavior`: Historical operational baseline of the endpoint.
  - `what_changed`: Identification of abrupt shifts in status code, latency, or response size.
  - `possible_causes`: Ranked hypotheses with confidence percentages (e.g., *Database connection pool exhausted: 85%*).
  - `impact`: Concrete business and engineering impact on client apps.
  - `suggested_checks`: Checklist of actionable debugging steps for on-call engineers.
  - `recommended_action`: Primary immediate action to restore availability.

### Smart Deterministic Heuristic Fallback
If an external API key is not configured, the network is offline, or the external AI service encounters rate limiting, the platform falls back to an internal heuristic engine (`generateHeuristicAnalysis`). The heuristic engine inspects status codes, network errors, DNS patterns, and latency metrics to deliver deterministic, structured SRE diagnoses offline.

---

## 3. Contract Guard & Schema Drift Detection

### Automated JSON Schema Generation
- With a single click, the engine captures a successful live response payload and automatically infers the JSON schema structure, field types, and nesting hierarchies up to 20 keys deep.

### Strict Schema Assertions
- Validates that mandatory response keys exist.
- Asserts field types: `string`, `integer`, `float`, `boolean`, `array`, `object`, or `null`.
- Enforces strict nullability constraints (flags unexpected `null` values on required fields).

### Structural Schema Drift Hashing
- Computes an **MD5 structural hash** over the response's key hierarchy and data types.
- If a backend change alters the schema structure—even if the status code remains `200 OK`—Contract Guard flags the check as degraded, records the schema drift, and alerts engineers to prevent silent client crashes.

### Payload Size Anomaly Detection
- Tracks the byte size of successful responses.
- Detects abrupt swings in payload size ($\ge 65\%$, e.g., a response dropping from 5.4 KB to 200 bytes).
- Flags potential silent failures such as empty lists returned due to database errors or unauthorized empty payloads.

---

## 4. Rate Limit & Token Quota Intelligence

### Multi-Provider Header Extraction
ApiMonitor normalizes and parses disparate rate-limiting headers across major API ecosystems:
- **Anthropic AI**: `anthropic-ratelimit-requests-limit`, `anthropic-ratelimit-requests-remaining`, `anthropic-ratelimit-tokens-limit`, `anthropic-ratelimit-tokens-remaining`, and reset timers.
- **OpenAI & OpenRouter**: `x-ratelimit-limit-requests`, `x-ratelimit-remaining-requests`, `x-ratelimit-limit-tokens`, `x-ratelimit-remaining-tokens`, and relative reset counters.
- **Standard RFC / GitHub / Stripe / Cloudflare**: `ratelimit-limit`, `ratelimit-remaining`, `ratelimit-reset`, and `retry-after`.

### Quota Capacity Meters
- Calculates exact real-time percentages of consumed vs. remaining requests and tokens.
- Visualizes capacity meters directly in the monitor view and system dashboard.
- Displays calculated countdowns to quota resets.

---

## 5. Automated SSL Certificate Lifecycle Tracking

- **Automated Discovery**: Runs twice daily (`ssl:check` at 00:00 and 12:00) across all managed project domains.
- **Certificate Inspection**: Opens secure TLS sockets to extract Certificate Authority (CA) issuer details, valid-from dates, and expiration timestamps.
- **Threshold-Based Escalation**: Automatically dispatches alerts when certificates enter warning windows:
  - 🟡 30 Days Remaining
  - 🟠 14 Days Remaining
  - 🔴 7 Days Remaining
  - 🚨 Expired

---

## 6. Single-Database Multi-Tenancy & Landlord Controls

### Architectural Isolation
- Single-database multi-tenancy utilizing global `BelongsToOrganization` Eloquent scopes.
- Every tenant query is automatically scoped by `organization_id`, ensuring strict data isolation without the overhead of maintaining individual databases per tenant.

### Multi-Strategy Tenant Resolution
1. **Custom Domain / Subdomain**: Maps inbound hostnames (`tenant.domain.com`) directly to organizations.
2. **Session Context**: Preserves the active organization in user session state.
3. **Encrypted Cookie**: Long-lived `last_organization` cookie maintains workspace preferences across browser restarts.
4. **Pre-Login Organization Selection**: Clean `/organization` portal allowing users to input their organization code before authenticating.

### Central Landlord Management
- Super-administrators accessing the default organization can open `/central/tenants` to:
  - Provision new tenant organizations with unique names, slugs, and codes.
  - Assign tenant administrators.
  - Allocate and adjust monthly check credit limits.
  - Toggle active/suspended states.

---

## 7. Public Status Pages

- **Zero-Authentication Dashboards**: Every project can expose a dedicated public status page at `/status/{slug}`.
- **Real-Time Operational Indicators**:
  - 🟢 **All Systems Operational** (100% healthy endpoints)
  - 🟡 **Degraded Performance** (one or more endpoints experiencing high latency)
  - 🟠 **Partial System Outage** (one or more endpoints failing)
  - 🔴 **Major System Outage** (all critical endpoints down)
- **Availability Metrics**: Displays all-time and 30-day uptime percentages.
- **Transparent Incident Disclosures**: Historical 30-day incident log displaying failure timestamps, duration, and resolution summaries.

---

## 8. Executive Reliability & SLA Reporting

### Composite API Health Score (0–100)
Calculates a weighted reliability score reflecting true service quality:
$$\text{Health Score} = (0.40 \times \text{Availability}) + (0.35 \times \text{Performance}) + (0.25 \times \text{Reliability})$$
- **Availability Component**: Percentage of successful checks within the selected timeframe (24h, 7d, 30d).
- **Performance Component**: Percentage of checks completing under the monitor's warning latency threshold.
- **Reliability Component**: Penalty deduction based on error rate percentage and incident occurrences.

### Percentile Latency & Latency Regression
- Computes exact response time percentiles: **$p_{50}$**, **$p_{95}$**, and **$p_{99}$**, alongside Min, Max, and Average latencies.
- **Mean Time to Recovery (MTTR)**: Tracks the average duration in minutes required to resolve outages.
- **Latency Regression Tracking**: Compares the current window's $p_{95}$ against historical baseline metrics and automatically flags performance regressions ($\ge 25\%$ degradation).
- **Export & Print**: Custom CSS print layout enabling clean, branded PDF exports for executive and compliance reporting.

---

## 9. Multi-Channel Alerting & Webhooks

- **Granular Trigger Settings**: Configurable per project for Down, Recovery, High Latency, Latency Regression, Contract Violations, Agent Offline, SSL Expiration, and Low Credits.
- **Supported Channels**:
  - **In-App Notification Center**: Real-time bell counter with polling, read/unread states, and direct incident links.
  - **Transactional Email**: Clean HTML emails detailing incident causes and response times.
  - **Slack Incoming Webhooks**: Structured Slack Block Kit attachments with severity color coding (Red for Down, Green for Recovered, Amber for Degraded).
  - **Discord Webhooks**: Rich Discord embed cards with endpoint details and dashboard links.
  - **Custom Webhooks**: Standardized JSON payloads for PagerDuty, Opsgenie, or internal microservices.

---

## 10. Authentication Profiles & Enterprise Security

- **Reusable Auth Profiles**: Central repository for endpoint credentials:
  - Bearer Token authentication (with `.env` / secret reference resolution).
  - Basic Authentication (Username / Password).
  - Custom API Key injection (Header or Query parameter).
  - Dynamic Custom Headers.
- **Modern Authentication**:
  - Passwordless **FIDO2 / WebAuthn Passkeys** support.
  - Time-based One-Time Password (TOTP) Two-Factor Authentication.
- **Automated Secret Redaction**: Recursive `SecretSanitizer` automatically scrubs tokens, passwords, and sensitive headers from check logs and error traces.
- **Immutable Audit Trail**: Tracks all configuration changes and user actions with before-and-after property diffs via Spatie Activity Log.
