# CloudWatch Alarms

## Overview

Amazon CloudWatch Alarms monitor your metrics and trigger actions when thresholds are breached. They allow you to respond automatically to changes in your AWS resources and applications, enabling proactive management and reducing mean time to resolution.

**Regional Service**: CloudWatch Alarms are regional resources that monitor metrics within the same region. Alarms cannot directly monitor metrics from other regions. For multi-region monitoring, you need to create separate alarms in each region or implement cross-region monitoring solutions.

## Exam Relevance

This module covers these Domain 1 tasks:
- Creating and maintaining alarms to initiate automated actions
- Implementing monitoring, logging, and remediation strategies
- Configuring notifications based on metric thresholds
- Using CloudWatch alarms for horizontal and vertical scaling

---

## Key Concepts

### Alarm Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Alarm** | Watches a single metric over a specified time period | Core monitoring component | CPU utilization > 80% for 5 minutes |
| **Alarm State** | Current condition of an alarm (OK, ALARM, INSUFFICIENT_DATA) | Indicates system health | An alarm in "ALARM" state indicates a problem |
| **Threshold** | Value that triggers a state change when crossed | Defines normal vs abnormal | 95% disk utilization as critical threshold |
| **Evaluation Period** | Number of periods to evaluate before changing state | Prevents false alarms | 3 consecutive periods above threshold |
| **Period** | Length of time to evaluate the metric | Determines sensitivity | 60 seconds for critical systems |
| **Action** | Response triggered when alarm state changes | Enables automated remediation | Auto scaling, SNS notification, or Lambda |
| **Composite Alarm** | Uses logical operators to combine multiple alarms | Reduces alert noise | Trigger only when both CPU AND memory are high |

#### Deep Dive into Alarm States

- **OK**: Metric is within defined threshold
- **ALARM**: Metric has crossed the threshold for the specified periods
- **INSUFFICIENT_DATA**: Not enough data to evaluate (common during startup or for new alarms)

> **🔍 Exam Alert:** The CloudOps exam often tests your understanding of how alarm states change based on different evaluation periods and datapoint settings.

### Alarm Actions

CloudWatch alarms can trigger various actions when state changes:

- **SNS Notifications**: Send alerts via email, SMS, or webhooks
- **Auto Scaling**: Add or remove EC2 instances
- **EC2 Actions**: Stop, terminate, reboot, or recover instances
- **Systems Manager**: Run automation documents
- **Lambda Functions**: Execute custom logic

### Advanced Configuration Options

| Setting | Description | Best Practice |
|---------|-------------|---------------|
| **Treat missing data as** | How to handle periods with no data | Use "notBreaching" for metrics that should exist continuously (like CPU), "missing" for sporadic metrics (like error counts) |
| **Datapoints to alarm** | M out of N data points that must breach | Use "3 out of 3" for immediate issues, "3 out of 5" for persistent but intermittent issues |
| **Alarm preview** | Visual representation of how the alarm will behave | Always review this to validate alarm behavior before saving |
| **Anomaly detection** | Machine learning-based thresholds | Use for metrics with variable patterns like traffic or usage |

---

## Service Configuration

### Basic Alarm Setup

Here's the foundational structure of a CloudWatch alarm:

1. **Target metric** - What you're monitoring
2. **Threshold** - When to trigger the alarm
3. **Evaluation criteria** - How many data points must breach the threshold
4. **Actions** - What happens when the alarm state changes

### Sample Use Cases

Here are common CloudWatch alarm scenarios:

1. **Infrastructure Monitoring**
   ```
   CPU utilization > 80% for 5 minutes → Scale up EC2 instances
   Memory utilization > 85% for 10 minutes → Send alert to operations team
   Disk space < 10% for 5 minutes → Execute cleanup script via SSM
   ```

2. **Application Performance**
   ```
   API latency > 200ms for 3 minutes → Route traffic to backup endpoint
   HTTP 5xx error rate > 1% for 2 minutes → Roll back deployment
   Queue depth > 1000 messages for 15 minutes → Add more consumers
   ```

3. **Cost Control**
   ```
   Daily estimated charges > $500 → Send notification to finance team
   Underutilized resources < 10% for 7 days → Flag for potential termination
   ```

---

## Implementation Examples

### AWS CLI Operations

```bash
# Create a basic alarm for high CPU utilization
aws cloudwatch put-metric-alarm \
  --alarm-name "high-cpu-alarm" \
  --alarm-description "Alarm when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --datapoints-to-alarm 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:alert-topic

# Create an alarm based on a metric filter from CloudWatch Logs
aws cloudwatch put-metric-alarm \
  --alarm-name "http-500-errors-alarm" \
  --alarm-description "Alarm when HTTP 500 errors exceed threshold" \
  --metric-name HTTP5xxErrorCount \
  --namespace WebServer/Errors \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 5 \
  --datapoints-to-alarm 3 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:critical-alerts

# Create anomaly detection alarm
aws cloudwatch put-anomaly-detector \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --stat Average

aws cloudwatch put-metric-alarm \
  --alarm-name "cpu-anomaly-alarm" \
  --comparison-operator GreaterThanUpperThreshold \
  --evaluation-periods 3 \
  --metrics '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/EC2",
          "MetricName": "CPUUtilization",
          "Dimensions": [{"Name": "InstanceId", "Value": "i-1234567890abcdef0"}]
        },
        "Period": 300,
        "Stat": "Average"
      }
    },
    {
      "Id": "ad1",
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)"
    }
  ]' \
  --threshold-metric-id ad1

# Describe existing alarms
aws cloudwatch describe-alarms --state-value ALARM

# Delete an alarm
aws cloudwatch delete-alarms --alarm-names "high-cpu-alarm"

# Set alarm state (for testing)
aws cloudwatch set-alarm-state \
  --alarm-name "high-cpu-alarm" \
  --state-value ALARM \
  --state-reason "Testing alarm notification"
```

### Terraform Configuration

```hcl
# Basic alarm for EC2 instance CPU utilization
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "high-cpu-utilization"
  alarm_description   = "This alarm monitors EC2 CPU utilization"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 3
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  treat_missing_data  = "notBreaching"
  
  dimensions = {
    InstanceId = "i-0123456789abcdef0"
  }
  
  # SNS topic to notify when alarm state changes
  alarm_actions = [aws_sns_topic.alerts.arn]
  ok_actions    = [aws_sns_topic.alerts.arn]
  
  tags = {
    Environment = "Production"
  }
}

# Alarm based on a metric filter from CloudWatch Logs
resource "aws_cloudwatch_metric_alarm" "http_errors" {
  alarm_name          = "http-500-errors"
  alarm_description   = "Alarm when HTTP 500 errors exceed threshold"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 5
  datapoints_to_alarm = 3
  threshold           = 5
  
  # Reference the metric created by the metric filter
  metric_name         = "HTTP5xxErrorCount"
  namespace           = "WebServer/Errors"
  period              = 60
  statistic           = "Sum"
  
  # Execute a Lambda function when the alarm triggers
  alarm_actions = [aws_lambda_function.notification_handler.arn]
  
  # Add dimensions if the metric filter created them
  dimensions = {
    "Service" = "WebApp"
  }
}

# Composite alarm that combines multiple conditions
resource "aws_cloudwatch_composite_alarm" "system_critical" {
  alarm_name        = "system-critical-condition"
  alarm_description = "Triggers when both CPU is high AND database connections are near limit"
  
  alarm_rule = "ALARM(${aws_cloudwatch_metric_alarm.high_cpu.alarm_name}) AND ALARM(${aws_cloudwatch_metric_alarm.db_connections.alarm_name})"
  
  # Actions to take when composite alarm triggers
  alarm_actions = [
    aws_sns_topic.critical_alerts.arn,
    aws_ssm_document.remediation_runbook.arn
  ]
}

# Anomaly detection alarm for application traffic
resource "aws_cloudwatch_metric_alarm" "traffic_anomaly" {
  alarm_name          = "abnormal-traffic-pattern"
  alarm_description   = "Detects unusual traffic patterns using anomaly detection"
  comparison_operator = "GreaterThanUpperThreshold"
  evaluation_periods  = 3
  threshold_metric_id = "e1"
  
  metric_query {
    id          = "m1"
    return_data = true
    
    metric {
      metric_name = "RequestCount"
      namespace   = "AWS/ApplicationELB"
      period      = 300
      stat        = "Sum"
      
      dimensions = {
        LoadBalancer = "app/my-load-balancer/50dc6c495c0c9188"
      }
    }
  }
  
  # Anomaly detection band configuration
  metric_query {
    id          = "e1"
    expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
    label       = "RequestCount (expected)"
    return_data = true
  }
  
  alarm_actions = [aws_sns_topic.security_alerts.arn]
}

# Auto Scaling alarm
resource "aws_cloudwatch_metric_alarm" "scale_up" {
  alarm_name          = "scale-up-alarm"
  alarm_description   = "Scale up when CPU is high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 75
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.web.name
  }
  
  alarm_actions = [aws_autoscaling_policy.scale_up.arn]
}
```

---

## Advanced Features

### Composite Alarms

Composite alarms combine multiple alarms using logical operators (AND, OR, NOT):

```
ALARM(high-cpu-alarm) AND ALARM(high-memory-alarm)
ALARM(primary-site-down) AND NOT ALARM(failover-active)
ALARM(api-errors) OR ALARM(database-errors)
```

Benefits:
- Reduce alarm noise by triggering only on combined conditions
- Create more meaningful operational alerts
- Enable complex remediation scenarios
- Avoid cascading alerts

### Anomaly Detection

CloudWatch can automatically create dynamic thresholds using machine learning:

- Learns normal metric patterns
- Adjusts for daily and weekly patterns
- Creates upper and lower bounds of "normal" behavior
- Triggers alarms when metrics deviate from expected patterns

Ideal for metrics with:
- Predictable patterns
- Daily or weekly cycles
- Seasonal variations
- Gradual growth trends


---

## Best Practices

### Alarm Design
- Set appropriate thresholds based on baseline metrics
- Use multiple evaluation periods to reduce false alarms
- Create alarms for both resource capacity and application performance
- Set actionable alarms that have clear resolution steps

### Organization
- Use consistent naming conventions for alarms
- Tag alarms for better management
- Group related alarms with composite alarms
- Document alarm purpose and expected actions

### Operations
- Test alarm configurations before deploying to production
- Review alarm history periodically to identify patterns
- Adjust thresholds based on operational experience
- Remove or suppress unnecessary alarms that create noise

### Threshold Selection
- **CPU**: 80% for warning, 90% for critical
- **Memory**: 85% for warning, 95% for critical
- **Disk**: 80% for warning, 90% for critical
- **Network**: Based on baseline + 2-3 standard deviations
- **Application metrics**: Based on SLA requirements

---

## Monitoring and Troubleshooting

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **False positives** | Alarms trigger too frequently | Adjust threshold or increase evaluation periods |
| **Delayed notifications** | Actions execute late | Check SNS topic configuration and subscriptions |
| **Missing data points** | INSUFFICIENT_DATA state | Verify metric is being published consistently |
| **Alarm not triggering** | Expected alarm state not reached | Review threshold, comparison operator, and evaluation criteria |
| **Action not executing** | Alarm triggers but no action | Check IAM permissions for alarm actions |

### Debugging Steps

1. **Check Alarm History**
   ```bash
   aws cloudwatch describe-alarm-history \
     --alarm-name "high-cpu-alarm" \
     --max-records 10
   ```

2. **Verify Metric Data**
   ```bash
   aws cloudwatch get-metric-statistics \
     --namespace AWS/EC2 \
     --metric-name CPUUtilization \
     --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
     --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
     --period 300 \
     --statistics Average
   ```

3. **Test Alarm Actions**
   ```bash
   aws cloudwatch set-alarm-state \
     --alarm-name "high-cpu-alarm" \
     --state-value ALARM \
     --state-reason "Testing alarm notification"
   ```

### Performance Optimization

- Use appropriate evaluation periods for metric stability
- Implement composite alarms to reduce notification noise
- Set up escalation policies for critical alarms
- Use tags to organize and filter alarms efficiently

---

## Cost Optimization

### Pricing Structure
- **Standard Alarms**: $0.10 per alarm per month
- **High-resolution Alarms**: $0.30 per alarm per month
- **Composite Alarms**: $0.50 per alarm per month
- **API Requests**: Standard CloudWatch API pricing applies

### Cost Optimization Strategies

1. **Right-size alarm quantities**
   - Consolidate similar alarms using dimensions
   - Use composite alarms instead of multiple individual alarms
   - Remove unused or redundant alarms

2. **Use appropriate alarm types**
   ```bash
   # Standard resolution for most use cases
   --period 300
   
   # High resolution only for critical systems
   --period 60
   ```

3. **Optimize evaluation periods**
   ```bash
   # Balance sensitivity vs cost
   --evaluation-periods 3  # Good balance
   --evaluation-periods 1  # More sensitive, more alarms
   --evaluation-periods 5  # Less sensitive, fewer false positives
   ```

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Alarms per region** | 10,000 | Per AWS account |
| **Actions per alarm** | 5 | SNS, Auto Scaling, EC2, Systems Manager |
| **Composite alarm rules** | 100 | Maximum conditions in alarm rule |
| **Metrics per alarm** | 1 | Standard alarms monitor single metric |
| **Evaluation periods** | 1-365 | Minimum and maximum evaluation periods |
| **Period** | 10 seconds - 1 day | For high-resolution metrics: 10, 30 seconds; Standard: 60 seconds or multiples of 60 |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding alarm states and state transitions
> - Configuring appropriate thresholds and evaluation periods
> - Using composite alarms for complex scenarios
> - Integrating alarms with Auto Scaling and SNS
> - Troubleshooting alarm configuration issues

### Key Exam Scenarios

1. **Auto Scaling Integration**: Configure alarms to trigger scaling policies
2. **Multi-metric Alerting**: Use composite alarms for complex conditions
3. **Cost Optimization**: Choose appropriate alarm types and configurations
4. **Troubleshooting**: Identify why alarms aren't triggering or actions aren't executing
5. **Anomaly Detection**: Implement ML-based thresholds for variable workloads


---

## Quick Reference

### Essential Commands
```bash
# Create alarm
aws cloudwatch put-metric-alarm --alarm-name "my-alarm" --metric-name CPUUtilization --namespace AWS/EC2 --threshold 80

# List alarms
aws cloudwatch describe-alarms --state-value ALARM

# Delete alarm
aws cloudwatch delete-alarms --alarm-names "my-alarm"

# Test alarm
aws cloudwatch set-alarm-state --alarm-name "my-alarm" --state-value ALARM --state-reason "Testing"
```

### Common Thresholds
```bash
# Infrastructure
CPU: 80% (warning), 90% (critical)
Memory: 85% (warning), 95% (critical)
Disk: 80% (warning), 90% (critical)

# Application
Error rate: 1% (warning), 5% (critical)
Response time: 200ms (warning), 500ms (critical)
Queue depth: 100 (warning), 1000 (critical)
```

---

## References

- [AWS CloudWatch Alarms User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)
- [CloudWatch Alarm States and Actions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html#alarm-states)
- [Composite Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Composite_Alarm.html)
- [CloudWatch Anomaly Detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html)

---

**Previous:** [← CloudWatch Logs](02-cloudwatch-logs.md) | **Next:** [CloudTrail →](04-cloudtrail.md)