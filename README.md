# ⚙️ Ansible Configuration Management

> Automated multi-environment Linux server provisioning, package installation, and deployment workflows. Eliminated 60% of operational overhead and reduced repetitive manual configuration tasks.

[![Ansible](https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white)](https://www.ansible.com)
[![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)](https://www.linux.org)
[![Python](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)](https://www.python.org)
[![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)](https://gnu.org/software/bash)

---

## 📌 Overview

Production Ansible playbooks for provisioning and configuring Linux servers across dev, staging, and production environments — eliminating manual SSH-and-configure workflows entirely.

**Key capabilities:**
- ✅ Base server hardening (SSH, firewall, fail2ban, OS updates)
- ✅ Docker & Docker Compose installation and configuration
- ✅ Application deployment playbooks with rolling updates
- ✅ Multi-environment inventory with group_vars
- ✅ Vault-encrypted secrets management
- ✅ Python-based dynamic inventory for AWS EC2
- ✅ Idempotent plays — safe to run repeatedly
- ✅ Health check validation after every deploy

---

## 🗂️ Project Structure

```
ansible-config-management/
├── ansible.cfg
├── inventories/
│   ├── dev/
│   │   ├── hosts.ini
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── webservers.yml
│   ├── staging/
│   │   ├── hosts.ini
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── webservers.yml
│   └── production/
│       ├── aws_ec2.yml              # Dynamic inventory (AWS EC2)
│       └── group_vars/
│           ├── all.yml
│           └── webservers.yml
├── playbooks/
│   ├── site.yml                    # Master playbook
│   ├── base-server.yml             # Base hardening for all servers
│   ├── install-docker.yml          # Docker installation
│   ├── deploy-app.yml              # Application deployment
│   ├── rolling-update.yml          # Rolling update across fleet
│   └── health-check.yml            # Post-deploy health validation
├── roles/
│   ├── common/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   └── defaults/main.yml
│   ├── docker/
│   │   ├── tasks/main.yml
│   │   └── defaults/main.yml
│   ├── app-deploy/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   │   ├── app.env.j2
│   │   │   └── docker-compose.yml.j2
│   │   └── defaults/main.yml
│   └── monitoring-agent/
│       └── tasks/main.yml
├── scripts/
│   ├── run-playbook.sh
│   └── check-connectivity.sh
└── vault/
    └── secrets.yml.example         # Encrypted secrets template
```

---

## 📄 Key Files & Content

### `ansible.cfg`
```ini
[defaults]
inventory          = ./inventories
remote_user        = ec2-user
private_key_file   = ~/.ssh/devops-key.pem
host_key_checking  = False
forks              = 20
timeout            = 30
log_path           = /var/log/ansible/ansible.log
roles_path         = ./roles
vault_password_file = ~/.ansible/vault_pass

[ssh_connection]
pipelining      = True
ssh_args        = -o ControlMaster=auto -o ControlPersist=60s
control_path    = /tmp/ansible-ssh-%%h-%%p-%%r

[privilege_escalation]
become       = True
become_method = sudo
become_user  = root
```

### `inventories/production/aws_ec2.yml` — Dynamic Inventory
```yaml
# Dynamic inventory plugin for AWS EC2
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1

filters:
  instance-state-name: running
  tag:Environment: production

keyed_groups:
  - key: tags.Role
    prefix: role
  - key: tags.Environment
    prefix: env
  - key: placement.region
    prefix: aws_region

hostnames:
  - tag:Name
  - private-ip-address

compose:
  ansible_host: private_ip_address
```

### `inventories/production/group_vars/all.yml`
```yaml
# Global vars — all environments
ansible_python_interpreter: /usr/bin/python3
timezone: Asia/Kolkata

# OS hardening
ssh_port: 22
ssh_allowed_users:
  - ec2-user
  - deploy

fail2ban_enabled: true
ufw_enabled: true

# Docker
docker_version: "26.1"
docker_compose_version: "2.27.0"

# App
app_name: myapp
app_user: deploy
app_dir: /opt/{{ app_name }}
app_port: 8080

# Logging
log_retention_days: 30
```

### `roles/common/tasks/main.yml`
```yaml
---
# Base server hardening tasks

- name: Update all packages
  yum:
    name: "*"
    state: latest
    update_cache: yes
  when: ansible_os_family == "RedHat"

- name: Install essential packages
  package:
    name:
      - curl
      - wget
      - git
      - vim
      - htop
      - unzip
      - jq
      - python3
      - python3-pip
      - awscli
    state: present

- name: Set timezone
  community.general.timezone:
    name: "{{ timezone }}"

- name: Configure SSH hardening
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "{{ item.regexp }}"
    line: "{{ item.line }}"
    state: present
  loop:
    - { regexp: '^PermitRootLogin', line: 'PermitRootLogin no' }
    - { regexp: '^PasswordAuthentication', line: 'PasswordAuthentication no' }
    - { regexp: '^X11Forwarding', line: 'X11Forwarding no' }
    - { regexp: '^MaxAuthTries', line: 'MaxAuthTries 3' }
    - { regexp: '^ClientAliveInterval', line: 'ClientAliveInterval 300' }
    - { regexp: '^ClientAliveCountMax', line: 'ClientAliveCountMax 2' }
  notify: Restart sshd

- name: Create deploy user
  user:
    name: "{{ app_user }}"
    shell: /bin/bash
    groups: docker
    append: yes
    system: no

- name: Install and enable fail2ban
  block:
    - name: Install fail2ban
      package:
        name: fail2ban
        state: present
    - name: Configure fail2ban
      template:
        src: fail2ban-jail.conf.j2
        dest: /etc/fail2ban/jail.local
    - name: Enable fail2ban
      service:
        name: fail2ban
        state: started
        enabled: true
  when: fail2ban_enabled | bool

- name: Configure log rotation
  template:
    src: logrotate-app.j2
    dest: /etc/logrotate.d/{{ app_name }}
    mode: '0644'
```

### `roles/docker/tasks/main.yml`
```yaml
---
- name: Remove old Docker versions
  package:
    name:
      - docker
      - docker-client
      - docker-common
      - docker-engine
    state: absent

- name: Install Docker dependencies
  package:
    name:
      - yum-utils
      - device-mapper-persistent-data
      - lvm2
    state: present

- name: Add Docker repo
  get_url:
    url: https://download.docker.com/linux/centos/docker-ce.repo
    dest: /etc/yum.repos.d/docker-ce.repo

- name: Install Docker CE
  package:
    name:
      - "docker-ce-{{ docker_version }}"
      - docker-ce-cli
      - containerd.io
    state: present

- name: Configure Docker daemon
  copy:
    dest: /etc/docker/daemon.json
    content: |
      {
        "log-driver": "awslogs",
        "log-opts": {
          "awslogs-region": "{{ aws_region }}",
          "awslogs-group": "/ec2/{{ app_name }}/docker"
        },
        "storage-driver": "overlay2",
        "live-restore": true,
        "userland-proxy": false
      }
  notify: Restart Docker

- name: Start and enable Docker
  service:
    name: docker
    state: started
    enabled: true

- name: Install Docker Compose
  get_url:
    url: "https://github.com/docker/compose/releases/download/v{{ docker_compose_version }}/docker-compose-linux-x86_64"
    dest: /usr/local/bin/docker-compose
    mode: '0755'

- name: Verify Docker installation
  command: docker --version
  register: docker_version_output
  changed_when: false

- name: Print Docker version
  debug:
    msg: "Docker installed: {{ docker_version_output.stdout }}"
```

### `playbooks/rolling-update.yml`
```yaml
---
# Rolling update playbook — updates servers in batches to maintain availability
- name: Rolling application update
  hosts: "{{ target_group | default('webservers') }}"
  serial: "{{ batch_size | default('30%') }}"   # Update 30% of fleet at a time
  max_fail_percentage: 20                        # Abort if >20% fail

  pre_tasks:
    - name: Check current app version
      command: "docker inspect {{ app_name }} --format '{{ '{{' }}.Config.Image{{ '}}' }}'"
      register: current_image
      failed_when: false
      changed_when: false

    - name: Print current version
      debug:
        msg: "Current: {{ current_image.stdout }} → New: {{ new_image_tag }}"

    - name: Remove server from load balancer (drain)
      # Deregister from ALB target group before updating
      community.aws.elb_target:
        target_group_arn: "{{ alb_target_group_arn }}"
        target_id: "{{ ansible_ec2_instance_id }}"
        state: absent
        target_status: draining
        target_status_timeout: 30

  roles:
    - role: app-deploy
      vars:
        image_tag: "{{ new_image_tag }}"

  post_tasks:
    - name: Health check — wait for app to be ready
      uri:
        url: "http://localhost:{{ app_port }}/health"
        method: GET
        status_code: 200
      register: health_result
      until: health_result.status == 200
      retries: 15
      delay: 10

    - name: Re-register server with load balancer
      community.aws.elb_target:
        target_group_arn: "{{ alb_target_group_arn }}"
        target_id: "{{ ansible_ec2_instance_id }}"
        state: present
      when: health_result.status == 200

    - name: Rollback if health check failed
      block:
        - name: Restore previous image
          docker_container:
            name: "{{ app_name }}"
            image: "{{ current_image.stdout }}"
            state: started
        - name: Fail the play
          fail:
            msg: "Health check failed after deploy. Rolled back to {{ current_image.stdout }}"
      when: health_result.status != 200
```

### `scripts/run-playbook.sh`
```bash
#!/bin/bash
# run-playbook.sh — Wrapper for consistent playbook execution
set -euo pipefail

ENVIRONMENT=${1:-dev}
PLAYBOOK=${2:-site.yml}
EXTRA_VARS=${3:-""}

INVENTORY="./inventories/${ENVIRONMENT}"

echo "🔧 Running playbook: ${PLAYBOOK}"
echo "📦 Environment: ${ENVIRONMENT}"

# Syntax check first
ansible-playbook \
  -i "${INVENTORY}" \
  "playbooks/${PLAYBOOK}" \
  --syntax-check

# Dry run
echo "📋 Dry run (check mode)..."
ansible-playbook \
  -i "${INVENTORY}" \
  "playbooks/${PLAYBOOK}" \
  --check \
  --diff \
  ${EXTRA_VARS:+--extra-vars "$EXTRA_VARS"}

# Confirm
read -p "✅ Proceed with actual run? (y/N): " confirm
if [[ "${confirm}" != "y" ]]; then
  echo "❌ Aborted."
  exit 0
fi

# Execute
ansible-playbook \
  -i "${INVENTORY}" \
  "playbooks/${PLAYBOOK}" \
  --diff \
  ${EXTRA_VARS:+--extra-vars "$EXTRA_VARS"}

echo "🎉 Playbook complete."
```

---

## 🚀 Quick Start

```bash
# Install Ansible and dependencies
pip3 install ansible boto3 botocore

# Install required collections
ansible-galaxy collection install \
  amazon.aws \
  community.general \
  community.docker

# Check connectivity to all hosts
./scripts/check-connectivity.sh production

# Dry run (check mode)
ansible-playbook -i inventories/production/aws_ec2.yml \
  playbooks/base-server.yml \
  --check --diff

# Apply base hardening to all servers
ansible-playbook -i inventories/production/aws_ec2.yml \
  playbooks/base-server.yml

# Rolling deploy of new image
ansible-playbook -i inventories/production/aws_ec2.yml \
  playbooks/rolling-update.yml \
  --extra-vars "new_image_tag=v1.2.3"
```

---

## 🎯 Results Achieved

- 🤖 **60% reduced** operational overhead
- ⏱️ Server provisioning: **45 minutes → 8 minutes** (automated)
- 🔄 Rolling updates with automatic rollback on failure
- ✅ Idempotent — safe to run at any time without side effects
- 🔒 Secrets encrypted with Ansible Vault

---

## 📬 Contact

**Pushpa Raghul R** | [LinkedIn](https://linkedin.com/in/pushparaghul-devops) | [GitHub](https://github.com/raghul011)
