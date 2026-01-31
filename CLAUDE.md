# CLAUDE.md - AI Assistant Guide

This document provides essential context for AI assistants working with this codebase.

## Project Overview

**Type:** Ansible Infrastructure Automation Framework
**Purpose:** Ubuntu server bootstrapping, configuration management, and maintenance for homelab environments
**Target Systems:** Ubuntu servers (Docker hosts, general servers) in home network
**Author:** silentijsje

## Quick Reference Commands

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml
pip install -r requirements-dev.txt

# Run linting (always run before committing)
yamllint .
ansible-lint --offline

# Test a specific role with Molecule
cd roles/common && molecule test

# Run playbooks
ansible-playbook playbooks/bootstrap.yml              # Bootstrap new servers
ansible-playbook playbooks/update.yml                 # Update existing servers
ansible-playbook playbooks/smb-shares.yml --ask-vault-pass  # Configure SMB shares

# Test connectivity
ansible all -m ping

# Dry-run (check mode)
ansible-playbook playbooks/bootstrap.yml --check
```

## Directory Structure

```
homelab-ai-test/
├── ansible.cfg                 # Ansible configuration (remote_user: stanley)
├── requirements.yml            # Ansible Galaxy dependencies
├── requirements-dev.txt        # Development/testing dependencies
├── .ansible-lint              # Ansible linter configuration
├── .yamllint                  # YAML linter configuration
├── .gitignore                 # Git ignore rules
│
├── inventory/
│   └── hosts.yml              # Server inventory (currently: docker01.ota.lan)
│
├── playbooks/
│   ├── bootstrap.yml          # Initial server setup (common -> security -> updates)
│   ├── update.yml             # System updates with automatic reboot
│   └── smb-shares.yml         # Samba share configuration
│
├── roles/
│   ├── common/                # Basic system config (timezone, packages, hostname, journald)
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── molecule/default/  # Molecule tests
│   ├── security/              # Security hardening (SSH, UFW, fail2ban, sysctl)
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   └── molecule/default/
│   ├── updates/               # Package updates (apt dist-upgrade, cleanup)
│   │   ├── tasks/main.yml
│   │   └── molecule/default/
│   └── smb_share/             # Samba file sharing
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/smb.conf.j2
│       ├── defaults/main.yml
│       └── molecule/default/
│
├── group_vars/
│   └── all/
│       ├── vars.yml           # Public variables (SSH settings, packages, firewall)
│       └── vault.yml          # Encrypted secrets (ansible-vault)
│
├── .github/
│   └── workflows/
│       └── ansible-test.yml   # CI/CD pipeline (lint + molecule tests)
│
├── README.md                  # Quick start guide
├── DOCUMENTATION.md           # Detailed documentation
└── TEST_COVERAGE_ANALYSIS.md  # Testing infrastructure analysis
```

## Key Configuration

### Ansible Settings (ansible.cfg)
- **Remote user:** `stanley`
- **Privilege escalation:** sudo to root (passwordless)
- **SSH pipelining:** Enabled for performance
- **Fact caching:** JSON files in `/tmp/ansible_facts` (24h timeout)
- **Host key checking:** Disabled (homelab environment)

### Linting Rules

**yamllint (.yamllint):**
- Max line length: 120 characters (warning level)
- 2-space indentation
- Truthy values: only `true`, `false`, `yes`, `no`

**ansible-lint (.ansible-lint):**
- Runs in offline mode
- Skips: `yaml[line-length]`, `name[missing]`, `load-failure`, `syntax-check[unknown-module]`
- Excludes: `.cache/`, `.git/`, `.github/`, `roles/*/molecule/`

## Development Workflow

### Before Making Changes
1. Read the relevant files to understand existing implementation
2. Check `group_vars/all/vars.yml` for configurable variables
3. Review existing patterns in similar roles/tasks

### Making Changes
1. Follow existing code style (2-space indentation, YAML syntax)
2. Use descriptive task names
3. Add appropriate tags to tasks
4. Use handlers for service restarts
5. Include validation where applicable (e.g., `validate:` parameter)

### After Making Changes
1. Run linting: `yamllint . && ansible-lint --offline`
2. Test with Molecule if modifying roles: `cd roles/<role> && molecule test`
3. Test in check mode: `ansible-playbook playbooks/<playbook>.yml --check`

## CI/CD Pipeline

**Triggers:** Push to `main`, `feature/**`, `claude/**` branches; PRs to `main`

**Pipeline stages:**
1. **Lint job:** yamllint + ansible-lint
2. **Molecule jobs (parallel, after lint passes):**
   - common role
   - security role
   - updates role
   - smb_share role

All tests must pass before merging to main.

## Role Patterns

### Task Structure
```yaml
- name: Descriptive task name
  module_name:
    param: value
  tags: ['role-name', 'feature']
  notify: handler-name  # If service restart needed
```

### Handler Pattern
```yaml
- name: restart service-name
  systemd:
    name: service-name
    state: restarted
```

### Template Validation
```yaml
- name: Deploy configuration
  template:
    src: config.j2
    dest: /etc/service/config
    validate: 'command -t %s'  # Validate before applying
  notify: restart service
```

## Important Conventions

### Variable Naming
- Public variables: `snake_case` in `group_vars/all/vars.yml`
- Vault variables: Prefix with `vault_` (e.g., `vault_smb_passwords`)
- Role defaults: In `roles/<role>/defaults/main.yml`

### Tags
- Use role name as primary tag (e.g., `common`, `security`)
- Add feature-specific tags (e.g., `packages`, `firewall`, `ssh`)
- Use `always` tag sparingly for critical pre/post tasks

### Error Handling
- Use `failed_when` for custom failure conditions
- Use `changed_when` to accurately report state changes
- Use `ignore_errors: true` only when failure is acceptable

### Secrets Management
- All secrets in `group_vars/all/vault.yml` (encrypted)
- Use `no_log: true` for tasks handling sensitive data
- Access vault: `ansible-vault edit group_vars/all/vault.yml`

## Testing with Molecule

Each role has a Molecule test scenario at `roles/<role>/molecule/default/`:
- `molecule.yml` - Test configuration (Docker driver, Ubuntu 22.04)
- `converge.yml` - Applies the role with test variables
- `verify.yml` - Verification tasks (checks expected state)

**Test sequence:** dependency -> create -> prepare -> converge -> idempotence -> verify -> destroy

## Common Pitfalls

1. **SSH lockout risk:** Be careful with security role SSH changes
2. **UID/GID conflicts:** smb_share role has complex user recreation logic
3. **Reboot handling:** update.yml auto-reboots when kernel updates require it
4. **Vault password:** Required for smb-shares.yml playbook
5. **Python interpreter:** Ensure `/usr/bin/python3` exists on target hosts

## File Modification Guidelines

### When Adding New Roles
1. Create standard structure: `tasks/`, `handlers/`, `defaults/`, `templates/`
2. Add Molecule test scenario
3. Register in relevant playbook
4. Update documentation

### When Modifying Existing Roles
1. Maintain idempotency
2. Update Molecule verify.yml if behavior changes
3. Test with `molecule test`

### When Adding Variables
1. Add to `group_vars/all/vars.yml` with comments
2. If sensitive, use `group_vars/all/vault.yml`
3. Document in DOCUMENTATION.md

## References

- **Quick start:** README.md
- **Detailed docs:** DOCUMENTATION.md
- **Test analysis:** TEST_COVERAGE_ANALYSIS.md
- **Ansible docs:** https://docs.ansible.com/
- **Molecule docs:** https://ansible.readthedocs.io/projects/molecule/
