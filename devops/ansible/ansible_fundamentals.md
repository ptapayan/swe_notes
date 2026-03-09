# Ansible Fundamentals — How It Actually Works

**Related:** [Playbooks & Roles](playbooks_and_roles.md) | [Variables & Templates](ansible_variables_and_templates.md) | [Ansible Best Practices](ansible_best_practices.md) | [GitHub Actions CI/CD](../github/github_actions_cicd.md)

---

## What Is Ansible?

Ansible is an **agentless, push-based configuration management and automation tool**. It connects to remote machines over SSH, pushes small programs (modules) to them, executes them, and removes them.

**How to think about it:** You describe the desired state of your infrastructure in YAML files. Ansible figures out what needs to change and makes it so.

```
┌──────────────────┐          SSH           ┌──────────────────┐
│  Control Node    │ ─────────────────────► │  Managed Node 1  │
│                  │                        │  (any Linux box)  │
│  • Ansible CLI   │          SSH           ├──────────────────┤
│  • Playbooks     │ ─────────────────────► │  Managed Node 2  │
│  • Inventory     │                        │                  │
│  • Modules       │          SSH           ├──────────────────┤
│                  │ ─────────────────────► │  Managed Node 3  │
│  Python 3.x      │                        │                  │
└──────────────────┘                        └──────────────────┘
                                             Requirements:
                                             • SSH access
                                             • Python on target
                                             • No agent needed
```

---

## How Ansible Works Under the Hood

When you run `ansible-playbook site.yml`, here is what actually happens:

```
1. Ansible reads the playbook (YAML → Python data structures)

2. For each play, Ansible resolves the target hosts from inventory

3. For each task, Ansible:
   a. Generates a Python script from the module + arguments
   b. Creates a temporary directory on the target: ~/.ansible/tmp/
   c. Copies the Python script to the target via SFTP/SCP
   d. Executes the script via SSH: python3 /tmp/ansible-xxx/module.py
   e. Captures the JSON output (changed: true/false, stdout, etc.)
   f. Removes the temporary files
   g. Evaluates the result and decides next action

4. Results are aggregated and displayed as the recap
```

**Key insight:** Ansible modules are NOT shell scripts. They are self-contained Python programs that run on the target, check current state, decide if changes are needed, make changes if necessary, and return structured JSON. This is why Ansible is idempotent — the module checks before acting.

```bash
# You can see exactly what Ansible sends to the target:
ansible-playbook site.yml -vvvv
# Shows SSH commands, temp file creation, module execution, cleanup
```

---

## Architecture Components

### Control Node

The machine where Ansible is installed and playbooks are executed from. Must be Linux/macOS (not Windows). This is typically a CI/CD runner, a bastion host, or your workstation.

### Managed Nodes

The target machines Ansible configures. Requirements:
- SSH access from the control node
- Python 3.x installed (most modules need it)
- No Ansible installation needed (agentless)

### Modules

Modules are the units of work. Each module is a Python script that performs a specific action:

```bash
# Using the apt module via ad-hoc command
ansible webservers -m apt -a "name=nginx state=present"

# What this does:
# 1. SSH into each webserver
# 2. Copy the apt module (Python script) to the target
# 3. Run it with args: name=nginx, state=present
# 4. Module checks if nginx is installed
# 5. If not installed → installs it → returns changed: true
# 6. If already installed → does nothing → returns changed: false
```

### Plugins

Plugins extend Ansible's core functionality:
- **Connection plugins:** How to connect (SSH, WinRM, Docker, local)
- **Callback plugins:** How to display output (JSON, YAML, minimal)
- **Lookup plugins:** Fetch data from external sources (files, URLs, vault)
- **Filter plugins:** Transform data in Jinja2 templates
- **Inventory plugins:** Dynamic inventory from cloud providers

---

## Inventory

The inventory defines **which machines** Ansible manages and how to connect to them.

### Static Inventory (INI Format)

```ini
# inventory/hosts
[webservers]
web1.example.com
web2.example.com ansible_host=10.0.1.2 ansible_port=2222

[databases]
db1.example.com
db2.example.com

[loadbalancers]
lb1.example.com

# Group of groups
[production:children]
webservers
databases
loadbalancers

# Variables for all hosts in a group
[webservers:vars]
ansible_user=deploy
http_port=80
```

### Static Inventory (YAML Format)

```yaml
# inventory/hosts.yml
all:
  children:
    production:
      children:
        webservers:
          hosts:
            web1.example.com:
            web2.example.com:
              ansible_host: 10.0.1.2
              ansible_port: 2222
          vars:
            ansible_user: deploy
            http_port: 80
        databases:
          hosts:
            db1.example.com:
            db2.example.com:
```

### Dynamic Inventory

Dynamic inventory scripts or plugins query an external source (AWS, GCP, Azure, Terraform state) and return the host list as JSON.

```bash
# AWS EC2 dynamic inventory plugin
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - us-east-1
  - us-west-2
keyed_groups:
  - key: tags.Environment
    prefix: env
  - key: instance_type
    prefix: type
filters:
  tag:Managed: ansible
  instance-state-name: running
```

```bash
# Verify dynamic inventory
ansible-inventory -i inventory/aws_ec2.yml --list
ansible-inventory -i inventory/aws_ec2.yml --graph
```

### Host Variables and Group Variables

```
inventory/
├── hosts.yml
├── host_vars/
│   ├── web1.example.com.yml    # variables specific to web1
│   └── db1.example.com.yml     # variables specific to db1
└── group_vars/
    ├── all.yml                  # variables for ALL hosts
    ├── webservers.yml           # variables for webservers group
    └── databases.yml            # variables for databases group
```

```yaml
# inventory/group_vars/webservers.yml
nginx_worker_processes: 4
nginx_worker_connections: 1024
ssl_certificate: /etc/ssl/certs/wildcard.pem
```

```yaml
# inventory/host_vars/web1.example.com.yml
nginx_worker_processes: 8    # override for this specific host
```

---

## ansible.cfg

Configuration file that controls Ansible's behavior. Searched in this order:
1. `$ANSIBLE_CONFIG` environment variable
2. `./ansible.cfg` (current directory)
3. `~/.ansible.cfg` (home directory)
4. `/etc/ansible/ansible.cfg` (global)

```ini
# ansible.cfg
[defaults]
inventory = inventory/hosts.yml
remote_user = deploy
private_key_file = ~/.ssh/ansible_key
host_key_checking = False        # disable for dynamic cloud environments
forks = 20                        # run on 20 hosts in parallel
timeout = 30                      # SSH connection timeout
retry_files_enabled = False       # don't create .retry files
stdout_callback = yaml            # human-readable output
interpreter_python = auto_silent  # auto-detect Python on target

[privilege_escalation]
become = True                     # use sudo by default
become_method = sudo
become_user = root
become_ask_pass = False           # don't prompt for sudo password

[ssh_connection]
pipelining = True                 # major performance boost (see best practices)
ssh_args = -o ControlMaster=auto -o ControlPersist=60s
```

---

## Ad-Hoc Commands

Quick one-off commands without writing a playbook:

```bash
# Ping all hosts (tests SSH connectivity + Python)
ansible all -m ping

# Run a shell command
ansible webservers -m shell -a "uptime"

# Copy a file to all webservers
ansible webservers -m copy -a "src=./app.conf dest=/etc/app.conf mode=0644"

# Install a package
ansible databases -m apt -a "name=postgresql state=present" --become

# Restart a service
ansible webservers -m service -a "name=nginx state=restarted" --become

# Gather facts (system information)
ansible web1.example.com -m setup

# Run on a subset
ansible webservers -m command -a "df -h" --limit web1.example.com
```

**What's happening:** Each ad-hoc command is a single-task playbook. The `-m` flag specifies the module, `-a` passes arguments. Ansible SSHs in, runs the module, and returns JSON results.

---

## Comparison with Other Tools

```
┌──────────────┬──────────┬───────────┬───────────┬────────────┐
│              │ Ansible  │ Chef      │ Puppet    │ Terraform  │
├──────────────┼──────────┼───────────┼───────────┼────────────┤
│ Architecture │ Agentless│ Agent     │ Agent     │ Agentless  │
│              │ Push     │ Pull      │ Pull      │ Push       │
├──────────────┼──────────┼───────────┼───────────┼────────────┤
│ Language     │ YAML     │ Ruby DSL  │ Puppet DSL│ HCL        │
├──────────────┼──────────┼───────────┼───────────┼────────────┤
│ Transport    │ SSH      │ HTTPS     │ HTTPS     │ Cloud API  │
├──────────────┼──────────┼───────────┼───────────┼────────────┤
│ State        │ No state │ Chef      │ Puppet    │ .tfstate   │
│              │ file     │ Server    │ Server    │ file       │
├──────────────┼──────────┼───────────┼───────────┼────────────┤
│ Primary Use  │ Config   │ Config    │ Config    │ Infra      │
│              │ Mgmt     │ Mgmt      │ Mgmt      │ Provision  │
├──────────────┼──────────┼───────────┼───────────┼────────────┤
│ Learning     │ Low      │ Medium    │ High      │ Medium     │
│ Curve        │          │           │           │            │
└──────────────┴──────────┴───────────┴───────────┴────────────┘
```

**Key distinction:** Ansible and Terraform are **complementary**, not competing:
- **Terraform:** Provisions infrastructure (create VMs, networks, load balancers)
- **Ansible:** Configures infrastructure (install packages, deploy code, manage services)

A typical workflow: Terraform creates EC2 instances → outputs IP addresses → Ansible configures them.

---

## Practical Examples / Real-World Patterns

### Verify Connectivity to All Hosts

```bash
# Quick connectivity check
ansible all -m ping -i inventory/production.yml

# With verbose output to debug SSH issues
ansible all -m ping -i inventory/production.yml -vvvv

# Check a specific group
ansible databases -m ping
```

### Quick System Information Gathering

```bash
# Get OS information
ansible all -m setup -a "filter=ansible_distribution*"

# Get memory information
ansible all -m setup -a "filter=ansible_memtotal_mb"

# Get all facts (huge output — pipe to file)
ansible web1 -m setup > web1_facts.json
```

### Emergency Operations (Ad-Hoc)

```bash
# Kill a runaway process on all webservers
ansible webservers -m shell -a "pkill -f 'runaway_process'" --become

# Check disk space across all hosts
ansible all -m shell -a "df -h / | tail -1"

# Deploy a hotfix config file
ansible webservers -m copy -a "src=hotfix.conf dest=/etc/app/config.conf backup=yes" --become
ansible webservers -m service -a "name=app state=restarted" --become
```

---

## Key Takeaways

| Concept | What to Remember |
|---|---|
| Architecture | Agentless, push-based. SSH + Python on target. No state file |
| How it works | Generates Python script → SCP to target → execute → capture JSON → cleanup |
| Inventory | Static (INI/YAML) or dynamic (cloud plugins). host_vars, group_vars |
| Modules | Self-contained Python programs. Check state first → change if needed → return JSON |
| Idempotent | Running the same playbook twice produces the same result. Modules check before acting |
| ansible.cfg | Controls defaults. Searched in CWD first, then home, then global |
| vs Terraform | Ansible configures machines. Terraform provisions infrastructure. Use both |
| Ad-hoc | Quick one-off commands: `ansible <group> -m <module> -a "<args>"` |
