# Security Runbooks

A collection of security runbooks, automation guides, and standard operating procedures (SOPs) developed from hands-on experience with AWS cloud infrastructure, security operations, and DevSecOps tooling.

## About

I'm Troy Johnson, a Computer Science graduate (Cloud Systems concentration) with a background as a U.S. Air Force Aircraft Fuels Operator. This repository documents real-world security processes, configurations, automation workflows, and best practices across AWS security and cloud operations.

## Certifications

- CompTIA Security+
- AWS Solutions Architect
- AWS AI Practitioner
- Currently studying: CompTIA CySA+

---

## Runbooks

### AWS Security
| Runbook | Description |
|---------|-------------|
| [CloudTrail Setup & Configuration](./aws-cloudtrail-setup.md) | Enable and configure CloudTrail for audit logging across AWS accounts |
| [GuardDuty Threat Detection](./aws-guardduty-setup.md) | Set up GuardDuty for continuous threat monitoring |
| [CloudWatch Security Alerting](./aws-cloudwatch-security-alerting.md) | Configure security-focused alarms and alerting pipelines |
| [IAM Least Privilege](./aws-iam-least-privilege.md) | Apply least privilege principles to IAM roles and policies |

### Automation & DevSecOps
| Runbook | Description |
|---------|-------------|
| [Ansible — EC2 Security Hardening](./ansible/ansible-ec2-hardening-runbook.md) | Automate CIS-aligned security hardening across EC2 instances using Ansible playbooks |
| [Jenkins — CI/CD Pipeline](./jenkins/jenkins-cicd-pipeline-runbook.md) | Build and deploy applications automatically with Jenkins, including AWS integration and troubleshooting |

---

## How the Automation Runbooks Connect

Jenkins handles *what code goes out and when*. Ansible handles *what state the servers are in when that code arrives*. Together they form a complete DevSecOps pipeline:

```
Code pushed to GitHub
        │
        ▼
    Jenkins         ← Detects change, runs tests, builds app
        │
        ▼
    Ansible         ← Hardens and configures target EC2 instances
        │
        ▼
   AWS Deploy       ← Jenkins deploys the application
        │
        ▼
  Notification      ← Slack alert on success or failure
```

---

## Related Projects

- [CloudTrail Log Analyzer](https://github.com/Trenddy/Cloudtrail-Log-Analyzer) — Python-based CloudTrail anomaly detection and spike identification
- [CloudTrail MCP Server](https://github.com/Trenddy/CloudTrail-MCP-Server) — MCP server exposing CloudTrail analysis as a tool for AI assistants
- [AWS Security Posture Dashboard](https://github.com/Trenddy/aws-security-posture-dashboard) — Live security posture reporting across GuardDuty, IAM, S3, and CloudTrail

---

## Purpose

These runbooks serve as living documents — practical references for securing and automating cloud environments, and a reflection of my ongoing growth in cloud security and DevSecOps.
