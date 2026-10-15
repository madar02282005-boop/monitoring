# PC & Laptop Monitoring and Diagnostic Management System

## Document purpose

This document is a project-wide guide to the codebase. It describes the product, users, features, major modules, architecture, data flow, API surface, setup, deployment options, and current implementation constraints. It is based on the repository source and configuration. Treat it as implementation documentation, not as proof of production readiness or measured research outcomes.

For a thesis or capstone structure with draft abstract and evaluation plan, see [PROJECT_MANUSCRIPT_STARTER.md](PROJECT_MANUSCRIPT_STARTER.md).

## 1. Project summary

The project is a web-based console for monitoring and managing desktop computers, laptops, workstations, mini PCs, all-in-one computers, and servers. Endpoint agents report hardware specifications, heartbeats, and telemetry to an API. Administrators and support users use the console to review inventory, connection state, system health, diagnostic findings, repair tickets, maintenance, reports, and audit activity.

The system combines three main parts:

1. **Web console:** React and TypeScript single-page application.
2. **API and operational logic:** Express server for local use, plus a Cloudflare Worker implementation for cloud deployment.
3. **Endpoint agents:** Generated PowerShell, Python, and Node.js scripts that register devices and send telemetry.

The repository supports two persistence modes: local JSON-file storage for the Express application and Supabase Postgres JSONB storage for the Cloudflare Worker deployment.

## 2. Users and access model

Two application roles are defined in `src/types/index.ts`:

- **Super Admin (`super_admin`):** May manage administrative records, settings, device inventory, diagnostics, tickets, maintenance, users, backups, and other restricted operations, subject to route-level rules.
- **User (`user`):** May sign in and access authorized monitoring and support functions. Device visibility is restricted according to assignment/authorization rules in the API.

Authorization is enforced in backend middleware and route handlers. Frontend visibility should not be treated as the security boundary. Agent requests use device credentials rather than a console user's session. A registration code pairs an agent with an inventory device; the code is consumed after successful registration, and subsequent agent requests use a per-device token.

## 3. Functional modules

### 3.1 Command center

The dashboard presents a summary of device totals and statuses, open and resolved tickets, active and critical findings, selected hardware warning counts, network health, recent alerts, and recent tickets. The summary is assembled through the reporting API and rendered by `src/components/dashboard/DashboardView.tsx`.

### 3.2 Computer inventory and device details

The inventory supports device records with asset ID, device name and type, assigned user, department, location, manufacturer/model/serial number, operating system, status, connection state, registration lifecycle, and agent version/capabilities. Device views expose the latest telemetry and relevant detail panels.

Device-related interface files include `DeviceList.tsx`, `DeviceCard.tsx`, `DeviceDetailsView.tsx`, `AddDeviceModal.tsx`, `InstallAgentModal.tsx`, and `AgentUpdateModal.tsx`. Supporting panels include connection history and network diagnostics. Device deletion behavior is also supported by `src/lib/device-purge.ts`.

### 3.3 Agent and telemetry management

Agent API routes support registration, power-event upload, heartbeat, telemetry submission, command polling, and generated script download. Registration can capture hardware specifications and agent metadata. Telemetry is associated with the registered device and can include:

- CPU utilization and temperature
- Memory use
- Storage capacity, usage, health, and SMART status
- GPU metrics, where supported
- Battery state and health, where supported
- Network adapter, traffic, Wi-Fi, and connectivity diagnostics
- Uptime, boot time, observed processes, fan speed, and motherboard temperature, where supported

The actual fields depend on the endpoint OS, agent version, privileges, and hardware. The server stores a latest telemetry record and a bounded series of selected telemetry snapshots. The default telemetry retention is 120 points, configurable in system settings.

The agent script generators reside in `server/agent-templates.ts`. Agent update and installation dialogs are in `src/components/devices/`.

### 3.4 Diagnostics and findings

`server/diagnostic-engine.ts` evaluates incoming telemetry against enabled diagnostic rules and manages associated issue records. Rule categories represented by the shared domain model include:

- High CPU usage
- High RAM usage
- Low disk space
- High CPU or GPU temperature
- Fan failure / low fan speed under thermal load
- Low battery and degraded battery health
- Disk SMART warning
- Device offline
- Network instability

Findings include severity, status, detection time, evidence, possible causes, recommended actions, and optional ticket linkage. Rules can be reviewed or edited through the diagnostics and settings interfaces. `src/components/diagnostics/DiagnosticRulesModal.tsx` manages rule editing; `DiagnosticsView.tsx` presents findings.

### 3.5 Health calculation

`server/health.ts` computes a deterministic health score from telemetry, connection state, active findings, recurring findings, and maintenance recency. The score begins at 100 and deducts points for configured conditions, with severity levels from Excellent to Critical. Without telemetry, the device receives a null score and Unavailable status plus a recommendation to install or reconnect the agent. This is a transparent heuristic, not a machine-learning model or validated failure prediction.

### 3.6 Repair tickets

Tickets link work to a device and may link to a diagnostic issue. Records include severity, priority, description, assignment, status, lifecycle dates, resolution, verification, notes, and attachment metadata. Ticket statuses include Open, Assigned, Diagnosing, In Repair, Waiting for Parts, Resolved, and Closed. Ticket-related UI resides in `src/components/tickets/`.

### 3.7 Maintenance and connection history

Technicians can enter maintenance records containing the reported problem, diagnosis, action, replaced parts, installed software, result, and notes. Administrators may create or cancel future maintenance windows for a device. Connection history records registration/reconnection, lost or offline states, Windows power events when reported, maintenance changes, and restore activity.

Active maintenance windows suppress automatic notifications and tickets associated with the covered device in supported operational logic, while telemetry and connection history continue. Windows sleep/shutdown events may be delivered only after the endpoint reconnects, because network transmission is unavailable while the computer is asleep or powered off.

### 3.8 Organization and user administration

Organization screens maintain departments and physical locations, including room, building, floor, and location type. User administration manages console accounts under administrator restrictions. These records associate devices and users with an operational context for filtering and reporting.

### 3.9 Reports, notifications, and audit

The reporting API provides a dashboard summary and supported exports. Notifications can be read individually or marked read in bulk. Audit records capture selected security and operational actions, including administrative changes and backup restore events. Audit log access is restricted to Super Admins in the local server route setup.

### 3.10 Backup and recovery

Operations endpoints export and restore operational data. The backup includes inventory, departments, locations, diagnostic rules and findings, tickets, maintenance, telemetry, network history, connection history, and maintenance windows. Passwords, sessions, registration codes, and device tokens are excluded.

Restore validates the backup structure and references before replacing operational records. It preserves user accounts, active sessions, and current monitoring settings; clears pending commands and notifications; preserves pairing for matching existing devices; and creates fresh registration codes for newly restored devices. A restore is audit logged. See [RECOVERY_AND_MONITORING.md](RECOVERY_AND_MONITORING.md) for operator-facing details.

## 4. Architecture

### 4.1 Frontend

The frontend is a React 19 and TypeScript single-page application built with Vite. Tailwind CSS supplies utility styling, Recharts is used for charting, Lucide provides icons, and Motion is present for UI animation. `src/main.tsx` initializes the application and configures API requests. `src/App.tsx` composes authentication, navigation, and the active page.

State responsibilities are divided between:

- `src/context/AuthContext.tsx`: current console user and authentication session state.
- `src/context/MonitoringContext.tsx`: loaded monitoring data, active navigation, selected device, refresh, and related application state.
- `src/types/index.ts`: shared domain types for frontend and server code.

### 4.2 Local Express runtime

`server.ts` mounts Express API routes, starts a five-second heartbeat-monitor loop, and runs Vite in middleware mode during development. The default port is 3000 and can be overridden with `PORT`. In production mode, the server serves built assets from `dist`.

The local database implementation in `server/db.ts` reads and writes `data/monitoring-db.json` by default. `DATA_DIR` can change the directory. Writes are debounced and written through a temporary file followed by rename. The local database initializes built-in development records when no valid store is present.

### 4.3 Cloudflare Worker runtime

`workers/api.ts` is the Cloudflare Worker entry point. Wrangler serves the built `dist` assets and routes `/api/*` through the Worker. A cron trigger runs every minute. The Worker uses the Supabase REST interface to read and write one row in `public.monitoring_state`; the application's data is held in the row's JSONB `data` value. `SUPABASE_SECRET_KEY` is a server-side secret and must not be included in browser code.

The migration `supabase/migrations/20260821130000_monitoring_state.sql` creates the state table and enables row-level security. Deployment steps are documented in [CLOUDFLARE_DEPLOY.md](CLOUDFLARE_DEPLOY.md).

### 4.4 Runtime differences

The Express and Worker implementations provide similar product concepts but are separate server implementations. Do not assume complete feature parity. For example, the local agent command endpoint currently responds with no command, while the Worker implementation includes scheduled command processing. The chosen runtime must be recorded and tested for any formal deployment or manuscript.

## 5. Data model

The primary record groups represented in `src/types/index.ts` and the persistence layer are:

| Record group | Examples of data |
|---|---|
| Users | Username, email, role, department, timestamps; password hashes are server-side only |
| Departments | Name, code, description, head, contact email |
| Locations | Name/type, building, floor, room, department |
| Devices | Asset identity, assignment, specifications, status, pairing, connection/power state |
| Hardware specifications | CPU, memory, storage, GPU, motherboard, BIOS, OS build |
| Telemetry | Latest payload and bounded historical snapshots |
| Diagnostic rules | Metric, threshold, duration metadata, severity, enablement, guidance |
| Diagnostic issues | Device, rule, evidence, severity, status, causes, recommendations |
| Repair tickets | Device/issue linkage, assignment, priority, status, notes, verification |
| Maintenance records | Work performed, technician, parts/software, outcome |
| Maintenance windows | Start/end, reason, creator, cancellation timestamp |
| Connection history | Event type, detail, occurrence and record timestamps |
| Notifications | Message, type, read state, related device/issue/ticket |
| Audit logs | User, action, entity, detail, timestamp |
| Settings | Heartbeat/offline and network thresholds, retention, ticket and alert behavior |

## 6. API surface

The local Express server mounts the following route groups. Exact behavior and access rules are defined in `server.ts` and `server/routes/`.

| Route group | Representative operations |
|---|---|
| `/api/auth` | Login, current session, logout |
| `/api/devices` | List, retrieve, create, update, delete, telemetry history, regenerate token |
| `/api/agent` | Register, power event, heartbeat, telemetry, commands, download agent |
| `/api/diagnostics` | List issues/rules, update rules, update finding status, create ticket from issue |
| `/api/tickets` | List/retrieve/create/update, notes, resolve, verify, resolve-and-log |
| `/api/maintenance` | List/create maintenance records |
| `/api/org` | Department and location CRUD operations |
| `/api/users` | User list and administration |
| `/api/settings` | Read and update monitoring settings |
| `/api/audit` | Read audit entries |
| `/api/notifications` | List, mark read, mark all read |
| `/api/reports` | Summary and report export |
| `/api/operations` | Backup/restore, device connection history, maintenance windows |
| `/api/health` | Service health response |

Agent route authentication is device credential based for telemetry functions. Dashboard and administration routes require a console session, with additional role checks for restricted mutations. The exact Worker route implementation should be checked separately when documenting a Worker deployment.

## 7. Setup and local operation

Requirements: Node.js compatible with the project dependencies, npm, and a Windows, macOS, or Linux development environment. Endpoint PowerShell event capabilities apply to Windows agents.

From the `COMPUTER-MONITORING` directory:

```bash
npm install
npm run dev
```

The local application is served at `http://localhost:3000`. The health endpoint is `http://localhost:3000/api/health`. The server creates/loads `data/monitoring-db.json` as its local store. For a fresh development database, ensure any existing data is backed up before changing or removing the store.

Other available scripts:

```bash
npm run lint       # TypeScript check
npm run build      # Build frontend assets and bundled local server
npm start          # Run the built local server
npm run worker:dev # Build assets and run Wrangler locally
npm run worker:deploy
```

`npm run clean` is defined for removing generated build outputs. Consult `package.json` for the authoritative scripts.

## 8. Configuration and deployment

### Local configuration

- `PORT`: Express listener port; defaults to 3000.
- `DATA_DIR`: local JSON database directory; defaults to `data/` under the working directory.
- `NODE_ENV`: production mode serves `dist`; development mode uses Vite middleware.
- `CORS_ORIGIN`: optional allowed-origin list used by the Express middleware.
- `VITE_API_URL`: optional API base URL used by the frontend when compiled.

### Cloud deployment

The Cloudflare instructions in `CLOUDFLARE_DEPLOY.md` describe creating the Supabase state table, configuring `SUPABASE_SECRET_KEY` in Wrangler, building, and deploying. `wrangler.jsonc` sets the Worker entry point, compatibility settings, asset binding, cron schedule, and Supabase URL variable. Verify account, project, URL, and secret configuration in the target environment rather than copying values from a development setup.

## 9. Security, privacy, and operational notes

- Never commit or publish actual Supabase secrets, device tokens, registration codes, user passwords, session tokens, or unredacted operational backups.
- The local authentication implementation hashes passwords with SHA-256 and keeps active sessions in process memory. Sessions are lost when the local server restarts. Evaluate a salted, adaptive password-hashing algorithm and durable session handling before production use.
- Use HTTPS for deployed agent-to-server communication and apply least-privilege controls to API and database secrets.
- Telemetry can contain usernames, process names, network details, and hardware identifiers. Define collection purpose, retention, access, user notice, and deletion practices for the deployment context.
- Backups contain sensitive operational records even though authentication and pairing secrets are excluded; restrict and protect backup files.
- The local JSON database is intended for local operation and is not a substitute for a transactional multi-instance database.
- Health scores and diagnostic thresholds are implementation rules. They should be reviewed by qualified support staff and validated against operational evidence.

## 10. Tests and verification

`tests/operations.test.ts` contains Node's built-in test cases for backup and restore validation, role restrictions, maintenance windows, connection transitions, power event deduplication, local-store aliases, Worker persistence/scheduled behavior, and generated PowerShell syntax parsing on Windows. The existence of test cases does not mean they have been executed or passed for a particular release.

To run the operations test file in a compatible environment:

```bash
node --import tsx --test tests/operations.test.ts
```

For release documentation, record the commit, runtime, environment, commands executed, test results, and any skipped tests.

## 11. Known constraints and items to verify

1. Confirm role permissions and device scoping against the precise deployed version.
2. Verify agent generation, installation, update, and pairing flows with actual supported operating systems.
3. Confirm telemetry freshness and retention under expected heartbeat volume.
4. Validate diagnostic thresholds and health-score interpretation with subject-matter experts.
5. Review security requirements, especially password hashing, session durability, secrets, and sensitive telemetry.
6. Check feature parity between the local Express and Cloudflare Worker implementations.
7. Test backup restoration on a safe copy of the selected deployment before relying on it for recovery.
8. Use actual measurements before making claims about reliability, accuracy, usability, cost savings, or repair-time improvement.

## 12. Repository map

```text
COMPUTER-MONITORING/
├── src/
│   ├── App.tsx                  Main page composition and navigation state
│   ├── main.tsx                 React bootstrap and API request setup
│   ├── context/                 Authentication and monitoring contexts
│   ├── components/              Dashboard, devices, diagnostics, tickets, etc.
│   ├── lib/                     Frontend support utilities
│   └── types/                   Shared application and telemetry types
├── server.ts                    Local Express/Vite application entry point
├── server/
│   ├── routes/                  Express endpoint handlers
│   ├── db.ts                    Local JSON persistence and seed data
│   ├── diagnostic-engine.ts     Telemetry rule processing
│   ├── health.ts                Deterministic device health score
│   ├── operations.ts            Backup, restore, history, power events, windows
│   └── agent-templates.ts       Generated endpoint-agent scripts
├── workers/api.ts               Cloudflare Worker API implementation
├── supabase/migrations/          Cloud state table migration
├── tests/                       Automated operations test cases
├── data/                         Local JSON database (runtime data; treat as sensitive)
├── PROJECT_DOCUMENTATION.md     This project-wide guide
└── PROJECT_MANUSCRIPT_STARTER.md Thesis/capstone manuscript scaffold
```

## 13. Related project documents

- [PROJECT_MANUSCRIPT_STARTER.md](PROJECT_MANUSCRIPT_STARTER.md) — draft manuscript structure and study completion checklist.
- [RECOVERY_AND_MONITORING.md](RECOVERY_AND_MONITORING.md) — connection history, Windows power events, maintenance suppression, backup and restore.
- [CLOUDFLARE_DEPLOY.md](CLOUDFLARE_DEPLOY.md) — Cloudflare Worker and Supabase deployment procedure.
