# Product — 001

## Goal
A new Python service that tracks API rate limit usage across internal microservices. Services push call data via REST; a dashboard shows per-API usage percentage and time series, with warnings visible when thresholds are breached.

## Scope
### In scope
- REST endpoint for services to report external API calls
- In-memory storage with 30-day retention
- YAML config file for rate limits per external API
- YAML config file for warning thresholds per external API (default 80%)
- Auto-creation of external API entries on first reported call
- Dashboard showing per-API current usage % (progress bar) and time series
- Threshold breach warning indicator on dashboard
- Docker containerization

### Out of scope
- Blocking or throttling actual API calls
- Enforcing rate limits on behalf of services
- Real-time notifications (Slack, email, webhooks)
- User authentication for the tracking service or dashboard
- Horizontal scaling / distributed storage
- Admin API for rate limit configuration
- Production-grade identity/access management

## User Stories

### US-1: Service reports an external API call
**As a** service
**I want** to report each external API call I make to a central tracking service
**So that** the organization has visibility into how close each external API is to its rate limit

**Acceptance Criteria:**
- [ ] POST /usage accepts a JSON body with: serviceName, apiId, timestamp, statusCode, and optionally rateLimitHeaders
- [ ] A call with a missing required field returns 400 with a descriptive error
- [ ] A successful POST returns 201 and stores the usage record in memory
- [ ] The API ID is a free-form string (e.g. "stripe", "twilio")
- [ ] Multiple calls from multiple services are all recorded correctly

**Notes:** Timestamp is sent by the calling service (ISO 8601 string). The tracking service does not validate the timestamp beyond format checking.

---

### US-2: Service views current rate limit usage per API
**As a** service owner
**I want** to see how close each external API is to its configured rate limit right now
**So that** I know whether my service is at risk of being throttled

**Acceptance Criteria:**
- [ ] GET /dashboard returns an HTML page
- [ ] The page shows a list of all external APIs that have been reported against
- [ ] For each API with a configured rate limit, a usage % is shown (based on the current / most recent complete sliding window)
- [ ] The usage % is displayed as a progress bar or similar visual indicator
- [ ] For APIs without a configured rate limit, the page shows "not configured" instead of a %
- [ ] The list refreshes on reload

**Notes:** The exact window for "current usage" — e.g. last 60 minutes for per-minute limits — should be derived from the rate limit config (see US-4).

---

### US-3: Service views historical usage time series per API
**As a** service owner
**I want** to see how rate limit usage for each API has changed over time
**So that** I can spot trends before hitting the limit

**Acceptance Criteria:**
- [ ] The dashboard page includes a time series chart per API
- [ ] The chart covers the last 30 days (or available data, whichever is shorter)
- [ ] Y-axis represents usage % of the configured rate limit
- [ ] X-axis represents time
- [ ] Multiple APIs can be shown on the same chart or on separate charts — implementation choice

**Notes:** Data is stored in memory; historical view is only as complete as the data retained (lost on service restart per design decision).

---

### US-4: Operator configures rate limits per external API
**As a** operator
**I want** to set the rate limit for each external API in a YAML config file
**So that** the dashboard shows meaningful usage % instead of "not configured"

**Acceptance Criteria:**
- [ ] A file `config/rate_limits.yaml` defines mappings of API name → rate limit (e.g. `stripe: 1000` meaning 1000 requests per minute)
- [ ] The format supports a time window suffix (e.g. `req/min`, `req/sec`, `req/hour`)
- [ ] The service reads the config file on startup and on restart
- [ ] If an API has a rate limit configured but no calls have been reported yet, it does not appear on the dashboard
- [ ] If the config file is malformed, the service fails to start with a clear error

**Notes:** Config file location within the container is an open item (see Open Items).

---

### US-5: Operator configures warning thresholds per external API
**As a** operator
**I want** to set the warning threshold for each external API (e.g. warn at 80% vs 90%)
**So that** teams get appropriate lead time before hitting the limit

**Acceptance Criteria:**
- [ ] A file `config/thresholds.yaml` defines mappings of API name → threshold percentage (e.g. `stripe: 80`)
- [ ] If an API is not in the thresholds file, the default threshold of 80% is used
- [ ] The threshold config is read on startup and on restart
- [ ] If the config file is malformed, the service fails to start with a clear error

**Notes:** If US-4 and US-5 use separate config files, two files are required. An operator configuring only rate limits (without thresholds) should still work with the default 80%.

---

### US-6: System auto-creates external API entries on first reported call
**As a** system
**I want** to accept usage reports for any API name without pre-registration
**So that** services can start reporting immediately without an onboarding step

**Acceptance Criteria:**
- [ ] If a POST /usage arrives with apiId "new-api" and "new-api" has no prior config, an entry for "new-api" is created automatically
- [ ] The auto-created entry has no configured rate limit and uses the default 80% threshold
- [ ] The API appears on the dashboard as "not configured" until a rate limit is added to the config file

---

### US-7: Dashboard shows warning when usage exceeds threshold
**As a** service owner
**I want** to see a visual warning on the dashboard when an API's usage exceeds its configured threshold
**So that** I can see at a glance which APIs are at risk

**Acceptance Criteria:**
- [ ] When an API's current usage % exceeds its threshold %, the dashboard displays a warning indicator (e.g. red color, warning icon, or highlighted row)
- [ ] The warning is visible on the per-API card and/or on the time series chart
- [ ] The warning is informational only — it does not send any notification and does not block or throttle calls
- [ ] When usage drops back below the threshold, the warning indicator clears

---

### US-8: Developer runs the service in Docker
**As a** developer
**I want** to build and run the service as a Docker container
**So that** it runs consistently in local development and production

**Acceptance Criteria:**
- [ ] A Dockerfile exists at the project root
- [ ] `docker build` produces a container image with no errors
- [ ] `docker run` starts the service with the dashboard accessible on a defined port (e.g. 8080)
- [ ] A docker-compose.yml exists for local development that starts the container
- [ ] Config files are accessible to the running container (either mounted as volumes or baked in — open item)

---

## Non-Functional Requirements
- **Storage**: In-memory only. Data is lost on service restart. Acceptable for this scale and use case.
- **Scale**: Supports <5 reporting services and <10K reported calls/day on a single instance.
- **Availability**: Not a critical-path service. Acceptable downtime is fine; no HA requirements.
- **Auth**: No authentication on any endpoint. Assumes network-level access control restricts access to internal services only.
- **Retention**: Data older than 30 days is discarded from memory.

## Assumptions
- The calling service is responsible for sending the correct timestamp and API ID in the POST body — the tracking service trusts this data.
- "Current usage %" is calculated based on the most recent complete time window matching the rate limit granularity (e.g. for `1000 req/min`, usage is counted for the current minute).
- Time series granularity is determined at implementation time (e.g. data points every minute or every 5 minutes).
- Python web framework choice (FastAPI vs Flask) is deferred to implementation.
- Dashboard frontend implementation (plain HTML/JS vs React/Vue) is deferred to implementation.
- Config file location in the Docker container (mounted volume vs baked into image) is deferred to implementation.

## Out of Scope / Deferred
- Python web framework choice — deferred to implementation
- Dashboard frontend choice (plain HTML/JS vs framework) — deferred to implementation
- Config file location within container — deferred to implementation
