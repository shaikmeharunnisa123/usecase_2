# Incident Report – Claims Backend Outage

## 1. Incident Summary

| Field | Details |
|---|---|
| Incident | Claims Backend Unavailability |
| Date | 23 November 2026 |
| Start Time | 06:14:47 UTC |
| Environment | Local Docker Compose environment |
| Affected Service | Claims Management Backend |
| Severity | Critical (configured alert severity) |
| Status | Resolved |

This incident was intentionally simulated to demonstrate incident detection, investigation, service recovery, and post-recovery verification.

## 2. Incident Description

A controlled outage was triggered by stopping the Claims Management backend container using Docker Compose.

Command executed:

    docker compose stop backend

The backend API became unavailable, and requests to the health endpoint failed to connect.

The frontend and PostgreSQL containers were not intentionally stopped during this exercise.

## 3. Detection and Monitoring

Prometheus identified that the backend metrics target was unavailable.

PromQL query:

    up{job="claims-backend"}

Observed value during the incident:

    0

The configured alert was:

| Setting | Value |
|---|---|
| Alert Name | ClaimsBackendDown |
| Condition | up{job="claims-backend"} == 0 |
| Evaluation Interval | 5 seconds |
| Pending Duration | 15 seconds |
| Severity | Critical |

The alert successfully transitioned to FIRING.

## 4. Investigation and Findings

The investigation involved checking:

- Backend health endpoint connectivity.
- Prometheus target status.
- Prometheus alert state.
- Backend container logs through Grafana Explore and Loki.

The Loki query used was:

    {service="backend"}

The logs showed the Gunicorn termination sequence:

- Handling signal: term
- Worker exiting
- Shutting down: Master

The error-filter query was:

    {service="backend"} |~ "ERROR|CRITICAL|Traceback"

No matching log volume was returned for the selected query and time range.

### Root Cause

The immediate cause of the outage was the intentional execution of:

    docker compose stop backend

This was a controlled failure simulation. No application defect or unexpected infrastructure failure was established by this exercise.

## 5. Impact

- The backend API was unavailable during the simulation.
- Backend health-check requests could not connect.
- Prometheus reported the backend target as DOWN.
- The configured critical alert entered FIRING.
- Frontend and database containers were not intentionally stopped.

The exercise did not establish any data loss or database failure.

## 6. Recovery Actions

The backend was restored using:

    docker compose start backend

Recovery was verified through the following checks:

1. Backend health endpoint returned HTTP 200.
2. Prometheus backend target returned to UP.
3. The `up{job="claims-backend"}` query returned 1.
4. The `ClaimsBackendDown` alert returned to INACTIVE.
5. Frontend returned HTTP 200.
6. PostgreSQL remained healthy.
7. Prometheus, Loki, and Grafana health checks succeeded.

The exact recovery timestamp was not separately recorded.

## 7. Lessons Learned

- Health endpoints provide a simple way to verify service availability.
- Prometheus target monitoring helps detect backend failures.
- Alert rules provide automated notification of service outages.
- Centralized logs help investigate service shutdown behavior.
- Recovery should always be followed by health and monitoring verification.

## 8. Recommended Preventive Actions

The following are recommendations for production environments:

- Configure appropriate container restart policies.
- Monitor service availability and response times.
- Review alert thresholds and notification channels.
- Maintain documented recovery procedures.
- Regularly test incident response and recovery processes.
- Correlate application logs, metrics, and deployment events during investigations.

## 9. Final Status

The backend was successfully restarted. Application health checks passed, Prometheus reported the target as UP, and the incident alert returned to INACTIVE.

**Incident Status: Resolved**
