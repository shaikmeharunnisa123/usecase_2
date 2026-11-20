# Use Case 08 – Observe, Alert, and Investigate the Application

## 1. Objective

Implement an observability solution for the Claims Management application to monitor application availability, request metrics, response time, server errors, and application logs.

The solution uses Prometheus, Grafana, Loki, and Grafana Alloy.

## 2. Architecture

The observability architecture contains the following components:

- **React Frontend:** Serves the application on port 8081.
- **Flask Backend:** Exposes application APIs and Prometheus metrics on port 5001.
- **PostgreSQL:** Stores application data.
- **Prometheus:** Collects backend metrics every 5 seconds.
- **Grafana:** Displays metrics and logs through dashboards and Explore.
- **Grafana Alloy:** Collects Docker container logs and forwards them to Loki.
- **Loki:** Stores and makes application logs searchable.

### Data Flow

**Application Metrics:**

Claims Backend → Prometheus → Grafana

**Application Logs:**

Docker Containers → Grafana Alloy → Loki → Grafana Explore

**Alerting:**

Prometheus → ClaimsBackendDown Alert

## 3. Technology Stack

| Technology | Purpose |
|---|---|
| Flask | Backend application |
| Prometheus | Metrics collection and alerting |
| Grafana | Dashboards and visualization |
| Loki | Log storage and querying |
| Grafana Alloy | Docker log collection |
| Docker Compose | Container orchestration |
| PostgreSQL | Application database |

## 4. Application Instrumentation

Added the `prometheus-client` Python dependency to the backend.

Implemented two custom Prometheus metrics:

- `claims_http_requests_total` – Counts HTTP requests by method, endpoint, and status code.
- `claims_http_request_duration_seconds` – Measures HTTP request duration.

The backend exposes metrics through:

`http://localhost:5001/metrics`

The existing health endpoint is:

`http://localhost:5001/api/health`

Updated the backend Dockerfile to run Gunicorn on port 5001 and direct access and error logs to standard output and error so Docker and Alloy can collect them.

Updated the frontend Nginx configuration to proxy API requests to the backend on port 5001.

## 5. Prometheus Configuration

Created the Prometheus configuration at:

`usecase-08/prometheus/prometheus.yml`

Configured:

- Scrape interval: 5 seconds.
- Evaluation interval: 5 seconds.
- Prometheus self-monitoring.
- Claims backend metrics collection from `backend:5001/metrics`.

Created the alert rule at:

`usecase-08/prometheus/alert-rules.yml`

### ClaimsBackendDown Alert

The alert checks whether the Claims backend target is unavailable.

Expression:

`up{job="claims-backend"} == 0`

The alert fires when the condition remains true for 15 seconds and uses the critical severity label.

## 6. Grafana Dashboard

Configured a Prometheus datasource and provisioned the Claims Overview dashboard.

The dashboard includes:

- Backend Status
- Request Rate
- Server Error Rate
- Average Response Time

Grafana URL:

`http://localhost:3000`

Local lab credentials:

- Username: `admin`
- Password: `admin`

## 7. Log Collection and Investigation

Configured Grafana Alloy to discover Docker containers and collect logs belonging to the `usecase2` Docker Compose project.

Alloy forwards the collected logs to Loki.

Created the Loki configuration at:

`usecase-08/loki/loki-config.yml`

Configured Loki as a Grafana datasource.

Example LogQL query used in Grafana Explore:

`{service="backend"}`

This query retrieves backend container logs for investigation.

## 8. Docker Compose Deployment

Created a separate observability Compose file:

`usecase-08/docker-compose.observability.yml`

The file defines four services:

- Prometheus
- Grafana
- Loki
- Alloy

The observability services connect to the existing `usecase2_network` Docker network to communicate with the application containers.

The application and observability stacks can be managed separately.

Start the observability stack:

```bash
docker compose -f usecase-08/docker-compose.observability.yml up -d
```

Check the observability containers:

```bash
docker compose -f usecase-08/docker-compose.observability.yml ps
```

9. Application and Observability URLs

Component

URL

Frontend

http://localhost:8081

Backend Health

http://localhost:5001/api/health

Backend Metrics

http://localhost:5001/metrics

Prometheus

http://localhost:9090

Grafana

http://localhost:3000

Loki Readiness

http://localhost:3100/ready

Alloy

http://localhost:12345

10. Validation and Testing

The following validations were performed:

Verified that the backend health endpoint returns HTTP 200.

Verified that the backend exposes custom Prometheus metrics.

Verified that the frontend successfully proxies API requests to the backend.

Verified that Prometheus successfully scrapes the Claims backend.

Verified that Grafana successfully queries Prometheus.

Verified that the Claims Overview dashboard displays application metrics.

Verified that Loki is ready and accepts log data.

Verified that Alloy collects and forwards Docker logs.

Verified that backend logs can be queried through Grafana Explore.

Verified that the Prometheus alert fires during backend failure and becomes inactive after recovery.

11. Incident Investigation – Backend Failure

Performed a controlled backend failure test.

Detection

Stopped the backend container:

docker compose stop backend

Prometheus detected that the backend target was unavailable.

The ClaimsBackendDown alert changed to FIRING after the configured 15-second duration.

Recovery

Started the backend container:

docker compose start backend

Verified backend health:

curl -i http://localhost:5001/api/health

The health endpoint returned HTTP 200.

After Prometheus successfully scraped the backend again, the alert returned to INACTIVE.

Outcome

Successfully demonstrated backend failure detection, alert state transition, service recovery, and alert resolution.

12. Directory Structure

usecase-08/
├── alloy/
│   └── config.alloy
├── evidence/
├── grafana/
│   ├── dashboards/
│   │   └── claims-dashboard.json
│   └── provisioning/
│       ├── dashboards/
│       │   └── dashboards.yml
│       └── datasources/
│           └── datasources.yml
├── loki/
│   └── loki-config.yml
├── prometheus/
│   ├── alert-rules.yml
│   └── prometheus.yml
├── docker-compose.observability.yml
└── README.md

13. Final Outcome

Successfully implemented application observability for the Claims Management application.

The solution provides:

Application metrics collection.

Grafana dashboards for monitoring.

Centralized Docker log collection.

Log investigation using LogQL.

Backend availability monitoring and alerting.

Controlled failure detection and recovery validation.

The backend outage and recovery scenarios were tested successfully using Prometheus alerting.