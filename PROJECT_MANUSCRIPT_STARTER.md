# PC Monitoring System — Manuscript Starter

> **Purpose:** This document gathers the project’s implemented features and technical details into a starting point for a capstone, thesis, or software engineering manuscript. It is an editable working draft, not a completed research report. Replace bracketed fields with the actual institution, authors, study setting, deployment details, measurements, and references. Do not present sample data or unmeasured benefits as research findings.

## Working title

**Design and Development of a PC and Laptop Monitoring and Diagnostic Management System**

Alternative title: **A Web-Based Computer Asset Monitoring, Automated Diagnostics, and Maintenance Management System**

## Author and study details

- Author(s): [Names]
- Program / department: [Program and department]
- Institution: [Institution]
- Adviser: [Name]
- Study site: [Organization or laboratory, if applicable]
- Manuscript date: [Date]
- Software version / commit: [Record before final evaluation]

## Abstract — editable draft

Organizations that manage shared computers need a way to see device condition, identify operational problems, and coordinate repair work. This project presents the design and development of a web-based PC and laptop monitoring and diagnostic management system. The system combines a React and TypeScript administrative console, a server-side API, and installable agents that report hardware and operating-system telemetry. Its functions include device inventory, heartbeat and connection tracking, threshold-based diagnostic findings, health summaries, repair tickets, maintenance records and schedules, organizational records, reports, notifications, audit logs, and operational backup and restore. The application supports a local Express deployment with file-based persistence and a Cloudflare Workers deployment backed by a Supabase JSONB state record. A role-aware interface separates administrator functions from ordinary user access, while agent registration codes and per-device credentials support device pairing and telemetry submission. This manuscript describes the system requirements, architecture, implementation, and a proposed evaluation. **[Add the study site, participant or device counts, evaluation procedure, and measured results only after conducting the evaluation.]**

**Keywords:** computer monitoring; hardware telemetry; endpoint management; automated diagnostics; maintenance management; web application

## 1. Introduction

### 1.1 Background

Computers used in [institution / organization] support [teaching, office, laboratory, or other work]. Maintaining an inventory and responding to faults can involve separate records for device specifications, connectivity, reported problems, repair history, and assigned personnel. This project brings those operational records and live agent telemetry into a single console.

The software is designed for an administrator or support team that needs to associate a computer with an asset identifier, location, department, and assigned user; review recent telemetry and diagnostic evidence; and track follow-up through ticketing and maintenance records. A lightweight endpoint agent reports measurements to the server. The web console presents the resulting status and provides administrative workflows.

> **Research framing to complete:** Describe the actual workflow and documented problem at the study site. Support claims about service delays, failures, inventory gaps, or costs with local evidence or credible sources. The repository alone does not establish that these problems occur at a particular institution.

### 1.2 Problem statement

The project addresses the need for a consolidated way to:

1. Maintain a searchable inventory of computers and their organizational assignments.
2. Observe agent connection state and recent hardware, system, and network telemetry.
3. Surface diagnostic findings based on configurable rules and telemetry evidence.
4. Coordinate incident response through repair tickets and maintenance records.
5. Preserve operational history and provide reports, audit records, and backups.

Convert these needs into the study site's specific problem statements after interviews, observation, or document review.

### 1.3 Objectives

**General objective:** Design and develop a web-based PC and laptop monitoring and diagnostic management system for [study site].

**Specific objectives:**

1. Develop an inventory for devices, departments, and physical locations.
2. Implement agent registration and collection of device telemetry.
3. Present connection state, telemetry history, health summaries, and diagnostic findings.
4. Provide workflows for tickets, maintenance, notifications, and audit history.
5. Implement role-based access and operational backup and restore.
6. Evaluate the system using [functional testing / usability instrument / performance measures / expert review] and report the results.

Revise these objectives to match the final approved research questions and evaluation plan.

### 1.4 Scope and delimitations

The current source includes a browser-based administrative console, API services, PowerShell / Python / Node.js agent templates, device pairing, telemetry ingestion, diagnostic evaluation, network diagnostics data structures, ticket and maintenance workflows, reporting, and two deployment paths. The application models devices, users, departments, locations, diagnostic rules and issues, tickets, maintenance records, notifications, audit logs, and settings.

The system depends on an installed and functioning agent for live device evidence. A computer without telemetry is deliberately represented as having unavailable health rather than receiving a fabricated health score. Agent and hardware capabilities vary by operating system and agent implementation. This project does not, based on the code reviewed, establish clinical-grade reliability, predictive maintenance accuracy, production-scale performance, or measured improvement in repair time. Those claims require a separate evaluation.

### 1.5 Significance

Potential beneficiaries include [IT administrators], [support technicians], [department heads], and [computer users] at the study site. The system provides a shared view of inventory, device signals, findings, and service records. Describe expected benefits as intended outcomes until they are measured. The evaluation should identify whether and how each stakeholder's workflow improved.

## 2. System overview

### 2.1 User roles and major screens

The application defines two roles: `super_admin` and `user`. Server-side middleware enforces authentication and role checks; hiding a screen or button in the interface is not the security boundary. The navigation covers:

| Area | Purpose |
|---|---|
| Command center | Fleet summary, health/status indicators, alerts, and recent tickets |
| Computers | Inventory, filters, device details, telemetry, connection history, and agent workflows |
| Diagnostics and findings | Review findings, evidence, rules, and recommended actions |
| Repair tickets | Track assignment, investigation, repair, resolution, verification, notes, and attachments |
| Maintenance history | Record service actions and schedule device maintenance windows |
| Laboratories and rooms | Maintain physical locations |
| Departments | Maintain organizational groupings |
| User accounts | Manage console accounts subject to administrator permissions |
| Reports and export | View summary data and export supported report formats |
| Audit trail | Review recorded administrative and operational changes |
| Rules and settings | Configure monitoring thresholds and operational settings; includes backup and recovery |

### 2.2 Device data and telemetry

Each device record can include an asset ID, name, type, assigned user, department, location, manufacturer, model, serial number, operating system, registration and device credentials, connection state, heartbeat timestamps, agent version and capabilities, and operational history. Hardware specifications are stored separately and linked by device ID.

Telemetry fields represented in the shared TypeScript model include CPU and memory use, CPU and GPU temperatures, storage capacity and health, battery state, network adapter and traffic information, Wi-Fi diagnostics, uptime and boot time, process observations, fan speed, and motherboard temperature. Availability depends on what a particular agent and computer can report. The API retains a latest telemetry object plus a bounded history of selected snapshots; the configured default retention is 120 telemetry points.

### 2.3 Diagnostics and health

Diagnostic rules have a code, metric, threshold, unit, duration field, severity, enabled state, possible causes, and recommended actions. The rule catalog covers CPU and memory load, low disk space, CPU and GPU temperature, fan behavior, low or degraded battery, storage SMART warning, offline devices, and network instability. The server's diagnostic engine evaluates incoming telemetry, creates or updates findings, and can link a finding to a repair ticket. Settings include heartbeat and offline thresholds, network diagnostic thresholds and cooldown, telemetry retention, automatic ticket behavior, and sound alerts.

The device health score is deterministic and evidence-based. When telemetry is missing, the score is null and the level is `Unavailable`. With telemetry, the score starts at 100 and is reduced by configured logic for connectivity, high CPU or memory use, low storage, high CPU temperature, active findings, recurring findings, and overdue maintenance. The result includes reasons, recommendations, level, and calculation timestamp. It is a software heuristic, not a validated prediction of hardware failure; do not describe it as predictive without an evaluation.

### 2.4 Tickets, maintenance, and history

Repair tickets contain a ticket number, linked device and optional diagnostic issue, severity and priority, description, status, technician assignment, dates, resolution and verification fields, notes, and attachment metadata. Maintenance records store the problem, diagnosis, actions performed, parts or software, result, technician, and notes. Device connection history records events such as registration, reconnect, connection loss, offline transitions, sleep and resume, shutdown and startup, maintenance scheduling or cancellation, and restoration.

Administrators can schedule and cancel device maintenance windows. During an active window, the operations logic suppresses automatic notifications and tickets associated with the monitored device while telemetry and connection history continue. Power-event reporting is best effort: sleep or shutdown can interrupt communication, so some Windows events may only be delivered after the device reconnects.

### 2.5 Security and roles

Console users authenticate through the API and receive a bearer session token. The frontend attaches the token to protected API requests. Device agents pair using a registration code and then use a device-specific token for heartbeat, telemetry, and power-event requests. Registration codes are single-use after pairing. Server routes apply session and role checks, and agent endpoints validate device credentials.

The current local auth implementation uses SHA-256 password hashing and stores active sessions in memory; sessions therefore do not survive a local server restart. These are implementation facts to disclose and review before production use. For a production security assessment, evaluate password hashing with a salted adaptive algorithm, session lifecycle and storage, secret rotation, transport security, least-privilege deployment, input validation, and retention of process/user telemetry. The manuscript should not include real secrets, credentials, personal data, or tokens.

### 2.6 Backup and restore

The operations module exports inventory, organizational data, rules and findings, tickets, maintenance records, telemetry, network history, and per-device connection and maintenance history. It excludes user passwords, sessions, registration codes, and device tokens. Restore validates format, record identifiers, asset IDs, dates, and references before replacing operational data; user access and current settings are preserved. New devices restored without an existing pairing receive fresh registration codes. Restore requires an explicit confirmation value and is recorded in the audit log. Backups should still be stored and handled as sensitive operational data.

## 3. Methodology and system development

Use the development method approved for the study (for example, iterative prototyping, Agile, or a software development life cycle). The repository does not identify a formal research method, participant sample, or completed user study. Fill in this section from actual project records.

### 3.1 Requirements gathering

- Stakeholders consulted: [roles and number]
- Methods: [interviews, observation, document review, survey]
- Functional requirements: [final list]
- Quality requirements: [security, usability, compatibility, reliability, performance]
- Approval or ethics process, if applicable: [details]

### 3.2 Design and implementation

The frontend is a single-page application built with React, TypeScript, Vite, Tailwind CSS, and Recharts. `src/App.tsx` composes the main navigation and views; shared data types are in `src/types/index.ts`; authentication and monitoring data are managed by React contexts. The local server uses Express and TypeScript (`server.ts`, `server/routes/`). Diagnostic logic, health calculations, and operational history/backup logic are in `server/diagnostic-engine.ts`, `server/health.ts`, and `server/operations.ts`.

Agent script generation is implemented in `server/agent-templates.ts`. The API includes registration, heartbeat, telemetry, power-event, command, and agent-download routes. Diagnostic evaluation is triggered when telemetry arrives. A five-second heartbeat check runs in the local server. The Cloudflare Worker implementation is in `workers/api.ts` and has a scheduled task configured for once per minute.

### 3.3 Architecture and persistence

The source supports two runtime configurations:

1. **Local / Express:** React assets are served through Vite in development; the Express server provides API routes. The default database is `data/monitoring-db.json`, with writes debounced by 500 ms. `DATA_DIR` can change the data directory.
2. **Cloudflare Worker:** The Worker serves static assets and API requests. Persistent application state is stored in a single row (`id = 1`) in the Supabase `public.monitoring_state` table as JSONB. The Worker accesses Supabase using a server-side secret. The included SQL migration enables row-level security and does not grant direct browser access.

The local and Worker implementations share domain concepts but should not be assumed to have identical behavior in every feature. The local agent command endpoint currently returns no command; the Worker includes scheduled processing and remote command handling. Identify the deployment used for the study and evaluate that exact build.

### 3.4 Operational flow

1. An administrator creates a device record and issues a registration code from the inventory workflow.
2. The agent is installed or generated with the server URL and registration code.
3. The agent registers; the server validates the code, records reported hardware details, consumes the pairing code, and issues/uses the device credential flow.
4. The agent submits heartbeat and telemetry data using its device token.
5. The API updates connection state, latest readings, retained history, and diagnostic findings.
6. A user reviews findings and creates or tracks a ticket; technicians record repair and verification details.
7. Administrative records, exports, backups, and audit entries support ongoing operations.

Confirm the exact pairing and token exchange against the deployed version before using this sequence as a formal protocol description.

### 3.5 Technology stack

| Layer | Technologies in the repository |
|---|---|
| User interface | React 19, TypeScript, Vite, Tailwind CSS, Recharts, Lucide icons, Motion |
| Local application server | Node.js, Express, TypeScript via `tsx` |
| Cloud runtime | Cloudflare Workers, Wrangler, static asset binding |
| Cloud persistence | Supabase Postgres; single JSONB state row via PostgREST |
| Endpoint integrations | Generated PowerShell, Python, and Node.js agent scripts |
| Testing framework present | Node.js built-in test runner (`node:test`) |

## 4. Evaluation plan and results

Do not fill this chapter with expected or invented results. Run the final deployed build with a documented protocol and record actual evidence.

### 4.1 Suggested functional evaluation matrix

| Feature | Example test case | Evidence to record |
|---|---|---|
| Login and roles | Valid / invalid login; ordinary user attempts restricted operation | Expected and actual status, role, result |
| Device inventory | Create, edit, view, and delete an asset | Result, response time, errors |
| Agent pairing | Valid, invalid, and reused registration code | Registration outcome; credential handling |
| Telemetry | Submit representative payloads and retrieve latest/history | Accepted fields, timestamps, retention behavior |
| Diagnostics | Cross rule thresholds and clear the condition | Finding creation/resolution and evidence |
| Tickets | Create, assign, update, resolve, verify | State transitions and audit trail |
| Maintenance | Schedule, cancel, and test active suppression | Window validity and notification/ticket behavior |
| Backup and restore | Export, corrupt input, restore valid backup | Validation outcome, preserved records, audit entry |
| Reports | Request summary and supported exports | Content, format, completeness |
| Deployment | Local and/or Worker health endpoint and user workflow | Version, configuration, logs, availability |

### 4.2 Measures to report

Select measures appropriate to the research questions. Possible measures include test pass rate, API response time under a stated workload, telemetry-to-dashboard delay, usability questionnaire scores, task completion time, and user-rated usefulness. Report the device count, duration, payload frequency, hardware/network environment, software version, test scripts, and any exclusions. A small controlled demonstration does not establish fleet-scale performance.

### 4.3 Existing automated test coverage

The repository contains `tests/operations.test.ts`. Its cases cover backup secret exclusion and restore validation, role checks and explicit restore confirmation, maintenance-window behavior, connection transition history, power-event deduplication, data aliases, Worker persistence and scheduled offline handling, and generated PowerShell syntax parsing on Windows. This describes test cases present in source; it does not assert that the test suite was run or passed for the current manuscript.

## 5. Discussion guide

Discuss findings against the objectives and collected evidence. Address:

- Which inventory and support tasks the system enabled at the study site.
- Whether agent readings were available and accurate on the tested device types.
- Whether configurable findings were understandable and actionable to support staff.
- How role controls, audit events, backup, and restore behaved in the evaluation.
- Performance and operational constraints observed under the tested conditions.
- Differences between the local Express and Cloudflare/Supabase deployment, if both were assessed.
- Risks around agent permissions, sensitive telemetry, credentials, retention, and network failures.

State limitations clearly: telemetry depends on agent installation, connectivity, and hardware/OS support; health scoring is deterministic heuristic logic; the repository does not provide evidence of predictive accuracy or large-scale performance; sample organization records are not study-site evidence; and feature parity between deployment paths needs verification.

## 6. Conclusion and recommendations — complete after evaluation

Summarize whether each objective was achieved, supported by measured evidence. Avoid broad claims beyond the study population and test conditions. Recommendations may include expanding device/OS coverage, strengthening authentication and session handling, conducting a larger usability or load study, validating diagnostic thresholds with domain experts, and improving operational monitoring. Retain only recommendations justified by the observed results.

## Figures and tables to prepare

1. System context diagram: administrator, ordinary user, endpoint agents, application server, and database.
2. Deployment architecture diagram: local Express/JSON file and Cloudflare Worker/Supabase paths.
3. Agent registration and telemetry sequence diagram.
4. Entity relationship or domain model for devices, locations, departments, telemetry, issues, tickets, and maintenance.
5. User interface screenshots from the final evaluated build, with sample/personal data removed.
6. Functional test results table and any study questionnaire summary.

## Repository map for manuscript authors

| Path | Manuscript use |
|---|---|
| `src/App.tsx` and `src/components/` | Screens, navigation, user workflows |
| `src/types/index.ts` | Domain entities and telemetry data structures |
| `src/context/` | Frontend session and monitoring data flow |
| `server.ts`, `server/routes/` | Local API, authentication, and authorization |
| `server/diagnostic-engine.ts` | Threshold evaluation and finding lifecycle |
| `server/health.ts` | Device health heuristic |
| `server/operations.ts` | Backups, restore, connection events, maintenance windows, power events |
| `server/agent-templates.ts` | Generated endpoint-agent scripts |
| `workers/api.ts` | Cloudflare Worker API and scheduled processing |
| `supabase/migrations/20260821130000_monitoring_state.sql` | Cloud persistence table and access policy |
| `tests/operations.test.ts` | Existing automated operations test cases |
| `RECOVERY_AND_MONITORING.md` | Connection history, power events, backup/restore behavior |
| `CLOUDFLARE_DEPLOY.md` | Cloud deployment instructions |

## References to add

Use the citation style required by the institution. Add primary references for endpoint monitoring, IT asset management, diagnostic or rule-based systems, software quality evaluation, security and privacy, and the selected development methodology. Cite official documentation for implementation-specific platform claims (React, Node.js, Express, Cloudflare Workers, and Supabase). Cite the actual study instruments and local policies used. Do not cite this repository as evidence that the system improved outcomes; use the evaluation results for that.

## Author completion checklist

- [ ] Replace every bracketed placeholder and remove instructions not appropriate for the final manuscript.
- [ ] Confirm the deployment path, version, environment, and final source revision.
- [ ] Describe the real study-site problem and requirements with evidence.
- [ ] Identify participants, devices, consent/ethics process, test protocol, and instruments where applicable.
- [ ] Execute and record the tests for the exact version being reported; distinguish automated tests from user evaluation.
- [ ] Insert actual results, analysis, and limitations; do not invent values.
- [ ] Capture sanitized screenshots and diagrams from the evaluated build.
- [ ] Verify all security, deployment, and API behavior claims against the selected runtime.
- [ ] Add complete citations and format the manuscript to institutional requirements.
- [ ] Remove usernames, emails, IP addresses, tokens, credentials, and identifying telemetry from examples.
