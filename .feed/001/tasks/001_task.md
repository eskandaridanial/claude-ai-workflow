# Task 001 — Project scaffold and FastAPI application

## Source
US-1, US-2, US-3, US-8 from product.md (all stories require a running service)

## Description
Set up the Python project directory structure and create a minimal FastAPI application that runs and responds to a health check.

## Dependencies
None.

## Details
- Create the following directory structure:
  ```
  /app
    main.py        # FastAPI application entry point
  config/
    rate_limits.yaml
    thresholds.yaml
  Dockerfile
  docker-compose.yml
  requirements.txt
  ```
- Use **FastAPI** as the Python web framework (deferred choice from product.md; FastAPI is the sensible modern default for a Python REST API)
- `main.py` must:
  - Create a FastAPI app instance
  - Define a `GET /health` endpoint returning `{"status": "ok"}`
  - Run with `uvicorn` on port 8080
- `requirements.txt` must include: `fastapi`, `uvicorn`, `pyyaml`, `pydantic`
- The config directory and config files are created as empty placeholders (actual loading is Task 002)

## Verification
- [ ] `python -m uvicorn main:app --port 8080` starts without errors
- [ ] `GET /health` returns `200` with `{"status": "ok"}`
- [ ] Directory structure matches the layout above
- [ ] `requirements.txt` includes all listed packages
