# Runbook – Claims Backend Incident Simulation and Recovery

## 1. Purpose

This runbook describes the steps to verify application health, simulate a controlled backend outage, investigate the incident using Prometheus and Loki, restore the backend, and verify recovery.

## 2. Environment

| Component | URL |
|---|---|
| Frontend | http://localhost:8081 |
| Backend Health | http://localhost:5001/api/health |
| Prometheus | http://localhost:9090 |
| Grafana | http://localhost:3000 |
| Loki | http://localhost:3100 |
| Alloy | http://localhost:12345 |

## 3. Prerequisites

- Docker and Docker Compose must be installed.
- Run commands from the project directory containing the Docker Compose configuration.
- Application and monitoring containers must be available.
- Prometheus must be configured to scrape the backend metrics endpoint.
- Grafana must have access to Prometheus and Loki.

**Important:** Perform the failure simulation only in the intended lab environment. Do not execute the stop command against a production backend.

## 4. Step 1 – Verify Initial Application Health

Check the running containers:

    docker compose ps

Check the backend health endpoint:

    curl -i http://localhost:5001/api/health

Expected result:

- HTTP status: 200 OK
- Backend status: healthy

Check the frontend:

    curl -I http://localhost:8081

Expected result: HTTP 200 OK.

## 5. Step 2 – Generate Application Traffic

Generate normal backend health-check traffic:

    for i in {1..30}; do
        curl -s http://localhost:5001/api/health
        echo
        sleep 1
    done

Open Prometheus at:

    http://localhost:9090

Run the following query:

    sum(rate(claims_http_requests_total[5m]))

Observe the application request rate.

## 6. Step 3 – Simulate Backend Failure

Record the incident start time:

    date -u

Stop only the backend container:

    docker compose stop backend

Verify that the backend health endpoint is unavailable:

    curl -i --max-time 5 http://localhost:5001/api/health

Expected result: connection failure because the backend is stopped.

## 7. Step 4 – Investigate the Incident

### 7.1 Check Prometheus Target

Open:

    http://localhost:9090/targets

Find the `claims-backend` target.

Expected result during the incident: DOWN.

### 7.2 Check Backend Availability Metric

Run:

    up{job="claims-backend"}

Expected value during the outage:

    0

### 7.3 Check Alert State

Open the Prometheus Alerts page:

    http://localhost:9090/alerts

Find:

    ClaimsBackendDown

Expected result: FIRING after the configured pending duration.

### 7.4 Investigate Backend Logs

Open Grafana:

    http://localhost:3000

Navigate to Explore and select Loki.

Run:

    {service="backend"}

Review the available backend logs and identify the shutdown sequence.

To search for error-related entries, use:

    {service="backend"} |~ "ERROR|CRITICAL|Traceback"

Note: A query returning no matching log volume means no matching entries were returned for the selected time range. It does not prove that errors never occurred.

## 8. Step 5 – Recover the Backend

Start the backend container:

    docker compose start backend

Check the backend health endpoint:

    curl -i http://localhost:5001/api/health

Expected result:

- HTTP status: 200 OK
- Backend status: healthy

## 9. Step 6 – Verify Monitoring Recovery

Check the Prometheus target page:

    http://localhost:9090/targets

Expected result: `claims-backend` is UP.

Run the metric query:

    up{job="claims-backend"}

Expected value:

    1

Open:

    http://localhost:9090/alerts

Expected result: `ClaimsBackendDown` returns to INACTIVE after the backend is healthy and the alert rule is reevaluated.

## 10. Step 7 – Perform Final Platform Checks

Check all application containers:

    docker compose ps

Check the frontend:

    curl -I http://localhost:8081

Check Prometheus:

    curl -s http://localhost:9090/-/healthy

Check Loki:

    curl -s http://localhost:3100/ready

Check Grafana:

    curl -s http://localhost:3000/api/health

Expected results:

| Component | Expected Result |
|---|---|
| Backend | HTTP 200, healthy |
| Frontend | HTTP 200 |
| PostgreSQL | Healthy |
| Prometheus | Healthy |
| Loki | Ready |
| Grafana | Database OK |

## 11. Troubleshooting Notes

### Backend does not start

Inspect backend logs:

    docker compose logs --tail=100 backend

Check container status:

    docker compose ps

### Prometheus target remains DOWN

- Verify that the backend is running.
- Check the backend `/metrics` endpoint.
- Inspect Prometheus target health and scrape errors.
- Verify the configured scrape target and network connectivity.

### Alert remains FIRING

- Confirm the backend target is UP.
- Re-run the `up{job="claims-backend"}` query.
- Allow time for the next rule evaluation.
- Inspect the alert expression and pending duration.

### Logs are unavailable in Loki

- Confirm that the Loki service is ready.
- Verify Alloy is running and collecting container logs.
- Check the selected time range and log labels in Grafana Explore.

## 12. Completion Criteria

The incident exercise is complete when:

- Backend health endpoint returns HTTP 200.
- Frontend is accessible.
- PostgreSQL remains healthy.
- Prometheus backend target is UP.
- `ClaimsBackendDown` alert is INACTIVE.
- Loki and Grafana health checks succeed.

## 13. Safety Reminder

This runbook is intended for controlled incident-response practice. Always confirm the target environment and service name before stopping or restarting containers.
