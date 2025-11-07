# AWS Security & Identity Cheatsheet

Quick reference guide for AWS CloudOps certification

## IAM Fundamentals

### Core Concepts
- **User**: Person or application with permanent credentials
- **Group**: Collection of users with same permissions
- **Role**: Temporary credentials assumed by users/services
- **Policy**: JSON document defining permissions
- **Permission Boundary**: Maximum permissions (not grants, only limits)

### Policy Types
- **Identity-Based**: Attached to users, groups, roles
- **Resource-Based**: Attached to resources (S3, KMS, Lambda)
- **Trust Policy**: Who can assume a role (roles only)
- **SCP**: Organization-level guardrails (AWS Organizations)
- **Permission Boundary**: Maximum allowed permissions

### Policy Evaluation Order
1. **Explicit Deny** - Always wins, cannot be overridden
2. **SCP** - Organization limits (if using AWS Organizations)
3. **Permission Boundary** - Maximum permissions cap
4. **Identity/Resource Policy** - Grants permissions
5. **Default Deny** - If no allow, access denied

### Key Points for Exam
- **Root User**: Can't be restricted, enable MFA immediately
- **Groups**: Can't be principals in policies (use roles instead)
- **Cross-Account**: Needs both resource policy AND role assumption
- **External ID**: Prevents confused deputy problem
- **IAM is global**: But some features are regional (Access Analyzer)

### Common Patterns
```json
// EC2 Instance Role Trust Policy
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}

// Cross-Account Role with External ID
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:root"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {"sts:ExternalId": "unique-secret"}
    }
  }]
}
```

## KMS & Data Protection

### Core Concepts
- **Customer Managed Key**: You control lifecycle, $1/month
- **AWS Managed Key**: AWS controls, free (e.g., aws/s3)
- **Data Key**: Encrypts large data (envelope encryption)
- **Key Policy**: Who can use/manage the key
- **Grant**: Temporary, programmatic access to keys

### Key Types
- **Symmetric (AES-256)**: Single key, most common, can't export
- **Asymmetric (RSA)**: Public/private pair, can export public key
- **HMAC**: Generate and verify message authentication codes

### Encryption Patterns
- **Server-Side**: AWS encrypts data before storing
  - SSE-S3: AWS managed keys (free)
  - SSE-KMS: KMS keys (audit trail, $1/month)
  - SSE-C: Customer provided keys (you manage)
- **Client-Side**: Encrypt before sending to AWS
- **Envelope Encryption**: KMS encrypts data key, data key encrypts data

### Key Points for Exam
- **Regional Service**: Keys don't work cross-region
- **Cannot Export**: Symmetric key material stays in KMS HSMs
- **Rotation**: Automatic yearly, free (AWS keeps all versions)
- **4KB Limit**: Direct encryption only for small data
- **Deletion**: 7-30 day waiting period (can't delete immediately)

### Service Integration
- **S3**: Per-object encryption, use bucket keys for cost savings
- **EBS**: Encrypt at launch or from encrypted snapshot
- **RDS**: Can't encrypt existing DB (must create from snapshot)
- **Secrets Manager**: Automatic rotation with KMS encryption
- **Lambda**: Encrypt environment variables

## GuardDuty & Threat Detection

### Core Concepts
- **Finding**: Security issue detected by GuardDuty
- **Detector**: Per-region configuration that analyzes data
- **Trusted IP List**: IPs to exclude from findings (whitelist)
- **Threat List**: Known malicious IPs to flag (blacklist)
- **Suppression Rule**: Auto-archive specific finding types

### Data Sources
- **VPC Flow Logs**: Network traffic patterns (auto-enabled, free)
- **DNS Logs**: Domain queries (auto-enabled, free)
- **CloudTrail Events**: API calls (auto-enabled, free)
- **S3 Data Events**: Object access (optional, costs extra)
- **EKS Audit Logs**: Kubernetes activity (optional, costs extra)

### Severity Levels
- **8.0-10.0 (High)**: Immediate action - compromise detected
- **4.0-7.9 (Medium)**: Investigate within 24 hours
- **0.1-3.9 (Low)**: Review during regular security reviews

### Finding Categories
- **Reconnaissance**: Pre-attack probing (port scans, API enumeration)
- **Instance Compromise**: EC2 attacked (crypto mining, backdoors)
- **Account Compromise**: Credentials stolen (impossible travel)
- **Bucket Compromise**: S3 attacked (data exfiltration)

### Key Points for Exam
- **Regional Service**: Enable in all regions
- **30-Day Learning**: Fewer findings initially during ML training
- **Doesn't Prevent**: Only detects threats (use with Security Groups)
- **Findings Retention**: 90 days (export to S3 for longer)
- **Multi-Account**: Use delegated administrator with auto-enable

### Common High-Severity Findings
- `CryptoCurrency:EC2/BitcoinTool.B!DNS` - Crypto mining
- `Backdoor:EC2/C&CActivity.B` - Command & control communication
- `UnauthorizedAccess:IAMUser/InstanceCredentialExfiltration` - Impossible travel
- `Exfiltration:S3/ObjectRead.Unusual` - Unusual S3 data access

## Secure Access Patterns

### Session Manager
- **What**: Browser/CLI shell access without SSH keys
- **Requirements**: SSM Agent, IAM role, internet or VPC endpoints
- **Benefits**: No open ports, full audit logging, IAM-based access
- **Cost**: Free (VPC endpoints cost if fully private)

### VPC Endpoints
- **Gateway Endpoint**: Free, route table entry (S3, DynamoDB only)
- **Interface Endpoint**: $0.01/hour per AZ (most services)
- **Use Case**: Keep traffic private, avoid NAT Gateway

### VPC Endpoints for Session Manager
Required 3 endpoints for private subnet access:
- `com.amazonaws.region.ssm` - Session Manager API
- `com.amazonaws.region.ssmmessages` - Session communication
- `com.amazonaws.region.ec2messages` - EC2 status updates

**Cost**: 3 endpoints × 3 AZs × $0.01/hour = $21.90/month

### EC2 Instance Connect
- **What**: Temporary SSH with IAM authentication
- **Key Validity**: 60 seconds after pushing public key
- **Limitation**: Requires port 22 open, less secure than Session Manager
- **Use Case**: Quick SSH to public instances

### IAM Identity Center (SSO)
- **What**: Single sign-on for multiple AWS accounts
- **Components**: Users/Groups, Permission Sets, Account Assignments
- **Benefits**: One login, temporary credentials, no IAM users
- **Use Case**: Multi-account access with corporate directory

### Federated Access (SAML)
- **What**: Use corporate identity provider (AD, Okta) for AWS access
- **Flow**: User → IdP authentication → AWS temporary credentials
- **Benefits**: No IAM users, centralized access control
- **Use Case**: Integrate existing corporate directory

### Key Points for Exam
- **Session Manager**: No SSH keys, works with private instances
- **Gateway Endpoints**: Always free for S3/DynamoDB
- **Interface Endpoints**: Only for private subnets without NAT
- **IAM Identity Center**: Recommended for multi-account SSO
- **SAML**: Legacy approach, Identity Center is modern alternative

## Exam Strategy Tips

### Service Selection
- **Access Control**: IAM for who can do what
- **Data Protection**: KMS for encryption, Secrets Manager for credentials
- **Threat Detection**: GuardDuty for ML-based detection
- **Secure Access**: Session Manager for instances, VPC endpoints for services

### Cost Optimization
- **KMS**: Use envelope encryption, enable S3 bucket keys
- **GuardDuty**: Disable S3 protection if not needed
- **VPC Endpoints**: Gateway (free) > Interface ($21.90/month) > NAT ($96/month)
- **Session Manager**: Free if instances have internet, costs for fully private

### Regional Considerations
- **IAM**: Global service (users, roles, policies)
- **KMS**: Regional (keys don't work across regions)
- **GuardDuty**: Regional (enable in all regions)
- **VPC Endpoints**: Regional (create per region)

### Common Patterns
- **IAM Role → EC2**: No access keys, automatic rotation
- **KMS Key → S3 Bucket**: Encrypted at rest with audit trail
- **GuardDuty → EventBridge → Lambda**: Automated threat response
- **Session Manager → VPC Endpoints**: Private instance access
- **Permission Boundary → Developer**: Prevent privilege escalation

### Security Layers (Defense in Depth)
1. **Network**: Security Groups, NACLs, VPC endpoints
2. **Identity**: IAM roles, permission boundaries, MFA
3. **Data**: KMS encryption, Secrets Manager
4. **Monitoring**: GuardDuty, CloudTrail, Config
5. **Response**: EventBridge automation, Session Manager access

---

> **🎯 Exam Focus**: Understand when to use each security service, their cost implications, and how they work together. Practice IAM policy evaluation order and troubleshooting "access denied" scenarios.