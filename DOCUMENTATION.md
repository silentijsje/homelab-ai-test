# Homelab Ansible Framework - Detailed Documentation

## Table of Contents
1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Configuration Files](#configuration-files)
4. [Inventory Management](#inventory-management)
5. [Playbooks](#playbooks)
6. [Roles](#roles)
7. [Variables](#variables)
8. [Usage Examples](#usage-examples)
9. [Security Considerations](#security-considerations)

## Overview

This Ansible framework provides automated configuration management for Ubuntu servers in a homelab environment. It follows Ansible best practices with a role-based architecture for modularity and reusability.

### Key Features
- Automated server bootstrapping
- Security hardening (SSH, firewall, fail2ban)
- System updates and package management
- Centralized configuration management
- Modular role-based design

## Architecture

### Directory Structure Explained

```
homelab-ai-test/
├── ansible.cfg              # Main Ansible configuration file
├── requirements.yml         # External dependencies (collections/roles)
├── README.md               # Quick start guide
├── DOCUMENTATION.md        # This file - detailed documentation
├── inventory/              # Server inventory definitions
│   └── hosts.yml          # YAML inventory file
├── playbooks/             # Executable playbooks
│   ├── bootstrap.yml      # Initial server setup
│   └── update.yml         # Server update operations
├── roles/                 # Reusable role definitions
│   ├── common/           # Basic system configuration
│   │   ├── tasks/        # Task definitions
│   │   │   └── main.yml
│   │   └── handlers/     # Event handlers
│   │       └── main.yml
│   ├── security/         # Security hardening
│   │   ├── tasks/
│   │   │   └── main.yml
│   │   ├── handlers/
│   │   │   └── main.yml
│   │   └── templates/    # Configuration templates
│   │       ├── jail.local.j2
│   │       └── 50unattended-upgrades.j2
│   └── updates/          # System updates
│       └── tasks/
│           └── main.yml
├── group_vars/           # Group-level variables
│   └── all.yml          # Variables for all hosts
└── host_vars/            # Host-specific variables (optional)
```

## Configuration Files

### ansible.cfg

The main Ansible configuration file that controls behavior across all playbook runs.

```ini
[defaults]
inventory = inventory/hosts.yml          # Default inventory location
host_key_checking = False               # Disable SSH host key verification (homelab use)
remote_user = ubuntu                    # Default SSH user
nocows = 1                             # Disable cow ASCII art in output
```

**Key Settings Explained:**

- **SSH Connection Pooling:**
  ```ini
  [ssh_connection]
  ssh_args = -o ControlMaster=auto -o ControlPersist=60s
  pipelining = True
  ```
  - `ControlMaster=auto`: Reuses SSH connections for better performance
  - `ControlPersist=60s`: Keeps connections open for 60 seconds
  - `pipelining=True`: Reduces SSH operations by batching commands

- **Privilege Escalation:**
  ```ini
  [privilege_escalation]
  become = True                         # Enable sudo by default
  become_method = sudo                  # Use sudo for privilege escalation
  become_user = root                    # Become root user
  become_ask_pass = False              # Don't prompt for sudo password
  ```
  **Security Note:** This assumes passwordless sudo is configured. For production, use `become_ask_pass = True` or Ansible Vault for credentials.

- **Fact Caching:**
  ```ini
  gathering = smart                     # Only gather facts when needed
  fact_caching = jsonfile              # Cache facts to JSON files
  fact_caching_connection = /tmp/ansible_facts
  fact_caching_timeout = 86400         # Cache for 24 hours
  ```
  Improves performance by avoiding redundant fact gathering.

### requirements.yml

Defines external Ansible collections needed by the playbooks.

```yaml
collections:
  - name: community.general
    version: ">=8.0.0"                 # Community-maintained modules
  - name: ansible.posix
    version: ">=1.5.0"                 # POSIX system modules
```

**Installation:**
```bash
ansible-galaxy collection install -r requirements.yml
```

## Inventory Management

### hosts.yml Structure

The inventory file defines all managed servers and their connection parameters.

```yaml
---
all:                                   # Root group containing all hosts
  children:                            # Child groups
    ubuntu_servers:                    # Group for Ubuntu servers
      hosts:                           # Individual host definitions
        docker01:                      # Host name (used in Ansible)
          ansible_host: docker01.ota.lan    # Actual hostname/IP
          ansible_user: ubuntu               # SSH user for this host
          # ansible_ssh_private_key_file: ~/.ssh/id_rsa  # Optional SSH key

      vars:                            # Group-level variables
        ansible_python_interpreter: /usr/bin/python3  # Python path
        ansible_user: ubuntu                          # Default SSH user
```

**Host Variables Explained:**
- `ansible_host`: The actual hostname or IP address to connect to
- `ansible_user`: SSH username for authentication
- `ansible_ssh_private_key_file`: Path to SSH private key (optional)
- `ansible_python_interpreter`: Python interpreter path on remote host

**Adding More Servers:**
```yaml
hosts:
  docker01:
    ansible_host: docker01.ota.lan
  docker02:
    ansible_host: docker02.ota.lan
  webserver01:
    ansible_host: 192.168.1.100
```

## Playbooks

### bootstrap.yml - Initial Server Setup

This playbook performs complete initial configuration of new Ubuntu servers.

```yaml
---
- name: Bootstrap Ubuntu Server
  hosts: ubuntu_servers              # Target group from inventory
  become: yes                        # Use sudo for all tasks
  gather_facts: yes                  # Collect system information
```

**Execution Flow:**

1. **Pre-tasks** (Run before roles):
   ```yaml
   pre_tasks:
     - name: Update apt cache
       apt:
         update_cache: yes
         cache_valid_time: 3600     # Only update if cache older than 1 hour
   ```

2. **Roles** (Applied in order):
   ```yaml
   roles:
     - common                        # Basic system setup
     - security                      # Security hardening
     - updates                       # System updates
   ```

3. **Post-tasks** (Run after roles):
   ```yaml
   post_tasks:
     - name: Check if reboot is required
       stat:
         path: /var/run/reboot-required
       register: reboot_required

     - name: Notify if reboot is needed
       debug:
         msg: "*** REBOOT REQUIRED ***"
       when: reboot_required.stat.exists
   ```

**Usage:**
```bash
# Bootstrap all servers
ansible-playbook playbooks/bootstrap.yml

# Bootstrap specific server
ansible-playbook playbooks/bootstrap.yml --limit docker01

# Dry run (check mode)
ansible-playbook playbooks/bootstrap.yml --check

# Skip specific roles
ansible-playbook playbooks/bootstrap.yml --skip-tags security
```

### update.yml - Server Updates

Applies system updates and security patches to existing servers.

```yaml
---
- name: Update Ubuntu Servers
  hosts: ubuntu_servers
  become: yes
  gather_facts: yes
```

**Main Tasks:**

1. **Update Package Cache:**
   ```yaml
   - name: Update apt cache
     apt:
       update_cache: yes
     changed_when: false             # Don't mark as changed
   ```

2. **Upgrade All Packages:**
   ```yaml
   - name: Upgrade all packages
     apt:
       upgrade: dist                 # Distribution upgrade
       update_cache: yes
       autoremove: yes               # Remove unused dependencies
       autoclean: yes                # Clean up downloaded packages
     register: upgrade_result
   ```

3. **Conditional Reboot:**
   ```yaml
   - name: Check if reboot is required
     stat:
       path: /var/run/reboot-required
     register: reboot_required

   - name: Reboot if required
     reboot:
       reboot_timeout: 300           # Wait up to 5 minutes
       post_reboot_delay: 30         # Wait 30 seconds after reboot
     when:
       - reboot_required.stat.exists
       - reboot_after_updates | default(false)
   ```

**Usage:**
```bash
# Update all servers (no reboot)
ansible-playbook playbooks/update.yml

# Update with automatic reboot
ansible-playbook playbooks/update.yml -e "reboot_after_updates=true"

# Update specific server
ansible-playbook playbooks/update.yml --limit docker01
```

## Roles

### Common Role

Handles basic system configuration that applies to all servers.

**Tasks (roles/common/tasks/main.yml):**

1. **Timezone Configuration:**
   ```yaml
   - name: Set timezone
     timezone:
       name: "{{ system_timezone }}"     # From group_vars/all.yml
     tags: ['common', 'timezone']
   ```

2. **Package Installation:**
   ```yaml
   - name: Install common packages
     apt:
       name: "{{ common_packages }}"     # List from group_vars
       state: present
       update_cache: yes
     tags: ['common', 'packages']
   ```
   Installs essential tools like vim, git, curl, htop, docker, etc.

3. **Locale Configuration:**
   ```yaml
   - name: Generate locale
     locale_gen:
       name: en_US.UTF-8
       state: present
     tags: ['common', 'locale']
   ```

4. **Hostname Setup:**
   ```yaml
   - name: Set hostname
     hostname:
       name: "{{ inventory_hostname }}"   # From inventory
     tags: ['common', 'hostname']

   - name: Update /etc/hosts
     lineinfile:
       path: /etc/hosts
       line: "127.0.1.1 {{ inventory_hostname }}"
       regexp: '^127\.0\.1\.1'
   ```

5. **Journald Configuration:**
   ```yaml
   - name: Configure journald
     lineinfile:
       path: /etc/systemd/journald.conf
       regexp: "{{ item.regexp }}"
       line: "{{ item.line }}"
     loop:
       - { regexp: '^#?SystemMaxUse=', line: 'SystemMaxUse=500M' }
       - { regexp: '^#?SystemMaxFileSize=', line: 'SystemMaxFileSize=100M' }
     notify: restart systemd-journald
   ```
   Limits log file sizes to prevent disk space issues.

6. **Disable Snapd (Optional):**
   ```yaml
   - name: Disable snapd service
     systemd:
       name: snapd
       state: stopped
       enabled: no
     failed_when: false               # Don't fail if service doesn't exist
   ```

**Handlers (roles/common/handlers/main.yml):**
```yaml
- name: restart systemd-journald
  systemd:
    name: systemd-journald
    state: restarted
```

### Security Role

Implements security hardening measures.

**Tasks (roles/security/tasks/main.yml):**

1. **SSH Hardening:**
   ```yaml
   - name: Configure SSH - Disable root login
     lineinfile:
       path: /etc/ssh/sshd_config
       regexp: '^#?PermitRootLogin'
       line: 'PermitRootLogin no'
     notify: restart sshd

   - name: Configure SSH - Disable password authentication
     lineinfile:
       path: /etc/ssh/sshd_config
       regexp: '^#?PasswordAuthentication'
       line: 'PasswordAuthentication no'
     notify: restart sshd
   ```

   Additional SSH settings:
   - `PubkeyAuthentication yes`: Enable key-based auth
   - `X11Forwarding no`: Disable X11 for security
   - `MaxAuthTries 3`: Limit authentication attempts
   - `ClientAliveInterval 300`: Timeout idle connections

2. **Firewall (UFW) Configuration:**
   ```yaml
   - name: Install UFW
     apt:
       name: ufw
       state: present

   - name: Set UFW default policies
     ufw:
       direction: "{{ item.direction }}"
       policy: "{{ item.policy }}"
     loop:
       - { direction: 'incoming', policy: 'deny' }
       - { direction: 'outgoing', policy: 'allow' }

   - name: Allow SSH through firewall
     ufw:
       rule: allow
       port: "{{ ssh_port }}"
       proto: tcp

   - name: Enable UFW
     ufw:
       state: enabled
   ```

3. **Fail2Ban Installation:**
   ```yaml
   - name: Install fail2ban
     apt:
       name: fail2ban
       state: present

   - name: Configure fail2ban
     template:
       src: jail.local.j2
       dest: /etc/fail2ban/jail.local
     notify: restart fail2ban
   ```

   **jail.local.j2 Template:**
   ```jinja
   [DEFAULT]
   bantime = 3600              # Ban for 1 hour
   findtime = 600              # Look for failures in 10 min window
   maxretry = 5                # Allow 5 retries before ban

   [sshd]
   enabled = true
   port = {{ ssh_port }}
   ```

4. **Automatic Security Updates:**
   ```yaml
   - name: Install unattended-upgrades
     apt:
       name: unattended-upgrades
       state: present

   - name: Configure automatic updates
     template:
       src: 50unattended-upgrades.j2
       dest: /etc/apt/apt.conf.d/50unattended-upgrades
   ```

   **50unattended-upgrades.j2 Template:**
   ```jinja
   Unattended-Upgrade::Allowed-Origins {
       "${distro_id}:${distro_codename}-security";
       "${distro_id}ESMApps:${distro_codename}-apps-security";
   };

   Unattended-Upgrade::Automatic-Reboot "false";
   Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
   Unattended-Upgrade::Remove-Unused-Dependencies "true";
   ```

5. **Network Hardening (sysctl):**
   ```yaml
   - name: Configure network hardening
     sysctl:
       name: "{{ item.key }}"
       value: "{{ item.value }}"
       state: present
       reload: yes
     loop:
       - { key: 'net.ipv4.conf.all.send_redirects', value: '0' }
       - { key: 'net.ipv4.conf.default.send_redirects', value: '0' }
       - { key: 'net.ipv4.conf.all.accept_source_route', value: '0' }
       - { key: 'net.ipv4.icmp_echo_ignore_broadcasts', value: '1' }
       - { key: 'net.ipv4.tcp_syncookies', value: '1' }
   ```

**Handlers (roles/security/handlers/main.yml):**
```yaml
- name: restart sshd
  systemd:
    name: sshd
    state: restarted

- name: restart fail2ban
  systemd:
    name: fail2ban
    state: restarted
```

### Updates Role

Manages system package updates.

**Tasks (roles/updates/tasks/main.yml):**

```yaml
- name: Update apt cache
  apt:
    update_cache: yes
  changed_when: false
  tags: ['updates']

- name: Upgrade all packages
  apt:
    upgrade: dist                    # Full distribution upgrade
    update_cache: yes
    autoremove: yes                  # Remove unused packages
    autoclean: yes                   # Clean package cache
  register: upgrade_result
  tags: ['updates']

- name: Display upgrade results
  debug:
    msg: "{{ upgrade_result.stdout_lines }}"
  when: upgrade_result.stdout_lines is defined
  tags: ['updates']

- name: Clean up package cache
  apt:
    autoclean: yes
    autoremove: yes
  tags: ['updates', 'cleanup']
```

## Variables

### group_vars/all.yml

Central configuration for all servers.

```yaml
---
# System Configuration
system_timezone: "Europe/Amsterdam"

# Common Packages
common_packages:
  - vim                    # Text editor
  - htop                   # Process monitor
  - curl                   # HTTP client
  - wget                   # File downloader
  - git                    # Version control
  - net-tools              # Network utilities (ifconfig, netstat)
  - tree                   # Directory tree viewer
  - unzip                  # Archive extraction
  - software-properties-common  # PPA management
  - apt-transport-https    # HTTPS support for apt
  - ca-certificates        # SSL certificates
  - gnupg                  # GPG encryption
  - lsb-release           # Linux Standard Base info
  - python3-pip           # Python package manager
  - jq                    # JSON processor
  - tmux                  # Terminal multiplexer
  - ncdu                  # Disk usage analyzer
  - docker.io             # Container runtime
  - docker-compose        # Container orchestration

# SSH Configuration
ssh_port: 22                         # SSH port (change for security)
ssh_permit_root_login: "no"          # Disable root SSH
ssh_password_authentication: "no"    # Force key-based auth
ssh_pubkey_authentication: "yes"     # Enable key auth

# Firewall Configuration
ufw_default_incoming: deny           # Block all incoming by default
ufw_default_outgoing: allow          # Allow all outgoing
ufw_allowed_services:                # Services to allow
  - ssh
  - http
  - https

# System Updates
auto_updates_enabled: true           # Enable automatic updates
reboot_after_updates: false          # Don't auto-reboot

# User Management
admin_users: []                      # List of admin users to create
# Example:
# admin_users:
#   - username: admin
#     shell: /bin/bash
#     groups: ['sudo', 'docker']
#     ssh_key: "ssh-rsa AAAA..."
```

**Variable Override Hierarchy:**
1. Command line: `-e "var=value"`
2. Host vars: `host_vars/docker01.yml`
3. Group vars: `group_vars/ubuntu_servers.yml`
4. Group vars: `group_vars/all.yml`

## Usage Examples

### Basic Operations

**Test Connectivity:**
```bash
# Ping all servers
ansible all -m ping

# Ping specific server
ansible docker01 -m ping

# Test with verbose output
ansible all -m ping -v
```

**Ad-hoc Commands:**
```bash
# Check disk space
ansible all -m shell -a "df -h"

# Check uptime
ansible all -m shell -a "uptime"

# Restart a service
ansible all -m service -a "name=nginx state=restarted" --become

# Update package cache
ansible all -m apt -a "update_cache=yes" --become
```

### Running Playbooks

**Bootstrap New Server:**
```bash
# Full bootstrap
ansible-playbook playbooks/bootstrap.yml

# Bootstrap specific server
ansible-playbook playbooks/bootstrap.yml --limit docker01

# Dry run (no changes)
ansible-playbook playbooks/bootstrap.yml --check

# With extra variables
ansible-playbook playbooks/bootstrap.yml -e "system_timezone=America/New_York"

# Verbose output
ansible-playbook playbooks/bootstrap.yml -vvv

# Start from specific task
ansible-playbook playbooks/bootstrap.yml --start-at-task="Install common packages"
```

**Update Servers:**
```bash
# Update all servers
ansible-playbook playbooks/update.yml

# Update with reboot
ansible-playbook playbooks/update.yml -e "reboot_after_updates=true"

# Update specific group
ansible-playbook playbooks/update.yml --limit ubuntu_servers

# Update with tags
ansible-playbook playbooks/update.yml --tags updates
```

**Using Tags:**
```bash
# Run only security tasks
ansible-playbook playbooks/bootstrap.yml --tags security

# Skip security tasks
ansible-playbook playbooks/bootstrap.yml --skip-tags security

# Multiple tags
ansible-playbook playbooks/bootstrap.yml --tags "common,packages"

# List available tags
ansible-playbook playbooks/bootstrap.yml --list-tags
```

### Advanced Operations

**Limit to Specific Hosts:**
```bash
# Single host
ansible-playbook playbooks/bootstrap.yml --limit docker01

# Multiple hosts
ansible-playbook playbooks/bootstrap.yml --limit docker01,docker02

# All except one
ansible-playbook playbooks/bootstrap.yml --limit 'all:!docker01'

# Using patterns
ansible-playbook playbooks/bootstrap.yml --limit 'docker*'
```

**Debugging:**
```bash
# Verbose output levels
ansible-playbook playbooks/bootstrap.yml -v      # verbose
ansible-playbook playbooks/bootstrap.yml -vv     # more verbose
ansible-playbook playbooks/bootstrap.yml -vvv    # very verbose
ansible-playbook playbooks/bootstrap.yml -vvvv   # debug level

# Syntax check
ansible-playbook playbooks/bootstrap.yml --syntax-check

# List hosts
ansible-playbook playbooks/bootstrap.yml --list-hosts

# List tasks
ansible-playbook playbooks/bootstrap.yml --list-tasks
```

**Parallel Execution:**
```bash
# Run on 5 hosts at a time
ansible-playbook playbooks/bootstrap.yml --forks 5

# Serial execution (one at a time)
ansible-playbook playbooks/bootstrap.yml --forks 1
```

## Security Considerations

### SSH Key Management

**Generate SSH Key Pair:**
```bash
ssh-keygen -t ed25519 -C "ansible@homelab" -f ~/.ssh/homelab_ansible
```

**Copy Key to Server:**
```bash
ssh-copy-id -i ~/.ssh/homelab_ansible.pub ubuntu@docker01.ota.lan
```

**Configure SSH Agent:**
```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/homelab_ansible
```

### Ansible Vault for Secrets

**Create Encrypted Variables:**
```bash
# Create new vault file
ansible-vault create group_vars/vault.yml

# Edit existing vault
ansible-vault edit group_vars/vault.yml

# Encrypt existing file
ansible-vault encrypt group_vars/secrets.yml
```

**Use Vault in Playbooks:**
```bash
# Run with vault password
ansible-playbook playbooks/bootstrap.yml --ask-vault-pass

# Use vault password file
ansible-playbook playbooks/bootstrap.yml --vault-password-file ~/.vault_pass
```

**Example Vault Variables:**
```yaml
# group_vars/vault.yml (encrypted)
vault_ssh_password: "secret_password"
vault_sudo_password: "sudo_secret"
vault_api_key: "api_key_here"
```

### Firewall Rules

**Custom Port Rules:**
```yaml
# group_vars/all.yml
custom_firewall_rules:
  - { port: 8080, proto: tcp, comment: "Web app" }
  - { port: 3306, proto: tcp, source: "192.168.1.0/24", comment: "MySQL from local network" }
```

**Apply Rules in Security Role:**
```yaml
- name: Allow custom ports
  ufw:
    rule: allow
    port: "{{ item.port }}"
    proto: "{{ item.proto }}"
    from_ip: "{{ item.source | default('any') }}"
    comment: "{{ item.comment }}"
  loop: "{{ custom_firewall_rules | default([]) }}"
```

### Audit and Compliance

**Check Security Status:**
```bash
# Verify SSH configuration
ansible all -m shell -a "grep -E '^(PermitRootLogin|PasswordAuthentication)' /etc/ssh/sshd_config" --become

# Check firewall status
ansible all -m shell -a "ufw status verbose" --become

# List open ports
ansible all -m shell -a "ss -tulpn" --become

# Check fail2ban status
ansible all -m shell -a "fail2ban-client status sshd" --become
```

**Logging:**
```yaml
# ansible.cfg
[defaults]
log_path = /var/log/ansible/ansible.log
```

## Troubleshooting

### Common Issues

**SSH Connection Failures:**
```bash
# Test SSH connectivity
ssh -vvv ubuntu@docker01.ota.lan

# Check SSH key permissions
chmod 600 ~/.ssh/homelab_ansible
chmod 700 ~/.ssh

# Verify SSH agent
ssh-add -l
```

**Permission Denied:**
```bash
# Check sudo configuration
ansible all -m shell -a "sudo -l" -u ubuntu

# Test with become
ansible all -m shell -a "whoami" --become
```

**Module Failures:**
```bash
# Check Python installation
ansible all -m shell -a "which python3"

# Verify Ansible collections
ansible-galaxy collection list
```

### Best Practices

1. **Version Control:** Always commit configuration changes
2. **Testing:** Use `--check` mode before applying changes
3. **Backups:** Backup critical configurations before changes
4. **Documentation:** Document custom variables and configurations
5. **Tags:** Use tags for selective task execution
6. **Idempotency:** Ensure playbooks can run multiple times safely
7. **Error Handling:** Use `failed_when` and `changed_when` appropriately
8. **Secrets Management:** Never commit sensitive data, use Ansible Vault
9. **Modularity:** Keep roles focused and reusable
10. **Testing:** Test on non-production systems first

## Extending the Framework

### Adding New Roles

```bash
# Create role structure
mkdir -p roles/monitoring/{tasks,handlers,templates,defaults}

# Create main task file
cat > roles/monitoring/tasks/main.yml <<EOF
---
- name: Install monitoring tools
  apt:
    name:
      - prometheus-node-exporter
      - collectd
    state: present
  tags: ['monitoring']
EOF
```

### Creating Host-Specific Variables

```yaml
# host_vars/docker01.yml
---
# Docker-specific settings
docker_daemon_options:
  log-driver: "json-file"
  log-opts:
    max-size: "10m"
    max-file: "3"

# Extra packages for docker host
extra_packages:
  - docker-compose-plugin
  - containerd
```

### Custom Playbooks

```yaml
# playbooks/docker-setup.yml
---
- name: Configure Docker Hosts
  hosts: docker01
  become: yes

  tasks:
    - name: Configure Docker daemon
      copy:
        content: "{{ docker_daemon_options | to_nice_json }}"
        dest: /etc/docker/daemon.json
      notify: restart docker

  handlers:
    - name: restart docker
      systemd:
        name: docker
        state: restarted
```

## Reference

### Useful Commands Cheat Sheet

```bash
# Inventory
ansible-inventory --list                 # List all hosts
ansible-inventory --graph                # Show host graph
ansible-inventory --host docker01        # Show host variables

# Facts
ansible docker01 -m setup                # Gather all facts
ansible docker01 -m setup -a "filter=ansible_distribution*"  # Filter facts

# Testing
ansible-playbook playbooks/bootstrap.yml --syntax-check
ansible-playbook playbooks/bootstrap.yml --check --diff
ansible-playbook playbooks/bootstrap.yml --list-tasks
ansible-playbook playbooks/bootstrap.yml --step  # Confirm each task

# Documentation
ansible-doc apt                          # Module documentation
ansible-doc -l                          # List all modules
ansible-doc -t connection -l            # List connection plugins
```

### File Locations

- **Ansible Config:** `/etc/ansible/ansible.cfg` or `~/.ansible.cfg` or `./ansible.cfg`
- **Logs:** `/var/log/ansible/ansible.log` (if configured)
- **Galaxy Collections:** `~/.ansible/collections/`
- **Facts Cache:** `/tmp/ansible_facts/` (as configured)

## Support and Contribution

For issues, questions, or contributions, please refer to the project repository.

**Author:** silentijsje
**License:** MIT (or specify your license)
**Version:** 1.0
