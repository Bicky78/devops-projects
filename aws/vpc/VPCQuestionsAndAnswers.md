# AWS VPC Interview Questions and Answers

Comprehensive VPC questions for DevOps engineers — covering networking fundamentals, subnets, routing, security, peering, Transit Gateway, VPN, and real-world scenario-based questions.

---

## Table of Contents

- [VPC Fundamentals](#vpc-fundamentals)
- [Subnets, Route Tables & Gateways](#subnets-route-tables--gateways)
- [Security Groups & NACLs](#security-groups--nacls)
- [VPC Peering & Transit Gateway](#vpc-peering--transit-gateway)
- [VPN & Direct Connect](#vpn--direct-connect)
- [VPC Endpoints](#vpc-endpoints)
- [Scenario-Based Questions](#scenario-based-questions)

---

## VPC Fundamentals

### 1. What is a VPC and why is it important?

**Answer:** A **Virtual Private Cloud (VPC)** is a logically isolated virtual network within AWS where you launch resources. It gives you full control over IP addressing, subnets, route tables, and network gateways.

**Key properties:**
- **Region-scoped** — a VPC lives in one AWS region
- **CIDR block** — IPv4 range (e.g., `10.0.0.0/16` = 65,536 IPs)
- **Default VPC** — AWS creates one per region (172.31.0.0/16)
- **Custom VPC** — you define CIDR, subnets, routing, security

```
VPC (10.0.0.0/16)
├── Public Subnet  (10.0.1.0/24) — AZ-a — Internet-facing
├── Public Subnet  (10.0.2.0/24) — AZ-b
├── Private Subnet (10.0.10.0/24) — AZ-a — Apps, databases
├── Private Subnet (10.0.11.0/24) — AZ-b
└── Isolated Subnet(10.0.20.0/24) — AZ-a — No internet at all
```

---

### 2. How do you choose a CIDR block for your VPC?

**Answer:**

**Best practices:**
- Use **RFC 1918** private ranges: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`
- Plan for **future growth** — `/16` gives 65,536 IPs
- **Never overlap CIDRs** with other VPCs you'll peer with or on-prem networks
- AWS allows `/16` to `/28` (16 IPs minimum)
- AWS reserves **5 IPs per subnet** (first 4 + last 1)

| CIDR | Total IPs | Usable IPs per /24 subnet |
|------|----------|--------------------------|
| `/16` | 65,536 | 251 per /24 |
| `/20` | 4,096 | 251 per /24 |
| `/24` | 256 | 251 |
| `/28` | 16 | 11 |

**Secondary CIDR:** You can add secondary CIDR blocks to an existing VPC if you run out of IPs (up to 5 CIDRs).

---

## Subnets, Route Tables & Gateways

### 3. What is the difference between a public subnet and a private subnet?

**Answer:**

| Feature | Public Subnet | Private Subnet |
|---------|--------------|----------------|
| **Route to internet** | Route to Internet Gateway (IGW) | Route to NAT Gateway (or no internet) |
| **Public IP** | Instances can have public/Elastic IPs | No public IPs |
| **Accessible from internet** | Yes (via security groups) | No (only via bastion/VPN/LB) |
| **Use case** | Load balancers, bastion hosts, NAT GW | App servers, databases, workers |

```
Public Subnet Route Table:
  0.0.0.0/0 → igw-xxx    (internet traffic goes to IGW)
  10.0.0.0/16 → local    (VPC internal traffic)

Private Subnet Route Table:
  0.0.0.0/0 → nat-xxx    (internet traffic goes through NAT GW)
  10.0.0.0/16 → local
```

---

### 4. What is a NAT Gateway and why is it needed?

**Answer:** A **NAT Gateway** allows instances in **private subnets** to access the internet (e.g., download packages, call APIs) without being directly accessible from the internet.

**NAT Gateway vs NAT Instance:**

| Feature | NAT Gateway | NAT Instance |
|---------|------------|-------------|
| **Managed** | Yes (AWS) | No (you manage EC2) |
| **Bandwidth** | Up to 100 Gbps | Depends on instance type |
| **Availability** | HA within AZ | Single point of failure |
| **Cost** | $0.045/hr + data processing | Instance cost only |
| **Maintenance** | Zero | Patching, monitoring |

```bash
# Create NAT Gateway (Terraform)
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_a.id  # Must be in PUBLIC subnet
  tags = { Name = "nat-gw-az-a" }
}

# Route private subnet through NAT GW
resource "aws_route" "private_nat" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.main.id
}
```

**Cost optimization:** Use one NAT GW per AZ for HA, or one shared NAT GW for non-critical workloads to save cost.

---

### 5. What is an Internet Gateway (IGW) and how does it differ from NAT Gateway?

**Answer:**

| Feature | Internet Gateway (IGW) | NAT Gateway |
|---------|----------------------|-------------|
| **Direction** | Bidirectional (inbound + outbound) | Outbound only |
| **Attaches to** | VPC (one per VPC) | Subnet (public subnet) |
| **Public IP needed** | Yes (on the instance) | No (NAT translates) |
| **Use case** | Public-facing resources | Private resources needing outbound |
| **Cost** | Free | $0.045/hr + per-GB |

---

## Security Groups & NACLs

### 6. What is the difference between Security Groups and NACLs?

**Answer:**

| Feature | Security Group | NACL |
|---------|---------------|------|
| **Level** | Instance / ENI level | Subnet level |
| **State** | **Stateful** (return traffic auto-allowed) | **Stateless** (must explicitly allow return) |
| **Rules** | Allow rules only | Allow AND deny rules |
| **Evaluation** | All rules evaluated together | Rules evaluated in order (lowest number first) |
| **Default** | Deny all inbound, allow all outbound | Allow all inbound and outbound |
| **Scope** | Applied to specific instances | Applied to all instances in subnet |

```bash
# Security Group — allow web traffic
resource "aws_security_group" "web" {
  vpc_id = aws_vpc.main.id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Reference another SG
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# NACL — block specific IP range
resource "aws_network_acl_rule" "deny_bad_ip" {
  network_acl_id = aws_network_acl.main.id
  rule_number    = 100
  protocol       = "tcp"
  rule_action    = "deny"
  cidr_block     = "203.0.113.0/24"  # Malicious range
  from_port      = 0
  to_port        = 65535
}
```

**Best practice:** Use Security Groups as the primary defense (stateful, simpler). Use NACLs as an additional layer for subnet-level blocking (e.g., blocking known malicious IPs).

---

### 7. Can a Security Group reference another Security Group? When would you do this?

**Answer:** Yes. This is a **best practice** for service-to-service communication within a VPC.

```
ALB (SG: sg-alb) → App (SG: sg-app) → DB (SG: sg-db)

sg-app allows inbound 8080 from sg-alb (not CIDR)
sg-db  allows inbound 5432 from sg-app (not CIDR)
```

**Why:**
- No need to track IP addresses (instances scale up/down)
- More secure — only tagged resources can communicate
- Cleaner than CIDR-based rules
- Auto-adapts when ASG adds/removes instances

---

## VPC Peering & Transit Gateway

### 8. What is VPC Peering and what are its limitations?

**Answer:** VPC Peering is a **point-to-point** networking connection between two VPCs that allows traffic using private IPs.

**Key characteristics:**
- Works **cross-account** and **cross-region**
- Traffic stays on AWS backbone (not over internet)
- No single point of failure or bandwidth bottleneck
- **Non-transitive** — if A↔B and B↔C, A cannot reach C through B

**Limitations:**
- No transitive routing
- CIDR blocks **cannot overlap**
- Maximum ~125 peering connections per VPC
- Cannot use the same VPC as both requester and accepter
- No edge-to-edge routing (can't route through peered VPC's IGW/VPN)

---

### 9. What is Transit Gateway and when do you use it instead of VPC Peering?

**Answer:** **Transit Gateway (TGW)** is a regional hub that connects multiple VPCs, on-premises networks, and VPNs through a single gateway.

| Feature | VPC Peering | Transit Gateway |
|---------|------------|----------------|
| **Topology** | Point-to-point (mesh) | Hub-and-spoke |
| **Transitive** | No | Yes |
| **Scale** | ~125 peers per VPC | 5,000 attachments |
| **Cross-region** | Yes (but each pair) | TGW Peering between regions |
| **On-prem** | Separate VPN per VPC | Single VPN to TGW |
| **Cost** | Free (data transfer only) | $0.05/hr per attachment + data |
| **Complexity** | Grows as N² | Centralized routing |

```
Without TGW (mesh — 6 peering connections for 4 VPCs):
VPC-A ↔ VPC-B
VPC-A ↔ VPC-C
VPC-A ↔ VPC-D
VPC-B ↔ VPC-C
VPC-B ↔ VPC-D
VPC-C ↔ VPC-D

With TGW (hub — 4 attachments):
VPC-A ↔ TGW ↔ VPC-B
             ↔ VPC-C
             ↔ VPC-D
             ↔ On-prem (VPN)
```

**Use TGW when:** >3 VPCs, need transitive routing, on-prem connectivity, centralized network management.

---

## VPN & Direct Connect

### 10. What is the difference between AWS Site-to-Site VPN and Direct Connect?

**Answer:**

| Feature | Site-to-Site VPN | Direct Connect |
|---------|-----------------|---------------|
| **Connection** | Encrypted over public internet | Dedicated private fiber |
| **Bandwidth** | Up to 1.25 Gbps per tunnel | 1 Gbps, 10 Gbps, 100 Gbps |
| **Latency** | Variable (internet) | Consistent, low |
| **Setup time** | Minutes | Weeks to months |
| **Cost** | Low (~$0.05/hr) | High (port + data transfer) |
| **Redundancy** | 2 tunnels per VPN | Need 2 connections for HA |
| **Encryption** | IPSec (built-in) | Not encrypted (add VPN overlay) |
| **Use case** | Quick setup, backup, dev/test | Production, large data transfer, consistent latency |

**Common pattern:** Direct Connect as primary + VPN as backup failover.

---

## VPC Endpoints

### 11. What are VPC Endpoints and why do you need them?

**Answer:** VPC Endpoints allow private connectivity to AWS services **without going through the internet, NAT Gateway, or VPN**.

| Type | How it works | Supported services | Cost |
|------|-------------|-------------------|------|
| **Gateway Endpoint** | Route table entry | S3, DynamoDB only | Free |
| **Interface Endpoint (PrivateLink)** | ENI in your subnet | 100+ services (ECR, STS, CloudWatch, etc.) | $0.01/hr + per-GB |

```bash
# Gateway endpoint for S3 (free — always use this)
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
  route_table_ids = [aws_route_table.private.id]
}

# Interface endpoint for ECR (needed for private EKS pulling images)
resource "aws_vpc_endpoint" "ecr_api" {
  vpc_id             = aws_vpc.main.id
  service_name       = "com.amazonaws.us-east-1.ecr.api"
  vpc_endpoint_type  = "Interface"
  subnet_ids         = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  security_group_ids = [aws_security_group.vpce.id]
  private_dns_enabled = true
}
```

**Why use endpoints:**
- **Security** — Traffic stays within AWS network
- **Cost savings** — Avoid NAT Gateway data processing charges
- **Performance** — Lower latency, higher throughput
- **Required for private EKS** — Nodes in private subnets need endpoints for ECR, STS, EC2, S3

---

## Scenario-Based Questions

### 12. Scenario: Your EC2 instance in a private subnet cannot reach the internet. How do you troubleshoot?

**Answer:** Systematic checklist:

1. **Route table** — Does the private subnet's route table have `0.0.0.0/0 → NAT Gateway`?
2. **NAT Gateway** — Is the NAT GW in a **public** subnet? Is its status "Available"?
3. **NAT Gateway's subnet** — Does that public subnet have a route `0.0.0.0/0 → IGW`?
4. **Elastic IP** — Does the NAT Gateway have an EIP attached?
5. **Security Group** — Does the instance's SG allow **outbound** traffic to the destination?
6. **NACL** — Does the subnet's NACL allow outbound AND the **ephemeral port range** (1024-65535) for return traffic?
7. **DNS** — Can the instance resolve DNS? Check `enableDnsHostnames` and `enableDnsSupport` on the VPC.

```bash
# On the instance
curl -v https://google.com          # Test internet
nslookup google.com                  # Test DNS
traceroute 8.8.8.8                   # Trace path
ip route show                        # Check default route
```

---

### 13. Scenario: You have 3 VPCs (Dev, Staging, Prod). Dev and Staging can communicate, but Prod must be isolated. How do you design this?

**Answer:**

**Option 1: VPC Peering (simple)**
- Peer Dev ↔ Staging only
- No peering for Prod
- Prod connects only through controlled interfaces (API Gateway, PrivateLink)

**Option 2: Transit Gateway with route tables**
```
TGW Route Table "non-prod":
  - Attached: Dev VPC, Staging VPC
  - Routes: Dev CIDR ↔ Staging CIDR

TGW Route Table "prod":
  - Attached: Prod VPC
  - Routes: Only shared-services CIDR (monitoring, logging)
  - No routes to Dev or Staging
```

**Option 3: PrivateLink for controlled access**
- Prod exposes specific services via PrivateLink endpoints
- Dev/Staging consume those endpoints — no full VPC connectivity

---

### 14. Scenario: Your NAT Gateway costs are very high. How do you reduce them?

**Answer:** NAT Gateway charges: **$0.045/hr** + **$0.045/GB** processed.

**Cost reduction strategies:**

| Strategy | Savings |
|----------|---------|
| **S3 Gateway Endpoint** | Free — avoids NAT for S3 traffic |
| **Interface Endpoints** | ECR, CloudWatch, STS — bypass NAT |
| **Consolidate NAT GWs** | One shared NAT GW instead of per-AZ (trade HA for cost) |
| **Review traffic** | VPC Flow Logs → identify top talkers through NAT |
| **Move workloads** | If service only calls AWS APIs → VPC endpoints eliminate NAT entirely |
| **EKS optimization** | Private ECR endpoint + S3 endpoint = huge savings for image pulls |

```bash
# Analyze NAT GW traffic with VPC Flow Logs + Athena
SELECT
  srcaddr, dstaddr, dstport,
  SUM(bytes) / 1024 / 1024 / 1024 AS gb_transferred
FROM vpc_flow_logs
WHERE interface_id = 'eni-natgw-xxx'
  AND action = 'ACCEPT'
GROUP BY srcaddr, dstaddr, dstport
ORDER BY gb_transferred DESC
LIMIT 20;
```

---

### 15. Scenario: You need to allow a third-party vendor to access a specific service in your VPC securely. How?

**Answer:**

| Option | When to use |
|--------|------------|
| **PrivateLink (recommended)** | Vendor has AWS account; expose service via NLB + PrivateLink endpoint |
| **VPC Peering** | Trusted partner, need full network access (rare) |
| **VPN** | Vendor is on-prem; Site-to-Site VPN |
| **API Gateway** | Expose API over HTTPS with auth (API key, JWT, Cognito) |
| **Security Group + IP whitelist** | Public-facing service, restrict to vendor's IP range |

**PrivateLink approach (most secure):**
```
Your VPC:  Service → NLB → VPC Endpoint Service
Vendor VPC: VPC Interface Endpoint → connects to your endpoint service

Traffic never traverses the public internet.
Vendor cannot access anything else in your VPC.
```

---

### 16. Scenario: An application in VPC-A needs to access RDS in VPC-B across regions. How?

**Answer:**

**Option 1: Cross-region VPC Peering**
```
VPC-A (us-east-1) ←— VPC Peering —→ VPC-B (eu-west-1)
  App server                            RDS instance
```
- Simple, low latency for AWS backbone
- Update route tables + security groups in both VPCs
- Consider latency impact of cross-region DB calls

**Option 2: Transit Gateway Peering**
- Better if multiple VPCs need cross-region access
- Centralized routing

**Option 3: RDS Read Replica**
- Create cross-region read replica in VPC-A's region
- App reads from local replica (lower latency)
- Writes still go to primary in VPC-B

---

### 17. Scenario: How do you design a VPC for an EKS cluster with maximum security?

**Answer:**

```
VPC (10.0.0.0/16)
├── Public Subnets (10.0.1.0/24, 10.0.2.0/24)
│   ├── ALB / NLB (ingress)
│   └── NAT Gateway
│
├── Private Subnets (10.0.10.0/24, 10.0.11.0/24)
│   ├── EKS Worker Nodes (managed node groups)
│   └── Pods (secondary CIDR if needed)
│
└── Isolated Subnets (10.0.20.0/24, 10.0.21.0/24)
    └── RDS, ElastiCache (no internet)
```

**VPC Endpoints required for private EKS:**
- `com.amazonaws.<region>.ec2`
- `com.amazonaws.<region>.ecr.api`
- `com.amazonaws.<region>.ecr.dkr`
- `com.amazonaws.<region>.s3` (Gateway)
- `com.amazonaws.<region>.sts`
- `com.amazonaws.<region>.logs` (CloudWatch)
- `com.amazonaws.<region>.elasticloadbalancing`

**Security layers:**
- **NACLs** — Subnet-level deny rules for known bad ranges
- **Security Groups** — Pod-level with SG for Pods (EKS feature)
- **Network Policies** — Kubernetes-level (Calico/Cilium) for pod-to-pod
- **Private API server** — EKS API endpoint set to private only

---

### 18. Scenario: Your VPC is running out of IP addresses. What do you do?

**Answer:**

| Solution | Effort | Impact |
|----------|--------|--------|
| **Add secondary CIDR** | Low | Adds IP range to existing VPC |
| **Use smaller subnets** | Medium | Reclaim unused IPs |
| **EKS: Custom networking** | Medium | Pods use secondary CIDR (separate from nodes) |
| **EKS: Prefix delegation** | Low | Assign /28 prefixes to ENIs → more pod IPs |
| **Migrate to larger VPC** | High | New VPC with bigger CIDR |

```bash
# Add secondary CIDR
aws ec2 associate-vpc-cidr-block --vpc-id vpc-xxx --cidr-block 100.64.0.0/16

# EKS custom networking — pods use secondary CIDR
# Set ENIConfig per AZ pointing to subnets in secondary CIDR
# kubectl set env daemonset aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
```

---

### 19. Scenario: How do you enable VPC Flow Logs and use them for security auditing?

**Answer:**

```bash
# Enable flow logs (Terraform)
resource "aws_flow_log" "vpc" {
  vpc_id                = aws_vpc.main.id
  traffic_type          = "ALL"   # ACCEPT, REJECT, or ALL
  log_destination       = aws_s3_bucket.flow_logs.arn
  log_destination_type  = "s3"
  max_aggregation_interval = 60   # 60 seconds (vs default 600)

  log_format = "$${version} $${account-id} $${interface-id} $${srcaddr} $${dstaddr} $${srcport} $${dstport} $${protocol} $${packets} $${bytes} $${start} $${end} $${action} $${log-status}"
}
```

**Security audit queries (Athena):**
```sql
-- Rejected traffic (potential attacks or misconfigurations)
SELECT srcaddr, dstaddr, dstport, COUNT(*) as attempts
FROM vpc_flow_logs
WHERE action = 'REJECT'
GROUP BY srcaddr, dstaddr, dstport
ORDER BY attempts DESC LIMIT 20;

-- Traffic to unexpected ports
SELECT srcaddr, dstport, protocol, SUM(bytes) as total_bytes
FROM vpc_flow_logs
WHERE dstport NOT IN (80, 443, 22, 5432, 6379)
  AND action = 'ACCEPT'
GROUP BY srcaddr, dstport, protocol
ORDER BY total_bytes DESC;

-- Data exfiltration detection (large outbound transfers)
SELECT srcaddr, dstaddr, SUM(bytes)/1024/1024/1024 AS gb_sent
FROM vpc_flow_logs
WHERE action = 'ACCEPT' AND flow_direction = 'egress'
GROUP BY srcaddr, dstaddr
HAVING SUM(bytes) > 1073741824   -- > 1 GB
ORDER BY gb_sent DESC;
```

---

### 20. What is AWS PrivateLink and how does it differ from VPC Peering?

**Answer:**

| Feature | PrivateLink | VPC Peering |
|---------|------------|------------|
| **Scope** | Service-level (expose one service) | Network-level (full VPC access) |
| **Direction** | Unidirectional (consumer → provider) | Bidirectional |
| **CIDR overlap** | Allowed | Not allowed |
| **Transitive** | No | No |
| **Security** | Very granular (one service) | Broad (entire VPC CIDR) |
| **Use case** | SaaS, vendor access, microservice APIs | VPC-to-VPC full connectivity |

---

### 21. Scenario: You are designing a multi-account networking strategy. What do you recommend?

**Answer:**

```
                       ┌──────────────────┐
                       │  Network Account │
                       │  (Transit GW)    │
                       └────────┬─────────┘
                                │
        ┌───────────────────────┼───────────────────────┐
        ▼                       ▼                       ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Prod Account │     │ Dev Account  │     │Shared Svcs   │
│ VPC-Prod     │     │ VPC-Dev      │     │VPC-Shared    │
│ 10.1.0.0/16  │     │ 10.2.0.0/16 │     │ 10.0.0.0/16 │
└──────────────┘     └──────────────┘     └──────────────┘
```

**Recommendations:**
- **Centralized networking account** — owns Transit Gateway, VPN, Direct Connect
- **Non-overlapping CIDRs** — plan IP space across all accounts (use IPAM)
- **TGW route tables** — isolate prod from non-prod
- **RAM (Resource Access Manager)** — share TGW, subnets across accounts
- **Centralized egress** — shared NAT/firewall in network account
- **AWS Network Firewall or third-party** — inspect traffic at TGW level
- **VPC IPAM** — centrally manage IP address allocation
