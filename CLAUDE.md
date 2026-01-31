# CLAUDE.md - AI Assistant Guide for Homelab Ansible Project

This document provides essential context for AI assistants working with this Ansible infrastructure-as-code repository.

## Project Overview

**Type:** Ansible Infrastructure Automation for Ubuntu Homelab Servers
**Author:** silentijsje
**Purpose:** Automated configuration management, security hardening, and maintenance of Ubuntu servers in a home lab environment

## Directory Structure

```
homelab-ai-test/
├── ansible.cfg              # Ansible runtime configuration
├── requirements.yml         # External Ansible collections
├── inventory/
│   └── hosts.yml           # Target server definitions
├── playbooks/              # Executable automation playbooks
│   ├── bootstrap.yml       # Initial server setup
│   ├── update.yml          # System updates and maintenance
│   └── smb-shares.yml      # SMB/Samba file share configuration
├── roles/                  # Reusable role modules
│   ├── common/             # Basic system configuration
│   ├── security/           # Security hardening (SSH, firewall, fail2ban)
│   ├── updates/            # System update management
│   └── smb_share/          # Samba share management
└── group_vars/
    └── all/
        ├── vars.yml        # Public configuration variables
        └── vault.yml       # Encrypted secrets (ansible-vault)
```

## Quick Command Reference

### Essential Commands

```bash
# Install dependencies
ansible-galaxy collection install -r requirements.yml

# Test connectivity
ansible all -m ping

# Run playbooks
ansible-playbook playbooks/bootstrap.yml           # Bootstrap new servers
ansible-playbook playbooks/update.yml              # Update existing servers
ansible-playbook playbooks/smb-shares.yml --ask-vault-pass  # Configure SMB shares

# Limit to specific host
ansible-playbook playbooks/bootstrap.yml --limit docker01

# Dry run (check mode)
ansible-playbook playbooks/bootstrap.yml --check --diff

# Syntax validation
ansible-playbook playbooks/bootstrap.yml --syntax-check

# Run with specific tags
ansible-playbook playbooks/bootstrap.yml --tags security
```

### Vault Operations

```bash
# Edit encrypted vault
ansible-vault edit group_vars/all/vault.yml

# View vault contents
ansible-vault view group_vars/all/vault.yml

# Run playbook with vault password
ansible-playbook playbooks/smb-shares.yml --ask-vault-pass
ansible-playbook playbooks/smb-shares.yml --vault-password-file ~/.vault_pass
```

## Code Conventions

### Naming Standards

| Type | Convention | Example |
|------|------------|---------|
| Playbooks | lowercase-with-hyphens | `smb-shares.yml` |
| Roles | lowercase_with_underscores | `smb_share` |
| Variables | snake_case | `common_packages`, `smb_workgroup` |
| Hosts/Groups | lowercase | `ubuntu_servers`, `docker01` |
| Tasks | Descriptive phrases | `Install common packages` |

### Task Patterns

All tasks should follow these patterns:

```yaml
- name: Descriptive task name
  ansible.builtin.module_name:
    param: value
  register: result_var          # When output is needed
  when: condition               # Conditional execution
  notify: handler_name          # Trigger handlers
  tags:
    - semantic_tag              # Always include tags
```

### Required Task Elements

1. **Tags:** Every task must have semantic tags (common, security, update, smb, etc.)
2. **FQCN:** Use fully qualified collection names (`ansible.builtin.apt`, not `apt`)
3. **no_log:** Add `no_log: true` for tasks handling sensitive data
4. **Idempotency:** Tasks must be safe to run multiple times

### Role Structure

```
roles/role_name/
├── tasks/main.yml       # Task definitions
├── handlers/main.yml    # Service restart handlers
├── templates/           # Jinja2 templates (*.j2)
├── defaults/main.yml    # Default variable values
└── files/               # Static files to copy
```

## Variable Management

### Variable Hierarchy (lowest to highest precedence)

1. Role defaults (`roles/*/defaults/main.yml`)
2. Group variables (`group_vars/all/vars.yml`)
3. Vault variables (`group_vars/all/vault.yml`)
4. Host variables (if needed)
5. Playbook variables
6. Extra vars (`-e "var=value"`)

### Security Rules

- **Public config:** `group_vars/all/vars.yml` - No secrets
- **Encrypted secrets:** `group_vars/all/vault.yml` - Passwords, keys, tokens
- **Never commit:** Unencrypted passwords, API keys, private keys

## Key Configuration Details

### Ansible Configuration (`ansible.cfg`)

- **Remote user:** `stanley`
- **Privilege escalation:** sudo to root (passwordless)
- **SSH pipelining:** Enabled for performance
- **Fact caching:** JSON files in `/tmp/ansible_facts` (24h TTL)
- **Host key checking:** Disabled (homelab use only)

### Dependencies (`requirements.yml`)

- `community.general` >= 8.0.0
- `ansible.posix` >= 1.5.0

## Adding New Features

### Adding a New Server

1. Edit `inventory/hosts.yml`:
```yaml
ubuntu_servers:
  hosts:
    newserver:
      ansible_host: 192.168.1.x
```

2. Test connectivity: `ansible newserver -m ping`
3. Bootstrap: `ansible-playbook playbooks/bootstrap.yml --limit newserver`

### Creating a New Role

```bash
mkdir -p roles/new_role/{tasks,handlers,templates,defaults}
```

Required files:
- `roles/new_role/tasks/main.yml` - Task definitions
- `roles/new_role/defaults/main.yml` - Default variables (if needed)

### Creating a New Playbook

```yaml
---
# playbooks/new-playbook.yml
- name: Descriptive playbook name
  hosts: ubuntu_servers
  become: true

  roles:
    - role: existing_role
    - role: another_role

  tasks:
    - name: Task description
      ansible.builtin.module:
        param: value
      tags:
        - relevant_tag
```

## Common Patterns

### Handler Pattern

```yaml
# In tasks/main.yml
- name: Update configuration
  ansible.builtin.template:
    src: config.j2
    dest: /etc/app/config
  notify: restart app

# In handlers/main.yml
- name: restart app
  ansible.builtin.service:
    name: app
    state: restarted
```

### Conditional Execution

```yaml
- name: Install package on Ubuntu
  ansible.builtin.apt:
    name: package
    state: present
  when: ansible_distribution == "Ubuntu"
```

### Registered Variables

```yaml
- name: Check service status
  ansible.builtin.command: systemctl is-active myservice
  register: service_status
  changed_when: false
  failed_when: false

- name: Start service if not running
  ansible.builtin.service:
    name: myservice
    state: started
  when: service_status.rc != 0
```

## Testing Changes

1. **Syntax check:** `ansible-playbook playbook.yml --syntax-check`
2. **Dry run:** `ansible-playbook playbook.yml --check --diff`
3. **Verbose output:** `ansible-playbook playbook.yml -vvv`
4. **List tasks:** `ansible-playbook playbook.yml --list-tasks`

## Troubleshooting

### Connection Issues

```bash
# Test SSH manually
ssh stanley@hostname

# Test with Ansible
ansible hostname -m ping -vvv
```

### Privilege Issues

```bash
# Test sudo access
ansible hostname -m command -a "whoami" --become
```

### Variable Debugging

```yaml
- name: Debug variable
  ansible.builtin.debug:
    var: variable_name
```

## Important Files to Review Before Changes

| File | Review When |
|------|-------------|
| `ansible.cfg` | Changing connection settings |
| `inventory/hosts.yml` | Adding/modifying servers |
| `group_vars/all/vars.yml` | Changing global configuration |
| `group_vars/all/vault.yml` | Managing secrets |
| `DOCUMENTATION.md` | Understanding existing features |

## Security Considerations

1. **SSH:** Key-based authentication only, root login disabled
2. **Firewall:** UFW enabled with deny-by-default incoming
3. **Fail2ban:** Brute-force protection enabled
4. **Updates:** Automatic security updates configured
5. **Secrets:** All passwords encrypted with Ansible Vault

## Git Workflow

- **Main branch:** Stable, production-ready code
- **Feature branches:** Use `claude/feature-name-xxx` pattern for AI-assisted development
- **Commits:** Descriptive messages explaining what and why

## Additional Documentation

- `README.md` - Quick start guide
- `DOCUMENTATION.md` - Comprehensive 1,300+ line detailed documentation
- `.vault-password-hint` - Vault password usage instructions
