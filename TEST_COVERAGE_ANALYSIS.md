# Test Coverage Analysis

## Executive Summary

This document analyzes the current test coverage of the homelab-ai-test Ansible codebase and proposes improvements to ensure reliability, prevent regressions, and catch issues before deployment to production infrastructure.

**Current State: No formal testing infrastructure exists.**

---

## Current Testing Situation

### What Exists Today

1. **Inline Validation Only**
   - SSH config validation: `validate: '/usr/sbin/sshd -t -f %s'` (roles/security/tasks/main.yml:10)
   - Samba config validation: `validate: 'testparm -s %s'` (roles/smb_share/tasks/main.yml:127)
   - Reboot verification: `test_command: uptime` (playbooks/update.yml:78)

2. **Manual Testing Methods**
   - Ansible's `--check` flag for dry-run mode
   - Manual `ansible all -m ping` connectivity tests

### What's Missing

- No automated test framework (Molecule, pytest, etc.)
- No YAML/Ansible linting (ansible-lint, yamllint)
- No CI/CD pipeline
- No integration tests
- No idempotency verification
- No role unit tests

---

## Critical Areas Requiring Test Coverage

### Priority 1: High-Risk Security Tasks

**Location:** `roles/security/tasks/main.yml`

| Task | Risk Level | Why It Needs Testing |
|------|------------|---------------------|
| SSH daemon configuration (lines 4-21) | **Critical** | Misconfiguration can lock out all users or expose server |
| UFW firewall rules (lines 23-57) | **Critical** | Wrong rules can block legitimate traffic or expose services |
| fail2ban configuration (lines 59-87) | **High** | Incorrect config can fail to protect or block legitimate users |
| sysctl security parameters (lines 124-139) | **High** | Network hardening affects all connectivity |

**Recommended Tests:**
- Verify SSH settings are applied correctly
- Verify only expected ports are open after UFW configuration
- Test that fail2ban bans work as expected
- Validate sysctl values are set correctly

### Priority 2: User/Group Management

**Location:** `roles/smb_share/tasks/main.yml` (lines 21-106)

| Task | Risk Level | Why It Needs Testing |
|------|------------|---------------------|
| UID/GID mismatch detection (lines 21-37) | **High** | Complex conditional logic prone to edge cases |
| User deletion with `pkill -9` (lines 39-46) | **Critical** | Can terminate critical processes if logic is wrong |
| User/group recreation (lines 48-90) | **High** | Must maintain correct ownership across systems |
| Samba password setting (lines 92-99) | **Medium** | Sensitive operation with vault integration |

**Recommended Tests:**
- Test user creation with specific UID/GID
- Test handling of existing users with mismatched UID
- Test idempotency (running twice produces same result)
- Verify group membership is correct

### Priority 3: Template Rendering

**Location:** `roles/smb_share/templates/smb.conf.j2`

| Concern | Risk Level | Why It Needs Testing |
|---------|------------|---------------------|
| Jinja2 conditionals (lines 42-47) | **Medium** | hosts_allow/deny only render when defined |
| Permission logic (lines 33-37) | **Medium** | rw/ro permission mapping |
| Variable defaults (lines 5-17, 31-41) | **Low** | Ensure defaults work when optional vars missing |

**Recommended Tests:**
- Test template with minimal variables
- Test template with all optional variables
- Test template with multiple shares
- Verify generated config passes `testparm`

### Priority 4: Update Playbook Reliability

**Location:** `playbooks/update.yml`

| Task | Risk Level | Why It Needs Testing |
|------|------------|---------------------|
| Automatic reboot logic (lines 71-80) | **High** | Unexpected reboots can cause service disruption |
| apt autopurge command (lines 50-53) | **Medium** | `changed_when` logic may misreport |
| Distro upgrade (lines 36-40) | **High** | Major upgrades can break systems |

**Recommended Tests:**
- Verify reboot only triggers when `/var/run/reboot-required` exists
- Test `changed_when` accurately reflects state
- Test idempotency of update tasks

### Priority 5: Common Role Idempotency

**Location:** `roles/common/tasks/main.yml`

| Task | Risk Level | Why It Needs Testing |
|------|------------|---------------------|
| Hostname configuration (lines 24-34) | **Low** | Edge cases with /etc/hosts |
| Service disabling with `ignore_errors` (lines 48-56) | **Low** | Silently ignores failures |

---

## Proposed Testing Framework

### 1. Ansible Lint (Static Analysis)

**Installation:**
```bash
pip install ansible-lint yamllint
```

**Configuration (`.ansible-lint`):**
```yaml
profile: production
exclude_paths:
  - .cache/
  - .git/
warn_list:
  - experimental
```

**Benefits:**
- Catches syntax errors before execution
- Enforces best practices
- Identifies deprecated modules
- Detects security issues

### 2. Molecule (Integration Testing)

**Installation:**
```bash
pip install molecule molecule-plugins[docker]
```

**Recommended Structure:**
```
roles/
  security/
    molecule/
      default/
        molecule.yml
        converge.yml
        verify.yml
  smb_share/
    molecule/
      default/
        molecule.yml
        converge.yml
        verify.yml
```

**Example `molecule.yml` for security role:**
```yaml
dependency:
  name: galaxy
driver:
  name: docker
platforms:
  - name: ubuntu-jammy
    image: ubuntu:22.04
    pre_build_image: true
    privileged: true
    command: /sbin/init
provisioner:
  name: ansible
verifier:
  name: ansible
```

**Example `verify.yml` for security role:**
```yaml
---
- name: Verify security configuration
  hosts: all
  gather_facts: false
  tasks:
    - name: Check SSH configuration
      lineinfile:
        path: /etc/ssh/sshd_config
        line: "PermitRootLogin no"
        state: present
      check_mode: yes
      register: ssh_check
      failed_when: ssh_check.changed

    - name: Verify UFW is enabled
      command: ufw status
      register: ufw_status
      changed_when: false
      failed_when: "'Status: active' not in ufw_status.stdout"

    - name: Check fail2ban is running
      service:
        name: fail2ban
        state: started
      check_mode: yes
      register: fail2ban_check
      failed_when: fail2ban_check.changed
```

### 3. CI/CD Pipeline (GitHub Actions)

**Example `.github/workflows/ansible-test.yml`:**
```yaml
name: Ansible Tests

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install ansible ansible-lint yamllint

      - name: Run yamllint
        run: yamllint .

      - name: Run ansible-lint
        run: ansible-lint

  molecule:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        role:
          - common
          - security
          - updates
          - smb_share
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'

      - name: Install dependencies
        run: |
          pip install ansible molecule molecule-plugins[docker]

      - name: Run Molecule tests
        run: |
          cd roles/${{ matrix.role }}
          molecule test
```

---

## Implementation Roadmap

### Phase 1: Static Analysis (Immediate)

1. Add `.ansible-lint` configuration
2. Add `.yamllint` configuration
3. Fix any linting errors
4. Add pre-commit hooks

**Estimated effort:** 1-2 hours

### Phase 2: Security Role Tests (Short-term)

1. Create Molecule scenario for security role
2. Write verification tests for:
   - SSH hardening
   - UFW configuration
   - fail2ban setup
   - sysctl parameters

**Estimated effort:** 4-6 hours

### Phase 3: SMB Share Role Tests (Short-term)

1. Create Molecule scenario for smb_share role
2. Write tests for:
   - User/group creation
   - UID/GID mismatch handling
   - Template rendering
   - Share accessibility

**Estimated effort:** 4-6 hours

### Phase 4: Full CI/CD Pipeline (Medium-term)

1. Set up GitHub Actions workflow
2. Add branch protection rules
3. Require passing tests before merge

**Estimated effort:** 2-3 hours

### Phase 5: Integration Tests (Long-term)

1. Create end-to-end test scenarios
2. Test full bootstrap workflow
3. Test update workflow with simulated reboot

**Estimated effort:** 8-12 hours

---

## Specific Test Cases to Implement

### Security Role Test Cases

```yaml
# tests/security_test_cases.yml
test_cases:
  ssh_hardening:
    - verify_root_login_disabled
    - verify_password_auth_disabled
    - verify_pubkey_auth_enabled
    - verify_custom_port_configured
    - verify_max_auth_tries_limited

  firewall:
    - verify_ufw_installed
    - verify_default_deny_incoming
    - verify_default_allow_outgoing
    - verify_ssh_port_allowed
    - verify_only_configured_services_allowed

  fail2ban:
    - verify_fail2ban_installed
    - verify_fail2ban_running
    - verify_sshd_jail_enabled
    - verify_custom_port_in_jail

  sysctl:
    - verify_icmp_broadcast_ignored
    - verify_syn_cookies_enabled
    - verify_source_routing_disabled
    - verify_redirects_disabled
```

### SMB Share Role Test Cases

```yaml
# tests/smb_share_test_cases.yml
test_cases:
  user_management:
    - create_user_with_specific_uid
    - create_group_with_specific_gid
    - handle_existing_user_correct_uid
    - handle_existing_user_wrong_uid
    - handle_existing_group_wrong_gid
    - verify_user_has_nologin_shell

  samba_config:
    - verify_smb_conf_syntax
    - verify_share_permissions
    - verify_valid_users_set
    - verify_force_user_group

  directory_setup:
    - verify_share_directory_exists
    - verify_correct_ownership
    - verify_correct_permissions

  idempotency:
    - run_twice_no_changes
    - run_after_manual_modification
```

---

## Risk Assessment Without Tests

| Scenario | Likelihood | Impact | Risk Level |
|----------|------------|--------|------------|
| SSH misconfiguration locks out admin | Medium | **Critical** | **High** |
| Firewall blocks legitimate traffic | Medium | High | **High** |
| User recreation breaks file ownership | Low | High | Medium |
| Update playbook causes unexpected reboot | Medium | Medium | Medium |
| Samba config syntax error breaks shares | Low | Medium | Low |
| Template renders incorrect permissions | Low | Medium | Low |

---

## Conclusion

The current codebase has **zero automated test coverage**, relying entirely on manual testing and Ansible's built-in validation. Given that this infrastructure manages:

- SSH access (critical for server management)
- Firewall rules (security boundary)
- User/group management (access control)
- Automatic system updates and reboots

**Implementing comprehensive testing is essential** to prevent configuration drift, catch regressions, and ensure changes don't break production infrastructure.

The recommended approach is to start with **static analysis (ansible-lint)** for immediate value, then progressively add **Molecule tests** for the highest-risk roles (security, smb_share), and finally implement a **CI/CD pipeline** to enforce quality gates.
