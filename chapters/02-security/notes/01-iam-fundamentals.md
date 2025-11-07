# IAM Fundamentals

## Overview

IAM controls who can access AWS resources and what they can do. It provides authentication (who you are) and authorization (what you can do) for every AWS API call.

**Global Service**: IAM works across all regions. Changes propagate globally with eventual consistency.

---

## Part 1: AWS Accounts & Root User

### When You Create an AWS Account

1. AWS gives you unique 12-digit Account ID (e.g., 123456789012)
2. Root user created automatically (your email)
3. Root user has complete access - can't be restricted
4. You manually create IAM users inside this account

**Single Account**:
```
AWS Account 123456789012
├── Root User (unrestricted)
├── IAM Users: alice, bob
└── Resources: EC2, S3, Lambda
```

**AWS Organization** (multi-account for large companies):
```
Organization
├── Management Account: 111111111111
├── Production Account: 222222222222
└── Development Account: 333333333333
```

### Root User Security

Root user is dangerous:
- Complete access to everything
- Can delete entire account
- Cannot be restricted by ANY IAM policy

**What to do**:
1. Enable MFA immediately
2. Create admin IAM user for daily work
3. Lock root credentials in safe
4. Never create access keys for root
5. Only use root for account settings or closing account

> **Exam Alert**: Root user can't be restricted. First step on new account: MFA + create admin user + lock away root.

---

## Part 2: IAM Identities (Who Can Access)

### Three Types of Identities

**1. IAM User**
- Permanent identity for humans
- Has password (console) and/or access keys (CLI/API)
- Example: Developer "alice" who logs into console

**2. IAM Role**
- Temporary identity that gets assumed
- Credentials auto-rotate
- Used by: EC2 instances, Lambda functions, cross-account access
- Example: EC2 instance assumes role to access S3

**3. IAM Group**
- Collection of users
- Makes permission management easier
- **Important**: Groups CANNOT authenticate - only users and roles can

| Identity | Can Login/Authenticate? | Can Be in Policies? |
|----------|------------------------|---------------------|
| IAM User | ✅ Yes | ✅ Yes |
| IAM Role | ✅ Yes (via AssumeRole) | ✅ Yes |
| IAM Group | ❌ No credentials | ❌ No |

> **Exam Alert**: Groups can't be principals in policies because they can't authenticate.

---

## Part 3: IAM Policies (What They Can Do)

Policies are JSON documents that grant or deny permissions.

### Three Types of Policies

**1. Identity-Based Policy** (attached to user/group/role)
- Says: "This user/role can do X to resource Y"
- NO "Principal" element (because it's attached TO the identity)

**2. Resource-Based Policy** (attached to S3/KMS/Lambda)
- Says: "This resource allows Principal X to do Y"
- HAS "Principal" element (specifies WHO can access)

**3. Trust Policy** (attached to ROLES only)
- Says: "These entities can assume this role"
- HAS "Principal" element (specifies WHO can assume)
- **Only used for roles, not users or resources**

### Visual Breakdown

```
┌─────────────────────────────────────────────────────┐
│ Identity-Based Policy (on User/Group/Role)         │
│ - No Principal element                              │
│ - Defines what THIS identity can access             │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Resource-Based Policy (on S3/KMS/Lambda)           │
│ - Has Principal element                             │
│ - Defines WHO can access THIS resource              │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│ Trust Policy (on Roles ONLY)                        │
│ - Has Principal element                             │
│ - Defines WHO can assume THIS role                  │
└─────────────────────────────────────────────────────┘
```

---

## Part 4: Policy Structure & Examples

### Identity-Based Policy Example

Attached to user "alice" or role "WebAppRole":

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

Notice: NO "Principal" - it's attached to the identity, so AWS knows who it applies to.

### Resource-Based Policy Example

Attached to S3 bucket "my-bucket":

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::123456789012:user/alice"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

Notice: HAS "Principal" - the bucket needs to know WHO is allowed.

### Trust Policy Example

Attached to IAM role "EC2-S3-Role":

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

Notice: HAS "Principal" - the role needs to know WHO can assume it.

**Trust Policy Flow**:
1. EC2 instance calls `sts:AssumeRole` with role ARN
2. AWS checks trust policy: Does it allow ec2.amazonaws.com?
3. If yes: AWS returns temporary credentials
4. EC2 uses those credentials to access resources

> **Key Point**: Trust policies are ONLY for roles. They control who can "wear the costume" (assume the role). Identity-based policies control what the costume lets you do.

---

## Part 5: Policy Elements

Every policy has these elements:

| Element | Required? | Description | Example |
|---------|-----------|-------------|---------|
| **Version** | Yes | Always use `"2012-10-17"` | `"2012-10-17"` |
| **Statement** | Yes | Array of permissions | `[{...}, {...}]` |
| **Effect** | Yes | Allow or Deny | `"Allow"` |
| **Action** | Yes | What operations | `"s3:GetObject"` |
| **Resource** | Yes* | What AWS resource | `"arn:aws:s3:::bucket/*"` |
| **Principal** | Sometimes** | Who (only in resource/trust policies) | `{"AWS": "arn:..."}` |
| **Condition** | No | Extra restrictions | IP, MFA, time |

*Required in identity-based and resource-based policies  
**Required in resource-based and trust policies (NOT in identity-based)

### Action Examples

```json
"Action": "s3:GetObject"              // Single action
"Action": ["s3:GetObject", "s3:PutObject"]  // Multiple actions
"Action": "s3:*"                      // All S3 actions
"Action": "ec2:Describe*"             // All EC2 Describe actions
```

### Resource Examples

```json
"Resource": "arn:aws:s3:::my-bucket"              // The bucket itself
"Resource": "arn:aws:s3:::my-bucket/*"            // Objects in bucket
"Resource": "arn:aws:dynamodb:*:*:table/MyTable"  // DynamoDB table
"Resource": "*"                                   // All resources (careful!)
```

### Condition Examples

```json
{
  "Condition": {
    "IpAddress": {"aws:SourceIp": "203.0.113.0/24"},
    "Bool": {"aws:MultiFactorAuthPresent": "true"},
    "StringEquals": {"aws:RequestedRegion": "us-east-1"}
  }
}
```

---

## Part 6: How IAM Makes Decisions

### The Two Big Rules

**Rule 1: Default Deny**  
If no policy says "Allow", the answer is "Deny"

**Rule 2: Explicit Deny Always Wins**  
If any policy says "Deny", the answer is "Deny" (even if other policies say "Allow")

### Evaluation Order

```
Request comes in
    ↓
1. Is there an Explicit Deny anywhere? → YES: DENIED
    ↓ NO
2. Is there an Allow? → NO: DENIED (default)
    ↓ YES
3. ALLOWED
```

### Advanced: Multiple Policy Types

When you have SCPs (Organizations), Permission Boundaries, and regular policies:

```
1. Explicit Deny anywhere? → DENIED
2. SCP allows? → NO: DENIED
3. Permission Boundary allows? → NO: DENIED
4. Identity or Resource policy allows? → NO: DENIED (default)
5. ALLOWED
```

All layers must allow. One deny anywhere blocks everything.

> **Exam Alert**: Default is DENY. Need at least one Allow. Any Deny wins.

---

## Part 7: Cross-Account Access

When user in Account A needs resource in Account B, you need policies on BOTH sides.

### Example: Bob (Account A) accessing S3 bucket (Account B)

**Step 1 - Identity Policy in Account A** (on Bob's user):
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::account-b-bucket/*"
}
```
Says: "Bob is allowed to try accessing account-b-bucket"

**Step 2 - Resource Policy in Account B** (on bucket):
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::111111111111:user/bob"},
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::account-b-bucket/*"
}
```
Says: "This bucket allows Bob from account 111111111111"

Both must exist. If either is missing, access denied.

> **Exam Alert**: Cross-account access always needs BOTH sides to allow. Identity policy + Resource policy.

---

## Part 8: Roles Deep Dive

### What's Special About Roles?

Roles have TWO types of policies:

**1. Trust Policy** (who can assume the role)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

**2. Permission Policy** (what the role can do after being assumed)
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": "*"
  }]
}
```

Think of it like a costume:
- Trust policy = Who can wear the costume
- Permission policy = What powers the costume gives you

### EC2 Instance Roles

EC2 instances can't use passwords or access keys. They use roles via "instance profiles".

**Setup**:
1. Create role with trust policy allowing `ec2.amazonaws.com`
2. Attach permission policy to role
3. Create instance profile pointing to role
4. Attach instance profile to EC2

**What happens**:
- EC2 assumes the role automatically
- Gets temporary credentials
- Credentials available at metadata endpoint
- Credentials auto-rotate every few hours

```bash
# Applications query this for credentials
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/role-name
```

### Common Trust Policy Principals

```json
// AWS service
{"Service": "lambda.amazonaws.com"}

// Specific user
{"AWS": "arn:aws:iam::123456789012:user/john"}

// Entire account (any user/role in that account)
{"AWS": "arn:aws:iam::123456789012:root"}

// Federated user (SAML/OIDC)
{"Federated": "arn:aws:iam::123456789012:saml-provider/MyProvider"}
```

> **Exam Alert**: EC2 needs instance profile (can't attach role directly). Lambda attaches role directly.

---

## Part 9: Permission Boundaries & SCPs

These are LIMITS, not grants.

### Permission Boundaries

Maximum permissions a user/role can have.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["ec2:*", "s3:*", "lambda:*"],
    "Resource": "*"
  }]
}
```

If you give user this boundary, they can NEVER get IAM, RDS, or other permissions - even if you attach policies granting them.

**How it works**:
- User must have BOTH regular policy AND boundary allowing action
- If boundary doesn't allow, action is denied
- Boundary can't grant new permissions, only restrict

**Use case**: Let developers create their own policies, but boundary ensures they can't escalate to admin.

### Service Control Policies (SCPs)

Apply to entire AWS accounts in an Organization.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": ["iam:*", "organizations:*"],
    "Resource": "*"
  }]
}
```

Even account admin can't use IAM or Organizations if SCP denies it.

> **Exam Alert**: Boundaries and SCPs only limit. They can't grant permissions.

---

## Part 10: Managed vs Inline Policies

### Managed Policies

Standalone policy you can attach to multiple users/roles.

**AWS Managed** (created by AWS):
- `arn:aws:iam::aws:policy/AdministratorAccess`
- `arn:aws:iam::aws:policy/ReadOnlyAccess`
- `arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess`

**Customer Managed** (you create):
- Can attach to multiple identities
- Versioning supported (up to 5 versions)
- Can rollback to previous version

### Inline Policies

Embedded directly in one user/role.
- Can't reuse
- Deleted when user/role is deleted
- No versioning
- Use rarely

> **Exam Alert**: Prefer managed policies. Inline only for one-off, identity-specific permissions.

---

## Part 11: Production Examples

### Web App EC2 Role

EC2 instance needs Parameter Store and CloudWatch Logs:

**Trust Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

**Permission Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ssm:GetParameter*"],
      "Resource": "arn:aws:ssm:*:*:parameter/myapp/*"
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:*:*:log-group:/aws/ec2/myapp*"
    }
  ]
}
```

### Lambda Function Role

Lambda processing DynamoDB and reading secrets:

**Trust Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
```

**Permission Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem", "dynamodb:PutItem", "dynamodb:Query"],
      "Resource": "arn:aws:dynamodb:*:*:table/MyAppTable*"
    },
    {
      "Effect": "Allow",
      "Action": ["secretsmanager:GetSecretValue"],
      "Resource": "arn:aws:secretsmanager:*:*:secret:myapp/database/*"
    }
  ]
}
```

### Cross-Account Role with MFA

Dev account assuming role in prod account:

**Trust Policy** (in prod account role):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
    "Action": "sts:AssumeRole",
    "Condition": {
      "StringEquals": {"sts:ExternalId": "unique-secret-2024"},
      "Bool": {"aws:MultiFactorAuthPresent": "true"},
      "NumericLessThan": {"aws:MultiFactorAuthAge": "3600"}
    }
  }]
}
```

This says:
- Allow users from account 111111111111
- Only if they provide correct ExternalId
- Only if they have MFA
- Only if MFA was used within last hour

---

## Part 12: Troubleshooting Tools

### Policy Simulator

Tests policies without making real API calls.

**URL**: https://policysim.aws.amazon.com/

**CLI**:
```bash
# Test if user can do action
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/alice \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/file.txt
```

**Results**:
- `allowed` - Policy explicitly allows
- `explicitDeny` - Policy explicitly denies
- `implicitDeny` - No policy allows (default deny)

**Use when**:
- "Access Denied" errors - see which policy blocks
- Before deploying new policy - test it first
- Understanding permission boundaries - see combined effect

### IAM Access Analyzer

Four capabilities:

**1. External Access** - Finds resources shared outside account (public S3, cross-account roles)

**2. Unused Permissions** - Compares CloudTrail logs vs granted permissions, shows what's never used

**3. Policy Validation** - Real-time checks for syntax errors, security warnings, wildcards

**4. Policy Generation** - Analyzes CloudTrail, generates least-privilege policy from actual usage

```bash
# Create analyzer
aws accessanalyzer create-analyzer \
  --analyzer-name my-account-analyzer \
  --type ACCOUNT

# List findings
aws accessanalyzer list-findings \
  --analyzer-arn arn:aws:access-analyzer:us-east-1:123456789012:analyzer/my-account-analyzer
```

### CloudTrail Debugging

```bash
# Find AccessDenied errors
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --query 'Events[?contains(CloudTrailEvent, `AccessDenied`)]'

# Decode error messages
aws sts decode-authorization-message \
  --encoded-message "encoded-string" \
  --query DecodedMessage --output text | jq '.'
```

---

## Part 13: AWS Cognito (Mobile & Web App Authentication)

### What is Cognito?

Cognito provides authentication and authorization for mobile and web applications. It gives temporary AWS credentials to users who log in through external identity providers (Google, Facebook, Amazon) or create accounts directly.

**Two Main Services**:

**1. Cognito User Pools** - User directory for app users
- Sign-up and sign-in for app users
- Social login (Google, Facebook, Amazon, Apple)
- MFA, password reset, email verification
- Returns JWT tokens for API authorization

**2. Cognito Identity Pools** - Provides temporary AWS credentials
- Exchanges authentication tokens for AWS credentials
- Works with User Pools, SAML providers, social logins
- Returns temporary credentials via STS AssumeRoleWithWebIdentity
- Gives direct access to AWS services (S3, DynamoDB, etc.)

### The Flow

**Mobile app user wants to upload photo to S3**:

```
1. User logs in via Google
   ↓
2. Google returns authentication token
   ↓
3. App sends token to Cognito Identity Pool
   ↓
4. Identity Pool calls STS AssumeRoleWithWebIdentity
   ↓
5. STS returns temporary AWS credentials (AccessKeyId, SecretAccessKey, SessionToken)
   ↓
6. App uses credentials to upload directly to S3
```

**No backend server needed!** The mobile app talks directly to AWS services.

### User Pools vs Identity Pools

| Feature | User Pools | Identity Pools |
|---------|-----------|----------------|
| Purpose | Authenticate app users | Provide AWS credentials |
| Returns | JWT tokens | Temporary AWS credentials |
| Use for | Protecting your API | Accessing AWS services |
| Login methods | Email/password, social | Any auth provider |
| Example | User logs into your app | User uploads to S3 |

**Together**:
```
User Pools (Authentication)
    ↓ (JWT token)
Identity Pools (AWS Authorization)
    ↓ (Temporary credentials)
AWS Services (S3, DynamoDB, etc.)
```

### Identity Pool Permissions

When Cognito exchanges tokens for credentials, it assumes an IAM role. You define what authenticated users can do:

**Authenticated Role Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:PutObject", "s3:GetObject"],
    "Resource": "arn:aws:s3:::user-uploads/${cognito-identity.amazonaws.com:sub}/*"
  }]
}
```

Notice: `${cognito-identity.amazonaws.com:sub}` - each user gets their own folder in S3!

**Unauthenticated Role Policy** (for guest users):
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject"],
    "Resource": "arn:aws:s3:::public-content/*"
  }]
}
```

### When to Use Cognito

| Scenario | Use Cognito? |
|----------|--------------|
| Mobile app with user sign-up | ✅ Yes - User Pools |
| Mobile app needs S3/DynamoDB access | ✅ Yes - Identity Pools |
| Web app social login (Google, Facebook) | ✅ Yes - User Pools |
| Corporate employees accessing AWS accounts | ❌ No - Use IAM Identity Center |
| API authentication with OAuth2 | ✅ Yes - User Pools |
| IoT devices | ❌ No - Use IoT Core with certificates |

### Real-World Example

**Photo sharing app**:
1. Users sign up with email (User Pool)
2. Users log in and get JWT token
3. App exchanges JWT for AWS credentials (Identity Pool)
4. Users upload photos directly to S3
5. Each user can only access their folder: `s3://photos/${user-id}/*`

**No application servers needed** for uploads - mobile app talks directly to S3!

> **Exam Alert**: Cognito is for mobile/web apps, not for AWS employee access. Use Identity Pools when apps need direct AWS service access. Use User Pools for app authentication.

### Cognito vs IAM Identity Center

| Feature | Cognito | IAM Identity Center |
|---------|---------|---------------------|
| For | Mobile/web app users | AWS account access |
| User count | Millions | Hundreds/thousands |
| Authentication | Social, email, custom | Corporate SAML/OIDC |
| Returns | Temporary credentials | Console/CLI access |
| Use case | Customer-facing apps | Employee AWS access |

---

## Part 14: Common Issues
