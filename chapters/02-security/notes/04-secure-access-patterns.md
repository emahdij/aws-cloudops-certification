# Secure Access Patterns

## Overview

Secure access patterns eliminate common attack vectors like exposed SSH ports, hardcoded credentials, and unencrypted network traffic. Modern AWS architectures use IAM roles, Systems Manager Session Manager, VPC endpoints, and federated access to remove the need for traditional bastion hosts and long-term credentials.

**Regional Considerations**: Most secure access services are regional (Session Manager, VPC endpoints), but IAM Identity Center and federated access patterns work globally. Design your access architecture with regional boundaries in mind.

## Exam Relevance

This module covers these Domain 2 tasks:
- Implementing secure remote access without SSH/RDP keys
- Using AWS Systems Manager Session Manager for instance access
- Setting up VPC endpoints for private service access
- Implementing federated access and single sign-on
- Eliminating bastion hosts and jump servers

---

## Key Concepts

### Secure Access Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Session Manager** | Browser/CLI shell access via Systems Manager | Eliminates SSH key management | Connect to private EC2 without bastion |
| **VPC Endpoint** | Private connection to AWS services | Keeps traffic off internet | S3 access without NAT gateway |
| **Instance Connect** | Temporary SSH with IAM authentication | Short-lived SSH keys | One-time key valid for 60 seconds |
| **IAM Identity Center** | Centralized SSO for AWS accounts | Single login, multiple accounts | Access 50 accounts with one login |
| **Federated Access** | Use external identity provider | Leverage existing corporate directory | Active Directory for AWS access |
| **Gateway Endpoint** | Route table entry for S3/DynamoDB | Free, high-performance access | S3 access without internet gateway |
| **Interface Endpoint** | ENI-based private endpoint | Private access to most AWS services | Private CloudWatch Logs access |

#### Deep Dive into Access Methods

**Session Manager Benefits**:
- No inbound security group rules required
- All sessions logged to CloudWatch/S3
- Works with private instances (no internet access needed)
- Supports port forwarding and tunneling
- IAM-based access control

**VPC Endpoint Types**:
- **Gateway Endpoints**: S3 and DynamoDB only, no cost
- **Interface Endpoints**: Most other services, charged per hour
- **PrivateLink**: Third-party SaaS integration

> **🔍 Exam Alert:** Gateway endpoints are free and only work with S3 and DynamoDB. Interface endpoints cost $0.01/hour per AZ and support most other AWS services.

---

## Part 1: Session Manager (Replace SSH)

### What It Does

Access EC2 instances through browser or CLI without SSH keys or open ports.

**Traditional SSH problems**:
- Port 22 exposed to internet (attack surface)
- SSH keys to manage and rotate
- No audit trail of who did what
- Bastion host costs $10-50/month

**Session Manager solution**:
- No inbound ports needed
- IAM controls who can connect
- All commands logged to CloudWatch/S3
- Works with private instances (no internet)
- Free (no bastion host)

### How It Works

```
User → AWS Console/CLI → Systems Manager → SSM Agent on EC2 → Shell Access
```

**Requirements**:
1. SSM Agent installed (pre-installed on Amazon Linux 2/2023)
2. IAM role on instance with `AmazonSSMManagedInstanceCore` policy
3. Instance can reach Systems Manager API (internet OR VPC endpoints)

### Two Scenarios

**Scenario A: Instance Has Internet Access** (NAT Gateway or Public IP)
- Instance connects to Systems Manager over internet
- **Cost**: $0 (Session Manager is free)
- **No VPC endpoints needed**

**Scenario B: Instance Is Fully Private** (No internet at all)
- Instance needs VPC endpoints to reach Systems Manager
- **Cost**: $21.90/month (3 VPC endpoints × 3 AZs)
- **3 VPC endpoints required** (NOT 3 VPCs!)

> **Exam Alert**: You don't create 3 VPCs. You create 3 different endpoint SERVICES in your existing VPC for Session Manager to work.

---

## Part 2: VPC Endpoints (Private AWS Access)

### What Are VPC Endpoints?

VPC Endpoints let your instances access AWS services WITHOUT going through the internet.

**The Problem Without Endpoints**:

When EC2 in private subnet needs to use S3/CloudWatch/etc:
```
EC2 (10.0.1.5) → NAT Gateway → Internet Gateway → Internet → AWS S3
```

This means:
- Traffic leaves your VPC
- Goes over the internet (less secure)
- NAT Gateway costs $32/month + $0.045 per GB
- Slower (extra hops)

**The Solution With Endpoints**:
```
EC2 (10.0.1.5) → VPC Endpoint → AWS S3 (stays in AWS network)
```

Benefits:
- Traffic never leaves AWS network
- No internet routing
- More secure (no internet exposure)
- Cheaper (some endpoints are free)
- Faster (direct connection)

### Why You Need Them

**Scenario 1**: Private EC2 needs to upload logs to S3
- Without endpoint: Pay for NAT Gateway ($32/month minimum)
- With gateway endpoint: Free

**Scenario 2**: Private EC2 needs Session Manager access
- Without endpoints: Need internet access via NAT or IGW
- With endpoints: Direct connection, completely isolated

**Scenario 3**: Compliance requirement - no internet access
- VPC endpoints let you access AWS services in fully isolated VPC
- No internet gateway, no NAT gateway, still works

### Two Types

**Gateway Endpoints** (Free):
- Only S3 and DynamoDB
- Added to route table
- No cost, no hourly charge
- High performance

**Interface Endpoints** (Paid):
- All other AWS services (CloudWatch, Systems Manager, etc)
- Creates ENI in your subnet
- $0.01/hour per AZ ($7.30/month per AZ)
- Plus data processing charges

### When to Use Each

| Scenario | Solution | Cost |
|----------|----------|------|
| Private EC2 uploading to S3 | Gateway endpoint | $0 |
| Private EC2 writing CloudWatch Logs | Interface endpoint | $7.30/month per AZ |
| Session Manager in private subnet | 3 interface endpoints | $21.90/month (3 AZs) |
| High-volume S3 access | Gateway endpoint | $0 |
| Low-volume API calls to 10 services | NAT Gateway might be cheaper | Compare costs |

**Cost comparison**:
- Gateway endpoint: Free
- Interface endpoint (1 AZ): $7.30/month + data
- Interface endpoint (3 AZs): $21.90/month + data
- NAT Gateway: $32/month + $0.045/GB

> **Exam Alert**: Gateway endpoints are FREE and only for S3/DynamoDB. Interface endpoints cost money but work with most services.

---

## Part 3: Session Manager Setup

### IAM Role for Instances

Instance needs this managed policy: `arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore`

This policy allows:
- Registering with Systems Manager
- Sending session data
- Uploading logs to S3/CloudWatch

### For Private Subnets (No Internet)

**The 3 VPC Endpoints You Need** (NOT 3 VPCs!):

You create 3 **different endpoint services** in your **one VPC**:

1. **com.amazonaws.region.ssm**
   - Session Manager API calls
   - Registers instance with Systems Manager

2. **com.amazonaws.region.ssmmessages**
   - Session communication channel
   - Your terminal commands flow through here

3. **com.amazonaws.region.ec2messages**
   - EC2 status updates
   - Health checks and metadata

**Why 3 endpoints?**
- Session Manager is split into 3 different AWS services
- Each service needs its own private connection
- All 3 must exist for Session Manager to work

**Per AZ cost**:
- 3 endpoints × $0.01/hour × 1 AZ = $7.30/month
- 3 endpoints × $0.01/hour × 3 AZs = $21.90/month

All endpoints need:
- Security group allowing HTTPS (443) from VPC CIDR
- Private DNS enabled

**Without these 3 endpoints**: Instance can't connect to Systems Manager, you can't start sessions.

### For Public Subnets (Has Internet)

**No VPC endpoints needed!**
- Instance uses internet to reach Systems Manager
- Cost: $0

---

## Part 4: VPC Endpoints Setup

### Gateway Endpoint (S3)

Added to route table, points S3 traffic through endpoint:

**Route table entry created automatically**:
```
Destination: pl-12345678 (S3 prefix list)
Target: vpce-1234567890abcdef0
```

**When EC2 accesses S3**: Traffic goes through endpoint, never leaves AWS network.

### Interface Endpoint (CloudWatch Logs)

Creates ENI in your subnet with private IP:

**DNS resolution**:
- Public DNS: `logs.us-east-1.amazonaws.com` → private IP (10.0.1.5)
- Traffic stays in VPC

**Security group** must allow:
- Inbound: HTTPS (443) from VPC CIDR
- Outbound: All traffic (or specific as needed)

### Common Commands

```bash
# Start session
aws ssm start-session --target i-1234567890abcdef0

# Port forwarding (access RDS through EC2)
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=3306,localPortNumber=9090"

# Then connect locally
mysql -h 127.0.0.1 -P 9090 -u admin -p
```

---

## Part 5: EC2 Instance Connect

Temporary SSH alternative (different from Session Manager).

**How it works**:
1. You generate SSH key pair
2. AWS pushes public key to instance for 60 seconds
3. You SSH within 60 seconds
4. Key automatically removed

**Limitations**:
- Instance must have public IP or you must be on VPN
- Only works with Amazon Linux 2, Ubuntu 16.04+
- 60-second window
- Requires port 22 open (less secure than Session Manager)

**When to use**: Quick SSH access to public instances. For production, use Session Manager instead.

---

## Part 6: IAM Identity Center (Multi-Account SSO)

### What Is It?

**The Problem**: You have 20 AWS accounts. Without SSO, you'd need 20 different IAM users with 20 different passwords.

**The Solution**: IAM Identity Center (formerly AWS SSO) gives you one login portal for all your AWS accounts.

### How It Works

```
1. Alice logs into SSO portal (https://mycompany.awsapps.com/start)
2. She sees a list of accounts she can access:
   - Production Account (ReadOnly role)
   - Development Account (PowerUser role)
   - Staging Account (Admin role)
3. She clicks "Development Account" → automatically assumes PowerUser role
4. Works for 8 hours, then session expires
```

**Behind the scenes**:
- IAM Identity Center creates IAM roles in each account automatically
- When Alice clicks an account, she assumes that role temporarily
- All her actions are logged under her identity (not a shared role)

### Why Use It?

- **One login** for all accounts (no password juggling)
- **Connect to your company directory** (Active Directory, Okta, Google Workspace)
- **Automatic role creation** in new accounts
- **Audit trail** - know exactly who did what in which account

### Key Concepts

**Permission Set**: Template defining what users can do in an account.

Example permission sets:
- `AdministratorAccess` - Full access
- `PowerUserAccess` - Everything except IAM
- `ReadOnlyAccess` - View only
- `BillingAccess` - Cost Explorer only

**Assignment**: Link user/group + permission set + account.

```
User: alice@company.com
  + Permission Set: DeveloperAccess
  + Account: 123456789012 (dev account)
  = Alice gets developer access to dev account
```

> **Exam Alert**: IAM Identity Center is AWS's recommended way for multi-account access. It replaces creating individual IAM users in each account.

---

## Part 7: Federated Access (Use Your Corporate Login)

### What Is Federation?

**The Problem**: Your company has 5,000 employees in Active Directory. You don't want to create 5,000 IAM users.

**The Solution**: Federation lets employees use their existing company credentials (Active Directory, Okta, Google) to access AWS.

### How It Works (Simple Explanation)

```
1. Employee tries to access AWS
2. AWS redirects them to company login page (Active Directory)
3. Employee enters company username/password
4. Active Directory says "Yes, this is Alice, she's in Engineering group"
5. AWS gives Alice temporary credentials based on her group
6. Alice can now use AWS (for 1-12 hours)
```

**Key point**: Alice never has a permanent IAM user. She gets temporary credentials every time she logs in.

### SAML 2.0 (The Technical Protocol)

SAML is just the language that your identity provider (Active Directory, Okta) and AWS use to talk to each other.

**The handshake**:
```
Company IdP: "This is Alice, she's authenticated"
AWS: "Okay, I trust you. Here are temporary credentials for Alice"
```

### Why Use Federation?

- **No IAM users to manage** - employees already exist in your directory
- **Password policy controlled centrally** - IT can enforce rules
- **When someone leaves** - disable their AD account, they lose AWS access immediately
- **MFA** - use your existing MFA system (RSA, Duo, etc.)

### Federation vs IAM Identity Center

| Feature | Federation (SAML) | IAM Identity Center |
|---------|------------------|---------------------|
| **Setup complexity** | Complex (manual SAML config) | Simple (AWS-managed) |
| **Best for** | Single account or custom needs | Multi-account organizations |
| **Identity source** | Your IdP (AD, Okta) | Your IdP OR AWS directory |
| **AWS recommendation** | Legacy approach | Modern approach |

> **Exam Alert**: Both federation and IAM Identity Center let you use corporate credentials. Identity Center is easier and recommended for multi-account setups.

### Real-World Example

**Before federation**:
```
Your company: 500 employees
AWS setup: 500 IAM users to create, manage, rotate passwords for
Problem: Employee leaves → must remember to delete IAM user
```

**After federation**:
```
Your company: 500 employees
AWS setup: 1 SAML provider, roles for different groups
Problem solved: Employee leaves → disable AD account, AWS access gone
```

---

## Part 8: Production Scenarios

### Scenario 1: Remove Bastion Host

**Current setup**:
- Bastion host in public subnet ($8/month + management)
- SSH keys stored on laptop
- Port 22 exposed to internet
- No audit trail

**New setup**:
1. Remove bastion host
2. Attach IAM role to private instances (AmazonSSMManagedInstanceCore)
3. Create VPC endpoints: ssm, ssmmessages, ec2messages ($21.90/month for 3 AZs)
4. Connect via Session Manager

**Result**: 
- No SSH keys
- No open ports
- All commands logged
- Save management time

### Scenario 2: Private S3 Access

**Current setup**:
- Private instances use NAT Gateway to reach S3
- NAT Gateway costs $32/month + $0.045/GB

**New setup**:
1. Create S3 gateway endpoint (free)
2. Attach to private subnet route table
3. Remove NAT Gateway route for S3 traffic

**Result**:
- S3 traffic stays on AWS network
- No NAT Gateway costs for S3
- Better performance

### Scenario 3: Multi-Account Access

**Current setup**:
- 20 accounts, each developer has IAM user in each
- 20 passwords to manage
- Switching accounts requires re-login

**New setup**:
1. Enable IAM Identity Center in management account
2. Create permission sets (DevAccess, ReadOnly, Admin)
3. Assign groups to accounts with permission sets
4. Users log in once: `https://company.awsapps.com/start`

**Result**:
- One login for 20 accounts
- Temporary credentials (no access keys)
- Centralized access management

### Scenario 4: Port Forwarding to RDS

**Problem**: RDS in private subnet, developer needs to connect from laptop.

**Solution**:
```bash
# Start port forwarding through EC2
aws ssm start-session \
  --target i-1234567890abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=3306,localPortNumber=9090"

# Connect to RDS through tunnel
mysql -h 127.0.0.1 -P 9090 -u admin -p
```

**How it works**:
- Session Manager creates tunnel from your laptop to EC2
- Your laptop port 9090 forwards to EC2 port 3306
- EC2 connects to RDS in private subnet
- No VPN needed

### Scenario 5: Corporate Directory Integration

**Problem**: Company has 1,000 employees in Active Directory. Don't want 1,000 IAM users.

**Solution with IAM Identity Center**:
1. Connect Identity Center to Active Directory
2. Sync AD groups to Identity Center
3. Create permission sets for different roles
4. Assign AD groups to AWS accounts

**Result**:
- Employees use AD credentials to access AWS
- When employee leaves, disable AD account → AWS access gone
- No IAM users to manage

---

## Part 9: Common Issues

| Problem | Cause | Solution |
|---------|-------|----------|
| **Can't start session** | No IAM role on instance | Attach role with AmazonSSMManagedInstanceCore |
| **Instance not in SSM** | Can't reach SSM API | Add internet route or VPC endpoints |
| **Session disconnects immediately** | SSM Agent not running | Start agent: `systemctl start amazon-ssm-agent` |
| **VPC endpoint not working** | Security group blocks 443 | Allow HTTPS from VPC CIDR |
| **Private DNS not resolving** | DNS hostnames disabled on VPC | Enable DNS hostnames in VPC settings |
| **Port forwarding fails** | Port already in use locally | Use different local port number |

### Debugging Session Manager

```bash
# Check if instance registered with SSM
aws ssm describe-instance-information

# Check SSM Agent status on instance
sudo systemctl status amazon-ssm-agent

# View agent logs
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log

# Restart agent
sudo systemctl restart amazon-ssm-agent
```

---

## Part 10: Cost Comparison

### Access Methods

| Method | Monthly Cost | Security | Use Case |
|--------|--------------|----------|----------|
| **Bastion host** | $8 (t3.micro) + $3 (EIP) = $11 | Medium | Legacy systems |
| **Session Manager (with internet)** | $0 | High | Instances with NAT/IGW |
| **Session Manager (fully private)** | $21.90 (3 endpoints × 3 AZs) | Highest | Isolated workloads |
| **VPN** | $36 (VPN connection) + $0.05/GB | High | Office connectivity |

**Understanding the costs**:
- **$0**: Instance has internet (NAT Gateway or public IP), no VPC endpoints needed
- **$21.90**: Instance fully private, needs 3 VPC endpoints to reach Systems Manager
- **Per endpoint**: $0.01/hour × 730 hours/month = $7.30/month
- **3 endpoints × 3 AZs**: $7.30 × 3 = $21.90/month

### VPC Endpoints

| Endpoint | Monthly Cost (3 AZs) | Use Case |
|----------|---------------------|----------|
| **Gateway (S3)** | $0 | Always use for S3 |
| **Gateway (DynamoDB)** | $0 | Always use for DynamoDB |
| **Interface (1 service)** | $21.90 | Critical private access |
| **NAT Gateway** | $96 + data | Multiple services, compare costs |

**Decision**: 
- 1-2 services → Interface endpoints cheaper than NAT
- 5+ services → NAT Gateway might be cheaper
- Always use gateway endpoints for S3/DynamoDB (free)

---

## Part 11: Limitations

| Resource | Limit |
|----------|-------|
| Concurrent sessions per account | 1,000 |
| Session duration (default) | 60 minutes |
| Session duration (max) | 24 hours |
| Port forwarding sessions per instance | 25 |
| VPC endpoints per VPC | 255 |
| Instance Connect key validity | 60 seconds |

---

## Part 12: Exam Tips

**Key Rules**:
- Session Manager = no SSH keys, no open ports
- Gateway endpoints = free, S3/DynamoDB only
- Interface endpoints = $0.01/hour per AZ, most services
- Private subnet Session Manager needs 3 endpoints
- IAM Identity Center = multi-account SSO
- SAML = corporate directory integration

**Common Scenarios**:

| Question | Answer |
|----------|--------|
| Access private instances | Session Manager + VPC endpoints |
| Remove bastion host | Session Manager |
| Private S3 access | Gateway endpoint (free) |
| Multi-account SSO | IAM Identity Center |
| Connect to private RDS | Session Manager port forwarding |
| Corporate AD integration | SAML federation or Identity Center |

**Exam Tricks**:
- Session Manager doesn't require public IP or SSH keys
- Gateway endpoints only work with S3 and DynamoDB
- Private DNS must be enabled on VPC for endpoints
- IAM Identity Center uses temporary credentials (no access keys)
- Instance Connect needs port 22 open (less secure than Session Manager)

---

## Quick Reference

### Session Manager Commands
```bash
# Start session
aws ssm start-session --target i-xxxxx

# Port forwarding
aws ssm start-session --target i-xxxxx \
  --document-name AWS-StartPortForwardingSession \
  --parameters "portNumber=3306,localPortNumber=9090"
```

### VPC Endpoints for Session Manager
- `com.amazonaws.region.ssm`
- `com.amazonaws.region.ssmmessages`
- `com.amazonaws.region.ec2messages`
- `com.amazonaws.region.logs` (if logging enabled)

### Required IAM Policy
- `AmazonSSMManagedInstanceCore` (AWS managed policy)

---

## References

- [Session Manager User Guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [IAM Identity Center](https://docs.aws.amazon.com/singlesignon/latest/userguide/)

---

**Previous:** [← GuardDuty and Threat Detection](03-guardduty-threat-detection.md) | **Next:** [Networking →](../../03-networking/)
