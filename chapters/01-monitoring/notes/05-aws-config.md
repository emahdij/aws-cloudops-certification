# AWS Config

## Overview

AWS Config provides a detailed view of the configuration of AWS resources in your account. It continuously monitors and records AWS resource configurations, allowing you to audit configuration history, assess compliance with configurations, and trigger automated remediation actions when resources deviate from expected states.

**Regional Service**: AWS Config is primarily a regional service. Configuration recorders, rules, and conformance packs are created and applied within specific regions. However, aggregators can provide multi-region and multi-account visibility.

## Exam Relevance

This module covers these Domain 1 tasks:
- Implementing configuration monitoring and compliance strategies
- Evaluating resources against compliance rules
- Setting up automated remediation
- Creating custom Config rules for organization-specific requirements

---

## Key Concepts

### Config Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Configuration Item (CI)** | Point-in-time snapshot of resource attributes | Core unit of configuration data | EC2 instance configuration including security groups, AMI, instance type |
| **Configuration History** | Collection of CIs for a resource over time | Enables change tracking and auditing | Track security group rule changes over past 6 months |
| **Configuration Recorder** | Engine that detects and captures resource changes | Required to use AWS Config | Setting up recorder in each region |
| **Config Rules** | Conditions for resource configuration compliance | Automated compliance checking | Ensure all EBS volumes are encrypted |
| **Remediation Actions** | Automated fixes for non-compliant resources | Self-healing infrastructure | Auto-encrypt unencrypted S3 buckets |

> **🔍 Exam Alert:** The SysOps exam often tests your understanding of which resource attributes are captured in configuration items and how to query this historical data.

### Real-World Example: Security Group Drift

**The Problem**: A developer accidentally opens port 22 (SSH) to 0.0.0.0/0 on a production security group.

**How Config Helps**:
1. **Configuration Recorder** captures the security group change
2. **Config Rule** detects the violation (unrestricted SSH access)
3. **Remediation Action** automatically removes the rule
4. **Configuration History** shows who made the change and when

---

## Service Configuration

### Basic Setup (Production Approach)

Most organizations start with this minimal but effective setup:

```hcl
# Enable AWS Config
resource "aws_config_configuration_recorder" "main" {
  name     = "security-compliance-recorder"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported = true
    include_global_resource_types = true
  }
}

# S3 bucket for configuration history
resource "aws_s3_bucket" "config" {
  bucket = "config-bucket-${data.aws_caller_identity.current.account_id}"
}

# Start recording
resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
}
```

### Essential Config Rules for Production

These are the "must-have" rules most organizations implement first:

```hcl
# Ensure EBS volumes are encrypted
resource "aws_config_config_rule" "encrypted_volumes" {
  name = "encrypted-volumes"
  
  source {
    owner             = "AWS"
    source_identifier = "ENCRYPTED_VOLUMES"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

# Check for unrestricted SSH access
resource "aws_config_config_rule" "restricted_ssh" {
  name = "no-unrestricted-ssh"
  
  source {
    owner             = "AWS"
    source_identifier = "INCOMING_SSH_DISABLED"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}

# Ensure S3 buckets are not publicly readable
resource "aws_config_config_rule" "s3_public_read" {
  name = "s3-bucket-public-read-prohibited"
  
  source {
    owner             = "AWS"
    source_identifier = "S3_BUCKET_PUBLIC_READ_PROHIBITED"
  }
  
  depends_on = [aws_config_configuration_recorder.main]
}
```

---

## Real-World Use Cases

### 1. Compliance Auditing

**Scenario**: Your organization needs to prove PCI-DSS compliance for credit card processing.

**Solution**: Deploy AWS Config with compliance-focused rules:
- All databases must be encrypted
- No unrestricted network access
- All changes must be logged

**Business Impact**: 
- Automated compliance reporting
- Reduced audit preparation time from weeks to hours
- Continuous compliance monitoring vs. annual audits

### 2. Change Management

**Scenario**: A production outage occurs after someone modifies a security group. You need to find what changed and when.

**Config Investigation Process**:
1. Check Configuration History for the affected security group
2. See exact changes made (which rules added/removed)
3. Identify who made the change via CloudTrail integration
4. Determine impact scope using resource relationships

### 3. Cost Optimization

**Scenario**: Identify underutilized resources that are costing money.

**Custom Config Rule Example**:
```python
def evaluate_compliance(configuration_item):
    # Simple custom rule for unused security groups
    if configuration_item['resourceType'] != 'AWS::EC2::SecurityGroup':
        return 'NOT_APPLICABLE'
    
    # Check if security group is attached to any instances
    if not configuration_item.get('relationships'):
        return 'NON_COMPLIANT'  # Unused security group
    
    return 'COMPLIANT'
```

### 4. Security Incident Response

**Scenario**: Security team suspects unauthorized changes to IAM policies.

**Investigation Steps**:
1. Review Configuration History for IAM roles/policies
2. Cross-reference with CloudTrail for API calls
3. Check Config Rules for policy compliance violations
4. Use findings to strengthen security controls

---

## Best Practices

### Implementation Strategy
1. **Start Small**: Begin with 5-10 essential rules
2. **Focus on Security**: Prioritize rules that prevent security violations
3. **Monitor Costs**: Each Config rule costs $0.001 per evaluation
4. **Use Managed Rules**: AWS-managed rules are tested and reliable

### Operational Guidelines
- Set up aggregators for multi-account visibility
- Enable remediation for critical security rules
- Review non-compliant resources weekly
- Archive old configuration data to S3 for cost savings

### Cost Management
- **Configuration Items**: $0.003 per item (first 100K free monthly)
- **Rule Evaluations**: $0.001 per evaluation
- **Typical Monthly Cost**: $50-200 for medium-sized environments

---

## Monitoring and Troubleshooting

### Common Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Missing configuration items** | Resources not appearing | Verify resource types are supported and included |
| **Rules showing "Insufficient Data"** | Rules not evaluating | Check IAM permissions |
| **High costs** | Unexpected charges | Review recording group settings |

### Key Metrics to Monitor
- Configuration items recorded per day
- Rule evaluation success rate
- Non-compliant resources count

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding configuration items and their contents
> - Deploying and managing Config rules
> - Implementing auto-remediation
> - Multi-account Config strategies

### Key Exam Scenarios

1. **Compliance Monitoring**: Identify non-compliant resources
2. **Change Tracking**: Determine who made configuration changes
3. **Resource Relationships**: Understand dependencies between resources
4. **Remediation**: Automatically fix non-compliant resources

---

## Quick Reference

### Essential AWS Managed Rules
```
ENCRYPTED_VOLUMES - EBS volumes must be encrypted
INCOMING_SSH_DISABLED - No unrestricted SSH access
S3_BUCKET_PUBLIC_READ_PROHIBITED - S3 buckets must not be publicly readable
ROOT_ACCESS_KEY_CHECK - Root account must not have access keys
```

### Common Resource Types
```
AWS::EC2::Instance - EC2 instances
AWS::EC2::SecurityGroup - Security groups
AWS::S3::Bucket - S3 buckets
AWS::RDS::DBInstance - RDS databases
AWS::IAM::Role - IAM roles
```

---

## References

- [AWS Config User Guide](https://docs.aws.amazon.com/config/latest/developerguide/WhatIsConfig.html)
- [AWS Config Managed Rules](https://docs.aws.amazon.com/config/latest/developerguide/managed-rules-by-aws-config.html)
- [AWS Config API Reference](https://docs.aws.amazon.com/config/latest/APIReference/Welcome.html)

---

**Previous:** [← CloudTrail](04-cloudtrail.md) | **Next:** [EventBridge →](06-eventbridge.md)