# AWS Monitoring Cheatsheet

Quick reference guide for AWS CloudOps certification

## CloudWatch Metrics

### Core Concepts
- **Metric**: Time-series data points (CPU, memory, network)
- **Namespace**: Organizes metrics (AWS/EC2, AWS/RDS, custom)
- **Dimension**: Key-value pairs that identify metrics (InstanceId, DBName)
- **Statistics**: Average, Sum, Min, Max, SampleCount, Percentiles
- **Resolution**: Standard (1-minute) vs High (1-second)

### Key Points for Exam
- **Retention**: 15 months for standard metrics
- **Regional Service**: Metrics exist only in their creation region
- **Custom Metrics**: Use PutMetricData API, costs $0.30 per metric
- **Percentiles**: p95 = 95% of requests faster than this value
- **High-Resolution**: More expensive, use only for critical systems

### Common Namespaces
- `AWS/EC2`: CPUUtilization, NetworkIn/Out, DiskReadOps
- `AWS/RDS`: DatabaseConnections, ReadLatency, WriteLatency
- `AWS/ELB`: RequestCount, TargetResponseTime, HTTPCode_Target_2XX
- `AWS/Lambda`: Duration, Errors, Invocations, Throttles

## CloudWatch Logs

### Core Concepts
- **Log Event**: Single log entry (max 256KB)
- **Log Stream**: Events from same source (one EC2 instance)
- **Log Group**: Container for streams with shared retention/permissions
- **Metric Filter**: Extract metrics from log patterns
- **Subscription Filter**: Stream logs to destinations (Lambda, Kinesis)

### Filter Patterns
- **Simple Text**: `ERROR`, `?ERROR ?WARN`, `-INFO`
- **Field-Based**: `[ip, user, timestamp, request, status_code=5*, size]`
- **JSON**: `{ $.level = "ERROR" && $.service = "payment" }`

### Key Points for Exam
- **Retention**: 1 day to 10 years, or never expire
- **Agent Required**: EC2 needs CloudWatch Agent for custom logs
- **Pricing**: $0.50/GB ingested, $0.03/GB-month storage
- **Log Insights**: Query with SQL-like syntax, $0.005/GB scanned

## CloudWatch Alarms

### Core Concepts
- **Alarm States**: OK, ALARM, INSUFFICIENT_DATA
- **Threshold**: Value that triggers state change
- **Evaluation Period**: Number of periods to check
- **Datapoints to Alarm**: M out of N datapoints must breach

### Alarm Actions
- **SNS**: Email, SMS notifications
- **Auto Scaling**: Scale out/in EC2 instances
- **EC2 Actions**: Stop, terminate, reboot, recover
- **Lambda**: Custom remediation logic

### Key Points for Exam
- **Missing Data**: Choose "notBreaching" for continuous metrics (CPU), "missing" for sporadic (errors)
- **Composite Alarms**: Combine multiple alarms with AND/OR logic
- **Anomaly Detection**: ML-based thresholds for variable patterns
- **Regional**: Cannot monitor cross-region metrics directly

## CloudTrail

### Core Concepts
- **Trail**: Configuration that records API calls
- **Management Events**: Control plane (CreateInstance, DeleteBucket)
- **Data Events**: Data plane (S3 GetObject, Lambda invoke) - costs extra
- **Insight Events**: ML detects unusual activity patterns

### Event Types
- **Management**: Enabled by default, administrative operations
- **Data**: Not enabled by default, resource-level operations
- **Global Service Events**: IAM, Route 53, CloudFront

### Key Points for Exam
- **Event History**: 90 days free for management events
- **Log File Integrity**: Enable validation for compliance
- **Multi-Region**: Configure trails to capture all regions
- **S3 Integration**: Store logs for long-term retention
- **CloudWatch Integration**: Real-time monitoring and alerting

## AWS Config

### Core Concepts
- **Configuration Item (CI)**: Point-in-time resource snapshot
- **Configuration Recorder**: Captures resource changes
- **Config Rules**: Evaluate compliance (AWS managed or custom)
- **Remediation**: Automated fixes for non-compliant resources

### Common Rules
- **encrypted-volumes**: All EBS volumes must be encrypted
- **restricted-ssh**: No security groups allow 0.0.0.0/0:22
- **s3-bucket-public-read-prohibited**: No public S3 buckets
- **root-user-mfa-enabled**: Root user must have MFA

### Key Points for Exam
- **Regional Service**: Configure in each region
- **Configuration History**: Track changes over time
- **Compliance Dashboard**: View overall compliance status
- **Aggregator**: Multi-region and multi-account visibility

## EventBridge

### Core Concepts
- **Event**: JSON message describing state change
- **Event Bus**: Routes events (default for AWS, custom for apps)
- **Rule**: Matches events and routes to targets
- **Target**: Destination (Lambda, SNS, SQS, Step Functions)
- **Event Pattern**: JSON filter for event matching

### Schedule Expressions
- **Rate**: `rate(5 minutes)`, `rate(1 hour)`, `rate(1 day)`
- **Cron**: `cron(0 2 * * ? *)` (daily 2 AM), `cron(0 18 ? * MON-FRI *)` (weekdays 6 PM)

### Key Points for Exam
- **Evolution of CloudWatch Events**: More features (custom buses, schema registry)
- **Event-Driven Architecture**: Decouple microservices
- **Scheduled Lambda**: Preferred over EC2 cron jobs
- **Cross-Region Replication**: Events can span regions

## Systems Manager

### Core Concepts
- **SSM Agent**: Software enabling Systems Manager (pre-installed on Amazon Linux/Windows)
- **Managed Instance**: EC2/on-premises with SSM Agent
- **Parameter Store**: Secure config storage (strings, encrypted)
- **Session Manager**: Browser-based shell access (no SSH keys)
- **Patch Manager**: Automated OS patching

### Key Components
- **Documents**: Define automation actions (AWS-UpdateSSMAgent)
- **Maintenance Windows**: Schedule patching/updates
- **Compliance**: Track patch and configuration compliance
- **Run Command**: Execute commands across multiple instances

### Key Points for Exam
- **Prerequisites**: SSM Agent, IAM role, internet/VPC endpoints
- **Security**: No SSH keys, IAM-based access control
- **Parameter Store**: Free tier (10,000 parameters), advanced ($0.05/10K/month)
- **Cross-Platform**: Works with Linux, Windows, on-premises

## X-Ray

### Core Concepts
- **Trace**: Complete request journey through application
- **Segment**: Work done by single service (Lambda, EC2)
- **Subsegment**: Operations within service (DB query, HTTP call)
- **Service Map**: Visual application architecture
- **Sampling**: Controls cost and performance impact

### SDK and Daemon
- **SDK**: Library integrated in application code
  - Instruments code, captures timing data
  - Sends to daemon via UDP port 2000
- **Daemon**: Software that forwards segments to X-Ray API
  - Required on EC2/ECS, built-in for Lambda
  - Buffers and batches data for efficiency

### Key Points for Exam
- **Distributed Tracing**: End-to-end request visibility
- **Performance Analysis**: Identify bottlenecks in microservices
- **Regional Service**: Configure per region
- **Pricing**: $5/million traces recorded, $0.50/million retrieved

## Exam Strategy Tips

### Service Selection
- **Performance Issues**: X-Ray for distributed tracing, CloudWatch for metrics
- **Security Auditing**: CloudTrail for API calls, Config for compliance
- **Automation**: EventBridge for event-driven, Systems Manager for operations
- **Log Analysis**: CloudWatch Logs Insights for queries

### Cost Optimization
- **CloudWatch**: Set appropriate retention, use standard resolution
- **CloudTrail**: Management events free, data events cost extra
- **X-Ray**: Smart sampling rules (10% normal, 100% errors)
- **Config**: Focus on critical compliance rules

### Regional Considerations
- **All services are regional** except CloudTrail global events
- **Multi-region**: Deploy monitoring in each region
- **Cross-region**: Use CloudWatch cross-region dashboards for visibility

### Common Patterns
- **CloudWatch Alarm → SNS → Lambda**: Automated remediation
- **CloudTrail → CloudWatch Logs → Metric Filter → Alarm**: Security monitoring
- **EventBridge Rule → Lambda**: Event-driven automation
- **Config Rule → Remediation**: Compliance automation

---

> **🎯 Exam Focus**: Understand when to use each service, their limitations, and how they integrate. Practice identifying the right monitoring solution for different scenarios.
