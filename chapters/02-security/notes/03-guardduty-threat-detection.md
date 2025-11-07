# GuardDuty and Threat Detection

## Overview

Amazon GuardDuty is a threat detection service that continuously monitors for malicious activity and unauthorized behavior in your AWS environment. It analyzes AWS CloudTrail events, VPC Flow Logs, DNS logs, and S3 data events using machine learning, anomaly detection, and integrated threat intelligence feeds.

**Regional Service**: GuardDuty must be enabled in each region where you want threat detection. However, it can be managed centrally through a delegated administrator account in AWS Organizations, with findings aggregated across regions using Security Hub.

## Exam Relevance

This module covers these Domain 2 tasks:
- Enabling and configuring GuardDuty for threat detection
- Understanding GuardDuty finding types and severity levels
- Integrating GuardDuty with incident response workflows
- Implementing automated responses to security findings
- Managing multi-account GuardDuty deployments

---

## Key Concepts

### GuardDuty Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Finding** | Security issue detected by GuardDuty | Actionable security alert | EC2 instance communicating with botnet |
| **Threat Intelligence** | Database of known malicious IPs and domains | Identifies known bad actors | IP from threat feed accessing resources |
| **Anomaly Detection** | ML-based behavioral analysis | Catches unknown threats | Unusual API call patterns |
| **Data Sources** | AWS logs analyzed by GuardDuty | Foundation for threat detection | VPC Flow, DNS, CloudTrail, S3 logs |
| **Severity** | Impact level (Low/Medium/High) | Prioritizes response | High: Cryptocurrency mining detected |
| **Suppression Rule** | Filters to reduce noise | Focuses on real threats | Ignore known scanner IPs |
| **Trusted IP List** | IPs to exclude from findings | Prevents false positives | Corporate VPN egress IPs |
| **Threat List** | Custom malicious IP/domain list | Additional threat intelligence | Internal blacklist |

#### Deep Dive into Finding Types

**Reconnaissance**:
- Unauthorized port scanning
- Unusual API enumeration
- Suspicious instance metadata access
- Severity: Low to Medium
- Action: Monitor, investigate source

**Instance Compromise**:
- Malware communication
- Cryptocurrency mining
- Backdoor communication
- Severity: High
- Action: Immediate isolation and investigation

**Account Compromise**:
- Credential exfiltration
- Impossible travel (API calls from distant locations)
- Privilege escalation attempts
- Severity: High to Critical
- Action: Rotate credentials, review IAM changes

**Bucket Compromise**:
- Suspicious S3 access patterns
- Data exfiltration attempts
- Public bucket exposure
- Severity: Medium to High
- Action: Review bucket policies, check CloudTrail

> **🔍 Exam Alert:** High severity findings require immediate response. Know the difference between reconnaissance (pre-attack) and compromise (active attack) findings.

### Data Sources Analyzed

| Data Source | What It Detects | Automatically Analyzed | Additional Cost |
|-------------|-----------------|----------------------|-----------------|
| **VPC Flow Logs** | Network communication patterns | Yes | No |
| **DNS Logs** | Domain query patterns | Yes | No |
| **CloudTrail Management Events** | API call anomalies | Yes | No |
| **CloudTrail S3 Data Events** | S3 object access patterns | Optional | Yes |
| **EKS Audit Logs** | Kubernetes activity | Optional | Yes |
| **RDS Login Activity** | Database login attempts | Optional | Yes |
| **EBS Malware Protection** | Volume malware scanning | Optional | Yes |

---

## Service Configuration

### Basic Setup

```bash
# Enable GuardDuty in current region
aws guardduty create-detector \
  --enable \
  --finding-publishing-frequency FIFTEEN_MINUTES

# Get detector ID
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)

# Enable S3 protection
aws guardduty update-detector \
  --detector-id $DETECTOR_ID \
  --data-sources '{"S3Logs":{"Enable":true}}'

# Enable EKS protection
aws guardduty update-detector \
  --detector-id $DETECTOR_ID \
  --data-sources '{"Kubernetes":{"AuditLogs":{"Enable":true}}}'
```

### Terraform Configuration

```hcl
# Enable GuardDuty
resource "aws_guardduty_detector" "main" {
  enable                       = true
  finding_publishing_frequency = "FIFTEEN_MINUTES"

  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          enable = true
        }
      }
    }
  }

  tags = {
    Environment = "production"
  }
}

# Trusted IP list (whitelist)
resource "aws_guardduty_ipset" "trusted" {
  detector_id = aws_guardduty_detector.main.id
  name        = "TrustedIPs"
  format      = "TXT"
  location    = "https://s3.amazonaws.com/${aws_s3_bucket.guardduty.id}/trusted-ips.txt"
  activate    = true
}

# Threat list (blacklist)
resource "aws_guardduty_threatintelset" "custom" {
  detector_id = aws_guardduty_detector.main.id
  name        = "CustomThreatList"
  format      = "TXT"
  location    = "https://s3.amazonaws.com/${aws_s3_bucket.guardduty.id}/threat-ips.txt"
  activate    = true
}
```

---

## Multi-Account Setup

### Understanding GuardDuty Administrator Account

When you have multiple AWS accounts (via AWS Organizations), you need centralized security monitoring. GuardDuty provides a **delegated administrator account** model.

**Without Administrator Account** (❌ Don't do this):
```
Account A: GuardDuty enabled → findings in Account A console
Account B: GuardDuty enabled → findings in Account B console
Account C: GuardDuty enabled → findings in Account C console
Security Team: Must log into 50 different accounts to check findings!
```

**With Administrator Account** (✅ Best practice):
```
Delegated Admin Account (Security Account)
├── Aggregates findings from all member accounts
├── Manages GuardDuty configuration for organization
├── Can enable/disable GuardDuty in member accounts
└── Single pane of glass for security team

Management Account (Organization root)
└── Designates which account is the admin

Member Accounts (Production, Dev, Test...)
└── Automatically enabled, findings sent to admin
```

### Benefits of Administrator Account

| Benefit | Description | Example |
|---------|-------------|---------|
| **Centralized Findings** | All findings in one place | Security team sees all 50 accounts' findings |
| **Auto-Enable** | New accounts get GuardDuty automatically | New dev account created → GuardDuty enabled |
| **Unified Management** | Configure settings organization-wide | Enable S3 protection for all accounts at once |
| **Aggregated Dashboard** | Single view of security posture | One dashboard showing all threats |
| **Shared Threat Intel** | Trusted/threat lists apply to all accounts | Add VPN IPs once, applies everywhere |

### The Setup Process

**Step 1: Designate Administrator** (in management account):
```bash
# Management account designates security account as admin
aws guardduty enable-organization-admin-account \
  --admin-account-id 123456789012
```

This says: "Account 123456789012 is now the GuardDuty boss for this organization."

**Step 2: Configure Auto-Enable** (in delegated admin account):
```bash
# Get detector ID in admin account
DETECTOR_ID=$(aws guardduty list-detectors --query 'DetectorIds[0]' --output text)

# Enable auto-enrollment for all current and future member accounts
aws guardduty update-organization-configuration \
  --detector-id $DETECTOR_ID \
  --auto-enable \
  --data-sources '{
    "S3Logs": {"AutoEnable": true},
    "Kubernetes": {"AuditLogs": {"AutoEnable": true}},
    "MalwareProtection": {
      "ScanEc2InstanceWithFindings": {
        "EbsVolumes": {"AutoEnable": true}
      }
    }
  }'
```

**What happens now**:
1. All existing member accounts: GuardDuty enabled automatically
2. New accounts created: GuardDuty enabled at creation
3. All findings: Sent to delegated admin account
4. Member accounts: Can still view their own findings locally

### Organization-Wide Deployment Architecture

```
┌─────────────────────────────────────────────────────────┐
│ Management Account (000000000000)                       │
│ - Manages AWS Organizations                             │
│ - Designates delegated admin                            │
│ - Does NOT view GuardDuty findings                      │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Delegated Admin Account (111111111111)                  │
│ - Security team's central account                       │
│ - Receives ALL findings from member accounts            │
│ - Configures GuardDuty settings organization-wide       │
│ - Manages trusted IPs, threat lists, suppression rules  │
└─────────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────────┐
│ Member Accounts                                         │
│                                                          │
│ Production (222222222222)                               │
│ ├── GuardDuty auto-enabled                             │
│ └── Findings sent to admin account                      │
│                                                          │
│ Development (333333333333)                              │
│ ├── GuardDuty auto-enabled                             │
│ └── Findings sent to admin account                      │
│                                                          │
│ Testing (444444444444)                                  │
│ ├── GuardDuty auto-enabled                             │
│ └── Findings sent to admin account                      │
└─────────────────────────────────────────────────────────┘
```

> **Exam Alert**: In multi-account setups, the delegated administrator account (not management account) views and manages GuardDuty findings. Auto-enable ensures new accounts are protected immediately.

### Terraform Multi-Account

```hcl
# In management account: designate admin
resource "aws_guardduty_organization_admin_account" "admin" {
  admin_account_id = "123456789012"
}

# In delegated admin account: configure organization
resource "aws_guardduty_organization_configuration" "main" {
  detector_id = aws_guardduty_detector.main.id
  auto_enable = true

  datasources {
    s3_logs {
      auto_enable = true
    }
    kubernetes {
      audit_logs {
        auto_enable = true
      }
    }
    malware_protection {
      scan_ec2_instance_with_findings {
        ebs_volumes {
          auto_enable = true
        }
      }
    }
  }
}
```

---

## Implementation Examples

### Automated Response with EventBridge

```hcl
# EventBridge rule for high severity findings
resource "aws_cloudwatch_event_rule" "guardduty_high" {
  name        = "guardduty-high-severity"
  description = "Trigger on high severity GuardDuty findings"

  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
    detail = {
      severity = [
        { numeric = [">=", 7] }
      ]
    }
  })
}

# SNS notification target
resource "aws_cloudwatch_event_target" "sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_high.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.security_alerts.arn
}

# Lambda remediation target
resource "aws_cloudwatch_event_target" "lambda" {
  rule      = aws_cloudwatch_event_rule.guardduty_high.name
  target_id = "InvokeRemediation"
  arn       = aws_lambda_function.guardduty_remediation.arn
}
```

### Lambda Remediation Function

```python
import boto3
import json

ec2 = boto3.client('ec2')
iam = boto3.client('iam')

def lambda_handler(event, context):
    finding = event['detail']
    finding_type = finding['type']
    severity = finding['severity']
    
    # Compromised EC2 instance
    if 'CryptoCurrency' in finding_type or 'Backdoor' in finding_type:
        instance_id = finding['resource']['instanceDetails']['instanceId']
        isolate_instance(instance_id)
        
    # Compromised credentials
    elif 'UnauthorizedAccess:IAMUser' in finding_type:
        user_name = finding['resource']['accessKeyDetails']['userName']
        disable_user_keys(user_name)
        
    # Send to Security Hub
    send_to_security_hub(finding)
    
    return {'statusCode': 200}

def isolate_instance(instance_id):
    """Isolate compromised instance"""
    # Create isolation security group
    sg = ec2.create_security_group(
        GroupName=f'quarantine-{instance_id}',
        Description='Quarantine security group',
        VpcId='vpc-xxxxx'
    )
    
    # Remove all rules (deny all traffic)
    ec2.modify_instance_attribute(
        InstanceId=instance_id,
        Groups=[sg['GroupId']]
    )
    
    # Create snapshot for forensics
    volumes = ec2.describe_volumes(
        Filters=[{'Name': 'attachment.instance-id', 'Values': [instance_id]}]
    )
    for volume in volumes['Volumes']:
        ec2.create_snapshot(
            VolumeId=volume['VolumeId'],
            Description=f'Forensic snapshot - GuardDuty finding'
        )

def disable_user_keys(user_name):
    """Disable all access keys for compromised user"""
    keys = iam.list_access_keys(UserName=user_name)
    for key in keys['AccessKeyMetadata']:
        iam.update_access_key(
            UserName=user_name,
            AccessKeyId=key['AccessKeyId'],
            Status='Inactive'
        )
```

---

## Finding Management

### Suppression Rules

```bash
# Create suppression rule
aws guardduty create-filter \
  --detector-id $DETECTOR_ID \
  --name "IgnoreKnownScanners" \
  --action ARCHIVE \
  --finding-criteria '{
    "Criterion": {
      "type": {
        "Eq": ["Recon:EC2/PortProbeUnprotectedPort"]
      },
      "service.action.networkConnectionAction.remoteIpDetails.ipAddressV4": {
        "Eq": ["192.0.2.1"]
      }
    }
  }'
```

### Terraform Suppression Rules

```hcl
# Suppress findings from known security scanners
resource "aws_guardduty_filter" "known_scanners" {
  detector_id = aws_guardduty_detector.main.id
  name        = "ignore-known-scanners"
  action      = "ARCHIVE"
  rank        = 1

  finding_criteria {
    criterion {
      field  = "type"
      equals = ["Recon:EC2/PortProbeUnprotectedPort"]
    }

    criterion {
      field  = "service.action.networkConnectionAction.remoteIpDetails.ipAddressV4"
      equals = ["203.0.113.1", "203.0.113.2"]  # Known scanner IPs
    }
  }
}
```

---

## Integration with Security Hub

```hcl
# Enable Security Hub
resource "aws_securityhub_account" "main" {}

# GuardDuty product subscription in Security Hub
resource "aws_securityhub_product_subscription" "guardduty" {
  depends_on  = [aws_securityhub_account.main]
  product_arn = "arn:aws:securityhub:${data.aws_region.current.name}::product/aws/guardduty"
}

# Security Hub findings filter
resource "aws_securityhub_insight" "critical_findings" {
  filters {
    severity_label {
      comparison = "EQUALS"
      value      = "CRITICAL"
    }

    product_name {
      comparison = "EQUALS"
      value      = "GuardDuty"
    }
  }

  group_by_attribute = "ResourceType"
  name              = "Critical GuardDuty Findings"
}
```

---

## Monitoring and Alerting

### CloudWatch Metrics

GuardDuty doesn't publish metrics directly. Create custom metrics from findings:

```hcl
# Metric filter for GuardDuty findings
resource "aws_cloudwatch_log_metric_filter" "guardduty_findings" {
  name           = "guardduty-high-severity-count"
  log_group_name = "/aws/events/guardduty"
  pattern        = "{ $.detail.severity >= 7 }"

  metric_transformation {
    name      = "HighSeverityFindings"
    namespace = "Security/GuardDuty"
    value     = "1"
  }
}

# Alarm on high severity findings
resource "aws_cloudwatch_metric_alarm" "guardduty_high" {
  alarm_name          = "guardduty-high-severity"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "HighSeverityFindings"
  namespace           = "Security/GuardDuty"
  period              = 300
  statistic           = "Sum"
  threshold           = 0
  alarm_description   = "Alert on any high severity GuardDuty finding"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]
}
```

---

## Cost Management

### Pricing Structure
- **CloudTrail analysis**: $0.90 per million events
- **VPC Flow Logs analysis**: $1.18 per GB
- **DNS logs analysis**: $0.40 per million queries
- **S3 data events**: $0.80 per million events (if enabled)
- **EKS audit logs**: $0.80 per million events (if enabled)
- **Malware protection**: $0.15 per GB scanned (if enabled)

### Cost Optimization

1. **Disable Unused Features**
   ```bash
   # Disable S3 protection if not needed
   aws guardduty update-detector \
     --detector-id $DETECTOR_ID \
     --data-sources '{"S3Logs":{"Enable":false}}'
   ```

2. **Use Suppression Rules**
   - Reduces noise and alert fatigue
   - No cost impact (findings still analyzed)

3. **Monitor Costs**
   ```bash
   # Check GuardDuty usage
   aws ce get-cost-and-usage \
     --time-period Start=2024-01-01,End=2024-01-31 \
     --granularity MONTHLY \
     --metrics UsageQuantity \
     --filter file://guardduty-filter.json
   ```

---

## Security Best Practices

### Finding Response Workflow

1. **Immediate (High Severity)**
   - Isolate affected resources
   - Disable compromised credentials
   - Create forensic snapshots
   - Notify security team

2. **Within 24 Hours (Medium Severity)**
   - Investigate finding details
   - Review related CloudTrail logs
   - Implement preventive controls
   - Document incident

3. **Weekly Review (Low Severity)**
   - Aggregate and analyze patterns
   - Update suppression rules
   - Adjust security controls

### Automation Guidelines

```python
# Priority-based response
def determine_response(finding):
    severity = finding['severity']
    finding_type = finding['type']
    
    if severity >= 8.0:  # High
        return 'immediate_response'
    elif severity >= 4.0:  # Medium
        return 'investigate_within_24h'
    else:  # Low
        return 'weekly_review'
```

---

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| **No findings appearing** | GuardDuty not enabled in region | Enable GuardDuty in all active regions |
| **High false positive rate** | Default sensitivity | Create suppression rules for known patterns |
| **Missing S3 findings** | S3 protection not enabled | Enable S3 logs in detector settings |
| **Findings not in Security Hub** | Product not subscribed | Subscribe to GuardDuty in Security Hub |
| **EventBridge rule not triggering** | Incorrect event pattern | Verify event pattern matches finding structure |

### Debugging Steps

```bash
# Check detector status
aws guardduty get-detector --detector-id $DETECTOR_ID

# List recent findings
aws guardduty list-findings \
  --detector-id $DETECTOR_ID \
  --finding-criteria '{
    "Criterion": {
      "updatedAt": {
        "GreaterThanOrEqual": 1234567890000
      }
    }
  }'

# Get finding details
aws guardduty get-findings \
  --detector-id $DETECTOR_ID \
  --finding-ids finding-id-here
```

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Trusted IP lists** | 1 per detector | Max 2000 IPs per list |
| **Threat lists** | 6 per detector | Max 250,000 IPs per list |
| **Filters** | 100 per detector | Suppression rules |
| **Member accounts** | 5,000 per admin | Organization mode |
| **Finding retention** | 90 days | Export to S3 for longer retention |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding finding severity levels and response priorities
> - Multi-account GuardDuty setup with Organizations (delegated administrator account)
> - Automated response using EventBridge and Lambda
> - Difference between suppression rules and trusted IP lists
> - Data sources and what each detects (VPC Flow Logs, CloudTrail, DNS logs)
> - Integration with Security Hub

### Key Exam Scenarios

1. **High severity finding**: Immediate isolation and credential rotation
2. **Multi-account deployment**: Use delegated administrator with auto-enable for centralized management
3. **False positives**: Create suppression rules or trusted IP lists
4. **Missing findings**: Verify GuardDuty enabled in correct region
5. **Cost control**: Disable unused data sources (S3, EKS)
6. **Administrator account**: Designate in management account, configure in delegated admin account

### Common Exam Questions

| Question Pattern | Answer Pattern |
|------------------|----------------|
| "Centralize GuardDuty findings across 50 accounts" | Designate delegated administrator account |
| "Automatically enable GuardDuty in new accounts" | Configure auto-enable in organization settings |
| "Respond to credential exfiltration finding" | Rotate credentials, isolate instance, check CloudTrail |
| "Reduce false positives from known scanners" | Add IPs to trusted IP list |
| "Stop generating findings for specific pattern" | Create suppression rule |
| "Missing S3 object access findings" | Enable S3 protection in data sources |

---

## Quick Reference

### Essential Commands
```bash
# Enable GuardDuty
aws guardduty create-detector --enable

# List findings
aws guardduty list-findings --detector-id <id>

# Get finding details
aws guardduty get-findings --detector-id <id> --finding-ids <id>

# Create suppression rule
aws guardduty create-filter --detector-id <id> --name "rule" --action ARCHIVE

# Enable S3 protection
aws guardduty update-detector --detector-id <id> --data-sources '{"S3Logs":{"Enable":true}}'
```

### Finding Severity Levels
- **8.0-10.0**: High - Immediate action required
- **4.0-7.9**: Medium - Investigate within 24 hours
- **0.1-3.9**: Low - Review during regular security reviews

---

## References

- [GuardDuty User Guide](https://docs.aws.amazon.com/guardduty/latest/ug/)
- [GuardDuty Finding Types](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_finding-types-active.html)
- [GuardDuty Best Practices](https://docs.aws.amazon.com/guardduty/latest/ug/guardduty_best-practices.html)
- [Automating GuardDuty](https://aws.amazon.com/blogs/security/how-to-use-amazon-guardduty-and-aws-web-application-firewall-to-automatically-block-suspicious-hosts/)

---

**Previous:** [← KMS and Data Protection](02-kms-data-protection.md) | **Next:** [Secure Access Patterns →](04-secure-access-patterns.md)
