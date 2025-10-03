# Lab 6: AWS Config and Configuration Monitoring

> **🎯 CloudMart Story:** CloudMart's infrastructure is growing fast. Someone keeps changing security group rules and you have no idea who or when. Set up AWS Config to track every configuration change and catch problems before they break things.

> **⚠️ COST ALERT:** We'll use FREE TIER only. Recording specific resources to avoid charges.
> **⏱️ Estimated Time:** 35-45 minutes
> **💡 Difficulty:** Beginner

## CloudMart Story Context
**Previous:** Security logging catching API calls and threats (Lab 5)
**Current:** Adding configuration tracking to see who changed what and when
**Next:** Chapter 2 will add compliance rules and automated fixes

## What You'll Build

- Configuration recording for all AWS resources
- Alerts when someone changes security groups or IAM policies
- Configuration timeline to see what changed when
- Dashboard to spot configuration problems

## Prerequisites

- Completed Labs 1-5 (CloudMart infrastructure running)
- Email address for configuration alerts

## Lab Steps

### Step 1: Enable AWS Config

1. **Navigate to AWS Config Console**
   - AWS Console → Config
   - Click "Get started"

2. **Configure Config settings**
   ```
   Resource types to record: Specific resource types
   - Click "Choose resource types"
   - Select only: EC2 Instances, Security Groups
   - This keeps us in free tier (vs recording all resources)
   
   Recording frequency: Continuous recording
   - Records changes as they happen
   - Alternative: Daily recording
   
   Amazon S3 bucket: Create new bucket
   - Bucket name: cloudmart-config-[your-account-id]
   
   AWS Config service-linked role: Create new role
   ```

   > **📝 Important:** Replace [your-account-id] with your actual account ID. Config needs unique bucket names.

3. **Enable Config**
   - Review settings and click "Confirm"
   - Config will start recording all resource configurations

### Step 2: Create Configuration Rules

We'll create rules to catch common configuration problems.

**Create security group rule**
   - Config → Rules → "Add rule"
   - Click "AWS Managed Rules" (these are pre-built rules by AWS)
   - Search for: `restricted-ssh`
   - Click "Select" next to this rule
   ```
   Rule name: cloudmart-no-ssh-from-internet
   Description: Checks whether security groups disallow unrestricted incoming SSH traffic
   Trigger: When configuration changes
   Resources: Security Groups (AWS::EC2::SecurityGroup)
   ```

   **What this catches:** Security groups that allow SSH (port 22) from 0.0.0.0/0 (unrestricted access).

### Step 3: Test Configuration Monitoring

Let's create some configuration changes to see Config in action.

1. **Connect to CloudMart instance**
   ```bash
   ssh -i cloudmart-keypair.pem ec2-user@YOUR-INSTANCE-PUBLIC-IP
   ```

2. **Create test security group (will trigger rules)**
   ```bash
   # Create a security group that allows SSH from anywhere (bad practice)
   aws ec2 create-security-group \
       --group-name CloudMart-Test-SG \
       --description "Test security group for Config monitoring"
   
   # Add rule that allows SSH from anywhere (this will trigger our Config rule)
   SG_ID=$(aws ec2 describe-security-groups --group-names CloudMart-Test-SG --query 'SecurityGroups[0].GroupId' --output text)
   
   aws ec2 authorize-security-group-ingress \
       --group-id $SG_ID \
       --protocol tcp \
       --port 22 \
       --cidr 0.0.0.0/0
   
   echo "Created security group $SG_ID that allows SSH from anywhere"
   ```

3. **Wait for Config to evaluate**
   - Config evaluates rules within 10-20 minutes
   - Go to Config → Rules to see evaluation status
   - The `cloudmart-no-ssh-from-internet` rule should show "NON_COMPLIANT"

### Step 4: Create Configuration Dashboard

1. **Create Config dashboard**
   ```
   Dashboard Name: CloudMart-Configuration-Monitor
   ```

2. **Add Config widgets**

   **Widget 1: Configuration Items Recorded**
   ```
   Widget Type: Number
   Metrics:
   - Config → ConfigurationItemsRecorded
   Period: 1 hour
   Title: "CloudMart Configuration Items Recorded"
   ```
### Step 5: Explore Configuration History

1. **Navigate to Config → Resources**
   - Find your CloudMart-WebServer-01 instance
   - Click on it to see configuration timeline

2. **View configuration changes**
   - See all changes made to your EC2 instance
   - Timeline shows when changes happened and what changed
   - Useful for troubleshooting "what changed when things broke"

3. **Check compliance dashboard**
   - Config → Dashboard
   - Shows compliance status across all rules
   - Red items need attention

## Verification Steps

### ✅ Test Your Setup

1. **Check Config is recording**
   - Config → Resources should show your AWS resources
   - Configuration items should have recent timestamps

2. **Verify rules are evaluating**
   - Config → Rules should show 1 rule (`cloudmart-no-ssh-from-internet`)
   - Should show "NON_COMPLIANT" after you create the test security group

3. **Test configuration timeline**
   - Find your EC2 instance in Config → Resources
   - Should show configuration history

4. **Clean up test resources**
   ```bash
   # Remove the test security group
   aws ec2 delete-security-group --group-id $SG_ID
   echo "Deleted test security group"
   ```

## Expected Costs

- **Total: $0.00** ✅ (Free tier)
- AWS Config: Free tier covers 1,000 rule evaluations/month
- S3 storage: Minimal (few KB of configuration data)

## Complete Cleanup - Delete Everything

Since this is a learning lab, let's clean up all Config resources:

**🧹 Delete Config Setup:**
```bash
# 1. Delete Config rules
aws configservice delete-config-rule --config-rule-name cloudmart-no-ssh-from-internet

# 2. Turn off Config recorder
aws configservice stop-configuration-recorder --configuration-recorder-name default

# 3. Delete Config recorder and delivery channel
aws configservice delete-configuration-recorder --configuration-recorder-name default
aws configservice delete-delivery-channel --delivery-channel-name default
```

**Delete S3 bucket manually:**
- Go to S3 console
- Find your `cloudmart-config-[account-id]` bucket
- Empty the bucket, then delete it

**Why delete everything:**
- Config charges add up quickly in production
- This was just for learning how it works
- You can always set it up again when you need it

## Cost Management & Next Steps

**💰 Current Monthly Costs:**
- Total: **$0.00** ✅ (Everything deleted)

**When to use Config in real life:**
- Production environments where compliance matters
- When you need to track configuration changes
- For troubleshooting "what changed before it broke"

## Troubleshooting

**Config not recording resources?**
- Check service-linked role permissions
- Verify S3 bucket policy allows Config writes
- Make sure Config is enabled in your region

**Rules not evaluating?**
- Rules can take 10-20 minutes for first evaluation
- Check rule trigger conditions (configuration changes vs periodic)
- Verify resources match rule scope

**High Config costs?**
- Review what resource types you're recording
- Consider recording only critical resource types
- Adjust delivery frequency if needed

**Configuration timeline empty?**
- Config only tracks changes after it's enabled
- Make some configuration changes to see timeline populate
- Wait 10-15 minutes for changes to appear

---

> **🎓 What You Accomplished:** CloudMart now tracks every configuration change. You can see who changed what and when, catch configuration drift, and ensure your infrastructure stays secure and compliant.
