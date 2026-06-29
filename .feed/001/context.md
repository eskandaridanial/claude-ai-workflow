# Context — 001

## Goal
A new Python service that tracks API rate limit usage across internal microservices. Services push call data via REST; a dashboard shows per-API usage percentage and time series, with warnings visible when thresholds are breached.

## Scope
### In scope
- Services report external API calls via REST POST
- Per-API usage dashboard showing current % used (progress bar) and time series
- Configurable rate limits per external API (YAML config file)
- Configurable warning thresholds per API (default 80%)
- Auto-creation of external API entries on first reported call
- 30-day data retention
- Docker deployment

### Out of scope
- Blocking or throttling actual API calls
- Enforcing rate limits on behalf of services
- Real-time notifications (Slack, email, webhooks)
- User authentication for the tracking service
- Horizontal scaling / distributed storage
- Admin API for rate limit configuration (config file only)

## Domain & Definitions
- **Service**: An internal microservice that makes calls to external third-party APIs (e.g. payment provider, SMS gateway)
- **External API**: A specific third-party API being called (e.g. "stripe", "twilio"). Identified by a string name
- **Rate Limit**: The maximum number of calls permitted within a defined time window for a given external API (e.g. "1000 req/min"). Configured per external API
- **Usage Record**: A single reported external API call, containing: service name, external API ID, timestamp, HTTP status code, and rate limit headers if available
- **Threshold**: The percentage of rate limit usage at which a warning is triggered. Default 80%, configurable per API
- **Dashboard**: The web UI showing per-API usage and historical time series

## Requirements
1. A running service exposes a REST endpoint that accepts POST with a usage record (service name, API ID, timestamp, status code, rate limit headers)
2. Usage records are stored in memory (30-day retention, then discarded)
3. A YAML config file defines rate limits: mapping of API name → limit (e.g. `stripe: 1000`)
4. A YAML config file defines thresholds: mapping of API name → percentage (e.g. `stripe: 80`), defaulting to 80% if absent
5. When a new external API is encountered in a usage record, it is auto-created in the system with no prior configuration required
6. A dashboard is served by the service showing:
   - Per-API: current usage % as a progress bar, based on the most recent full window
   - Per-API: time series of usage over time
7. When usage % exceeds a threshold, this is visually indicated on the dashboard
8. The service runs inside a Docker container

## Constraints
- Scale: <5 services reporting, <10K reported calls/day — in-memory storage is sufficient
- Warning mechanism: Dashboard only. No Slack, email, or webhook notifications
- Stack: Python (web framework TBD)
- Deployment: Docker (single container)
- Rate limit configuration: YAML file, requires service restart to take effect
- No authentication on the reporting endpoint or dashboard (internal service only, network-level access control assumed)

## Dependencies
- None — brand new greenfield service

## Edge Cases & Decisions
- **Missing rate limit headers**: If a reported call does not include rate limit headers from the external API, the call is still recorded. The service cannot self-detect limits without headers
- **New external API first call**: External API entry is auto-created with no explicit setup. Rate limit and threshold use defaults (no limit configured yet, 80% threshold). An operator must add the rate limit to the config file to enable accurate usage tracking
- **Threshold breach**: Warning is shown on the dashboard but no active notification is sent. Teams check manually as needed
- **Service restart**: In-memory storage means all data is lost on restart. Acceptable given small scale and non-critical nature
- **No authentication**: Reporting endpoint and dashboard are open. Assumes network-level access control (only internal services can reach it)

## Success Criteria
- A service can POST a usage record and it appears in the dashboard
- Dashboard shows correct usage % for an API with a configured rate limit
- Dashboard shows time series of usage over time
- When usage exceeds threshold, warning indicator is visible
- Data is retained for 30 days
- Docker container builds and runs successfully
- New external API appears automatically after first reported call

## Open Items
- Python web framework choice (FastAPI vs Flask vs other) — deferred to implementation
- Dashboard frontend (simple HTML/JS vs a proper UI framework) — deferred to implementation
- Config file location within container (mounted volume vs baked in) — deferred to implementation
