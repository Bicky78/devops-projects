# Python Scripting for DevOps Interview Questions and Answers

A focused collection of Python scripting interview questions for DevOps engineers — covering automation, APIs, file handling, subprocess, cloud SDKs, and real-world scripts.

---

## Table of Contents

- [Python Fundamentals for DevOps](#python-fundamentals-for-devops)
- [File & Data Handling](#file--data-handling)
- [System & Process Automation](#system--process-automation)
- [API & Network Automation](#api--network-automation)
- [Cloud & Infrastructure Scripting](#cloud--infrastructure-scripting)
- [Real-World DevOps Scripts](#real-world-devops-scripts)
- [Testing & Best Practices](#testing--best-practices)

---

## Python Fundamentals for DevOps

### 1. Why is Python used alongside shell scripting in DevOps?

**Answer:**

| Feature | Shell (Bash) | Python |
|---------|-------------|--------|
| **Best for** | Quick file/process tasks, glue scripts | Complex logic, APIs, data processing |
| **Error handling** | Basic (`$?`, `trap`) | Rich (try/except, custom exceptions) |
| **Data structures** | Arrays only | Dicts, lists, sets, classes |
| **API calls** | `curl` (basic) | `requests` library (full featured) |
| **Parsing** | `grep`/`awk`/`sed` | `json`, `yaml`, `re` modules |
| **Portability** | Linux/Mac only | Cross-platform |
| **Dependency** | Always available on Linux | Needs Python installed |

**Rule of thumb:**
- **< 50 lines, system tasks** → Shell
- **> 50 lines, APIs, complex logic** → Python
- **CI/CD glue** → Shell
- **Infrastructure automation** → Python (Boto3, Azure SDK)

---

### 2. What are the key Python data types a DevOps engineer should know?

**Answer:**

```python
# Strings — config values, paths, URLs
host = "web-server-01"
url = f"http://{host}:8080/health"

# Lists — ordered, mutable collections
servers = ["web1", "web2", "db1"]
servers.append("cache1")
for s in servers:
    print(s)

# Tuples — immutable, used for fixed data
endpoint = ("api.example.com", 443)

# Dictionaries — key-value pairs (configs, JSON)
config = {
    "host": "localhost",
    "port": 5432,
    "db": "myapp",
    "ssl": True,
}
print(config["host"])
print(config.get("timeout", 30))  # Default if missing

# Sets — unique values, fast lookups
active_hosts = {"web1", "web2", "web3"}
failed_hosts = {"web2", "web4"}
healthy = active_hosts - failed_hosts  # {"web1", "web3"}

# Boolean
is_production = os.getenv("ENV") == "production"
```

---

### 3. How do you use environment variables in Python?

**Answer:**

```python
import os

# Read environment variable (returns None if missing)
db_host = os.getenv("DB_HOST", "localhost")
db_port = int(os.getenv("DB_PORT", "5432"))

# Required variable (raises KeyError if missing)
api_key = os.environ["API_KEY"]

# Set environment variable (for child processes)
os.environ["MODE"] = "production"

# Check if variable exists
if "DEBUG" in os.environ:
    print("Debug mode enabled")

# Load .env file (requires python-dotenv)
from dotenv import load_dotenv
load_dotenv()  # Loads variables from .env file
secret = os.getenv("SECRET_KEY")
```

---

### 4. How do error handling and exceptions work in Python?

**Answer:**

```python
import sys
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Basic try/except
try:
    result = 10 / 0
except ZeroDivisionError as e:
    logger.error(f"Math error: {e}")

# Multiple exception types
try:
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    data = response.json()
except requests.ConnectionError:
    logger.error("Cannot connect to server")
    sys.exit(1)
except requests.Timeout:
    logger.error("Request timed out")
except requests.HTTPError as e:
    logger.error(f"HTTP error: {e.response.status_code}")
except Exception as e:
    logger.error(f"Unexpected error: {e}")
finally:
    logger.info("Cleanup complete")

# Custom exception
class DeploymentError(Exception):
    def __init__(self, service, message):
        self.service = service
        super().__init__(f"Deployment failed for {service}: {message}")

# Raise custom exception
def deploy(service, version):
    if not version:
        raise DeploymentError(service, "version is required")
```

---

### 5. What are list comprehensions and how are they useful in DevOps?

**Answer:**

```python
# Filter running pods
pods = [
    {"name": "web-1", "status": "Running"},
    {"name": "web-2", "status": "CrashLoopBackOff"},
    {"name": "db-1", "status": "Running"},
]

running = [p["name"] for p in pods if p["status"] == "Running"]
# ["web-1", "db-1"]

failed = [p["name"] for p in pods if p["status"] != "Running"]
# ["web-2"]

# Filter log lines
with open("/var/log/app.log") as f:
    errors = [line.strip() for line in f if "ERROR" in line]

# Generate server names
servers = [f"web-{i:02d}.prod.example.com" for i in range(1, 11)]
# ["web-01.prod.example.com", ..., "web-10.prod.example.com"]

# Dict comprehension
status_map = {p["name"]: p["status"] for p in pods}
# {"web-1": "Running", "web-2": "CrashLoopBackOff", "db-1": "Running"}
```

---

### 6. How do you use `argparse` for CLI tools?

**Answer:**

```python
#!/usr/bin/env python3
"""DevOps deployment CLI tool."""
import argparse
import sys

def parse_args():
    parser = argparse.ArgumentParser(description="Deploy application")
    parser.add_argument("service", help="Service name to deploy")
    parser.add_argument("-e", "--env", required=True,
                        choices=["dev", "staging", "prod"],
                        help="Target environment")
    parser.add_argument("-t", "--tag", default="latest",
                        help="Image tag (default: latest)")
    parser.add_argument("-d", "--dry-run", action="store_true",
                        help="Preview without deploying")
    parser.add_argument("-v", "--verbose", action="store_true",
                        help="Enable verbose output")
    return parser.parse_args()

def main():
    args = parse_args()
    print(f"Deploying {args.service}:{args.tag} to {args.env}")
    if args.dry_run:
        print("[DRY RUN] No changes made")
        return
    # ... deploy logic ...

if __name__ == "__main__":
    main()

# Usage:
# python deploy.py myapp -e prod -t v1.2.3
# python deploy.py myapp -e staging --dry-run
```

---

## File & Data Handling

### 7. How do you read and write files in Python?

**Answer:**

```python
from pathlib import Path

# Read entire file
content = Path("/etc/hosts").read_text()

# Read line by line (memory efficient for large files)
with open("/var/log/app.log") as f:
    for line in f:
        if "ERROR" in line:
            print(line.strip())

# Write to file
Path("/tmp/output.txt").write_text("Hello\n")

# Append to file
with open("/tmp/output.txt", "a") as f:
    f.write("More data\n")

# Read/write binary files
data = Path("archive.tar.gz").read_bytes()
Path("copy.tar.gz").write_bytes(data)

# Safe write with temp file (atomic)
import tempfile, shutil
with tempfile.NamedTemporaryFile(mode="w", delete=False) as tmp:
    tmp.write("new config data\n")
    tmp_path = tmp.name
shutil.move(tmp_path, "/etc/myapp/config.conf")
```

---

### 8. How do you parse JSON and YAML in Python?

**Answer:**

```python
import json
import yaml  # pip install pyyaml

# --- JSON ---
# Parse JSON string
data = json.loads('{"name": "myapp", "replicas": 3}')
print(data["replicas"])  # 3

# Read JSON file
with open("config.json") as f:
    config = json.load(f)

# Write JSON file (pretty-printed)
with open("output.json", "w") as f:
    json.dump(config, f, indent=2)

# --- YAML ---
# Read YAML file
with open("values.yaml") as f:
    values = yaml.safe_load(f)

# Write YAML file
with open("output.yaml", "w") as f:
    yaml.dump(values, f, default_flow_style=False)

# Parse Kubernetes manifest with multiple documents
with open("resources.yaml") as f:
    docs = list(yaml.safe_load_all(f))
    for doc in docs:
        print(doc["kind"], doc["metadata"]["name"])

# Convert YAML to JSON
with open("values.yaml") as f:
    data = yaml.safe_load(f)
print(json.dumps(data, indent=2))
```

---

### 9. How do you work with CSV and log files?

**Answer:**

```python
import csv
import re
from collections import Counter

# --- CSV ---
# Read CSV
with open("servers.csv") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(f"{row['hostname']}: {row['ip']}")

# Write CSV
servers = [
    {"hostname": "web1", "ip": "10.0.0.1", "role": "frontend"},
    {"hostname": "db1", "ip": "10.0.0.2", "role": "database"},
]
with open("inventory.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["hostname", "ip", "role"])
    writer.writeheader()
    writer.writerows(servers)

# --- Log Parsing ---
# Count HTTP status codes from access log
status_codes = Counter()
with open("/var/log/nginx/access.log") as f:
    for line in f:
        match = re.search(r'" (\d{3}) ', line)
        if match:
            status_codes[match.group(1)] += 1

for code, count in status_codes.most_common(10):
    print(f"  {code}: {count}")
```

---

### 10. How do you use `pathlib` for file system operations?

**Answer:**

```python
from pathlib import Path
import shutil

# Path operations
config = Path("/etc/nginx/nginx.conf")
print(config.exists())        # True/False
print(config.is_file())       # True
print(config.name)            # nginx.conf
print(config.stem)            # nginx
print(config.suffix)          # .conf
print(config.parent)          # /etc/nginx

# Find files
log_dir = Path("/var/log")
for log_file in log_dir.glob("**/*.log"):    # Recursive
    size_mb = log_file.stat().st_size / (1024 * 1024)
    if size_mb > 100:
        print(f"Large log: {log_file} ({size_mb:.1f}MB)")

# Create directories
deploy_dir = Path("/opt/releases/v1.2.3")
deploy_dir.mkdir(parents=True, exist_ok=True)

# Copy and move
shutil.copy2(src, dst)       # Copy with metadata
shutil.copytree(src, dst)    # Copy directory tree
shutil.move(src, dst)        # Move file/directory

# Delete
Path("/tmp/old_file.txt").unlink(missing_ok=True)
shutil.rmtree("/tmp/old_dir")
```

---

## System & Process Automation

### 11. How do you run shell commands from Python?

**Answer:**

```python
import subprocess

# Simple command
result = subprocess.run(["ls", "-la", "/var/log"],
                        capture_output=True, text=True)
print(result.stdout)
print(result.returncode)    # 0 = success

# Check for errors (raises CalledProcessError on failure)
result = subprocess.run(
    ["kubectl", "get", "pods", "-o", "json"],
    capture_output=True, text=True, check=True
)

# Shell command with pipes (use shell=True carefully)
result = subprocess.run(
    "ps aux | grep nginx | grep -v grep",
    shell=True, capture_output=True, text=True
)

# Run with timeout
try:
    result = subprocess.run(
        ["docker", "build", "-t", "myapp", "."],
        capture_output=True, text=True, timeout=300
    )
except subprocess.TimeoutExpired:
    print("Build timed out!")

# Stream output in real-time
process = subprocess.Popen(
    ["docker", "logs", "-f", "mycontainer"],
    stdout=subprocess.PIPE, text=True
)
for line in process.stdout:
    print(line, end="")
```

**Security note:** Avoid `shell=True` with user input — it's vulnerable to shell injection. Pass commands as lists instead.

---

### 12. How do you handle processes and signals in Python?

**Answer:**

```python
import signal
import sys
import os
import subprocess

# Graceful shutdown handler
def signal_handler(signum, frame):
    print(f"\nReceived signal {signum}, shutting down gracefully...")
    cleanup()
    sys.exit(0)

signal.signal(signal.SIGTERM, signal_handler)
signal.signal(signal.SIGINT, signal_handler)   # Ctrl+C

# Check if process is running
def is_process_running(pid):
    try:
        os.kill(pid, 0)   # Signal 0 = check existence
        return True
    except ProcessError:
        return False

# Run background process
proc = subprocess.Popen(["python", "worker.py"])
print(f"Started worker PID: {proc.pid}")

# Wait with timeout
try:
    proc.wait(timeout=60)
except subprocess.TimeoutExpired:
    proc.terminate()       # SIGTERM
    proc.wait(timeout=10)  # Wait for graceful shutdown
    proc.kill()            # SIGKILL if still running
```

---

### 13. How do you use `concurrent.futures` for parallel execution?

**Answer:**

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import subprocess

servers = ["web1", "web2", "web3", "db1", "db2"]

def check_server(host):
    """Ping a server and return status."""
    result = subprocess.run(
        ["ping", "-c", "1", "-W", "2", host],
        capture_output=True
    )
    return host, result.returncode == 0

# Run checks in parallel (threads — good for I/O bound tasks)
with ThreadPoolExecutor(max_workers=10) as executor:
    futures = {executor.submit(check_server, s): s for s in servers}

    for future in as_completed(futures):
        host, is_up = future.result()
        status = "UP" if is_up else "DOWN"
        print(f"  [{status:4}] {host}")

# ProcessPoolExecutor for CPU-bound tasks
from concurrent.futures import ProcessPoolExecutor

def analyze_log(log_file):
    """Count errors in a log file."""
    count = 0
    with open(log_file) as f:
        for line in f:
            if "ERROR" in line:
                count += 1
    return log_file, count

log_files = list(Path("/var/log/myapp/").glob("*.log"))
with ProcessPoolExecutor() as executor:
    for log_file, error_count in executor.map(analyze_log, log_files):
        print(f"  {log_file.name}: {error_count} errors")
```

---

## API & Network Automation

### 14. How do you make HTTP requests with `requests`?

**Answer:**

```python
import requests

# GET request
response = requests.get("https://api.github.com/repos/kubernetes/kubernetes",
                        timeout=10)
response.raise_for_status()   # Raise exception for 4xx/5xx
data = response.json()
print(f"Stars: {data['stargazers_count']}")

# POST request with JSON body
response = requests.post(
    "https://slack.com/api/chat.postMessage",
    headers={"Authorization": f"Bearer {SLACK_TOKEN}"},
    json={
        "channel": "#alerts",
        "text": "Deployment complete :white_check_mark:"
    },
    timeout=10
)

# With authentication
response = requests.get(
    "https://api.example.com/v1/servers",
    headers={"Authorization": f"Bearer {token}"},
    params={"status": "active", "limit": 100},
    timeout=10
)

# Upload file
with open("report.pdf", "rb") as f:
    response = requests.post(
        "https://api.example.com/upload",
        files={"file": f},
        timeout=30
    )

# Session (reuses connection + cookies)
session = requests.Session()
session.headers.update({"Authorization": f"Bearer {token}"})
r1 = session.get("https://api.example.com/users")
r2 = session.get("https://api.example.com/servers")
```

---

### 15. How do you implement retry logic for API calls?

**Answer:**

```python
import time
import requests
from functools import wraps

# Method 1: Manual retry
def request_with_retry(url, max_retries=3, backoff=2):
    for attempt in range(1, max_retries + 1):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except (requests.ConnectionError, requests.Timeout) as e:
            if attempt == max_retries:
                raise
            wait = backoff ** attempt
            print(f"Attempt {attempt} failed, retrying in {wait}s: {e}")
            time.sleep(wait)

# Method 2: Decorator
def retry(max_attempts=3, delay=2, exceptions=(Exception,)):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    if attempt == max_attempts:
                        raise
                    time.sleep(delay * attempt)
        return wrapper
    return decorator

@retry(max_attempts=3, delay=5, exceptions=(requests.RequestException,))
def fetch_data(url):
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()

# Method 3: urllib3 Retry (built into requests)
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retries = Retry(total=3, backoff_factor=1,
                status_forcelist=[500, 502, 503, 504])
session.mount("https://", HTTPAdapter(max_retries=retries))
response = session.get("https://api.example.com/data")
```

---

### 16. How do you send alerts to Slack or Teams from Python?

**Answer:**

```python
import requests
import json

# --- Slack (Incoming Webhook) ---
def send_slack_alert(webhook_url, message, severity="info"):
    colors = {"info": "#36a64f", "warning": "#ffcc00", "critical": "#ff0000"}
    payload = {
        "attachments": [{
            "color": colors.get(severity, "#36a64f"),
            "title": f"Alert: {severity.upper()}",
            "text": message,
            "footer": "DevOps Bot",
        }]
    }
    response = requests.post(webhook_url, json=payload, timeout=10)
    response.raise_for_status()

send_slack_alert(
    os.environ["SLACK_WEBHOOK"],
    "Disk usage on web-01 is at 92%",
    severity="critical"
)

# --- Microsoft Teams (Incoming Webhook) ---
def send_teams_alert(webhook_url, title, message):
    payload = {
        "@type": "MessageCard",
        "summary": title,
        "sections": [{
            "activityTitle": title,
            "text": message,
        }]
    }
    requests.post(webhook_url, json=payload, timeout=10)
```

---

## Cloud & Infrastructure Scripting

### 17. How do you use Boto3 (AWS SDK) for DevOps tasks?

**Answer:**

```python
import boto3

# --- EC2 ---
ec2 = boto3.client("ec2", region_name="us-east-1")

# List instances
response = ec2.describe_instances(
    Filters=[{"Name": "tag:Environment", "Values": ["production"]}]
)
for res in response["Reservations"]:
    for inst in res["Instances"]:
        name = next((t["Value"] for t in inst.get("Tags", [])
                      if t["Key"] == "Name"), "N/A")
        print(f"{inst['InstanceId']}  {name:30}  {inst['State']['Name']}")

# Stop instances by tag
ec2.stop_instances(InstanceIds=["i-0123456789abcdef0"])

# --- S3 ---
s3 = boto3.client("s3")

# Upload file
s3.upload_file("backup.tar.gz", "my-bucket", "backups/backup.tar.gz")

# List objects
response = s3.list_objects_v2(Bucket="my-bucket", Prefix="backups/")
for obj in response.get("Contents", []):
    print(f"  {obj['Key']}  {obj['Size'] / 1024 / 1024:.1f}MB")

# --- Secrets Manager ---
secrets = boto3.client("secretsmanager")
secret = secrets.get_secret_value(SecretId="prod/db-password")
db_password = json.loads(secret["SecretString"])["password"]
```

---

### 18. How do you interact with Kubernetes from Python?

**Answer:**

```python
from kubernetes import client, config  # pip install kubernetes

# Load kubeconfig
config.load_kube_config()       # From ~/.kube/config
# config.load_incluster_config()  # Inside a pod

v1 = client.CoreV1Api()
apps_v1 = client.AppsV1Api()

# List pods in a namespace
pods = v1.list_namespaced_pod(namespace="production")
for pod in pods.items:
    print(f"{pod.metadata.name:40} {pod.status.phase}")

# Get pod logs
logs = v1.read_namespaced_pod_log(
    name="myapp-7d4b8c6f5-abc12",
    namespace="production",
    tail_lines=50
)
print(logs)

# Scale deployment
body = {"spec": {"replicas": 5}}
apps_v1.patch_namespaced_deployment_scale(
    name="myapp", namespace="production", body=body
)

# List unhealthy pods
for pod in pods.items:
    for cs in pod.status.container_statuses or []:
        if cs.restart_count > 3:
            print(f"WARNING: {pod.metadata.name} restarted {cs.restart_count} times")
```

---

### 19. How do you work with Docker SDK in Python?

**Answer:**

```python
import docker  # pip install docker

client = docker.from_env()

# List running containers
for container in client.containers.list():
    print(f"{container.short_id}  {container.name:30}  {container.status}")

# Run a container
container = client.containers.run(
    "python:3.11-slim",
    command="python -c 'print(\"Hello from Docker\")'",
    detach=True,
    remove=True,
)
print(container.logs().decode())

# Build image
image, logs = client.images.build(
    path=".",
    tag="myapp:latest",
    rm=True,
)
for chunk in logs:
    if "stream" in chunk:
        print(chunk["stream"], end="")

# Stop all containers with label
for container in client.containers.list(filters={"label": "env=staging"}):
    print(f"Stopping {container.name}")
    container.stop(timeout=30)

# Prune unused resources
client.containers.prune()
client.images.prune(filters={"dangling": True})
```

---

## Real-World DevOps Scripts

### 20. Write a Python script to check health of multiple endpoints.

**Answer:**

```python
#!/usr/bin/env python3
"""Health check script for multiple services."""
import requests
import sys
from concurrent.futures import ThreadPoolExecutor, as_completed

ENDPOINTS = [
    {"name": "API",     "url": "http://api:8080/health"},
    {"name": "Web",     "url": "http://web:3000/health"},
    {"name": "Auth",    "url": "http://auth:9090/health"},
]
TIMEOUT = 5

def check(endpoint):
    try:
        r = requests.get(endpoint["url"], timeout=TIMEOUT)
        status = "OK" if r.status_code == 200 else f"FAIL ({r.status_code})"
    except requests.RequestException as e:
        status = f"FAIL ({e})"
    return endpoint["name"], status

def main():
    failed = []
    with ThreadPoolExecutor(max_workers=5) as pool:
        futures = {pool.submit(check, ep): ep for ep in ENDPOINTS}
        for future in as_completed(futures):
            name, status = future.result()
            icon = "OK" if status == "OK" else "FAIL"
            print(f"[{icon:4}] {name}: {status}")
            if status != "OK":
                failed.append(name)

    if failed:
        print(f"\nFailed services: {', '.join(failed)}")
        sys.exit(1)

if __name__ == "__main__":
    main()
```

---

### 21. Write a Python script for automated backup to S3.

**Answer:**

```python
#!/usr/bin/env python3
"""Backup local directory to S3 with timestamped archive."""
import boto3
import tarfile
import os
from datetime import datetime
from pathlib import Path

BACKUP_DIR = "/opt/myapp/data"
BUCKET = "my-backups"
PREFIX = "myapp"
RETENTION_DAYS = 30

def create_archive(source_dir):
    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    archive_name = f"/tmp/{PREFIX}_{timestamp}.tar.gz"
    with tarfile.open(archive_name, "w:gz") as tar:
        tar.add(source_dir, arcname=os.path.basename(source_dir))
    size_mb = Path(archive_name).stat().st_size / (1024 * 1024)
    print(f"Created archive: {archive_name} ({size_mb:.1f}MB)")
    return archive_name

def upload_to_s3(file_path):
    s3 = boto3.client("s3")
    key = f"backups/{os.path.basename(file_path)}"
    s3.upload_file(file_path, BUCKET, key)
    print(f"Uploaded to s3://{BUCKET}/{key}")

def cleanup_old_backups():
    s3 = boto3.client("s3")
    response = s3.list_objects_v2(Bucket=BUCKET, Prefix="backups/")
    cutoff = datetime.now().timestamp() - (RETENTION_DAYS * 86400)
    for obj in response.get("Contents", []):
        if obj["LastModified"].timestamp() < cutoff:
            s3.delete_object(Bucket=BUCKET, Key=obj["Key"])
            print(f"Deleted old backup: {obj['Key']}")

def main():
    archive = create_archive(BACKUP_DIR)
    try:
        upload_to_s3(archive)
        cleanup_old_backups()
    finally:
        os.remove(archive)
        print("Local archive cleaned up")

if __name__ == "__main__":
    main()
```

---

### 22. Write a Python script to parse and analyze Kubernetes events.

**Answer:**

```python
#!/usr/bin/env python3
"""Analyze Kubernetes events and flag issues."""
from kubernetes import client, config
from collections import Counter
from datetime import datetime, timezone

config.load_kube_config()
v1 = client.CoreV1Api()

events = v1.list_event_for_all_namespaces()

warning_events = [e for e in events.items if e.type == "Warning"]
reasons = Counter(e.reason for e in warning_events)

print(f"Total events: {len(events.items)}")
print(f"Warning events: {len(warning_events)}\n")

print("--- Top Warning Reasons ---")
for reason, count in reasons.most_common(10):
    print(f"  {reason:30} {count}")

print("\n--- Recent Warnings (last 1 hour) ---")
now = datetime.now(timezone.utc)
for event in warning_events:
    if event.last_timestamp:
        age = (now - event.last_timestamp).total_seconds()
        if age < 3600:
            print(f"  [{event.involved_object.namespace}/{event.involved_object.name}]")
            print(f"    Reason: {event.reason}")
            print(f"    Message: {event.message}")
            print(f"    Count: {event.count}")
            print()
```

---

### 23. Write a Python script that monitors a log file in real-time.

**Answer:**

```python
#!/usr/bin/env python3
"""Tail a log file and alert on error patterns."""
import time
import re
import sys

ALERT_PATTERNS = [
    (re.compile(r"ERROR|FATAL|CRITICAL", re.IGNORECASE), "critical"),
    (re.compile(r"OOM|OutOfMemory", re.IGNORECASE), "critical"),
    (re.compile(r"connection refused|timeout", re.IGNORECASE), "warning"),
]

def tail_file(filepath):
    """Yield new lines as they are appended to a file."""
    with open(filepath) as f:
        f.seek(0, 2)  # Go to end of file
        while True:
            line = f.readline()
            if line:
                yield line.strip()
            else:
                time.sleep(0.5)

def main():
    log_file = sys.argv[1] if len(sys.argv) > 1 else "/var/log/app.log"
    print(f"Monitoring: {log_file}")

    for line in tail_file(log_file):
        for pattern, severity in ALERT_PATTERNS:
            if pattern.search(line):
                print(f"[{severity.upper():8}] {line}")
                # send_slack_alert(line, severity)
                break

if __name__ == "__main__":
    main()
```

---

## Testing & Best Practices

### 24. How do you test Python scripts?

**Answer:**

```python
# test_deploy.py
import pytest
from unittest.mock import patch, MagicMock
from deploy import check_health, deploy_service

# Basic test
def test_check_health_success():
    with patch("deploy.requests.get") as mock_get:
        mock_get.return_value = MagicMock(status_code=200)
        assert check_health("http://localhost:8080") is True

def test_check_health_failure():
    with patch("deploy.requests.get") as mock_get:
        mock_get.side_effect = ConnectionError("refused")
        assert check_health("http://localhost:8080") is False

# Test with fixtures
@pytest.fixture
def sample_config():
    return {"host": "localhost", "port": 8080, "env": "test"}

def test_deploy_service(sample_config):
    with patch("deploy.subprocess.run") as mock_run:
        mock_run.return_value = MagicMock(returncode=0)
        result = deploy_service(sample_config)
        assert result is True
```

```bash
# Run tests
pytest test_deploy.py -v
pytest --cov=deploy test_deploy.py    # With coverage
```

---

### 25. What are Python scripting best practices for DevOps?

**Answer:**

```python
#!/usr/bin/env python3
"""
Script description: What it does.
Usage: python script.py --env prod --service myapp
"""
import argparse
import logging
import sys
import os

# 1. Structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)

# 2. Main function pattern
def main():
    args = parse_args()
    try:
        result = do_work(args)
        logger.info(f"Completed: {result}")
    except Exception as e:
        logger.error(f"Failed: {e}")
        sys.exit(1)

# 3. Entry point guard
if __name__ == "__main__":
    main()
```

**Key practices:**
- **Use `logging`** — not `print()` for production scripts
- **Use `argparse`** — for CLI arguments, not `sys.argv` directly
- **Use `pathlib`** — instead of `os.path` for file operations
- **Use `subprocess.run()`** — not `os.system()`
- **Use virtual environments** — `python -m venv .venv`
- **Pin dependencies** — `pip freeze > requirements.txt`
- **Add type hints** — `def deploy(service: str, tag: str) -> bool:`
- **Use `if __name__ == "__main__":`** — always
- **Use `shellcheck` equivalent** — `pylint`, `flake8`, `ruff`
- **Handle secrets via env vars** — never hardcode
- **Add `requirements.txt`** — for every script that imports external packages
