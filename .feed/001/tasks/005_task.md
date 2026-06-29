# Task 005 — Dashboard with current usage %, time series, and warning indicator

## Source
US-2, US-3, US-7 from product.md (dashboard views and threshold warnings)

## Description
Implement the dashboard HTML page showing per-API usage percentage and time series charts, with visual warning indicators when thresholds are exceeded.

## Dependencies
- Task 001 (FastAPI app)
- Task 002 (config loading — needed to know rate limits and thresholds)
- Task 003 (storage engine — needed to query records for current % and time series)

## Details

### Endpoint
- **Endpoint**: `GET /dashboard`
- **Response**: HTML page (Content-Type: text/html)

### Page content
The dashboard HTML page must include:

1. **Header**: Title "Rate Limit Tracker"

2. **Per-API current usage cards**:
   - For each known API (from the registry), display a card
   - If the API has a configured rate limit:
     - Calculate usage % for the current window (e.g. current minute for `req/min`, current hour for `req/hour`)
     - Display as a progress bar (visual fill representing %)
     - Show numeric % and absolute counts (e.g. "750 / 1000 req/min")
   - If the API has no configured rate limit: show "Not configured"
   - If usage % > threshold: show a visual warning indicator (red/orange color or warning icon)
   - If usage % ≤ threshold: normal (green or neutral)

3. **Time series chart per API**:
   - For each API with data, show a line/area chart
   - X-axis: time (last 30 days or available data)
   - Y-axis: usage % (0–100%)
   - Use a JavaScript charting library (e.g. Chart.js via CDN, or a simple SVG/canvas implementation)
   - Implementation choice is deferred — use what is simplest and works in a single HTML file

4. **Refresh**: Page refreshes on reload (no auto-refresh required)

### Data calculation
- **Current usage %**: Count usage records for the API within the most recent complete time window matching the rate limit granularity, divided by the configured limit
  - e.g. for `stripe: 1000 req/min` — count stripe calls in the current minute, compare to 1000
  - For `req/hour` — count in the current hour; for `req/day` — count in the current day
- **Time series data points**: Aggregate usage at the appropriate granularity (e.g. one data point per minute for `req/min`, per hour for `req/hour`)
- **Threshold check**: Compare current usage % to the configured threshold for that API (default 80%)

### Dashboard data endpoint
To keep the HTML simple, it is acceptable to embed the data directly in the HTML page (server-rendered) or to serve it via a separate internal API endpoint (e.g. `GET /api/dashboard-data`) that the JavaScript calls. Either approach is acceptable.

## Verification
- [ ] GET /dashboard returns 200 with Content-Type text/html
- [ ] Page shows an API card for each known API
- [ ] An API with a configured rate limit shows a usage % with a progress bar
- [ ] An API without a configured rate limit shows "Not configured"
- [ ] When usage % > threshold, a warning indicator is visible (color/icon)
- [ ] Page shows a time series chart for each API
- [ ] The time series covers available data up to 30 days
- [ ] Page reload reflects current data
