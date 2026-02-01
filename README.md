# Ubuntu Server Ansible Framework

Ansible framework for bootstrapping and updating Ubuntu servers in a homelab environment.

## Documentation

For detailed information about the framework, code explanations, and advanced usage, see [DOCUMENTATION.md](DOCUMENTATION.md).

## Current Servers

- **docker01.ota.lan** - Docker host server

## Structure

```
.
├── ansible.cfg             # Ansible configuration
├── README.md              # This file - Quick start guide
├── DOCUMENTATION.md       # Detailed documentation and code explanations
├── requirements.yml       # External dependencies
├── inventory/             # Inventory files
│   └── hosts.yml         # Server inventory
├── playbooks/            # Main playbooks
│   ├── bootstrap.yml     # Bootstrap new servers
│   ├── update.yml        # Update existing servers
│   └── smb-shares.yml    # Configure SMB shares
├── roles/                # Ansible roles
│   ├── common/          # Common configuration
│   ├── security/        # Security hardening
│   ├── updates/         # System updates
│   └── smb_share/       # SMB share configuration
├── group_vars/           # Group variables
│   └── all/             # Variables for all hosts
│       ├── vars.yml     # Public variables
│       └── vault.yml    # Encrypted secrets (ansible-vault)
└── host_vars/            # Host-specific variables
```

## Prerequisites

- Ansible 2.16+ installed on control machine
- SSH access to target servers
- Sudo privileges on target servers

## Quick Start

### 1. Install Dependencies

Install required Ansible collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

### 2. Configure Inventory

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

Edit `group_vars/all/vars.yml` to set your preferences.

For secrets (passwords, API keys), edit the encrypted vault:
```bash
ansible-vault edit group_vars/all/vault.yml
```

### 3. Bootstrap New Servers

```bash
ansible-playbook playbooks/bootstrap.yml
```

### 4. Update Existing Servers

```bash
# Update all servers (automatic reboot if required)
ansible-playbook playbooks/update.yml

# Update specific server (e.g., docker01)
ansible-playbook playbooks/update.yml --limit docker01
```

**Note:** Servers will automatically reboot if updates require it (kernel updates, etc.)

### 5. Test Connectivity

```bash
# Test connection to all servers
ansible all -m ping

# Test connection to docker01
ansible docker01 -m ping
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
Updates and cleans up existing servers:
- System package updates
- Security patches
- Full distro upgrades (Ubuntu version updates)
- Removal of unused packages and configurations
- Automatic package cache cleanup

### smb-shares.yml
Configures SMB (Samba) file shares:
- Creates users and groups with specified UID/GID
- Recreates users if UID/GID mismatch detected
- Creates share directories with proper permissions
- Deploys Samba configuration
- Opens firewall for Samba

```bash
# Configure SMB shares (requires vault password)
ansible-playbook playbooks/smb-shares.yml --ask-vault-pass

# Configure on specific server
ansible-playbook playbooks/smb-shares.yml --limit docker01 --ask-vault-pass
```

## Testing

Test connectivity to all servers:
```bash
ansible all -m ping
```

Run in check mode (dry-run):
```bash
ansible-playbook playbooks/bootstrap.yml --check
```

## SMB Share Configuration

Define shares in `group_vars/all/vars.yml`:
```yaml
smb_shares:
  - name: "documents"
    path: "/srv/samba/documents"
    user: "smbdocs"
    group: "smbdocs"
    uid: 2001
    gid: 2001
    permission: "rw"  # or "ro" for read-only
```

Add passwords in `group_vars/all/vault.yml` (encrypted):
```yaml
vault_smb_passwords:
  documents: "YourSecurePassword"
```

## Security

- SSH key-based authentication recommended
- Disable password authentication
- Configure firewall rules
- Regular security updates
- Secrets stored in encrypted Ansible Vault

## Author

Created by silentijsje
