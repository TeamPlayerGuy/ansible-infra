# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Ansible infrastructure codebase for provisioning and managing Kubernetes clusters on Raspberry Pi and ARM hardware. Uses a layered role approach from base OS configuration through Kubernetes setup to feature enablement.

## Common Commands

```bash
# Syntax check a playbook
ansible-playbook --syntax-check playbooks/day1/1-bootstrap-os.yml

# Gather facts on target hosts
ansible-playbook playbooks/testing/facts.yml -e 'target_hosts=k8s_new_workers'

# Run bootstrap on specific host group
ansible-playbook playbooks/day1/1-bootstrap-os.yml -l k8s_new_workers

# Run with specific tags
ansible-playbook playbooks/day2/add-user.yml --tags service

# Dry run (check mode)
ansible-playbook playbooks/day1/1-bootstrap-os.yml --check

# Run base role on a host
ansible-playbook playbooks/base_role.yml
```

## Architecture

### Directory Structure

```
infra-host/
├── ansible.cfg          # Ansible configuration (inventory path, roles path, privilege escalation)
├── collections/         # Ansible collections (placeholder)
├── docs/                # Documentation (placeholder)
├── inventory/
│   ├── hosts.yml        # Host definitions and groups
│   ├── group_vars/      # Group-level variables
│   │   ├── all/         # Global defaults (ansible_user, python interpreter, SSH key)
│   │   └── [group]/     # Group-specific overrides (e.g., k8s_new_workers/packages.yml)
│   └── host_vars/       # Host-specific overrides
├── playbooks/
│   ├── day1/            # Bootstrap sequence: OS (1) → K8s (2) → Cluster services (3)
│   ├── day2/            # Operational tasks (user management, maintenance)
│   ├── testing/         # Validation and debugging playbooks
│   └── base_role.yml    # Runs base_unix_os role for testing
├── plugins/
│   ├── filters/         # Custom Jinja2 filters (placeholder)
│   └── modules/         # Custom Ansible modules (placeholder)
├── roles/               # Layered roles (see Role Layering below)
└── templates/           # Jinja2 templates
```

### Role Layering (Applied in Order)

1. **base_unix_os** - Base OS config for all K8s nodes
   - Package installation, kernel modules (overlay, br_netfilter)
   - Swap disabled, sysctl networking (IP forwarding, bridge netfilter)
   - SSH hardening (no root login, key-only auth, AllowUsers)
   - Oh My Bash installation

2. **OS Family Roles** - OS-specific package management and repos
   - **os_debian** - Secure apt repos (Docker, Kubernetes), GPG keyrings
   - **os_rhel** - RHEL/CentOS specifics (stub)
   - **os_ubuntu** - Ubuntu specifics (stub)

3. **Platform Roles** - Hardware-specific configurations
   - **platform_pi** - Raspberry Pi fan control, bootloader DTB override
   - **platform_pi_5b** - Pi 5 Model B specifics (stub)
   - **platform_generic_arm** - Generic ARM boards (stub)
   - **platform_iota** - IOTA boards (stub)
   - **platform_proxmox** - Proxmox VMs (stub)
   - **platform_vmware** - VMware VMs (stub)

4. **Kubernetes Roles** - K8s node setup
   - **k8s_base** - Common K8s packages (containerd.io, kubelet, kubeadm, kubectl)
   - **k8s_master** - Control plane setup via kubeadm init (stub)
   - **k8s_worker** - Worker node setup via kubeadm join (stub)

5. **Feature Roles** - Optional workload-specific features
   - **feature_storage_longhorn** - Longhorn prerequisites (nfs-common, open-iscsi, iscsid service)
   - **feature_compute_transcode** - Transcoding workloads (stub)

### Inventory Structure

Host groups follow a live/new pattern for production vs staging:

```yaml
k8s_live_masters:     # Production control plane (kubepi2)
k8s_live_workers:     # Production workers (kubepi3, kubepi16wrk001)
k8s_new_masters:      # Staging control plane
k8s_new_workers:      # Staging workers (suarm16wrk001, suarm16wrk002)
k8s_live_cluster:     # Children: k8s_live_masters + k8s_live_workers
k8s_new_cluster:      # Children: k8s_new_masters + k8s_new_workers
```

### Bootstrap Workflow

Pre-requisites (manual):
1. Static IP configured on eth0
2. mDNS (Avahi) installed
3. SSH keys in authorized_keys, node known to host
4. Sudoers configured for NOPASSWD:ALL for ansible accounts

Bootstrap sequence:
1. **OS Bootstrap** (1-bootstrap-os.yml) - Swap off, kernel modules, bootloader, packages
2. **K8s Bootstrap** (2-bootstrap-k8s.yml) - HTTPS apt, keyrings, containerd, K8s install, CNI
3. **Cluster Bootstrap** (3-bootstrap-cluster.yml) - Helm, Metrics Server, cluster services

## Code Conventions

### Required Patterns

- **SPDX header**: All YAML files start with `#SPDX-License-Identifier: MIT-0`
- **Fully-qualified module names**: Use `ansible.builtin.command`, `ansible.posix.mount`, etc.
- **Task imports**: Main task file (`tasks/main.yml`) imports subtasks from separate files
- **OS-aware logic**: Use facts for cross-platform compatibility
  ```yaml
  ssh_service_name: "{{ 'ssh' if ansible_facts.os_family == 'Debian' else 'sshd' }}"
  ```

### Role Structure

```
role_name/
├── tasks/main.yml      # Imports subtasks
├── tasks/*.yml         # Individual task files by function
├── defaults/main.yml   # Default variables (lowest priority)
├── vars/main.yml       # Role variables (higher priority)
├── files/              # Static config files (prefixed: k8s-modules.conf)
├── handlers/main.yml   # Event handlers (e.g., restart ssh)
└── meta/main.yml       # Role metadata
```

### Configuration Files

- Static configs go in `roles/[role]/files/` with component prefix (e.g., `k8s-modules.conf`, `k8s-sysctl.conf`)
- Include comment header: `# Managed by Ansible` or `# Managed by Ansible [role_name] role`

### Idempotency

- Use `when` clauses for conditional execution
- Use `register` with `changed_when: false` for read-only commands
- Use `check_mode: false` for commands that must run even in check mode

### Tags

Use tags to run specific task subsets:
```bash
ansible-playbook playbooks/base_role.yml --check --diff --tags ssh
```

Common tags: `base_packages`, `kernel`, `swap`, `ssh`, `omb`, `secure_repo`

## Documentation State

Last documented commit: 9732b5d

When updating this file, diff from the above commit to HEAD to identify changes, then update the hash after documenting.
