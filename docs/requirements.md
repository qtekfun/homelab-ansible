# Project Requirements — Phase 1

## Context

| Aspect | Detail |
|---|---|
| Production hypervisor | Proxmox VE |
| Testing environment | Local VM on Fedora (libvirt/KVM) |
| VM operating system | Ubuntu / Debian |
| Scale | < 10 VMs |
| Secret management | Ansible Vault |
| Inventory type | Static YAML |
| CI/CD | GitHub Actions |

---

## Functional Requirements

### FR-01 — VM Bootstrap
When executed against a freshly created VM, the project must:
- Configure hostname, timezone, and locale
- Create a non-root admin user with sudo privileges
- Deploy authorized SSH keys for that user
- Install base packages (curl, vim, git, htop, etc.)

### FR-02 — SSH Hardening
- Configurable SSH port (default: 22, overridable per environment)
- Disable password-based authentication (keys only)
- Disable `PermitRootLogin`
- Configure idle session timeouts (`ClientAliveInterval`, `ClientAliveCountMax`)
- Restrict `MaxAuthTries` and `LoginGraceTime`

### FR-03 — Firewall (UFW)
- Enable UFW with `deny` as the default incoming policy
- Allow SSH on the configured port
- Additional rules configurable per host group or role
- Support for per-environment profiles (testing vs. production)

### FR-04 — User Management
- Define users declaratively via variables (name, groups, SSH keys)
- Support multiple admin users
- Remove users when they are removed from the variable list

### FR-05 — Automatic Security Updates
- Install and configure `unattended-upgrades`
- Configurable maintenance window (schedule)
- Enable only security updates by default
- Optional automatic reboot after updates

### FR-06 — OS Hardening
- Apply security-focused `sysctl` parameters (network stack hardening, SYN cookies, etc.)
- Install and configure `fail2ban` for SSH protection
- Disable unnecessary services

### FR-07 — Secret Management with Ansible Vault
- All sensitive variables must be encrypted with Ansible Vault
- One Vault password file per environment (production / testing), excluded from Git
- Document the workflow for creating and editing encrypted secrets

### FR-08 — Static Inventory per Environment
- Separate YAML inventory files for `production` and `testing`
- Host groups: `all`, `web`, `monitoring`, `infra`
- Group variables and per-host variables supported

### FR-09 — Idempotency
- All playbooks and roles must be safely re-executable with no unintended side effects

### FR-10 — CI/CD Pipeline (GitHub Actions)
- Lint pipeline: `yamllint` + `ansible-lint`
- Triggered on every push and pull request to `master`

---

## Non-Functional Requirements

### NFR-01 — Secure by Default
All roles must ship with secure default values. Relaxing a restriction requires an explicit override by the operator.

### NFR-02 — Modular Role Structure
Each concern (common, security, updates) is an independent, reusable Ansible role. Playbooks are orchestrators only — no business logic inline.

### NFR-03 — No Secrets in Git
`.vault_password*` files and any unencrypted sensitive variables must be listed in `.gitignore`. Only Vault-encrypted files may be committed.

### NFR-04 — Self-Documented Variables
Each role must have a `defaults/main.yml` with every variable listed and documented via inline comments.

### NFR-05 — Multi-Environment Compatibility
The same codebase must work against the local testing VM (Fedora/libvirt) and Proxmox production VMs. The only difference between environments is the inventory.

### NFR-06 — Minimal External Dependencies
Avoid unnecessary third-party collections. Only `community.general` and `ansible.posix` are justified in Phase 1.

### NFR-07 — Clean Lint
The project must pass `yamllint` and `ansible-lint` with zero warnings or errors in CI.

---

## Scope

### In Scope — Phase 1

| Item | Description |
|---|---|
| Project skeleton | Directory structure, `ansible.cfg`, `.gitignore` |
| `common` role | Bootstrap: user, SSH keys, hostname, timezone, base packages |
| `security` role | SSH hardening, UFW, sysctl, fail2ban |
| `updates` role | Unattended-upgrades configuration |
| Inventories | `production` and `testing` with group vars |
| Ansible Vault setup | Encrypted group vars, documented workflow |
| Playbooks | `site.yml`, `bootstrap.yml`, `security.yml` |
| CI/CD | GitHub Actions lint pipeline |
| Documentation | `README.md` with usage instructions |

### Out of Scope — Phase 1

- VM creation on Proxmox (project configures existing VMs only)
- Dynamic inventory from Proxmox API
- Application-specific roles (nginx, postgres, prometheus, etc.)
- Ansible GUI (AWX, Semaphore)
- Automated backups

---

## Implementation Strategy

```
Phase 1a — Skeleton & base configuration
  ├── Directory structure
  ├── ansible.cfg
  ├── .gitignore
  └── Inventory scaffolding (production + testing)

Phase 1b — Core roles
  ├── Role: common   (bootstrap)
  ├── Role: security (SSH hardening + UFW + sysctl + fail2ban)
  └── Role: updates  (unattended-upgrades)

Phase 1c — Orchestration playbooks
  ├── site.yml      (applies all roles)
  ├── bootstrap.yml (new VMs only)
  └── security.yml  (hardening only, re-runnable)

Phase 1d — CI/CD
  ├── .github/workflows/lint.yml
  └── requirements.txt (yamllint, ansible-lint)
```

---

## Proposed Directory Structure

```
homelab-ansible/
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── all.vault.yml        # encrypted with Vault
│   └── testing/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           └── all.vault.yml        # encrypted with Vault
├── roles/
│   ├── common/
│   ├── security/
│   └── updates/
├── playbooks/
│   ├── site.yml
│   ├── bootstrap.yml
│   └── security.yml
├── docs/
│   └── requirements.md              # this file
├── .github/
│   └── workflows/
│       └── lint.yml
├── ansible.cfg
├── requirements.yml
├── .gitignore
└── README.md
```
