# Ansible EC2 Hardening Runbook

> **What this is:** A step-by-step guide for using Ansible to automatically apply security hardening settings to AWS EC2 instances. Written for people who are learning Ansible and want to understand *why* each step exists, not just what to type.

---

## What Problem Does This Solve?

Imagine you have 20 EC2 instances running in AWS. Each one needs the same security settings: SSH configured correctly, unnecessary services disabled, firewall rules applied, audit logging enabled. Doing this manually on each server is:

- **Slow** — SSH into each box, run commands, repeat
- **Error-prone** — you'll miss a step on at least one server
- **Not auditable** — no record of what was changed or when

Ansible solves this by letting you describe the *desired state* of your servers in a file called a **playbook**, then applying that state to all your servers at once. One command, all 20 servers, consistent results.

---

## Core Concepts (Read Before Continuing)

| Term | What It Means |
|------|---------------|
| **Playbook** | A YAML file describing what you want Ansible to do |
| **Inventory** | The list of servers Ansible will target |
| **Task** | A single action inside a playbook (e.g., "disable SSH root login") |
| **Module** | The built-in Ansible tool used to run a task (e.g., `lineinfile`, `service`) |
| **Handler** | A task that only runs when triggered by a change (e.g., restart SSH after config changes) |
| **Idempotent** | Running the playbook multiple times produces the same result — no unintended side effects |

> **Why idempotency matters:** You can safely re-run this playbook at any time. If the setting is already correct, Ansible skips it. This makes it safe to run on a schedule as a compliance check.

---

## Prerequisites

### On your control machine (where you run Ansible from):
```bash
# Install Ansible
pip install ansible

# Install AWS collection for dynamic inventory
ansible-galaxy collection install amazon.aws

# Verify installation
ansible --version
```

### On your AWS environment:
- EC2 instances running Amazon Linux 2 or Ubuntu 22.04
- A key pair (.pem file) for SSH access
- IAM role or credentials with EC2 read permissions (for dynamic inventory)
- Security group allowing inbound SSH (port 22) from your control machine

> **Why a key pair?** Ansible connects to servers over SSH. It needs your private key to authenticate without a password. Keep your .pem file secure and never commit it to Git.

---

## Project Structure

```
ansible-ec2-hardening/
├── inventory/
│   ├── aws_ec2.yml          # Dynamic inventory — auto-discovers EC2 instances
│   └── hosts.ini            # Static inventory — manual list (used for local testing)
├── roles/
│   └── hardening/
│       ├── tasks/
│       │   └── main.yml     # The actual hardening steps
│       ├── handlers/
│       │   └── main.yml     # Actions triggered by changes (e.g., restart sshd)
│       └── defaults/
│           └── main.yml     # Default variable values (easily overridden)
├── hardening.yml            # Main playbook — ties everything together
├── ansible.cfg              # Ansible configuration
└── README.md
```

> **Why use roles?** Roles let you organize tasks into reusable, self-contained units. Instead of one giant playbook file, you have a clean structure that's easy to maintain and share. Industry standard practice.

---

## Step 1: Configure Inventory

### Option A — Dynamic Inventory (Recommended for AWS)

Dynamic inventory automatically discovers your EC2 instances instead of you manually listing IPs. Ansible queries the AWS API and groups instances by tags, region, or instance type.

**`inventory/aws_ec2.yml`**
```yaml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2

# Group instances by their "Environment" tag
keyed_groups:
  - key: tags.Environment
    prefix: env

# Only target running instances
filters:
  instance-state-name: running

# How Ansible will connect to each instance
compose:
  ansible_host: public_ip_address
```

> **Why dynamic inventory?** In real cloud environments, server IPs change constantly — instances are launched and terminated all the time. Dynamic inventory adapts automatically. You never have to manually update a list of IPs.

### Option B — Static Inventory (For Testing)

**`inventory/hosts.ini`**
```ini
[web_servers]
10.0.1.10
10.0.1.11

[db_servers]
10.0.2.20

[all:vars]
ansible_user=ec2-user
ansible_ssh_private_key_file=~/.ssh/your-key.pem
```

---

## Step 2: Configure ansible.cfg

**`ansible.cfg`**
```ini
[defaults]
inventory = inventory/aws_ec2.yml
remote_user = ec2-user
private_key_file = ~/.ssh/your-key.pem
host_key_checking = False

# How many servers to work on simultaneously
forks = 10

[ssh_connection]
# Reuse SSH connections for speed (important at scale)
pipelining = True
```

> **Why `host_key_checking = False`?** In AWS, new instances have unknown SSH fingerprints. Disabling this check prevents Ansible from prompting you to accept each server's fingerprint. In a production environment, you'd want a more controlled approach, but for ops automation this is standard practice.

---

## Step 3: Write the Hardening Tasks

**`roles/hardening/tasks/main.yml`**

```yaml
---
# ============================================================
# SSH HARDENING
# Why: SSH is the primary attack surface for remote servers.
# Locking it down significantly reduces risk.
# ============================================================

- name: Disable SSH root login
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PermitRootLogin'
    line: 'PermitRootLogin no'
    state: present
  notify: restart sshd
  # Why: Root has unlimited privileges. If an attacker gets root SSH access,
  # it's game over. Force all logins through a regular user account.

- name: Disable password authentication (key-based only)
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^PasswordAuthentication'
    line: 'PasswordAuthentication no'
    state: present
  notify: restart sshd
  # Why: Passwords can be brute-forced. SSH keys are cryptographically
  # strong and not vulnerable to brute force attacks.

- name: Set SSH idle timeout to 5 minutes
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^ClientAliveInterval'
    line: 'ClientAliveInterval 300'
    state: present
  notify: restart sshd
  # Why: Unattended SSH sessions are a risk. If someone walks away from
  # an open terminal, this ensures the session closes automatically.

- name: Limit SSH to specific users
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^AllowUsers'
    line: 'AllowUsers ec2-user ansible-svc'
    state: present
  notify: restart sshd
  # Why: Allowlisting users means even if an attacker creates a new account,
  # they can't SSH in with it.

# ============================================================
# FIREWALL CONFIGURATION
# Why: Only open ports that your application actually needs.
# Everything else should be blocked by default.
# ============================================================

- name: Install firewalld
  package:
    name: firewalld
    state: present

- name: Start and enable firewalld
  service:
    name: firewalld
    state: started
    enabled: yes
  # Why: firewalld persists across reboots (enabled: yes). Without this,
  # your firewall rules disappear when the instance restarts.

- name: Allow SSH through firewall
  firewalld:
    service: ssh
    permanent: yes
    state: enabled
    immediate: yes

- name: Allow HTTP traffic (if web server)
  firewalld:
    service: http
    permanent: yes
    state: enabled
    immediate: yes
  when: "'web_servers' in group_names"
  # Why: The 'when' condition means this only runs on servers in the
  # web_servers inventory group. DB servers won't get port 80 opened.

- name: Block all other inbound traffic
  firewalld:
    zone: public
    target: DROP
    permanent: yes
    state: present
  notify: reload firewalld
  # Why: Default-deny means any new port is blocked until explicitly opened.
  # This is the correct security posture.

# ============================================================
# SYSTEM UPDATES
# Why: Unpatched systems are the #1 attack vector.
# ============================================================

- name: Apply all security patches
  package:
    name: '*'
    state: latest
    security: yes
  # Why: 'security: yes' limits updates to security patches only,
  # avoiding potentially breaking application updates.

# ============================================================
# AUDIT LOGGING
# Why: You need to know WHO did WHAT and WHEN on your servers.
# Required for compliance (PCI-DSS, SOC2, CIS benchmarks).
# ============================================================

- name: Install auditd
  package:
    name: audit
    state: present

- name: Start and enable auditd
  service:
    name: auditd
    state: started
    enabled: yes

- name: Configure audit rules for privileged commands
  copy:
    dest: /etc/audit/rules.d/hardening.rules
    content: |
      # Log all privilege escalation (sudo usage)
      -a always,exit -F arch=b64 -S execve -F euid=0 -k privileged

      # Log changes to user accounts
      -w /etc/passwd -p wa -k identity
      -w /etc/shadow -p wa -k identity
      -w /etc/sudoers -p wa -k sudoers

      # Log SSH authentication events
      -w /var/log/secure -p wa -k auth
    mode: '0640'
  notify: restart auditd
  # Why: These rules create a paper trail. If an attacker escalates privileges
  # or modifies user accounts, auditd captures it. Critical for incident response.

# ============================================================
# DISABLE UNNECESSARY SERVICES
# Why: Every running service is a potential attack surface.
# If you don't need it, turn it off.
# ============================================================

- name: Disable unnecessary services
  service:
    name: "{{ item }}"
    state: stopped
    enabled: no
  loop:
    - bluetooth
    - cups        # Printing service — not needed on a server
    - avahi-daemon  # Network discovery — not needed on a server
  ignore_errors: yes
  # Why: ignore_errors because some services may not exist on all distros.
  # This is safe — if the service isn't there, nothing happens.

# ============================================================
# FILE PERMISSIONS
# Why: Incorrect permissions on sensitive files can expose
# password hashes and configuration to unauthorized users.
# ============================================================

- name: Secure /etc/passwd permissions
  file:
    path: /etc/passwd
    owner: root
    group: root
    mode: '0644'

- name: Secure /etc/shadow permissions
  file:
    path: /etc/shadow
    owner: root
    group: shadow
    mode: '0640'
  # Why: /etc/shadow contains password hashes. Only root and shadow group
  # should be able to read it. 0640 = root rw, shadow r, others nothing.
```

---

## Step 4: Define Handlers

**`roles/hardening/handlers/main.yml`**

```yaml
---
- name: restart sshd
  service:
    name: sshd
    state: restarted
  # Triggered by SSH config changes. Handlers only run once at the end
  # of the play, even if notified multiple times. This prevents sshd
  # from restarting after every single SSH setting change.

- name: reload firewalld
  service:
    name: firewalld
    state: reloaded

- name: restart auditd
  service:
    name: auditd
    state: restarted
```

> **Why handlers?** If three tasks all notify `restart sshd`, Ansible only restarts it once at the end. Without handlers, you'd restart it three times unnecessarily — or forget to restart it at all.

---

## Step 5: Write the Main Playbook

**`hardening.yml`**

```yaml
---
- name: EC2 Security Hardening
  hosts: all          # Target all hosts in inventory (filter with --limit)
  become: yes         # Run tasks with sudo (required for system changes)
  become_method: sudo

  roles:
    - hardening

  pre_tasks:
    - name: Gather facts about the system
      setup:
      # Why: 'setup' collects system info (OS type, IP, hostname).
      # Used for conditional logic like 'when: ansible_os_family == "RedHat"'

    - name: Confirm we can reach the host
      ping:
      # Why: Fail fast. Better to know immediately that a host is unreachable
      # than to find out mid-playbook after some tasks already ran.

  post_tasks:
    - name: Log hardening completion
      shell: echo "Hardening completed on $(hostname) at $(date)" >> /var/log/ansible-hardening.log
      # Why: Creates an audit trail directly on each server showing when
      # hardening was last applied.
```

---

## Running the Playbook

### Dry run first (always do this)
```bash
ansible-playbook hardening.yml --check --diff
```
> `--check` simulates the run without making changes. `--diff` shows exactly what would change. **Always dry run before touching production.**

### Target specific servers
```bash
# Only run on servers tagged Environment=production
ansible-playbook hardening.yml --limit env_production

# Only run on one server (useful for testing)
ansible-playbook hardening.yml --limit 10.0.1.10
```

### Run for real
```bash
ansible-playbook hardening.yml
```

### Run with verbose output (for debugging)
```bash
ansible-playbook hardening.yml -vvv
```

---

## Verification: Did It Work?

After the playbook runs, verify on a target server:

```bash
# SSH in and check
ssh -i ~/.ssh/your-key.pem ec2-user@<instance-ip>

# Verify SSH config
sudo grep "PermitRootLogin\|PasswordAuthentication" /etc/ssh/sshd_config

# Check firewall rules
sudo firewall-cmd --list-all

# Check audit is running
sudo systemctl status auditd

# Check the hardening log
cat /var/log/ansible-hardening.log
```

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| `UNREACHABLE! SSH connection refused` | Wrong IP or security group not open | Verify SG allows port 22 from your IP |
| `Permission denied (publickey)` | Wrong key file or wrong user | Check `ansible_user` and `private_key_file` in ansible.cfg |
| `sudo: a password is required` | EC2 user doesn't have passwordless sudo | Add `ansible_become_pass` or configure sudoers |
| `No inventory was parsed` | aws_ec2 plugin not installed | Run `ansible-galaxy collection install amazon.aws` |
| `handler not found` | Typo in handler name | Handler name in `notify:` must exactly match handler `name:` |

---

## Rollback Procedure

Ansible doesn't have a built-in "undo." If hardening breaks something:

1. **SSH still works?** Re-run playbook with corrected values
2. **SSH broken?** Use AWS Systems Manager Session Manager to access the instance without SSH
3. **Nuclear option:** Terminate the instance and launch a fresh one from your base AMI

> **Best practice:** Take an AMI snapshot before running hardening on production instances. Recovery takes 5 minutes instead of hours.

---

## Alignment with CIS Benchmarks

This runbook addresses several CIS Amazon Linux 2 Benchmark controls:

| CIS Control | This Runbook Covers |
|-------------|---------------------|
| 5.2.8 | Disable SSH root login |
| 5.2.9 | Disable SSH password auth |
| 5.2.16 | Configure SSH idle timeout |
| 3.4.1 | Install firewalld |
| 4.1.1 | Configure auditd |
| 6.1.2 | Secure /etc/passwd permissions |

> CIS Benchmarks are the industry standard for system hardening. Being able to reference these in an interview or audit shows professional-level understanding of compliance.

---

*Part of the [aws-ops-runbooks](../README.md) repository — see also the [Jenkins CI/CD Runbook](../jenkins/cicd-pipeline-runbook.md).*
