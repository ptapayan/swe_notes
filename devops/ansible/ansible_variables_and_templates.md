# Ansible Variables and Templates — Deep Dive

**Related:** [Ansible Fundamentals](ansible_fundamentals.md) | [Playbooks & Roles](playbooks_and_roles.md) | [Ansible Best Practices](ansible_best_practices.md)

---

## Variable Precedence (The 22 Levels)

Ansible has a complex variable precedence hierarchy. Understanding it is critical for debugging "why does this variable have that value?"

```
LOWEST PRECEDENCE (easily overridden)
  1.  command-line values (e.g., -u user) — NOT variables
  2.  role defaults (roles/x/defaults/main.yml)
  3.  inventory file or script group vars
  4.  inventory group_vars/all
  5.  playbook group_vars/all
  6.  inventory group_vars/*
  7.  playbook group_vars/*
  8.  inventory file or script host vars
  9.  inventory host_vars/*
  10. playbook host_vars/*
  11. host facts / cached set_facts
  12. play vars
  13. play vars_prompt
  14. play vars_files
  15. role vars (roles/x/vars/main.yml)
  16. block vars (in a block/when)
  17. task vars (inline in a task)
  18. include_vars
  19. set_facts / registered vars
  20. role params (when using roles: with vars)
  21. include params
  22. extra vars (-e "var=value") ← ALWAYS WINS
HIGHEST PRECEDENCE (overrides everything)
```

**How to think about it:** There are really 4 tiers that matter:

```
┌──────────────────────────────────────────────────┐
│  TIER 4: Extra vars (-e) — emergency overrides   │  ALWAYS WINS
├──────────────────────────────────────────────────┤
│  TIER 3: Role vars, task vars, set_fact          │  High priority
├──────────────────────────────────────────────────┤
│  TIER 2: Inventory vars, play vars               │  Environment-specific
├──────────────────────────────────────────────────┤
│  TIER 1: Role defaults                           │  Sensible defaults
└──────────────────────────────────────────────────┘
```

**Practical rule:** Put defaults in `roles/x/defaults/main.yml` (low precedence, easy to override). Put constants in `roles/x/vars/main.yml` (high precedence, hard to override).

---

## Defining Variables

### In Inventory

```yaml
# inventory/group_vars/webservers.yml
nginx_port: 80
nginx_workers: 4
app_env: production
```

```yaml
# inventory/host_vars/web1.example.com.yml
nginx_workers: 8          # override for this specific host
```

### In Playbooks

```yaml
- hosts: webservers
  vars:
    http_port: 80
    app_name: my-api
    deploy_user: deploy

  vars_files:
    - vars/common.yml
    - "vars/{{ ansible_os_family }}.yml"

  vars_prompt:
    - name: deploy_version
      prompt: "Which version to deploy?"
      default: latest
      private: false
```

### In Role Defaults (Low Precedence)

```yaml
# roles/nginx/defaults/main.yml
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_keepalive_timeout: 65
nginx_server_name: _
nginx_listen_port: 80
nginx_error_log_level: warn
```

### Registered Variables

```yaml
- name: Check if app is running
  command: systemctl is-active myapp
  register: app_status
  ignore_errors: true

- name: Start app if not running
  service:
    name: myapp
    state: started
  when: app_status.rc != 0

- name: Debug output
  debug:
    var: app_status
    # app_status.stdout → "active" or "inactive"
    # app_status.rc → 0 (success) or non-zero (failure)
    # app_status.changed → bool
```

### set_fact (Runtime Variables)

```yaml
- name: Calculate deployment timestamp
  set_fact:
    deploy_timestamp: "{{ ansible_date_time.iso8601 }}"
    deploy_dir: "/opt/releases/{{ app_version }}-{{ ansible_date_time.epoch }}"

- name: Set derived variables
  set_fact:
    database_url: "postgresql://{{ db_user }}:{{ db_pass }}@{{ db_host }}:{{ db_port }}/{{ db_name }}"
```

---

## Facts (System Information)

Facts are automatically gathered variables about the target system. Ansible runs the `setup` module at the beginning of each play.

```bash
# See all facts for a host
ansible web1.example.com -m setup

# Filter specific facts
ansible web1.example.com -m setup -a "filter=ansible_memtotal_mb"
```

### Commonly Used Facts

```yaml
# OS Information
ansible_distribution         # "Ubuntu"
ansible_distribution_version # "22.04"
ansible_os_family            # "Debian"
ansible_kernel               # "5.15.0-91-generic"
ansible_architecture         # "x86_64"

# Network
ansible_hostname             # "web1"
ansible_fqdn                 # "web1.example.com"
ansible_default_ipv4.address # "10.0.1.5"
ansible_all_ipv4_addresses   # ["10.0.1.5", "172.17.0.1"]

# Hardware
ansible_memtotal_mb          # 16384
ansible_processor_vcpus      # 4
ansible_devices               # disk information

# Date/Time
ansible_date_time.iso8601    # "2024-01-15T10:30:00Z"
ansible_date_time.epoch      # "1705312200"
```

### Custom Facts

Place files in `/etc/ansible/facts.d/` on the managed node:

```bash
# /etc/ansible/facts.d/app.fact (must be executable or .json/.ini)
#!/bin/bash
cat <<EOF
{
  "version": "2.1.0",
  "environment": "production",
  "deployed_at": "2024-01-15T10:30:00Z"
}
EOF
```

```yaml
# Access as ansible_local
- debug:
    var: ansible_local.app.version   # "2.1.0"
```

### Disabling Fact Gathering

```yaml
- hosts: webservers
  gather_facts: false       # skip fact gathering (faster for simple tasks)
  tasks:
    - name: Quick task that doesn't need facts
      command: uptime
```

---

## Jinja2 Templating

Ansible uses Jinja2 for variable substitution, conditionals, loops, and filters in templates and playbook values.

### Variable Substitution

```yaml
# In playbooks: use {{ }}
- name: Create config
  template:
    src: app.conf.j2
    dest: "/opt/{{ app_name }}/config/app.conf"
```

```jinja2
{# In templates: app.conf.j2 #}
[server]
hostname = {{ ansible_hostname }}
port = {{ app_port }}
environment = {{ app_env }}
workers = {{ ansible_processor_vcpus * 2 }}
database_url = {{ database_url }}
```

### Filters

Filters transform data. Ansible adds many filters on top of Jinja2's built-in ones.

```yaml
# String filters
"{{ my_string | upper }}"                   # "HELLO"
"{{ my_string | lower }}"                   # "hello"
"{{ my_string | replace('old', 'new') }}"   # string replacement
"{{ my_string | regex_replace('\\d+', 'N') }}"  # regex replace
"{{ my_string | b64encode }}"               # base64 encode
"{{ my_string | hash('sha256') }}"          # SHA-256 hash

# Default values (critical for optional variables)
"{{ optional_var | default('fallback') }}"
"{{ optional_var | default(omit) }}"        # omit the parameter entirely

# List/Dict filters
"{{ my_list | join(', ') }}"                # join list to string
"{{ my_list | unique }}"                    # unique elements
"{{ my_list | sort }}"                      # sort
"{{ my_list | length }}"                    # count elements
"{{ my_list | map('upper') | list }}"       # transform each element
"{{ my_dict | dict2items }}"                # dict → list of {key, value}
"{{ my_list | items2dict }}"                # list of {key, value} → dict
"{{ my_list | selectattr('active', 'equalto', true) | list }}"  # filter objects

# Type conversion
"{{ '8080' | int }}"                        # string → int
"{{ my_dict | to_json }}"                   # dict → JSON string
"{{ my_dict | to_yaml }}"                   # dict → YAML string
"{{ my_json_string | from_json }}"          # JSON string → dict

# Network filters
"{{ '10.0.1.0/24' | ipaddr('network') }}"  # "10.0.1.0"
"{{ '10.0.1.5/24' | ipaddr('address') }}"  # "10.0.1.5"

# Path filters
"{{ '/etc/nginx/nginx.conf' | basename }}"  # "nginx.conf"
"{{ '/etc/nginx/nginx.conf' | dirname }}"   # "/etc/nginx"
```

### Conditionals in Templates

```jinja2
{# nginx.conf.j2 #}
{% if nginx_ssl_enabled %}
server {
    listen 443 ssl;
    ssl_certificate {{ nginx_ssl_cert }};
    ssl_certificate_key {{ nginx_ssl_key }};
{% else %}
server {
    listen {{ nginx_port }};
{% endif %}
    server_name {{ nginx_server_name }};

    {% if nginx_auth_enabled | default(false) %}
    auth_basic "Restricted";
    auth_basic_user_file /etc/nginx/htpasswd;
    {% endif %}

    location / {
        proxy_pass http://{{ app_host }}:{{ app_port }};
    }
}
```

### Loops in Templates

```jinja2
{# /etc/hosts.j2 #}
127.0.0.1 localhost

{% for host in groups['webservers'] %}
{{ hostvars[host]['ansible_default_ipv4']['address'] }}  {{ host }}
{% endfor %}

{# upstream.conf.j2 #}
upstream backend {
{% for server in backend_servers %}
    server {{ server.host }}:{{ server.port }} weight={{ server.weight | default(1) }};
{% endfor %}
}
```

### Conditional Logic in Templates (Advanced)

```jinja2
{# database.conf.j2 #}
{% set max_connections = ansible_memtotal_mb // 10 %}
{% if max_connections > 500 %}
{% set max_connections = 500 %}
{% endif %}

max_connections = {{ max_connections }}

{% for key, value in db_params.items() | sort %}
{{ key }} = {{ value }}
{% endfor %}
```

---

## when Conditionals

```yaml
tasks:
  # Simple boolean
  - name: Install nginx
    apt:
      name: nginx
    when: install_nginx | default(true)

  # OS-specific
  - name: Install on Debian
    apt:
      name: nginx
    when: ansible_os_family == "Debian"

  - name: Install on RHEL
    yum:
      name: nginx
    when: ansible_os_family == "RedHat"

  # Multiple conditions (AND)
  - name: Configure production
    template:
      src: prod.conf.j2
      dest: /etc/app.conf
    when:
      - app_env == "production"
      - ansible_memtotal_mb >= 4096

  # OR condition
  - name: Install on supported OS
    package:
      name: nginx
    when: ansible_distribution == "Ubuntu" or ansible_distribution == "Debian"

  # Based on registered variable
  - name: Check service
    command: systemctl is-active nginx
    register: nginx_status
    ignore_errors: true

  - name: Start if not running
    service:
      name: nginx
      state: started
    when: nginx_status.rc != 0

  # Variable defined/undefined checks
  - name: Use custom config
    template:
      src: custom.conf.j2
      dest: /etc/app.conf
    when: custom_config is defined

  - name: Use default config
    copy:
      src: default.conf
      dest: /etc/app.conf
    when: custom_config is not defined
```

---

## Loops

```yaml
# Simple loop
- name: Create users
  user:
    name: "{{ item }}"
    state: present
  loop:
    - alice
    - bob
    - charlie

# Loop with dictionaries
- name: Create users with details
  user:
    name: "{{ item.name }}"
    groups: "{{ item.groups }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
  loop:
    - { name: alice, groups: sudo }
    - { name: bob, groups: "docker,sudo" }
    - { name: charlie, groups: docker, shell: /bin/zsh }

# Loop over a dictionary
- name: Set sysctl params
  sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
  loop: "{{ sysctl_params | dict2items }}"
  vars:
    sysctl_params:
      net.ipv4.ip_forward: 1
      vm.swappiness: 10

# Loop with index
- name: Create numbered configs
  template:
    src: worker.conf.j2
    dest: "/etc/workers/worker-{{ idx }}.conf"
  loop: "{{ worker_configs }}"
  loop_control:
    index_var: idx
    label: "{{ item.name }}"     # cleaner output (don't dump entire item)

# Nested loops (with subelements)
- name: Add SSH keys for users
  authorized_key:
    user: "{{ item.0.name }}"
    key: "{{ item.1 }}"
  loop: "{{ users | subelements('ssh_keys') }}"
```

---

## Magic Variables

Special variables automatically available in every play:

```yaml
# inventory_hostname: the name of the current host (as in inventory)
- debug:
    msg: "Running on {{ inventory_hostname }}"

# groups: dictionary of all groups and their hosts
- debug:
    msg: "Webservers: {{ groups['webservers'] }}"

# hostvars: access variables from other hosts
- debug:
    msg: "DB IP: {{ hostvars['db1']['ansible_default_ipv4']['address'] }}"

# group_names: list of groups the current host belongs to
- debug:
    msg: "This host is in: {{ group_names }}"

# play_hosts: list of all hosts in the current play
- debug:
    msg: "All hosts in this play: {{ play_hosts }}"

# ansible_play_batch: hosts in current batch (when using serial)
# role_path: path to the current role directory
# playbook_dir: path to the playbook directory
```

---

## Ansible Vault (Encrypting Secrets)

Vault encrypts sensitive data so it can be stored safely in version control.

```bash
# Encrypt a variable file
ansible-vault encrypt inventory/group_vars/production/secrets.yml

# Create a new encrypted file
ansible-vault create inventory/group_vars/production/secrets.yml

# Edit an encrypted file
ansible-vault edit inventory/group_vars/production/secrets.yml

# Decrypt (view) an encrypted file
ansible-vault view inventory/group_vars/production/secrets.yml

# Encrypt a single string (inline in YAML)
ansible-vault encrypt_string 'supersecret123' --name 'db_password'
# Output:
# db_password: !vault |
#   $ANSIBLE_VAULT;1.1;AES256
#   3832366438373...
```

```yaml
# Use vault-encrypted variables normally
# inventory/group_vars/production/secrets.yml (encrypted)
db_password: supersecret123
api_key: sk-1234567890
ssl_private_key: |
  -----BEGIN PRIVATE KEY-----
  ...
  -----END PRIVATE KEY-----
```

```bash
# Run playbook with vault password
ansible-playbook site.yml --ask-vault-pass
ansible-playbook site.yml --vault-password-file ~/.vault_pass.txt
ansible-playbook site.yml --vault-password-file /path/to/vault_script.sh
```

**Best practice:** Separate vault files from regular variables:

```
group_vars/
└── production/
    ├── vars.yml        # unencrypted variables
    └── vault.yml       # encrypted secrets (prefix vars with vault_)
```

```yaml
# vars.yml
db_password: "{{ vault_db_password }}"    # reference vault variable

# vault.yml (encrypted)
vault_db_password: "supersecret123"
```

---

## Practical Examples / Real-World Patterns

### Dynamic Configuration Based on Host Specs

```yaml
- name: Configure app based on available resources
  set_fact:
    app_workers: "{{ [ansible_processor_vcpus * 2, 16] | min }}"
    app_max_memory: "{{ (ansible_memtotal_mb * 0.75) | int }}m"
    db_pool_size: "{{ [ansible_processor_vcpus * 5, 50] | min }}"

- name: Deploy app config
  template:
    src: app.conf.j2
    dest: /etc/app/config.yml
```

### Cross-Host Variable Access

```yaml
# On the API server, configure it to connect to the database server
- hosts: api_servers
  tasks:
    - name: Configure database connection
      template:
        src: database.yml.j2
        dest: /opt/api/config/database.yml
      vars:
        db_host: "{{ hostvars[groups['databases'][0]]['ansible_default_ipv4']['address'] }}"
        db_port: 5432
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Precedence | Extra vars (-e) always win. Role defaults are lowest. 22 levels total |
| Role defaults vs vars | defaults/ = easily overridden. vars/ = hard to override |
| Facts | Auto-gathered system info. `ansible_*` variables. Disable with `gather_facts: false` |
| Jinja2 filters | `default()`, `join()`, `to_json`, `dict2items`, `selectattr()` |
| when | Conditional task execution. Supports `and`, `or`, `is defined`, `is not defined` |
| Loops | `loop:` with lists or dicts. Use `loop_control.label` for clean output |
| Magic variables | `hostvars`, `groups`, `inventory_hostname`, `group_names` |
| Vault | Encrypt secrets in-repo. `ansible-vault encrypt/edit/view`. Use vault_password_file in CI |
| Templates | Jinja2 files with `{% if %}`, `{% for %}`, `{{ var \| filter }}` |
