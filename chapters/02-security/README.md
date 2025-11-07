# Chapter 2: Security & Identity

> **Domain Coverage**: Security and Compliance (25% of exam)

Learn AWS security services to protect your infrastructure and implement least-privilege access patterns through hands-on practice.

## Learning Path

### 📖 1. Study the Theory
Start with detailed notes covering all exam topics and real-world security patterns:

| Service | What You'll Master | Exam Weight |
|---------|-------------------|-------------|
| **[IAM Fundamentals](notes/01-iam-fundamentals.md)** | Users, roles, policies, permission boundaries, policy evaluation | High |
| **[KMS & Data Protection](notes/02-kms-data-protection.md)** | Encryption at rest, key management, Secrets Manager | High |
| **[GuardDuty & Threat Detection](notes/03-guardduty-threat-detection.md)** | Threat detection, automated incident response | Medium |
| **[Secure Access Patterns](notes/04-secure-access-patterns.md)** | Session Manager, VPC endpoints, federated access | Medium |

### 🛠️ 2. Build with Hands-On Labs

Complete these labs to implement production-ready security controls:

| Lab | Security Domain | What You'll Build | Time | Cost |
|-----|-----------------|------------------|------|------|
| **[01-iam-permission-boundary-ec2-role](labs/basic/01-iam-permission-boundary-ec2-role.md)** | IAM Access Control | Least-privilege EC2 role with permission boundary | 20-30min | Free |
| **[01-s3-kms-encryption-policy](labs/intermediate/01-s3-kms-encryption-policy.md)** | Data Protection | S3 encryption with KMS, deny unencrypted uploads | 25-35min | ~$1-2 |
| **[02-guardduty-eventbridge-sns](labs/intermediate/02-guardduty-eventbridge-sns.md)** | Threat Detection | GuardDuty with automated alert routing | 30-40min | ~$2-5 |

### 📝 3. Test Your Knowledge
Practice exam-style questions: **[Practice Questions](exam_questions/)**

### 📋 4. Quick Review
Use the **[Security Cheatsheet](cheatsheets/security_cheatsheet.md)** for rapid review of all services, key concepts, and exam tips. Perfect for final review before the exam.

## 💰 Cost Information

- **Free Labs**: 1 out of 3 labs completely free
- **Paid Labs**: ~$3-7 total for all intermediate labs
- **Cost Updates**: AWS pricing changes frequently - check each lab for current estimates

## Getting Started

**Recommended Path:**
1. **Start with the notes** - [IAM Fundamentals](notes/01-iam-fundamentals.md) provides the foundation
2. **Follow the lab sequence** - Begin with [Basic IAM Lab](labs/basic/01-iam-permission-boundary-ec2-role.md)
3. **Test yourself** - Use practice questions after completing labs
4. **Final review** - Use the cheatsheet before taking the exam