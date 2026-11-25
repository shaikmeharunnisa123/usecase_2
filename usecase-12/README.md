# Use Case 12 – Rotate Secrets Without Breaking the Application

## 1. Objective

Demonstrate how to rotate an application secret using HashiCorp Vault
and synchronize the new value with Kubernetes without exposing the
secret or changing application code.

The lab also demonstrates a controlled missing-secret failure and
recovery from Vault.

## 2. Environment

| Component | Configuration |
|---|---|
| Operating environment | Windows, WSL Ubuntu, Docker Desktop |
| Vault container | usecase12-vault |
| Vault address | http://127.0.0.1:8201 |
| Vault secret path | secret/midhlab/usecase12 |
| Vault key | api_token |
| Kubernetes namespace | claims-k8s-lab |
| Kubernetes Secret | claims-usecase12-secret |
| Kubernetes key | API_TOKEN |
| Deployment | claims-backend |
| Application health endpoint | /api/health |
| Git branch | feature/usecase12 |

## 3. Architecture and Secret Flow

Vault is the source of truth for the lab API token.

Vault
  |
  | Read selected secret version
  v
Kubernetes Secret: claims-usecase12-secret
  |
  | Environment variable reference
  v
Deployment: claims-backend
  |
  v
Backend Pod: API_TOKEN

Updating Vault does not automatically update an existing Kubernetes
Secret. The Kubernetes Secret must be synchronized and the Deployment
restarted so a replacement Pod loads the updated value.

## 4. Implementation Summary

### Step 1 – Secret selection

Selected a lab-only API token and documented the Vault path, key,
Kubernetes Secret, namespace, and Deployment.

### Step 2 – Initial secret creation

Generated an initial random token, stored it in Vault Version 1,
created the Kubernetes Secret, and configured the backend Deployment
to reference API_TOKEN.

### Step 3 – Secret rotation in Vault

Generated a new random token and stored it in Vault.

Vault metadata confirmed that the current secret version became 2.
The raw secret value was never printed or saved in evidence.

### Step 4 – Kubernetes synchronization

Updated claims-usecase12-secret with the rotated value and restarted
the claims-backend Deployment.

The rollout completed successfully.

### Step 5 – Application verification

Verified that:
- API_TOKEN was loaded in the backend Pod.
- The Vault value and Pod environment value matched.
- The backend Pod was Running.
- The application health endpoint returned HTTP 200.

### Step 6 – Security audit

The following checks passed:
- No raw Version 1 or Version 2 secret values found in Use Case 12 files.
- No matching authentication or application error patterns found in
  the checked backend logs.
- No Vault token pattern found by the Git-tracked content scan.

### Step 7 – Controlled failure and recovery

Deleted the Kubernetes Secret intentionally and restarted the
Deployment.

The replacement Pod entered CreateContainerConfigError because the
referenced Secret was missing. Kubernetes events confirmed:

secret "claims-usecase12-secret" not found

Recovered the value from Vault, recreated the Kubernetes Secret,
and restarted the Deployment.

The recovery rollout succeeded. The backend returned to 1/1 Running,
the health endpoint returned HTTP 200, and the recovered Pod value
matched the Vault value.

## 5. Final Verified Results

| Verification | Result |
|---|---|
| Vault secret Version 2 | PASS |
| Kubernetes Secret synchronized | PASS |
| Backend Pod loaded API_TOKEN | PASS |
| Vault and Pod secret comparison | PASS |
| Backend rollout after rotation | PASS |
| Application health after rotation | HTTP 200 |
| Raw secret exposure audit | PASS |
| Backend log pattern audit | PASS |
| Vault token pattern audit | PASS |
| Controlled missing-secret failure | Verified |
| Kubernetes missing-secret event | Verified |
| Recovery from Vault | PASS |
| Backend rollout after recovery | PASS |
| Application health after recovery | HTTP 200 |
| Vault and recovered Pod comparison | PASS |

## 6. Rollback Plan

Use rollback if a newly rotated credential causes authentication,
startup, readiness, or application failures.

Vault Version 1 contains the previous credential. To keep Vault as
the source of truth, promote that previous value into Vault as a new
version, then synchronize Kubernetes.

### Emergency rollback procedure

The following commands are documented for emergency use only.
They were not executed because the Version 2 rotation succeeded.

```bash
ROLLBACK_TOKEN=$(vault kv get \
  -version=1 \
  -field=api_token \
  secret/midhlab/usecase12)

vault kv put secret/midhlab/usecase12 \
  api_token="$ROLLBACK_TOKEN"

kubectl create secret generic claims-usecase12-secret \
  --from-literal=API_TOKEN="$ROLLBACK_TOKEN" \
  -n claims-k8s-lab \
  --dry-run=client \
  -o yaml \
  | kubectl apply -f -

unset ROLLBACK_TOKEN

kubectl rollout restart deployment/claims-backend \
  -n claims-k8s-lab

kubectl rollout status deployment/claims-backend \
  -n claims-k8s-lab \
  --timeout=180s
```

After rollback, verify the latest Vault version, Kubernetes Secret
synchronization, backend Pod status, and application health.

## 7. Security Practices

- Use only lab credentials in this exercise.
- Never print raw secret values.
- Never save Vault root tokens or decoded Kubernetes Secret values
  in Git or evidence.
- Do not enable shell tracing with `set -x` while handling secrets.
- Unset temporary secret variables after use.
- Keep Vault development mode restricted to local lab practice.
- Vault development storage is in-memory and is not production storage.

## 8. Evidence Directory

All execution evidence is stored under `evidence/`.

- 01-secret-selection-evidence.txt
- 02-kubernetes-secret-baseline.txt
- 02-backend-secret-reference.txt
- 03-vault-rotation-metadata.txt
- 04-application-health.txt
- 05-pod-secret-loaded.txt
- 05-secret-sync-verification.txt
- 05-pod-status.txt
- 06-secret-exposure-check.txt
- 06-log-validation.txt
- 06-vault-token-check.txt
- 07-controlled-failure-pods.txt
- 07-controlled-failure-events.txt
- 07-recovery-rollout.txt
- 07-recovery-pods.txt
- 07-recovery-secret-reference.txt
- 07-health-after-recovery.txt
- 07-recovery-secret-sync.txt

## 9. Conclusion

Use Case 12 demonstrated a secret rotation and recovery workflow
using Vault and Kubernetes.

The application consumed the rotated secret after Kubernetes
synchronization and Deployment restart.

A deliberate missing-secret failure was reproduced, diagnosed using
Kubernetes events, and recovered using Vault without modifying
application code.

The final backend was healthy, and the recovered Pod secret matched
the Vault secret.
