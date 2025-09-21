# Lab 4: CloudWatch Alarms and Automated Response

> **🎯 CloudMart Story:** CloudMart's monitoring is collecting metrics and logs, but you're still reactive to problems. Set up alerts to catch issues before customers notice and implement automated recovery.

> **⚠️ COST ALERT:** Uses FREE TIER alarm limits (10 alarms). SNS notifications free up to 1,000 emails/month.
> **⏱️ Estimated Time:** 35-45 minutes
> **💡 Difficulty:** Beginner

## CloudMart Story Context
**Previous:** Infrastructure monitoring and log collection active (Labs 1-3)
**Current:** Adding proactive alerting and automated recovery
**Next:** Chapter 2 will add security monitoring and compliance

## What You'll Build

- Email notification system for infrastructure problems
- Cost protection alerts
- Application error monitoring
- Automated recovery for EC2 instances

## Prerequisites

- Completed Labs 1-3 (Cost monitoring, metrics, and logs active)
- CloudMart web server running with monitoring
- Email address for alerts

## Lab Steps

### Step 1: Create SNS Topic for Notifications

1. **Navigate to SNS Console**
   - AWS Console → Simple Notification Service
   - Click "Create topic"

2. **Create CloudMart alert topic**
   ```
   Type: Standard
   Name: CloudMart-Alerts
   Display Name: CloudMart Infrastructure Alerts
   ```

3. **Create email subscription**
   - Click on the topic you just created
   - Click "Create subscription"
   
   ```
   Protocol: Email
   Endpoint: your-email@domain.com
   ```
   
   **Important:** Check your email and confirm the subscription!

### Step 2: Create Infrastructure Health Alarm

1. **Navigate to CloudWatch Alarms**
   - CloudWatch → Alarms → All alarms
   - Click "Create alarm"

2. **Create CPU utilization alarm**
   ```
   Metric: EC2 → CPUUtilization → Instance: CloudMart-WebServer-01
   
   Conditions:
   - Threshold Type: Static
   - Whenever CPUUtilization is: Greater than 80
   - For: 2 out of 3 datapoints within 15 minutes
   
   Actions:
   - Notification: CloudMart-Alerts
   - EC2 action: Recover this instance
   
   Name: CloudMart-Infrastructure-Health
   Description: CloudMart web server needs attention
   ```

### Step 3: Create Application Error Alarm

> **📋 Prerequisites Check:** This alarm uses metrics from Lab 3. If you see "Insufficient Data", complete Lab 3 metric filters first.

1. **Create application error alarm**
   ```
   Metric: CloudMart/Logs → ErrorCount
   
   Conditions:
   - Threshold Type: Static
   - Whenever ErrorCount is: Greater than 5
   - For: 2 out of 3 datapoints within 15 minutes
   
   Statistic: Sum
   Period: 5 minutes
   
   Notification: CloudMart-Alerts
   
   Name: CloudMart-Application-Errors
   Description: CloudMart application has errors
   ```

### Step 4: Create Cost Protection Alarm

1. **Create billing alarm**
   ```
   Metric: AWS/Billing → EstimatedCharges → Currency: USD
   
   Conditions:
   - Threshold Type: Static
   - Whenever EstimatedCharges is: Greater than 5.00
   - For: 1 out of 1 datapoints within 6 hours
   
   Notification: CloudMart-Alerts
   
   Name: CloudMart-Cost-Alert
   Description: CloudMart AWS costs exceeded $5
   ```

### Step 5: Add Alarm Widgets to Dashboard

1. **Open existing CloudMart Infrastructure Dashboard**
   - Navigate to CloudWatch → Dashboards
   - Open "CloudMart-Infrastructure-Monitor" (created in Lab 2)

2. **Add alarm status widget**
   
   **Add Widget: CloudMart Alarm Status**
   ```
   Widget Type: Alarm status
   Alarms:
   - CloudMart-Infrastructure-Health
   - CloudMart-Application-Errors
   - CloudMart-Cost-Alert
   Title: "CloudMart Alert Status"
   ```

### Step 6: Test Alarm System

1. **Connect to CloudMart instance**
   ```bash
   ssh -i cloudmart-keypair.pem ec2-user@YOUR-INSTANCE-PUBLIC-IP
   ```

2. **Test CPU alarm**
   ```bash
   echo "Testing CPU alarm..."
   yes > /dev/null &
   CPU_PID=$!
   echo "High CPU load started (PID: $CPU_PID)"
   echo "Check for email alert in 15 minutes"
   
   sleep 300  # 5 minutes
   kill $CPU_PID
   echo "Test completed"
   ```

3. **Test application error alarm**
   ```bash
   echo "Testing error alarm..."
   
   for i in {1..10}; do
       echo "$(date): ERROR: Payment failed - user_id=user$((RANDOM % 100)) reason=card_declined" | sudo tee -a /var/log/cloudmart.log
       sleep 1
   done
   echo "Error test completed - check CloudWatch metrics in 10 minutes"
   ```

## Verification Steps

### ✅ Test Your Setup

1. **Check all alarms created**
   - Navigate to CloudWatch → Alarms
   - Should see 3 alarms in "OK" state

2. **Verify SNS subscription**
   - Check email for subscription confirmation
   - Status should be "Confirmed"

3. **Test one alarm**
   - Run CPU test
   - Get email within 15 minutes

## Expected Costs

- **Total: $0.00** ✅ (Within free tier)
- CloudWatch Alarms: Free tier (10 alarms, using 3)
- SNS notifications: Free tier (1,000 emails/month)
- CloudWatch metrics: Free tier (existing from Lab 2)

## Next Lab Preview

**Lab 5** will implement CloudTrail for security monitoring, tracking all API calls and user activities in your CloudMart environment.

## Troubleshooting

**Alarms not triggering?**
- Check alarm conditions and thresholds
- Verify sufficient data points exist
- Ensure metrics are being published from CloudWatch Agent

**Email notifications not received?**
- Check spam folder
- Verify SNS subscription is "Confirmed"
- Check SNS topic permissions

**Auto-recovery not working?**
- Verify EC2 instance type supports recovery (most do)
- Check alarm is configured with EC2 action
- Monitor CloudWatch Events for recovery attempts

**Application error alarms showing "Insufficient Data"?**
- Verify metric filters exist from Lab 3
- Check log groups have data: CloudWatch → Logs → Log groups
- Ensure application is writing to `/var/log/cloudmart.log` (not application.log)
- Metric filters only process new log events after creation

**Billing alarms not showing data?**
- Billing metrics only available in us-east-1 region
- Ensure "Receive CloudWatch Billing Alerts" is enabled in Account Settings
- Wait up to 24 hours for first billing data to appear

---

> **🎯 CloudMart Checkpoint:** Your infrastructure now has proactive monitoring with email alerts. Problems will be detected automatically, and critical issues like instance failures will self-recover.
