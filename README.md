# homelab-ansible

Ansible project for provisioning and hardening virtual machines in a self-hosted homelab environment.

## Overview

This project automates the baseline configuration of Ubuntu/Debian VMs running on Proxmox VE, covering user management, SSH hardening, firewall setup, and automatic security updates — everything needed to bring a fresh VM to a production-ready state safely and repeatably.

## Environments

| Environment | Hypervisor | Purpose |
|---|---|---|
| `production` | Proxmox VE | Live homelab VMs |
| `testing` | libvirt/KVM (Fedora) | Local development and role testing |

## Project Structure

```
homelab-ansible/
├── inventories/
│   ├── production/       # Production hosts and group vars
│   └── testing/          # Testing hosts and group vars
├── roles/
│   ├── common/           # Bootstrap: user, hostname, base packages
│   ├── security/         # SSH hardening, UFW, sysctl, fail2ban
│   └── updates/          # Unattended security updates
├── playbooks/
│   ├── site.yml          # Full provisioning (all roles)
│   ├── bootstrap.yml     # New VMs only
│   └── security.yml      # Hardening only, safe to re-run
├── docs/
│   └── requirements.md   # Project requirements (FR + NFR)
└── .github/workflows/    # CI/CD: lint pipeline
```

## Requirements

- Ansible >= 2.15
- Python >= 3.10
- `community.general` and `ansible.posix` collections

Install Python dependencies:

```bash
pip install -r requirements.txt
```

Install Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Usage

### Run full provisioning

```bash
ansible-playbook playbooks/site.yml -i inventories/production/hosts.yml --vault-password-file .vault_password
```

### Bootstrap a new VM

```bash
ansible-playbook playbooks/bootstrap.yml -i inventories/production/hosts.yml --vault-password-file .vault_password -l <hostname>
```

### Re-run hardening only

```bash
ansible-playbook playbooks/security.yml -i inventories/production/hosts.yml --vault-password-file .vault_password
```

### Testing environment

```bash
ansible-playbook playbooks/site.yml -i inventories/testing/hosts.yml --vault-password-file .vault_password_testing
```

## Secret Management

Sensitive variables are encrypted with [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html).

```bash
# Encrypt a new secrets file
ansible-vault encrypt inventories/production/group_vars/all.vault.yml

# Edit encrypted secrets
ansible-vault edit inventories/production/group_vars/all.vault.yml

# View encrypted secrets
ansible-vault view inventories/production/group_vars/all.vault.yml
```

The vault password file (`.vault_password`) is listed in `.gitignore` and must never be committed.

## CI/CD

GitHub Actions runs `yamllint` and `ansible-lint` on every push and pull request to `master`. See [`.github/workflows/lint.yml`](.github/workflows/lint.yml).

## Documentation

- [Requirements (FR + NFR)](docs/requirements.md)

## License

MIT
