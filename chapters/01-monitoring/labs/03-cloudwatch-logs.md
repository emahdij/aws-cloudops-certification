# Lab 3: CloudWatch Logs and Application Monitoring

> **🎯 CloudMart Story:** CloudMart's web server is running but you can't see what's happening inside. You need log monitoring to catch errors, track user activity, and debug issues before customers complain.

> **⚠️ COST ALERT:** Uses FREE TIER log ingestion (5GB/month). Log configuration stays within limits.
> **⏱️ Estimated Time:** 25-35 minutes
> **💡 Difficulty:** Beginner

## CloudMart Story Context
**Previous:** Web server running with metrics
**Current:** Adding log collection and error tracking  
**Next:** Lab 4 will add alerts when logs show problems

## What You'll Build

- Log collection from web server and application
- Error tracking dashboard
- Log analysis for troubleshooting
- Production log retention

## Prerequisites

- Completed Lab 2 (CloudMart web server running)
- CloudWatch Agent installed and configured

## Lab Steps

### Step 1: Update CloudWatch Agent for Logs

1. **Connect to CloudMart web server**
   ```bash
   ssh -i cloudmart-keypair.pem ec2-user@YOUR-INSTANCE-PUBLIC-IP
   ```

2. **Update existing CloudWatch Agent config**
   ```bash
   sudo cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'EOF'
   {
       "logs": {
           "logs_collected": {
               "files": {
                   "collect_list": [
                       {
                           "file_path": "/var/log/httpd/access_log",
                           "log_group_name": "/cloudmart/apache/access",
                           "log_stream_name": "{instance_id}",
                           "timezone": "UTC"
                       },
                       {
                           "file_path": "/var/log/httpd/error_log",
                           "log_group_name": "/cloudmart/apache/error", 
                           "log_stream_name": "{instance_id}",
                           "timezone": "UTC"
                       },
                       {
                           "file_path": "/var/log/cloudmart.log",
                           "log_group_name": "/cloudmart/application",
                           "log_stream_name": "{instance_id}",
                           "timezone": "UTC"
                       }
                   ]
               }
           }
       },
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
   EOF
   ```

3. **Restart CloudWatch Agent**
   ```bash
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
       -a fetch-config \
       -m ec2 \
       -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
       -s
   ```

### Step 2: Create Application Logging

1. **Create application log script**
   ```bash
   cat > /home/ec2-user/app-logger.sh << 'EOF'
   #!/bin/bash
   
   LOG_FILE="/var/log/cloudmart.log"
   
   log_event() {
       timestamp=$(date '+%Y-%m-%d %H:%M:%S')
       message="[$timestamp] $1"
       echo "$message" | tee -a $LOG_FILE
   }
   
   echo "CloudMart Application Logger Started"
   echo "Press Ctrl+C to stop logging"
   echo "Logs will appear here and in $LOG_FILE"
   echo "----------------------------------------"
   
   # Generate 20 events then stop
   for i in {1..20}; do
       # Random events that happen in production
       case $((RANDOM % 6)) in
           0) log_event "INFO: User login successful - user_id=user$((RANDOM % 100))" ;;
           1) log_event "INFO: Product viewed - product_id=prod$((RANDOM % 50)) category=electronics" ;;
           2) log_event "INFO: Order placed - order_id=ord$((RANDOM % 1000)) amount=$((RANDOM % 500 + 50))" ;;
           3) log_event "WARN: Slow database query - duration=$((RANDOM % 3000 + 1000))ms" ;;
           4) log_event "ERROR: Payment failed - user_id=user$((RANDOM % 100)) reason=card_declined" ;;
           5) log_event "ERROR: Database connection timeout - retrying..." ;;
       esac
       
       sleep 3  # 3 seconds between events
   done
   
   echo "----------------------------------------"
   echo "Generated 20 log events. Script completed."
   EOF
   
   chmod +x /home/ec2-user/app-logger.sh
   ```

2. **Run application logging script**
   ```bash
   # Run script and watch logs appear
   /home/ec2-user/app-logger.sh
   ```

3. **Generate web traffic**
   ```bash
   # Open new terminal window/tab for traffic generation
   # Traffic to create access logs
   for i in {1..20}; do
       echo "Request $i: GET /"
       curl -s "http://localhost/" > /dev/null
       echo "Request $i: GET /nonexistent (404 error)"  
       curl -s "http://localhost/nonexistent" > /dev/null
       sleep 2
   done
   echo "Generated 20 web requests (10 successful, 10 404 errors)"
   ```

### Step 3: Set Up Log Retention

1. **Navigate to CloudWatch Logs**
   - AWS Console → CloudWatch → Logs → Log groups

2. **Set retention for cost control**
   
   For each log group, click on it and set retention:
   ```
   /cloudmart/apache/access → Retention: 30 days
   /cloudmart/apache/error → Retention: 90 days  
   /cloudmart/application → Retention: 30 days
   ```

   **Why these settings:**
   - Access logs: 30 days (normal traffic patterns)
   - Error logs: 90 days (debugging may need history)
   - Application logs: 30 days (business metrics)

### Step 4: Create Error Tracking

1. **Navigate to Log Groups**
   - CloudWatch → Logs → Log groups
   - Select `/cloudmart/application`

2. **Create error count metric**
   - Click "Metric filters" tab
   - Click "Create metric filter"
   
   ```
   Filter Pattern: ERROR
   Test Pattern: Use existing log data
   
   Metric Details:
   - Filter Name: CloudMart-Errors
   - Metric Namespace: CloudMart/Logs
   - Metric Name: ErrorCount
   - Metric Value: 1
   ```

   > **📝 Note:** Metric filters only process new log events after creation. Existing logs won't generate metrics retroactively.

3. **Create HTTP error metric**
   - Select `/cloudmart/apache/access` log group
   - Create metric filter
   
   ```
   Filter Pattern: [ip, identity, user, timestamp, request, status_code="404", size, referer, user_agent]
   
   Metric Details:
   - Filter Name: CloudMart-404-Errors
   - Metric Namespace: CloudMart/Logs
   - Metric Name: NotFoundErrors  
   - Metric Value: 1
   ```

### Step 5: Add Log Widgets to Existing Dashboard

1. **Open existing dashboard**
   - Go to CloudMart-Infrastructure-Monitor dashboard
   - Click "Edit"

2. **Add log-based widgets**

   **Widget: Application Errors**
   ```
   Widget Type: Number
   Metrics:
   - CloudMart/Logs → ErrorCount
   Period: 1 hour
   Statistic: Sum
   Title: "Application Errors (Last Hour)"
   ```

   **Widget: 404 Errors**
   ```
   Widget Type: Line graph
   Metrics:
   - CloudMart/Logs → NotFoundErrors
   Period: 5 minutes  
   Statistic: Sum
   Title: "Page Not Found Errors"
   ```

3. **Save dashboard**

### Step 6: Use Logs Insights for Troubleshooting

1. **Navigate to Logs Insights**
   - CloudWatch → Logs → Insights

2. **Create troubleshooting queries**

   **Query 1: Recent Errors**
   ```sql
   fields @timestamp, @message
   | filter @message like /ERROR/
   | sort @timestamp desc
   | limit 20
   ```

   **Query 2: Top Error Types**
   ```sql
   fields @message
   | filter @message like /ERROR/
   | stats count() by @message
   | sort count desc
   | limit 10
   ```

   **Query 3: Web Traffic Summary**
   ```sql
   fields @timestamp, @message
   | filter @logStream like /access/
   | stats count() by bin(5m)
   | sort @timestamp desc
   ```

3. **Save queries for future use**
   - Click "Save query" for each
   - Name them: "Recent Errors", "Error Summary", "Traffic Volume"

## Verification Steps

### ✅ Test Your Setup

1. **Check log groups exist**
   - 3 CloudMart log groups should be visible
   - Log streams should show recent data

2. **Verify metric filters**
   - Navigate to CloudWatch → Metrics → CloudMart/Logs
   - Should see ErrorCount and NotFoundErrors

3. **Test dashboard**
   - CloudMart-Infrastructure-Monitor should show new log widgets
   - Error counts should update as events occur

4. **Generate test errors**
   ```bash
   # Create errors to see in logs
   curl -s "http://localhost/missing-page" > /dev/null
   echo "[$(date '+%Y-%m-%d %H:%M:%S')] ERROR: Test error for monitoring" >> /var/log/cloudmart.log
   ```

## Expected Costs

- **Total: $0.00** ✅ (Within free tier)
- Log ingestion: Free tier (5 GB/month)
- Log storage: Minimal with retention policies
- Logs Insights: Free tier (5 GB scanned/month)
- Metric filters: No additional cost

## Cleanup and Cost Management

### 🧹 Keep Resources Running
**Important:** Keep your CloudMart infrastructure running for Lab 4! We'll add alerts for these logs and metrics.

### 💰 Current Total AWS Costs (Labs 1-3)
```
✅ Lab 1: $0.00 (Cost monitoring)
✅ Lab 2: $0.00 (Infrastructure monitoring)  
✅ Lab 3: $0.00 (Log monitoring)
────────────────────────────────────
📊 CloudMart Total: $0.00/month
```

## Next Lab Preview

In **Lab 4**, you'll create CloudWatch Alarms to automatically notify you when the metrics and logs show problems, including automated responses.

## Troubleshooting

**Logs not appearing?**
- Check CloudWatch Agent status: `sudo systemctl status amazon-cloudwatch-agent`
- Verify log file permissions: `ls -la /var/log/cloudmart.log`
- Check agent logs: `sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`

**Metric filters not working?**
- Test filter patterns with sample log data
- Ensure metric namespace is correct
- Wait 5-15 minutes for first data points

**Application logger stopped?**
- Script runs for 20 events then stops automatically
- To run again: `/home/ec2-user/app-logger.sh`
- Check log file: `tail -f /var/log/cloudmart.log`

---

> **🎓 Learning Checkpoint:** CloudMart now has log collection and error tracking. You can see what's happening inside your application and catch problems before users notice them.
