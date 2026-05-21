# Ansible Project

## To run the ansible playbook from cli 
`ansible-playbook -i inventory/hosts/inventory.ini playbook.yml`

# Essential Ansible Modules for Basic Tasks

## 1. Package Management
```yaml
# Install/Remove packages
- name: Install nginx
  package:
    name: nginx
    state: present

- name: Remove apache
  package:
    name: apache2
    state: absent
```

## 2. File Operations
```yaml
# Copy files
- name: Copy config file
  copy:
    src: nginx.conf
    dest: /etc/nginx/nginx.conf

# Create directory
- name: Create directory
  file:
    path: /opt/myapp
    state: directory
    mode: '0755'

# Download files
- name: Download file
  get_url:
    url: https://example.com/file.tar.gz
    dest: /tmp/file.tar.gz
```

## 3. Service Management
```yaml
# Start/stop services
- name: Start nginx
  service:
    name: nginx
    state: started
    enabled: yes

- name: Restart service
  service:
    name: nginx
    state: restarted
```

## 4. User Management
```yaml
# Create user
- name: Create user
  user:
    name: myuser
    state: present
    shell: /bin/bash

# Add to group
- name: Add user to sudo group
  user:
    name: myuser
    groups: sudo
    append: yes
```

## 5. Command Execution
```yaml
# Run commands
- name: Run shell command
  command: uptime
  register: result

- name: Run shell script
  shell: |
    echo "Hello"
    date
    whoami
```

## 6. Template Files
```yaml
# Use Jinja2 templates
- name: Deploy config template
  template:
    src: config.j2
    dest: /etc/app/config.conf
    mode: '0644'
```

## 7. Git Operations
```yaml
# Clone repository
- name: Clone git repo
  git:
    repo: https://github.com/user/repo.git
    dest: /opt/repo
    version: main
```

## 8. System Information
```yaml
# Gather facts
- name: Display OS info
  debug:
    msg: "OS: {{ ansible_os_family }}, Version: {{ ansible_distribution_version }}"

# Check disk space
- name: Check disk usage
  command: df -h
  register: disk_info
```

## 9. Cron Jobs
```yaml
# Add cron job
- name: Add backup cron
  cron:
    name: "daily backup"
    minute: "0"
    hour: "2"
    job: "/usr/local/bin/backup.sh"
```

## 10. Firewall
```yaml
# Open port (ufw)
- name: Allow nginx port
  ufw:
    rule: allow
    port: '80'
    proto: tcp
```

## 11. Archive Operations
```yaml
# Extract archive
- name: Extract tar.gz
  unarchive:
    src: /tmp/file.tar.gz
    dest: /opt/app
    remote_src: yes
```

## 12. Line in File
```yaml
# Add line to file
- name: Add line to config
  lineinfile:
    path: /etc/hosts
    line: "192.168.1.100 webserver"
    state: present
```

## Most Common Basic Tasks:
1. **package** - Install/remove software
2. **copy/template** - Manage configuration files
3. **service** - Control services
4. **command/shell** - Run commands
5. **file** - Manage files/directories
6. **user/group** - Manage users
7. **debug** - Display information

These cover 90% of basic infrastructure automation tasks.
