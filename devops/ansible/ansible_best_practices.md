# Ansible Best Practices (FAANG-Level Engineering)

**Related:** [Ansible Fundamentals](ansible_fundamentals.md) | [Playbooks & Roles](playbooks_and_roles.md) | [Variables & Templates](ansible_variables_and_templates.md) | [GitHub Actions CI/CD](../github/github_actions_cicd.md)

---

## 1. Idempotency — The Most Important Concept

Every Ansible task should produce the **same result** whether you run it once or 100 times. If the desired state already exists, nothing changes.

```yaml
# IDEMPOTENT: apt checks if nginx is installed before doing anything
- name: Install nginx
  apt:
    name: nginx
    state: present
  # Run 1: installs nginx → changed
  # Run 2: nginx exists → ok (no change)

# NOT IDEMPOTENT: shell always runs
- name: Install nginx (BAD)
  shell: apt-get install -y nginx
  # Run 1: installs → changed
  # Run 2: installs again → changed (even though it's already installed)

# FIX: make shell idempotent with creates/removes
- name: Download binary (idempotent)
  shell: curl -o /usr/local/bin/tool https://example.com/tool
  args:
    creates: /usr/local/bin/tool    # skip if file already exists
```

### How to Ensure Idempotency

```yaml
# BAD: always changes, appends on every run
- name: Add line to config
  shell: echo "export PATH=/opt/bin:$PATH" >> /etc/profile

# GOOD: lineinfile is idempotent
- name: Add line to config
  lineinfile:
    path: /etc/profile
    line: "export PATH=/opt/bin:$PATH"
    state: present

# BAD: always restarts
- name: Restart nginx
  command: systemctl restart nginx

# GOOD: only starts if not running, only restarts when notified
- name: Ensure nginx running
  service:
    name: nginx
    state: started
  # Use handlers with notify for conditional restart
```

**Test for idempotency:** Run the playbook twice. The second run should show `changed=0`.

```bash
ansible-playbook site.yml    # first run: changed=15
ansible-playbook site.yml    # second run: changed=0 (idempotent!)
```

---

## 2. Directory Structure Conventions

### Standard Project Layout

```
ansible-project/
├── ansible.cfg                     # project-level config
├── requirements.yml                # Galaxy roles/collections
├── site.yml                        # master playbook
├── webservers.yml                  # playbook per tier
├── databases.yml
├── inventory/
│   ├── production/
│   │   ├── hosts.yml               # production inventory
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── vars.yml        # unencrypted vars
│   │   │   │   └── vault.yml       # encrypted vars
│   │   │   ├── webservers.yml
│   │   │   └── databases.yml
│   │   └── host_vars/
│   │       └── web1.example.com.yml
│   └── staging/
│       ├── hosts.yml
│       └── group_vars/
│           └── all/
│               ├── vars.yml
│               └── vault.yml
├── roles/
│   ├── common/
│   ├── nginx/
│   ├── postgresql/
│   └── app_deploy/
├── playbooks/                      # additional playbooks
│   ├── rolling_deploy.yml
│   └── database_backup.yml
├── library/                        # custom modules
├── filter_plugins/                 # custom Jinja2 filters
└── files/                          # static files
```

**Key principles:**
- One inventory directory per environment (production, staging, dev)
- Separate encrypted vault files from plain variables
- One role per concern (not one giant role)
- Master `site.yml` imports per-tier playbooks

```yaml
# site.yml — imports everything
---
- import_playbook: webservers.yml
- import_playbook: databases.yml
- import_playbook: monitoring.yml
```

---

## 3. Role Design Patterns

### Single Responsibility

```yaml
# BAD: one role does everything
roles/
└── webserver/          # installs OS packages AND nginx AND app AND monitoring

# GOOD: each role does one thing
roles/
├── common/             # base OS config, users, SSH
├── nginx/              # nginx installation and config
├── app_deploy/         # application deployment
└── monitoring_agent/   # monitoring setup
```

### Parameterized Roles with Defaults

```yaml
# roles/nginx/defaults/main.yml — sensible defaults
nginx_worker_processes: "{{ ansible_processor_vcpus }}"
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_client_max_body_size: 10m
nginx_ssl_enabled: false
nginx_ssl_cert: ""
nginx_ssl_key: ""
nginx_log_format: combined
nginx_sites: []
```

```yaml
# Using the role with overrides
- hosts: webservers
  roles:
    - role: nginx
      vars:
        nginx_ssl_enabled: true
        nginx_ssl_cert: /etc/ssl/certs/wildcard.pem
        nginx_ssl_key: /etc/ssl/private/wildcard.key
        nginx_sites:
          - name: myapp
            server_name: app.example.com
            upstream_port: 3000
```

### Role Dependency Management

```yaml
# roles/app_deploy/meta/main.yml
dependencies:
  - role: common
  - role: nginx
    vars:
      nginx_ssl_enabled: true
```

**Warning:** Role dependencies are tricky. A role dependency runs **before** the dependent role. If two roles depend on the same role, it runs twice (unless `allow_duplicates: false`).

---

## 4. Testing

### Ansible Lint

```bash
# Install
pip install ansible-lint

# Run
ansible-lint site.yml
ansible-lint roles/nginx/

# Common rules it checks:
# - YAML syntax
# - Deprecated syntax
# - Task naming conventions
# - Module usage best practices
# - Jinja2 spacing
```

```yaml
# .ansible-lint
skip_list:
  - yaml[line-length]
  - name[missing]

warn_list:
  - experimental

exclude_paths:
  - .cache/
  - .github/
```

### Molecule (Role Testing Framework)

Molecule creates ephemeral environments (Docker containers, VMs) to test roles.

```bash
# Initialize molecule scenario in a role
cd roles/nginx
molecule init scenario --driver-name docker

# Run the full test cycle
molecule test

# Test cycle steps:
# 1. create    → spin up Docker container
# 2. converge  → run the role against the container
# 3. idempotence → run the role again (verify changed=0)
# 4. verify    → run assertions (testinfra or Ansible verify)
# 5. destroy   → tear down the container
```

```yaml
# roles/nginx/molecule/default/molecule.yml
driver:
  name: docker
platforms:
  - name: ubuntu-22
    image: ubuntu:22.04
    pre_build_image: true
  - name: debian-12
    image: debian:12
    pre_build_image: true
provisioner:
  name: ansible
verifier:
  name: ansible
```

```yaml
# roles/nginx/molecule/default/converge.yml
---
- name: Converge
  hosts: all
  become: true
  roles:
    - role: nginx
      vars:
        nginx_ssl_enabled: false
```

```yaml
# roles/nginx/molecule/default/verify.yml
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Check nginx is installed
      command: nginx -v
      changed_when: false

    - name: Check nginx is running
      service_facts:

    - name: Assert nginx is running
      assert:
        that:
          - ansible_facts.services['nginx.service'].state == 'running'
```

---

## 5. Error Handling

### block / rescue / always

```yaml
tasks:
  - name: Deploy application
    block:
      - name: Download release
        get_url:
          url: "https://releases.example.com/app-{{ version }}.tar.gz"
          dest: /tmp/app.tar.gz

      - name: Extract and deploy
        unarchive:
          src: /tmp/app.tar.gz
          dest: /opt/app
          remote_src: true

      - name: Restart service
        service:
          name: myapp
          state: restarted

    rescue:
      - name: Rollback to previous version
        copy:
          src: /opt/app.backup/
          dest: /opt/app/
          remote_src: true

      - name: Restart with old version
        service:
          name: myapp
          state: restarted

      - name: Notify team
        slack:
          token: "{{ slack_token }}"
          msg: "Deployment of {{ version }} failed on {{ inventory_hostname }}"
          channel: "#deployments"

    always:
      - name: Clean up temp files
        file:
          path: /tmp/app.tar.gz
          state: absent

      - name: Verify app is responding
        uri:
          url: http://localhost:8080/health
          status_code: 200
        register: health_check
        retries: 5
        delay: 3
        until: health_check.status == 200
```

### Error Control

```yaml
# Ignore errors (task continues on failure)
- name: Check optional service
  command: systemctl status optional-service
  register: optional_result
  ignore_errors: true

# Custom failure condition
- name: Check disk space
  shell: df -h / | tail -1 | awk '{print $5}' | sed 's/%//'
  register: disk_usage
  failed_when: disk_usage.stdout | int > 90

# Custom changed condition
- name: Check app version
  command: /opt/app/bin/app --version
  register: app_version
  changed_when: false           # read-only command, never "changed"
```

### Retry Logic

```yaml
- name: Wait for service to be ready
  uri:
    url: http://localhost:8080/health
    status_code: 200
  register: result
  until: result.status == 200
  retries: 30                    # try 30 times
  delay: 10                     # wait 10 seconds between retries
```

---

## 6. Performance Optimization

### SSH Pipelining

The single biggest performance improvement. Instead of copying the module to the target, Ansible pipes it through the existing SSH connection.

```ini
# ansible.cfg
[ssh_connection]
pipelining = True
```

**Requirement:** `requiretty` must be disabled in sudoers on the target. This is the default on most modern systems.

**Performance impact:** 2-5x faster on SSH-heavy playbooks.

### Forks (Parallelism)

```ini
# ansible.cfg
[defaults]
forks = 50     # default is 5, increase for large inventories
```

**What's happening:** Ansible runs tasks on `forks` hosts simultaneously. With 100 hosts and `forks=50`, Ansible processes 50 hosts at a time, then the next 50.

### Fact Caching

```ini
# ansible.cfg
[defaults]
gathering = smart                          # gather only if facts are missing/expired
fact_caching = jsonfile                     # cache facts to files
fact_caching_connection = /tmp/ansible_facts
fact_caching_timeout = 3600                # cache for 1 hour
```

### Mitogen (Replacement SSH Connection Plugin)

Mitogen replaces Ansible's default SSH+SFTP module execution with a more efficient pure-Python approach. 2-7x speed improvement.

```bash
pip install mitogen
```

```ini
# ansible.cfg
[defaults]
strategy_plugins = /path/to/mitogen/ansible_mitogen/plugins/strategy
strategy = mitogen_linear
```

### Async Tasks

For long-running tasks that don't need to block:

```yaml
- name: Run long database migration
  command: /opt/app/migrate.sh
  async: 3600              # allow up to 1 hour
  poll: 0                  # don't wait (fire and forget)
  register: migration_job

# Later, check if it's done
- name: Check migration status
  async_status:
    jid: "{{ migration_job.ansible_job_id }}"
  register: job_result
  until: job_result.finished
  retries: 360
  delay: 10
```

### Limiting Scope

```bash
# Run only on specific hosts
ansible-playbook site.yml --limit web1.example.com

# Run only specific tags
ansible-playbook site.yml --tags deploy

# Check mode (dry run)
ansible-playbook site.yml --check --diff
```

---

## 7. Scaling to Thousands of Hosts

```
Strategy for 5000+ hosts:
  1. Increase forks (50-100)
  2. Enable pipelining
  3. Enable fact caching
  4. Use mitogen
  5. Use serial for rolling deploys
  6. Use async for long tasks
  7. Minimize fact gathering (gather_facts: false where possible)
  8. Use callback plugins for output (yaml, json — NOT default)
```

### Serial Execution (Rolling Deploys)

```yaml
- hosts: webservers
  serial:
    - 1                    # deploy to 1 host first (canary)
    - "10%"                # then 10% of remaining
    - "100%"               # then all remaining
  max_fail_percentage: 5   # abort if >5% fail

  tasks:
    - include_role:
        name: app_deploy
```

---

## 8. Secrets Management

### Vault for In-Repo Secrets

```bash
# Best practice: separate vault files
inventory/
└── production/
    └── group_vars/
        └── all/
            ├── vars.yml       # references vault vars
            └── vault.yml      # encrypted with ansible-vault
```

```yaml
# vars.yml (unencrypted, readable)
db_password: "{{ vault_db_password }}"
api_key: "{{ vault_api_key }}"

# vault.yml (encrypted)
vault_db_password: supersecret
vault_api_key: sk-1234567890
```

### External Secret Backends

```yaml
# HashiCorp Vault lookup
- name: Get secret from Vault
  set_fact:
    db_password: "{{ lookup('hashi_vault', 'secret/data/db:password') }}"

# AWS Secrets Manager lookup
- name: Get secret from AWS
  set_fact:
    db_password: "{{ lookup('amazon.aws.aws_secret', 'prod/db-password') }}"

# Environment variable (CI/CD pipelines)
- name: Get secret from env
  set_fact:
    db_password: "{{ lookup('env', 'DB_PASSWORD') }}"
```

---

## 9. Dynamic Inventory from Cloud Providers

### AWS EC2

```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
filters:
  tag:Environment: production
  instance-state-name: running
keyed_groups:
  - key: tags.Role
    prefix: role
  - key: placement.availability_zone
    prefix: az
compose:
  ansible_host: private_ip_address
```

### GCP

```yaml
# inventory/gcp.yml
plugin: google.cloud.gcp_compute
projects:
  - my-gcp-project
zones:
  - us-central1-a
  - us-central1-b
filters:
  - status = RUNNING
  - labels.managed = ansible
keyed_groups:
  - key: labels.role
    prefix: role
```

---

## 10. CI/CD Integration

```yaml
# .github/workflows/ansible.yml
name: Ansible Deploy
on:
  push:
    branches: [main]
    paths:
      - 'ansible/**'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install ansible ansible-lint
      - run: ansible-lint ansible/

  deploy:
    needs: lint
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - run: pip install ansible boto3
      - run: |
          echo "${{ secrets.SSH_KEY }}" > /tmp/key
          chmod 600 /tmp/key
          echo "${{ secrets.VAULT_PASSWORD }}" > /tmp/vault_pass
      - run: |
          cd ansible
          ansible-playbook site.yml \
            -i inventory/production/hosts.yml \
            --vault-password-file /tmp/vault_pass \
            --private-key /tmp/key \
            --diff
```

---

## Production Checklist

```
Idempotency:
  [ ] Second playbook run shows changed=0
  [ ] No shell/command modules without creates/removes
  [ ] All tasks use proper modules (apt, service, template) not raw commands

Security:
  [ ] Secrets encrypted with ansible-vault
  [ ] Vault password not in repo (CI secret, password file)
  [ ] SSH keys use dedicated deploy keys (not personal)
  [ ] Least privilege: become only when needed

Reliability:
  [ ] Health checks after deployment (uri module with retries)
  [ ] Rolling deploys with serial and max_fail_percentage
  [ ] block/rescue for rollback capability
  [ ] Error handling with failed_when and ignore_errors where appropriate

Performance:
  [ ] Pipelining enabled
  [ ] Forks ≥ 20
  [ ] Fact caching enabled for large inventories
  [ ] gather_facts: false where not needed

Testing:
  [ ] ansible-lint passes
  [ ] Molecule tests for roles
  [ ] --check --diff dry run before production
```

---

## Key Takeaways

| Practice | What to Remember |
|---|---|
| Idempotency | Run twice, changed=0 second time. Use modules, not shell |
| Directory structure | Separate inventory per env. Separate vault from vars |
| Role design | One role = one concern. Defaults for flexibility |
| Testing | ansible-lint + molecule. Test on multiple OS versions |
| Error handling | block/rescue/always. Rollback capability |
| Performance | Pipelining + forks + fact caching = 5-10x faster |
| Secrets | Vault for in-repo. External backends for rotation |
| Scaling | Mitogen, async tasks, serial deploys for large fleets |
| CI/CD | Lint in PR, deploy on merge. Vault password from CI secrets |
