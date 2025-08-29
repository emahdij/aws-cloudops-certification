# EventBridge

## Overview

Amazon EventBridge is a serverless event bus service that makes it easy to connect applications using data from your own apps, SaaS applications, and AWS services. It acts as a central hub for routing events between different components of your architecture, enabling event-driven applications and microservices.

**Regional Service**: EventBridge is a regional service, but it can receive events from and send events to other regions through cross-region replication. Each region maintains its own event buses, rules, and targets. For multi-region architectures, you'll need to configure EventBridge in each region or use cross-region event routing.

## Exam Relevance

This module covers these Domain 1 tasks:
- Implementing event-driven monitoring and automation
- Creating automated responses to AWS service events
- Integrating with third-party monitoring tools
- Building scalable notification and remediation workflows

---

## Key Concepts

### EventBridge Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Event** | JSON message describing a change in state | Core unit of event-driven architecture | EC2 instance state change notification |
| **Event Bus** | Pipeline that receives and routes events | Central routing mechanism | Default bus for AWS events, custom bus for applications |
| **Rule** | Pattern that matches events and routes them to targets | Defines what events trigger which actions | Route EC2 state changes to SNS topic |
| **Target** | Destination where matched events are sent | Enables automated responses | Lambda function, SNS topic, SQS queue |
| **Event Pattern** | JSON structure that filters which events match a rule | Controls precision of event routing | Match only "running" to "stopped" state changes |
| **Source** | System that generates events | Identifies origin of events | aws.ec2, myapp.orders, github.com |
| **Archive** | Long-term storage for events | Compliance and replay capabilities | Store security events for 7 years |

#### Deep Dive into Event Structure

Every EventBridge event has a standardized JSON structure:
```json
{
  "version": "0",
  "id": "6a7e8feb-b491-4cf7-a9f1-bf3703467718",
  "detail-type": "EC2 Instance State-change Notification",
  "source": "aws.ec2",
  "account": "123456789012",
  "time": "2023-11-15T22:00:00Z",
  "region": "us-west-2",
  "detail": {
    "instance-id": "i-1234567890abcdef0",
    "state": "terminated"
  }
}
```

> **🔍 Exam Alert:** Understanding event structure is crucial for writing effective event patterns. The exam often tests your ability to create patterns that match specific event attributes.

### EventBridge vs. CloudWatch Events

EventBridge is the evolution of CloudWatch Events with additional capabilities:

| Feature | CloudWatch Events | EventBridge |
|---------|------------------|-------------|
| **Custom Event Buses** | No | Yes |
| **Third-party Sources** | Limited | Extensive (Salesforce, GitHub, etc.) |
| **Schema Registry** | No | Yes |
| **Cross-region Replication** | No | Yes |
| **Archive and Replay** | No | Yes |
| **Event Discovery** | No | Yes |

---

## Service Configuration

### Event Bus Types

#### Default Event Bus
- Automatically available in every AWS account
- Receives events from AWS services
- Cannot be deleted
- Used for most AWS service integrations

#### Custom Event Bus
- Created for specific applications or environments
- Provides isolation between different event sources
- Can be shared across accounts
- Useful for microservices architectures

### Basic EventBridge Setup

```bash
# Create a custom event bus
aws events create-event-bus --name "production-app-events"

# Create a rule on the default bus
aws events put-rule \
  --name "ec2-state-change" \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"],
    "detail": {
      "state": ["stopped", "terminated"]
    }
  }' \
  --state ENABLED

# Add a target to the rule
aws events put-targets \
  --rule "ec2-state-change" \
  --targets "Id"="1","Arn"="arn:aws:sns:us-east-1:123456789012:ec2-alerts"
```

---

## Implementation Examples

### Terraform Configuration

```hcl
# Custom event bus for application events
resource "aws_cloudwatch_event_bus" "app_events" {
  name = "production-app-events"
  
  tags = {
    Environment = "production"
    Purpose     = "application-events"
  }
}

# Rule for EC2 instance state changes
resource "aws_cloudwatch_event_rule" "ec2_state_change" {
  name        = "ec2-instance-state-change"
  description = "Capture EC2 instance state changes"
  
  event_pattern = jsonencode({
    source      = ["aws.ec2"]
    detail-type = ["EC2 Instance State-change Notification"]
    detail = {
      state = ["stopped", "terminated", "running"]
    }
  })
  
  tags = {
    Purpose = "infrastructure-monitoring"
  }
}

# SNS target for notifications
resource "aws_cloudwatch_event_target" "sns_target" {
  rule      = aws_cloudwatch_event_rule.ec2_state_change.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.alerts.arn
}

# Lambda target for automated remediation
resource "aws_cloudwatch_event_target" "lambda_target" {
  rule      = aws_cloudwatch_event_rule.ec2_state_change.name
  target_id = "TriggerRemediation"
  arn       = aws_lambda_function.instance_handler.arn
}

# Rule for application errors from custom events
resource "aws_cloudwatch_event_rule" "app_errors" {
  name           = "application-error-handler"
  description    = "Handle critical application errors"
  event_bus_name = aws_cloudwatch_event_bus.app_events.name
  
  event_pattern = jsonencode({
    source      = ["myapp.orders"]
    detail-type = ["Order Processing Error"]
    detail = {
      severity = ["critical", "high"]
    }
  })
}
```

### Real-World Event Patterns

#### Security Events
```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"],
  "detail": {
    "severity": [7.0, 8.0, 8.1, 8.2, 8.3, 8.4, 8.5, 8.6, 8.7, 8.8, 8.9]
  }
}
```

#### Cost Management
```json
{
  "source": ["aws.budgets"],
  "detail-type": ["Budget Notification"],
  "detail": {
    "notificationType": ["ACTUAL"],
    "thresholdType": ["PERCENTAGE"]
  }
}
```

#### Auto Scaling Events
```json
{
  "source": ["aws.autoscaling"],
  "detail-type": ["EC2 Instance Launch Successful", "EC2 Instance Launch Unsuccessful"],
  "detail": {
    "StatusCode": ["Failed"]
  }
}
```

---

## Advanced Features

### Schema Registry

EventBridge can automatically discover and store schemas for your events:

```bash
# Enable schema discovery
aws events put-event-source-mapping \
  --function-name "my-event-processor" \
  --event-source-arn "arn:aws:events:us-east-1:123456789012:event-bus/production-app-events"

# List discovered schemas
aws schemas list-schemas --registry-name "discovered-schemas"
```

### Archive and Replay

Store events for compliance or debugging purposes:

```hcl
# Archive important events
resource "aws_cloudwatch_event_archive" "security_events" {
  name             = "security-events-archive"
  event_source_arn = aws_cloudwatch_event_bus.app_events.arn
  description      = "Archive security-related events"
  retention_days   = 2555  # 7 years for compliance
  
  event_pattern = jsonencode({
    source = ["aws.guardduty", "aws.securityhub"]
  })
}

# Replay archived events for testing
resource "aws_cloudwatch_event_replay" "test_replay" {
  name               = "security-events-replay"
  description        = "Replay security events for testing"
  event_source_arn   = aws_cloudwatch_event_archive.security_events.arn
  event_start_time   = "2023-11-01T00:00:00Z"
  event_end_time     = "2023-11-02T00:00:00Z"
  destination {
    arn = aws_cloudwatch_event_bus.app_events.arn
  }
}
```

### Cross-Region Event Routing

```hcl
# Rule in us-east-1 that sends events to us-west-2
resource "aws_cloudwatch_event_rule" "cross_region" {
  name        = "replicate-to-west"
  description = "Send critical events to backup region"
  
  event_pattern = jsonencode({
    source      = ["myapp.orders"]
    detail-type = ["Critical System Alert"]
  })
}

resource "aws_cloudwatch_event_target" "west_region" {
  rule      = aws_cloudwatch_event_rule.cross_region.name
  target_id = "SendToWestRegion"
  arn       = "arn:aws:events:us-west-2:123456789012:event-bus/backup-events"
  
  role_arn = aws_iam_role.cross_region_events.arn
}
```

---

## Scheduled Events

EventBridge supports scheduled events using cron expressions or rate expressions, making it ideal for automating recurring tasks.

### Schedule Expression Types

#### Rate Expressions
```bash
# Every 5 minutes
rate(5 minutes)

# Every hour
rate(1 hour)

# Every day
rate(1 day)
```

#### Cron Expressions
```bash
# Every day at 2 AM UTC
cron(0 2 * * ? *)

# Every weekday at 6 PM
cron(0 18 ? * MON-FRI *)

# First day of every month at midnight
cron(0 0 1 * ? *)

# Every 15 minutes during business hours (9 AM - 5 PM, Mon-Fri)
cron(0/15 9-17 ? * MON-FRI *)
```

### Common Scheduled Use Cases

#### Lambda Function Scheduling
```hcl
# Schedule Lambda function to run every night
resource "aws_cloudwatch_event_rule" "nightly_cleanup" {
  name                = "nightly-cleanup-task"
  description         = "Run cleanup Lambda every night at 2 AM"
  schedule_expression = "cron(0 2 * * ? *)"
}

resource "aws_cloudwatch_event_target" "cleanup_lambda" {
  rule      = aws_cloudwatch_event_rule.nightly_cleanup.name
  target_id = "CleanupTarget"
  arn       = aws_lambda_function.cleanup_function.arn
}
```

#### Infrastructure Scheduling
```hcl
# Start development instances in the morning
resource "aws_cloudwatch_event_rule" "start_dev_instances" {
  name                = "start-dev-instances"
  description         = "Start development instances at 8 AM weekdays"
  schedule_expression = "cron(0 8 ? * MON-FRI *)"
}

# Stop development instances in the evening
resource "aws_cloudwatch_event_rule" "stop_dev_instances" {
  name                = "stop-dev-instances"
  description         = "Stop development instances at 6 PM weekdays"
  schedule_expression = "cron(0 18 ? * MON-FRI *)"
}
```

#### Backup and Maintenance
```bash
# Create scheduled rule via CLI
aws events put-rule \
  --name "weekly-backup" \
  --schedule-expression "cron(0 2 ? * SUN *)" \
  --description "Weekly backup every Sunday at 2 AM"

# Add Lambda target for backup
aws events put-targets \
  --rule "weekly-backup" \
  --targets "Id"="1","Arn"="arn:aws:lambda:us-east-1:123456789012:function:backup-function"
```

> **🔍 Exam Alert:** EventBridge scheduled rules are the recommended way to invoke Lambda functions on a schedule. They're more cost-effective and manageable than running EC2 instances with cron jobs.

---

## Real-World Use Cases

### 1. Infrastructure Automation

**Scenario**: Automatically tag and configure new EC2 instances when they launch.

```python
# Lambda function triggered by EC2 launch events
import json
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Extract instance details from EventBridge event
    instance_id = event['detail']['instance-id']
    instance_type = event['detail']['instance-type']
    
    # Apply standardized tags
    tags = [
        {'Key': 'AutoTagged', 'Value': 'true'},
        {'Key': 'LaunchTime', 'Value': event['time']},
        {'Key': 'InstanceType', 'Value': instance_type},
        {'Key': 'Environment', 'Value': 'production'}
    ]
    
    try:
        ec2.create_tags(Resources=[instance_id], Tags=tags)
        
        # Configure monitoring based on instance type
        if instance_type.startswith('m5.large'):
            ec2.monitor_instances(InstanceIds=[instance_id])
            
        return {'statusCode': 200, 'message': f'Tagged instance {instance_id}'}
    except Exception as e:
        print(f'Error tagging instance {instance_id}: {str(e)}')
        return {'statusCode': 500, 'error': str(e)}
```

### 2. Security Response Automation

**Scenario**: Automatically isolate compromised instances detected by GuardDuty.

```python
# Lambda function for security response
import json
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')
    
    # Parse GuardDuty finding
    finding = event['detail']
    severity = finding['severity']
    
    if severity >= 7.0:  # High/Critical findings
        # Extract affected instance ID
        instance_id = finding['service']['resourceRole']['ebsVolumeDetails']['instanceId']
        
        # Create isolation security group
        response = ec2.create_security_group(
            GroupName=f'isolation-{instance_id}',
            Description='Isolation security group for compromised instance'
        )
        isolation_sg_id = response['GroupId']
        
        # Attach isolation security group to instance
        ec2.modify_instance_attribute(
            InstanceId=instance_id,
            Groups=[isolation_sg_id]
        )
        
        # Send notification
        sns = boto3.client('sns')
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
            Message=f'Instance {instance_id} has been isolated due to security finding',
            Subject='Security Alert: Instance Isolated'
        )
        
    return {'statusCode': 200}
```

### 3. Application Health Monitoring

**Scenario**: Monitor custom application events and trigger scaling or alerts.

```json
{
  "version": "0",
  "id": "custom-app-event",
  "detail-type": "Order Processing Metrics",
  "source": "myapp.orders",
  "time": "2023-11-15T10:30:00Z",
  "detail": {
    "queue_depth": 1500,
    "processing_rate": 50,
    "error_rate": 0.02,
    "region": "us-east-1"
  }
}
```

### 4. Cost Optimization Automation

**Scenario**: Automatically stop development instances outside business hours.

```hcl
# Schedule rule for stopping dev instances
resource "aws_cloudwatch_event_rule" "stop_dev_instances" {
  name                = "stop-dev-instances"
  description         = "Stop development instances at 6 PM"
  schedule_expression = "cron(0 18 ? * MON-FRI *)"  # 6 PM weekdays
}

resource "aws_cloudwatch_event_target" "stop_instances_lambda" {
  rule      = aws_cloudwatch_event_rule.stop_dev_instances.name
  target_id = "StopDevInstances"
  arn       = aws_lambda_function.instance_scheduler.arn
}
```

---

## Best Practices

### Event Design
- Use consistent event structure across your applications
- Include all necessary context in the event detail
- Use descriptive event sources and detail types
- Follow JSON schema for event validation

### Rule Configuration
- Create specific event patterns to avoid unnecessary Lambda invocations
- Use multiple small rules rather than complex patterns
- Test event patterns thoroughly before production deployment
- Monitor rule metrics to ensure proper event matching

### Target Configuration
- Use dead letter queues for failed event processing
- Implement retry logic in Lambda functions
- Consider using SQS for buffering high-volume events
- Set appropriate timeout values for targets

### Security
- Use IAM roles with minimal required permissions
- Enable CloudTrail logging for EventBridge API calls
- Encrypt event buses with customer-managed KMS keys
- Regularly review cross-account event bus policies

---

## Monitoring and Troubleshooting

### Key Metrics to Monitor

| Metric | Purpose | Alert Threshold |
|--------|---------|-----------------|
| **SuccessfulInvocations** | Successful rule executions | Track for baseline |
| **FailedInvocations** | Failed rule executions | > 0 failures |
| **MatchedEvents** | Events matching rules | Monitor for patterns |
| **ThrottledRules** | Rules hitting rate limits | > 0 throttled |

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Events not matching rules** | Expected targets not triggered | Review event pattern syntax, check event structure |
| **High event volume** | Throttling errors | Implement batching, use SQS as buffer |
| **Target failures** | Dead letter queue messages | Check target permissions, implement retry logic |
| **Cross-account issues** | Permission denied errors | Verify resource policies and IAM roles |

### Debugging Steps

```bash
# Check if events are being received
aws events test-event-pattern \
  --event-pattern '{
    "source": ["aws.ec2"],
    "detail-type": ["EC2 Instance State-change Notification"]
  }' \
  --event '{
    "source": "aws.ec2",
    "detail-type": "EC2 Instance State-change Notification",
    "detail": {"state": "running"}
  }'

# Monitor rule metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Events \
  --metric-name MatchedEvents \
  --dimensions Name=RuleName,Value=ec2-state-change \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 \
  --statistics Sum
```

---

## Cost Optimization

### Pricing Structure
- **Custom Event Bus**: $1.00 per million events published
- **Rules**: $1.00 per million events processed by rules
- **Cross-region Replication**: $0.01 per GB transferred
- **Archive**: $0.10 per GB-month stored
- **Schema Registry**: $1.00 per million events processed

### Cost Optimization Strategies

1. **Optimize Event Patterns**
   - Use specific patterns to reduce unnecessary rule evaluations
   - Combine related events into single rules where possible

2. **Use Batching**
   ```python
   # Batch events before publishing
   events = []
   for order in orders:
       events.append({
           'Source': 'myapp.orders',
           'DetailType': 'Order Processed',
           'Detail': json.dumps(order)
       })
   
   # Send in batches of 10
   for i in range(0, len(events), 10):
       batch = events[i:i+10]
       eventbridge.put_events(Entries=batch)
   ```

3. **Archive Lifecycle Management**
   ```hcl
   resource "aws_cloudwatch_event_archive" "short_term" {
     name           = "short-term-events"
     retention_days = 30  # Instead of default indefinite storage
   }
   ```

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Event buses per account** | 100 | Per region |
| **Rules per event bus** | 300 | Per event bus |
| **Targets per rule** | 5 | Maximum destinations |
| **Event size** | 256 KB | Maximum event payload |
| **API rate limits** | 2,400 requests/second | PutEvents API |
| **Archive retention** | Indefinite | Unless specified |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding event patterns and how they match events
> - Knowing when to use EventBridge vs. SNS vs. SQS
> - Configuring cross-region event routing
> - Implementing automated responses to AWS service events
> - Troubleshooting event delivery issues

### Key Exam Scenarios

1. **Automated Infrastructure Response**: React to EC2, RDS, or Auto Scaling events
2. **Security Automation**: Respond to GuardDuty or Security Hub findings
3. **Cost Management**: Automate responses to budget alerts
4. **Application Integration**: Connect microservices through events
5. **Compliance**: Archive events for audit and replay capabilities

---

## Quick Reference

### Essential Commands
```bash
# Create rule
aws events put-rule --name "my-rule" --event-pattern '{"source":["aws.ec2"]}'

# Add target
aws events put-targets --rule "my-rule" --targets "Id"="1","Arn"="arn:aws:sns:us-east-1:123456789012:alerts"

# Test pattern
aws events test-event-pattern --event-pattern '{"source":["aws.ec2"]}' --event '{"source":"aws.ec2"}'

# List rules
aws events list-rules
```

### Common Event Sources
```bash
# AWS Services
aws.ec2, aws.rds, aws.autoscaling, aws.guardduty, aws.budgets

# Custom Applications  
myapp.orders, myapp.users, myapp.payments

# Third-party
github.com, salesforce.com, zendesk.com
```

---

## References

- [Amazon EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/latest/userguide/)
- [EventBridge Event Patterns](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html)
- [EventBridge Schema Registry](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-schema.html)
- [EventBridge API Reference](https://docs.aws.amazon.com/eventbridge/latest/APIReference/)

---

**Previous:** [← AWS Config](05-aws-config.md) | **Next:** [Systems Manager →](07-systems-manager.md)