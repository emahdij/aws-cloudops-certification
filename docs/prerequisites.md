# AWS Prerequisites

Before starting the AWS CloudOps course, you should understand these core concepts.

## AWS Global Infrastructure

### Regions
A region is a geographic area with multiple data centers:
- **Examples**: us-east-1 (Virginia), eu-west-1 (Ireland), ap-southeast-1 (Singapore)
- **Choose based on**: Where your users are, compliance requirements, cost
- **Each region is isolated**: Failures in one region don't affect others

### Availability Zones (AZs)
Availability zones are separate data centers within a region:
- **Purpose**: Protect against data center failures
- **Naming**: us-east-1a, us-east-1b, us-east-1c
- **Best practice**: Spread your applications across multiple AZs
- **Each region has**: At least 2 AZs, usually 3-6

### Edge Locations
These are content delivery points around the world:
- **Purpose**: Deliver content faster to users
- **Service**: Primarily used by CloudFront (AWS's CDN)
- **Scale**: 400+ locations globally

## Essential AWS Services
You don’t need to master these services yet, but it’s good to know what they do. We’ll use them in hands-on labs later, and sometimes you’ll see them before their full chapter. Just get familiar with their names and what they’re for.

### Compute Services
- **EC2**: Virtual servers (like renting a computer in the cloud)
- **Lambda**: Run code without managing servers
- **Auto Scaling**: Automatically add/remove servers based on demand

### Storage Services
- **S3**: Store files and data (like Dropbox but for applications)
- **EBS**: Hard drives that attach to EC2 instances
- **EFS**: Shared file storage that multiple servers can access

### Networking Services
- **VPC**: Your private network in AWS (like your own data center)
- **Route 53**: Domain name system (DNS) for websites
- **CloudFront**: Content delivery network to speed up websites

### Database Services
- **RDS**: Managed SQL databases (MySQL, PostgreSQL, etc.)
- **DynamoDB**: Fast NoSQL database for web applications
- **ElastiCache**: In-memory caching for better performance

### Security & Identity
- **IAM**: Control who can access what in your AWS account
- **KMS**: Manage encryption keys
- **CloudTrail**: Track who did what in your AWS account

## Core AWS Concepts

### Shared Responsibility Model
AWS and you share security responsibilities:
- **AWS handles**: Physical security, infrastructure, some managed services
- **You handle**: Your data, operating systems, applications, network settings
- **Think of it like**: AWS secures the building, you secure your apartment

### IAM (Identity and Access Management)
This is crucial for security:
- **Users**: Individual people or applications
- **Groups**: Collections of users with similar permissions
- **Roles**: Temporary permissions for AWS services
- **Policies**: Documents that define what actions are allowed
- **Golden rule**: Give the minimum permissions needed (principle of least privilege)

### Free Tier
AWS offers free usage for learning:
- **Always free**: Some services have permanent free limits
- **12 months free**: New accounts get extra free usage for a year
- **Trials**: Short-term free access to premium services
- **Important**: Set up billing alerts to avoid surprises

## Networking Fundamentals

### IP Addresses and Subnets
Basic networking concepts you'll encounter:
- **Private IP ranges**: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16
- **CIDR notation**: 10.0.0.0/16 means about 65,000 IP addresses
- **Public vs private**: Public IPs can reach the internet, private cannot
- **Subnets**: Smaller networks within your main network

### Security Groups vs NACLs
Two types of firewalls in AWS:
- **Security Groups**: Virtual firewalls attached to individual resources
- **NACLs**: Network firewalls attached to subnets
- **Key difference**: Security groups remember connections, NACLs don't

## Cost Management Basics

### Pricing Models
AWS offers different ways to pay:
- **On-demand**: Pay by the hour/second (most expensive, most flexible)
- **Reserved**: Commit to 1-3 years, get big discounts
- **Spot**: Bid on unused capacity (cheapest, can be interrupted)

### Monitoring Costs
Essential tools to avoid bill shock:
- **Billing Dashboard**: See current costs
- **Budgets**: Set spending alerts
- **Cost Explorer**: Understand where money goes
- **Free tier monitoring**: Track free tier usage

### Common Free Tier Limits
Know these to stay within free usage:
- **EC2**: 750 hours/month of t2.micro or t3.micro instances
- **S3**: 5 GB storage, 20,000 GET requests
- **CloudWatch**: 10 custom metrics
- **RDS**: 750 hours/month of db.t2.micro instances

## AWS Management Tools

### AWS Management Console
The web interface for AWS:
- **Good for**: Learning, one-off tasks, visual monitoring
- **Not great for**: Repetitive tasks, automation

### AWS CLI
Command line access to AWS:
- **Required for**: This course's hands-on labs
- **Good for**: Automation, scripting, advanced users
- **Installation**: Covered in the Environment Setup guide

---

**Ready for the next step?** Continue to [Environment Setup](../chapters/00-configuration/setting-up-environment.md) to configure your development environment.