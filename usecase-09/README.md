# Use Case 09 – Simulate an Incident and Recover

## 1. Overview

This use case demonstrates a controlled backend outage, incident detection, investigation, recovery, and post-recovery verification using the Claims Management application and its monitoring stack.

The objective is to understand how DevOps and SRE engineers detect service failures, investigate metrics and logs, restore services, and verify system health.

## 2. Technology Stack

- Docker and Docker Compose
- React frontend served through Nginx
- Flask/Gunicorn backend
- PostgreSQL database
- Prometheus for metrics collection and alerting
- Grafana for monitoring and visualization
- Loki for log aggregation
- Grafana Alloy for log collection

## 3. Application Architecture

The application and monitoring flow is:

- User accesses the frontend through port 8081.
- Frontend communicates with the backend through port 5001.
- Backend connects to PostgreSQL.
- Prometheus scrapes backend metrics through the /metrics endpoint.
- Prometheus evaluates the ClaimsBackendDown alert.
- Grafana displays monitoring dashboards and alert information.
- Grafana Alloy collects container logs and forwards them to Loki.
- Grafana Explore is used to investigate backend logs.

## 4. Services and Ports

| Component | Port |
|---|---:|
| Frontend | 8081 |
| Backend | 5001 |
| Prometheus | 9090 |
| Grafana | 3000 |
| Loki | 3100 |
| Grafana Alloy | 12345 |

PostgreSQL is available to the application through the Docker Compose network.

## 5. Incident Simulation

### 5.1 Initial Health Verification

The backend was initially verified as healthy:

    curl -i http://localhost:5001/api/health

The API returned HTTP 200 with the following response:

    {"service":"claims-backend","status":"healthy"}

### 5.2 Generate Normal Traffic

Thirty health-check requests were generated to create normal application traffic:

    for i in {1..30}; do
        curl -s http://localhost:5001/api/health
        echo
        sleep 1
    done

Prometheus was used to observe the request rate with this query:

    sum(rate(claims_http_requests_total[5m]))

### 5.3 Trigger the Backend Failure

The incident simulation started at:

- Date: 23 November 2026
- Start time: 06:14:47 UTC
- Action: Stop only the backend container.

The following command was executed:

    docker compose stop backend

The backend health-check request then failed because the backend was unavailable:

    curl -i --max-time 5 http://localhost:5001/api/health

### 5.4 Incident Detection

Prometheus detected that the backend target was DOWN.

The following PromQL query was used:

    up{job="claims-backend"}

The query returned a value of 0 during the outage.

The configured alert was:

| Setting | Value |
|---|---|
| Alert name | ClaimsBackendDown |
| Expression | up{job="claims-backend"} == 0 |
| Evaluation interval | 5 seconds |
| Pending duration | 15 seconds |
| Severity | Critical |

The alert transitioned to FIRING during the incident.

### 5.5 Log Investigation

Grafana Explore was used to inspect backend logs in Loki.

The query used was:

    {service="backend"}

The logs showed the Gunicorn shutdown sequence, including:

- Handling signal: term
- Worker exiting
- Shutting down: Master

The following query was also used to search for application errors:

    {service="backend"} |~ "ERROR|CRITICAL|Traceback"

No matching log volume was returned for this query during the selected time range. The incident was caused by a controlled container stop rather than a demonstrated application exception.

## 6. Recovery and Verification

### 6.1 Restore the Backend

The backend was restarted using:

    docker compose start backend

The health endpoint was checked again:

    curl -i http://localhost:5001/api/health

The backend returned HTTP 200 with a healthy status.

### 6.2 Verify Monitoring Recovery

After recovery:

- Prometheus backend target returned to UP.
- The `up{job="claims-backend"}` query returned 1.
- The `ClaimsBackendDown` alert returned to INACTIVE.

### 6.3 Final Platform Verification

The following checks were performed:

| Component | Verification | Result |
|---|---|---|
| Backend | `/api/health` | HTTP 200, healthy |
| Frontend | Port 8081 | HTTP 200 |
| PostgreSQL | Docker Compose status | Healthy |
| Prometheus | `/-/healthy` | Healthy |
| Loki | `/ready` | Ready |
| Grafana | `/api/health` | Database OK |

The application and monitoring services were operational after recovery.

