# Use Case 7 – Automate Operations with Ansible and Vault

## 1. Objective

This use case demonstrates how Ansible can automate operational configuration management and how HashiCorp Vault can securely store and rotate sensitive values.

The implementation uses a local Ubuntu environment through WSL. No production systems or credentials are involved.

## 2. Tools and Technologies

- Ubuntu on WSL
- Ansible
- HashiCorp Vault
- Docker
- Git

## 3. Project Structure

```text
usecase-07/
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── playbooks/
│   └── site.yml
├── evidence/
│   ├── 01-inventory.txt
│   ├── 02-ping.txt
│   ├── 03-facts.txt
│   ├── 04-check-mode.txt
│   ├── 05-first-run.txt
│   ├── 06-second-run.txt
│   ├── 07-controlled-failure.txt
│   ├── 08-recovery.txt
│   ├── 09-vault-status.txt
│   ├── 10-vault-masked-read.txt
│   ├── 11-vault-rotation.txt
│   └── 12-vault-metadata.txt
├── vault/
└── README.md
```

## 4. Ansible Configuration

The Ansible configuration defines the project inventory and default execution settings.

The inventory contains a local managed host:

```ini
[lab]
localhost ansible_connection=local
```

The local connection allows Ansible to execute tasks on the WSL machine without requiring SSH.

## 5. Configuration Management Playbook

The playbook `playbooks/site.yml` performs the following tasks:

1. Connects to the lab host.
2. Gathers system facts.
3. Creates `/tmp/usecase07-managed.conf`.
4. Writes the Claims Management application configuration.
5. Displays a confirmation message.

The managed configuration contains:

```ini
application=claims-management
environment=lab
managed_by=ansible
use_case=07
```

The changes are restricted to the temporary configuration file.

## 6. Check Mode and Idempotency

Ansible check mode was used to preview the changes without applying them:

```bash
ansible-playbook playbooks/site.yml --check --diff
```

The playbook was then executed for the first time. The configuration file was created, and Ansible reported one changed task.

The playbook was executed again without modifying the configuration.

The results were:

| Execution | Changed tasks |
|---|---:|
| First run | 1 |
| Second run | 0 |

This demonstrates **idempotency**: repeatedly running the same playbook does not make unnecessary changes when the system is already in the desired state.

## 7. Connection Failure and Recovery

A separate temporary inventory was created with an unreachable test host at `192.0.2.1`.

The Ansible ping operation was used to simulate a connection failure. The connection timed out as expected.

The original localhost inventory was then tested again, and the connection succeeded.

This demonstrated how Ansible reports unreachable hosts and how normal operations can be verified after a failed connection test.

## 8. HashiCorp Vault Integration

A temporary HashiCorp Vault development server was started in Docker.

A temporary lab password was entered interactively and stored in Vault's KV secrets engine at:

```text
secret/midhlab/app
```

The password was read through Vault with its output redirected to `/dev/null`. Only a success message was saved as evidence; the secret value was not printed.

### Secret Rotation

The temporary password was updated with a new value.

Vault metadata confirmed:

- Secret path: `secret/midhlab/app`
- Current version: 2
- Version 1 and version 2 recorded in metadata

This demonstrated secret versioning and rotation without exposing the secret value.

## 9. Evidence

The `evidence/` directory contains the command outputs and verification results for:

- Inventory configuration
- Ansible connectivity and system facts
- Check mode
- First and second playbook executions
- Controlled connection failure and recovery
- Vault server status
- Masked secret read
- Secret rotation
- Vault metadata and version verification

## 10. Security Considerations

- Only temporary lab credentials were used.
- The Vault development server was used for learning and testing, not production.
- Secret values and Vault tokens were not intentionally included in the evidence files.
- The temporary Vault server and its development data should not be treated as persistent or production-secure storage.
- Real deployments should use an appropriately secured Vault configuration, access policies, and protected credentials.

## 11. Cleanup

After collecting all evidence, stop and remove the temporary Vault container:

```bash
docker rm -f usecase07-vault
```

Clear the temporary Vault environment variables:

```bash
unset VAULT_TOKEN
unset VAULT_ADDR
```

Remove the temporary failure-test inventory:

```bash
rm -f /tmp/usecase07-bad-inventory.ini
```

The Ansible-managed configuration file can also be removed after verification:

```bash
rm -f /tmp/usecase07-managed.conf
```

## 12. Conclusion

Use Case 7 demonstrated configuration management with Ansible, idempotent playbook execution, connection failure and recovery testing, and secure temporary secret storage and rotation using HashiCorp Vault.

The implementation and supporting evidence are maintained in the `usecase-07` directory.
