# Lab 2: CloudWatch Metrics and Infrastructure Dashboard

> **🎯 CloudMart Story:** CloudMart is ready to launch their first web server. You need to establish baseline infrastructure monitoring before any traffic hits the application.

> **⚠️ COST ALERT:** Uses FREE TIER resources only. One t2.micro EC2 instance included in free tier.
> **⏱️ Estimated Time:** 30-60 minutes
> **💡 Difficulty:** Beginner

## What You'll Build

- CloudMart web server (t2.micro EC2)
- Infrastructure monitoring dashboard
- Custom application metrics
- Foundation for all future monitoring labs

## Lab Steps

### Step 1: Launch CloudMart Web Server

1. **Navigate to EC2 Console**
   - AWS Console → EC2
   - Click "Launch Instance"

2. **Configure CloudMart instance**
   ```
   Name: CloudMart-WebServer-01
   AMI: Amazon Linux 2023 (Free tier eligible)
   Instance Type: t2.micro (Free tier eligible)
   Key Pair: Create new → "cloudmart-keypair"
   ```

3. **Security Group configuration**
   ```
   Security Group Name: CloudMart-WebServer-SG
   Rules:
   - HTTP (80): 0.0.0.0/0
   - HTTPS (443): 0.0.0.0/0  
   ```


4. **Launch instance**
   - Review configuration
   - Click "Launch instance"
   - Download key pair (save securely)

### Step 2: Connect and Set Up Web Server

1. **Connect to your instance**
   ```bash
   # Replace with your instance's public IP
   ssh -i cloudmart-keypair.pem ec2-user@YOUR-INSTANCE-PUBLIC-IP
   ```

2. **Update and install Apache web server  the system**
   ```bash
   sudo yum update -y
   sudo yum install -y httpd
   sudo systemctl start httpd
   sudo systemctl enable httpd
   ```

3. **Create CloudMart homepage**
   ```bash
   sudo tee /var/www/html/index.html > /dev/null << 'EOF'
    <!DOCTYPE html>
    <html>
    <head>
        <title>CloudMart - E-Commerce Platform</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; background: #f5f5f5; }
            .container { max-width: 800px; margin: 0 auto; background: white; padding: 30px; border-radius: 8px; }
            .header { color: #2196F3; margin-bottom: 20px; }
            .status { background: #4CAF50; color: white; padding: 15px; border-radius: 5px; margin: 20px 0; }
            .info { background: #e3f2fd; padding: 15px; border-radius: 5px; margin: 15px 0; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1 class="header">🛒 CloudMart E-Commerce Platform</h1>
            <div class="status">✅ Web Server Online - Ready for Monitoring</div>
            
            <div class="info">
                <h3>🔍 Monitoring Lab Environment</h3>
                <p>This CloudMart instance will be configured for monitoring with:</p>
                <ul>
                    <li>EC2 Instance metrics collection</li>
                    <li>CloudWatch Agent for detailed monitoring</li>
                    <li>Custom application metrics</li>
                    <li>Log collection setup</li>
                </ul>
            </div>
            </div>
        </div>
    </body>
    </html>
    EOF
   ```

4. **Test the web server**
   - Open browser and visit your instance's public IP
   - You should see the CloudMart homepage

### Step 3: Create CloudMart Infrastructure Dashboard

1. **Navigate to CloudWatch Dashboards**
   - CloudWatch → Dashboards
   - Click "Create dashboard"

2. **Create dashboard**
   ```
   Dashboard Name: CloudMart-Infrastructure-Monitor
   ```

3. **Add infrastructure widgets**

   **Widget 1: EC2 Instance Health**
    ```
    Widget Type: Line graph
    Metrics:
    - AWS/EC2 → CPUUtilization → Instance: CloudMart-WebServer-01
    - AWS/EC2 → StatusCheckFailed → Instance: CloudMart-WebServer-01
    Period: 5 minutes
    Title: "CloudMart Web Server Health"
    ```


   **Widget 2: Network Performance**
   ```
   Widget Type: Line graph  
   Metrics:
   - AWS/EC2 → NetworkIn → Instance: CloudMart-WebServer-01
   - AWS/EC2 → NetworkOut → Instance: CloudMart-WebServer-01
   Period: 5 minutes
   Title: "CloudMart Network Traffic"
   ```

   **Widget 3: Storage Performance**
   ```
   Widget Type: Line graph
   Metrics:
   - AWS/EC2 → EBSReadBytes → Instance: CloudMart-WebServer-01  
   - AWS/EC2 → EBSWriteBytes → Instance: CloudMart-WebServer-01
   Period: 5 minutes
   Title: "CloudMart Storage I/O"
   ```

### Step 4: Set Up CloudWatch Permissions

Before installing the CloudWatch Agent, your EC2 instance needs permission to send metrics to CloudWatch.

1. **Create IAM role for EC2**
   - Go to IAM → Roles → Create role
   - Select "AWS service" → "EC2" → Next
   - Search and select: `CloudWatchAgentServerPolicy`
   - Click Next → Next
   - Role name: `CloudMart-EC2-CloudWatch-Role`
   - Create role

2. **Attach role to your EC2 instance**
   - Go to EC2 → Instances → Select your CloudMart-WebServer-01
   - Actions → Security → Modify IAM role
   - Select: `CloudMart-EC2-CloudWatch-Role`
   - Update IAM role


### Step 5: Install CloudWatch Agent

Now we'll add detailed monitoring to see CPU, memory, and disk usage.

1. **Download CloudWatch Agent**
   ```bash
   wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
   ```

2. **Install the agent**
   ```bash
   sudo rpm -U ./amazon-cloudwatch-agent.rpm
   ```

3. **Create CloudWatch Agent config file**
   ```bash
   # Create the config directory
   sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/

# Create a config file for metrics

Put the following JSON config into `/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json`:

```json
{
    "metrics": {
        "namespace": "CloudMart/Application",
        "metrics_collected": {
            "disk": {
                "measurement": ["used_percent"],
                "metrics_collection_interval": 300,
                "resources": ["/"]
            },
            "mem": {
                "measurement": ["mem_used_percent"],
                "metrics_collection_interval": 300
            }
        }
    }
}
```

   **What this does:**
   - **Memory metrics**: Shows RAM usage percentage (custom metric)
   - **Disk metrics**: Shows how full the disk is (custom metric)
   - **Custom namespace**: Groups all metrics under "CloudMart/Application"
   - **5-minute intervals**: Collects data every 5 minutes (300 seconds)

   **📊 Metric Types:**
   - **Free AWS metrics**: CPU usage is automatically collected by AWS (no cost)
   - **Custom metrics**: Memory and disk usage count as 2 custom metrics (uses 2/10 of your free tier limit)

4. **Start the CloudWatch Agent**
   ```bash
   # Start the agent with our config
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
       -a fetch-config \
       -m ec2 \
       -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
       -s
   ```

5. **Check if it's working**
   ```bash
   # Check agent status
   sudo systemctl status amazon-cloudwatch-agent
   
   # Check for any errors in the logs
   sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
   ```

### Step 6: Add Metrics to Dashboard

1. **Return to dashboard**
   - Go back to CloudMart-Infrastructure-Monitor
   - Click "Edit"

2. **Add custom metric widgets**

   **Widget 4: Memory Usage**
   ```
   Widget Type: Gauge
   Metrics:
   - CloudMart/Application → mem_used_percent → Instance: CloudMart-WebServer-01
   Title: "CloudMart Memory Usage"
   Y-axis: 0-100
   ```

   **Widget 5: Disk Usage**
   ```
   Widget Type: Gauge  
   Metrics:
   - CloudMart/Application → disk_used_percent → Instance: CloudMart-WebServer-01
   Title: "CloudMart Disk Usage"
   Y-axis: 0-100
   ```

## Verification Steps

### ✅ Test Your Setup

1. **Verify web server**
   - Visit your instance's public IP in browser
   - Should see CloudMart homepage

2. **Check dashboard**
   - All widgets showing data (may take 5-15 minutes)
   - No error states


## What You Built for CloudMart

```
🏢 CloudMart Infrastructure - Week 1 Complete
├── 🖥️  Production Web Server
│   ├── t2.micro EC2 with CloudMart application
│   ├── Enhanced monitoring (CPU, memory, disk)
│   └── Custom application health metrics
├── 📊 Infrastructure Dashboard
│   ├── System performance metrics
│   ├── Network and storage monitoring  
│   ├── Application health status
│   └── Business metrics (order rate)
└── 🔍 Monitoring Foundation
    ├── CloudWatch Agent configured
```

## Expected Costs

- **Total: $0.00** ✅ (Free tier)
- EC2 t2.micro: Free tier (750 hours/month)
- CloudWatch metrics: Free tier (10 custom metrics)
- CloudWatch Agent: No additional cost
- Data transfer: Free tier

## Cleanup and Cost Management

### 🧹 Keep Resources Running
**Important:** Keep your CloudMart web server running for Lab 3! We'll build on this infrastructure.


> **💡 Metric Cost Tip:** AWS automatically provides CPU, network, and storage metrics for free. Memory and disk metrics from CloudWatch Agent count as custom metrics (2 used so far).

## Next Lab Preview

In **Lab 3**, you'll centralize CloudMart's application logs and set up log-based monitoring for errors and performance issues.