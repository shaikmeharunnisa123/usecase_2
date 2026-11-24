# Root Cause Analysis – Use Case 10
## Jenkins Agent Failure and Recovery

### 1. Incident Summary

A controlled Jenkins agent outage was simulated by marking the Built-In Node temporarily offline.

A pipeline was triggered during the outage. Jenkins accepted the build request but could not execute it because no available executor was present.

### 2. Symptoms

The pipeline remained in the Build Queue.

The Jenkins console displayed:

Still waiting to schedule task
Waiting for next available executor

### 3. Root Cause

The primary cause was the deliberate offline state of the Jenkins Built-In Node.

Since no other executor was available, Jenkins could not schedule the pipeline.

### 4. Investigation

- Inspected the Jenkins Build Queue.
- Reviewed the pipeline console output.
- Checked the Built-In Node status.
- Confirmed that the node was offline.
- Verified that the pipeline was waiting for an executor.

### 5. Recovery

The Built-In Node was brought back online through the Jenkins Nodes interface.

Jenkins automatically resumed queued Build #3 after the executor became available.

The pipeline completed successfully.

### 6. Additional Infrastructure Verification

- Docker CLI was available at `/usr/bin/docker`.
- Jenkins PATH already included `/usr/bin`.
- Docker daemon connectivity was successful.
- Harbor returned HTTP 401, confirming registry reachability.
- Harbor authentication using `harbor-credentials` succeeded.

No Docker PATH correction was necessary in this environment.

### 7. Additional Configuration Issue

During initial pipeline configuration, Docker attempted to access Harbor over HTTPS instead of HTTP.

The Docker Desktop insecure-registry configuration and registry address were corrected to use `host.docker.internal:80`.

The subsequent Harbor credential check passed.

This was a separate registry configuration issue, not the root cause of the controlled agent outage.

### 8. Validation

Build #2 established a successful baseline.

Build #3 was queued during the controlled outage and automatically resumed after the Built-In Node was restored.

Build #3 completed with:

Finished: SUCCESS

All five pipeline stages passed.

No application source-code changes were required.

### 9. Preventive Measures

- Monitor Jenkins node availability.
- Monitor executor capacity and Build Queue activity.
- Verify Docker CLI and daemon availability.
- Maintain sufficient disk space.
- Monitor Harbor registry availability.
- Store registry credentials securely in Jenkins.
- Maintain a dedicated CI infrastructure health-check pipeline.
- Document infrastructure incidents and recovery actions.

### 10. Conclusion

The primary incident was caused by the Built-In Node being offline, preventing Jenkins from allocating an executor.

Restoring the node allowed the queued pipeline to resume automatically and complete successfully.

This exercise demonstrated the importance of Jenkins agent availability and infrastructure health checks.
