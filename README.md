# runner-appbaq-ansible

The virtual machine provisioned through the companion repository
`runner-appbaq-terraform` is configured as a self-hosted GitHub Actions runner.
The runner is registered with the target application repository, whose
infrastructure is intended for AKS. No AKS resources are provisioned by this
playbook.

## Structure

- [Main playbook](playbook.yml): the configuration role is applied to inventory
  hosts with privilege escalation.
- [Role defaults](roles/runner_setup/defaults/main.yml): GitHub parameters,
  the runner version and checksum are defined here.
- [Role tasks](roles/runner_setup/tasks/main.yml): installation, registration
  and service startup are performed here.
- [Ansible configuration](ansible.cfg): local settings are defined here.

## Parameters

Deployment-specific values, addresses and authentication data are not reproduced
in this documentation.

| Parameter | Purpose |
| --- | --- |
| `runner_setup_github_owner` | Owner of the target repository |
| `runner_setup_github_repo` | Repository with which the runner is registered |
| `runner_setup_version` | Linux x64 package version |
| `runner_setup_archive_checksum` | SHA-256 checksum of the archive for the selected version |
| `runner_setup_github_token` | Temporary registration token supplied at runtime |

The version and checksum must be updated together using the
[official runner releases](https://github.com/actions/runner/releases).
An outdated version may be rejected during registration.
The token placeholder in the role defaults is not a usable token; a temporary
value must be supplied through the workflow.

## Installation and Registration

1. The runner directory is created with the expected permissions.
2. The binary is checked for existence, and its version is read if present.
3. The archive is downloaded and its checksum is verified if the binary is
   absent or its version differs from the configured version.
4. The archive is extracted. Replacement of the package after a failed
   registration is not blocked by an existing copy of `config.sh`.
5. Registration is performed only if the `.runner` file is absent.
6. The service is installed if needed, and its startup is requested.

On a fresh VM, replacement of a GitHub registration with the same name is
allowed through `--replace`. An offline entry left after the previous VM was
destroyed can therefore be replaced. Distinct names must be assigned to
machines that are intended to run concurrently.

Re-registration is prevented by the `.runner` guard. The target repository
identity is not rechecked when that file exists; a target change must therefore
be handled as an explicit reconfiguration.

## Execution and Diagnostics

The repository is checked out from its default branch by the Terraform workflow.
Role changes must therefore be merged into that branch before deployment is
rerun. The inventory, SSH access and temporary token are supplied through this
workflow; no secrets should be added to the README or repository.

Raw registration command output is hidden through `no_log`. On failure, the
exit code, stdout and stderr are displayed after the token has been replaced
with `[REDACTED]`, and the playbook is failed. Service startup errors are not
ignored.

After deployment, the runner's presence and **Online / Idle** status must be
verified in the target application repository's Actions settings. Continuous
availability monitoring is not provided by a successful job.

## Local Checks and Limitations

The syntax check can be run from the repository root in a Linux or WSL
environment with Ansible installed:

```bash
ansible-playbook --syntax-check playbook.yml
```

No connection to the VM is made by this check. Warnings may be displayed if
no inventory is supplied. Under WSL, the local configuration may be ignored
if the mounted directory is considered world-writable.

Updating an already registered and running runner has not been validated
end to end. GitHub deregistration is not automated during Terraform destruction;
an offline entry may be left until it is removed or replaced during a later
deployment.