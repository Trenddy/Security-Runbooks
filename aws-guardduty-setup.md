# SOP: AWS GuardDuty Threat Detection Setup

**Author:** Troy Johnson  
**Version:** 1.0  
**Last Updated:** May 2026  
**Purpose:** To enable and configure AWS GuardDuty for continuous threat detection and monitoring.

---

## Overview
AWS GuardDuty is a managed threat detection service that continuously monitors for malicious activity and unauthorized behavior across your AWS accounts, workloads, and data.

---

## Prerequisites
- AWS account with administrative access
- CloudTrail enabled (see CloudTrail SOP)
- IAM permissions to enable GuardDuty

---

## Steps

### 1. Navigate to GuardDuty
- Log into the AWS Management Console
- Search for **GuardDuty** in the services search bar
- Click **Get Started**

### 2. Enable GuardDuty
- Click **Enable GuardDuty**
- GuardDuty will immediately begin analyzing CloudTrail, VPC Flow Logs, and DNS logs
- No additional configuration is required to begin detection

### 3. Configure Findings
- Navigate to **Findings** in the left menu
- Set findings export frequency (recommended: every 15 minutes)
- Create an **S3 export** to archive findings for long term storage
- Enable **CloudWatch Events** integration to trigger automated alerts

### 4. Set Up Notifications
- Navigate to **SNS** in the AWS Console
- Create a new topic (e.g. `guardduty-alerts`)
- Subscribe your email or ticketing system to the topic
- Create a **CloudWatch Events rule** to forward GuardDuty findings to SNS

### 5. Configure Trusted IP Lists (Optional)
- Navigate to **Lists** in GuardDuty
- Upload a trusted IP list to reduce false positives from known safe sources

---

## Verification
- Navigate to **Findings** dashboard
- Use the **Generate Sample Findings** option to confirm alerts are working
- Verify SNS notifications are received for sample findings
- Confirm findings are being exported to S3

---

## Severity Levels
| Level | Score | Response |
|-------|-------|----------|
| High | 7.0 - 8.9 | Immediate investigation required |
| Medium | 4.0 - 6.9 | Investigate within 24 hours |
| Low | 1.0 - 3.9 | Review during regular security audits |

---

## Notes
- Enable GuardDuty across all AWS regions for full coverage
- Review findings weekly at minimum
- Suppress known false positives using suppression rules to reduce alert fatigue
- Consider enabling GuardDuty at the AWS Organizations level for multi-account environments
