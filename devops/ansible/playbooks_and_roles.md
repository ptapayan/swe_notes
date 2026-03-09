# Ansible Playbooks and Roles — Deep Dive

**Related:** [Ansible Fundamentals](ansible_fundamentals.md) | [Variables & Templates](ansible_variables_and_templates.md) | [Ansible Best Practices](ansible_best_practices.md) | [Docker Compose](../docker/docker_compose.md)

---

## Playbook Structure

A playbook is a YAML file containing one or more **plays**. Each play maps a group of hosts to a list of tasks.

```yaml
---
# site.yml — a playbook with two plays
- name: Configure webservers         # Play 1
  hosts: webservers                   # target group from inventory
  become: true                        # use sudo
  vars:
    http_port: 80

  pre_tasks:                          # run before roles
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600

  roles:                              # apply roles
    - common
    - nginx

  tasks:                              # run after roles
    - name: Ensure nginx is running
      service:
        name: nginx
        state: started
        enabled: true

  post_tasks:                         # run after tasks
    - name: Verify nginx is responding
      uri:
        url: "http://localhost:{{ http_port }}"
        status_code: 200

  handlers:                           # run only when notified
    - name: Restart nginx
      service:
        name: nginx
        state: restarted

- name: Configure databases           # Play 2
  hosts: databases
  become: true
  roles:
    - postgresql
```

### Execution Order Within a Play

```
1. pre_tasks
2. handlers triggered by pre_tasks
3. roles
4. tasks
5. handlers triggered by roles and tasks
6. post_tasks
7. handlers triggered by post_tasks
```

---

## Tasks

A task is a single call to an Ansible module. Each task runs on all target hosts before moving to the next task (by default).

```yaml
tasks:
  - name: Install nginx                    # human-readable description
    apt:                                    # module name
      name: nginx                           # module parameters
      state: present
    become: true                            # sudo for this task
    when: ansible_os_family == "Debian"     # conditional
    tags:
      - nginx                               # tag for selective execution
    register: install_result                # capture output
    notify: Restart nginx                   # trigger handler if changed
```

### Task Outcomes

```
ok       → module ran, no changes needed (idempotent check passed)
changed  → module ran, made changes
skipped  → task skipped (when condition was false)
failed   → module returned an error
```

---

## Handlers

Handlers are tasks that only run when **notified**. They run once at the end of the play, even if notified multiple times.

```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx                # triggers handler if changed

  - name: Update SSL certificate
    copy:
      src: cert.pem
      dest: /etc/ssl/cert.pem
    notify: Restart nginx                # same handler — only runs once

handlers:
  - name: Restart nginx                  # must match the notify name exactly
    service:
      name: nginx
      state: restarted
```

**What's happening:** If the nginx config changes, the handler is **queued**. If the SSL cert also changes, the handler is queued again (but won't run twice). At the end of the play (after all tasks), queued handlers execute in the order they're defined.

**Force handlers to run mid-play:**

```yaml
tasks:
  - name: Update config
    template: ...
    notify: Restart nginx

  - meta: flush_handlers    # run all queued handlers NOW

  - name: Run smoke tests   # runs after nginx has restarted
    uri: ...
```

---

## Essential Modules

### Package Management

```yaml
# Debian/Ubuntu
- name: Install packages
  apt:
    name:
      - nginx
      - curl
      - htop
    state: present
    update_cache: true

# RHEL/CentOS/Fedora
- name: Install packages
  yum:
    name:
      - httpd
      - curl
    state: present

# Generic (auto-detects package manager)
- name: Install package
  package:
    name: git
    state: present
```

### File Operations

```yaml
# Create directory
- name: Create app directory
  file:
    path: /opt/myapp
    state: directory
    owner: deploy
    group: deploy
    mode: "0755"

# Copy file from control node to target
- name: Copy config file
  copy:
    src: files/app.conf           # relative to role or playbook
    dest: /etc/myapp/app.conf
    owner: root
    group: root
    mode: "0644"
    backup: true                  # create backup of existing file

# Template (Jinja2 rendering)
- name: Deploy config from template
  template:
    src: templates/app.conf.j2    # Jinja2 template
    dest: /etc/myapp/app.conf
    owner: root
    mode: "0644"
  notify: Restart app
```

### Service Management

```yaml
- name: Start and enable nginx
  service:
    name: nginx
    state: started
    enabled: true                 # start on boot

- name: Restart service
  systemd:
    name: myapp
    state: restarted
    daemon_reload: true           # reload systemd if unit file changed
```

### Command Execution

```yaml
# command module (no shell features — safer)
- name: Check disk space
  command: df -h /
  register: disk_output
  changed_when: false             # command is read-only, never "changed"

# shell module (supports pipes, redirects, env vars)
- name: Find large files
  shell: find /var/log -type f -size +100M | head -5
  register: large_files
  changed_when: false

# script module (run a local script on the remote)
- name: Run deployment script
  script: scripts/deploy.sh
  args:
    creates: /opt/myapp/.deployed   # skip if file exists (idempotency)
```

### User and Group Management

```yaml
- name: Create application user
  user:
    name: deploy
    shell: /bin/bash
    groups: sudo,docker
    append: true                    # add to groups without removing existing
    create_home: true
    state: present

- name: Add SSH key for deploy user
  authorized_key:
    user: deploy
    key: "{{ lookup('file', 'files/deploy_key.pub') }}"
    state: present
```

### Docker Module

```yaml
- name: Pull and run Docker container
  docker_container:
    name: my-api
    image: my-registry/api:{{ app_version }}
    state: started
    restart_policy: unless-stopped
    ports:
      - "8080:3000"
    env:
      NODE_ENV: production
      DATABASE_URL: "{{ database_url }}"
    volumes:
      - app-data:/app/data
    networks:
      - name: app-network
```

---

## Roles

Roles are the primary way to organize and reuse Ansible code. A role is a structured directory of tasks, handlers, variables, templates, and files.

### Role Directory Structure

```
roles/
└── nginx/
    ├── defaults/
    │   └── main.yml          # default variables (lowest precedence)
    ├── vars/
    │   └── main.yml          # role variables (higher precedence than defaults)
    ├── tasks/
    │   └── main.yml          # task definitions
    ├── handlers/
    │   └── main.yml          # handler definitions
    ├── templates/
    │   └── nginx.conf.j2     # Jinja2 templates
    ├── files/
    │   └── index.html        # static files
    ├── meta/
    │   └── main.yml          # role metadata and dependencies
    └── README.md
```

### Role Example: nginx

```yaml
# roles/nginx/defaults/main.yml
nginx_worker_processes: auto
nginx_worker_connections: 1024
nginx_server_name: localhost
nginx_listen_port: 80
```

```yaml
# roles/nginx/tasks/main.yml
---
- name: Install nginx
  apt:
    name: nginx
    state: present
  notify: Restart nginx

- name: Deploy nginx config
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s         # validate config before applying
  notify: Restart nginx

- name: Deploy site config
  template:
    src: site.conf.j2
    dest: /etc/nginx/sites-available/default
  notify: Reload nginx

- name: Ensure nginx is running
  service:
    name: nginx
    state: started
    enabled: true
```

```yaml
# roles/nginx/handlers/main.yml
---
- name: Restart nginx
  service:
    name: nginx
    state: restarted

- name: Reload nginx
  service:
    name: nginx
    state: reloaded
```

```yaml
# roles/nginx/meta/main.yml
---
dependencies:
  - role: common                  # common role runs first
  - role: ssl
    vars:
      ssl_domain: "{{ nginx_server_name }}"
```

### Using Roles in Playbooks

```yaml
# Method 1: roles section (classic)
- hosts: webservers
  roles:
    - common
    - nginx
    - role: app_deploy
      vars:
        app_version: "2.1.0"

# Method 2: include_role (dynamic, in tasks section)
- hosts: webservers
  tasks:
    - name: Apply base configuration
      include_role:
        name: common

    - name: Deploy application
      include_role:
        name: app_deploy
      vars:
        app_version: "2.1.0"
      when: deploy_enabled | default(true)
```

---

## Includes vs Imports (Static vs Dynamic)

```
import_* → STATIC: parsed at playbook load time
include_* → DYNAMIC: processed at runtime when reached
```

```yaml
# IMPORT (static): task file is parsed immediately
- import_tasks: install.yml          # always loaded, tags apply
  when: install_needed               # condition applied to EACH task in file

# INCLUDE (dynamic): task file is loaded at runtime
- include_tasks: install.yml         # loaded only when this task runs
  when: install_needed               # condition applied to the include itself

# Same for roles
- import_role:
    name: nginx                      # static: always loaded

- include_role:
    name: nginx                      # dynamic: loaded when reached
  when: use_nginx | default(true)
```

**When to use which:**
- `import_*`: When you always want the tasks (simpler, better for `--list-tasks`)
- `include_*`: When tasks are conditional, looped, or determined at runtime

---

## Tags

Tags allow selective execution of tasks:

```yaml
tasks:
  - name: Install packages
    apt: ...
    tags:
      - install
      - packages

  - name: Deploy config
    template: ...
    tags:
      - config
      - deploy

  - name: Restart service
    service: ...
    tags:
      - restart
      - deploy
```

```bash
# Run only tasks tagged "deploy"
ansible-playbook site.yml --tags deploy

# Run everything EXCEPT "install" tasks
ansible-playbook site.yml --skip-tags install

# List all tags in a playbook
ansible-playbook site.yml --list-tags
```

---

## Ansible Galaxy

Galaxy is Ansible's package manager for roles and collections.

```bash
# Install a role from Galaxy
ansible-galaxy install geerlingguy.docker

# Install from requirements file
ansible-galaxy install -r requirements.yml

# Create a new role skeleton
ansible-galaxy init my_new_role
```

```yaml
# requirements.yml
roles:
  - name: geerlingguy.docker
    version: 7.1.0
  - name: geerlingguy.certbot

collections:
  - name: community.docker
    version: 3.4.0
  - name: amazon.aws
    version: 7.0.0
```

---

## Practical Examples / Real-World Patterns

### Complete Web Server Deployment

```yaml
# deploy_web.yml
---
- name: Deploy web application
  hosts: webservers
  become: true
  vars:
    app_version: "{{ version | default('latest') }}"

  pre_tasks:
    - name: Update package cache
      apt:
        update_cache: true
        cache_valid_time: 3600

  tasks:
    - name: Create app directory
      file:
        path: /opt/webapp
        state: directory
        owner: www-data
        mode: "0755"

    - name: Download application archive
      get_url:
        url: "https://releases.example.com/app-{{ app_version }}.tar.gz"
        dest: /tmp/app.tar.gz
        checksum: "sha256:{{ app_checksum }}"

    - name: Extract application
      unarchive:
        src: /tmp/app.tar.gz
        dest: /opt/webapp
        remote_src: true
      notify: Restart app

    - name: Deploy application config
      template:
        src: app.conf.j2
        dest: /opt/webapp/config/app.conf
      notify: Restart app

    - name: Ensure app service is running
      systemd:
        name: webapp
        state: started
        enabled: true

  handlers:
    - name: Restart app
      systemd:
        name: webapp
        state: restarted
```

### Rolling Deployment with Serial

```yaml
- name: Rolling deploy
  hosts: webservers
  serial: "25%"               # deploy to 25% of hosts at a time
  max_fail_percentage: 10      # abort if >10% of hosts fail
  become: true

  tasks:
    - name: Remove from load balancer
      uri:
        url: "http://{{ lb_host }}/api/remove/{{ inventory_hostname }}"
        method: POST

    - name: Deploy new version
      include_role:
        name: app_deploy

    - name: Wait for app to be ready
      uri:
        url: "http://localhost:{{ app_port }}/health"
        status_code: 200
      register: health
      until: health.status == 200
      retries: 30
      delay: 5

    - name: Add back to load balancer
      uri:
        url: "http://{{ lb_host }}/api/add/{{ inventory_hostname }}"
        method: POST
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Playbook | YAML file with plays. Each play = hosts + tasks |
| Play execution | pre_tasks → roles → tasks → post_tasks. Handlers at end |
| Handlers | Only run when notified. Run once even if notified multiple times |
| Modules | Python programs run on target. Check state → change if needed |
| Roles | Structured directories: tasks/, handlers/, templates/, defaults/, vars/, files/ |
| Role defaults | `defaults/main.yml` = lowest precedence (easily overridden) |
| import_* | Static: parsed at load time. Tags and limits work predictably |
| include_* | Dynamic: loaded at runtime. For conditional/looped includes |
| Tags | Selective execution: `--tags deploy`, `--skip-tags install` |
| Galaxy | Package manager for roles/collections: `ansible-galaxy install` |
