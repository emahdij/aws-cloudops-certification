# Lab 5: CloudTrail Audit and Security Monitoring

> **🎯 CloudMart Story:** CloudMart is processing customer orders and payments. You need to know who's doing what in your AWS account. Set up audit logging to catch suspicious activity before it becomes a problem.

> **⚠️ COST ALERT:** Management events are free. S3 storage costs about $0.50/month for typical usage.
> **⏱️ Estimated Time:** 30-40 minutes
> **💡 Difficulty:** Beginner

## CloudMart Story Context
**Previous:** Monitoring and alerts working (Labs 1-4)
**Current:** Adding security audit trail to track API calls and spot problems
**Next:** Lab 6 will monitor configuration changes with AWS Config

## What You'll Build

- CloudTrail to log every API call
- Alerts for suspicious activities like root account usage
- Security dashboard to spot problems quickly
- Log analysis tools for investigating incidents

## Prerequisites

- Completed Labs 1-4 (CloudMart infrastructure running)
- Email address for security alerts

## Lab Steps

### Step 1: Create CloudTrail for CloudMart

1. **Navigate to CloudTrail Console**
   - AWS Console → CloudTrail
   - Click "Create trail"

2. **Configure the trail**
   ```
   Trail Name: CloudMart-Audit-Trail
   
   Storage Location:
   ✅ Create new S3 bucket
   - Bucket name: cloudmart-audit-logs-[your-account-id]
   - Log file prefix: audit/
   
   Additional Settings:
   ✅ Log file validation (detects tampering)
   ✅ Enable for all regions (REQUIRED for IAM events)
   ```

   > **📝 Important:** Replace [your-account-id] with your actual account ID. Bucket names must be globally unique.

   > **💡 S3 Lifecycle Policy:** We recommend enabling an S3 lifecycle policy to automatically delete logs older than 30 days to control costs. If you're not familiar with S3 lifecycle policies, skip this for now and continue with the lab.

3. **Configure CloudWatch Logs integration**
   ```
   CloudWatch Logs:
   ✅ Enabled
   - Log group: /cloudmart/cloudtrail
   - IAM Role: Create new → CloudMart-CloudTrail-Role
   ```

   **Why CloudWatch Logs:** This sends CloudTrail events to CloudWatch so we can create alarms and search logs in real-time.

4. **Choose events to log**
   ```
   Management Events:
   ✅ Read events
   ✅ Write events
   
   Data Events: ❌ Disabled (keeps costs low)
   Insights Events: ❌ Disabled (not needed)
   ```

5. **Create the trail**

### Step 2: Create Security Alerts

We'll create metric filters to catch suspicious activities and turn them into CloudWatch alarms.

1. **Navigate to CloudWatch Logs**
   - CloudWatch → Log groups → /cloudmart/cloudtrail

2. **Create root account usage filter**
   - Click "Metric filters" → "Create metric filter"
   ```
   Filter Pattern: { $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }
   
   Metric Details:
   - Filter Name: CloudMart-Root-Account-Usage
   - Metric Namespace: CloudMart/Security
   - Metric Name: RootAccountUsage
   - Metric Value: 1
   ```

   **What this catches:** Anyone using the root account (which you should never do in production).

3. **Create failed login attempts filter**
   ```
   Filter Pattern: { ($.errorCode = "SigninFailure") || ($.errorMessage = "Failed authentication") }
   
   Metric Details:
   - Filter Name: CloudMart-Failed-Logins
   - Metric Namespace: CloudMart/Security  
   - Metric Name: FailedLogins
   - Metric Value: 1
   ```

4. **Create IAM changes filter**
   ```
   Filter Pattern: { ($.eventName = CreatePolicy) || ($.eventName = DeletePolicy) || ($.eventName = AttachRolePolicy) || ($.eventName = DetachRolePolicy) }
   
   Metric Details:
   - Filter Name: CloudMart-IAM-Changes
   - Metric Namespace: CloudMart/Security
   - Metric Name: IAMChanges
   - Metric Value: 1
   ```

   **What this catches:** Any changes to IAM policies and roles.

> **📝 Note:** These filters only catch NEW events after you create them. They won't show historical data.

### Step 2.1: Generate Data for Your Metrics

Your metric filters won't have data until CloudTrail events match them. Let's generate some test events:

1. **Generate CloudTrail events**
   ```bash
   # Create some S3 activity (safe, generates CloudTrail events)
   aws s3 mb s3://cloudmart-test-$(date +%s) 2>/dev/null || echo "Bucket creation attempted"
   aws s3 ls > /dev/null 2>&1
   
   # Create an IAM policy (generates IAM change event)
   aws iam create-policy --policy-name CloudMart-Test-Policy --policy-document '{
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": "s3:GetObject",
       "Resource": "*"
     }]
   }' 2>/dev/null || echo "Policy creation attempted - may already exist"
   ```

2. **Wait for metrics to populate**
   - CloudTrail events take 5-15 minutes to appear in CloudWatch Logs
   - Metric filters process events and create data points
   - Go to CloudWatch → Metrics → CloudMart/Security to check

3. **Verify data is flowing**
   - CloudWatch → Log groups → /cloudmart/cloudtrail
   - Should see recent log streams with your activity
   - CloudWatch → Metrics → Browse → CloudMart/Security
   - Should see IAMChanges metric with data points

> **💡 Tip:** If you don't see metrics after 20 minutes, check that your CloudTrail is sending logs to the /cloudmart/cloudtrail log group.

### Step 3: Create Security Alarms

1. **Create root account usage alarm**
   ```
   Metric: CloudMart/Security → RootAccountUsage
   
   Conditions:
   - Threshold Type: Static
   - Whenever RootAccountUsage is: Greater than or equal to 1
   - For: 1 out of 1 datapoints within 5 minutes
   
   Actions:
   - Notification: CloudMart-Alerts (from Lab 4)
   
   Name: CloudMart-Root-Account-Usage
   Description: Root account activity detected
   ```

2. **Create failed login alarm**
   ```
   Metric: CloudMart/Security → FailedLogins
   
   Conditions:
   - Threshold Type: Static
   - Whenever FailedLogins is: Greater than 3
   - For: 2 out of 3 datapoints within 15 minutes
   
   Actions:
   - Notification: CloudMart-Alerts
   
   Name: CloudMart-Failed-Logins
   Description: Multiple failed login attempts detected
   ```

3. **Create IAM changes alarm**
   ```
   Metric: CloudMart/Security → IAMChanges
   
   Conditions:
   - Threshold Type: Static
   - Whenever IAMChanges is: Greater than or equal to 1
   - For: 1 out of 1 datapoints within 5 minutes
   
   Actions:
   - Notification: CloudMart-Alerts
   
   Name: CloudMart-IAM-Changes
   Description: IAM policy modifications detected
   ```

### Step 4: Create Security Dashboard

1. **Create security dashboard**
   ```
   Dashboard Name: CloudMart-Security-Monitor
   ```

2. **Add security widgets**

   **Widget 1: Security Events Overview**
   ```
   Widget Type: Number
   Metrics:
   - CloudMart/Security → RootAccountUsage (Sum, 24 hours)
   - CloudMart/Security → FailedLogins (Sum, 24 hours)
   - CloudMart/Security → IAMChanges (Sum, 24 hours)
   Title: "24-Hour Security Events"
   ```

   **Widget 2: Security Event Timeline**
   ```
   Widget Type: Line graph
   Metrics:
   - CloudMart/Security → FailedLogins (Statistic: Sum)
   - CloudMart/Security → IAMChanges (Statistic: Sum)
   Period: 1 hour
   Title: "Security Event Timeline"
   ```

   **Widget 3: Security Alarms**
   ```
   Widget Type: Alarm status
   Alarms:
   - CloudMart-Root-Account-Usage
   - CloudMart-Failed-Logins
   - CloudMart-IAM-Changes
   Title: "Security Alert Status"
   ```

### Step 5: Search Your Logs

1. **Navigate to Logs Insights**
   - CloudWatch → Logs → Insights
   - Select log group: /cloudmart/cloudtrail

2. **Run some useful queries**

   **Query 1: Recent API Activity**
   ```sql
   fields @timestamp, userIdentity.userName, eventName, sourceIPAddress
   | filter userIdentity.type != "AWSService"
   | sort @timestamp desc
   | limit 20
   ```

   **Query 2: IAM Activity**
   ```sql
   fields @timestamp, eventName, userIdentity.userName, requestParameters
   | filter eventName like /Policy|Role|User/
   | sort @timestamp desc
   | limit 20
   ```

3. **Save useful queries**
   - Click "Save" for queries you'll use again
   - Name them: "Recent Activity", "Failed Operations", "IAM Activity"

## Verification Steps

### ✅ Test Your Setup

1. **Check CloudTrail is logging**
   - CloudTrail console → Trails → CloudMart-Audit-Trail
   - Status should show "Logging"
   - S3 bucket should have log files

2. **Verify CloudWatch integration**
   - CloudWatch → Log groups → /cloudmart/cloudtrail
   - Should see log streams with data

3. **Check security alarms**
   - CloudWatch → Alarms
   - Should see 3 security alarms in "OK" or "INSUFFICIENT_DATA" state

4. **Test dashboard**
   - CloudMart-Security-Monitor should load
   - Widgets may show "No data" initially - this is normal

## Expected Costs

- **Total: $0.50-$1.00/month** ✅
- CloudTrail management events: Free
- S3 storage: $0.50/month (minimal log volume)
- CloudWatch Logs: Free tier covers typical usage
- SNS notifications: Free tier (1,000 emails/month)

## Cost Management & Next Steps

**💰 Current Monthly Costs:**
- Total: **$0.50-$1.00** ✅ (S3 storage only)


## Next Lab Preview

In **Lab 6**, you'll add AWS Config to monitor configuration compliance and catch configuration drift.

## Troubleshooting

**CloudTrail not logging?**
- Check trail status in CloudTrail console
- Ensure trail is enabled for all regions
- Verify S3 bucket policy allows CloudTrail writes

**No data in CloudWatch Logs?**
- CloudTrail takes 5-15 minutes to send logs
- Check IAM role has CloudWatch Logs permissions
- Verify log group exists: /cloudmart/cloudtrail

**Metric filters not working?**
- Filters only process NEW events after creation
- Generate some AWS API activity using Step 6
- Wait 15-20 minutes for first data points

**Security alarms stuck in INSUFFICIENT_DATA?**
- This is normal for new alarms
- Generate test events using Step 6 commands
- Wait 15-20 minutes for metrics to appear

**Log Insights queries return no results?**
- Ensure CloudTrail is sending logs to CloudWatch
- Check time range (try last 24 hours)
- Generate some API activity first

---

> **🎓 What You Accomplished:** CloudMart now has security monitoring that tracks every API call. You can detect threats in real-time and investigate incidents with detailed audit trails. This is your security safety net.
