# SOP: AWS CloudTrail Setup & Configuration

**Author:** Troy Johnson  
**Version:** 1.0  
**Last Updated:** May 2026  
**Purpose:** To enable and configure AWS CloudTrail for audit logging and account activity monitoring.

---

## Overview
AWS CloudTrail records API calls and account activity across your AWS infrastructure. This runbook outlines the standard process for enabling CloudTrail in a new or existing AWS environment.

---

## Prerequisites
- AWS account with administrative access
- S3 bucket for log storage
- Basic understanding of IAM permissions

---

## Steps

### 1. Navigate to CloudTrail
- Log into the AWS Management Console
- Search for **CloudTrail** in the services search bar
- Click **Create Trail**

### 2. Configure the Trail
- **Trail name:** Enter a descriptive name (e.g. `security-audit-trail`)
- **Storage location:** Create a new S3 bucket or select an existing one
- Enable **Log file validation** to ensure log integrity
- Enable **SSE-KMS encryption** for log security

### 3. Configure Event Types
- Enable **Management events** (read and write)
- Enable **Data events** if monitoring S3 or Lambda activity
- Enable **Insights events** to detect unusual API activity

### 4. Review and Create
- Review all settings
- Click **Create trail**
- Confirm the trail appears as **Enabled** in the dashboard

---

## Verification
- Navigate to the S3 bucket selected during setup
- Confirm log files are being delivered within 15 minutes
- Test by performing an API action and confirming it appears in the logs

---

## Notes
- Apply CloudTrail to all regions to ensure full coverage
- Set S3 bucket lifecycle policies to manage log retention costs
- Restrict S3 bucket access using bucket policies and IAM roles
