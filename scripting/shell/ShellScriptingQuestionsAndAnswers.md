# Shell Scripting Interview Questions and Answers

A focused collection of shell scripting and Bash interview questions for DevOps engineers — covering shell types, variables, control flow, text processing, error handling, and real-world automation scripts.

---

## Table of Contents

- [Shell Types & Fundamentals](#shell-types--fundamentals)
- [Variables & Input](#variables--input)
- [Control Flow & Logic](#control-flow--logic)
- [Functions & Arrays](#functions--arrays)
- [Text Processing](#text-processing)
- [File Operations & I/O Redirection](#file-operations--io-redirection)
- [Error Handling & Debugging](#error-handling--debugging)
- [DevOps Automation Scripts](#devops-automation-scripts)
---

## Shell Types & Fundamentals

### 1. What is the difference between `sh`, `bash`, `zsh`, `ksh`, and `dash`?

**Answer:** These are all Unix shells — command-line interpreters — but with different features:

| Shell | Full Name | Key Characteristics |
|-------|-----------|-------------------|
| **sh** | Bourne Shell | Original Unix shell. POSIX-compliant. Minimal features. |
| **bash** | Bourne Again Shell | Most common on Linux. Superset of `sh`. Arrays, `[[ ]]`, brace expansion. |
| **zsh** | Z Shell | Default on macOS. Bash-compatible + advanced autocomplete, globbing, plugins (Oh My Zsh). |
| **ksh** | Korn Shell | Combines `sh` reliability with `csh` features. Common in enterprise Unix (AIX, Solaris). |
| **dash** | Debian Almquist Shell | Lightweight POSIX shell. Default `/bin/sh` on Debian/Ubuntu. Faster but fewer features than Bash. |

```bash
# Check your current shell
echo $SHELL
echo $0

# List available shells
cat /etc/shells

# Change default shell
chsh -s /bin/bash

# Run script with specific shell
bash script.sh
zsh script.sh
```

**For DevOps scripts:**
- Use `#!/bin/bash` for feature-rich scripts (arrays, `[[ ]]`, string manipulation)
- Use `#!/bin/sh` for maximum portability across systems (Docker alpine, minimal containers)
- If `#!/bin/sh` → avoid Bash-only features (`[[ ]]`, arrays, `{1..10}`)

---

### 2. What is a shell and what is shell scripting?

**Answer:** A **shell** is a command-line interpreter that provides a user interface for accessing OS services. **Shell scripting** is writing a series of commands in a file to be executed by the shell automatically.

```bash
# Create a script
cat > hello.sh << 'EOF'
#!/bin/bash
echo "Hello, $(whoami)! Today is $(date +%A)."
EOF

# Make executable and run
chmod +x hello.sh
./hello.sh
```

**Why shell scripting in DevOps?**
- Automate repetitive tasks (deployments, backups, monitoring)
- Glue between tools (Docker, Kubernetes, Terraform, Ansible)
- CI/CD pipeline scripts
- System administration and provisioning
- Log analysis and alerting

---

### 3. What is the shebang (`#!`) line?

**Answer:** The shebang tells the OS which interpreter to use when executing the script.

```bash
#!/bin/bash          # Use Bash
#!/bin/sh            # Use POSIX shell (portable)
#!/usr/bin/env bash  # Find bash in PATH (most portable)
#!/usr/bin/env python3  # Python script
```

**Best practice:** Use `#!/usr/bin/env bash` — it searches `PATH` for bash, so it works even if bash is installed in a non-standard location.

**Without shebang:** The script runs with the current shell, which may cause unexpected behavior if the script uses Bash-specific features.

---

### 4. What is the difference between `source script.sh`, `./script.sh`, and `bash script.sh`?

**Answer:**

| Method | New shell? | Variables persist? | Needs execute permission? |
|--------|-----------|-------------------|--------------------------|
| `source script.sh` (or `. script.sh`) | No (runs in current shell) | Yes | No |
| `./script.sh` | Yes (subshell) | No | Yes (`chmod +x`) |
| `bash script.sh` | Yes (subshell) | No | No |

```bash
# Example showing the difference
cat > test.sh << 'EOF'
#!/bin/bash
MY_VAR="hello"
cd /tmp
EOF

# source — affects current shell
source test.sh
echo $MY_VAR   # "hello"
pwd             # /tmp

# ./test.sh — subshell, changes don't persist
./test.sh
echo $MY_VAR   # empty
pwd             # original directory
```

**Use `source` when:** Loading environment variables, activating virtualenvs, sourcing `.bashrc`.

---

### 5. What are the different types of variables in shell scripting?

**Answer:**

| Type | Scope | Example |
|------|-------|---------|
| **Local variables** | Current shell only | `name="John"` |
| **Environment variables** | Inherited by child processes | `export PATH=/usr/bin:$PATH` |
| **Special variables** | Set by the shell | `$?`, `$$`, `$!`, `$0` |
| **Positional parameters** | Script arguments | `$1`, `$2`, `$@`, `$#` |

```bash
# Local variable
name="DevOps"
echo $name

# Environment variable (available to child processes)
export DB_HOST="localhost"

# Read-only variable
readonly PI=3.14

# Unset a variable
unset name

# Variable with default value
echo ${LANG:-"en_US.UTF-8"}    # Use default if unset
echo ${LANG:="en_US.UTF-8"}    # Set default if unset
```

---

## Variables & Input

### 6. What are positional parameters and special variables?

**Answer:**

| Variable | Meaning |
|----------|---------|
| `$0` | Script name |
| `$1` to `$9` | First to ninth argument |
| `${10}` | Tenth argument (braces required for 10+) |
| `$#` | Total number of arguments |
| `$@` | All arguments (each as separate word) |
| `$*` | All arguments (as single string) |
| `$$` | PID of current shell |
| `$!` | PID of last background process |
| `$?` | Exit status of last command (0=success) |
| `$_` | Last argument of previous command |

```bash
#!/bin/bash
echo "Script: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Arg count: $#"
echo "PID: $$"
echo "Last exit: $?"
```

---

### 7. What is the difference between `"$@"` and `"$*"`?

**Answer:**

```bash
#!/bin/bash
# Called as: ./script.sh "hello world" "foo bar"

echo "--- Using \"\$@\" ---"
for arg in "$@"; do
    echo "Arg: $arg"
done
# Arg: hello world
# Arg: foo bar

echo "--- Using \"\$*\" ---"
for arg in "$*"; do
    echo "Arg: $arg"
done
# Arg: hello world foo bar    ← All joined as one string
```

**Rule:** Always use `"$@"` (with quotes) when passing arguments to other commands — it preserves spaces in arguments.

---

### 8. How do you read user input and validate it?

**Answer:**

```bash
# Basic read
read -p "Enter your name: " name
echo "Hello, $name"

# Read with timeout
read -t 10 -p "Enter password (10s): " pass

# Silent input (passwords)
read -s -p "Password: " pass
echo

# Read with default value
read -p "Environment [prod]: " env
env=${env:-prod}

# Validate numeric input
read -p "Enter port: " port
if [[ "$port" =~ ^[0-9]+$ ]] && [ "$port" -ge 1 ] && [ "$port" -le 65535 ]; then
    echo "Valid port: $port"
else
    echo "Invalid port" >&2
    exit 1
fi

# Validate yes/no
read -p "Continue? (y/n): " answer
case "$answer" in
    [yY]|[yY][eE][sS]) echo "Proceeding..." ;;
    *) echo "Aborted"; exit 0 ;;
esac
```

---

### 9. How does string manipulation work in Bash?

**Answer:**

```bash
str="Hello, World! Welcome to DevOps"

# Length
echo ${#str}                    # 30

# Substring
echo ${str:0:5}                 # Hello
echo ${str:7:5}                 # World

# Replace
echo ${str/World/Bash}          # Hello, Bash! Welcome to DevOps
echo ${str//o/0}                # Replace all 'o' with '0'

# Remove pattern (from front)
path="/home/user/scripts/deploy.sh"
echo ${path##*/}                # deploy.sh  (remove longest match from front)
echo ${path#*/}                 # home/user/scripts/deploy.sh (shortest)

# Remove pattern (from back)
echo ${path%/*}                 # /home/user/scripts  (remove shortest from back)
echo ${path%%/*}                # (empty — removes longest from back)

# Default values
echo ${UNSET_VAR:-"default"}    # Use default if unset
echo ${UNSET_VAR:="default"}    # Set and use default if unset
echo ${UNSET_VAR:+"alternate"}  # Use alternate if set
${REQUIRED_VAR:?"Error: var not set"}  # Exit with error if unset

# Uppercase/lowercase (Bash 4+)
echo ${str^^}                   # HELLO, WORLD! WELCOME TO DEVOPS
echo ${str,,}                   # hello, world! welcome to devops
```

---

## Control Flow & Logic

### 10. How do you write `if` conditions in shell?

**Answer:**

```bash
# Numeric comparison
if [ "$age" -ge 18 ]; then
    echo "Adult"
elif [ "$age" -ge 13 ]; then
    echo "Teenager"
else
    echo "Child"
fi

# String comparison
if [ "$env" = "prod" ]; then
    echo "Production"
fi

# File tests
if [ -f "/etc/nginx/nginx.conf" ]; then
    echo "Config exists"
fi
if [ -d "/var/log" ]; then
    echo "Directory exists"
fi
```

**Comparison operators:**

| Numeric | String | Meaning |
|---------|--------|---------|
| `-eq` | `=` or `==` | Equal |
| `-ne` | `!=` | Not equal |
| `-gt` | | Greater than |
| `-ge` | | Greater or equal |
| `-lt` | | Less than |
| `-le` | | Less or equal |
| | `-z` | String is empty |
| | `-n` | String is not empty |

**File test operators:**

| Operator | Test |
|----------|------|
| `-f` | Regular file exists |
| `-d` | Directory exists |
| `-e` | Any file/directory exists |
| `-r` | Readable |
| `-w` | Writable |
| `-x` | Executable |
| `-s` | File exists and not empty |
| `-L` | Symbolic link |

---

### 11. What is the difference between `[ ]` and `[[ ]]`?

**Answer:**

| Feature | `[ ]` (test) | `[[ ]]` (Bash built-in) |
|---------|-------------|------------------------|
| **Standard** | POSIX (works in sh) | Bash/Zsh only |
| **Word splitting** | Yes (must quote variables) | No |
| **Pattern matching** | No | Yes (`==` with glob) |
| **Regex** | No | Yes (`=~`) |
| **Logical operators** | `-a`, `-o` | `&&`, `||` |

```bash
# [ ] — POSIX, portable
if [ -f "$file" ] && [ -r "$file" ]; then
    echo "Readable file"
fi

# [[ ]] — Bash, more powerful
if [[ "$string" == *.log ]]; then
    echo "Log file"
fi

# Regex matching (Bash only)
if [[ "$email" =~ ^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$ ]]; then
    echo "Valid email"
fi

# Safe with unquoted variables (no word splitting)
if [[ $var == "hello" ]]; then
    echo "Match"
fi
```

**Rule of thumb:** Use `[[ ]]` in Bash scripts. Use `[ ]` when writing portable `sh` scripts.

---

### 12. How do you write loops in shell?

**Answer:**

```bash
# For loop
for i in {1..5}; do
    echo "Count: $i"
done

# C-style for loop
for ((i=0; i<10; i++)); do
    echo $i
done

# Loop over files
for file in /var/log/*.log; do
    echo "Processing: $file"
done

# Loop over command output
for pod in $(kubectl get pods -o name); do
    echo "Pod: $pod"
done

# While loop
count=0
while [ $count -lt 5 ]; do
    echo "Count: $count"
    ((count++))
done

# Read file line by line
while IFS= read -r line; do
    echo "Line: $line"
done < /etc/hosts

# Until loop (runs until condition is true)
until kubectl get pod myapp --no-headers | grep -q "Running"; do
    echo "Waiting for pod..."
    sleep 5
done

# Infinite loop
while true; do
    check_health || alert
    sleep 60
done
```

---

### 13. How does the `case` statement work?

**Answer:**

```bash
#!/bin/bash
case "$1" in
    start)
        echo "Starting service..."
        systemctl start myapp
        ;;
    stop)
        echo "Stopping service..."
        systemctl stop myapp
        ;;
    restart)
        echo "Restarting..."
        systemctl restart myapp
        ;;
    status)
        systemctl status myapp
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

**Pattern matching in case:**
```bash
case "$input" in
    [yY]|[yY][eE][sS])  echo "Yes" ;;
    [nN]|[nN][oO])       echo "No" ;;
    *.tar.gz)            tar xzf "$input" ;;
    *.zip)               unzip "$input" ;;
    *)                   echo "Unknown" ;;
esac
```

---

### 14. What are `&&`, `||`, and `;` operators?

**Answer:**

| Operator | Behavior | Example |
|----------|----------|---------|
| `&&` | Run next only if previous **succeeded** | `make && make install` |
| `||` | Run next only if previous **failed** | `cd /app || exit 1` |
| `;` | Run next **regardless** of previous result | `echo "start"; echo "end"` |

```bash
# Common DevOps patterns

# Exit if directory doesn't exist
cd /opt/deploy || { echo "Deploy dir missing"; exit 1; }

# Run tests, deploy only if tests pass
npm test && npm run deploy

# Try primary, fallback to secondary
curl -s http://primary:8080/health || curl -s http://secondary:8080/health

# Ensure cleanup happens
mkdir -p /tmp/build && build_app; rm -rf /tmp/build
```

---

## Functions & Arrays

### 15. How do you define and use functions?

**Answer:**

```bash
# Function definition
greet() {
    local name="$1"        # local variable (doesn't leak outside)
    local greeting="${2:-Hello}"
    echo "$greeting, $name!"
}

# Call function
greet "World"              # Hello, World!
greet "DevOps" "Hey"       # Hey, DevOps!

# Function with return value
is_running() {
    systemctl is-active --quiet "$1"
    return $?    # 0 = running, non-zero = not running
}

if is_running nginx; then
    echo "Nginx is running"
fi

# Function returning string (via stdout)
get_ip() {
    hostname -I | awk '{print $1}'
}
my_ip=$(get_ip)
echo "My IP: $my_ip"
```

**Best practices:**
- Always use `local` for function variables
- Use meaningful function names
- Return exit codes (0=success) via `return`
- Return data via `echo` (capture with `$(...)`)

---

### 16. How do you use arrays in Bash?

**Answer:**

```bash
# Indexed array
servers=("web1" "web2" "db1" "db2")
echo ${servers[0]}           # web1
echo ${servers[@]}           # All elements
echo ${#servers[@]}          # Length: 4

# Add element
servers+=("cache1")

# Loop over array
for server in "${servers[@]}"; do
    echo "Checking: $server"
    ping -c 1 "$server" &>/dev/null && echo "  UP" || echo "  DOWN"
done

# Associative array (Bash 4+)
declare -A config
config[host]="localhost"
config[port]="5432"
config[db]="myapp"

echo "Connecting to ${config[host]}:${config[port]}/${config[db]}"

# Array from command output
pods=($(kubectl get pods -o name))
for pod in "${pods[@]}"; do
    kubectl logs "$pod" --tail=5
done
```

---

## Text Processing

### 17. Explain `grep`, `sed`, and `awk` for DevOps use cases.

**Answer:**

```bash
# grep — filter lines
grep "ERROR" /var/log/app.log
grep -c "ERROR" /var/log/app.log            # Count matches
grep -ri "connection refused" /var/log/      # Recursive, case-insensitive
grep -v "DEBUG" app.log                      # Exclude DEBUG lines
grep -A 3 -B 1 "Exception" app.log          # 3 lines after, 1 before

# sed — stream editor (find & replace)
sed 's/8080/9090/g' config.yml              # Replace (stdout)
sed -i 's/8080/9090/g' config.yml           # In-place edit
sed -n '10,20p' file.txt                    # Print lines 10-20
sed '/^#/d' config.conf                     # Delete comment lines
sed -i '/^$/d' file.txt                     # Delete empty lines

# awk — column extraction & processing
awk '{print $1, $4}' access.log             # Print columns 1 and 4
awk -F: '{print $1}' /etc/passwd            # Custom delimiter
df -h | awk '$5+0 > 80 {print $1, $5}'     # Disks above 80%
kubectl get pods | awk 'NR>1 {print $1, $3}' # Skip header, print name+status
```

**When to use which:**

| Tool | Best for |
|------|----------|
| `grep` | Filtering lines matching a pattern |
| `sed` | Find & replace, line deletion/insertion |
| `awk` | Column extraction, conditional processing, calculations |

---

### 18. How do you use `cut`, `sort`, `uniq`, `tr`, and `xargs`?

**Answer:**

```bash
# cut — extract columns
cut -d: -f1 /etc/passwd                     # First field, delimiter ':'
cut -c1-10 file.txt                         # First 10 characters

# sort — sort lines
sort file.txt                                # Alphabetical
sort -n file.txt                             # Numeric
sort -t: -k3 -n /etc/passwd                 # Sort by 3rd field numerically
sort -u file.txt                             # Sort + unique

# uniq — remove duplicates (requires sorted input)
sort access.log | uniq -c | sort -rn | head  # Top 10 most frequent lines

# tr — translate/delete characters
echo "Hello" | tr 'a-z' 'A-Z'               # HELLO
echo "  extra   spaces  " | tr -s ' '       # Squeeze spaces
cat file.txt | tr -d '\r'                    # Remove Windows line endings

# xargs — build command from stdin
find /tmp -name "*.log" -mtime +7 | xargs rm
kubectl get pods --no-headers | awk '/Error/{print $1}' | xargs kubectl delete pod
echo "web1 web2 web3" | xargs -n1 ping -c1  # Ping each one
```

---

## File Operations & I/O Redirection

### 19. How does I/O redirection work?

**Answer:**

```bash
# File descriptors: 0=stdin, 1=stdout, 2=stderr

# Redirect stdout
command > file.txt       # Overwrite
command >> file.txt      # Append

# Redirect stderr
command 2> error.log

# Redirect both separately
command > output.log 2> error.log

# Redirect both to same file
command > all.log 2>&1   # POSIX
command &> all.log       # Bash shorthand

# Discard output
command > /dev/null 2>&1

# Pipe stdout to next command
cat access.log | grep "404" | wc -l

# Here document
cat > config.yaml << EOF
server:
  host: ${HOST}
  port: ${PORT}
EOF

# Here string
grep "admin" <<< "$user_list"
```

---

### 20. How do you work with files in shell scripts?

**Answer:**

```bash
# Check file existence
if [ -f "$file" ]; then echo "File exists"; fi
if [ -d "$dir" ]; then echo "Dir exists"; fi
if [ -s "$file" ]; then echo "File is not empty"; fi

# Read file line by line
while IFS= read -r line; do
    process "$line"
done < input.txt

# Write to file
echo "data" > file.txt          # Overwrite
echo "more data" >> file.txt    # Append

# Process CSV
while IFS=',' read -r name age city; do
    echo "Name=$name, Age=$age, City=$city"
done < people.csv

# Create temp file safely
tmpfile=$(mktemp /tmp/myapp.XXXXXX)
trap "rm -f $tmpfile" EXIT     # Cleanup on exit
echo "data" > "$tmpfile"

# File locking (prevent concurrent access)
exec 200>/var/lock/myapp.lock
flock -n 200 || { echo "Already running"; exit 1; }
```

---

## Error Handling & Debugging

### 21. How do you handle errors in shell scripts?

**Answer:**

```bash
#!/bin/bash

# Strict mode — highly recommended for DevOps scripts
set -euo pipefail

# set -e  → Exit immediately on any error
# set -u  → Treat unset variables as errors
# set -o pipefail → Pipe fails if any command in pipe fails

# Check exit status manually
if ! docker build -t myapp .; then
    echo "Build failed!" >&2
    exit 1
fi

# Using $?
command_that_might_fail
if [ $? -ne 0 ]; then
    echo "Command failed with exit code $?"
fi

# Trap for cleanup
cleanup() {
    echo "Cleaning up..."
    rm -rf "$tmpdir"
    docker rm -f "$container_id" 2>/dev/null
}
trap cleanup EXIT         # Run cleanup on exit (success or failure)
trap cleanup ERR          # Run cleanup on error

# Custom error handler
error_handler() {
    echo "ERROR: Line $1 failed with exit code $2" >&2
}
trap 'error_handler ${LINENO} $?' ERR
```

**Common exit codes:**

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General error |
| 2 | Misuse of shell command |
| 126 | Permission denied (not executable) |
| 127 | Command not found |
| 128+n | Killed by signal n (e.g., 137 = SIGKILL) |

---

### 22. How do you debug a shell script?

**Answer:**

```bash
# Method 1: set -x (trace execution)
#!/bin/bash
set -x    # Print each command before executing
deploy_app
set +x    # Stop tracing

# Method 2: Run with -x flag
bash -x script.sh

# Method 3: Debug specific section
set -x
problematic_function
set +x

# Method 4: Add logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*" >&2
}
log "Starting deployment..."
log "Building image: $IMAGE"

# Method 5: Verbose mode
bash -v script.sh    # Print lines as they are read (before expansion)

# Method 6: Check syntax without running
bash -n script.sh    # Syntax check only

# Method 7: Use PS4 for detailed trace
export PS4='+${BASH_SOURCE}:${LINENO}: ${FUNCNAME[0]:+${FUNCNAME[0]}(): }'
set -x
```

---

### 23. What is the `trap` command?

**Answer:** `trap` catches signals and executes cleanup code when they occur.

```bash
#!/bin/bash

# Cleanup on exit (always runs)
trap 'rm -f /tmp/lockfile; echo "Cleaned up"' EXIT

# Cleanup on Ctrl+C
trap 'echo "Interrupted!"; exit 130' INT

# Cleanup on termination signal
trap 'echo "Terminated!"; exit 143' TERM

# Ignore a signal
trap '' HUP    # Ignore hangup

# Common DevOps pattern
tmpdir=$(mktemp -d)
trap "rm -rf '$tmpdir'" EXIT

# Download, process, cleanup happens automatically
curl -o "$tmpdir/data.tar.gz" https://example.com/data.tar.gz
tar xzf "$tmpdir/data.tar.gz" -C "$tmpdir"
process_data "$tmpdir"
# tmpdir is cleaned up when script exits
```

**Common signals:**

| Signal | Number | Trigger |
|--------|--------|---------|
| `EXIT` | — | Script exits (any reason) |
| `INT` | 2 | Ctrl+C |
| `TERM` | 15 | `kill` command |
| `HUP` | 1 | Terminal closed |
| `ERR` | — | Any command fails (with `set -e`) |

---

## DevOps Automation Scripts

### 24. Write a script to check if a service is running and restart it if not.

**Answer:**

```bash
#!/bin/bash
set -euo pipefail

SERVICE="nginx"
MAX_RETRIES=3
RETRY_DELAY=5

check_service() {
    systemctl is-active --quiet "$SERVICE"
}

restart_service() {
    local attempt=1
    while [ $attempt -le $MAX_RETRIES ]; do
        echo "Attempt $attempt: Restarting $SERVICE..."
        sudo systemctl restart "$SERVICE"
        sleep $RETRY_DELAY

        if check_service; then
            echo "$SERVICE restarted successfully."
            return 0
        fi
        ((attempt++))
    done
    echo "CRITICAL: $SERVICE failed to restart after $MAX_RETRIES attempts!" >&2
    return 1
}

if check_service; then
    echo "$SERVICE is running."
else
    echo "$SERVICE is NOT running!"
    restart_service
fi
```

---

### 25. Write a script to monitor disk usage and send an alert.

**Answer:**

```bash
#!/bin/bash
set -euo pipefail

THRESHOLD=80
ALERT_EMAIL="ops@example.com"
HOSTNAME=$(hostname)

check_disk() {
    df -h --output=pcent,target | tail -n +2 | while read -r usage mount; do
        usage_num=${usage%\%}
        if [ "$usage_num" -gt "$THRESHOLD" ]; then
            echo "WARNING: $mount is at ${usage} on $HOSTNAME"
        fi
    done
}

alerts=$(check_disk)
if [ -n "$alerts" ]; then
    echo "$alerts"
    # Uncomment to send email:
    # echo "$alerts" | mail -s "Disk Alert: $HOSTNAME" "$ALERT_EMAIL"
fi
```

---

### 26. Write a script to rotate and compress log files.

**Answer:**

```bash
#!/bin/bash
set -euo pipefail

LOG_DIR="/var/log/myapp"
MAX_AGE=30     # days
MAX_SIZE=100   # MB

# Compress logs older than 1 day
find "$LOG_DIR" -name "*.log" -mtime +1 -exec gzip {} \;

# Delete compressed logs older than MAX_AGE days
find "$LOG_DIR" -name "*.log.gz" -mtime +$MAX_AGE -delete

# Truncate active log files larger than MAX_SIZE MB
for logfile in "$LOG_DIR"/*.log; do
    [ -f "$logfile" ] || continue
    size_mb=$(du -m "$logfile" | cut -f1)
    if [ "$size_mb" -gt "$MAX_SIZE" ]; then
        echo "Truncating $logfile (${size_mb}MB)"
        cp "$logfile" "${logfile}.$(date +%Y%m%d)"
        truncate -s 0 "$logfile"
    fi
done

echo "Log rotation complete: $(date)"
```

---

### 27. Write a script to perform health checks on multiple servers.

**Answer:**

```bash
#!/bin/bash
set -uo pipefail

SERVERS=("web1:80" "web2:80" "api:8080" "db:5432")
TIMEOUT=5

for entry in "${SERVERS[@]}"; do
    host="${entry%%:*}"
    port="${entry##*:}"

    if timeout "$TIMEOUT" bash -c "echo > /dev/tcp/$host/$port" 2>/dev/null; then
        echo "[OK]   $host:$port"
    else
        echo "[FAIL] $host:$port" >&2
    fi
done
```

---

### 28. Write a script to automate Docker image build and push.

**Answer:**

```bash
#!/bin/bash
set -euo pipefail

REGISTRY="registry.example.com"
IMAGE_NAME="myapp"
GIT_SHA=$(git rev-parse --short HEAD)
BRANCH=$(git rev-parse --abbrev-ref HEAD)
TAG="${GIT_SHA}"

echo "Building ${IMAGE_NAME}:${TAG}..."
docker build -t "${REGISTRY}/${IMAGE_NAME}:${TAG}" \
             -t "${REGISTRY}/${IMAGE_NAME}:latest" \
             --build-arg BUILD_DATE="$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
             --build-arg GIT_SHA="${GIT_SHA}" \
             .

echo "Running tests..."
docker run --rm "${REGISTRY}/${IMAGE_NAME}:${TAG}" npm test

echo "Pushing to registry..."
docker push "${REGISTRY}/${IMAGE_NAME}:${TAG}"
if [ "$BRANCH" = "main" ]; then
    docker push "${REGISTRY}/${IMAGE_NAME}:latest"
fi

echo "Deployed: ${REGISTRY}/${IMAGE_NAME}:${TAG}"
```

---

### 29. Write a script to parse and analyze access logs.

**Answer:**

```bash
#!/bin/bash
set -euo pipefail

LOG_FILE="${1:-/var/log/nginx/access.log}"

if [ ! -f "$LOG_FILE" ]; then
    echo "Log file not found: $LOG_FILE" >&2
    exit 1
fi

echo "=== Access Log Analysis: $LOG_FILE ==="
echo

echo "--- Top 10 IP Addresses ---"
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10

echo
echo "--- Top 10 Requested URLs ---"
awk '{print $7}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -10

echo
echo "--- HTTP Status Code Distribution ---"
awk '{print $9}' "$LOG_FILE" | sort | uniq -c | sort -rn

echo
echo "--- Requests Per Hour ---"
awk '{print $4}' "$LOG_FILE" | cut -d: -f2 | sort | uniq -c

echo
echo "--- 5xx Errors ---"
awk '$9 ~ /^5/ {print $1, $7, $9}' "$LOG_FILE" | head -20
```

---

### 30. What are best practices for writing production shell scripts?

**Answer:**

```bash
#!/bin/bash
# Best practice template for DevOps scripts

set -euo pipefail     # Strict mode: fail fast
IFS=$'\n\t'           # Safer word splitting

# Constants
readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
readonly SCRIPT_NAME="$(basename "$0")"
readonly LOG_FILE="/var/log/${SCRIPT_NAME%.sh}.log"

# Logging
log()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [INFO]  $*" | tee -a "$LOG_FILE"; }
warn() { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [WARN]  $*" | tee -a "$LOG_FILE" >&2; }
err()  { echo "[$(date '+%Y-%m-%d %H:%M:%S')] [ERROR] $*" | tee -a "$LOG_FILE" >&2; }

# Cleanup
cleanup() {
    # Remove temp files, release locks, etc.
    log "Cleanup complete"
}
trap cleanup EXIT

# Usage
usage() {
    cat << EOF
Usage: $SCRIPT_NAME [OPTIONS]

Options:
    -e, --env       Environment (dev|staging|prod)
    -v, --verbose   Enable verbose output
    -h, --help      Show this help
EOF
    exit 0
}

# Parse arguments
ENV=""
VERBOSE=false
while [[ $# -gt 0 ]]; do
    case "$1" in
        -e|--env)     ENV="$2"; shift 2 ;;
        -v|--verbose) VERBOSE=true; shift ;;
        -h|--help)    usage ;;
        *)            err "Unknown option: $1"; usage ;;
    esac
done

# Validate required args
[[ -z "$ENV" ]] && { err "Environment is required"; usage; }

# Main logic
main() {
    log "Starting deployment to $ENV..."
    # ... script logic ...
    log "Deployment complete"
}

main "$@"
```

**Key practices summary:**
- **Always use `set -euo pipefail`** — fail fast on errors
- **Quote all variables** — `"$var"` not `$var`
- **Use `local` in functions** — prevent variable leakage
- **Use `trap` for cleanup** — temp files, locks, processes
- **Add logging** — timestamps, levels, log to file
- **Validate inputs** — check arguments, files, permissions
- **Use `readonly` for constants** — prevent accidental changes
- **Add usage/help** — `-h` flag for documentation
- **Use `shellcheck`** — lint your scripts in CI

---

### 31. What is `shellcheck` and how do you use it?

**Answer:** `shellcheck` is a static analysis tool for shell scripts — it catches bugs, syntax issues, and bad practices.

```bash
# Install
brew install shellcheck      # macOS
apt install shellcheck       # Ubuntu

# Run
shellcheck script.sh

# Common warnings it catches:
# SC2086 — Double quote to prevent globbing/splitting
# SC2046 — Quote to prevent word splitting
# SC2034 — Variable appears unused
# SC2155 — Declare and assign separately to avoid masking return values
# SC2006 — Use $(...) instead of legacy `...`
```

**Example fixes:**
```bash
# BAD (SC2086):
rm $file                     # Breaks if file has spaces

# GOOD:
rm "$file"

# BAD (SC2155):
local output=$(command)      # Masks exit code

# GOOD:
local output
output=$(command)

# BAD (SC2006):
files=`ls *.txt`

# GOOD:
files=$(ls *.txt)
```

**CI integration:**
```yaml
# GitHub Actions
- name: Lint shell scripts
  run: find . -name "*.sh" -exec shellcheck {} +
```
