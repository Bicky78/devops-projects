# Linux for DevOps Interview Questions and Answers

Focused on what DevOps engineers actually get asked — commands, troubleshooting, services, networking, and system administration.

---

## Table of Contents

- [Core Concepts & Commands](#core-concepts--commands)
- [File System & Permissions](#file-system--permissions)
- [Process & Service Management](#process--service-management)
- [Networking](#networking)
- [Troubleshooting Scenarios](#troubleshooting-scenarios)

---

## Core Concepts & Commands

### 1. What is the Linux kernel?

**Answer:** The kernel is the core of the OS that manages CPU, memory, devices, and processes. It acts as a bridge between hardware and applications.

```bash
# Check kernel version
uname -r

# Check OS details
cat /etc/os-release
```

**Kernel space vs User space:**
- **Kernel space:** Protected memory where the OS runs with full hardware access (ring 0)
- **User space:** Where applications run with restricted access — cannot directly touch hardware

---

### 2. What is the Linux boot sequence?

**Answer:**

```
BIOS/UEFI → Bootloader (GRUB) → Kernel → init/systemd → Services → Login
```

| Stage | What happens |
|-------|-------------|
| **BIOS/UEFI** | Hardware initialization, POST, finds bootloader |
| **GRUB** | Loads kernel, offers boot options |
| **Kernel** | Decompresses, initializes drivers, mounts root filesystem |
| **systemd** | PID 1 — starts services, mounts filesystems, sets targets |
| **Target** | `multi-user.target` (CLI) or `graphical.target` (GUI) |
| **Login** | TTY or SSH login prompt |

---

### 3. What are the most essential Linux commands for DevOps?

**Answer:**

| Category | Commands |
|----------|---------|
| **Files** | `ls`, `cp`, `mv`, `rm`, `find`, `cat`, `head`, `tail`, `less` |
| **Text processing** | `grep`, `awk`, `sed`, `sort`, `uniq`, `cut`, `wc`, `xargs` |
| **Disk** | `df -h`, `du -sh`, `lsblk`, `fdisk`, `mount` |
| **Memory** | `free -h`, `vmstat`, `swapon -s` |
| **Process** | `ps aux`, `top`, `htop`, `kill`, `pgrep`, `pkill` |
| **Network** | `ip addr`, `ss`, `curl`, `ping`, `traceroute`, `netstat`, `dig`, `nslookup` |
| **Services** | `systemctl start/stop/status/enable/disable` |
| **Logs** | `journalctl`, `tail -f`, `dmesg` |
| **Users** | `useradd`, `usermod`, `passwd`, `whoami`, `id` |
| **Archives** | `tar`, `gzip`, `zip`, `unzip` |
| **Remote** | `ssh`, `scp`, `rsync` |

---

### 4. Explain `grep`, `awk`, and `sed` — when do you use each?

**Answer:**

```bash
# grep — search/filter lines by pattern
grep "error" /var/log/syslog
grep -ri "timeout" /var/log/       # recursive, case-insensitive

# awk — extract/process columns from structured text
awk '{print $1, $4}' access.log    # print 1st and 4th columns
df -h | awk '$5 > 80 {print $0}'   # disks above 80% usage

# sed — find and replace text in streams/files
sed 's/old/new/g' file.txt         # replace all occurrences
sed -i 's/8080/9090/g' config.yml  # in-place edit
```

| Tool | Best for |
|------|----------|
| `grep` | Filtering lines matching a pattern |
| `awk` | Column extraction and conditional processing |
| `sed` | Stream editing and text replacement |

---

### 5. What is swap space?

**Answer:** Swap is disk space used as an extension of RAM when physical memory is exhausted. It prevents OOM (Out of Memory) crashes but is **much slower** than RAM.

```bash
# Check swap usage
free -h
swapon --show

# Create a swap file
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Make persistent (add to /etc/fstab)
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

---

### 6. What is the difference between hard links and soft links?

**Answer:**

| Feature | Hard Link | Soft (Symbolic) Link |
|---------|-----------|---------------------|
| **Points to** | Same inode (data) | File path (name) |
| **Cross filesystem** | No | Yes |
| **Original deleted** | Data still accessible | Link breaks (dangling) |
| **Directories** | Cannot link directories | Can link directories |

```bash
# Hard link
ln file.txt hardlink.txt

# Soft link
ln -s /path/to/file.txt symlink.txt
```

---

## File System & Permissions

### 7. Explain Linux file permissions.

**Answer:**

```
-rwxr-xr-- 1 user group 1024 Jan 1 12:00 file.txt
│├─┤├─┤├─┤
│ │   │  └── Others: r-- (read only)
│ │   └───── Group:  r-x (read + execute)
│ └───────── Owner:  rwx (read + write + execute)
└─────────── File type: - (regular), d (directory), l (link)
```

**Numeric (octal):** `r=4`, `w=2`, `x=1`

```bash
chmod 755 file.txt   # rwxr-xr-x
chmod 644 file.txt   # rw-r--r--
chmod 600 file.txt   # rw-------

# Change ownership
chown user:group file.txt

# Special permissions
chmod u+s file       # SUID — run as file owner
chmod g+s dir        # SGID — new files inherit group
chmod +t dir         # Sticky bit — only owner can delete
```

---

### 8. What is `umask`?

**Answer:** `umask` sets the default permissions for newly created files and directories by masking (removing) bits.

```bash
# Check current umask
umask        # e.g., 0022

# Default: files = 666 - umask, directories = 777 - umask
# umask 0022: files = 644 (rw-r--r--), dirs = 755 (rwxr-xr-x)

# Set umask
umask 0027   # files = 640, dirs = 750
```

---

### 9. How do you mount/unmount filesystems?

**Answer:**

```bash
# List block devices
lsblk

# Mount
sudo mount /dev/sdb1 /mnt/data

# Unmount
sudo umount /mnt/data

# Permanent mount — add to /etc/fstab
/dev/sdb1  /mnt/data  ext4  defaults  0  2
# Then: sudo mount -a
```

---

### 10. What is LVM and why use it?

**Answer:** LVM (Logical Volume Manager) provides flexible disk management — resize volumes without downtime.

```
Physical Volumes (PV) → Volume Groups (VG) → Logical Volumes (LV)
/dev/sda1, /dev/sdb1  →  vg_data          →  lv_app, lv_logs
```

```bash
# Extend a logical volume
sudo lvextend -L +10G /dev/vg_data/lv_app
sudo resize2fs /dev/vg_data/lv_app    # ext4
# or
sudo xfs_growfs /dev/vg_data/lv_app   # XFS
```

---

## Process & Service Management

### 11. How do you manage services with systemd?

**Answer:**

```bash
# Service management
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx     # Reload config without restart
sudo systemctl status nginx

# Enable/disable at boot
sudo systemctl enable nginx
sudo systemctl disable nginx

# Check failed services
systemctl --failed

# View service logs
journalctl -u nginx -f          # Follow logs
journalctl -u nginx --since "1 hour ago"
```

---

### 12. What is the difference between a process and a daemon?

**Answer:**

| Feature | Process | Daemon |
|---------|---------|--------|
| **Lifecycle** | Started by user, terminates | Runs in background continuously |
| **Terminal** | Attached to terminal | Detached from terminal |
| **Examples** | `ls`, `grep`, `python script.py` | `sshd`, `nginx`, `crond` |

```bash
# View all processes
ps aux

# View specific process
ps aux | grep nginx

# Find PID
pgrep nginx

# Kill process
kill 1234       # Graceful (SIGTERM)
kill -9 1234    # Force (SIGKILL)

# What are zombie processes?
# State Z in ps — child process finished but parent hasn't read its exit status
ps aux | grep Z
```

---

### 13. How do you schedule tasks with cron?

**Answer:**

```bash
# Edit crontab
crontab -e

# Format: MIN HOUR DOM MON DOW COMMAND
# ┌───── minute (0-59)
# │ ┌───── hour (0-23)
# │ │ ┌───── day of month (1-31)
# │ │ │ ┌───── month (1-12)
# │ │ │ │ ┌───── day of week (0-7)
# │ │ │ │ │
  0 2 * * * /opt/scripts/backup.sh        # Daily at 2 AM
  */5 * * * * /opt/scripts/health-check.sh  # Every 5 minutes
  0 0 * * 0 /opt/scripts/weekly-cleanup.sh  # Every Sunday midnight

# List cron jobs
crontab -l

# System-wide cron
ls /etc/cron.d/ /etc/cron.daily/ /etc/cron.hourly/
```

**Cron not running? Check:**
- `systemctl status crond` (or `cron`)
- Script has execute permission (`chmod +x`)
- Use absolute paths in script
- Check `/var/log/cron` or `journalctl -u cron`

---

### 14. What is `logrotate` and why is it important?

**Answer:** `logrotate` prevents logs from consuming all disk space by rotating, compressing, and removing old log files.

```bash
# Config: /etc/logrotate.conf and /etc/logrotate.d/

# Example: /etc/logrotate.d/myapp
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    missingok
    notifempty
    postrotate
        systemctl reload myapp
    endscript
}

# Test
logrotate -d /etc/logrotate.d/myapp  # Dry run
```

---

## Networking

### 15. How do you troubleshoot network connectivity?

**Answer:**

```bash
# Layer-by-layer approach:

# 1. Check interface is up
ip addr show
ip link show

# 2. Check IP and routing
ip route show
ping -c 3 8.8.8.8        # Test connectivity

# 3. Check DNS
cat /etc/resolv.conf
dig example.com
nslookup example.com

# 4. Check port connectivity
ss -tulpn                  # List listening ports
curl -v http://server:8080 # Test HTTP
telnet server 22           # Test port

# 5. Check firewall
sudo iptables -L -n
sudo ufw status
sudo firewall-cmd --list-all
```

---

### 16. How do you configure SSH securely?

**Answer:**

```bash
# Generate SSH key pair
ssh-keygen -t ed25519 -C "your.email@example.com"

# Copy key to remote server
ssh-copy-id user@server

# Harden /etc/ssh/sshd_config:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
Port 2222                    # Change default port
AllowUsers deploy admin

# Restart SSH
sudo systemctl restart sshd
```

---

### 17. What is the difference between `scp` and `rsync`?

**Answer:**

| Feature | `scp` | `rsync` |
|---------|-------|---------|
| **Transfer** | Full file every time | Only changed parts (delta) |
| **Resume** | No | Yes (`--partial`) |
| **Compression** | No built-in | Yes (`-z`) |
| **Bandwidth** | No limit option | `--bwlimit` |
| **Best for** | Quick single file copy | Large/repeated transfers, backups |

```bash
# scp
scp file.tar.gz user@server:/opt/

# rsync (preferred)
rsync -avz --progress /local/dir/ user@server:/remote/dir/
```

---

### 18. How do you check which process is using a specific port?

**Answer:**

```bash
# Using ss (modern)
ss -tulpn | grep :8080

# Using lsof
lsof -i :8080

# Using fuser
fuser 8080/tcp

# Kill process on port
fuser -k 8080/tcp
```

---

## Troubleshooting Scenarios

### 19. A server is slow. How do you diagnose?

**Answer:**

```bash
# Step 1: Quick overview
top           # CPU, memory, processes
uptime        # Load average (compare to CPU core count)

# Step 2: Identify bottleneck
vmstat 1 5    # CPU, I/O wait, context switches
iostat -x 1   # Disk I/O (look for high %util)
free -h       # Memory (is swap being used heavily?)

# Step 3: Find offending process
top -o %CPU   # Sort by CPU
top -o %MEM   # Sort by memory
iotop         # I/O per process

# Step 4: Check disk
df -h         # Disk full?
du -sh /var/* | sort -rh | head  # Largest directories

# Decision tree:
# High CPU → identify process → optimize or kill
# High I/O wait → disk issue → check IOPS, logs growing
# High memory → OOM risk → identify leak, add swap
# High load + low CPU → I/O blocked processes
```

---

### 20. Disk space on /var is 100% full. What do you do?

**Answer:**

```bash
# 1. Find what's consuming space
du -sh /var/* | sort -rh | head

# 2. Usually it's logs
du -sh /var/log/* | sort -rh | head

# 3. Clean safely
sudo journalctl --vacuum-size=100M   # Trim journal logs
sudo find /var/log -name "*.gz" -mtime +30 -delete  # Old compressed logs
sudo truncate -s 0 /var/log/large-logfile.log        # Empty active log

# 4. Check for deleted but open files (still holding space)
lsof +L1 | grep deleted

# 5. Restart service holding deleted files
sudo systemctl restart rsyslog

# 6. Set up logrotate to prevent recurrence
```

---

### 21. A service is running but users cannot access it. How do you debug?

**Answer:**

```bash
# 1. Is the service actually listening?
ss -tulpn | grep :80

# 2. Is the firewall blocking?
sudo iptables -L -n | grep 80
# or
sudo ufw status
sudo firewall-cmd --list-ports

# 3. Is SELinux blocking?
getenforce
# If Enforcing, check:
ausearch -m AVC --recent

# 4. Is the app bound to correct interface?
# 127.0.0.1:80 = localhost only
# 0.0.0.0:80 = all interfaces (correct for external access)

# 5. Test locally on the server
curl localhost:80

# 6. Check application logs
journalctl -u myservice -f
```

---

### 22. A user gets "Permission denied" error. What do you check?

**Answer:**

```bash
# 1. Check file permissions and ownership
ls -la /path/to/file

# 2. Check user groups
id username
groups username

# 3. Check ACL (if set)
getfacl /path/to/file

# 4. Check SELinux context
ls -Z /path/to/file

# 5. Check if directory has execute permission
# (needed to traverse directories)
ls -ld /path/to/

# 6. Check sudoers if running as sudo
sudo -l -U username
```

---

### 23. SSH login suddenly stops working. How do you troubleshoot?

**Answer:**

```bash
# 1. Check SSH service
sudo systemctl status sshd

# 2. Check if port is listening
ss -tulpn | grep :22

# 3. Check firewall
sudo iptables -L -n | grep 22

# 4. Check SSH config for errors
sudo sshd -t    # Test config syntax

# 5. Check auth logs
tail -f /var/log/auth.log          # Debian/Ubuntu
tail -f /var/log/secure            # RHEL/CentOS

# 6. Check user's authorized_keys permissions
ls -la ~/.ssh/
# Should be: .ssh = 700, authorized_keys = 600

# 7. Check account status
passwd -S username   # Locked?
chage -l username    # Expired?
```

---

### 24. A cron job is not executing. How do you investigate?

**Answer:**

```bash
# 1. Verify cron job exists
crontab -l

# 2. Check cron service
systemctl status crond    # RHEL
systemctl status cron     # Ubuntu

# 3. Check cron logs
grep CRON /var/log/syslog       # Ubuntu
journalctl -u crond             # RHEL

# 4. Common issues:
# - Script lacks execute permission → chmod +x script.sh
# - Uses relative paths → use absolute paths
# - Missing environment variables → source profile or define in crontab
# - Wrong user's crontab → check with sudo crontab -l
# - PATH not set → add PATH=/usr/bin:/bin at top of crontab
```

---

### 25. A server rebooted unexpectedly. How do you investigate?

**Answer:**

```bash
# 1. Check previous boot logs
journalctl -b -1         # Logs from previous boot
journalctl -b -1 -p err  # Only errors

# 2. Check for OOM kills
dmesg | grep -i "oom\|killed"
journalctl -b -1 | grep -i "oom"

# 3. Check for kernel panic
journalctl -b -1 | grep -i "panic"

# 4. Check system logs
last -x reboot     # Reboot history
last -x shutdown    # Shutdown history

# 5. Check for scheduled reboots
crontab -l | grep reboot
cat /etc/crontab | grep reboot

# 6. Check hardware (if applicable)
dmesg | grep -i "error\|fault\|fail"
```

---

### 26. How do you use the `find` command effectively?

**Answer:** `find` is essential for locating files, cleaning up disk, and scripting.

```bash
# Find files by name
find /var/log -name "*.log"
find / -iname "nginx.conf"          # Case-insensitive

# Find by size
find /tmp -size +100M               # Files larger than 100MB
find / -size +1G -exec ls -lh {} \; # List large files with details

# Find by modification time
find /var/log -mtime +30            # Modified more than 30 days ago
find /home -mmin -60                # Modified in last 60 minutes

# Find and delete
find /tmp -type f -mtime +7 -delete # Delete files older than 7 days

# Find and execute
find /app -name "*.bak" -exec rm {} \;
find /opt -type f -perm 777 -exec chmod 644 {} \;

# Find empty files/directories
find /var -empty -type f
find /home -empty -type d

# Combine conditions
find /var/log -name "*.log" -size +50M -mtime +30
```

---

### 27. How do you configure firewall rules in Linux?

**Answer:**

**iptables (traditional):**
```bash
# List rules
sudo iptables -L -n -v

# Allow SSH
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT

# Allow HTTP/HTTPS
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Block an IP
sudo iptables -A INPUT -s 10.0.0.5 -j DROP

# Allow established connections
sudo iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Default deny incoming
sudo iptables -P INPUT DROP

# Save rules
sudo iptables-save > /etc/iptables/rules.v4
```

**firewalld (RHEL/CentOS):**
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http --permanent
sudo firewall-cmd --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
```

**ufw (Ubuntu):**
```bash
sudo ufw status
sudo ufw allow 22/tcp
sudo ufw allow 80,443/tcp
sudo ufw deny from 10.0.0.5
sudo ufw enable
```

---

### 28. How do you use `journalctl` for log analysis?

**Answer:** `journalctl` queries the systemd journal — a centralized log for all services.

```bash
# View all logs
journalctl

# Follow logs in real-time
journalctl -f

# Logs for a specific service
journalctl -u nginx
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2024-01-01" --until "2024-01-02"

# Filter by priority (0=emergency to 7=debug)
journalctl -p err                  # Errors and above
journalctl -p warning -u sshd     # Warnings from sshd

# Logs from current/previous boot
journalctl -b                      # Current boot
journalctl -b -1                   # Previous boot
journalctl --list-boots             # List all boots

# Kernel messages
journalctl -k

# Output formats
journalctl -o json-pretty          # JSON format
journalctl -o short-precise        # Microsecond timestamps

# Disk usage and cleanup
journalctl --disk-usage
sudo journalctl --vacuum-size=200M  # Trim to 200MB
sudo journalctl --vacuum-time=7d    # Keep only last 7 days
```

---

### 29. What are systemd targets and how do they relate to runlevels?

**Answer:** Targets replaced traditional SysV runlevels — they define system states.

| SysV Runlevel | systemd Target | Description |
|--------------|----------------|-------------|
| 0 | `poweroff.target` | Shut down |
| 1 | `rescue.target` | Single-user / rescue mode |
| 3 | `multi-user.target` | Multi-user, no GUI (servers) |
| 5 | `graphical.target` | Multi-user with GUI |
| 6 | `reboot.target` | Reboot |

```bash
# Check current target
systemctl get-default

# Set default target
sudo systemctl set-default multi-user.target

# Switch target at runtime
sudo systemctl isolate rescue.target

# List all targets
systemctl list-units --type=target
```

---

### 30. How do you manage users and groups in Linux?

**Answer:**

```bash
# Create user
sudo useradd -m -s /bin/bash -G sudo devops
sudo passwd devops

# Modify user
sudo usermod -aG docker devops      # Add to docker group
sudo usermod -L devops               # Lock account
sudo usermod -U devops               # Unlock account

# Delete user
sudo userdel -r devops               # Remove user + home directory

# Group management
sudo groupadd developers
sudo groupdel developers
sudo gpasswd -a devops developers    # Add user to group
sudo gpasswd -d devops developers    # Remove user from group

# View user info
id devops
groups devops
cat /etc/passwd | grep devops
cat /etc/group | grep developers

# Sudo access
sudo visudo
# Add: devops ALL=(ALL) NOPASSWD: ALL

# Password policy
sudo chage -l devops                 # View password expiry
sudo chage -M 90 devops             # Max 90 days password age
```

**Important files:**
- `/etc/passwd` — User accounts
- `/etc/shadow` — Encrypted passwords
- `/etc/group` — Group definitions
- `/etc/sudoers` — Sudo privileges
