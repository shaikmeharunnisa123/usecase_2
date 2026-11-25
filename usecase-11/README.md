# Use Case 11: Fix ImagePullBackOff and Registry Access

## 1. Objective

The objective of this use case is to troubleshoot and resolve Kubernetes image-pull failures when a container image is unavailable or cannot be accessed from a private container registry.

This exercise demonstrates how to:

- Reproduce an image-pull failure using an invalid image tag.
- Identify ErrImagePull and ImagePullBackOff errors.
- Investigate Pod events using kubectl describe pod.
- Verify connectivity to a private Harbor registry.
- Configure Kubernetes authentication using an image pull secret.
- Correct the image reference in a Deployment.
- Verify successful image pulling and application startup.
- Confirm that the Deployment remains healthy after recovery.

## 2. Environment

| Component | Configuration |
|---|---|
| Operating System | Windows with WSL2 Ubuntu |
| Container Runtime | Docker Desktop |
| Kubernetes Context | docker-desktop |
| Kubernetes Version | v1.36.1 |
| Kubernetes Namespace | claims-k8s-lab |
| Container Registry | Harbor |
| Registry Address | host.docker.internal:80 |
| Harbor Project | claims |
| Application | Claims Frontend |
| Test Deployment | claims-frontend-usecase11 |
| Image Pull Secret | harbor-pull-secret |

The existing Claims Management application was already deployed in the claims-k8s-lab namespace.

A separate Deployment named claims-frontend-usecase11 was created for this exercise to avoid modifying the existing application Deployment.

## 3. Registry and Image Configuration

The valid frontend image was built and pushed to Harbor.

**Valid image:**

`host.docker.internal:80/claims/claims-frontend:usecase-11-good`

**Deliberately invalid image:**

`host.docker.internal:80/claims/claims-frontend:usecase-11-missing`

**Valid image digest:**

`sha256:757d030fde3103aaa3b49da5cd2dad781cc82c5399752917ae76c93b38c119c5`

The digest was recorded in evidence/00-good-image-digest.txt.

The Harbor project claims was configured as a private project. Therefore, Kubernetes required registry credentials to pull the image.

## 4. Implementation and Troubleshooting

### Step 1: Verify Harbor Registry Availability

Harbor was configured to use HTTP on port 80.

The registry endpoint was checked using:

`curl -i http://localhost:80/v2/`

The registry returned HTTP 401 Unauthorized. This response was expected because authentication was required. It confirmed that the registry endpoint was reachable.

Docker was configured to recognize host.docker.internal:80 as an insecure registry.

Docker authentication was performed using:

`docker login host.docker.internal:80`

The image was pushed to Harbor:

`docker push host.docker.internal:80/claims/claims-frontend:usecase-11-good`

### Step 2: Create and Verify the Kubernetes Image Pull Secret

A Kubernetes Docker registry secret was created in the same namespace as the Deployment.

Credentials were entered interactively rather than written directly into the command history.

```bash
read -p "Harbor username: " HARBOR_USER
read -s -p "Harbor password: " HARBOR_PASSWORD
echo

kubectl create secret docker-registry harbor-pull-secret \
  --docker-server=host.docker.internal:80 \
  --docker-username="$HARBOR_USER" \
  --docker-password="$HARBOR_PASSWORD" \
  --namespace=claims-k8s-lab

unset HARBOR_USER HARBOR_PASSWORD
