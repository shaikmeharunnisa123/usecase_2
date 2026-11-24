# Use Case 10 – Troubleshoot Jenkins Agent Failure and Recovery

## 1. Overview

This use case demonstrates how to identify, troubleshoot, and recover from a Jenkins agent failure.

The Built-In Node is deliberately taken offline to simulate an infrastructure failure. A pipeline is triggered during the outage and remains in the Build Queue because no executor is available.

After restoring the node, Jenkins automatically resumes the queued pipeline. Docker, disk space, Harbor connectivity, and Harbor credentials are verified to confirm that the CI infrastructure is operational.

The application source code is not modified as part of this exercise.

## 2. Objectives

- Understand Jenkins Built-In Node and executor availability.
- Simulate a controlled Jenkins agent outage.
- Identify why a pipeline remains in the Build Queue.
- Restore the Jenkins agent.
- Verify Docker CLI and Docker daemon connectivity.
- Verify disk availability.
- Test Harbor registry connectivity.
- Validate Harbor credentials stored in Jenkins.
- Confirm successful pipeline recovery without application-code changes.

## 3. Technology Stack

- Jenkins
- Docker and Docker Desktop
- Docker Compose
- WSL Ubuntu
- Harbor Container Registry
- Git and GitHub

## 4. Environment and Architecture

Jenkins is running inside a Docker container with persistent Jenkins home storage.

The Jenkins container communicates with the Docker Desktop daemon through the mounted Docker socket.

Harbor is running separately using Docker Compose.

### Architecture Flow

Developer / Jenkins User
        |
        v
Jenkins Controller
        |
        v
Built-In Node and Executor
        |
        +---- Agent Check
        |
        +---- Docker CLI
        |       |
        |       v
        |   Docker Desktop Daemon
        |
        +---- Disk Check
        |
        +---- Harbor Connectivity
        |       |
        |       v
        |   Harbor Registry
        |
        +---- Harbor Credential Check
                |
                v
           Docker Login

## 5. Jenkins Configuration

Jenkins URL:

http://localhost:8080

Jenkins container:

usecase10-jenkins

Persistent Jenkins volume:

usecase10-jenkins-home

Dedicated pipeline job:

usecase-10-agent-recovery

The Jenkins container uses a custom image with Docker CLI and curl installed.

The Docker socket is mounted into the Jenkins container to allow Jenkins to communicate with the Docker daemon.

## 6. Harbor Configuration

Harbor is deployed using Docker Compose.

Harbor installation directory:

/home/mehar/harbor-setup/harbor

Harbor registry endpoint:

http://localhost/v2/

From the Jenkins container, the registry is accessed using:

http://host.docker.internal/v2/

The registry returns HTTP 401 Unauthorized when accessed without authentication. This is expected and confirms that the registry endpoint is reachable and requires authentication.

Docker Desktop was configured to recognize the HTTP registry endpoint as an insecure registry:

host.docker.internal:80

The Jenkins pipeline uses the registry address:

host.docker.internal:80

A Harbor robot account was created and its credentials were stored in Jenkins using the credential ID:

harbor-credentials

The credential is used through Jenkins' withCredentials mechanism. Secret values are not stored in the repository or printed in the documentation.

## 7. Dedicated Recovery Pipeline

A separate Jenkins pipeline named usecase-10-agent-recovery was created to isolate CI infrastructure checks from the application pipeline.

The pipeline contains five stages.

### Stage 1 – Agent Check

- Displays the Jenkins node name.
- Displays the workspace location.
- Verifies the execution user.
- Displays the hostname.

### Stage 2 – Docker Check

- Verifies Docker CLI availability.
- Displays the Docker CLI version.
- Checks Docker daemon connectivity.
- Displays the Docker server version.

### Stage 3 – Disk Check

- Checks available disk space.
- Verifies the root filesystem and Jenkins workspace.

### Stage 4 – Harbor Connectivity

- Sends an HTTP request to the Harbor registry endpoint.
- Checks the HTTP response.
- Accepts HTTP 401 as an expected unauthenticated registry response.

### Stage 5 – Harbor Credential Check

- Retrieves the Harbor username and password from Jenkins credentials.
- Authenticates Docker with Harbor using password-stdin.
- Logs out after authentication.

## 8. Baseline Pipeline Validation

The first successful baseline was verified using Build #2.

All five pipeline stages completed successfully:

- Agent Check
- Docker Check
- Disk Check
- Harbor Connectivity
- Harbor Credential Check

During initial configuration, Docker attempted to access Harbor over HTTPS instead of HTTP. The registry endpoint and Docker Desktop insecure-registry configuration were corrected to use host.docker.internal:80.

After this correction, Harbor authentication succeeded.

## 9. Controlled Jenkins Agent Failure

The Jenkins Built-In Node was deliberately marked temporarily offline.

The offline reason was:

Use Case 10 controlled agent outage

A new pipeline execution was triggered while the node was offline.

### Observed Result

Build #3 remained waiting for an available executor.

The Jenkins console displayed:

Still waiting to schedule task
Waiting for next available executor

This confirmed that Jenkins could not allocate an executor while the only execution node was offline.

## 10. Agent Recovery

The Built-In Node was brought back online through the Jenkins Nodes interface.

The queued Build #3 automatically started after the executor became available.

The pipeline completed successfully with:

Finished: SUCCESS

No application source-code changes were required.

## 11. Docker PATH Verification

The reference guide includes a Docker CLI PATH troubleshooting scenario.

In this environment, the Docker CLI was already available at:

/usr/bin/docker

The Jenkins PATH already included:

/usr/bin

The following command was used to verify the configuration:

docker exec usecase10-jenkins sh -c 'echo "Docker location:"; command -v docker; echo "Jenkins PATH:"; echo "$PATH"'

The output confirmed that Docker was available in the Jenkins environment.

Therefore, no Docker PATH correction was necessary in this implementation.

## 12. Final Validation

The following checks were completed:

| Validation | Result |
|---|---|
| Jenkins agent availability | Passed |
| Docker CLI verification | Passed |
| Docker daemon connectivity | Passed |
| Disk space verification | Passed |
| Harbor registry connectivity | Passed |
| Harbor credential authentication | Passed |
| Controlled agent outage | Verified |
| Build Queue waiting behavior | Verified |
| Agent recovery | Passed |
| Automatic queued build resumption | Passed |
| Docker PATH verification | Passed |

## 13. Conclusion

This use case demonstrated how a Jenkins infrastructure failure can prevent a pipeline from executing even when the application itself has no issue.

By checking node availability and executor status, the cause of the queued pipeline was identified. Restoring the Built-In Node allowed Jenkins to automatically resume the waiting build.

Docker, disk, Harbor connectivity, and Harbor authentication checks confirmed that the CI environment was operational.

The exercise demonstrates the importance of Jenkins agent monitoring, executor availability, infrastructure health checks, and secure credential management.
