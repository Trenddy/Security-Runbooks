# SOP: IAM Least Privilege Configuration

**Author:** Troy Johnson  
**Version:** 1.0  
**Last Updated:** May 2026  
**Purpose:** To establish and enforce least privilege access controls within AWS Identity and Access Management (IAM).

---

## Overview
The principle of least privilege ensures that users, roles, and services are granted only the minimum permissions required to perform their job functions. This runbook outlines the standard process for configuring IAM in alignment with least privilege best practices.

---

## Prerequisites
- AWS account with administrative access
- Understanding of AWS services in use within the environment
- CloudTrail enabled for auditing IAM changes (see CloudTrail SOP)

---

## Steps

### 1. Secure the Root Account
- Log into the AWS Console as root
- Navigate to **IAM** → **Security recommendations**
- Enable **MFA on the root account** immediately
- Do not create access keys for the root account
- Store root credentials in a secure password manager
- Use root only for tasks that explicitly require it

### 2. Create Individual IAM Users
- Navigate to **IAM** → **Users** → **Create User**
- Create a unique user for every individual requiring access
- Never share IAM credentials between users
- Assign users to groups rather than attaching policies directly

### 3. Create IAM Groups by Job Function
- Navigate to **IAM** → **User Groups** → **Create Group**
- Create groups based on job roles, for example:
  - `security-analysts`
  - `cloud-engineers`
  - `read-only-auditors`
- Attach permission policies to groups, not individual users

### 4. Apply Least Privilege Policies
- Navigate to **IAM** → **Policies** → **Create Policy**
- Use the **Visual Editor** or JSON to define permissions
- Grant only the specific actions required for the job function
- Example minimal S3 read-only policy:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::example-bucket",
        "arn:aws:s3:::example-bucket/*"
      ]
    }
  ]
}

### 5. Enforce MFA for All Users
- Navigate to **IAM** → **Policies** → **Create Policy**
- Create and attach a policy that denies all actions if MFA is not active
- Require users to set up MFA before accessing any resources

### 6. Configure Password Policy
- Navigate to **IAM** → **Account settings** → **Edit password policy**
- Apply the following minimum settings:
  - Minimum password length: 14 characters
  - Require uppercase and lowercase letters
  - Require numbers and symbols
  - Password expiration: 90 days
  - Prevent password reuse: last 10 passwords

### 7. Use IAM Roles for Services
- Navigate to **IAM** → **Roles** → **Create Role**
- Assign roles to AWS services instead of embedding credentials in code
- Scope role permissions to only what the service requires
- Rotate role trust policies regularly

---

## Verification
- Navigate to **IAM** → **Credential Report** → **Download Report**
- Confirm no users have unused credentials older than 90 days
- Confirm all users have MFA enabled
- Confirm root account has MFA enabled and no active access keys
- Run **IAM Access Analyzer** to identify overly permissive policies

---

## IAM Health Checklist
| Check | Expected Status |
|-------|----------------|
| Root MFA enabled | Yes |
| Root access keys | None |
| All users have MFA | Yes |
| No inline policies on users | Confirmed |
| Access keys rotated within 90 days | Yes |
| IAM Access Analyzer enabled | Yes |

---

## Notes
- Review IAM permissions quarterly using Access Advisor to remove unused permissions
- Never embed IAM credentials in code — use roles and environment variables
- Enable IAM Access Analyzer in every region to catch unintended public access
- Tag all IAM roles and users for easier auditing and cost allocation
