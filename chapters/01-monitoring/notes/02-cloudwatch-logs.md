# CloudWatch Logs

## Overview

Amazon CloudWatch Logs is a service that enables you to centralize logs from all your systems, applications, and AWS services. It provides storage, monitoring, and access to log data, allowing you to troubleshoot issues, detect anomalies, and understand application behavior.

**Regional Service**: CloudWatch Logs is a regional service. Log groups and their contents exist only in the region where they were created. For multi-region applications, you need to collect and query logs in each region separately or implement custom solutions for centralized logging.

## Exam Relevance

This module covers these Domain 1 tasks:
- Implementing monitoring, logging, and remediation strategies
- Analyzing log data for troubleshooting and identifying operational issues
- Configuring logs collection from AWS services and applications
- Creating metric filters and subscription filters

---

## Key Concepts

### Log Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Log Event** | Single log entry/record with timestamp and message | Basic unit of log data | Application error message with timestamp |
| **Log Stream** | Sequence of log events from same source | Organizes logs by source | Logs from a specific EC2 instance |
| **Log Group** | Container for log streams with same retention and permissions | Organizes logs by application/component | All logs for a web application |
| **Metric Filter** | Pattern that extracts data from logs to create metrics | Enables metrics from logs | Count of HTTP 500 errors in application logs |
| **Subscription Filter** | Pattern that delivers matching log events to destinations | Enables real-time processing | Sending security logs to Lambda for analysis |
| **Retention Period** | How long logs are kept before automatic deletion | Controls storage costs | Setting 90-day retention for compliance logs |

#### Deep Dive into Log Concepts

- **Log Event**: Individual log record containing timestamp (UTC), raw message, event metadata (max size: 256 KB)
- **Log Stream**: Sequence of events from same source with chronological order
- **Log Group**: Organizational unit controlling retention, permissions, and encryption

> **🔍 Exam Alert:** Log groups contain multiple log streams, which contain multiple log events.

### Log Sources and Integration

| AWS Service | Log Integration | Common Use Cases |
|-------------|-----------------|------------------|
| **EC2** | CloudWatch Agent | Application logs, system logs |
| **Lambda** | Automatic logging | Function execution logs |
| **ECS/Fargate** | awslogs driver | Container application logs |
| **API Gateway** | Access/execution logs | API request/response logging |
| **VPC** | Flow Logs | Network traffic analysis |
| **CloudTrail** | Management events | API call auditing |
| **RDS** | Error/slow query logs | Database troubleshooting |

---

## Service Configuration

### CloudWatch Logs Agent

#### Unified CloudWatch Agent Configuration
```json
{
  "agent": {
    "run_as_user": "cwagent"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/apache2/access.log",
            "log_group_name": "apache-access-logs",
            "log_stream_name": "{instance_id}",
            "timestamp_format": "%d/%b/%Y:%H:%M:%S %z"
          }
        ]
      }
    }
  }
}
```

#### Legacy CloudWatch Logs Agent
```ini
[general]
state_file = /var/awslogs/state/agent-state

[/var/log/messages]
file = /var/log/messages
log_group_name = system-logs
log_stream_name = {instance_id}
datetime_format = %b %d %H:%M:%S
```

---

## Metric Filters and Patterns

### Filter Pattern Types

#### 1. Simple Text Patterns
```
# Match logs containing "ERROR"
ERROR

# Match logs containing either ERROR or WARN
?ERROR ?WARN

# Exclude INFO logs
-INFO
```

#### 2. Field-Based Patterns (Space-Delimited)
```
# Apache/NGINX access log pattern
[ip, user, username, timestamp, request, status_code, size, referrer, agent]

# Extract only 5xx errors
[ip, user, username, timestamp, request, status_code=5*, size, referrer, agent]
```

#### 3. JSON Patterns
```
# Match JSON logs with specific field values
{ $.level = "ERROR" }

# Complex JSON pattern
{ $.level = "ERROR" && $.service = "payment" && $.amount > 100 }
```

### Common Metric Filter Examples

#### HTTP Error Monitoring
```bash
# Pattern for HTTP 4xx errors
[ip, user, username, timestamp, request, status_code=4*, size]

# Pattern for HTTP 5xx errors  
[ip, user, username, timestamp, request, status_code=5*, size]
```

#### Application Error Tracking
```bash
# JSON application logs
{ $.level = "ERROR" }

# Lambda function errors
ERROR

# Custom application patterns
[timestamp, level="ERROR", service, message]
```

---

## Implementation Examples

### AWS CLI Operations

```bash
# Create log group with retention
aws logs create-log-group --log-group-name "web-app-logs"
aws logs put-retention-policy --log-group-name "web-app-logs" --retention-in-days 7

# Create metric filter for errors
aws logs put-metric-filter \
  --log-group-name "web-app-logs" \
  --filter-name "ErrorCount" \
  --filter-pattern "ERROR" \
  --metric-transformations \
    metricName=ApplicationErrors,metricNamespace=WebApp,metricValue=1

# Query logs
aws logs start-query \
  --log-group-name "web-app-logs" \
  --start-time 1634567890 \
  --end-time 1634571490 \
  --query-string 'fields @timestamp, @message | filter @message like /ERROR/'
```

### Terraform Configuration

```hcl
# Log group with encryption
resource "aws_cloudwatch_log_group" "app_logs" {
  name              = "/aws/application/web-app"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.logs.arn

  tags = {
    Environment = "production"
    Application = "web-app"
  }
}

# Metric filter for HTTP errors
resource "aws_cloudwatch_log_metric_filter" "http_errors" {
  name           = "http-4xx-errors"
  log_group_name = aws_cloudwatch_log_group.app_logs.name
  pattern        = "[ip, user, username, timestamp, request, status_code=4*, size]"

  metric_transformation {
    name      = "HTTP4xxErrors"
    namespace = "WebApp/HTTP"
    value     = "1"
  }
}

# Subscription filter to Lambda
resource "aws_cloudwatch_log_subscription_filter" "error_processor" {
  name            = "error-processor"
  log_group_name  = aws_cloudwatch_log_group.app_logs.name
  filter_pattern  = "ERROR"
  destination_arn = aws_lambda_function.log_processor.arn
}
```

---

## CloudWatch Logs Insights

### Query Examples

#### Basic Queries
```sql
# Recent errors
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20

# Count errors by hour
stats count() by bin(5m)
| filter @message like /ERROR/
```

#### Advanced Queries
```sql
# Parse JSON logs
fields @timestamp, level, service, message
| filter level = "ERROR"
| stats count() by service

# Lambda performance analysis
filter @type = "REPORT"
| fields @requestId, @duration, @billedDuration, @maxMemoryUsed
| sort @duration desc
| limit 10
```

---

## Subscription Filters

### Destinations and Use Cases

| Destination | Use Case | Configuration |
|-------------|----------|---------------|
| **Lambda** | Real-time processing, alerting | Function ARN + permission |
| **Kinesis Data Streams** | High-throughput processing | Stream ARN + role |
| **Kinesis Data Firehose** | S3/Redshift delivery | Firehose ARN + role |

### Lambda Subscription Example

```python
import json
import gzip
import base64

def lambda_handler(event, context):
    # Decode CloudWatch Logs data
    compressed_payload = base64.b64decode(event['awslogs']['data'])
    uncompressed_payload = gzip.decompress(compressed_payload)
    log_data = json.loads(uncompressed_payload)
    
    # Process log events
    for log_event in log_data['logEvents']:
        print(f"Log: {log_event['message']}")
        
        # Custom processing logic here
        if 'ERROR' in log_event['message']:
            # Send alert, write to DynamoDB, etc.
            pass
    
    return {'statusCode': 200}
```

---

## Best Practices

### Organization and Naming
- Use consistent log group naming: `/aws/service/application`
- Separate environments: `/aws/lambda/prod-function` vs `/aws/lambda/dev-function`
- Use descriptive names for metric filters

### Performance and Cost
- Set appropriate retention periods (1 day to 10 years)
- Use log levels efficiently (ERROR, WARN, INFO, DEBUG)
- Consider log sampling for high-volume applications
- Export old logs to S3 for archival

### Security
- Enable encryption at rest with KMS
- Use IAM policies for access control
- Audit log access with CloudTrail

---

## Monitoring and Alerting

### Key Metrics to Monitor

| Metric | Purpose | Threshold Example |
|--------|---------|-------------------|
| **IncomingLogEvents** | Monitor log ingestion | > 1000 events/min |
| **IncomingBytes** | Track data volume | > 100 MB/hour |
| **ForwardedLogEvents** | Subscription filter delivery | < expected rate |
| **DeliveryErrors** | Failed log delivery | > 0 errors |

### CloudWatch Alarms

```bash
# Create alarm for high error rate
aws cloudwatch put-metric-alarm \
  --alarm-name "High-Application-Errors" \
  --alarm-description "Alert when error rate is high" \
  --metric-name "ApplicationErrors" \
  --namespace "WebApp" \
  --statistic "Sum" \
  --period 300 \
  --threshold 10 \
  --comparison-operator "GreaterThanThreshold" \
  --evaluation-periods 2
```

---

## Troubleshooting

### Common Issues

| Issue | Symptoms | Solutions |
|-------|----------|-----------|
| **Logs not appearing** | Missing log data in console | Check agent status, IAM permissions, network connectivity |
| **Throttling errors** | API rate limit exceeded | Implement exponential backoff, batch requests |
| **High costs** | Unexpected billing | Review retention policies, optimize log levels |
| **Metric filters not working** | No metrics generated | Verify filter patterns, test with sample logs |
| **Subscription filter failures** | Delivery errors | Check destination permissions, monitor error metrics |

### Debugging Steps

1. **Verify Agent Status**
   ```bash
   # Check agent status
   sudo systemctl status amazon-cloudwatch-agent
   
   # View agent logs
   sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
   ```

2. **Test Filter Patterns**
   ```bash
   # Use Logs Insights to test patterns
   aws logs start-query \
     --log-group-name "test-logs" \
     --start-time $(date -d '1 hour ago' +%s) \
     --end-time $(date +%s) \
     --query-string 'fields @message | filter @message like /your-pattern/'
   ```

---

## Cost Optimization

### Pricing Structure
- **Ingestion**: $0.50 per GB
- **Storage**: $0.03 per GB-month
- **Insights Queries**: $0.005 per GB scanned
- **Data Transfer**: Standard AWS data transfer rates

### Cost Optimization Strategies

1. **Right-size retention periods**
   - Development: 1-7 days
   - Staging: 7-30 days
   - Production: 30-365 days
   - Compliance: 1-10 years

2. **Optimize log levels**
   ```json
   {
     "production": ["ERROR", "WARN"],
     "staging": ["ERROR", "WARN", "INFO"],
     "development": ["ERROR", "WARN", "INFO", "DEBUG"]
   }
   ```

3. **Use S3 export for archival**
   ```bash
   # Export logs to S3 for long-term storage
   aws logs create-export-task \
     --log-group-name "old-logs" \
     --from 1609459200000 \
     --to 1612137600000 \
     --destination "my-log-archive-bucket"
   ```

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Log event size** | 256 KB | Maximum per event |
| **Batch size** | 1 MB | PutLogEvents API |
| **API rate** | 5 requests/sec/stream | Per log stream |
| **Log groups** | 1,000,000 | Per region per account |
| **Metric filters** | 100 | Per log group |
| **Subscription filters** | 2 | Per log group |
| **Query concurrency** | 20 | Concurrent Insights queries |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding metric filter patterns and transformations
> - Knowing subscription filter destinations and use cases
> - Log retention policies and cost implications
> - Troubleshooting log delivery issues
> - CloudWatch Logs Insights query syntax basics

### Key Exam Scenarios

1. **Application Error Monitoring**: Create metric filters to track application errors
2. **Real-time Log Processing**: Use subscription filters with Lambda
3. **Cost Optimization**: Set appropriate retention periods
4. **Security Compliance**: Enable encryption and proper IAM policies
5. **Troubleshooting**: Analyze logs with Insights queries

---

## Quick Reference

### Essential Commands
```bash
# Create log group
aws logs create-log-group --log-group-name "app-logs"

# Set retention
aws logs put-retention-policy --log-group-name "app-logs" --retention-in-days 30

# Create metric filter
aws logs put-metric-filter --log-group-name "app-logs" --filter-name "errors" --filter-pattern "ERROR" --metric-transformations metricName=Errors,metricNamespace=App,metricValue=1

# Query logs
aws logs start-query --log-group-name "app-logs" --start-time 1234567890 --end-time 1234571490 --query-string 'fields @message | filter @message like /ERROR/'
```

### Common Patterns
```bash
# HTTP status codes
[ip, user, username, timestamp, request, status_code, size]

# JSON error logs
{ $.level = "ERROR" }

# Lambda errors
ERROR

# Application logs
[timestamp, level, service, message]
```

---

## References
- [AWS CloudWatch Logs User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/)
- [CloudWatch Logs Insights Query Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)
- [Filter and Pattern Syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html)
- [CloudWatch Logs API Reference](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/)

---

**Previous:** [← CloudWatch Metrics](01-cloudwatch-metrics.md) | **Next:** [CloudWatch Alarms →](03-cloudwatch-alarms.md)