# SOP: CloudWatch Security Alerting Setup

**Author:** Troy Johnson  
**Version:** 1.0  
**Last Updated:** May 2026  
**Purpose:** To configure AWS CloudWatch metric filters and alarms for security-relevant events.

---

## Overview
AWS CloudWatch enables real-time monitoring of AWS resources and applications. This runbook covers setting up security-focused metric filters and alarms based on CloudTrail log data to detect and alert on suspicious activity.

---

## Prerequisites
- AWS account with administrative access
- CloudTrail enabled and logging to S3 (see CloudTrail SOP)
- CloudWatch Logs group configured to receive CloudTrail logs
- SNS topic created for alert notifications

---

## Steps

### 1. Send CloudTrail Logs to CloudWatch
- Navigate to **CloudTrail** in the AWS Console
- Select your trail and click **Edit**
- Under **CloudWatch Logs**, click **Enabled**
- Create or select an existing Log Group (e.g. `cloudtrail-security-logs`)
- Assign an IAM role with CloudWatch Logs write permissions
- Save changes

### 2. Create Metric Filters
Navigate to **CloudWatch** → **Log Groups** → select your CloudTrail log group → **Metric Filters** → **Create Metric Filter**

Create the following filters:

#### Unauthorized API Calls
Metric name: `UnauthorizedAPICalls`

Filter pattern:
{ ($.errorCode = "AccessDenied") || ($.errorCode = "UnauthorizedOperation") }

#### Root Account Usage
Metric name: `RootAccountUsage`

Filter pattern:
{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }

#### IAM Policy Changes
Metric name: `IAMPolicyChanges`

Filter pattern:
{ ($.eventName = DeleteGroupPolicy) || ($.eventName = DeleteRolePolicy) || ($.eventName = PutGroupPolicy) || ($.eventName = PutRolePolicy) || ($.eventName = PutUserPolicy) }

#### Console Login Without MFA
Metric name: `ConsoleLoginWithoutMFA`

Filter pattern:
{ ($.eventName = "ConsoleLogin") && ($.additionalEventData.MFAUsed != "Yes") }

---

### 3. Create CloudWatch Alarms
For each metric filter created above:
- Navigate to **CloudWatch** → **Alarms** → **Create Alarm**
- Select the metric you created
- Set threshold to **greater than or equal to 1** occurrence
- Set evaluation period to **1 minute**
- Under **Notification**, select your SNS topic
- Name the alarm descriptively (e.g. `Alert-RootAccountUsage`)
- Click **Create Alarm**

---

## Verification
- Trigger a test event for each filter (e.g. attempt an unauthorized API call)
- Confirm the metric count increases in CloudWatch
- Confirm SNS notification is received within 5 minutes
- Review alarm state changes in the CloudWatch Alarms dashboard

---

## Key Alarms Reference
| Alarm | Trigger | Priority |
|-------|---------|----------|
| Unauthorized API Calls | Any AccessDenied error | High |
| Root Account Usage | Any root login or action | Critical |
| IAM Policy Changes | Any policy modification | High |
| Console Login Without MFA | MFA not used on login | High |

---

## Notes
- Always alert on root account usage — root should never be used for day to day tasks
- Review and tune alarm thresholds regularly to reduce false positives
- Consider routing Critical alarms to PagerDuty or a ticketing system for faster response
- Store metric filter configurations in version control for auditability
