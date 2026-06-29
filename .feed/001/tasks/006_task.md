# Task 006 — Dockerfile and docker-compose.yml

## Source
US-8 from product.md (Docker deployment)

## Description
Create a Dockerfile and docker-compose.yml so the service can be built and run inside a container.

## Dependencies
- Task 001 (project scaffold with requirements.txt must exist)

## Details

### Dockerfile
- Based on `python:3.11-slim` or similar
- Working directory: `/app`
- Copy `requirements.txt` and run `pip install -r requirements.txt`
- Copy application code into `/app`
- Expose port `8080`
- Use `uvicorn main:app --host 0.0.0.0 --port 8080` as the entry point
- Config files (`config/rate_limits.yaml`, `config/thresholds.yaml`) should be mounted as volumes (see docker-compose.yml), not baked into the image

### docker-compose.yml
- Service name: `rate-limit-tracker`
- Build context: `.` (the project root)
- Ports: `8080:8080`
- Volumes: mount `./config` to `/app/config` for runtime config access
- Command: `uvicorn main:app --host 0.0.0.0 --port 8080`
- No external services needed (no database)

### Project root files
- `Dockerfile` at project root
- `docker-compose.yml` at project root

## Verification
- [ ] `docker build .` completes without errors
- [ ] `docker run -p 8080:8080 $(docker build -q .)` starts the service and the dashboard is accessible on port 8080
- [ ] `docker-compose up` starts the service with config files mounted
- [ ] Config file changes on the host are reflected in the running container without rebuild (volume mount)
