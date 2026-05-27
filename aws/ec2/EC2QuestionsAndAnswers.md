# AWS EC2 Interview Questions and Answers

Comprehensive EC2 questions for DevOps engineers — covering instance types, AMIs, storage, networking, Auto Scaling, pricing, and real-world scenario-based questions.

---

## Table of Contents

- [EC2 Fundamentals](#ec2-fundamentals)
- [Instance Types & AMIs](#instance-types--amis)
- [Storage (EBS, Instance Store, EFS)](#storage-ebs-instance-store-efs)
- [Networking & Security](#networking--security)
- [Auto Scaling & Load Balancing](#auto-scaling--load-balancing)
- [Pricing & Cost Optimization](#pricing--cost-optimization)
- [Scenario-Based Questions](#scenario-based-questions)

---

## EC2 Fundamentals

### 1. What is EC2 and what are its key features?

**Answer:** Amazon EC2 (Elastic Compute Cloud) provides resizable virtual servers (instances) in the cloud.

**Key features:**
- **On-demand scalability** — Launch/terminate instances in seconds
- **Multiple instance types** — Optimized for compute, memory, storage, GPU
- **Multiple pricing models** — On-Demand, Reserved, Spot, Savings Plans
- **AMIs** — Pre-configured machine images for quick launches
- **Security Groups** — Virtual firewalls per instance
- **Elastic IPs** — Static public IPs
- **Placement Groups** — Control instance placement for performance
- **User Data** — Bootstrap scripts on launch

---

### 2. Explain the EC2 instance lifecycle.

**Answer:**

```
launch → pending → running → stopping → stopped → terminated
                      ↓          ↑
                  shutting-down   |
                      ↓          |
                  terminated     |
                                 |
                   running ← starting (from stopped)
```

| State | Billing | EBS | Instance Store | Public IP |
|-------|---------|-----|---------------|-----------|
| **pending** | No | Attached | N/A | Not assigned yet |
| **running** | Yes | Attached | Available | Assigned (if enabled) |
| **stopping** | No (after full min) | Persists | Lost | Released |
| **stopped** | No (EBS charges apply) | Persists | Lost | Released |
| **terminated** | No | Deleted (default) | Lost | Released |

---

## Instance Types & AMIs

### 3. Explain EC2 instance type families and when to use each.

**Answer:**

| Family | Prefix | Optimized for | Use Case |
|--------|--------|-------------- |----------|
| **General Purpose** | `t3, t3a, m5, m6i, m7g` | Balanced CPU/memory | Web servers, small DBs, dev/test |
| **Compute Optimized** | `c5, c6i, c7g` | High CPU | Batch processing, ML inference, gaming |
| **Memory Optimized** | `r5, r6i, x2idn` | High memory | In-memory DBs (Redis), big data, SAP |
| **Storage Optimized** | `i3, i4i, d3` | High I/O | Data warehouses, Elasticsearch, HDFS |
| **Accelerated** | `p4, p5, g5, inf2` | GPU/FPGA | ML training, video encoding, HPC |

**Instance naming:** `m5.xlarge`
- `m` = family (general purpose)
- `5` = generation
- `xlarge` = size (vCPUs + memory)

**Graviton (ARM):** `m7g, c7g, r7g` — 20-40% better price/performance vs x86. Use for containerized workloads, web servers, microservices.

---

### 4. What is an AMI and how do you create a custom AMI?

**Answer:** An **Amazon Machine Image (AMI)** is a template containing OS, application server, and applications used to launch instances.

```bash
# Create AMI from running instance
aws ec2 create-image \
  --instance-id i-0abc123 \
  --name "myapp-v1.5-$(date +%Y%m%d)" \
  --description "MyApp v1.5 with Java 17" \
  --no-reboot   # Don't reboot instance during image creation

# Share AMI cross-account
aws ec2 modify-image-attribute \
  --image-id ami-xxx \
  --launch-permission "Add=[{UserId=123456789012}]"

# Copy AMI to another region
aws ec2 copy-image \
  --source-image-id ami-xxx \
  --source-region us-east-1 \
  --region eu-west-1 \
  --name "myapp-v1.5-eu"
```

**AMI best practices:**
- **Golden AMI** — Bake OS hardening, agents (Datadog, SSM), base packages
- **Packer** — Automate AMI creation in CI/CD
- **Deregister old AMIs** — Clean up unused AMIs + snapshots to save cost
- **Encrypt AMIs** — Enable EBS encryption by default

---

### 5. What is the difference between `t3` burstable instances and `m5` fixed-performance instances?

**Answer:**

| Feature | t3 (burstable) | m5 (fixed) |
|---------|---------------|------------|
| **CPU model** | Baseline + burst credits | Full CPU always available |
| **Baseline** | 20-40% CPU (varies by size) | 100% CPU |
| **Credits** | Earn when idle, spend when busy | N/A |
| **Cost** | 30-50% cheaper | Higher |
| **Unlimited mode** | Pay extra for sustained burst | N/A |
| **Use case** | Low avg CPU (web, dev, bastion) | Consistent CPU (app servers, CI runners) |

**Scenario:** `t3.medium` has 20% baseline. If your app averages 15% CPU with occasional spikes to 80%, `t3` is perfect. If it consistently runs at 60% CPU, use `m5` — burstable credits will deplete.

```bash
# Check CPU credit balance
aws cloudwatch get-metric-data \
  --metric-data-queries '[{"Id":"credits","MetricStat":{"Metric":{"Namespace":"AWS/EC2","MetricName":"CPUCreditBalance","Dimensions":[{"Name":"InstanceId","Value":"i-xxx"}]},"Period":300,"Stat":"Average"}}]' \
  --start-time 2024-01-15T00:00:00Z --end-time 2024-01-15T23:59:59Z
```

---

## Storage (EBS, Instance Store, EFS)

### 6. What are the EBS volume types and when do you use each?

**Answer:**

| Type | IOPS | Throughput | Use Case |
|------|------|-----------|----------|
| **gp3** (General Purpose SSD) | 3,000-16,000 | 125-1,000 MB/s | Default for most workloads |
| **gp2** (General Purpose SSD) | 100-16,000 (burst) | 128-250 MB/s | Legacy — migrate to gp3 |
| **io2 Block Express** | Up to 256,000 | 4,000 MB/s | Critical databases, SAP |
| **io1** | Up to 64,000 | 1,000 MB/s | High-performance databases |
| **st1** (Throughput HDD) | 500 | 500 MB/s | Big data, data warehouses |
| **sc1** (Cold HDD) | 250 | 250 MB/s | Infrequent access, archival |

**gp3 vs gp2:**
- gp3 is **20% cheaper** than gp2
- gp3 IOPS and throughput are **independently configurable**
- gp2 IOPS scale with volume size (3 IOPS/GB)

```bash
# Migrate gp2 → gp3
aws ec2 modify-volume --volume-id vol-xxx --volume-type gp3 --iops 4000 --throughput 250
```

---

### 7. What is the difference between EBS, Instance Store, and EFS?

**Answer:**

| Feature | EBS | Instance Store | EFS |
|---------|-----|---------------|-----|
| **Persistence** | Persists independently | Lost on stop/terminate | Persists |
| **Scope** | Single AZ (one instance) | Tied to instance | Multi-AZ (shared) |
| **Performance** | Configurable IOPS | Very high (NVMe local) | Moderate |
| **Snapshot** | Yes (S3 backed) | No | AWS Backup |
| **Size** | 1 GiB - 64 TiB | Fixed per instance type | Elastic (auto-grows) |
| **Use case** | Boot volumes, databases | Temp data, caches, buffers | Shared storage, CMS, ML |

---

## Networking & Security

### 8. What are Elastic IPs and when should you use them?

**Answer:** An **Elastic IP (EIP)** is a static IPv4 address that you can associate/disassociate with instances.

**When to use:**
- Instance must have a **fixed public IP** (e.g., whitelisted by external service)
- **Failover** — quickly remap EIP to a standby instance

**When NOT to use:**
- Behind a load balancer (use ALB/NLB DNS)
- Dynamic workloads (Auto Scaling)

**Charges:** Free when associated with a running instance. **$0.005/hr** when unattached or on a stopped instance.

---

### 9. What is an EC2 Placement Group?

**Answer:**

| Strategy | Behavior | Use Case |
|----------|---------|----------|
| **Cluster** | All instances in same rack/AZ | Low latency, HPC, big data |
| **Spread** | Each instance on different hardware (max 7/AZ) | Critical instances needing max isolation |
| **Partition** | Groups of instances on different racks | Large distributed workloads (Hadoop, Kafka) |

```bash
# Create cluster placement group
aws ec2 create-placement-group --group-name hpc-cluster --strategy cluster

# Launch into placement group
aws ec2 run-instances --placement "GroupName=hpc-cluster" ...
```

---

### 10. What is EC2 Instance Connect and Session Manager? How do they replace SSH?

**Answer:**

| Method | How it works | Key advantage |
|--------|-------------|---------------|
| **EC2 Instance Connect** | Pushes temporary SSH key via IAM | No permanent SSH keys to manage |
| **SSM Session Manager** | Agent-based shell via AWS API | No open SSH port (port 22), no bastion needed |
| **Traditional SSH** | Key pair + port 22 open | Simplest but least secure |

```bash
# Session Manager — connect without SSH
aws ssm start-session --target i-0abc123

# No need for:
# - Port 22 open in security group
# - Bastion host
# - SSH key management
# - Public IP on instance

# Requires: SSM Agent installed + IAM permissions
```

**Best practice:** Use **Session Manager** for production. All sessions are logged to CloudWatch/S3 for audit.

---

## Auto Scaling & Load Balancing

### 11. How does EC2 Auto Scaling work?

**Answer:**

```
                    CloudWatch Alarm
                    (CPU > 70%)
                         │
                         ▼
                 ┌──────────────┐
                 │ Auto Scaling │
                 │    Group     │
                 └──────┬───────┘
                        │
           ┌────────────┼────────────┐
           ▼            ▼            ▼
        Instance     Instance     Instance
        (AZ-a)       (AZ-b)       (AZ-a)
```

**Components:**
- **Launch Template** — Instance configuration (AMI, type, SG, user data)
- **ASG** — Min/max/desired count, AZ distribution, health checks
- **Scaling Policy** — When and how to scale

**Scaling policies:**

| Type | How it works | Example |
|------|-------------|---------|
| **Target Tracking** | Maintain a metric at target value | Keep CPU at 50% |
| **Step Scaling** | Scale in steps based on alarm thresholds | CPU >70% → +2, >90% → +4 |
| **Scheduled** | Scale at specific times | Scale up at 9 AM, down at 6 PM |
| **Predictive** | ML-based forecasting | Anticipate traffic patterns |

```bash
# Create target tracking policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target-50 \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 50.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

---

### 12. What is the difference between ALB, NLB, and CLB?

**Answer:**

| Feature | ALB | NLB | CLB (legacy) |
|---------|-----|-----|-------------|
| **Layer** | 7 (HTTP/HTTPS) | 4 (TCP/UDP/TLS) | 4 + 7 |
| **Routing** | Path, host, header, query string | Port-based | Basic |
| **Performance** | Good | Ultra-low latency, millions of RPS | Moderate |
| **Static IP** | No (use Global Accelerator) | Yes (Elastic IP per AZ) | No |
| **WebSocket** | Yes | Yes | No |
| **gRPC** | Yes | Yes (TCP passthrough) | No |
| **SSL termination** | Yes | Yes (TLS) | Yes |
| **Use case** | Web apps, microservices, APIs | TCP services, gaming, IoT, gRPC | Legacy only |

**Choose ALB:** HTTP/HTTPS routing, path-based routing, multiple target groups.
**Choose NLB:** Low latency, static IPs, TCP/UDP, extreme performance, Kubernetes (AWS LB Controller).

---

## Pricing & Cost Optimization

### 13. Explain EC2 pricing models and when to use each.

**Answer:**

| Model | Discount | Commitment | Best for |
|-------|---------|-----------|----------|
| **On-Demand** | 0% | None | Short-term, unpredictable, dev/test |
| **Reserved (RI)** | Up to 72% | 1 or 3 years | Steady-state production |
| **Savings Plans** | Up to 72% | 1 or 3 years ($/hr) | Flexible across instance types |
| **Spot** | Up to 90% | None (can be interrupted) | Batch jobs, CI/CD, fault-tolerant |
| **Dedicated Host** | Varies | Optional | Compliance, BYOL licensing |

**Savings Plans vs Reserved Instances:**

| Feature | Savings Plans | Reserved Instances |
|---------|--------------|-------------------|
| **Flexibility** | Any instance type/region/OS | Specific instance type + region |
| **Discount** | Similar | Similar |
| **Ease** | Commit $/hr | Commit instance count |
| **Recommendation** | Preferred for most cases | When you know exact sizing |

---

### 14. How do you use Spot Instances effectively?

**Answer:**

```bash
# Request spot instances
aws ec2 run-instances \
  --instance-type c5.xlarge \
  --instance-market-options '{"MarketType":"spot","SpotOptions":{"SpotInstanceType":"persistent","InstanceInterruptionBehavior":"stop"}}' \
  ...
```

**Best practices for Spot:**
- **Diversify instance types** — Use multiple types/AZs (capacity pools)
- **Handle interruptions** — 2-minute warning via metadata or EventBridge
- **Use Spot Fleet / ASG mixed instances** — Automatically falls back to On-Demand
- **Checkpointing** — Save work periodically for batch jobs

```bash
# Check for spot interruption (from instance)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/spot/termination-time
# Returns 200 + timestamp if termination is imminent, 404 if safe
```

**ASG with mixed instances:**
```json
{
  "MixedInstancesPolicy": {
    "LaunchTemplate": { "LaunchTemplateSpecification": { "LaunchTemplateId": "lt-xxx" } },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 30,
      "SpotAllocationStrategy": "capacity-optimized"
    },
    "Overrides": [
      { "InstanceType": "c5.xlarge" },
      { "InstanceType": "c5a.xlarge" },
      { "InstanceType": "c5d.xlarge" },
      { "InstanceType": "m5.xlarge" }
    ]
  }
}
```

---

## Scenario-Based Questions

### 15. Scenario: Your EC2 instance is running but you cannot SSH into it. How do you troubleshoot?

**Answer:**

| Check | What to verify |
|-------|---------------|
| **Security Group** | Inbound rule for port 22 from your IP? |
| **NACL** | Allows inbound 22 AND outbound ephemeral ports (1024-65535)? |
| **Route table** | Instance in public subnet with IGW route? |
| **Public IP / EIP** | Instance has a public IP? |
| **Key pair** | Using correct private key? Permissions `chmod 400`? |
| **OS firewall** | `iptables`/`ufw` blocking on the instance? |
| **Instance state** | Status checks passing (system + instance)? |
| **Disk full** | EBS volume full → OS can't start sshd |

**If you can't SSH at all:**
```bash
# Use Session Manager (no SSH needed)
aws ssm start-session --target i-0abc123

# Use EC2 Serial Console (for boot issues)
aws ec2-instance-connect send-serial-console-ssh-public-key \
  --instance-id i-0abc123 --serial-port 0 --ssh-public-key file://key.pub
```

---

### 16. Scenario: An EC2 instance is showing high CPU but the application team says traffic is normal. What do you investigate?

**Answer:**

```bash
# 1. Identify the process consuming CPU
top -c    # or htop
ps aux --sort=-%cpu | head -20

# 2. Check if it's a known process
# Common culprits: runaway log processing, zombie processes, crypto miner

# 3. Check system-level causes
dmesg | tail -50                   # Kernel messages
cat /var/log/syslog | tail -100    # System logs
iostat -x 5                        # Disk I/O bottleneck causing CPU wait
vmstat 5                           # Check for swapping (high si/so)

# 4. Check if instance type is appropriate
# t3 burstable: CPU credit exhausted → throttled → appears as 100% CPU
aws cloudwatch get-metric-statistics --namespace AWS/EC2 \
  --metric-name CPUCreditBalance --dimensions Name=InstanceId,Value=i-xxx \
  --start-time ... --end-time ... --period 300 --statistics Average

# 5. Check for EBS throughput bottleneck
# High iowait % = disk-bound, not CPU-bound
iostat -xz 5
```

---

### 17. Scenario: Your Auto Scaling Group is not scaling out despite high CPU. What's wrong?

**Answer:**

| Possible cause | How to check |
|---------------|-------------|
| **Max capacity reached** | ASG max count already met |
| **Cooldown period** | Recent scale event, waiting for cooldown |
| **No scaling policy** | ASG exists but no policy attached |
| **CloudWatch alarm** | Alarm not in ALARM state (check threshold, evaluation period) |
| **Launch failures** | Insufficient capacity in AZ, SG/subnet misconfigured |
| **Service quota** | Reached EC2 instance limit for the account |
| **Desired ≠ actual** | ASG desired count is high but instances failing health checks and being replaced |

```bash
# Check ASG activities
aws autoscaling describe-scaling-activities --auto-scaling-group-name my-asg --max-items 20

# Check if instances are launching but failing
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names my-asg \
  --query 'AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Running:Instances|length(@)}'
```

---

### 18. Scenario: You need to migrate an EC2 instance to a different AZ without downtime. How?

**Answer:**

**EC2 cannot be moved between AZs directly.** Approach:

1. **Create AMI** from current instance
2. **Launch new instance** from AMI in target AZ
3. **Attach/recreate** EBS volumes (copy snapshots if needed)
4. **Update DNS / EIP** — Remap Elastic IP or update Route 53
5. **Verify** application is healthy on new instance
6. **Terminate** old instance

**For zero downtime:**
- Place both behind an ALB/NLB
- Add new instance to target group
- Wait for health checks to pass
- Remove old instance from target group
- Terminate old instance

---

### 19. Scenario: An EBS volume is running out of space on a production instance. How do you expand it without downtime?

**Answer:** EBS volumes can be **expanded online** (no restart required for most cases).

```bash
# 1. Modify volume size
aws ec2 modify-volume --volume-id vol-xxx --size 200  # Increase to 200 GB

# 2. Wait for optimization to complete
aws ec2 describe-volumes-modifications --volume-ids vol-xxx

# 3. Extend the filesystem (on the instance)
# For ext4:
sudo growpart /dev/xvda 1              # Expand partition
sudo resize2fs /dev/xvda1              # Expand filesystem

# For xfs:
sudo growpart /dev/xvda 1
sudo xfs_growfs /                      # Expand filesystem

# Verify
df -h
```

**Note:** You can only **increase** size, never decrease. You can also change volume type (gp2 → gp3) online.

---

### 20. Scenario: Your application needs to be highly available across AZs. Design the architecture.

**Answer:**

```
               ┌──────────────┐
               │   Route 53   │
               │  (DNS CNAME)  │
               └──────┬───────┘
                      │
               ┌──────▼───────┐
               │     ALB      │ (Multi-AZ)
               └──┬────────┬──┘
                  │        │
         ┌────────▼─┐  ┌──▼────────┐
         │  AZ-a    │  │   AZ-b    │
         │ ┌──────┐ │  │ ┌──────┐  │
         │ │ ASG  │ │  │ │ ASG  │  │
         │ │ EC2  │ │  │ │ EC2  │  │
         │ │ EC2  │ │  │ │ EC2  │  │
         │ └──────┘ │  │ └──────┘  │
         │          │  │           │
         │ ┌──────┐ │  │ ┌──────┐  │
         │ │RDS   │ │  │ │RDS   │  │
         │ │Primary│ │  │ │Standby│ │
         │ └──────┘ │  │ └──────┘  │
         └──────────┘  └───────────┘
```

**Key components:**
- **ALB** — Distributes traffic across AZs
- **ASG** — Min 2 instances, spread across AZs
- **RDS Multi-AZ** — Automatic failover to standby
- **ElastiCache Multi-AZ** — Automatic replica failover
- **S3** — Static assets (inherently multi-AZ)
- **Health checks** — ALB health checks + Route 53 health checks

---

### 21. Scenario: How do you implement a zero-downtime deployment on EC2?

**Answer:**

**Option 1: Rolling deployment (ASG)**
```bash
# Update launch template with new AMI
aws ec2 create-launch-template-version \
  --launch-template-id lt-xxx \
  --source-version 1 \
  --launch-template-data '{"ImageId":"ami-new"}'

# Start instance refresh (rolling update)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name my-asg \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300
  }'
```

**Option 2: Blue/green with ALB**
1. Create new ASG (green) with new AMI
2. Register green target group with ALB
3. Run health checks / smoke tests
4. Shift ALB listener from blue → green target group
5. Drain and terminate blue ASG

**Option 3: CodeDeploy**
- Automated blue/green or in-place deployments
- Integrates with ASG, ALB, lifecycle hooks

---

### 22. Scenario: Your EC2 instance has a status check failure. What does it mean and how do you fix it?

**Answer:**

| Check | What it means | How to fix |
|-------|--------------|-----------|
| **System status check** | AWS infrastructure problem (hardware, network, power) | Stop and start instance (migrates to new host) |
| **Instance status check** | OS-level issue (kernel panic, disk full, misconfigured network) | Reboot, or fix OS config via Serial Console / user data |

```bash
# Check status
aws ec2 describe-instance-status --instance-ids i-xxx

# Fix system status failure — stop/start (NOT reboot)
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 start-instances --instance-ids i-xxx
# Instance is now on different underlying hardware

# If instance won't boot — use rescue approach:
# 1. Stop instance
# 2. Detach root EBS volume
# 3. Attach to a rescue instance
# 4. Mount, fix files (fstab, network, etc.)
# 5. Reattach to original instance
# 6. Start
```

---

### 23. How do you implement EC2 instance hardening for production?

**Answer:**

**Hardening checklist:**
- **Minimal AMI** — Start with minimal OS, install only what's needed
- **Patch management** — SSM Patch Manager for automated patching
- **Disable password auth** — SSH key only (`PasswordAuthentication no`)
- **No root SSH** — `PermitRootLogin no`
- **Remove unnecessary packages** — Reduce attack surface
- **CIS Benchmarks** — Use CIS hardened AMIs or automate with Ansible
- **IMDSv2** — Require token-based metadata access
- **EBS encryption** — Enable default encryption
- **Security group** — Minimal open ports, no 0.0.0.0/0 for SSH
- **IAM instance profile** — Least privilege, no hardcoded credentials
- **Inspector** — Scan for vulnerabilities

```bash
# Require IMDSv2 (prevent SSRF attacks)
aws ec2 modify-instance-metadata-options \
  --instance-id i-xxx \
  --http-tokens required \
  --http-endpoint enabled

# Enable EBS default encryption
aws ec2 enable-ebs-encryption-by-default
```

---

### 24. Scenario: You have 50 EC2 instances and need to run a command on all of them. How?

**Answer:**

```bash
# SSM Run Command — no SSH needed
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=production" \
  --parameters commands=["sudo yum update -y","sudo systemctl restart myapp"] \
  --timeout-seconds 600 \
  --max-concurrency "10" \
  --max-errors "5"

# Check results
aws ssm list-command-invocations \
  --command-id "cmd-xxx" \
  --details
```

**Alternatives:**
- **Ansible** — Agentless (SSH-based), good for complex playbooks
- **SSM Run Command** — AWS-native, IAM-based auth, no SSH port needed
- **SSM State Manager** — Enforces desired state on schedule
- **User Data** — Only at launch time

---

### 25. Scenario: How would you reduce your EC2 bill by 40% without impacting performance?

**Answer:**

| Strategy | Savings | Effort |
|----------|---------|--------|
| **Right-size instances** | 20-40% | Low — use AWS Compute Optimizer |
| **Savings Plans / RIs** | Up to 72% | Low — commit to steady usage |
| **Spot for non-critical** | Up to 90% | Medium — CI/CD, batch, dev |
| **gp2 → gp3 migration** | 20% on EBS | Low — modify volume type |
| **Graviton instances** | 20-40% | Medium — test ARM compatibility |
| **Stop idle instances** | 100% of idle cost | Low — schedule dev instances |
| **Delete unattached EBS** | $0.10/GB/month | Low — find orphaned volumes |
| **Release unused EIPs** | $3.60/month each | Low |

```bash
# Find unattached EBS volumes
aws ec2 describe-volumes --filters Name=status,Values=available \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]' --output table

# Find stopped instances
aws ec2 describe-instances --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,Tags[?Key==`Name`].Value|[0]]' --output table
```
