# AWS Systems Manager

## Overview

AWS Systems Manager is a unified service that provides operational insights and automation capabilities for managing your AWS infrastructure and applications. It offers a single interface to view and control your compute resources, patch operating systems, collect system inventory, and run commands across multiple instances simultaneously.

**Regional Service**: Systems Manager is a regional service that operates on resources within the same region. However, Session Manager and some automation features can work across regions when properly configured. Multi-region setups require separate configuration in each region.

## Exam Relevance

This module covers these Domain 1 tasks:
- Implementing automated patch management and system updates
- Managing instance configurations and system compliance
- Setting up secure remote access without SSH/RDP
- Creating operational runbooks and automation documents
- Monitoring system inventory and configuration drift

---

## Key Concepts

### Systems Manager Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **SSM Agent** | Software that enables Systems Manager functionality | Required on all managed instances | Pre-installed on Amazon Linux, Windows Server AMIs |
| **Managed Instance** | EC2 or on-premises server with SSM Agent | Core unit for management operations | EC2 instance appearing in Systems Manager console |
| **Document** | Defines actions that Systems Manager performs | Automation and standardization | AWS-UpdateSSMAgent for agent updates |
| **Parameter Store** | Secure storage for configuration data | Centralized configuration management | Database connection strings, API keys |
| **Session Manager** | Browser-based shell access to instances | Secure access without SSH keys | Connect to private instances without bastion hosts |
| **Patch Manager** | Automated operating system patching | Critical security and compliance tool | Monthly security patches for production servers |
| **Compliance** | Configuration and patch compliance monitoring | Ensures systems meet organizational standards | Track instances missing security patches |

#### Deep Dive into Core Components

**SSM Agent**:
- Lightweight software that processes requests from Systems Manager
- Communicates with Systems Manager service via HTTPS
- Runs with minimal system resources
- Updates automatically by default

**Managed Instances**:
- Must have SSM Agent installed and running
- Require IAM instance profile with appropriate permissions
- Need internet connectivity or VPC endpoints to reach Systems Manager

> **🔍 Exam Alert:** Understanding the prerequisites for managed instances is crucial. The exam often tests troubleshooting scenarios where instances don't appear in Systems Manager.

### Systems Manager vs. Traditional Tools

| Task | Traditional Approach | Systems Manager Approach |
|------|---------------------|-------------------------|
| **Remote Access** | SSH/RDP with key management | Session Manager with IAM authentication |
| **Patch Management** | Manual updates or custom scripts | Patch Manager with maintenance windows |
| **Configuration Management** | Configuration files on each server | Parameter Store with centralized management |
| **System Inventory** | Manual audits or third-party tools | Inventory service with automatic collection |
| **Compliance Monitoring** | Separate compliance tools | Built-in compliance dashboard |

---

## Service Configuration

### Basic Setup for Production

Most organizations start with this foundational setup:

```hcl
# IAM role for EC2 instances to use Systems Manager
resource "aws_iam_role" "ssm_role" {
  name = "EC2-SSM-Role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# Attach AWS managed policy for Systems Manager
resource "aws_iam_role_policy_attachment" "ssm_policy" {
  role       = aws_iam_role.ssm_role.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

# Instance profile for EC2 instances
resource "aws_iam_instance_profile" "ssm_profile" {
  name = "EC2-SSM-Profile"
  role = aws_iam_role.ssm_role.name
}

# Example EC2 instance with Systems Manager enabled
resource "aws_instance" "web_server" {
  ami                    = "ami-0abcdef1234567890"  # Amazon Linux 2
  instance_type          = "t3.micro"
  iam_instance_profile   = aws_iam_instance_profile.ssm_profile.name
  
  tags = {
    Name        = "WebServer-01"
    Environment = "Production"
    PatchGroup  = "WebServers"
  }
}
```

### Essential Parameter Store Configuration

```hcl
# Database connection string (encrypted)
resource "aws_ssm_parameter" "db_connection" {
  name  = "/myapp/database/connection-string"
  type  = "SecureString"
  value = "postgres://user:password@db.example.com:5432/myapp"
  
  tags = {
    Environment = "Production"
    Application = "MyApp"
  }
}

# Application configuration
resource "aws_ssm_parameter" "app_config" {
  name  = "/myapp/config/api-endpoint"
  type  = "String"
  value = "https://api.myapp.com/v1"
}

# Feature flags
resource "aws_ssm_parameter" "feature_flag" {
  name  = "/myapp/features/new-checkout"
  type  = "String"
  value = "enabled"
}
```

---

## Real-World Use Cases

### 1. Secure Remote Access

**Scenario**: Security team requires access to private EC2 instances without maintaining SSH keys or bastion hosts.

**Solution**: Use Session Manager for browser-based access with IAM authentication.

**Benefits**:
- No SSH keys to manage or rotate
- All sessions logged in CloudTrail
- Works with private instances (no public IPs needed)
- Multi-factor authentication support

**Implementation**:
```bash
# Connect to instance via Session Manager
aws ssm start-session --target i-1234567890abcdef0

# Run commands on multiple instances
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Production" \
  --parameters 'commands=["df -h", "free -m"]'
```

### 2. Automated Patch Management

**Scenario**: IT team needs to patch 200+ production servers monthly while minimizing downtime.

**Solution**: Use Patch Manager with maintenance windows for scheduled patching.

```hcl
# Patch baseline for Amazon Linux
resource "aws_ssm_patch_baseline" "amazon_linux" {
  name             = "AmazonLinux-Security-Updates"
  description      = "Security updates for Amazon Linux instances"
  operating_system = "AMAZON_LINUX_2"
  
  approval_rule {
    approve_after_days = 7
    patch_filter {
      key    = "CLASSIFICATION"
      values = ["Security", "Critical"]
    }
  }
  
  tags = {
    Environment = "Production"
  }
}

# Maintenance window for patching
resource "aws_ssm_maintenance_window" "patch_window" {
  name              = "production-patch-window"
  description       = "Monthly patching window"
  schedule          = "cron(0 2 ? * SUN *)"  # 2 AM every Sunday
  duration          = 4
  cutoff            = 1
  allow_unassociated_targets = false
  
  tags = {
    Environment = "Production"
  }
}
```

**Real Production Impact**:
- Reduced patch deployment time from 2 days to 4 hours
- Eliminated 90% of manual patching errors
- Achieved 99% patch compliance across fleet

### 3. Configuration Management

**Scenario**: Application configuration needs to be updated across 50 instances without redeployment.

**Solution**: Store configuration in Parameter Store and have applications read values dynamically.

```python
# Application code to read configuration
import boto3

def get_config():
    ssm = boto3.client('ssm')
    
    try:
        # Get multiple parameters at once
        response = ssm.get_parameters(
            Names=[
                '/myapp/database/connection-string',
                '/myapp/config/api-endpoint',
                '/myapp/features/new-checkout'
            ],
            WithDecryption=True
        )
        
        config = {}
        for param in response['Parameters']:
            key = param['Name'].split('/')[-1]  # Get last part of path
            config[key] = param['Value']
            
        return config
    except Exception as e:
        print(f"Error fetching configuration: {e}")
        return {}
```

### 4. System Inventory and Compliance

**Scenario**: Compliance team needs to track software inventory and ensure all instances meet security standards.

**Solution**: Use Systems Manager Inventory and Compliance features.

```hcl
# Inventory collection
resource "aws_ssm_association" "inventory" {
  name = "AWS-GatherSoftwareInventory"
  
  targets {
    key    = "tag:Environment"
    values = ["Production"]
  }
  
  schedule_expression = "rate(7 days)"
  
  parameters = {
    applications = "Enabled"
    files        = "Enabled"
    services     = "Enabled"
  }
}

# Compliance association for security configuration
resource "aws_ssm_association" "security_config" {
  name = "AWS-ConfigureAWSPackage"
  
  targets {
    key    = "tag:Environment"
    values = ["Production"]
  }
  
  parameters = {
    action = "Install"
    name   = "AmazonCloudWatchAgent"
  }
}
```

### 5. Automated Troubleshooting

**Scenario**: Operations team spends hours collecting diagnostic information when issues occur.

**Custom Automation Document**:
```hcl
resource "aws_ssm_document" "troubleshoot_web_server" {
  name          = "TroubleshootWebServer"
  document_type = "Command"
  document_format = "YAML"
  
  content = <<DOC
schemaVersion: '2.2'
description: Collect troubleshooting information from web servers
parameters:
  instanceIds:
    type: StringList
    description: Instance IDs to troubleshoot
mainSteps:
- action: aws:runShellScript
  name: collectDiagnostics
  inputs:
    runCommand:
    - echo "=== System Information ==="
    - uname -a
    - echo "=== Disk Usage ==="
    - df -h
    - echo "=== Memory Usage ==="
    - free -m
    - echo "=== Running Processes ==="
    - ps aux --sort=-%cpu | head -20
    - echo "=== Network Connections ==="
    - netstat -tulpn | grep :80
    - echo "=== Application Logs ==="
    - tail -50 /var/log/httpd/error_log
DOC
}
```

---

## Best Practices

### Implementation Strategy
1. **Start with Session Manager**: Replace SSH/RDP access first
2. **Implement Parameter Store**: Centralize configuration management
3. **Roll out Patch Manager**: Automate security updates
4. **Enable Inventory**: Track system state and compliance
5. **Create Automation**: Build runbooks for common tasks

### Security Guidelines
- Use IAM roles instead of access keys for EC2 instances
- Enable CloudTrail logging for all Systems Manager activities
- Encrypt sensitive parameters with KMS
- Regularly rotate Parameter Store values
- Use least-privilege access for Systems Manager operations

### Operational Excellence
- Tag instances consistently for targeted operations
- Use maintenance windows for disruptive operations
- Monitor Systems Manager operations with CloudWatch
- Test automation documents in non-production first
- Create standardized runbooks for common scenarios

---

## Monitoring and Troubleshooting

### Key Metrics to Monitor

| Metric | Purpose | Alert Threshold |
|--------|---------|-----------------|
| **CommandsSucceeded** | Successful command executions | Track for baseline |
| **CommandsFailed** | Failed command executions | > 5% failure rate |
| **ComplianceByConfigRule** | Configuration compliance status | < 95% compliance |
| **PatchComplianceOverallCompliance** | Patch compliance percentage | < 90% compliance |

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Instance not managed** | Instance doesn't appear in console | Check SSM Agent status, IAM permissions, network connectivity |
| **Commands timing out** | Operations fail with timeout | Increase timeout values, check instance performance |
| **Parameter access denied** | Applications can't read parameters | Verify IAM permissions, parameter exists |
| **Session Manager connection fails** | Can't establish browser session | Check VPC endpoints, security groups, agent status |

### Debugging Steps

```bash
# Check SSM Agent status on instance
sudo systemctl status amazon-ssm-agent

# View SSM Agent logs
sudo tail -f /var/log/amazon/ssm/amazon-ssm-agent.log

# Test connectivity to Systems Manager
aws ssm describe-instance-information --filters "Key=InstanceIds,Values=i-1234567890abcdef0"

# Check command execution history
aws ssm list-command-invocations --command-id "command-id-here"
```

---

## Cost Optimization

### Pricing Structure
- **Parameter Store**: First 10,000 requests per month free, then $0.05 per 10,000 requests
- **Advanced Parameters**: $0.05 per advanced parameter per month
- **Automation**: $0.002 per step execution
- **Session Manager**: No additional charges (uses existing EC2 costs)
- **Patch Manager**: No additional charges for the service itself

### Cost Optimization Strategies

1. **Use Standard Parameters**: Only use advanced parameters when needed
   - Standard: Up to 4KB value, no parameter policies
   - Advanced: Up to 8KB value, parameter policies, longer history

2. **Optimize Automation**: 
   - Use fewer steps in automation documents
   - Batch operations when possible

3. **Efficient Parameter Organization**:
   ```bash
   # Good: Hierarchical organization
   /myapp/prod/database/host
   /myapp/prod/database/port
   /myapp/prod/api/endpoint
   
   # Avoid: Too many individual parameters
   /myapp/prod/db_host_1
   /myapp/prod/db_host_2
   /myapp/prod/db_host_3
   ```

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Managed instances** | 200,000 | Per region per account |
| **Parameters** | 10,000 | Standard parameters per region |
| **Advanced parameters** | 100,000 | Per region per account |
| **Concurrent executions** | 100 | Run Command per account |
| **Automation executions** | 100 | Concurrent per account |
| **Session Manager sessions** | 1,000 | Concurrent per region |
| **Parameter value size** | 4KB (standard), 8KB (advanced) | Maximum size |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding when to use Systems Manager vs. other AWS services
> - Troubleshooting managed instance connectivity issues
> - Configuring patch management and maintenance windows
> - Using Parameter Store for configuration management
> - Implementing Session Manager for secure access

### Key Exam Scenarios

1. **Secure Access**: Replace SSH/RDP with Session Manager
2. **Patch Management**: Automate OS updates with maintenance windows
3. **Configuration Management**: Use Parameter Store for application settings
4. **Compliance Monitoring**: Track system compliance with Inventory
5. **Automation**: Create runbooks for operational tasks

---

## Quick Reference

### Essential Commands
```bash
# Connect to instance
aws ssm start-session --target i-1234567890abcdef0

# Run command on instances
aws ssm send-command --document-name "AWS-RunShellScript" --targets "Key=tag:Environment,Values=Production" --parameters 'commands=["uptime"]'

# Get parameter value
aws ssm get-parameter --name "/myapp/database/host" --with-decryption

# List managed instances
aws ssm describe-instance-information
```

### Common Document Names
```bash
# AWS-managed documents
AWS-RunShellScript       # Run shell commands
AWS-RunPowerShellScript  # Run PowerShell commands
AWS-UpdateSSMAgent       # Update SSM Agent
AWS-ConfigureAWSPackage  # Install/uninstall packages
AWS-GatherSoftwareInventory  # Collect system inventory
```

---

## References

- [AWS Systems Manager User Guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/)
- [Systems Manager Prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-prereqs.html)
- [Parameter Store User Guide](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [Session Manager Working with Sessions](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions.html)

---

**Previous:** [← EventBridge](06-eventbridge.md) | **Next:** [X-Ray →](08-x-ray.md)