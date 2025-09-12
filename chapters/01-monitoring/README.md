# Chapter 1: Monitoring & Observability

> **Domain Coverage**: Monitoring, Logging, and Remediation (22% of exam)

Learn AWS monitoring services to maintain visibility into your infrastructure and applications through hands-on practice with the CloudMart e-commerce platform.

## Learning Path

### 📖 1. Study the Theory
Start with detailed notes covering all exam topics and real-world scenarios:

| Service | What You'll Master | Exam Weight |
|---------|-------------------|-------------|
| **[CloudWatch Metrics](notes/01-cloudwatch-metrics.md)** | Monitor infrastructure performance with time-series data | High |
| **[CloudWatch Logs](notes/02-cloudwatch-logs.md)** | Centralize and analyze application and system logs | High |
| **[CloudWatch Alarms](notes/03-cloudwatch-alarms.md)** | Set up automated alerts and responses to threshold breaches | High |
| **[CloudTrail](notes/04-cloudtrail.md)** | Audit API calls and track user activities across AWS | Medium |
| **[AWS Config](notes/05-aws-config.md)** | Monitor configuration compliance and detect drift | Medium |
| **[EventBridge](notes/06-eventbridge.md)** | Create event-driven automation and integrations | Medium |
| **[Systems Manager](notes/07-systems-manager.md)** | Manage and automate operational tasks at scale | Medium |
| **[X-Ray](notes/08-x-ray.md)** | Trace application performance and debug issues | Low |

### 🛠️ 2. Build with Hands-On Labs

> **💡 The CloudMart Story:** Follow CloudMart's journey from startup through enterprise. Each lab builds on the previous ones - **don't skip any labs** as they're connected by a continuous story.

Complete these labs in order to build real AWS monitoring infrastructure:

| Lab | CloudMart Milestone | What You'll Build | Time | Cost |
|-----|-------------------|------------------|------|------|
| **[01-cost-and-billing-monitoring](labs/01-cost-and-billing-monitoring.md)** | Account Setup | Cost monitoring & budget alerts | 20-30min | Free |
| **[02-cloudwatch-metrics-dashboard](labs/02-cloudwatch-metrics-dashboard.md)** | First Server | Infrastructure monitoring & custom metrics | 30-40min | Free |
| **[03-cloudwatch-logs-centralization](labs/03-cloudwatch-logs-centralization.md)** | Log Analytics | Centralized logging & log-based metrics | 25-35min | Free |
| **[04-cloudwatch-alarms-automation](labs/04-cloudwatch-alarms-automation.md)** | Proactive Monitoring | Intelligent alerts & auto-recovery | 35-45min | Free |
| **[05-cloudtrail-audit-logging](labs/05-cloudtrail-audit-logging.md)** | Security & Compliance | API audit logging & security monitoring | 30-40min | Free |
| **[06-config-compliance-monitoring](labs/06-config-compliance-monitoring.md)** | Compliance Phase | Resource compliance & drift detection | 40-50min | ~$2-5 |
| **[07-eventbridge-event-automation](labs/07-eventbridge-event-automation.md)** | Event Automation | Automated responses to AWS events | 45-60min | ~$1-3 |
| **[08-systems-manager-basic](labs/08-systems-manager-basic.md)** | Remote Access | Session Manager & Run Command | 30-40min | ~$1-2 |
| **[09-systems-manager-advanced](labs/09-systems-manager-advanced.md)** | Operations Management | Patch management & automation | 40-50min | ~$2-3 |
| **[10-xray-basic-tracing](labs/10-xray-basic-tracing.md)** | Performance Foundation | Basic application tracing & service maps | 30-45min | Free |
| **[11-xray-advanced-performance](labs/11-xray-advanced-performance.md)** | Performance Analytics | Advanced tracing & optimization | 45-60min | Free |

### 📝 3. Test Your Knowledge
Practice exam-style questions: **[Practice Questions](exam_questions/)**

### 📋 4. Quick Review
Use the **[Monitoring Cheatsheet](cheatsheets/monitoring_cheatsheet.md)** for rapid review of all services, key concepts, and exam tips. Perfect for final review before the exam.

## 💰 Cost Information

- **Free Labs**: 1-5 + 10-11 (7 out of 11 labs completely free)
- **Paid Labs**: 6-9 use minimal resources (~$8-18 total for entire chapter)
- **Cost Updates**: AWS pricing changes frequently - check each lab for current estimates

## Getting Started

**Recommended Path:**
1. **Start with the notes** - [CloudWatch Metrics](notes/01-cloudwatch-metrics.md) provides the foundation
2. **Follow the lab sequence** - Begin with [Lab 1](labs/01-cost-and-billing-monitoring.md) and complete in order
3. **Test yourself** - Use practice questions after completing labs
4. **Final review** - Use the cheatsheet before taking the exam