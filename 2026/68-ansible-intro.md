# Day 68 — Introduction to Ansible and Inventory Setup

## Task 1: Understanding Ansible

### What is Configuration Management? Why do we need it?

Once infrastructure exists (servers, VMs, cloud instances), someone has to decide *what runs on them*. Configuration management is the practice of **defining and enforcing the desired state of your servers as code** — which packages are installed, which services are running, which users exist, which files are present, and what their contents are.

Without it, you end up with *snowflake servers* — machines configured manually over time that slowly drift from each other. Nobody knows why production works but staging doesn't. Rebuilding a failed server becomes a guessing game.

Configuration management solves this by making server state:
- **Repeatable** — run the same definition on 100 servers, get identical results
- **Auditable** — your config files live in Git, so you know who changed what and when
- **Recoverable** — when a server dies, you re-provision and re-apply. Done.

### How is Ansible Different from Chef, Puppet, and Salt?

| Tool | Model | Agent Required? | Language | Learning Curve |
|---|---|---|---|---|
| **Ansible** | Push (control node sends tasks) | No — uses SSH | YAML (Playbooks) | Low |
| **Chef** | Pull (agent polls chef-server) | Yes | Ruby (Recipes/Cookbooks) | High |
| **Puppet** | Pull (agent polls puppet-master) | Yes | Puppet DSL | High |
| **SaltStack** | Push + Pull hybrid | Optional (minion agent) | YAML + Jinja2 | Medium |

Ansible's key differentiator: **no agent**. Every other tool requires you to install and maintain a daemon on every managed server. Ansible uses SSH, which every Linux server already runs. This makes it far easier to manage ephemeral cloud infrastructure where servers come and go.

### What Does "Agentless" Mean? How Does Ansible Connect?

Agentless means **there is no persistent software running on the managed node waiting for instructions**. Ansible works by:

1. Reading your inventory (list of servers)
2. Opening an SSH connection from the control node to each managed node
3. Copying a small Python script (the module) to the managed node via SSH
4. Executing it, capturing the output
5. Removing the script and closing the connection

The managed node needs only: **SSH daemon running + Python installed**. Both come pre-installed on virtually every Linux AMI.

### Ansible Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      CONTROL NODE                           │
│                  (WSL on local machine)                     │
│                                                             │
│  ┌──────────────┐  ┌─────────────┐  ┌──────────────────┐   │
│  │  inventory   │  │  playbooks  │  │   ansible.cfg    │   │
│  │ .ini / .yaml │  │  .yml files │  │  (configuration) │   │
│  └──────────────┘  └─────────────┘  └──────────────────┘   │
│                                                             │
│  Ansible Engine ──── reads above ──── executes modules      │
└────────────────────────┬────────────────────────────────────┘
                         │ SSH (port 22)
          ┌──────────────┼──────────────┐
          │              │              │
   ┌──────▼─────┐ ┌──────▼─────┐ ┌─────▼──────┐
   │ web-server │ │ app-server │ │ db-server  │
   │  EC2 inst  │ │  EC2 inst  │ │  EC2 inst  │
   │ ap-south-1 │ │ ap-south-1 │ │ ap-south-1 │
   └────────────┘ └────────────┘ └────────────┘
         MANAGED NODES (no agent installed)
```

**Key components:**
- **Control Node** — the machine where Ansible is installed and run. Here: WSL on local machine. Only one needed.
- **Managed Nodes** — the servers Ansible configures. Here: 3 EC2 t3.micro instances in ap-south-1.
- **Inventory** — the list of managed nodes, grouped by role. Ansible reads this to know *where* to run tasks.
- **Modules** — the units of work. `yum` installs packages. `copy` transfers files. `service` starts/stops daemons. Ansible ships with thousands.
- **Playbooks** — YAML files that tie it all together: "on these hosts, run these tasks in this order."

---

## Task 2: Lab Environment Setup

### Approach Used: Terraform (`for_each`)

Used Terraform to provision 3 EC2 instances in `ap-south-1` with `for_each` on the EC2 resource block, assigning distinct names (`web-server`, `app-server`, `db-server`) via the `Name` tag. Public IPs were retrieved from `outputs.tf`.

**Instance specs:**
- Region: `ap-south-1` (Mumbai)
- Type: `t3.micro`
- AMI: Ubuntu 22.04
- Key pair: used for SSH access from WSL control node

SSH connectivity verified:
```bash
ssh -i ~/your-key.pem ubuntu@<web-server-ip>
ssh -i ~/your-key.pem ubuntu@<app-server-ip>
ssh -i ~/your-key.pem ubuntu@<db-server-ip>
```

---

## Task 3: Ansible Installation

Installed on **WSL (Ubuntu) — the control node**.

```bash
sudo apt update
sudo apt install ansible -y
ansible --version
```

**Why only on the control node?**

Ansible's architecture is asymmetric by design. The control node is the "brain" — it reads playbooks, builds task lists, connects to remotes, and collects results. The managed nodes are purely recipients. They need no Ansible knowledge — just SSH access and Python. This means you can start managing a brand new server the moment it boots, with zero bootstrapping.

---

## Task 4: Inventory File

**Project structure:**
```
ansible-practice/
├── inventory.ini
└── ansible.cfg
```

**`inventory.ini`:**
```ini
[web]
web-server ansible_host=<REDACTED>

[app]
app-server ansible_host=<REDACTED>

[db]
db-server ansible_host=<REDACTED>

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/your-key.pem
```

> Note: `ansible_user=ubuntu` because AMI is Ubuntu 22.04. Amazon Linux uses `ec2-user`.

**Ping verification:**
```bash
ansible all -i inventory.ini -m ping
```

Expected output:
```
web-server | SUCCESS => { "ping": "pong" }
app-server | SUCCESS => { "ping": "pong" }
db-server  | SUCCESS => { "ping": "pong" }
```

---

## Task 5: Ad-Hoc Commands

### Check uptime on all servers
```bash
ansible all -i inventory.ini -m command -a "uptime"
```

### Check free memory on web servers only
```bash
ansible web -i inventory.ini -m command -a "free -h"
```

### Check disk space on all servers
```bash
ansible all -i inventory.ini -m command -a "df -h"
```

### Install git on web group
```bash
ansible web -i inventory.ini -m apt -a "name=git state=present" --become
```

### Copy a file to all servers
```bash
echo "Hello from Ansible" > hello.txt
ansible all -i inventory.ini -m copy -a "src=hello.txt dest=/tmp/hello.txt"
```

### Verify the file was copied
```bash
ansible all -i inventory.ini -m command -a "cat /tmp/hello.txt"
```

### What does `--become` do? When do you need it?

`--become` tells Ansible to **escalate privileges to root** on the managed node — equivalent to prefixing a command with `sudo`. By default Ansible connects as the SSH user (`ubuntu`), which is an unprivileged user. Any operation that touches system-level resources — installing packages, managing services, editing files in `/etc/`, managing users — requires root. `--become` triggers `sudo` on the remote side before running the module.

---

## Task 6: Inventory Groups and Patterns

**Updated `inventory.ini` with group-of-groups:**
```ini
[web]
web-server ansible_host=<REDACTED>

[app]
app-server ansible_host=<REDACTED>

[db]
db-server ansible_host=<REDACTED>

[application:children]
web
app

[all_servers:children]
application
db

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/your-key.pem
```

**Running against different groups:**
```bash
ansible application -i inventory.ini -m ping     # web + app servers
ansible db -i inventory.ini -m ping              # only db server
ansible all_servers -i inventory.ini -m ping     # all three
```

**Using patterns:**
```bash
ansible 'web:app' -i inventory.ini -m ping       # OR: web or app servers
ansible 'all:!db' -i inventory.ini -m ping       # NOT: everyone except db
```

**`ansible.cfg` to drop the `-i` flag:**
```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
remote_user = ubuntu
private_key_file = ~/your-key.pem
```

After this, `ansible all -m ping` works with no `-i` flag. Ansible reads `ansible.cfg` from the current directory first.

---

## `command` vs `shell` Module

| | `command` | `shell` |
|---|---|---|
| Runs via | Direct exec | `/bin/sh -c` |
| Supports pipes (`\|`) | No | Yes |
| Supports redirects (`>`, `>>`) | No | Yes |
| Supports env vars like `$HOME` | No | Yes |
| Safer against injection | Yes | No |

**Rule of thumb:** Use `command` by default. Only reach for `shell` when you need pipes, redirects, or shell-specific syntax. `shell` runs through the shell interpreter, which introduces security surface area if any part of the command is user-supplied input.

```bash
# Needs shell module — pipe is a shell feature
ansible all -m shell -a "ps aux | grep nginx"

# command module is fine here
ansible all -m command -a "df -h"
```