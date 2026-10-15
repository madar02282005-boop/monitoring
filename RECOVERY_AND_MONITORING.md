# Connection history, power events, maintenance, and backups

## Where to find the features

- Open a computer in **Devices**, then choose **Connection & maintenance**. History records pairing, lost connections, offline transitions, reconnects, Windows power events, and maintenance changes. Each device keeps up to 250 events.
- A **Super Admin** can schedule or cancel a maintenance window in that tab. Times are entered in the browser's local timezone. During an active window, telemetry and connection history continue, while automatic notifications and tickets are suppressed. Suppression ends automatically at the window's end time.
- A **Super Admin** can download and restore operational backups from **Rules & settings → Backup and recovery**. Download a current backup before restoring. Files must be smaller than 9 MB for the web restore tool.

## Windows power reporting

Install or update to Windows agent **2.7.0-ps1** using the existing Agent Update or installation flow. It advertises the `power_events` capability. It tries to report sleep/wake events immediately through `Win32_PowerManagementEvent`, then replays Windows System event-log records after successful reconnection. The upload cursor is saved locally only after a successful telemetry request. Event replay is limited to the previous seven days.

Sleep and shutdown interrupt network access. Immediate delivery is best effort, so some events appear only after waking or starting Windows again. When Windows does not supply usable event evidence, the dashboard shows an unknown cause. Older Windows agents and other agent types still report connections but do not supply Windows power events.

## What backups contain

Backups include inventory, departments, locations, diagnostic rules and findings, tickets, maintenance records, telemetry, network history, and each device's connection history and maintenance windows. Passwords, sessions, registration codes, and device tokens are excluded.

Restore validates the format, collection IDs, asset IDs, maintenance dates, and device references before replacing operational records. User accounts, logged-in sessions, and current monitoring settings are preserved. Pending commands and notifications are cleared. A matching existing device keeps its current pairing; computers missing from the current database receive fresh registration codes and require a new installation package. A restore is recorded in the audit log.

An export is a manual backup. Previously deleted records can only be recovered from an existing backup. If `public.monitoring_state` was deleted in Supabase, first run `supabase/migrations/20260821130000_monitoring_state.sql` to recreate it, then sign in and restore the exported file.

## API and validation

Both the local Express server and Cloudflare Worker provide:

- `GET /api/operations/backup`
- `POST /api/operations/restore` with `{ backup, confirmation: "RESTORE" }`
- `GET /api/operations/devices/:id/connection-history`
- `GET|POST /api/operations/devices/:id/maintenance-windows`
- `DELETE /api/operations/devices/:id/maintenance-windows/:windowId`
- `POST /api/agent/power-event` with a device bearer credential

The dashboard routes require a user session. Backup and maintenance mutations require Super Admin. Ordinary users can read only their assigned computers' operations. Device credentials authorize only their own power events.

Run `npm run lint`, `node --import tsx --test tests/operations.test.ts`, and `npm run build` for checks. Deployment uses the existing `CLOUDFLARE_DEPLOY.md` workflow; then update installed Windows agents to enable power reporting.
