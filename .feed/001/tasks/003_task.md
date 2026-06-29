# Task 003 — UsageRecord data model and in-memory storage engine

## Source
US-1, US-6 from product.md (usage reporting and auto-creation)

## Description
Define the UsageRecord data model and implement the in-memory storage engine with 30-day retention.

## Dependencies
- Task 001 (FastAPI app must exist as the runtime context)

## Details

### UsageRecord data model
A `UsageRecord` has the following fields:
- `id`: auto-generated integer (UUID or auto-increment)
- `service_name`: string — name of the internal service making the call
- `api_id`: string — identifier of the external API (e.g. "stripe", "twilio")
- `timestamp`: datetime — ISO 8601 timestamp sent by the calling service
- `status_code`: integer — HTTP status code returned by the external API
- `rate_limit_headers`: dict or None — optional `X-RateLimit-Limit`, `X-RateLimit-Remaining`, etc.
- `created_at`: datetime — server-side time when the record was stored

### In-memory storage engine
- Use a Python `list` or `dict` in memory to store `UsageRecord` objects
- No database — all data is lost on service restart
- Implement a background cleanup: records older than 30 days are discarded
- Cleanup runs on startup and periodically (e.g. on each new record insertion, or via a timer)

### API registry
- Maintain a `set` of known `api_id` values — auto-created on first POST /usage
- New API entries have no configured rate limit and use default threshold of 80%
- The registry is in-memory only and lost on restart

## Verification
- [ ] A `UsageRecord` can be created and stored in memory
- [ ] Records are retrievable from storage
- [ ] Records older than 30 days are discarded on cleanup
- [ ] A new `api_id` seen for the first time is added to the registry
- [ ] Storage survives multiple record insertions
- [ ] Storage is empty after service restart (by design)
