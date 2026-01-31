# Ansible for Homelab

Infrastructure as code for your homelab.

## When to Use

- Consistent server configuration
- Automate repetitive tasks
- Document infrastructure
- Disaster recovery setup

## Installation

```bash
# Ubuntu/Debian
apt install ansible

# macOS
brew install ansible

# pip (any platform)
pip install ansible
```

## Project Structure

```
homelab-ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       └── servers.yml
├── playbooks/
│   ├── site.yml
│   ├── docker.yml
│   └── monitoring.yml
└── roles/
    ├── common/
    ├── docker/
    └── monitoring/
```

## Inventory

### hosts.yml

```yaml
all:
  children:
    servers:
      hosts:
        proxmox:
          ansible_host: 192.168.1.10
        nas:
          ansible_host: 192.168.1.20
        docker-host:
          ansible_host: 192.168.1.30
    
    containers:
      hosts:
        jellyfin:
          ansible_host: 192.168.1.50
          ansible_connection: lxc
          ansible_lxc_host: proxmox

  vars:
    ansible_user: admin
    ansible_ssh_private_key_file: ~/.ssh/homelab
```

### group_vars/all.yml

```yaml
---
timezone: America/Los_Angeles
dns_servers:
  - 192.168.1.10
  - 1.1.1.1

admin_users:
  - name: admin
    ssh_key: "ssh-ed25519 AAAA..."
```

## Basic Playbook

### playbooks/site.yml

```yaml
---
- name: Configure all servers
  hosts: servers
  become: true
  
  tasks:
    - name: Update apt cache
      apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_os_family == "Debian"

    - name: Install common packages
      apt:
        name:
          - vim
          - htop
          - curl
          - git
          - tmux
        state: present

    - name: Set timezone
      timezone:
        name: "{{ timezone }}"

    - name: Configure SSH
      template:
        src: templates/sshd_config.j2
        dest: /etc/ssh/sshd_config
      notify: Restart SSH

  handlers:
    - name: Restart SSH
      service:
        name: sshd
        state: restarted
```

## Roles

### roles/docker/tasks/main.yml

```yaml
---
- name: Install Docker dependencies
  apt:
    name:
      - apt-transport-https
      - ca-certificates
      - curl
      - gnupg
    state: present

- name: Add Docker GPG key
  apt_key:
    url: https://download.docker.com/linux/ubuntu/gpg
    state: present

- name: Add Docker repository
  apt_repository:
    repo: "deb https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
    state: present

- name: Install Docker
  apt:
    name:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-compose-plugin
    state: present
    update_cache: true

- name: Add user to docker group
  user:
    name: "{{ ansible_user }}"
    groups: docker
    append: true

- name: Start Docker service
  service:
    name: docker
    state: started
    enabled: true
```

### roles/docker/handlers/main.yml

```yaml
---
- name: Restart Docker
  service:
    name: docker
    state: restarted
```

## Common Patterns

### Install Packages

```yaml
- name: Install packages
  apt:
    name: "{{ packages }}"
    state: present
  vars:
    packages:
      - nginx
      - certbot
```

### Copy Files

```yaml
- name: Copy configuration
  copy:
    src: files/nginx.conf
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
  notify: Reload Nginx
```

### Template Files

```yaml
- name: Configure application
  template:
    src: templates/app.conf.j2
    dest: /etc/myapp/config.yml
```

### Docker Containers

```yaml
- name: Deploy Traefik
  community.docker.docker_container:
    name: traefik
    image: traefik:v3.0
    state: started
    restart_policy: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /etc/traefik:/etc/traefik
```

### Docker Compose

```yaml
- name: Deploy stack
  community.docker.docker_compose_v2:
    project_src: /opt/myapp
    state: present
```

### Create User

```yaml
- name: Create admin users
  user:
    name: "{{ item.name }}"
    groups: sudo,docker
    shell: /bin/bash
    create_home: true
  loop: "{{ admin_users }}"

- name: Add SSH keys
  authorized_key:
    user: "{{ item.name }}"
    key: "{{ item.ssh_key }}"
  loop: "{{ admin_users }}"
```

### Cron Jobs

```yaml
- name: Schedule backup
  cron:
    name: "Daily backup"
    minute: "0"
    hour: "2"
    job: "/opt/scripts/backup.sh"
```

### Firewall (UFW)

```yaml
- name: Allow SSH
  ufw:
    rule: allow
    port: '22'
    proto: tcp

- name: Enable UFW
  ufw:
    state: enabled
    default: deny
```

## Secrets Management

### Ansible Vault

```bash
# Create encrypted vars file
ansible-vault create group_vars/secrets.yml

# Edit encrypted file
ansible-vault edit group_vars/secrets.yml

# Run with vault password
ansible-playbook site.yml --ask-vault-pass
```

### secrets.yml

```yaml
---
db_password: supersecret
api_key: abc123
```

### Using Secrets

```yaml
- name: Configure database
  template:
    src: db.conf.j2
    dest: /etc/myapp/db.conf
  vars:
    password: "{{ db_password }}"
```

## Homelab Playbooks

### playbooks/monitoring.yml

```yaml
---
- name: Deploy monitoring stack
  hosts: docker-host
  become: true

  vars:
    monitoring_dir: /opt/monitoring

  tasks:
    - name: Create monitoring directory
      file:
        path: "{{ monitoring_dir }}"
        state: directory

    - name: Copy docker-compose
      copy:
        src: files/monitoring/docker-compose.yml
        dest: "{{ monitoring_dir }}/docker-compose.yml"

    - name: Copy Prometheus config
      template:
        src: templates/prometheus.yml.j2
        dest: "{{ monitoring_dir }}/prometheus.yml"

    - name: Deploy stack
      community.docker.docker_compose_v2:
        project_src: "{{ monitoring_dir }}"
        state: present
```

### playbooks/backup.yml

```yaml
---
- name: Configure backups
  hosts: servers
  become: true

  tasks:
    - name: Install restic
      apt:
        name: restic
        state: present

    - name: Create backup script
      template:
        src: templates/backup.sh.j2
        dest: /opt/scripts/backup.sh
        mode: '0755'

    - name: Schedule backup
      cron:
        name: "Nightly backup"
        minute: "0"
        hour: "2"
        job: "/opt/scripts/backup.sh"
```

## Running Playbooks

```bash
# Run entire site
ansible-playbook playbooks/site.yml

# Run specific playbook
ansible-playbook playbooks/docker.yml

# Limit to specific host
ansible-playbook playbooks/site.yml --limit docker-host

# Dry run
ansible-playbook playbooks/site.yml --check

# Verbose output
ansible-playbook playbooks/site.yml -vvv

# With tags
ansible-playbook playbooks/site.yml --tags "docker,monitoring"
```

## ansible.cfg

```ini
[defaults]
inventory = inventory/hosts.yml
remote_user = admin
private_key_file = ~/.ssh/homelab
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

## Best Practices

1. **Use roles** - modular, reusable configurations
2. **Version control** - git repository for playbooks
3. **Use vault** - encrypt secrets
4. **Test with check mode** - dry run before applying
5. **Idempotent tasks** - safe to run multiple times
6. **Tag everything** - selective execution
7. **Document** - comments in playbooks
