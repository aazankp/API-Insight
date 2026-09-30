# Screenshots Capture & Sanitization Guide

Since the production source code is private and runs with secure environment dependencies, this guide specifies the exact screenshots required to complete the public showcase repository, along with instructions on capturing and sanitizing each screen.

---

## 🛡️ Mandatory Privacy & Redaction Rules

Before saving and committing any screenshot to `screenshots/`:

1. **Emails**: Mask or replace real user emails (e.g., use `admin@example.com`, `engineer@company.org`).
2. **Personal Information**: Blur real employee names, profile avatars, and phone numbers.
3. **API Keys & Bearer Tokens**: Ensure credentials in Auth Profile forms, agent tokens, or request headers are redacted (e.g., `ag_••••••••••••`).
4. **Target URLs**: Redact internal hostnames, corporate staging domain names, or private IP addresses (use mock values like `https://api.example.com/v1/health` or `http://microservice.internal.local`).
5. **Customer & Business Data**: Obscure any confidential customer IDs, proprietary schema field names, or production database references.
6. **File Format & Sizing**: Save all images in PNG format with crisp dimensions (recommended: 1440x900 or 1920x1080), placed directly inside the `screenshots/` directory.

---

## 📸 Required Screenshots Checklist

| # | Recommended Filename | Page Route | UI View / State to Capture | Key Elements to Highlight |
| :-: | :--- | :--- | :--- | :--- |
| **01** | `login-and-passkeys.png` | `/login` & `/organization` | Authentication screen showing Organization code selection and FIDO2 Passkeys button. | Organization Code input, email/password form, WebAuthn Passkey button. |
| **02** | `dashboard-overview.png` | `/dashboard` | Main operational dashboard with active monitors, status cards, and metrics. | Uptime metric cards, quota capacity meter, active monitors table, status filters. |
| **03** | `monitor-observability-view.png`| `/monitors/{id}` | Detailed monitor view showing latency percentiles and time-series telemetry. | Response latency chart, $p_{50}/p_{95}/p_{99}$ latency metrics, SLA uptime, check log table. |
| **04** | `ai-detective-modal.png` | `/monitors/{id}` (Click **API Detective**) | Modal showing Gemini 1.5 Flash AI root-cause analysis output. | Root cause analysis, blast radius, suggested checks, recommended engineering action. |
| **05** | `contract-guard-editor.png` | `/monitors/{id}` (Tab: **Contract Guard**) | Schema validation interface with inferred JSON schema. | Inferred field types (`string`, `int`), nullability rules, structural MD5 hash. |
| **06** | `project-management.png` | `/projects/{id}` | Project settings view with SSL certificate details and alert channels. | SSL certificate validity badge (days remaining), alert channel checkboxes (Slack, Discord, Email). |
| **07** | `private-agent-management.png` | `/agents` & `/agents/{id}` | Private on-premise agent registry and connection status. | Connected agents table, online heartbeat status, setup token generation modal. |
| **08** | `incident-intelligence-log.png` | `/incidents` | Outage log table with duration and AI diagnostic summary. | Open vs. resolved badges, failure counts, duration minutes, likely root-cause summary. |
| **09** | `public-status-page.png` | `/status/{slug}` | Public status page view accessible without authentication. | System operational banner (Operational / Degraded), 30-day uptime bars, endpoint operational badges. |
| **10** | `executive-sla-report.png` | `/monitors/{id}/report` | Print-ready executive reliability and compliance SLA report. | Composite Health Score (0–100), latency regression summary, MTTR, print action button. |
| **11** | `auth-profiles-registry.png` | `/auth-profiles` | Reusable credential profiles for Bearer, Basic, and API Key authentication. | Profile list with masked token references, header injection settings. |
| **12** | `central-tenant-manager.png` | `/central/tenants` | Super-admin Landlord console for multi-tenant management. | Tenant table with codes, check credit quotas, active/suspended toggles. |
| **13** | `rbac-roles-permissions.png` | `/roles` & `/permissions` | Role-based access control matrix with tenant scoping. | Roles table (Super Admin, Admin, User), granular permission assignment checklist. |
| **14** | `user-management.png` | `/users` | Organization user management and team invites. | User table, assigned roles, 2FA status badges. |
| **15** | `audit-activity-log.png` | `/activity-logs` | Immutable audit trail of administrative actions. | Event name, causer, timestamp, and sanitized property diffs. |

---

## 📂 Destination Directory Structure

Place all captured and sanitized PNG files in the `screenshots/` directory:

```text
public-showcase/
├── screenshots/
│   ├── README.md
│   ├── login-and-passkeys.png
│   ├── dashboard-overview.png
│   ├── monitor-observability-view.png
│   ├── ai-detective-modal.png
│   ├── contract-guard-editor.png
│   ├── project-management.png
│   ├── private-agent-management.png
│   ├── incident-intelligence-log.png
│   ├── public-status-page.png
│   ├── executive-sla-report.png
│   ├── auth-profiles-registry.png
│   ├── central-tenant-manager.png
│   ├── rbac-roles-permissions.png
│   ├── user-management.png
│   └── audit-activity-log.png
```
