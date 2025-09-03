# AWS X-Ray

## Overview

AWS X-Ray is a distributed tracing service that helps developers analyze and debug production applications. It provides insights into how your application and its underlying services are performing, enabling you to identify bottlenecks, understand dependencies, and pinpoint the root cause of performance issues and errors.

**Regional Service**: X-Ray is a regional service that traces requests within a region. For multi-region applications, you need to configure X-Ray in each region. However, X-Ray can trace requests that span multiple AWS services within the same region, providing end-to-end visibility.

## Exam Relevance

This module covers these Domain 1 tasks:
- Implementing distributed tracing for microservices architectures
- Analyzing application performance and identifying bottlenecks
- Troubleshooting service dependencies and latency issues
- Integrating X-Ray with AWS services and custom applications

---

## Key Concepts

### X-Ray Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Trace** | Complete request journey through your application | Shows end-to-end transaction flow | User login request across API Gateway, Lambda, and RDS |
| **Segment** | Individual service or component in a trace | Represents work done by a single service | Lambda function execution time and status |
| **Subsegment** | Detailed breakdown within a segment | Shows granular operations | Database query within a Lambda function |
| **Service Map** | Visual representation of application architecture | Identifies dependencies and bottlenecks | Shows API Gateway → Lambda → RDS connections |
| **Annotation** | Key-value pairs for filtering and indexing | Enables targeted analysis | Environment=production, UserType=premium |
| **Metadata** | Additional trace information (not indexed) | Provides context for debugging | Full request/response bodies, error details |
| **Sampling** | Controls which requests to trace | Manages cost and performance impact | Trace 10% of normal requests, 100% of errors |

#### Deep Dive into Tracing Concepts

**Trace**:
- Represents a single request's journey through your application
- Contains timing data, service calls, and error information
- Unique trace ID links all related segments and subsegments

**Segments and Subsegments**:
- **Segment**: Work done by a single service (Lambda function, EC2 instance)
- **Subsegment**: Granular operations within a service (database calls, HTTP requests)
- Both contain timing, status, and error information

> **🔍 Exam Alert:** Understanding the difference between segments and subsegments is crucial. Segments represent services, while subsegments represent operations within those services.

### X-Ray vs. Traditional Monitoring

| Aspect | Traditional Monitoring | X-Ray Distributed Tracing |
|--------|----------------------|---------------------------|
| **Scope** | Individual service metrics | End-to-end request flow |
| **Visibility** | Service-level health | Inter-service dependencies |
| **Troubleshooting** | Correlate logs manually | Automatic trace correlation |
| **Performance Analysis** | Service-specific latency | Request-level latency breakdown |
| **Error Investigation** | Service logs only | Complete error context |

---

## X-Ray SDK and Daemon

Understanding the roles of the X-Ray SDK and daemon is crucial for implementing distributed tracing effectively.

### X-Ray SDK

The X-Ray SDK is a library that you integrate into your application code to collect trace data.

#### SDK Responsibilities
- **Instrument your code** to create segments and subsegments
- **Capture trace data** including timing, metadata, and annotations  
- **Handle sampling decisions** based on configured sampling rules
- **Send segment data** to the X-Ray daemon via UDP on port 2000
- **Automatically instrument** AWS SDK calls and popular libraries

#### SDK Features
| Feature | Description | Example |
|---------|-------------|---------|
| **Auto-instrumentation** | Automatically traces AWS SDK calls | DynamoDB queries, S3 operations |
| **Custom segments** | Manual instrumentation for specific operations | Database connections, external API calls |
| **Annotations** | Key-value pairs for filtering traces | user_id=12345, environment=prod |
| **Metadata** | Additional context information | Request/response bodies, debug info |
| **Sampling** | Controls which requests to trace | Sample 10% of normal requests, 100% of errors |

#### SDK Installation Examples
```python
# Python
pip install aws-xray-sdk

# Import and configure
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Automatically instrument AWS SDK
patch_all()
```

```javascript
// Node.js
npm install aws-xray-sdk-core

// Import and configure
const AWSXRay = require('aws-xray-sdk-core');
const AWS = AWSXRay.captureAWS(require('aws-sdk'));
```

### X-Ray Daemon

The X-Ray daemon is a software application that listens for traffic on UDP port 2000, gathers raw segment data, and sends it to the AWS X-Ray API.

#### Daemon Responsibilities
- **Listen on UDP port 2000** for segment data from the SDK
- **Buffer segments** to optimize API calls to X-Ray service
- **Forward segments** in batches to AWS X-Ray API
- **Handle retries** and error handling for API calls
- **Manage IAM credentials** for authentication with X-Ray service

#### Daemon Deployment Options

| Platform | Installation Method | Notes |
|----------|-------------------|-------|
| **EC2** | Install as service/process | Requires IAM role with X-Ray write permissions |
| **ECS** | Run as sidecar container | Use `amazon/aws-xray-daemon` Docker image |
| **Fargate** | Sidecar container | X-Ray daemon runs alongside application |
| **Lambda** | Built-in (no separate daemon needed) | AWS manages the daemon automatically |
| **Elastic Beanstalk** | Configuration option | Enable via `.ebextensions` or console |

#### EC2 Daemon Setup
```bash
# Download and install X-Ray daemon on EC2
wget https://s3.us-east-2.amazonaws.com/aws-xray-assets.us-east-2/xray-daemon/aws-xray-daemon-linux-3.x.zip
unzip aws-xray-daemon-linux-3.x.zip
sudo cp xray /usr/local/bin/xray
sudo /usr/local/bin/xray -o -n us-east-1 &
```

#### ECS Daemon Configuration
```json
{
  "name": "xray-daemon",
  "image": "amazon/aws-xray-daemon",
  "cpu": 32,
  "memoryReservation": 256,
  "portMappings": [
    {
      "containerPort": 2000,
      "protocol": "udp"
    }
  ]
}
```

### SDK and Daemon Interaction

```
Application Code (SDK)  →  X-Ray Daemon  →  AWS X-Ray Service
     │                         │                    │
     │ 1. Creates segments      │ 3. Batches data    │ 5. Stores traces
     │ 2. Sends via UDP:2000    │ 4. Sends to API    │ 6. Provides service map
```

#### Key Interaction Points
1. **SDK instruments code** and creates segment data
2. **SDK sends segments** to daemon via UDP port 2000
3. **Daemon buffers segments** for efficient transmission  
4. **Daemon forwards batches** to X-Ray API with authentication
5. **X-Ray service processes** and stores trace data
6. **Console displays** service maps and trace analysis

> **🔍 Exam Alert:** Remember that the SDK collects trace data and sends it to the daemon, while the daemon buffers and forwards this data to the X-Ray service. Lambda functions have the daemon built-in, but EC2 instances need the daemon installed separately.

---

## Service Configuration

### Basic X-Ray Setup

Most applications start with this foundational configuration:

```hcl
# IAM role for X-Ray daemon
resource "aws_iam_role" "xray_role" {
  name = "XRayServiceRole"
  
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

# Attach AWS managed policy for X-Ray
resource "aws_iam_role_policy_attachment" "xray_policy" {
  role       = aws_iam_role.xray_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess"
}

# X-Ray sampling rule for cost control
resource "aws_xray_sampling_rule" "production_sampling" {
  rule_name      = "ProductionSampling"
  priority       = 9000
  version        = 1
  reservoir_size = 2
  fixed_rate     = 0.1  # Sample 10% of requests
  url_path       = "*"
  host           = "*"
  http_method    = "*"
  service_name   = "my-production-app"
  service_type   = "*"
  resource_arn   = "*"
}

# Higher sampling for errors
resource "aws_xray_sampling_rule" "error_sampling" {
  rule_name      = "ErrorSampling"
  priority       = 1000  # Higher priority (lower number)
  version        = 1
  reservoir_size = 5
  fixed_rate     = 1.0   # Sample 100% of errors
  url_path       = "*"
  host           = "*"
  http_method    = "*"
  service_name   = "*"
  service_type   = "*"
  resource_arn   = "*"
  
  attributes = {
    "aws.xray.error" = "true"
  }
}
```

### Lambda Integration

```hcl
# Lambda function with X-Ray tracing enabled
resource "aws_lambda_function" "api_handler" {
  filename         = "function.zip"
  function_name    = "api-handler"
  role            = aws_iam_role.lambda_role.arn
  handler         = "index.handler"
  runtime         = "python3.9"
  
  # Enable X-Ray tracing
  tracing_config {
    mode = "Active"  # Options: Active, PassThrough
  }
  
  environment {
    variables = {
      _X_AMZN_TRACE_ID = ""  # Automatically populated by Lambda
    }
  }
}

# API Gateway with X-Ray tracing
resource "aws_api_gateway_stage" "prod" {
  deployment_id = aws_api_gateway_deployment.prod.id
  rest_api_id   = aws_api_gateway_rest_api.api.id
  stage_name    = "prod"
  
  # Enable X-Ray tracing
  xray_tracing_enabled = true
}
```

---

## Real-World Use Cases

### 1. Microservices Performance Troubleshooting

**Scenario**: E-commerce application experiencing slow checkout process. Users complain about 5-second delays during peak hours.

**Solution**: Use X-Ray to trace the complete checkout flow and identify bottlenecks.

**Investigation Process**:
1. **Service Map Analysis**: Identified that payment service calls were taking 3+ seconds
2. **Trace Analysis**: Found 90% of latency was in database queries
3. **Root Cause**: Payment service was making 15 separate database calls instead of batching
4. **Resolution**: Optimized queries, reduced latency from 5s to 800ms

**Business Impact**:
- Increased conversion rate by 12%
- Reduced customer support tickets by 35%
- Improved user experience during peak shopping periods

### 2. Error Rate Investigation

**Scenario**: API Gateway showing 5% error rate, but individual services report healthy status.

**Solution**: X-Ray traces revealed the complete error context.

```python
# Lambda function with X-Ray instrumentation
import json
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all

# Automatically instrument AWS SDK calls
patch_all()

@xray_recorder.capture('lambda_handler')
def lambda_handler(event, context):
    try:
        # Create subsegment for database operation
        with xray_recorder.in_subsegment('database_query'):
            result = query_database(event['user_id'])
            
        # Add annotations for filtering
        xray_recorder.current_subsegment().put_annotation('user_type', 'premium')
        xray_recorder.current_subsegment().put_annotation('operation', 'user_lookup')
        
        # Add metadata for debugging
        xray_recorder.current_subsegment().put_metadata('request_data', event)
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
        
    except Exception as e:
        # X-Ray automatically captures exceptions
        xray_recorder.current_subsegment().add_exception(e)
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def query_database(user_id):
    # This will show up as a subsegment in X-Ray
    import boto3
    dynamodb = boto3.resource('dynamodb')
    table = dynamodb.Table('Users')
    
    response = table.get_item(Key={'user_id': user_id})
    return response.get('Item', {})
```

**Findings**:
- Errors occurred only for requests with specific user attributes
- Database timeout happening for users with large transaction histories
- Solution: Implemented pagination and caching for heavy users

### 3. Third-Party Service Dependencies

**Scenario**: Application integrates with multiple external APIs (payment processor, shipping provider, tax calculator). Need to identify which external service causes failures.

**Solution**: X-Ray traces external HTTP calls and provides visibility into third-party performance.

**Key Insights**:
- Payment processor had 99.9% uptime but 2-second average response time
- Shipping API failed 1% of requests but responded quickly when successful
- Tax service was the most reliable but only used for 20% of transactions

**Optimization Results**:
- Implemented circuit breaker pattern for unreliable services
- Added caching for tax calculations
- Set up failover payment processor for peak times

### 4. Cold Start Analysis

**Scenario**: Serverless application experiencing performance issues during traffic spikes due to Lambda cold starts.

**Solution**: Use X-Ray to analyze initialization time and optimize function performance.

**X-Ray Insights**:
- Cold start initialization took 2-3 seconds
- 80% of cold start time was spent importing libraries
- Warm Lambda functions processed requests in 50-100ms

**Optimizations Applied**:
- Reduced package size by 60%
- Implemented connection pooling
- Used provisioned concurrency for critical functions
- Result: Cold start time reduced to 500ms

---

## Best Practices

### Implementation Strategy
1. **Start with High-Value Services**: Enable tracing on customer-facing APIs first
2. **Use Sampling Wisely**: Sample 100% of errors, 10% of normal requests
3. **Instrument Strategically**: Focus on service boundaries and external calls
4. **Add Meaningful Annotations**: Use consistent annotation keys for filtering

### Performance Guidelines
- Keep trace data under 64KB per segment
- Use subsegments for granular operations (>10ms)
- Avoid tracing every internal function call
- Sample based on business value, not just volume

### Operational Excellence
- Create X-Ray service maps for each environment
- Monitor X-Ray service health with CloudWatch metrics
- Set up alerts for trace ingestion failures
- Review sampling rules monthly to optimize costs

---

## Monitoring and Troubleshooting

### Key Metrics to Monitor

| Metric | Purpose | Alert Threshold |
|--------|---------|-----------------|
| **TracesReceived** | Trace ingestion volume | Track for baseline |
| **TracesDropped** | Failed trace uploads | > 1% drop rate |
| **LatencyHigh** | 95th percentile response time | > SLA threshold |
| **ErrorRate** | Percentage of failed requests | > 1% error rate |

### Common Issues and Solutions

| Issue | Symptoms | Solution |
|-------|----------|----------|
| **Missing traces** | Expected traces not appearing | Check X-Ray daemon status, IAM permissions |
| **High costs** | Unexpected X-Ray bills | Review sampling rules, reduce trace volume |
| **Incomplete traces** | Segments missing from traces | Verify service instrumentation, check network connectivity |
| **Performance impact** | Application slowdown with X-Ray | Optimize sampling rates, reduce traced operations |

### Debugging Steps

```bash
# Check X-Ray service statistics
aws xray get-service-graph --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# Get trace summaries for analysis
aws xray get-trace-summaries --time-range-type TimeRangeByStartTime --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# Retrieve specific trace details
aws xray batch-get-traces --trace-ids "1-5f4e4b2a-234567890abcdef0"
```

---

## Cost Optimization

### Pricing Structure
- **Traces**: $5.00 per million traces recorded
- **Traces Retrieved**: $0.50 per million traces retrieved
- **Trace Storage**: First 30 days included, then $1.00 per million traces per month
- **Insights**: $0.50 per million traces analyzed

### Cost Optimization Strategies

1. **Smart Sampling**
   ```hcl
   # Low-cost sampling for high-volume endpoints
   resource "aws_xray_sampling_rule" "health_check" {
     rule_name      = "HealthCheckSampling"
     priority       = 8000
     fixed_rate     = 0.01  # Sample only 1% of health checks
     url_path       = "/health"
     service_name   = "*"
   }
   
   # High sampling for critical business operations
   resource "aws_xray_sampling_rule" "payment" {
     rule_name      = "PaymentSampling"
     priority       = 2000
     fixed_rate     = 0.5   # Sample 50% of payment requests
     url_path       = "/payment/*"
     service_name   = "payment-service"
   }
   ```

2. **Optimize Trace Size**
   - Limit metadata to essential debugging information
   - Use annotations instead of metadata for filtering
   - Avoid tracing internal utility functions

3. **Retention Management**
   - Use shorter retention for non-critical traces
   - Export traces to S3 for long-term analysis
   - Implement trace lifecycle policies

---

## Limitations and Quotas

| Resource | Limit | Notes |
|----------|-------|-------|
| **Trace size** | 500 KB | Per complete trace |
| **Segment size** | 64 KB | Per individual segment |
| **Annotations per segment** | 50 | Key-value pairs for indexing |
| **Metadata size** | 64 KB | Per segment metadata |
| **API rate limits** | 2,000 TPS | PutTraceSegments API |
| **Trace retention** | 30 days | Default retention period |
| **Service map nodes** | 10,000 | Maximum services displayed |

---

## Exam Tips

> **🎯 Exam Focus Areas:**
> - Understanding when to use X-Ray vs. CloudWatch vs. CloudTrail
> - Configuring sampling rules for cost optimization
> - Interpreting service maps and trace data
> - Troubleshooting missing or incomplete traces
> - Integrating X-Ray with serverless applications

### Key Exam Scenarios

1. **Performance Troubleshooting**: Use X-Ray to identify slow services in microservices architecture
2. **Error Investigation**: Trace failed requests to find root cause
3. **Cost Management**: Configure sampling rules to control X-Ray costs
4. **Serverless Monitoring**: Enable tracing for Lambda functions and API Gateway
5. **Third-Party Integration**: Monitor external service dependencies

---

## Quick Reference

### Essential Commands
```bash
# Get service graph
aws xray get-service-graph --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# List trace summaries
aws xray get-trace-summaries --time-range-type TimeRangeByStartTime --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)

# Get sampling rules
aws xray get-sampling-rules

# Create sampling rule
aws xray create-sampling-rule --sampling-rule file://sampling-rule.json
```

### Common Annotations
```bash
# Service identification
service_name, environment, version

# Request classification
user_type, request_type, feature_flag

# Business context
customer_tier, region, operation_type
```

---

## References

- [AWS X-Ray Developer Guide](https://docs.aws.amazon.com/xray/latest/devguide/)
- [X-Ray SDK Documentation](https://docs.aws.amazon.com/xray/latest/devguide/xray-sdk-python.html)
- [X-Ray Sampling Rules](https://docs.aws.amazon.com/xray/latest/devguide/xray-console-sampling.html)
- [X-Ray API Reference](https://docs.aws.amazon.com/xray/latest/api/Welcome.html)

---

**Previous:** [← Systems Manager](07-systems-manager.md) | **Back to Chapter:** [Monitoring & Observability](../README.md)