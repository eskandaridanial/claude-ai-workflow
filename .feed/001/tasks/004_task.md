# Task 004 — POST /usage endpoint

## Source
US-1, US-6 from product.md (reporting calls, auto-creation)

## Description
Implement the REST endpoint that receives usage records from internal services and stores them.

## Dependencies
- Task 001 (FastAPI app must exist)
- Task 003 (storage engine and UsageRecord model)

## Details
- **Endpoint**: `POST /usage`
- **Request body** (JSON):
  ```json
  {
    "serviceName": "payment-service",
    "apiId": "stripe",
    "timestamp": "2026-06-29T10:00:00Z",
    "statusCode": 200,
    "rateLimitHeaders": {
      "X-RateLimit-Limit": "1000",
      "X-RateLimit-Remaining": "950"
    }
  }
  ```
  - `serviceName` (string, required)
  - `apiId` (string, required)
  - `timestamp` (ISO 8601 string, required) — validated as ISO format only, not checked for reasonableness
  - `statusCode` (integer, required)
  - `rateLimitHeaders` (object, optional) — keys and values are strings

- **Response on success**: `201 Created` with the created record (including server-generated `id` and `createdAt`)

- **Response on missing required field**: `400 Bad Request` with `{"error": "MISSING_FIELD", "field": "<fieldname>"}`

- **Behavior**:
  1. Parse and validate request body
  2. Create a `UsageRecord` with server-side `id` and `createdAt`
  3. Store in memory (Task 003 storage engine)
  4. If `apiId` is new, auto-create entry in API registry (handled by Task 003)
  5. Return 201 with the record

## Verification
- [ ] POST /usage with all required fields returns 201 and the record is stored
- [ ] POST /usage with a missing `serviceName` returns 400 with `{"error": "MISSING_FIELD", "field": "serviceName"}`
- [ ] POST /usage with a missing `apiId` returns 400 with `{"error": "MISSING_FIELD", "field": "apiId"}`
- [ ] POST /usage with invalid timestamp format returns 400
- [ ] A new `apiId` (not previously seen) auto-creates an entry in the API registry
- [ ] Records from multiple services are all stored correctly
