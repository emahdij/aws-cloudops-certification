# CloudTrail

## Overview

AWS CloudTrail is a service that enables governance, compliance, operational auditing, and risk auditing of your AWS account. It records API calls and related events made by or on behalf of your AWS account, providing a comprehensive audit trail of all actions taken in your AWS environment.


**Regional and Global Service Aspects**: 
- CloudTrail itself is deployed as a regional service, but it can monitor both regional and global services
- **Trail Configuration**: A trail can be configured as a single-region trail or a multi-region trail
- **Global Service Events**: CloudTrail can record events from global services (like IAM, Route 53, CloudFront) in any region where you enable CloudTrail
- **Best Practice**: Configure multi-region trails that include global service events to ensure comprehensive coverage

This unique capability makes CloudTrail essential for organizations that need complete visibility across their entire AWS environment regardless of regional boundaries.

## Exam Relevance

This module covers these Domain 1 tasks:
- Implementing monitoring, logging, and remediation strategies
- Configuring audit trails for API calls and user activities
- Analyzing CloudTrail logs for security and compliance
- Integrating CloudTrail with CloudWatch for real-time monitoring

---

## Key Concepts

### CloudTrail Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Trail** | Configuration that records AWS API calls and delivers log files | Core audit mechanism | Organization trail capturing all account activity |
| **Event** | Record of an activity in an AWS account | Individual audit record | User creating an EC2 instance |
| **Event History** | 90-day record of management events | Built-in audit capability | View who deleted an S3 bucket last week |
| **Log File** | JSON file containing CloudTrail event records | Storage format for events | File with 1000+ API calls from one hour |
| **Management Events** | Control plane operations | Track administrative actions | CreateUser, DeleteStack, ModifyDBInstance |
| **Data Events** | Data plane operations on resources | Monitor data access patterns | S3 GetObject, Lambda function invocations |
| **Insight Events** | Unusual activity patterns detected by ML | Automated anomaly detection | Sudden spike in API call frequency |

#### Deep Dive into Event Types

**Management Events (Control Plane)**
- Administrative operations that modify AWS resources
- Enabled by default in all trails
- Examples: EC2 instance launch, VPC configuration changes, RDS modifications
- Critical for security auditing and compliance

**Data Events (Data Plane)**
- Resource operations on or within a resource
- Not enabled by default (additional cost)
- Examples: S3 object operations, Lambda function executions
- Important for detailed access monitoring

**Insight Events**
- Machine learning analysis of unusual activity
- Automatically detects patterns like unusual API call rates
- Additional cost but valuable for security monitoring

> **🔍 Exam Alert:** Know the difference between management and data events, and understand that data events require explicit configuration and incur additional costs.

### CloudTrail Integration Points

| AWS Service | Integration Type | Use Case |
|-------------|------------------|----------|
| **CloudWatch Logs** | Log stream delivery | Real-time monitoring and alerting |
| **CloudWatch Events/EventBridge** | Event-driven triggers | Automated response to API calls |
| **S3** | Log file storage | Long-term audit log retention |
| **SNS** | Notifications | Alert on log file delivery |
| **Lambda** | Event processing | Custom analysis and response |
| **Athena** | Log analysis | SQL queries on historical logs |

---

## Service Configuration

### Trail Configuration Options

#### Basic Trail Setup
```bash
# Create a basic trail
aws cloudtrail create-trail \
  --name "my-organization-trail" \  # Descriptive name for the trail
  --s3-bucket-name "my-cloudtrail-logs-bucket" \  # S3 bucket where logs will be stored
  --include-global-service-events \  # Include events from global services like IAM
  --is-multi-region-trail \  # Record events from all regions
  --enable-log-file-validation  # Enable log file integrity validation

# Start logging
aws cloudtrail start-logging --name "my-organization-trail"  # Activate the trail
```

#### Advanced Trail Configuration
```bash
# Create a comprehensive trail with data events and insights
aws cloudtrail create-trail \
  --name "advance-trail" \
  --s3-bucket-name "my-audit-logs" \
  --s3-key-prefix "cloudtrail-logs/" \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

# Configure data events for S3
aws cloudtrail put-event-selectors \
  --trail-name "advance-trail" \  # Target trail
  --event-selectors '[
    {
      "ReadWriteType": "All",  # Capture both read and write events
      "IncludeManagementEvents": true,  # Include management events
      "DataResources": [
        {
          "Type": "AWS::S3::Object",  # S3 object-level operations
          "Values": ["arn:aws:s3:::my-sensitive-bucket/*"]  # Specific bucket to monitor
        },
        {
          "Type": "AWS::Lambda::Function",  # Lambda execution monitoring
          "Values": ["arn:aws:lambda:*"]  # All Lambda functions
        }
      ]
    }
  ]'

# Enable insights
aws cloudtrail put-insight-selectors \
  --trail-name "advance-trail" \  # Target trail
  --insight-selectors '[
    {
      "InsightType": "ApiCallRateInsight"  # Detect unusual API call patterns
    }
  ]'
```

> **Note:** IAM permissions and roles for CloudTrail will be covered in detail in the IAM security chapter. This section shows basic integration points only.

---

## Implementation Examples

### Terraform Configuration

```hcl
# S3 bucket for CloudTrail logs
resource "aws_s3_bucket" "cloudtrail" {
  bucket        = "my-org-cloudtrail-logs-${random_id.bucket_suffix.hex}"  # Unique bucket name with random suffix
  force_destroy = false  # Prevent accidental deletion

  tags = {
    Purpose = "CloudTrail Logs"
    Compliance = "Required"
  }
}

resource "random_id" "bucket_suffix" {
  byte_length = 4  # Generate random suffix for global uniqueness
}

# S3 bucket policy for CloudTrail
resource "aws_s3_bucket_policy" "cloudtrail" {
  bucket = aws_s3_bucket.cloudtrail.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AWSCloudTrailAclCheck"  # Allow CloudTrail to check bucket ACL
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.cloudtrail.arn
        Condition = {
          StringEquals = {
            "AWS:SourceArn" = "arn:aws:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/organization-trail"
          }
        }
      },
      {
        Sid    = "AWSCloudTrailWrite"  # Allow CloudTrail to write log files
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:PutObject"
        Resource = "${aws_s3_bucket.cloudtrail.arn}/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
            "AWS:SourceArn" = "arn:aws:cloudtrail:${data.aws_region.current.name}:${data.aws_caller_identity.current.account_id}:trail/organization-trail"
          }
        }
      }
    ]
  })
}

# CloudWatch Log Group for CloudTrail
resource "aws_cloudwatch_log_group" "cloudtrail" {
  name              = "/aws/cloudtrail/organization-trail"  # Logical name for log group
  retention_in_days = 30  # Keep logs for 30 days
  kms_key_id        = aws_kms_key.cloudtrail.arn  # Encrypt logs with KMS

  tags = {
    Purpose = "CloudTrail Logs"
  }
}

# Basic IAM role for CloudTrail to CloudWatch Logs integration
# Note: Detailed IAM coverage will be in the IAM chapter
resource "aws_iam_role" "cloudtrail_logs" {
  name = "CloudTrail_CloudWatchLogs_Role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow",
      Principal = { Service = "cloudtrail.amazonaws.com" },
      Action = "sts:AssumeRole"
    }]
  })
}

# KMS key for CloudTrail encryption
resource "aws_kms_key" "cloudtrail" {
  description             = "KMS key for CloudTrail encryption"
  deletion_window_in_days = 7  # Recovery period if scheduled for deletion

  # Basic policy to allow CloudTrail encryption
  # Note: Detailed KMS policies will be covered in the Security chapter
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions",
        Effect = "Allow",
        Principal = { AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root" },
        Action   = "kms:*",
        Resource = "*"
      },
      {
        Sid    = "Allow CloudTrail to encrypt logs",
        Effect = "Allow",
        Principal = { Service = "cloudtrail.amazonaws.com" },
        Action = ["kms:GenerateDataKey*", "kms:DescribeKey"],
        Resource = "*"
      }
    ]
  })

  tags = {
    Purpose = "CloudTrail Encryption"
  }
}

# CloudTrail with comprehensive configuration
resource "aws_cloudtrail" "organization_trail" {
  name           = "organization-trail"  # Trail name
  s3_bucket_name = aws_s3_bucket.cloudtrail.bucket  # Target S3 bucket
  s3_key_prefix  = "cloudtrail-logs/"  # Prefix for organization

  # Multi-region and global service events
  is_multi_region_trail        = true  # Record events from all regions
  include_global_service_events = true  # Include global services like IAM
  enable_log_file_validation   = true  # Enable log file integrity validation

  # CloudWatch Logs integration
  cloud_watch_logs_group_arn = "${aws_cloudwatch_log_group.cloudtrail.arn}:*"  # Send to CloudWatch Logs
  cloud_watch_logs_role_arn  = aws_iam_role.cloudtrail_logs.arn  # Role for CloudWatch delivery

  # Encryption
  kms_key_id = aws_kms_key.cloudtrail.arn  # Encrypt logs with KMS

  # Data events configuration
  event_selector {
    read_write_type           = "All"  # Both read and write operations
    include_management_events = true  # Include management events

    # S3 data events for specific buckets
    data_resource {
      type   = "AWS::S3::Object"  # S3 object-level operations
      values = ["arn:aws:s3:::my-sensitive-bucket/*"]  # Specific bucket to monitor
    }

    # Lambda data events
    data_resource {
      type   = "AWS::Lambda::Function"  # Lambda execution monitoring
      values = ["arn:aws:lambda:*"]  # All Lambda functions
    }
  }

  # Insights configuration
  insight_selector {
    insight_type = "ApiCallRateInsight"  # Detect unusual API call patterns
  }

  depends_on = [
    aws_s3_bucket_policy.cloudtrail,  # Ensure bucket policy exists first
    aws_cloudwatch_log_group.cloudtrail
  ]

  tags = {
    Environment = "Production"
    Compliance  = "Required"
  }
}

# Data sources
data "aws_caller_identity" "current" {}  # Get current account ID
data "aws_region" "current" {}  # Get current region
```

### CloudWatch Integration

```bash
# Create metric filter for failed console logins
aws logs put-metric-filter \
  --log-group-name "/aws/cloudtrail/organization-trail" \  # CloudTrail log group
  --filter-name "ConsoleLoginFailures" \  # Name for the filter
  --filter-pattern '{ ($.errorCode = "*UnauthorizedOperation") || ($.errorCode = "AccessDenied*") || ($.errorCode = "InvalidUserID*") }' \  # JSON pattern matching failed logins
  --metric-transformations \
    metricName=ConsoleLoginFailures,metricNamespace=Security,metricValue=1  # Create Security metric

# Create metric filter for unusual API calls (non-IAM specific)
aws logs put-metric-filter \
  --log-group-name "/aws/cloudtrail/organization-trail" \  # CloudTrail log group
  --filter-name "UnusualAPIActivity" \  # Name for the filter
  --filter-pattern '{ ($.sourceIPAddress != "internal.amazonaws.com") && (($.eventName = DeleteTrail) || ($.eventName = UpdateTrail) || ($.eventName = StopLogging)) }' \  # Pattern for CloudTrail modification events
  --metric-transformations \
    metricName=UnusualAPIActivity,metricNamespace=Security,metricValue=1  # Create Security metric

# Create alarm for unusual API calls
aws cloudwatch put-metric-alarm \
  --alarm-name "CloudTrail-Modification-Alarm" \  # Alarm name
  --alarm-description "Alert when CloudTrail is modified" \  # Description
  --metric-name "UnusualAPIActivity" \  # Metric from filter above
  --namespace "Security" \  # Custom namespace
  --statistic "Sum" \  # Count occurrences
  --period 300 \  # 5-minute periods
  --threshold 1 \  # Trigger on any modification
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --evaluation-periods 1 \  # Single period
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:security-alerts"  # Send to security team
```

---

## Advanced Features

### Log File Integrity Validation

CloudTrail log file integrity validation provides a way to determine whether a CloudTrail log file was modified, deleted, or unchanged after CloudTrail delivered it. This feature is crucial for compliance and security auditing.

#### How It Works

When enabled, CloudTrail creates digest files that contain:
- **Hash values** of each log file using SHA-256 algorithm
- **Digital signatures** using SHA-256 with RSA algorithm
- **Metadata** about log file delivery

#### Key Components

| Component | Description | Purpose |
|-----------|-------------|---------|
| **Digest Files** | Files containing hashes of log files | Verify log file integrity |
| **Hash Values** | SHA-256 hash of each log file | Detect modifications |
| **Digital Signatures** | Cryptographic signatures | Ensure authenticity |

#### Validation Process

```bash
# Enable log file validation (when creating trail)
aws cloudtrail create-trail \
  --name "secure-trail" \
  --s3-bucket-name "audit-logs-bucket" \
  --enable-log-file-validation

# Validate log file integrity using AWS CLI
aws cloudtrail validate-logs \
  --trail-arn "arn:aws:cloudtrail:us-east-1:123456789012:trail/secure-trail" \
  --start-time 2023-11-01T00:00:00Z \
  --end-time 2023-11-02T00:00:00Z

# Check specific digest file
aws s3 cp s3://audit-logs-bucket/AWSLogs/123456789012/CloudTrail-Digest/us-east-1/2023/11/01/digest-file.json.gz ./
```

#### Best Practices

- **Always enable** log file validation for compliance environments
- **Regularly validate** log files using the AWS CLI
- **Monitor digest files** for any missing or corrupted entries
- **Store digest files securely** with appropriate S3 permissions

> **🔍 Exam Alert:** Log file integrity validation is essential for regulatory compliance and detecting unauthorized changes to CloudTrail logs. Remember that this feature creates digest files that can be used to verify log file integrity.

### Event History Analysis

```bash
# Look up events from the last 7 days
aws cloudtrail lookup-events \
  --start-time $(date -u -v-7d +%Y-%m-%dT%H:%M:%SZ) \  # 7 days ago (macOS format)
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \  # Current time
  --lookup-attributes AttributeKey=EventName,AttributeValue=StopInstances  # Find EC2 instance stops

# Filter by resource name
aws cloudtrail lookup-events \
  --start-time $(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ) \  # 1 day ago
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=my-s3-bucket  # Actions on specific bucket

# Filter by event source (AWS service)
aws cloudtrail lookup-events \
  --start-time $(date -u -v-24H +%Y-%m-%dT%H:%M:%SZ) \  # Last 24 hours
  --lookup-attributes AttributeKey=EventSource,AttributeValue=s3.amazonaws.com  # All S3 events
```

### Log Analysis with CloudWatch Logs Insights

```sql
# Find all failed API calls in the last hour
fields @timestamp, sourceIPAddress, userIdentity.type, eventName, errorCode, errorMessage
| filter errorCode exists
| sort @timestamp desc
| limit 100

# Analyze API calls by service
fields eventSource, eventName
| stats count(*) as apiCount by eventSource
| sort apiCount desc
| limit 20

# Track resource creation events
fields @timestamp, userIdentity.userName, eventName, sourceIPAddress
| filter eventName like /Create/
| sort @timestamp desc
| limit 50

# Monitor critical security group changes
fields @timestamp, userIdentity.userName, eventName, requestParameters.groupId
| filter eventSource = 'ec2.amazonaws.com' 
  and (eventName = 'AuthorizeSecurityGroupIngress' 
       or eventName = 'ModifySecurityGroupRules'
       or eventName = 'RevokeSecurityGroupEgress')
| sort @timestamp desc

# Analyze API call patterns for insights
fields @timestamp, eventName, sourceIPAddress, userAgent
| filter eventName exists
| stats count() by eventName
| sort count desc
| limit 20
```

### Data Events Configuration

```hcl
# Advanced event selector for specific resources
resource "aws_cloudtrail" "data_events" {
  name = "data-events-trail"  # Descriptive name
  
  # Advanced event selector (newer format)
  advanced_event_selector {
    name = "Log S3 data events for sensitive buckets"  # Selector description
    
    field_selector {
      field  = "category"
      equals = ["Data"]  # Data events only
    }
    
    field_selector {
      field  = "resources.type"
      equals = ["AWS::S3::Object"]  # S3 object operations
    }
    
    field_selector {
      field  = "resources.ARN"
      starts_with = [
        "arn:aws:s3:::sensitive-bucket/",  # Monitor specific prefix
        "arn:aws:s3:::financial-data/"  # Monitor another prefix
      ]
    }
  }
  
  advanced_event_selector {
    name = "Log Lambda executions"  # Separate selector for Lambda
    
    field_selector {
      field  = "category"
      equals = ["Data"]  # Data events only
    }
    
    field_selector {
      field  = "resources.type"
      equals = ["AWS::Lambda::Function"]  # Lambda function invocations
    }
  }
}
```

---

## Security and Compliance

### Best Practices for Security

```bash
# Enable MFA delete on CloudTrail S3 bucket
aws s3api put-bucket-versioning \
  --bucket "my-cloudtrail-logs-bucket" \  # Target bucket
  --versioning-configuration Status=Enabled,MFADelete=Enabled \  # Enable MFA for deletion
  --mfa "arn:aws:iam::123456789012:mfa/admin-user 123456"  # MFA device and token

# Configure bucket public access block
aws s3api put-public-access-block \
  --bucket "my-cloudtrail-logs-bucket" \  # Target bucket
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"  # Block all public access
```

### Compliance Monitoring

```hcl
# Config rule to ensure CloudTrail is enabled
resource "aws_config_config_rule" "cloudtrail_enabled" {
  name = "cloudtrail-enabled"  # Rule name

  source {
    owner             = "AWS"  # AWS-managed rule
    source_identifier = "CLOUD_TRAIL_ENABLED"  # Specific rule ID
  }

  depends_on = [aws_config_configuration_recorder.main]  # Ensure Config recorder exists
}

# Config rule for CloudTrail log file validation
resource "aws_config_config_rule" "cloudtrail_log_file_validation" {
  name = "cloudtrail-log-file-validation-enabled"  # Rule name

  source {
    owner             = "AWS"  # AWS-managed rule
    source_identifier = "CLOUD_TRAIL_LOG_FILE_VALIDATION_ENABLED"  # Specific rule ID
  }

  depends_on = [aws_config_configuration_recorder.main]  # Ensure Config recorder exists
}
```

---

## Monitoring and Alerting

### Key Metrics to Monitor

| Metric | Purpose | Alert Threshold |
|--------|---------|-----------------|
| **ConsoleLoginFailures** | Failed authentication attempts | > 5 failures in 5 minutes |
| **ApiCallVolume** | Unusual API activity | > 2 standard deviations from baseline |
| **UnauthorizedOperations** | Permission denied events | > 10 events in 5 minutes |
| **CloudTrailChanges** | Changes to audit configuration | Any modification |
| **NetworkACLChanges** | Security group modifications | Any change in production |

### CloudWatch Alarms for Security

```bash
# Create alarm for unauthorized API calls
aws cloudwatch put-metric-alarm \
  --alarm-name "Unauthorized-API-Calls" \  # Descriptive name
  --alarm-description "Alert on unauthorized API calls" \  # Purpose
  --metric-name "UnauthorizedOperations" \  # Metric from filter
  --namespace "Security" \  # Custom namespace
  --statistic "Sum" \  # Count occurrences
  --period 300 \  # 5-minute periods
  --threshold 10 \  # Alert on 10+ occurrences
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 1 \  # Single period
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:security-alerts"  # Send to security team

# Create alarm for CloudTrail configuration changes
aws cloudwatch put-metric-alarm \
  --alarm-name "CloudTrail-Changes" \  # Descriptive name
  --alarm-description "Alert on CloudTrail changes" \  # Purpose
  --metric-name "CloudTrailChanges" \  # Metric from filter
  --namespace "Security" \  # Custom namespace
  --statistic "Sum" \  # Count occurrences
  --period 60 \  # 1-minute periods (faster detection)
  --threshold 1 \  # Alert on any change
  --comparison-operator "GreaterThanOrEqualToThreshold" \
  --evaluation-periods 1 \  # Single period
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:security-alerts"  # Send to security team
```

### Automated Response with Lambda

```python
import json
import boto3
import gzip
import base64

def lambda_handler(event, context):
    # Decode CloudWatch Logs data from CloudTrail
    compressed_payload = base64.b64decode(event['awslogs']['data'])
    uncompressed_payload = gzip.decompress(compressed_payload)
    log_data = json.loads(uncompressed_payload)
    
    sns = boto3.client('sns')
    
    for log_event in log_data['logEvents']:
        message = json.loads(log_event['message'])
        
        # Check for high-risk events
        if is_high_risk_event(message):
            alert_message = format_security_alert(message)
            
            # Send SNS notification
            sns.publish(
                TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
                Message=alert_message,
                Subject='HIGH PRIORITY: Security Event Detected'
            )
    
    return {'statusCode': 200}

def is_high_risk_event(event):
    # Focus on CloudTrail and security configuration events
    high_risk_events = [
        'DeleteTrail',
        'StopLogging',
        'UpdateTrail',
        'DeleteBucket',
        'ModifyVpc',
        'AuthorizeSecurityGroupIngress',
        'ModifyInstanceAttribute'
    ]
    
    return (
        event.get('eventName') in high_risk_events or
        (event.get('eventSource') == 'cloudtrail.amazonaws.com' and 'error' in event)
    )

def format_security_alert(event):
    return f"""
    Security Event Detected:
    
    Event: {event.get('eventName', 'Unknown')}
    Source: {event.get('eventSource', 'Unknown')}
    User: {event.get('userIdentity', {}).get('userName', 'Unknown')}
    Source IP: {event.get('sourceIPAddress', 'Unknown')}
    Time: {event.get('eventTime', 'Unknown')}
    Region: {event.get('awsRegion', 'Unknown')}
    Error: {event.get('errorCode', 'None')}
    
    Please investigate immediately.
    """
```

---

## Troubleshooting

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Trail not logging** | No new log files in S3 | Check trail status, S3 bucket policy, ensure trail is enabled |
| **Missing events** | Expected events not appearing | Verify trail configuration, check if events are data events |
| **High costs** | Unexpected CloudTrail charges | Review data events configuration, optimize event selectors |
| **Permission errors** | Access denied errors | Verify IAM policies for CloudTrail service |
| **Log file validation failures** | Integrity check failures | Check for manual log file modifications |

### Debugging Steps

1. **Check Trail Status**
   ```bash
   # Verify if trail is currently logging
   aws cloudtrail get-trail-status --name "organization-trail"  # Shows logging status
   ```

2. **Verify Logging Status**
   ```bash
   # Check trail configuration
   aws cloudtrail describe-trails --trail-name-list "organization-trail"  # Shows trail settings
   ```

3. **Test Event Delivery**
   ```bash
   # Make a test API call and check if it appears
   aws s3 list-buckets  # Simple API call that should be logged
   
   # Check recent events
   aws cloudtrail lookup-events \
     --start-time $(date -u -v-5m +%Y-%m-%dT%H:%M:%SZ) \  # Last 5 minutes
     --lookup-attributes AttributeKey=EventName,AttributeValue=ListBuckets  # Look for our test call
   ```

4. **Validate Log File Integrity**
   ```bash
   # Check log file integrity
   aws cloudtrail validate-logs \
     --trail-arn "arn:aws:cloudtrail:us-east-1:123456789012:trail/organization-trail" \  # Trail ARN
     --start-time $(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ)  # Check last day's logs
   ```

---

## Cost Optimization

### Pricing Structure
- **Management Events**: First copy free per region, additional copies $2.00 per 100,000 events
- **Data Events**: $0.10 per 100,000 events (S3), $0.20 per 100,000 events (Lambda)
- **Insights Events**: $0.35 per 100,000 events analyzed
- **S3 Storage**: Standard S3 pricing for log files
- **CloudWatch Logs**: $0.50 per GB ingested, $0.03 per GB stored

### Cost Optimization Strategies

1. **Right-size Data Events**
   ```bash
   # Monitor specific buckets instead of all
   --data-resources Type=AWS::S3::Object,Values=arn:aws:s3:::critical-bucket/*  # Only critical buckets
   
   # Instead of monitoring all Lambda functions
   --data-resources Type=AWS::Lambda::Function,Values=arn:aws:lambda:us-east-1:123456789012:function:critical-function  # Only critical functions
   ```

2. **Optimize Log Retention**
   ```hcl
   # Set appropriate CloudWatch Logs retention
   resource "aws_cloudwatch_log_group" "cloudtrail" {
     retention_in_days = 30  # Instead of indefinite retention
   }
   
   # Use S3 lifecycle policies for log files
   resource "aws_s3_bucket_lifecycle_configuration" "cloudtrail" {
     bucket = aws_s3_bucket.cloudtrail.id
     
     rule {
       id     = "cloudtrail_logs"
       status = "Enabled"
       
       transition {
         days          = 30  # After 30 days
         storage_class = "STANDARD_IA"  # Move to infrequent access
       }
       
       transition {
         days          = 90  # After 90 days
         storage_class = "GLACIER"  # Move to Glacier
       }
       
       transition {
         days          = 365  # After a year
         storage_class = "DEEP_ARCHIVE"  # Move to Deep Archive
       }
     }
   }
   ```

3. **Use Event Selectors Efficiently**
   ```hcl
   # Target specific resources instead of broad patterns
   advanced_event_selector {
     field_selector {
       field = "resources.ARN"
       starts_with = ["arn:aws:s3:::prod-", "arn:aws:s3:::finance-"]  # Only specific prefixes
       # Instead of ["arn:aws:s3:::*"]  # All buckets (expensive)
     }
   }
   ```

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Trails per region** | 5 | Per AWS account |
| **Event selectors per trail** | 5 | Advanced event selectors |
| **Data resources per selector** | 250 | S3 buckets, Lambda functions, etc. |
| **Insights selectors per trail** | 2 | ApiCallRateInsight, etc. |
| **Event history** | 90 days | Free tier, no trail required |
| **Log file size** | Up to 50 MB | Before compression |
| **API rate limits** | Varies by operation | LookupEvents: 2 RPS per account |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding CloudTrail event types and their use cases
> - Configuring trails for compliance and security requirements
> - Integrating CloudTrail with CloudWatch for monitoring
> - Analyzing CloudTrail logs for security incidents
> - Cost optimization strategies for data events

### Key Exam Scenarios

1. **Security Incident Response**: Use CloudTrail to investigate unauthorized access
2. **Compliance Auditing**: Configure trails for regulatory requirements
3. **Cost Management**: Optimize data events to control CloudTrail costs
4. **Real-time Monitoring**: Integrate with CloudWatch for immediate alerts
5. **Multi-Account Setup**: Configure organization trails for centralized logging

---

## Quick Reference

### Essential Commands
```bash
# Create trail
aws cloudtrail create-trail --name "my-trail" --s3-bucket-name "logs-bucket"  # Basic trail setup

# Start logging
aws cloudtrail start-logging --name "my-trail"  # Enable trail

# Check status
aws cloudtrail get-trail-status --name "my-trail"  # Verify logging status

# Look up events
aws cloudtrail lookup-events --start-time $(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ)  # Last day's events
```

### Common Event Names
```bash
# EC2 Events (Infrastructure)
RunInstances, TerminateInstances, CreateSecurityGroup  # VM lifecycle and security

# S3 Events (Storage)
CreateBucket, DeleteBucket, PutBucketPolicy  # Bucket management

# RDS Events (Database)
CreateDBInstance, ModifyDBInstance, DeleteDBInstance  # Database operations

# Console Events (Access)
ConsoleLogin, AssumeRole, GetSessionToken  # Authentication and session management
```

---

## References

- [AWS CloudTrail User Guide](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/)
- [CloudTrail Event Reference](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference.html)
- [CloudTrail Log File Examples](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-log-file-examples.html)
- [CloudTrail Security Best Practices](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/best-practices-security.html)

---

**Previous:** [← CloudWatch Alarms](03-cloudwatch-alarms.md) | **Next:** [AWS Config →](05-aws-config.md)