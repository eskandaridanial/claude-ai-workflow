# Task 002 — YAML configuration loading

## Source
US-4, US-5 from product.md (rate limits and thresholds config)

## Description
Implement loading and validation of the two YAML config files: `config/rate_limits.yaml` and `config/thresholds.yaml`. Both files are read on application startup.

## Dependencies
None (this task does not depend on Task 001; it defines config file schemas and loading functions independently).

## Details
- **`config/rate_limits.yaml`** — format:
  ```yaml
  stripe: "1000 req/min"
  twilio: "500 req/min"
  ```
  Each value is a string of the form `<number> <unit>` where unit is `req/sec`, `req/min`, `req/hour`, or `req/day`.

- **`config/thresholds.yaml`** — format:
  ```yaml
  stripe: 80
  twilio: 90
  ```
  Each value is an integer percentage (0–100). If an API is absent, default threshold is 80.

- The service must load both files on startup. If either file is missing or malformed, the service must fail to start with a clear error message naming the file and the issue.

- Create a Python module (e.g. `config.py`) that exposes:
  - `rate_limits: dict[str, RateLimit]` — keyed by API name
  - `thresholds: dict[str, int]` — keyed by API name, default 80 for missing keys
  - `RateLimit` is a simple value object with `.limit` (int) and `.window_seconds` (int) fields

- `config/rate_limits.yaml` and `config/thresholds.yaml` must exist as placeholder files with example content

## Verification
- [ ] Loading `config/rate_limits.yaml` with valid content succeeds and returns correctly parsed `RateLimit` objects
- [ ] Loading `config/thresholds.yaml` with valid content succeeds and returns correct integer values
- [ ] Missing config file causes a startup error with a descriptive message
- [ ] Malformed YAML causes a startup error with a descriptive message
- [ ] Threshold defaults to 80 for APIs not present in `thresholds.yaml`
