# AWS CloudOps Administrator Learning Guide

**Hands-on AWS CloudOps learning without endless videos. Build real skills, pass the certification.**

<!-- ## Quick Start

1. **New to AWS?** → [Beginner Path](docs/START_HERE.md) (3-5 hours to productive)
2. **Have AWS experience?** → [Roadmap](roadmap.md) (jump to gaps)
3. **Ready to build?** → [Projects](projects/) (showcase-ready deployments) -->

## What You'll Learn

- **Production AWS operations** (not just exam tricks)
- **Infrastructure as Code** with Terraform/CloudFormation
- **Monitoring & incident response** with CloudWatch, alarms, runbooks
- **Security best practices** with IAM, least-privilege, encryption
- **Cost optimization** and real-world operational patterns

## 🚀 Getting Started

**New to AWS concepts?** → [**AWS Prerequisites**](docs/prerequisites.md) (Core AWS concepts you should understand first)

**New to this guide?** → [**Environment Setup**](chapters/00-configuration/setting-up-environment.md) (Start here to configure AWS CLI and Terraform)

## 📖 How to Use This Guide

Throughout these chapters, you'll see AWS CLI commands and Terraform configurations used in examples. Don't worry about memorizing these commands—the goal is to become familiar with their patterns and usage. Each chapter concludes with hands-on labs where you can practice these commands and solidify your understanding in a real environment.

## 📚 Learning Path

### Available Now ✅
1. [**Environment Setup**](chapters/00-configuration/) - AWS CLI, Terraform, development environment
2. [**CloudWatch**](chapters/01-monitoring/README.md) - Complete metrics guide with examples
   - Includes [practice exam questions](chapters/01-monitoring/exam_questions/README.md) to test your knowledge

### Coming Soon 🚧
- **Security & Identity** - IAM, KMS, least-privilege
- **Networking** - VPC, security groups, endpoints
- **Compute** - EC2, Auto Scaling, containers
- **Storage** - S3, EBS, backups
- **Database** - RDS, DynamoDB operations
- **Deployment** - CloudFormation, automation
- **Cost & Advanced** - Budgets, optimization

<!-- ### Hands-on Projects
- [**Observability Baseline**](projects/observability-baseline/) - Start here for monitoring setup -->

<!-- ## 🎯 Track Progress

Use [Progress Tracker](progress_tracker/README.md) to mark completed chapters and labs. -->

## ⚠️ Cost Warning

Labs use real AWS resources. Always clean up after labs to avoid charges. See [cleanup script](resources/scripts/aws_resource_cleanup.sh).

---

**Ready for hands-on practice?** → Start with the [CloudWatch Labs](chapters/01-monitoring/labs/)