# Ansible Interview Questions and Answers

A comprehensive collection of Ansible interview questions and answers for preparation — from beginner to advanced level.

---

## Table of Contents

- [Common Questions](#common-questions)
- [Basic Questions](#basic-questions)
- [Process-Related Questions](#process-related-questions)
- [Advanced Questions](#advanced-questions)
- [Scenario-Based / Practical Questions](#scenario-based--practical-questions)

---

## Common Questions

### 1. What is Ansible?

**Answer:** Ansible is an open-source IT automation engine that automates provisioning, configuration management, application deployment, orchestration, and many other IT processes. It uses a simple, human-readable language (YAML) to describe automation jobs. Ansible is agentless — it connects to managed nodes via SSH and pushes small programs called "modules" to perform tasks, then removes them when finished.

### 2. What are three advantages of using Ansible?

**Answer:**
- **Agentless:** No agent software needs to be installed on managed nodes — it uses SSH.
- **Simple YAML syntax:** Playbooks don't require unique coding skills; they're human-readable.
- **Orchestration:** You can orchestrate the deployment of an entire application across multi-tier environments, regardless of where deployment occurs.
- **Idempotent:** Running the same playbook multiple times produces the same result without unintended side effects.
- **Extensible:** Supports custom modules written in any language that returns JSON.

### 3. How does Ansible function?

**Answer:** Ansible works by connecting to nodes (hosts) and pushing out small programs called "Ansible modules." It runs these modules over SSH and removes them when finished. The control node (where Ansible is installed) reads the inventory file to determine which hosts to manage, then executes tasks defined in playbooks against those hosts.

### 4. What is Ansible Tower (AWX)?

**Answer:** Ansible Tower (now called AWX in its open-source form, and Red Hat Ansible Automation Platform commercially) is a web-based UI and REST API that acts as a centralized hub for Ansible automation. It provides:
- Role-based access control (RBAC)
- Job scheduling
- Graphical inventory management
- Real-time job output and logging
- A dashboard for monitoring
- Integrated notifications
- Playbook workflows and tower clusters

It is free for up to 10 nodes.

### 5. Describe the Ansible architecture.

**Answer:** The main components of Ansible architecture include:
- **Control Node:** The machine where Ansible is installed and runs from
- **Managed Nodes:** The remote systems Ansible manages
- **Inventory:** A list of managed nodes (hosts) organized in groups, containing databases and servers
- **Playbooks:** YAML files for task automation defining which tasks need executing
- **Modules:** Units of code that Ansible executes on managed nodes (e.g., `apt`, `yum`, `copy`, `file`)
- **Plugins:** Extend Ansible functionality (connection plugins, callback plugins, lookup plugins, etc.)
- **APIs:** For communicating with cloud services
- **CMDB:** A repository or data warehouse for configuration data

### 6. What is Ansible Galaxy?

**Answer:** Ansible Galaxy is a community hub for finding, sharing, and downloading Ansible roles and collections. It works like a package manager for Ansible content. You can install roles from Galaxy using `ansible-galaxy install <role_name>`.

### 7. What is Red Hat Ansible?

**Answer:** Red Hat Ansible is the commercial, enterprise-supported version of Ansible offered by Red Hat. It includes Ansible Automation Platform, which provides Ansible Tower/Controller, Automation Hub, and enterprise support.

### 8. What are Ansible Roles?

**Answer:** Roles are a way to organize playbooks into reusable components. A role has a defined directory structure:
```
roles/
  myrole/
    tasks/main.yml
    handlers/main.yml
    templates/
    files/
    vars/main.yml
    defaults/main.yml
    meta/main.yml
```
Roles enable code reuse, modularity, and sharing via Ansible Galaxy.

### 9. What are Ansible Playbooks?

**Answer:** Playbooks are YAML files that define a set of tasks to be executed on managed hosts. They describe the desired state of a system. A playbook consists of one or more "plays" that map a group of hosts to roles or tasks.

Example:
```yaml
---
- name: Install and start Nginx
  hosts: webservers
  become: true
  tasks:
    - name: Install nginx
      apt:
        name: nginx
        state: present
    - name: Start nginx
      service:
        name: nginx
        state: started
```

### 10. What are variable names in Ansible?

**Answer:** Variable names in Ansible are identifiers used to store values that can be reused across playbooks, roles, and templates. They must start with a letter and can contain letters, numbers, and underscores (e.g., `http_port`, `max_retries`). Variables can be defined in inventory, playbooks, roles, command line, or external files.

### 11. What are environment variables in Ansible?

**Answer:** Environment variables are system-level variables from the OS environment. In Ansible, you can access them using the `ansible_env` fact (e.g., `ansible_env.HOME`) or the `lookup('env', 'VAR_NAME')` plugin. You can also set environment variables for tasks using the `environment` keyword.

### 12. What two technical skills do SREs need to use Ansible?

**Answer:**
- **Sysadmin skills:** Understanding of Linux/Unix systems, networking, and infrastructure
- **DevOps knowledge:** Understanding of CI/CD pipelines, infrastructure as code, and automation principles

### 13. What skills should you aim to refine as an SRE using Ansible?

**Answer:** Key skills to improve include Docker container knowledge, Ansible module development, Jinja2 templating, cloud provider integrations (AWS, Azure, GCP), and CI/CD pipeline integration.

---

## Basic Questions

### 14. What is the difference between Ansible and Puppet?

**Answer:**

| Feature | Ansible | Puppet |
|---------|---------|--------|
| Language | YAML (Playbooks) | Ruby (Puppet DSL) |
| Architecture | Agentless (SSH-based) | Agent-based (requires Puppet agent) |
| Configuration | Push-based | Pull-based |
| Ease of learning | Simpler, human-readable | Steeper learning curve |
| Modules | Written in any language (returns JSON) | Written in Ruby |
| Setup | Minimal (just SSH) | Requires Puppet master & agents |

### 15. Is Ansible a configuration management tool?

**Answer:** Yes, Ansible is an open-source configuration management and automation tool. It automates complex tasks and is used in multi-tier application environments for provisioning, configuration management, application deployment, and orchestration.

### 16. Name five key Ansible Tower features.

**Answer:**
1. **Tower CLI tool** — Command-line interface for interacting with Tower
2. **Integrated notifications** — Slack, email, webhook integrations
3. **Dashboard** — Visual overview of job status and inventory health
4. **Tower clusters** — High availability and horizontal scaling
5. **Playbook workflows** — Chain multiple playbooks with conditional logic

### 17. What are two soft skills SREs need to use Ansible?

**Answer:**
- **Communication:** Ability to collaborate with team members, document automation, and explain infrastructure decisions to stakeholders
- **Problem-solving:** Troubleshoot issues, debug playbooks, and develop solutions for complex infrastructure challenges

### 18. What is looping in Ansible?

**Answer:** Looping is a mechanism to repeat tasks over a list of items. Ansible provides several loop constructs:

```yaml
- name: Install packages
  apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - git
    - curl
```

Other loop types include `with_items`, `with_dict`, `with_fileglob`, `with_sequence`, and `loop` with filters.

### 19. What is CI/CD and why is it important for Ansible?

**Answer:** CI/CD stands for Continuous Integration / Continuous Delivery (or Deployment). It's a set of practices for automating code integration, testing, and deployment. Ansible fits into CI/CD pipelines as the deployment and configuration automation tool — it can provision infrastructure, deploy applications, and configure services as part of the pipeline.

### 20. Can you manage Windows Nano Server with Ansible?

**Answer:** Yes, Ansible can manage Windows Nano Server. Ansible connects to Windows hosts using WinRM (Windows Remote Management) instead of SSH. You use `win_*` modules (e.g., `win_command`, `win_service`, `win_package`) for Windows-specific tasks.

### 21. What is Ansible Vault?

**Answer:** Ansible Vault is a feature that allows you to encrypt sensitive data such as passwords, keys, and secrets within Ansible files. Commands include:
```bash
ansible-vault create secrets.yml      # Create encrypted file
ansible-vault edit secrets.yml        # Edit encrypted file
ansible-vault encrypt existing.yml    # Encrypt existing file
ansible-vault decrypt secrets.yml     # Decrypt file
ansible-vault view secrets.yml        # View without decrypting
```
Use `--vault-password-file` or `--ask-vault-pass` when running playbooks.

### 22. What is the ad-hoc command in Ansible?

**Answer:** Ad-hoc commands are one-liner Ansible commands that perform quick tasks without writing a playbook. They use the `ansible` command:
```bash
ansible all -m ping                              # Ping all hosts
ansible webservers -m shell -a "uptime"          # Run shell command
ansible dbservers -m service -a "name=mysql state=restarted" --become
```

### 23. What are handlers in Ansible?

**Answer:** Handlers are special tasks that run only when notified by another task using the `notify` keyword. They typically restart services after configuration changes:
```yaml
tasks:
  - name: Update nginx config
    template:
      src: nginx.conf.j2
      dest: /etc/nginx/nginx.conf
    notify: Restart nginx

handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: restarted
```
Handlers run once at the end of the play, even if notified multiple times.

### 24. How are variable names different from environment variables?

**Answer:** Variable names are Ansible-specific variables defined within playbooks, roles, or inventory for use in automation tasks. Environment variables are OS-level variables that exist in the shell environment of the managed node. Ansible variables are accessed with `{{ var_name }}`, while environment variables are accessed via `ansible_env.VAR` or `lookup('env', 'VAR')`.

### 25. Why is it important to learn Ansible in SRE roles?

**Answer:** Ansible enables SREs to automate repetitive infrastructure tasks, ensure configuration consistency across environments, reduce human error, enable infrastructure as code practices, and speed up incident response through automated remediation playbooks.

### 26. Can you build reusable content using Ansible?

**Answer:** Yes. Ansible supports reusable content through:
- **Roles** — Encapsulate tasks, handlers, variables, templates, and files
- **Collections** — Package and distribute roles, modules, and plugins
- **Include/Import** — Include task files, playbooks, and roles dynamically or statically

### 27. Why do SREs use Ansible?

**Answer:** SREs use Ansible for automating provisioning, configuration management, application deployments, rolling updates, security compliance, and disaster recovery. Its agentless nature and simplicity make it ideal for managing large-scale infrastructure.

---

## Process-Related Questions

### 28. How do you create empty files using Ansible?

**Answer:** Use the `file` module with two key parameters:
```yaml
- name: Create an empty file
  file:
    path: /tmp/myfile.txt
    state: touch
    mode: '0644'
```
- **path** — The location where the file will be created
- **state: touch** — Creates a new empty file (or updates timestamp if it exists)

### 29. How do you set the environment variable for an entire Playbook?

**Answer:** Use the `environment` keyword at the play level:
```yaml
- name: My Playbook
  hosts: all
  environment:
    HTTP_PROXY: http://proxy.example.com:8080
    APP_ENV: production
  tasks:
    - name: Run a command with env vars
      shell: echo $APP_ENV
```

### 30. How do you create encrypted files using Ansible?

**Answer:** Use `ansible-vault create`:
```bash
ansible-vault create secrets.yml
```
This opens an editor to enter content, then encrypts the file. To use in a playbook:
```yaml
vars_files:
  - secrets.yml
```
Run with: `ansible-playbook site.yml --ask-vault-pass` or `--vault-password-file=vault.pass`

### 31. When would you use Ansible tags?

**Answer:** Tags allow you to selectively run parts of a playbook:
```yaml
tasks:
  - name: Install packages
    apt: name=nginx state=present
    tags: [install, packages]

  - name: Configure nginx
    template: src=nginx.conf.j2 dest=/etc/nginx/nginx.conf
    tags: [configure]
```
Run specific tags: `ansible-playbook site.yml --tags "configure"`
Skip tags: `ansible-playbook site.yml --skip-tags "install"`

### 32. How do you filter tasks in tags?

**Answer:** Use `--tags` or `--skip-tags` on the command line, or configure `TAGS_RUN` and `TAGS_SKIP` in `ansible.cfg`:
```bash
ansible-playbook site.yml --tags "deploy,configure"
ansible-playbook site.yml --skip-tags "debug"
```

### 33. How do you upgrade Ansible?

**Answer:**
```bash
# Using pip
sudo pip install ansible --upgrade

# Using pip for specific version
sudo pip install ansible==2.17.0

# Using package manager (apt)
sudo apt update && sudo apt upgrade ansible

# Using Homebrew (macOS)
brew upgrade ansible
```

### 34. When would you use module utilities in Ansible?

**Answer:** Module utilities are shared Python code that Ansible modules can import. Use them when writing custom modules that share common functionality — like argument parsing, API client logic, or error handling — to avoid code duplication.

### 35. What are core modules in Ansible?

**Answer:** Core modules are modules maintained by the Ansible engineering team and shipped with Ansible by default. Examples: `copy`, `file`, `template`, `service`, `apt`, `yum`, `command`, `shell`, `debug`, `set_fact`.

### 36. What are extras modules in Ansible?

**Answer:** Extras modules (now part of collections) are community-maintained modules that extend Ansible's functionality. They cover cloud providers (AWS, Azure, GCP), networking equipment, databases, and more. Install via: `ansible-galaxy collection install amazon.aws`.

### 37. How do you use encrypted files to automate password inputs?

**Answer:** Store passwords in a vault-encrypted file and reference them in playbooks:
```bash
ansible-vault create passwords.yml
```
```yaml
# passwords.yml (encrypted)
db_password: mysecretpass
api_key: abc123
```
```yaml
# playbook.yml
vars_files:
  - passwords.yml
tasks:
  - name: Configure database
    mysql_user:
      name: admin
      password: "{{ db_password }}"
```

### 38. How do you loop over a list of grouped hosts in a template?

**Answer:** Use `groups` variable in Jinja2 templates:
```jinja2
{% for host in groups['webservers'] %}
server {{ host }} {{ hostvars[host]['ansible_host'] }}:80;
{% endfor %}
```

### 39. How do you make reusable content in Ansible?

**Answer:** Use roles with proper directory structure and naming conventions:
```bash
ansible-galaxy init myrole
```
This creates the standard role structure. You can then share roles via Ansible Galaxy or internal Git repos, and include them in playbooks:
```yaml
roles:
  - common
  - nginx
  - monitoring
```

### 40. How do you access and change documentation in Ansible?

**Answer:** Access documentation locally using `ansible-doc`:
```bash
ansible-doc apt            # View module docs
ansible-doc -l             # List all modules
ansible-doc -s apt         # Show module snippet/parameters
```
To contribute to Ansible documentation, go to the Git repository, fork it, make changes, add a commit message, and submit a pull request.

---

## Advanced Questions

### 41. How do you access Shell Environment Variables?

**Answer:** Use the `lookup` plugin or `ansible_env` fact:
```yaml
# Using lookup
- debug:
    msg: "Home dir is {{ lookup('env', 'HOME') }}"

# Using ansible_env (from gathered facts)
- debug:
    msg: "Home dir is {{ ansible_env.HOME }}"
```

### 42. How do you make EC2 management faster in Ansible?

**Answer:** Use dynamic inventory scripts or the `aws_ec2` inventory plugin instead of static inventory files. This auto-discovers EC2 instances:
```yaml
# aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
keyed_groups:
  - key: tags.Name
    prefix: tag_Name
```
Also use `async` and `poll` for parallel task execution, and `ec2_instance` module for managing instances.

### 43. Can you use Docker modules in Ansible?

**Answer:** Yes, Ansible has Docker modules in the `community.docker` collection:
```bash
ansible-galaxy collection install community.docker
```
```yaml
- name: Run a Docker container
  community.docker.docker_container:
    name: my_app
    image: nginx:latest
    ports:
      - "8080:80"
    state: started
```

### 44. How do you access an Ansible_Variables list?

**Answer:** Use the `debug` module with `vars`:
```yaml
- debug:
    var: hostvars[inventory_hostname]   # All host variables
- debug:
    var: vars                            # All variables in scope
- debug:
    var: ansible_facts                   # All gathered facts
```

### 45. What is idempotency in Ansible?

**Answer:** Idempotency means that running the same playbook multiple times produces the same result without unintended side effects. If the system is already in the desired state, Ansible reports "ok" and makes no changes. Most Ansible modules are idempotent by design. The `command` and `shell` modules are exceptions — use `creates` or `removes` parameters to add idempotency.

### 46. What is the difference between Ansible and Chef?

**Answer:**

| Feature | Ansible | Chef |
|---------|---------|------|
| Language | YAML | Ruby DSL |
| Architecture | Agentless (push) | Agent-based (pull) |
| Terminology | Playbooks, Tasks | Recipes, Cookbooks |
| Setup | Minimal | Requires Chef server + client |
| Learning curve | Low | High |
| Transport | SSH | Chef client daemon |

### 47. Which programming language is used to write Ansible Playbooks?

**Answer:** Ansible Playbooks are written in **YAML** (Yet Another Markup Language / YAML Ain't Markup Language). Ansible itself is written in Python, and modules can be written in any language that returns JSON.

### 48. Is Ansible open-source?

**Answer:** Yes, Ansible core is open-source under the GNU GPL v3 license. Red Hat also offers the commercial Ansible Automation Platform with enterprise support.

### 49. What are the server requirements for Ansible?

**Answer:**
- **Control node:** Linux/macOS with Python 3.9+ installed. Ansible does NOT run on Windows as a control node.
- **Managed nodes:** SSH access (Linux) or WinRM (Windows), Python 2.6+ or 3.5+ installed.
- No agent installation needed on managed nodes.

### 50. Can you connect to another device in Ansible?

**Answer:** Yes, use the `ping` module to test connectivity after adding the device to inventory:
```bash
ansible newhost -m ping -i inventory.ini
```

### 51. Can SREs create their own modules in Ansible?

**Answer:** Yes, you can write custom modules in any language (Python, Bash, Ruby, etc.) as long as they accept JSON input and return JSON output. Place custom modules in a `library/` directory in your project or in `~/.ansible/plugins/modules/`.

### 52. What is a Fact in Ansible?

**Answer:** Facts are system properties gathered automatically by Ansible from managed nodes during the "Gathering Facts" task. They include hostname, IP addresses, OS family, memory, disk space, etc. Access via `ansible_facts` or `ansible_*` variables:
```yaml
- debug:
    msg: "OS is {{ ansible_facts['os_family'] }}"
```
Disable with `gather_facts: false`.

### 53. What does `ask_pass` do in Ansible?

**Answer:** `ask_pass` (or `--ask-pass` / `-k` on CLI) controls whether Ansible prompts for an SSH password. Default is `false` (assumes SSH keys). Set to `true` in `ansible.cfg` if password-based SSH is used:
```ini
[defaults]
ask_pass = True
```

### 54. What does `ask_sudo_pass` do in Ansible?

**Answer:** `ask_sudo_pass` (or `--ask-become-pass` / `-K` on CLI) prompts for the sudo/become password when privilege escalation is required.

### 55. What does `ask_vault_pass` do in Ansible?

**Answer:** `ask_vault_pass` (or `--ask-vault-pass` on CLI) prompts for the vault password to decrypt vault-encrypted files during playbook execution. Alternatively, use `--vault-password-file` to pass it from a file.

### 56. What does `callback_plugin` do in Ansible?

**Answer:** Callback plugins control the output and behavior of Ansible when responding to events. They can customize the display format, send notifications, or log to external systems. Configure in `ansible.cfg`:
```ini
[defaults]
stdout_callback = yaml
callback_whitelist = timer, profile_tasks
```

### 57. How do you delegate tasks in Ansible?

**Answer:** Use the `delegate_to` keyword:
```yaml
- name: Add host to load balancer
  command: /usr/local/bin/add_to_lb {{ inventory_hostname }}
  delegate_to: lb.example.com

- name: Run locally
  command: echo "Running on control node"
  delegate_to: localhost
```

### 58. What is Ansible Register?

**Answer:** `register` captures the output of a task into a variable for later use:
```yaml
- name: Check disk space
  command: df -h /
  register: disk_output

- name: Show output
  debug:
    msg: "{{ disk_output.stdout }}"
```
The registered variable contains `stdout`, `stderr`, `rc`, `changed`, and more.

### 59. How does Ansible synchronize module function?

**Answer:** The `synchronize` module is a wrapper around `rsync` for efficient file transfer between hosts:
```yaml
- name: Sync files to remote
  synchronize:
    src: /local/path/
    dest: /remote/path/
    mode: push
```
It supports push/pull modes, delete options, and rsync filters.

---

## Scenario-Based / Practical Questions

### 60. How do you handle rolling updates with zero downtime?

**Answer:** Use the `serial` keyword to update hosts in batches:
```yaml
- name: Rolling update
  hosts: webservers
  serial: 2        # Update 2 hosts at a time
  tasks:
    - name: Pull latest code
      git: repo=https://github.com/myapp.git dest=/app
    - name: Restart service
      service: name=myapp state=restarted
```

### 61. How do you handle errors in Ansible?

**Answer:** Several mechanisms:
```yaml
tasks:
  - name: Might fail
    command: /bin/false
    ignore_errors: true        # Continue on failure

  - name: Rescue block
    block:
      - command: /bin/false
    rescue:
      - debug: msg="Task failed, running rescue"
    always:
      - debug: msg="This always runs"
```

### 62. What is the difference between `include` and `import` in Ansible?

**Answer:**
- **`import_*`** (static) — Processed at playbook parse time. All tasks are loaded upfront. Tags and conditionals apply to each imported task.
- **`include_*`** (dynamic) — Processed at runtime when encountered. Allows looping over included files. Tags on the include statement don't automatically apply to included tasks.

### 63. What is `ansible.cfg` and what is its precedence order?

**Answer:** `ansible.cfg` is the configuration file for Ansible. Precedence order (highest to lowest):
1. `ANSIBLE_CONFIG` environment variable
2. `./ansible.cfg` (current directory)
3. `~/.ansible.cfg` (home directory)
4. `/etc/ansible/ansible.cfg` (global)

### 64. What is the difference between `command` and `shell` module?

**Answer:**
- **`command`** — Executes commands without a shell. Does NOT support pipes, redirects, or environment variables.
- **`shell`** — Executes commands through `/bin/sh`. Supports pipes (`|`), redirects (`>`), and env vars.

```yaml
# command - simpler, safer
- command: ls /tmp

# shell - when you need shell features
- shell: cat /var/log/syslog | grep error > /tmp/errors.txt
```

### 65. How do you use Jinja2 templates in Ansible?

**Answer:** Use the `template` module with `.j2` files:
```yaml
- name: Deploy config
  template:
    src: app.conf.j2
    dest: /etc/app/app.conf
```
```jinja2
# app.conf.j2
server_name={{ ansible_hostname }}
port={{ http_port | default(8080) }}
workers={{ ansible_processor_vcpus * 2 }}
```

### 66. What are Ansible Collections?

**Answer:** Collections are a packaging format for distributing Ansible content — including roles, modules, plugins, and playbooks. Install from Galaxy:
```bash
ansible-galaxy collection install amazon.aws
ansible-galaxy collection install community.docker
```
Reference modules with FQCN: `amazon.aws.ec2_instance`, `community.docker.docker_container`.

### 67. How do you use dynamic inventory in Ansible?

**Answer:** Dynamic inventory scripts or plugins auto-discover hosts from cloud providers or CMDBs:
```yaml
# inventory/aws_ec2.yml
plugin: amazon.aws.aws_ec2
regions:
  - ap-south-1
filters:
  instance-state-name: running
keyed_groups:
  - key: tags.Environment
    prefix: env
compose:
  ansible_host: public_ip_address
```
Run: `ansible-playbook -i inventory/aws_ec2.yml playbook.yml`

### 68. How do you debug Ansible playbooks?

**Answer:**
```bash
# Increase verbosity
ansible-playbook site.yml -v      # verbose
ansible-playbook site.yml -vvv    # more verbose
ansible-playbook site.yml -vvvv   # connection debugging

# Use debug module
- debug: var=myvar
- debug: msg="Value is {{ myvar }}"

# Step through tasks
ansible-playbook site.yml --step

# Start at specific task
ansible-playbook site.yml --start-at-task="Install nginx"

# Check mode (dry run)
ansible-playbook site.yml --check --diff
```

### 69. What is `become` in Ansible?

**Answer:** `become` is the privilege escalation mechanism (replacing the deprecated `sudo`):
```yaml
- name: Install package
  apt:
    name: nginx
    state: present
  become: true              # Escalate privileges
  become_user: root         # Target user (default: root)
  become_method: sudo       # Method (default: sudo)
```

### 70. How do you manage secrets and sensitive data?

**Answer:**
1. **Ansible Vault** — Encrypt files or individual variables
2. **Environment variables** — Pass via `lookup('env', 'SECRET')`
3. **External secret managers** — HashiCorp Vault, AWS Secrets Manager, etc.
4. **no_log: true** — Prevent sensitive task output from being logged
```yaml
- name: Set password
  user:
    name: admin
    password: "{{ vault_admin_password }}"
  no_log: true
```

---

> **Source:** Questions adapted from [TestGorilla - 61 Ansible Interview Questions](https://www.testgorilla.com/blog/ansible-interview-questions/) with additional practical questions and detailed answers added for comprehensive preparation.
