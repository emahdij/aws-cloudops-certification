# Setting Up Your AWS Development Environment

This guide will help you set up your local development environment for working with AWS.

## 🎯 Learning Objectives

By the end of this setup, you will have:
- A fully configured AWS CLI with proper authentication
- Terraform installed and configured for AWS
- Understanding of AWS security best practices
- Multiple environment profiles configured
- Verification tests to ensure everything works correctly

## 📊 Overview

To effectively work with AWS, you need to set up:

1. **AWS Command Line Interface (CLI)**: For interacting with AWS services via command line
2. **Terraform**: For infrastructure as code deployment (essential for this course)
3. **AWS credentials**: For secure access to your AWS resources
4. **Development tools**: Additional utilities for enhanced productivity

## 📋 Prerequisites

Before starting this setup, ensure you have:

- **AWS Account**: Free tier is sufficient for practice (with billing alerts configured)
- **Administrative Access**: To your local machine for installing software
- **Command Line Knowledge**: Basic familiarity with terminal/command prompt
- **Internet Connection**: Stable connection for downloading tools and accessing AWS
- **Text Editor**: VS Code, Vim, or your preferred editor for configuration files
- **Git** (optional): For version control of your infrastructure code

## 🔧 Required Dependencies

The following tools are required for this AWS CloudOps preparation course:

### Core Requirements
- **AWS CLI v2**: Latest version (v2.x recommended)
- **Terraform**: Version 1.0+ (we'll use latest stable)
- **Git**: For version control
- **jq**: JSON processor for AWS CLI output parsing

### Optional but Recommended
- **AWS Session Manager Plugin**: For secure EC2 access
- **kubectl**: If working with EKS
- **Docker**: For containerized applications
- **Python 3.8+**: For custom scripts and AWS SDK

---

## AWS CLI Setup

The AWS Command Line Interface (CLI) is a unified tool to manage your AWS services from the terminal.

### Installation by Operating System

<details>
<summary><b>🍎 macOS Installation</b></summary>

**Option 1: Using Homebrew (Recommended)**

```bash
# Install Homebrew if you don't have it
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install AWS CLI
brew install awscli

# Verify installation
aws --version
```

**Option 2: Using the official installer**

```bash
# Download the installer
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"

# Run the installer
sudo installer -pkg AWSCLIV2.pkg -target /

# Verify installation
aws --version
```
</details>

<details>
<summary><b>🐧 Linux Installation</b></summary>

**For Debian/Ubuntu-based distributions:**

```bash
# Update package lists
sudo apt-get update

# Install required dependencies
sudo apt-get install -y unzip curl

# Download the AWS CLI installation bundle
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Unzip the installer
unzip awscliv2.zip

# Run the install script
sudo ./aws/install

# Verify installation
aws --version

# Clean up
rm -rf aws awscliv2.zip
```

**For Amazon Linux/RHEL/CentOS:**

```bash
# Download the AWS CLI installation bundle
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"

# Install unzip if needed
sudo yum install -y unzip

# Unzip the installer
unzip awscliv2.zip

# Run the install script
sudo ./aws/install

# Verify installation
aws --version

# Clean up
rm -rf aws awscliv2.zip
```
</details>

<details>
<summary><b>🪟 Windows Installation</b></summary>

**Option 1: Using the MSI installer (Recommended)**

1. Download the AWS CLI MSI installer from [AWS CLI Download Page](https://awscli.amazonaws.com/AWSCLIV2.msi)
2. Run the downloaded MSI installer and follow the on-screen instructions
3. Open Command Prompt or PowerShell and verify installation: `aws --version`

**Option 2: Using Windows Package Manager (winget)**

```powershell
# Install AWS CLI using winget
winget install -e --id Amazon.AWSCLI

# Verify installation
aws --version
```
</details>

---

## IAM Setup for AWS CLI

Before configuring the AWS CLI, you need to set up proper IAM credentials following security best practices.

### Creating an IAM User for CLI Access

<details>
<summary><b>👤 Step-by-step IAM user creation</b></summary>

1. **Sign in to the AWS Management Console**
   - Go to the [AWS Management Console](https://console.aws.amazon.com/)
   - Sign in with your root account credentials (but only for this setup)

2. **Navigate to IAM**
   - Search for "IAM" in the services search bar
   - Click on "IAM" (Identity and Access Management)

3. **Create a new IAM user**
   - In the navigation pane, click "Users"
   - Click "Add users"
   - Enter a username (e.g., "cli-user")
   - Click "Next: Permissions"

4. **Assign permissions**
   - Choose "Attach existing policies directly"
   - For CloudOps practice, you could attach:
     - "ReadOnlyAccess" for safe exploration
     - "AmazonEC2FullAccess" for EC2 administration
     - "AmazonS3FullAccess" for S3 management
     - Or create a custom policy based on your needs
   - **Important:** Follow the principle of least privilege


5. **Review and create**
   - Review the user details and permissions
   - Click "Create user"

6. **Secure the credentials**
   - **Important:** This is the only time AWS will show you the secret access key
   - Download the .csv file or copy both the access key ID and secret access key
   - Store them securely (password manager recommended)
</details>

### IAM Security Best Practices

- **Never use root account credentials** for programmatic access
- **Create individual IAM users** for each person needing access
- **Use IAM roles** for applications running on EC2 instances
- **Implement least privilege** - grant only the permissions needed
- **Enable MFA** for your root account and IAM users
- **Rotate access keys** regularly (every 90 days recommended)
- **Remove unused credentials** to reduce security risks

---

## AWS CLI Configuration

After installing AWS CLI and creating IAM credentials, you need to configure the CLI:

### Basic Configuration

```bash
aws configure
```

You'll be prompted to enter:
- AWS Access Key ID
- AWS Secret Access Key
- Default region name (e.g., us-east-1)
- Default output format (json recommended)

### Testing Your AWS CLI Configuration

After configuration, verify that your credentials work correctly with these test commands:

<details>
<summary><b>🧪 Essential AWS CLI tests</b></summary>

```bash
# Get your current identity
aws sts get-caller-identity

# List your S3 buckets
aws s3 ls

# List EC2 instances (if you have permission)
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,AZ:Placement.AvailabilityZone}' --output table

# Check your IAM permissions
aws iam get-user

# Test with a dry run (no actual changes)
aws ec2 start-instances --instance-ids i-abcd1234 --dry-run
```
</details>

---

## Managing Multiple AWS Credentials

Most AWS professionals work with multiple accounts or roles. Here's how to manage different credentials:

### Using Named Profiles

<details>
<summary><b>🔄 Setting up and using named profiles</b></summary>

```bash
# Create a named profile
aws configure --profile dev-account

# Use a specific profile for a command
aws s3 ls --profile dev-account

# Set a profile as default for current session
export AWS_PROFILE=dev-account  # Linux/macOS
set AWS_PROFILE=dev-account     # Windows Command Prompt
$env:AWS_PROFILE="dev-account"  # Windows PowerShell
```

**View and edit your profiles manually:**
- In Linux/macOS: `~/.aws/config` and `~/.aws/credentials`
- In Windows: `%USERPROFILE%\.aws\config` and `%USERPROFILE%\.aws\credentials`

**Example config file with multiple profiles:**

```ini
# ~/.aws/credentials
[default]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

[dev]
aws_access_key_id=AKIAI44QH8DHBEXAMPLE
aws_secret_access_key=je7MtGbClwBF/2Zp9Utk/h3yCo8nvbEXAMPLEKEY

[prod]
aws_access_key_id=AKIAIOSFODNN7EXAMPLE2
aws_secret_access_key=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY2
```

```ini
# ~/.aws/config
[default]
region=us-east-1
output=json

[profile dev]
region=us-west-2
output=json

[profile prod]
region=eu-west-1
output=json
```
</details>

### Advanced Authentication (Optional)

For this initial setup, we'll focus on basic AWS account configuration. More advanced authentication methods like **role assumption** and **AWS SSO** will be covered in later chapters when we discuss security and multi-account strategies.

> **📝 Note**: Advanced topics like IAM role assumption, cross-account access, and AWS SSO integration are important for production environments but add complexity that's not needed for initial learning. We'll cover these in detail in Chapter 2 (Security) and Chapter 7 (Advanced Topics).

---

## Environment Variables for AWS

Instead of storing credentials in the AWS CLI config, you can use environment variables:

<details>
<summary><b>🌐 Setting environment variables on different platforms</b></summary>

### For Linux/macOS

```bash
# Add to your .bashrc or .zshrc
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"

# For temporary session token
export AWS_SESSION_TOKEN="your-session-token"
```

### For Windows (PowerShell)

```powershell
$env:AWS_ACCESS_KEY_ID="your-access-key"
$env:AWS_SECRET_ACCESS_KEY="your-secret-key"
$env:AWS_DEFAULT_REGION="us-east-1"

# For temporary session token
$env:AWS_SESSION_TOKEN="your-session-token"
```

### For Windows (Command Prompt)

```cmd
set AWS_ACCESS_KEY_ID=your-access-key
set AWS_SECRET_ACCESS_KEY=your-secret-key
set AWS_DEFAULT_REGION=us-east-1

REM For temporary session token
set AWS_SESSION_TOKEN=your-session-token
```

</details>

---

## Terraform Setup

Terraform is an infrastructure as code (IaC) tool that allows you to build, change, and version infrastructure safely and efficiently.

### Installation by Operating System

<details>
<summary><b>🍎 macOS Terraform Installation</b></summary>

**Using Homebrew (Recommended)**

```bash
# Install Terraform
brew install terraform

# Verify installation
terraform -version
```

**Manual Installation**

```bash
# Download latest version
curl -O https://releases.hashicorp.com/terraform/1.4.6/terraform_1.4.6_darwin_amd64.zip

# Extract the binary
unzip terraform_1.4.6_darwin_amd64.zip

# Move to a directory in your PATH
sudo mv terraform /usr/local/bin/

# Verify installation
terraform -version

# Clean up
rm terraform_1.4.6_darwin_amd64.zip
```
</details>

<details>
<summary><b>🐧 Linux Terraform Installation</b></summary>

**For Debian/Ubuntu-based distributions:**

```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install
sudo apt update && sudo apt install terraform

# Verify installation
terraform -version
```

**For Amazon Linux/RHEL/CentOS:**

```bash
# Install required packages
sudo yum install -y yum-utils

# Add HashiCorp repository
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo

# Install Terraform
sudo yum -y install terraform

# Verify installation
terraform -version
```
</details>

<details>
<summary><b>🪟 Windows Terraform Installation</b></summary>

**Option 1: Using Chocolatey**

```powershell
# Install Chocolatey if you don't have it (run as Administrator)
Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

# Install Terraform
choco install terraform

# Verify installation
terraform -version
```

**Option 2: Manual Installation**

1. Download the latest version from [Terraform Downloads](https://www.terraform.io/downloads.html)
2. Extract the zip file
3. Move the terraform.exe to a directory in your PATH
4. Verify installation in Command Prompt or PowerShell: `terraform -version`
</details>

### Terraform Configuration for AWS

<details>
<summary><b>🏗️ Setting up Terraform with AWS</b></summary>

Create a basic Terraform provider configuration:

```bash
# Create a directory for your Terraform project
mkdir ~/terraform-aws && cd ~/terraform-aws

# Create main.tf file
cat > main.tf << 'EOL'
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"  # Change to your preferred region
  
  # Option 1: Use default profile
  # profile = "default"
  
  # Option 2: Use named profile
  # profile = "dev-account"
  
  # Option 3: Use environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY)
  
  # Option 4: Assume role for cross-account access
  # assume_role {
  #   role_arn = "arn:aws:iam::123456789012:role/TerraformExecutionRole"
  #   session_name = "terraform-session"
  # }
}
EOL

# Initialize Terraform
terraform init
```

**Testing your Terraform AWS Configuration:**

Create a test file to verify access:

```bash
cat > test.tf << 'EOL'
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "user_id" {
  value = data.aws_caller_identity.current.user_id
}

output "arn" {
  value = data.aws_caller_identity.current.arn
}
EOL

# Run terraform plan to verify access
terraform plan
```

If the plan runs successfully without authentication errors, your Terraform AWS configuration is working correctly.
</details>

---

## Verifying Your Setup

Let's verify everything works correctly:

### Quick Verification Checklist

✅ **AWS CLI Installation**: `aws --version` shows AWS CLI version 2.x  
✅ **Terraform Installation**: `terraform -version` shows Terraform version 1.0+  
✅ **AWS Authentication**: `aws sts get-caller-identity` returns your account info  
✅ **Basic AWS Access**: `aws s3 ls` and `aws ec2 describe-regions` work without errors  
✅ **Terraform AWS Provider**: `terraform init` and `terraform plan` work with AWS provider  

### Comprehensive AWS CLI Verification

<details>
<summary><b>🔍 Essential AWS CLI tests</b></summary>

```bash
# 1. Verify your identity
echo "=== Testing AWS Identity ==="
aws sts get-caller-identity

# 2. Test basic read access
echo "=== Testing Basic Access ==="
aws ec2 describe-regions --output table --query 'Regions[0:5].[RegionName,Endpoint]'
aws s3 ls

# 3. Test service-specific access
echo "=== Testing Service Access ==="
# EC2
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId,State.Name,InstanceType]' --output table

# S3
aws s3api list-buckets --query 'Buckets[*].[Name,CreationDate]' --output table

# CloudWatch
aws cloudwatch list-metrics --namespace AWS/EC2 --query 'Metrics[0]' --output json

# 4. Test your permissions
echo "=== Testing IAM Permissions ==="
aws iam get-user
aws iam list-attached-user-policies --user-name $(aws iam get-user --query 'User.UserName' --output text)

# 5. Check AWS limits (service quotas)
echo "=== Testing Service Quotas ==="
aws service-quotas list-service-quotas --service-code ec2 --query 'Quotas[0:5]'
```
</details>

### Terraform AWS Provider Verification

<details>
<summary><b>🏗️ Terraform verification tests</b></summary>

```bash
# Create a test directory
mkdir -p ~/aws-terraform-test && cd ~/aws-terraform-test

# Create a test file
cat > test.tf << 'EOL'
# This tests read access to various AWS services

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
  # profile = "your-profile-name" # Uncomment if using a specific profile
}

data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {}
data "aws_ec2_instance_type_offerings" "example" {
  location_type = "region"
  filter {
    name   = "instance-type"
    values = ["t3.micro", "t3.small"]
  }
}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
  description = "Current AWS Account ID"
}

output "current_region" {
  value = data.aws_region.current.name
  description = "Current AWS Region"
}

output "available_azs" {
  value = data.aws_availability_zones.available.names
  description = "Available Availability Zones"
}

output "instance_types" {
  value = data.aws_ec2_instance_type_offerings.example.instance_types
  description = "Available EC2 Instance Types"
}
EOL

# Initialize and test
terraform init
terraform plan
terraform apply -auto-approve

# Clean up
terraform destroy -auto-approve
cd ~ && rm -rf ~/aws-terraform-test
```

**Expected Results:**
- `terraform init` downloads AWS provider successfully
- `terraform plan` shows no errors and displays planned data sources
- `terraform apply` executes successfully and shows account info
- `terraform destroy` cleans up without errors

</details>

---

## 🚨 Troubleshooting Common Issues

### AWS CLI Issues

<details>
<summary><b>❌ "aws: command not found"</b></summary>

**Cause**: AWS CLI not in PATH or not installed correctly

**Solutions**:
```bash
# Check if AWS CLI is installed
which aws

# If not found, reinstall (macOS with Homebrew)
brew reinstall awscli

# Or check PATH
echo $PATH

# Add to PATH if needed (add to ~/.zshrc)
export PATH=$PATH:/usr/local/bin
```
</details>

<details>
<summary><b>❌ "Unable to locate credentials"</b></summary>

**Cause**: AWS credentials not configured

**Solutions**:
```bash
# Check credentials file
cat ~/.aws/credentials

# Check config file
cat ~/.aws/config

# Reconfigure credentials
aws configure

# Check environment variables
echo $AWS_ACCESS_KEY_ID
echo $AWS_SECRET_ACCESS_KEY
echo $AWS_PROFILE
```
</details>

<details>
<summary><b>❌ "An error occurred (UnauthorizedOperation)"</b></summary>

**Cause**: Insufficient IAM permissions

**Solutions**:
- Check IAM user permissions in AWS Console
- Verify attached policies
- Test with minimal permissions first
- Use `aws iam simulate-principal-policy` to test permissions
</details>

### Terraform Issues

<details>
<summary><b>❌ "terraform: command not found"</b></summary>

**Cause**: Terraform not in PATH

**Solutions**:
```bash
# Check installation
which terraform

# Reinstall (macOS with Homebrew)
brew reinstall terraform

# Or download manually and add to PATH
```
</details>

<details>
<summary><b>❌ "Error configuring the backend"</b></summary>

**Cause**: AWS credentials or region issues

**Solutions**:
```bash
# Verify AWS credentials work
aws sts get-caller-identity

# Check region configuration
aws configure get region

# Verify S3 access (if using S3 backend)
aws s3 ls
```
</details>

---

## 🔐 Security Best Practices

### Credential Management
- **Never commit AWS credentials** to version control
- **Use IAM roles** for EC2 instances instead of hardcoded credentials
- **Rotate access keys** regularly (every 90 days)
- **Enable MFA** for your AWS account and IAM users
- **Use least privilege principle** when assigning permissions
- **Monitor credential usage** with CloudTrail

### AWS CLI Security
- Use **named profiles** for multiple accounts
- Configure **MFA for sensitive operations**
- Use **temporary credentials** when possible
- **Review and clean up** unused profiles regularly

### Terraform Security
- Use **remote state** with encryption (S3 + DynamoDB)
- **Never store secrets** in Terraform files
- Use **Terraform Cloud** or **AWS Secrets Manager** for sensitive data
- **Enable state locking** to prevent concurrent modifications
- **Review plans** carefully before applying

---

## 📚 Additional Tools and Enhancements

### Shell Completion (Recommended)

<details>
<summary><b>🚀 Enable auto-completion for productivity</b></summary>

**For AWS CLI (zsh):**
```bash
# Add to ~/.zshrc
echo 'complete -C aws_completer aws' >> ~/.zshrc
source ~/.zshrc
```

**For Terraform (zsh):**
```bash
# Add to ~/.zshrc
echo 'autoload -U +X bashcompinit && bashcompinit' >> ~/.zshrc
echo 'complete -o nospace -C terraform terraform' >> ~/.zshrc
source ~/.zshrc
```
</details>

### Useful Aliases

<details>
<summary><b>⚡ Time-saving aliases</b></summary>

Add these to your `~/.zshrc`:
```bash
# AWS aliases
alias awsp='export AWS_PROFILE='
alias awswho='aws sts get-caller-identity'
alias awsregions='aws ec2 describe-regions --query "Regions[].RegionName" --output table'

# Terraform aliases
alias tf='terraform'
alias tfi='terraform init'
alias tfp='terraform plan'
alias tfa='terraform apply'
alias tfd='terraform destroy'
alias tfv='terraform validate'
alias tff='terraform fmt'

# Reload shell
source ~/.zshrc
```
</details>

### Development Environment Variables

<details>
<summary><b>🌍 Useful environment variables</b></summary>

Add to your `~/.zshrc`:
```bash
# AWS Configuration
export AWS_DEFAULT_REGION=us-east-1
export AWS_DEFAULT_OUTPUT=json
export AWS_CLI_AUTO_PROMPT=on-partial  # Enable auto-prompt
export AWS_PAGER=""  # Disable pager for AWS CLI output

# Terraform Configuration
export TF_LOG=INFO  # Enable Terraform logging (use ERROR, WARN, INFO, DEBUG, or TRACE)
export TF_LOG_PATH=./terraform.log  # Log to file

# Reload shell
source ~/.zshrc
```
</details>

output "instance_types" {
  value = slice(data.aws_ec2_instance_type_offerings.example.instance_types, 0, 5)
}
EOL

# Initialize and apply
terraform init
terraform apply -auto-approve

# Clean up
terraform destroy -auto-approve
```

---

## 📝 Summary

You have successfully set up your AWS development environment! Here's what you've accomplished:

### ✅ Environment Setup Complete
- **AWS CLI v2**: Installed and configured with proper authentication
- **Terraform**: Ready for infrastructure as code deployment
- **Security**: Following best practices with IAM users and least privilege
- **Multiple Profiles**: Configured for different environments/accounts
- **Verification**: Tested access to AWS services and Terraform functionality

### 🔍 Quick Status Check
Run these commands to verify everything is working:

```bash
# Check versions
aws --version && terraform -version

# Verify AWS access
aws sts get-caller-identity

# Test Terraform
terraform version
```

---

## 🎯 Best Practices Summary

### 🔐 Security
- **Never commit AWS credentials** to version control
- **Use IAM roles** for EC2 instances instead of hardcoded credentials
- **Rotate access keys** regularly (every 90 days)
- **Enable MFA** for your AWS account and IAM users
- **Follow least privilege** principle for permissions

### ⚡ Productivity
- **Use named profiles** for multiple AWS accounts
- **Enable shell completion** for AWS CLI and Terraform
- **Use aliases** for frequently used commands
- **Set environment variables** for consistent configuration
- **Use `--dry-run`** flag to test commands safely

### 🏗️ Infrastructure Management
- **Use remote state** storage with S3 and DynamoDB for locking
- **Structure code** into reusable modules
- **Use workspaces** for managing multiple environments
- **Always run `terraform plan`** before `terraform apply`
- **Tag resources** appropriately for cost tracking and organization

---

## 🚀 Next Steps

Now that your environment is configured, you're ready to dive into AWS CloudOps topics:

### Immediate Next Steps
1. **Explore AWS CLI**: Practice with basic AWS commands
2. **Set up billing alerts**: Configure cost monitoring for your account
3. **Create a test VPC**: Use Terraform to deploy basic infrastructure

### Course Progression
Continue with the structured learning path:

**🔍 Chapter 1: [Monitoring and Observability](../01-monitoring/README.md)**
- CloudWatch metrics, logs, and alarms
- AWS CloudTrail for audit logging
- AWS X-Ray for application tracing

---

## 📚 Additional Resources

### Documentation
- [AWS CLI User Guide](https://docs.aws.amazon.com/cli/latest/userguide/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)

### Community
- [AWS Community Forums](https://forums.aws.amazon.com/)
- [Terraform Community](https://discuss.hashicorp.com/c/terraform-core/27)
- [r/aws Subreddit](https://www.reddit.com/r/aws/)

---

**🎉 Congratulations!** Your AWS development environment is ready. Time to start building and learning!

**Next**: [Chapter 1 - Monitoring and Observability](../01-monitoring/README.md)
