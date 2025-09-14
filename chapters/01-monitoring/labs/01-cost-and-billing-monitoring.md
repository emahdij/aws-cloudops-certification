# Lab 1: Cost and Billing Monitoring for CloudMart

> **🎯 CloudMart Story:** CloudMart just got their AWS account. As the CloudOps engineer, your first priority is setting up cost monitoring to prevent surprise bills during the learning journey.

> **⚠️ COST ALERT:** This lab uses only FREE TIER resources. No charges expected.
> **⏱️ Estimated Time:** 20-30 minutes
> **💡 Difficulty:** Beginner

## CloudMart Story Context
**Previous:** New AWS account created
**Current:** Setting up financial guardrails and cost visibility
**Next:** Lab 2 will launch first infrastructure while cost monitoring tracks expenses

## What You'll Build

- Billing dashboard for real-time cost tracking
- Budget alerts to prevent overspending
- Cost anomaly detection
- Foundational monitoring that persists through all future labs

## Prerequisites

- New AWS account (free tier eligible)
- Root user access for billing configuration
- Email address for alerts

## Lab Steps

### Step 1: Enable Billing Alerts

> **📊 Important:** By default, billing metrics are NOT available in CloudWatch. You must enable billing alerts first to make billing data accessible for dashboards and monitoring.

1. **Sign in as root user** (required for billing)
   - Go to AWS Console → Account Settings
   - Navigate to "Billing Preferences"

2. **Enable CloudWatch billing metrics**
   ```
   ✅ Receive PDF Invoice By Email
   ✅ Receive Free Tier Usage Alerts  
   ✅ Receive CloudWatch Billing Alerts
   ```
   
   **Why this matters:** Checking "Receive CloudWatch Billing Alerts" enables billing metrics in CloudWatch. Without this, you won't see any cost data in Step 2.

3. **Set notification email**
   - Enter your email for alerts
   - Verify email address

### Step 2: Create CloudMart Cost Dashboard

1. **Navigate to CloudWatch**
   - Go to CloudWatch → Dashboards
   - Click "Create dashboard"

2. **Create "CloudMart-Cost-Monitor" dashboard**
   ```
   Dashboard Name: CloudMart-Cost-Monitor
   ```

3. **Add billing widgets**
   
   **Widget 1: Total Cost**
   ```
   Widget Type: Number
   Metric: Billing → "Total Estimated Charge"
   Currency: USD
   Statistic: Maximum
   Period: 6 hours
   ```

   **Widget 2: Daily Costs**
   ```
   Widget Type: Line
   Metric: Billing → "Total Estimated Charge"
   Statistic: Maximum
   Period: 1 day
   ```

### Step 3: Create Budget Alerts

1. **Navigate to AWS Budgets**
   - AWS Console → Billing → Budgets
   - Click "Create budget"

2. **Create "CloudMart-Monthly-Budget"**
   ```
   Budget Type: Cost budget
   Budget Name: CloudMart-Monthly-Budget
   Budget Amount: $10.00
   Time Period: Monthly
   ```

3. **Set up alerts**
   ```
   Alert 1: 
   - Threshold: 80% of budget ($8.00)
   - Email: your-email@domain.com
   
   Alert 2:
   - Threshold: 100% of budget ($10.00) 
   - Email: your-email@domain.com
   ```

### Step 4: Enable Cost Anomaly Detection

1. **Navigate to Cost Anomaly Detection**
   - Billing Console → Cost Anomaly Detection
   - Click "Create monitor"

2. **Configure anomaly monitor**
   ```
   Monitor Name: CloudMart-Anomaly-Monitor
   Monitor Type: AWS services
   Services: All AWS services
   ```

3. **Create subscription**
   ```
   Subscription Name: CloudMart-Anomaly-Alerts
   Threshold: $1.00
   Frequency: Daily
   Recipients: your-email@domain.com
   ```

## Verification Steps

### ✅ Test Your Setup

1. **Check dashboard loads**
   - Visit CloudMart-Cost-Monitor dashboard
   - Verify all widgets display data (may take 24 hours for first data)

2. **Verify budget**
   - Check budget shows $10 limit
   - Confirm alert emails configured

3. **Test anomaly detection**
   - Verify monitor is "Active"
   - Check subscription email


## Expected Costs

- **Total: $0.00** ✅
- CloudWatch metrics: Free tier (first 10 metrics)
- Budgets: Free (first 2 budgets)
- Anomaly Detection: Free tier
- SNS notifications: Free tier (first 1,000 emails)

## Cost Management & Next Steps

**💰 Current Monthly Costs:**
- Total: **$0.00** ✅ (All free tier resources)

**🧹 Cleanup Options:**
```bash
# Option 1: Keep everything for Chapter 2 (Recommended)
# - Cost monitoring will track all future resources
# - Essential foundation for all chapters
# - Cost: $0/month

# Option 2: Not applicable - no resources to clean up
# This lab creates only monitoring - no infrastructure
```

## Next Lab Preview

In **Lab 2**, you'll launch CloudMart's first EC2 instance and set up infrastructure monitoring. The cost monitoring you built today will track every resource you create.

## Troubleshooting

**Dashboard shows no data?**
- Wait 24 hours for first billing metrics
- Ensure billing alerts are enabled
- **Important:** Billing metrics are only available in us-east-1 region
- Check you're viewing correct region (us-east-1 for billing)

**Budget alerts not working?**
- Verify email address
- Check spam folder
- Ensure you have IAM permissions for billing