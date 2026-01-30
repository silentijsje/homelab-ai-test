# Ubuntu Server Ansible Framework

Ansible framework for bootstrapping and updating Ubuntu servers.

## Structure

```
.
├── ansible.cfg           # Ansible configuration
├── inventory/            # Inventory files
│   └── hosts.yml        # Server inventory
├── playbooks/           # Main playbooks
│   ├── bootstrap.yml    # Bootstrap new servers
│   └── update.yml       # Update existing servers
├── roles/               # Ansible roles
│   ├── common/         # Common configuration
│   ├── security/       # Security hardening
│   └── updates/        # System updates
├── group_vars/          # Group variables
│   └── all.yml         # Variables for all hosts
└── host_vars/           # Host-specific variables
```

## Prerequisites

- Ansible 2.16+ installed on control machine
- SSH access to target servers
- Sudo privileges on target servers

## Quick Start

### 1. Configure Inventory

Edit `inventory/hosts.yml` and add your servers:

```yaml
all:
  children:
    ubuntu_servers:
      hosts:
        server1:
          ansible_host: 192.168.1.10
        server2:
          ansible_host: 192.168.1.11
```

### 2. Configure Variables

Edit `group_vars/all.yml` to set your preferences.

### 3. Bootstrap New Servers

```bash
ansible-playbook playbooks/bootstrap.yml
```

### 4. Update Existing Servers

```bash
ansible-playbook playbooks/update.yml
```

## Playbooks

### bootstrap.yml
Bootstraps new Ubuntu servers with:
- Common packages and tools
- Security hardening
- User management
- SSH configuration
- System updates

### update.yml
Updates existing servers:
- System package updates
- Security patches
- Service restarts if needed

## Testing

Test connectivity to all servers:
```bash
ansible all -m ping
```

Run in check mode (dry-run):
```bash
ansible-playbook playbooks/bootstrap.yml --check
```

## Security

- SSH key-based authentication recommended
- Disable password authentication
- Configure firewall rules
- Regular security updates

## Author

Created by silentijsje
