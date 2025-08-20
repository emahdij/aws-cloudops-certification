# CloudWatch Metrics

## Overview

Amazon CloudWatch Metrics is the foundation of AWS monitoring. It collects and tracks metrics (time-series data) from your AWS resources and applications, providing visibility into system performance, resource utilization, and application health.

**Regional Service**: CloudWatch Metrics is a regional service. Metrics exist only in the region where they are created and cannot be aggregated across regions natively. When designing multi-region architectures, you'll need to monitor each region's metrics separately or use custom solutions for cross-region visibility.

## Exam Relevance

This module covers these Domain 1 tasks:
- Collecting metrics using native AWS services and CloudWatch agent
- Creating CloudWatch dashboards for visualization
- Understanding the metrics that drive CloudWatch alarms

---

## Key Concepts

### Metric Fundamentals

| Concept | Description | CloudOps Importance | Example |
|---------|-------------|-------------------|---------|
| **Metric** | Time-ordered measurement of resource or application | Core to monitoring strategy | `CPUUtilization` for an EC2 instance |
| **Namespace** | Container for organizing related metrics | Critical for metric organization | `AWS/EC2` for EC2 metrics, `MyApplication/Production` for custom app metrics |
| **Dimension** | Name-value pair that uniquely identifies a metric | Essential for filtering | `InstanceId=i-1234567890abcdef0` to identify a specific EC2 instance |
| **Resolution** | Data point granularity (standard or high-resolution) | Affects cost and troubleshooting speed | Standard: 1-minute data points; High-resolution: 1-second data points |
| **Retention** | How long metrics are stored | Important for compliance and investigation | 15-month retention for standard metrics with 1-hour period |
| **Statistics** | Calculations performed on metrics (Avg, Sum, Min, Max, etc.) | Used for alerts and analysis | `Average` of `CPUUtilization` over 5 minutes to trigger an alarm |

#### Metrics Deep Dive

- **Metric**: Time-ordered measurement of resource or application (CPU, memory, network)
  - Each metric data point includes timestamp, value, and optional unit
  - Example: `CPUUtilization` tracks CPU usage percentage for EC2 instances

- **Namespace**: Container for organizing related metrics
  - AWS services use namespaces like `AWS/EC2`, `AWS/RDS`
  - Custom applications should use unique namespaces (e.g., `MyApp/Production`)
  - Exam tip: Know the standard namespaces for common AWS services

- **Dimension**: Name-value pair that uniquely identifies a metric
  - Example: `InstanceId=i-1234567890abcdef0` identifies a specific EC2 instance
  - Up to 10 dimensions per metric
  - Critical for filtering and organizing metrics
  - Common dimensions: `InstanceId`, `DBInstanceIdentifier`, `LoadBalancerName`

#### Resolution and Retention

- **Resolution**: Data point granularity
  - **Standard**: 1-minute intervals (default)
  - **High-Resolution**: 1-second intervals (higher cost)
  - CloudOps tip: Use high-resolution metrics only for critical components

- **Retention**: How long metrics are stored (automatic policy)
  - Standard metrics: 15 months
  - Data points with period < 60 seconds: 3 hours
  - Data points with period of 60 seconds: 15 days
  - Data points with period of 300 seconds: 63 days
  - Data points with period of 3600 seconds: 15 months

> **🔍 Exam Alert:** You'll likely be tested on retention periods for different metric resolutions. Remember that higher-resolution metrics have shorter retention periods.

#### Statistics

Statistics define how metric data points are aggregated over a time period:

- **Average**: Mean value over the period
- **Sum**: Total of all values over the period  
- **Minimum/Maximum**: Lowest/highest value in the period
- **SampleCount**: Number of data points within the period
- **Percentile**: Value below which a percentage of data points fall (e.g., p95, p99)
- **Trimmed Mean**: Average with outliers removed (e.g., TM(25%:75%))
- **Interquartile Mean (IQM)**: Average of values between p25 and p75

> **📊 Understanding Percentiles**: Percentiles are crucial for performance monitoring. The 95th percentile (p95) means 95% of requests were faster than this value - more useful than averages which can hide outliers. For example, if p95 response time is 200ms, only 5% of users experienced slower responses.
> 
> **Why Percentiles Matter**: A service might have an average response time of 100ms, but if p99 is 5 seconds, some users are having terrible experiences. Use p95 or p99 for SLAs instead of averages.
> 
> **Trimmed Mean (TM)**: Removes outliers before calculating the average. TM(25%:75%) removes the bottom 25% and top 25% of values, then averages the middle 50%. This provides a more stable metric than regular averages when dealing with noisy data or occasional spikes.
> 
> **Interquartile Mean (IQM)**: Calculates the average of values between the 25th percentile (p25) and 75th percentile (p75). Similar to TM(25%:75%) but specifically focuses on the middle half of your data distribution. Useful for understanding "typical" performance while ignoring both best-case and worst-case scenarios.
> 
> **When to Use Each**: Use regular averages for stable metrics, percentiles for user experience SLAs, trimmed means for noisy data with outliers, and IQM when you want to understand normal operational performance excluding extremes.
> 
> **Learn More**: [High Performance Browser Networking - Primer on Latency](https://hpbn.co/primer-on-latency-and-bandwidth/) provides excellent context on why percentiles matter for user experience.

---

## Default Metrics vs. Custom Metrics

### Default Metrics
- Automatically collected by AWS services at no additional cost
- Different services expose different metrics
- Examples:
  - EC2: CPUUtilization, NetworkIn/Out, StatusCheckFailed
  - RDS: DatabaseConnections, CPUUtilization, FreeStorageSpace
  - ELB: RequestCount, TargetResponseTime, HTTPCode_Target_5XX_Count

> **💡 Practice Tip:** Familiarize yourself with the default metrics for key services. Create a study card for each service with its most important metrics.

### Custom Metrics
- User-defined metrics for application-specific monitoring
- Published manually via API or using CloudWatch agent
- Charged per metric ($0.30 per metric per month)
- Use cases: application response times, business metrics, custom error rates, memory utilization

> **💰 Cost Optimization:** Group similar metrics with dimensions rather than creating separate metrics when possible to reduce costs.

## Using Dimensions for Metric Organization

Dimensions allow you to slice and filter your metric data along different attributes without creating separate metrics. This approach provides several benefits:

1. **Simplified Management**: Fewer metrics to track and manage
2. **Flexible Analysis**: Filter and aggregate data easily across different dimensions
3. **Consistent Visualization**: Create dashboards that dynamically filter by dimension

> **⚠️ IMPORTANT CloudWatch Pricing Clarification**: In CloudWatch, every unique combination of a metric name and its set of dimensions is counted as a separate, billable metric. For example:
> - `CPUUtilization` with `{InstanceId: i-123, Environment: prod}` = 1 metric
> - `CPUUtilization` with `{InstanceId: i-456, Environment: prod}` = 1 metric  
> - `CPUUtilization` with `{InstanceId: i-123, Environment: dev}` = 1 metric
> 
> **Total**: 3 separate billable metrics × $0.30 = $0.90/month
> 

### Examples of Effective Metric Grouping

**Inefficient Approach** (12 separate metric names):
- `login_service_response_time`
- `login_service_error_count`
- `login_service_request_count`
- `payment_service_response_time`
- `payment_service_error_count`
- `payment_service_request_count`
- `catalog_service_response_time`
- `catalog_service_error_count`
- `catalog_service_request_count`
- `checkout_service_response_time`
- `checkout_service_error_count`
- `checkout_service_request_count`

**Efficient Approach** (3 metric names with dimensions):
- `service_response_time` with `ServiceName` dimension (4 unique combinations)
- `service_error_count` with `ServiceName` dimension (4 unique combinations)  
- `service_request_count` with `ServiceName` dimension (4 unique combinations)

> **💡 Key Insight**: The cost per unique combination remains the same, but dimensions provide operational benefits for organization, filtering, and management.

### Knowledge Check
1. What's the difference between standard and high-resolution metrics?
2. How long are 1-minute metrics retained in CloudWatch?
3. What's the maximum number of dimensions you can use for a single metric?
4. What namespace would you use for custom application metrics?

<details>
<summary>See answers</summary>

1. Standard metrics have 1-minute granularity, while high-resolution metrics have 1-second granularity.
2. 1-minute metrics (60-second period) are retained for 15 days.
3. You can use up to 10 dimensions per metric.
4. For custom application metrics, you should use a custom namespace (e.g., "MyApplication/Production").
</details>

---

## CloudWatch Agent

The CloudWatch agent is essential for collecting system-level metrics and logs from EC2 instances and on-premises servers.

### When to Use the CloudWatch Agent
- When you need memory or disk metrics from EC2 (not available by default)
  > **Why not default?** EC2 metrics come from the hypervisor level, which can see CPU and network but cannot access OS-level information like memory usage or disk space inside the instance.
- To collect custom application logs
- For on-premises server monitoring
- To collect metrics at sub-minute intervals

### CloudWatch Agent Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    EC2 Instance / On-Premises               │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐    ┌─────────────────────────────────┐ │
│  │   Application   │    │     CloudWatch Agent           │ │
│  │                 │    │  ┌─────────────┐ ┌────────────┐ │ │
│  │  - Writes logs  │───▶│  │ Log         │ │ Metrics    │ │ │
│  │  - Custom       │    │  │ Collection  │ │ Collection │ │ │
│  │    metrics      │───▶│  │             │ │            │ │ │
│  └─────────────────┘    │  └─────────────┘ └────────────┘ │ │
│                         │         │              │       │ │
│  ┌─────────────────┐    │         ▼              ▼       │ │
│  │   OS System     │    │  ┌─────────────────────────────┐ │ │
│  │  - Memory       │───▶│  │    Unified Agent Core       │ │ │
│  │  - Disk I/O     │    │  │  (Processes & Batches)      │ │ │
│  │  - Process      │    │  └─────────────────────────────┘ │ │
│  └─────────────────┘    └─────────────────┬───────────────┘ │
└──────────────────────────────────────────┼─────────────────┘
                                           │
                                           ▼
┌─────────────────────────────────────────────────────────────┐
│                     AWS CloudWatch                         │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────┐              ┌─────────────────────┐   │
│  │ CloudWatch      │              │ CloudWatch Metrics  │   │
│  │ Logs            │              │                     │   │
│  │                 │              │ - Custom metrics    │   │
│  │ - Log streams   │              │ - System metrics    │   │
│  │ - Log groups    │              │ - Performance data  │   │
│  └─────────────────┘              └─────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

### Installing and Configuring

<details>
<summary>CloudWatch Agent Installation Steps</summary>

```bash
# Install on Amazon Linux 2/2023
sudo yum install amazon-cloudwatch-agent -y

# Install on Ubuntu/Debian
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb

# Install on macOS (for local testing with LocalStack)
# Download the installer package manually:
curl -O https://s3.amazonaws.com/amazoncloudwatch-agent/darwin/amd64/latest/amazon-cloudwatch-agent.pkg
sudo installer -pkg amazon-cloudwatch-agent.pkg -target /

# Configure using interactive wizard (recommended for beginners)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
# ↳ This creates a configuration file through guided questions

# Start the agent with the configuration created by wizard
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Alternative: Use a predefined configuration file (for automation)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s \
  -c file:/path/to/your/custom-config.json

# Check agent status
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -m ec2 -a query-config

# Stop the agent (if needed)
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -m ec2 -a stop

# View agent logs for troubleshooting
sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

**Prerequisites:**
- EC2 instance with appropriate IAM role (CloudWatchAgentServerPolicy)
- AWS CLI configured or instance profile attached
- Sufficient disk space for agent and logs (~100MB)

</details>

### Example Agent Configuration
```json
{
  "agent": {
    "metrics_collection_interval": 60
  },
  "metrics": {
    "namespace": "MyCustomNamespace",
    "metrics_collected": {
      "cpu": {
        "resources": ["*"],
        "measurement": [
          "usage_idle",
          "usage_user", 
          "usage_system"
        ],
        "totalcpu": true
      },
      "mem": {
        "measurement": [
          "used_percent",
          "available"
        ]
      },
      "disk": {
        "resources": ["/", "/tmp"],
        "measurement": [
          "used_percent",
          "inodes_free"
        ]
      },
      "diskio": {
        "resources": ["*"],
        "measurement": [
          "io_time",
          "write_bytes",
          "read_bytes"
        ]
      }
    },
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "InstanceType": "${aws:InstanceType}",
      "Environment": "Production"
    }
  }
}
```

**Configuration Key Explanations:**
- `metrics_collection_interval`: How often to collect metrics (seconds) - Lower values = more granular data but higher costs
- `namespace`: Where metrics appear in CloudWatch - Use descriptive names like "Production/WebServers" for organization  
- `cpu.resources`: Monitor all CPUs ("*") or specific ones ["cpu0", "cpu1"]
- `cpu.measurement`: 
  - `usage_idle`: Percentage of time CPU is idle
  - `usage_user`: Percentage of time CPU is used by user processes  
  - `usage_system`: Percentage of time CPU is used by system processes
- `cpu.totalcpu`: Include overall CPU percentage across all cores
- `mem.measurement`:
  - `used_percent`: Memory usage as percentage (most useful for alarms)
  - `available`: Available memory in bytes
- `disk.resources`: Monitor specific partitions ["/", "/tmp"] or add others like ["/", "/var", "/opt"] as needed
- `disk.measurement`:
  - `used_percent`: Disk space usage as percentage
  - `inodes_free`: Available inodes (file system objects)
- `diskio.resources`: Monitor all disk devices ("*")
- `diskio.measurement`:
  - `io_time`: Time spent on I/O operations
  - `write_bytes`: Bytes written to disk  
  - `read_bytes`: Bytes read from disk
- `append_dimensions`: These help you filter and group metrics in CloudWatch dashboards
  - `InstanceId`: Automatically add EC2 instance ID
  - `InstanceType`: Automatically add EC2 instance type
  - `Environment`: Custom dimension for environment

For more details, refer to the [official CloudWatch Agent documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html#CloudWatch-Agent-Configuration-File-Logssection).

---

## Implementing CloudWatch Dashboards

CloudWatch dashboards provide a customizable view of your metrics and alarms.

### Dashboard Capabilities
- Visualize metrics across AWS services
- Create custom widgets (graphs, numbers, text)
- Share dashboards across accounts
- Set automatic refresh intervals

### Dashboard Creation

#### Using AWS CLI

```bash
# Create a dashboard with multiple widget types
aws cloudwatch put-dashboard \
  --dashboard-name "MyApplicationDashboard" \
  --dashboard-body '{
    "widgets": [
      {
        "type": "metric",
        "x": 0,
        "y": 0,
        "width": 12,
        "height": 6,
        "properties": {
          "metrics": [
            ["AWS/EC2", "CPUUtilization", "InstanceId", "i-012345678901"]
          ],
          "period": 300,
          "stat": "Average",
          "region": "us-east-1",
          "title": "EC2 CPU Utilization",
          "yAxis": {
            "left": {
              "min": 0,
              "max": 100
            }
          },
          "view": "timeSeries"
        }
      },
      {
        "type": "text",
        "x": 0,
        "y": 6,
        "width": 24,
        "height": 2,
        "properties": {
          "markdown": "## Application Health Metrics\nThis dashboard shows the health of our main application components.\n\n### Key Metrics:\n- **CPU**: Should stay below 80%\n- **Memory**: Should stay below 85%\n- **Disk**: Should stay below 90%"
        }
      },
      {
        "type": "metric",
        "x": 12,
        "y": 0,
        "width": 12,
        "height": 6,
        "properties": {
          "metrics": [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", "database-1"],
            [".", "DatabaseConnections", ".", "."],
            [".", "FreeStorageSpace", ".", "."]
          ],
          "period": 300,
          "stat": "Average",
          "region": "us-east-1",
          "title": "RDS Performance",
          "annotations": {
            "horizontal": [
              {
                "label": "CPU Alert Threshold",
                "value": 80
              }
            ]
          }
        }
      }
    ]
  }'

# Get dashboard details (returns the JSON definition)
aws cloudwatch get-dashboard --dashboard-name "MyApplicationDashboard"

# List all dashboards
aws cloudwatch list-dashboards

# Delete dashboard
aws cloudwatch delete-dashboards --dashboard-names "MyApplicationDashboard"

# macOS zsh: Create dashboard from file for complex configurations
cat > dashboard-config.json << 'EOF'
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 24, "height": 6,
      "properties": {
        "metrics": [
          ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/my-lb/50dc6c495c0c9188"],
          [".", "HTTPCode_Target_4XX_Count", ".", "."],
          [".", "HTTPCode_Target_5XX_Count", ".", "."]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Load Balancer Requests and Errors",
        "view": "timeSeries",
        "stacked": false
      }
    }
  ]
}
EOF

# Create dashboard from file
aws cloudwatch put-dashboard \
  --dashboard-name "LoadBalancerDashboard" \
  --dashboard-body file://dashboard-config.json
```

**Dashboard JSON Structure Explained:**
- **Widget Types**: 
  - `"type": "metric"` - Time series charts, single value displays
  - `"type": "text"` - Markdown documentation, instructions, context  
  - `"type": "log"` - CloudWatch Logs query results
  - `"type": "number"` - Single metric value display
- **Positioning**: 
  - `x`, `y` - Position on dashboard grid (0-based)
  - `width`, `height` - Widget size (1-24 units width, 1-1000 units height)
- **Metrics Array**: Each array represents `[Namespace, MetricName, DimensionName, DimensionValue]`
- **Dot Notation**: `[".", "MetricName", ".", "."]` reuses previous namespace and dimensions
- **Properties**:
  - `period` - Data point interval in seconds (60, 300, 900, 3600)
  - `stat` - Statistic: Average, Sum, Min, Max, SampleCount, or pXX (percentiles)
  - `view` - Chart type: timeSeries, singleValue, pie
  - `annotations` - Add horizontal/vertical lines for thresholds

> **💡 Dashboard Design Tips:**
> - **Layout**: Use a 24-unit grid system for widget positioning
> - **Widget Types**: 
>   - `metric` - Time series charts, single value displays
>   - `text` - Markdown documentation, instructions, context
>   - `log` - CloudWatch Logs query results
>   - `number` - Single metric value display
> - **Sizing**: Standard widgets are 12x6 (half width), full-width text is 24x2
> - **Metrics Array**: Use `[".", "MetricName", ".", "."]` to reuse namespace and dimensions
> - **Performance**: Limit to 100 metrics per dashboard for optimal loading
> - **Sharing**: Dashboards can be shared publicly or with specific AWS accounts

#### Using Terraform

```HCL
resource "aws_cloudwatch_dashboard" "main" {
  dashboard_name = "application-dashboard"
  
  dashboard_body = jsonencode({
    widgets = [
      {
        type = "metric"
        x    = 0
        y    = 0
        width = 12
        height = 6
        
        properties = {
          metrics = [
            ["AWS/EC2", "CPUUtilization", "InstanceId", "i-012345678901"]
          ]
          period = 300
          stat = "Average"
          region = "us-east-1"
          title = "EC2 CPU Utilization"
        }
      },
      {
        type = "metric"
        x    = 12
        y    = 0
        width = 12
        height = 6
        
        properties = {
          metrics = [
            ["AWS/RDS", "CPUUtilization", "DBInstanceIdentifier", "database-1"]
          ]
          period = 300
          stat = "Average"
          region = "us-east-1"
          title = "RDS CPU Utilization"
        }
      },
      {
        type = "metric"
        x    = 0
        y    = 6
        width = 24
        height = 6
        
        properties = {
          view = "timeSeries"
          stacked = false
          metrics = [
            ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/my-lb/50dc6c495c0c9188"],
            [".", "HTTPCode_Target_4XX_Count", ".", "."],
            [".", "HTTPCode_Target_5XX_Count", ".", "."]
          ]
          region = "us-east-1"
          title = "Load Balancer Requests and Errors"
        }
      }
    ]
  })
}
```

---

## Working with Metrics

### Using AWS CLI

```bash
# List metrics for a specific namespace
aws cloudwatch list-metrics --namespace AWS/EC2

# List metrics for a specific instance
aws cloudwatch list-metrics \
  --namespace AWS/EC2 \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0

# Get metric statistics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --start-time $(date -u -v-1d +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 3600 \
  --statistics Average Maximum Minimum

# Publish a custom metric
aws cloudwatch put-metric-data \
  --namespace "MyApplication" \
  --metric-name "OrderProcessingTime" \
  --value 120 \
  --unit Milliseconds \
  --dimensions Service=Payment,Region=US
```

---

## Advanced Features

### Metric Math

**What is Metric Math?**
Metric Math allows you to perform mathematical calculations on CloudWatch metrics to create derived insights without storing additional custom metrics. Instead of just viewing raw CPU or request metrics, you can calculate error rates, efficiency ratios, or combine multiple metrics into meaningful business indicators.

**Why use Metric Math?**
- **Cost-effective**: Calculate derived metrics on-demand without paying $0.30/month for each custom metric
- **Real-time insights**: Get error rates, percentages, and ratios instantly
- **Better alerting**: Create alarms on calculated values like "error rate > 5%" instead of raw error counts
- **Business metrics**: Transform technical metrics into business-relevant KPIs

**Common use cases:**
- Error rate: `(errors / total_requests) * 100`
- Memory utilization: `(used_memory / total_memory) * 100`
- Cost per request: `daily_cost / request_count`

```bash
# Example: Calculate error rate percentage from ELB metrics
aws cloudwatch get-metric-data \
  --start-time $(date -u -v-1H +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --metric-data-queries '[
    {
      "Id": "m1",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB",
          "MetricName": "HTTPCode_Target_2XX_Count",
          "Dimensions": [
            {
              "Name": "LoadBalancer",
              "Value": "app/my-lb/50dc6c495c0c9188"
            }
          ]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    },
    {
      "Id": "m2",
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApplicationELB", 
          "MetricName": "RequestCount",
          "Dimensions": [
            {
              "Name": "LoadBalancer",
              "Value": "app/my-lb/50dc6c495c0c9188"
            }
          ]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    },
    {
      "Id": "error_rate",
      "Expression": "((m2 - m1) / m2) * 100",
      "Label": "Error Rate Percentage"
    },
    {
      "Id": "success_rate",
      "Expression": "(m1 / m2) * 100",
      "Label": "Success Rate Percentage"
    }
  ]'

**Command Structure Explained:**
- **Time Range**: Uses `date` command for dynamic "last hour" instead of hardcoded dates
- **Metric IDs**: 
  - `"Id": "m1"` - Unique identifier for successful requests (HTTPCode_Target_2XX_Count)
  - `"Id": "m2"` - Unique identifier for total requests (RequestCount)
  - `"Id": "error_rate"` - Calculated metric using math expression
- **MetricStat Properties**:
  - `Namespace` - AWS service namespace (AWS/ApplicationELB)
  - `MetricName` - Specific metric (HTTPCode_Target_2XX_Count, RequestCount)
  - `Dimensions` - Filters to specific load balancer
  - `Period` - 5-minute intervals (300 seconds)
  - `Stat` - "Sum" totals all values in each period
- **Expressions**:
  - `((m2 - m1) / m2) * 100` - Error rate: (total - success) / total * 100
  - `(m1 / m2) * 100` - Success rate: success / total * 100
```

**Expected output format:**
```json
{
    "MetricDataResults": [
        {
            "Id": "error_rate",
            "Label": "Error Rate Percentage",
            "Timestamps": ["2025-08-25T14:00:00.000Z", "2025-08-25T14:05:00.000Z"],
            "Values": [2.5, 1.8],
            "StatusCode": "Complete"
        },
        {
            "Id": "success_rate", 
            "Label": "Success Rate Percentage",
            "Timestamps": ["2025-08-25T14:00:00.000Z", "2025-08-25T14:05:00.000Z"],
            "Values": [97.5, 98.2],
            "StatusCode": "Complete"
        }
    ]
}
```

**Output Explanation:**
- `Timestamps` - Time points for each data value (ISO 8601 format)
- `Values` - Calculated percentages (error rate: 2.5%, 1.8% | success rate: 97.5%, 98.2%)
- `StatusCode` - "Complete" means calculation succeeded, "PartialData" means some data missing

**Common Metric Math Functions:**

- **AVG([m1, m2])**: Average of metrics
- **SUM([m1, m2])**: Sum of metrics  
- **MIN([m1, m2])**: Minimum value of metrics
- **MAX([m1, m2])**: Maximum value of metrics
- **RATE(m1)**: Rate of change
- **DIFF(m1, m2)**: Difference between metrics
- **FILL(m1, value)**: Replace missing data points with a value
- **SEARCH()**: Dynamically find metrics matching a pattern
- **IF(condition, true_val, false_val)**: Conditional logic
- **INSIGHT_RULE_METRIC()**: Use CloudWatch Insights rules

---

### Anomaly Detection

**What is Anomaly Detection?**
CloudWatch Anomaly Detection uses machine learning to analyze metric patterns and automatically detect when values deviate significantly from expected behavior. Instead of setting fixed thresholds, it learns what's "normal" for your application and alerts when metrics behave unusually.

**How it works:**
1. Analyzes up to 2 weeks of historical data
2. Creates a "band" of expected values
3. Triggers alarms when metrics fall outside this band
4. Continuously learns and adjusts the model

```HCL
resource "aws_cloudwatch_metric_alarm" "anomaly" {
  alarm_name          = "cpu-anomaly-detection"
  comparison_operator = "GreaterThanUpperThreshold"
  evaluation_periods  = 3
  threshold_metric_id = "e1"
  alarm_description   = "This alarm detects CPU utilization anomalies based on ML patterns"
  treat_missing_data  = "breaching"
  
  metric_query {
    id          = "m1"
    return_data = true
    
    metric {
      metric_name = "CPUUtilization"
      namespace   = "AWS/EC2"
      period      = 300
      stat        = "Average"
      
      dimensions = {
        InstanceId = "i-1234567890abcdef0"
      }
    }
  }
  
  metric_query {
    id          = "e1"
    expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
    label       = "CPUUtilization (expected range)"
    return_data = true
  }
}
```

**Terraform Configuration Explained:**
- `comparison_operator = "GreaterThanUpperThreshold"` - Triggers when metric exceeds upper band
- `evaluation_periods = 3` - Alert after 3 consecutive anomalous periods 
- `threshold_metric_id = "e1"` - References the anomaly detection band
- `treat_missing_data = "breaching"` - How to handle missing data points
- **Metric Query m1**: The raw metric data to monitor
- **Metric Query e1**: The ML-generated expected range
- `ANOMALY_DETECTION_BAND(m1, 2)` - Creates upper/lower bounds based on historical patterns
- The "2" means alert when metric is >2 standard deviations from expected

**Using AWS CLI for Anomaly Detection:**
```bash
# Create anomaly detector for a specific metric
aws cloudwatch put-anomaly-detector \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --stat "Average"

# Get anomaly detection results  
aws cloudwatch get-metric-data \
  --start-time $(date -u -v-2H +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --metric-data-queries '[
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
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
      "Label": "Expected CPU Range"
    }
  ]'

# List all anomaly detectors
aws cloudwatch describe-anomaly-detectors

# Delete anomaly detector
aws cloudwatch delete-anomaly-detector \
  --namespace "AWS/EC2" \
  --metric-name "CPUUtilization" \
  --dimensions Name=InstanceId,Value=i-1234567890abcdef0 \
  --stat "Average"
```

**CLI Command Explanations:**
- `put-anomaly-detector` - Starts ML model training (takes ~3 hours for initial model)
- `get-metric-data` - Retrieves both actual values and expected range
- `describe-anomaly-detectors` - Lists all active anomaly detectors
- `delete-anomaly-detector` - Removes the ML model and stops monitoring
- **Time Range**: Uses dynamic "last 2 hours" for recent anomaly data

> **💡 Anomaly Detection Tips:**
> - **Training period**: Requires 3+ hours of data for initial model, 2 weeks for best accuracy
> - **Standard deviations**: Use 1 for sensitive detection, 2 for balanced, 3+ for less sensitive
> - **Best for**: Metrics with patterns (daily/weekly cycles, gradual trends)
> - **Not ideal for**: Highly erratic metrics or metrics that change frequently
> - **Cost**: $0.30 per anomaly detector per month (same as custom metrics)

---

### Metric Streams

**What are Metric Streams?**
Metric Streams are a feature used to continuously stream CloudWatch metrics to a destination of your choice. This is the most efficient way to get metric data out of CloudWatch in near real-time.

**Why use Metric Streams?**
- **Third-Party Observability**: Send metrics to platforms like Datadog, New Relic, Splunk, or Grafana.
- **Data Lake Integration**: Archive and analyze metric data in Amazon S3.
- **Custom Processing**: Send metrics to Kinesis Data Firehose for custom transformation and analysis.

**How it works:**
You create a metric stream that filters which metrics to include/exclude and defines a destination (Kinesis Data Firehose). The stream then automatically sends metric updates.

```bash
# Example: Create a Metric Stream to a Kinesis Data Firehose
aws cloudwatch put-metric-stream \
    --name "my-app-metrics-stream" \
    --firehose-arn "arn:aws:firehose:us-east-1:123456789012:deliverystream/my-firehose-stream" \
    --role-arn "arn:aws:iam::123456789012:role/CloudWatch-Stream-Role" \
    --output-format "json" \
    --include-filters Namespace=MyApplication

# List all metric streams
aws cloudwatch list-metric-streams

# Stop a metric stream
aws cloudwatch stop-metric-streams --names "my-app-metrics-stream"

# Start a metric stream
aws cloudwatch start-metric-streams --names "my-app-metrics-stream"

# Delete a metric stream
aws cloudwatch delete-metric-streams --names "my-app-metrics-stream"
```

> **💡 CloudOps Tip**: Use Metric Streams instead of legacy polling methods (GetMetricData APIs) for exporting metrics. It's more scalable, cost-effective, and provides data with lower latency.

---

# Best Practices

- Use detailed monitoring for critical resources (1-minute intervals)
- Create actionable dashboards that group related metrics
- Implement custom metrics for application-specific monitoring
- Use dimensions consistently for effective filtering and grouping
- Apply metric math for derived metrics rather than calculating and storing them
- Enable anomaly detection for variable workloads with unpredictable patterns
- Limit high-resolution metrics to minimize costs
- Use consistent tagging for resources to improve metric organization
- Automate dashboard creation with Infrastructure as Code
- Monitor billing metrics in the AWS/Billing namespace to track costs

---

# Limitations and Quotas

- Up to 10 dimensions per metric
- Up to 30 metrics per PutMetricData request
- API rate limits: PutMetricData (1,000 TPS), GetMetricData (50 TPS)
- Maximum of 500 widgets per dashboard
- Up to 5,000 alarms per region per account
- Maximum of 30 metric math functions per GetMetricData operation

---

# Cost Considerations

- **Default Metrics**: Free
- **Custom Metrics**: $0.30 per metric per month (each unique metric name + dimension combination)
- **High-Resolution Metrics**: $0.30 per metric per month (for up to 3.6M data points)
- **API Requests**: First million API requests free, then $0.01 per 1,000 requests
- **Dashboards**: $3.00 per dashboard per month for the first 3 dashboards, then free
- **Metric Insights Queries**: $0.01 per query
- **Anomaly Detection**: $0.30 per anomaly detector per month
- **Metric Streams**: Pricing depends on the destination service (e.g., Kinesis Data Firehose, S3)

---

## References

- [Amazon CloudWatch User Guide](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/)
- [CloudWatch Metrics API Reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/)
- [CloudWatch Agent Configuration Reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html)
- [CloudWatch Anomaly Detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html)

---

**Next:** [CloudWatch Logs →](02-cloudwatch-logs.md)
