# 🔐 AWS IAM Hands-On Lab Guide
### Production-Grade Identity & Access Management

---

> **Author:** Sajal Jana  
> **Level:** Beginner to Intermediate  
> **AWS Account:** Free Tier Compatible  
> **Last Updated:** March 2026

---

## 📋 Table of Contents

1. [Introduction](#introduction)
2. [What You Will Learn](#what-you-will-learn)
3. [Prerequisites](#prerequisites)
4. [Lab Architecture Overview](#lab-architecture-overview)
5. [Phase 1 — Users & Groups](#phase-1--users--groups)
   - [Step 1: Access the IAM Console](#step-1-access-the-iam-console)
   - [Step 2: Create IAM User Groups](#step-2-create-iam-user-groups)
   - [Step 3: Create Developers & Auditors Groups](#step-3-create-developers--auditors-groups)
   - [Step 4: Create IAM Users](#step-4-create-iam-users)
   - [Step 5: Create Bob and Carol](#step-5-create-bob-and-carol)
6. [Phase 2 — Testing Permissions](#phase-2--testing-permissions)
   - [Step 6: Test Bob's Developer Access](#step-6-test-bobs-developer-access)
   - [Step 7: Test Carol's Auditor Access](#step-7-test-carols-auditor-access)
7. [Phase 3 — IAM Roles](#phase-3--iam-roles)
   - [Step 8: Create an IAM Role for EC2](#step-8-create-an-iam-role-for-ec2)
8. [Phase 4 — Custom IAM Policies](#phase-4--custom-iam-policies)
   - [Step 9: Write a Custom IAM Policy](#step-9-write-a-custom-iam-policy)
   - [Step 10: Attach Custom Policy & Test Deny](#step-10-attach-custom-policy--test-deny)
9. [Phase 5 — Security Best Practices](#phase-5--security-best-practices)
   - [Step 11: Set Account Password Policy](#step-11-set-account-password-policy)
   - [Step 12: Generate IAM Credentials Report](#step-12-generate-iam-credentials-report)
   - [Step 13: Enable MFA on Root Account](#step-13-enable-mfa-on-root-account)
10. [Lab Cleanup](#lab-cleanup)
11. [Best Practices Checklist](#best-practices-checklist)
12. [What to Learn Next](#what-to-learn-next)

---

## Introduction

**AWS IAM (Identity and Access Management)** is the backbone of AWS security. It controls **who can do what** in your AWS account. Every serious production environment is built on a strong IAM foundation.

> 🔑 **Core Principle:** Never use your root account for daily work. Use IAM users, groups, and roles with the least privilege required.

This lab simulates a **real-world company scenario** where you are a Cloud Administrator setting up access control for a startup team.

---

## What You Will Learn

| Topic | Description |
|---|---|
| IAM Users & Groups | Structure team access like a real company |
| IAM Policies | Managed and custom least-privilege policies |
| IAM Roles | Secure service-to-service access (EC2 → S3) |
| Permission Testing | Verify access controls work correctly |
| Password Policy | Enforce enterprise-grade password rules |
| MFA | Protect the root account with Multi-Factor Auth |
| Credentials Report | Audit all IAM users for compliance |

---

## Prerequisites

Before starting this lab, ensure you have:

- [ ] An **AWS Free Tier** account ([Sign up here](https://aws.amazon.com/free/))
- [ ] Access to the **AWS Management Console**
- [ ] A smartphone with an **Authenticator App** installed:
  - [Google Authenticator](https://support.google.com/accounts/answer/1066447)
  - [Authy](https://authy.com/) *(recommended — supports backup)*
  - [Microsoft Authenticator](https://www.microsoft.com/en-us/security/mobile-authenticator-app)
- [ ] A web browser with **incognito/private mode** support

> ⚠️ **Important:** Log in as the **root user** for the initial setup steps only.

---

## Lab Architecture Overview

Here is what you will build by the end of this lab:

```
AWS Account
│
├── 🔒 Password Policy        (12 chars, 90-day expiry, complexity rules)
├── 🛡️  Root MFA               (Multi-Factor Authentication enabled)
│
├── 👥 IAM Groups
│    ├── prod-admins           → AdministratorAccess
│    ├── prod-developers       → EC2 + S3 + Lambda Full Access
│    └── prod-auditors         → ReadOnlyAccess
│
├── 👤 IAM Users
│    ├── alice                 → prod-admins group
│    ├── bob                   → prod-developers group + custom policies
│    └── carol                 → prod-auditors group
│
├── 📝 Custom IAM Policies
│    ├── prod-s3-readonly-specific-bucket
│    └── prod-deny-destructive-ec2-actions
│
└── 🎭 IAM Role
     └── prod-ec2-s3-read-role  (Allows EC2 to read from S3)
```

---

## Phase 1 — Users & Groups

### Step 1: Access the IAM Console

The very first real-world best practice is to **never use your root account for daily operations**. Root has unlimited power with no restrictions. In this step, we access IAM to start building proper access controls.

**Instructions:**

1. Log in to [AWS Management Console](https://console.aws.amazon.com) as **root**
2. In the top search bar, type `IAM`
3. Click on **IAM** under Services
4. You will land on the **IAM Dashboard**

> 📸 **Screenshot Guide:**
> ```
> AWS Console → Search Bar → Type "IAM" → Click IAM Service
> ```

**What to observe on the dashboard:**

Look at the **"Security recommendations"** section. AWS will warn you about:
- ✅ Root account has no MFA → *We will fix this in Phase 5*
- ✅ No IAM users created yet → *We will fix this now*

---

### Step 2: Create IAM User Groups

> 💡 **Why Groups first, not Users?**  
> In production, you **never assign permissions directly to users**.  
> - Create **Groups** (e.g., Admins, Developers, Auditors)  
> - Assign **permissions to the Group**  
> - Add **Users into Groups**  
>
> When a developer joins → add to group. When they leave → remove from group. ✅

**Real-World Scenario:**
> You are a Cloud Admin at a startup with 3 types of staff:
> - **Admins** → Full AWS access
> - **Developers** → EC2, S3, Lambda access only
> - **Auditors** → Read-only access (see everything, change nothing)

**Create the "prod-admins" Group:**

1. On the left sidebar, click **"User groups"**
2. Click the orange **"Create group"** button
3. Enter **Group name:**
   ```
   prod-admins
   ```
4. Scroll to **"Attach permissions policies"**
5. Search for and check ✅:
   ```
   AdministratorAccess
   ```
6. Click **"Create user group"**

> 📸 **Screenshot Guide:**
> ```
> IAM → User Groups → Create Group → Name: prod-admins → Attach: AdministratorAccess
> ```

---

### Step 3: Create Developers & Auditors Groups

**Create the "prod-developers" Group:**

1. Click **"Create group"**
2. Enter **Group name:**
   ```
   prod-developers
   ```
3. Search and check ✅ each of these 3 policies:
   ```
   AmazonEC2FullAccess
   AmazonS3FullAccess
   AWSLambda_FullAccess
   ```
4. Click **"Create user group"**

**Create the "prod-auditors" Group:**

1. Click **"Create group"**
2. Enter **Group name:**
   ```
   prod-auditors
   ```
3. Search and check ✅:
   ```
   ReadOnlyAccess
   ```
4. Click **"Create user group"**

**Expected Result — 3 Groups Created:**

| Group | Policy | Purpose |
|---|---|---|
| `prod-admins` | AdministratorAccess | Full AWS control |
| `prod-developers` | EC2 + S3 + Lambda Full Access | Dev work only |
| `prod-auditors` | ReadOnlyAccess | Compliance & review |

---

### Step 4: Create IAM Users

> 💡 **Production Rule:** Every person gets their **own individual user account**. Never share credentials. This ensures accountability and traceability in audit logs.

**Users we will create:**

| User | Represents | Group |
|---|---|---|
| `alice` | Cloud Administrator | prod-admins |
| `bob` | Developer | prod-developers |
| `carol` | Auditor | prod-auditors |

**Create User "alice":**

1. On the left sidebar, click **"Users"**
2. Click **"Create user"**
3. Enter **User name:**
   ```
   alice
   ```
4. ✅ Check **"Provide user access to the AWS Management Console"**
5. Select **"I want to create an IAM user"**
6. Under **Console password**, select **"Custom password"** and enter:
   ```
   Admin@123456
   ```
7. **Uncheck** "Users must create a new password at next sign-in" *(for lab simplicity)*
8. Click **"Next"**
9. Select **"Add user to group"**
10. ✅ Check **`prod-admins`**
11. Click **"Next"** → **"Create user"**

> 📌 **Important:** After creation, AWS shows the **console sign-in URL**:
> ```
> https://123456789012.signin.aws.amazon.com/console
> ```
> **Save this URL** — you will need it to log in as IAM users later!

---

### Step 5: Create Bob and Carol

**Create User "bob":**

1. Click **"Create user"**
2. **User name:** `bob`
3. ✅ Check console access → Custom password: `Dev@123456`
4. Uncheck password reset requirement
5. Click **"Next"** → ✅ Select **`prod-developers`** → **"Create user"**

**Create User "carol":**

1. Click **"Create user"**
2. **User name:** `carol`
3. ✅ Check console access → Custom password: `Audit@123456`
4. Uncheck password reset requirement
5. Click **"Next"** → ✅ Select **`prod-auditors`** → **"Create user"**

**Verify all 3 users exist on the Users page:**

| User | Group | Password |
|---|---|---|
| `alice` | prod-admins | Admin@123456 |
| `bob` | prod-developers | Dev@123456 |
| `carol` | prod-auditors | Audit@123456 |

> 💡 **Production Insight:** In real companies, user creation is often automated via:
> - **AWS IAM Identity Center (SSO)** for large teams
> - **Terraform or CloudFormation** (Infrastructure as Code)
> - **SCIM provisioning** from HR systems like Okta or Azure AD

---

## Phase 2 — Testing Permissions

### Step 6: Test Bob's Developer Access

The best way to **verify IAM permissions** is to actually log in as each user and test what they can and cannot do.

**Open an Incognito/Private Browser Window:**

| Browser | Shortcut |
|---|---|
| Chrome | `Ctrl + Shift + N` |
| Firefox | `Ctrl + Shift + P` |
| Edge | `Ctrl + Shift + N` |

**Login as "bob":**

1. Paste your IAM sign-in URL in the incognito window
2. Enter:
   - **Username:** `bob`
   - **Password:** `Dev@123456`

**Run these permission tests:**

| Test | Service | Expected Result | Why |
|---|---|---|---|
| ✅ Test 1 | Open EC2 Dashboard | **Access Granted** | AmazonEC2FullAccess policy |
| ✅ Test 2 | Open S3 Console | **Access Granted** | AmazonS3FullAccess policy |
| ❌ Test 3 | Open IAM → Users | **Access Denied** | No IAM policy attached |
| ❌ Test 4 | Open RDS Console | **Access Denied** | No RDS policy attached |

> 🎯 **Least-privilege is working!** Bob can only access what his developer role needs.

---

### Step 7: Test Carol's Auditor Access

**Open a second incognito window** (keep bob's window open).

**Login as "carol":**
- **Username:** `carol`
- **Password:** `Audit@123456`

**Run these permission tests:**

| Test | Action | Expected Result |
|---|---|---|
| ✅ Test 1 | View EC2 Instances list | **Allowed** — ReadOnly policy |
| ❌ Test 2 | Click "Launch instance" → Submit | **Access Denied** — no write actions |
| ✅ Test 3 | Open IAM → View Users list | **Allowed** — ReadOnly policy |
| ❌ Test 4 | Click "Create user" | **Access Denied** — no write actions |
| ✅ Test 5 | Open S3 → View buckets | **Allowed** — ReadOnly policy |
| ❌ Test 6 | Try to delete an S3 bucket | **Access Denied** — no write actions |

> 💡 **Production Insight:** In real companies, auditors are given `ReadOnlyAccess` so they can review infrastructure for compliance checks (SOC2, ISO 27001) without any risk of accidentally modifying production resources. This is a **mandatory security control** in most compliance frameworks.

---

## Phase 3 — IAM Roles

### Step 8: Create an IAM Role for EC2

> 💡 **Roles vs Users:**
> - **Users** → For humans (with username & password)
> - **Roles** → For AWS services (EC2, Lambda, ECS, etc.)
>
> ❌ **Wrong way:** Store AWS access keys inside the EC2 server *(security risk!)*  
> ✅ **Right way:** Attach an **IAM Role** to EC2 so it can access S3 automatically and securely

This is called an **"Instance Profile"** — used in every serious production environment.

**Create an IAM Role for EC2 → S3 access:**

*(Go back to your root/admin browser window first)*

1. In IAM Console, click **"Roles"** → **"Create role"**
2. **Trusted entity type:** `AWS service`
3. **Use case:** `EC2`
4. Click **"Next"**
5. Search and check ✅: `AmazonS3ReadOnlyAccess`
6. Click **"Next"**
7. **Role name:**
   ```
   prod-ec2-s3-read-role
   ```
8. **Description:**
   ```
   Allows EC2 instances to read from S3 buckets
   ```
9. Click **"Create role"**

**Architecture of what you just built:**

```
EC2 Instance
     │
     │ assumes role
     ▼
prod-ec2-s3-read-role
     │
     │ has permission
     ▼
AmazonS3ReadOnlyAccess
     │
     │ can read
     ▼
All S3 Buckets
```

**Common IAM Roles in Production:**

| Scenario | Role Policy Used |
|---|---|
| EC2 app reads from S3 | `AmazonS3ReadOnlyAccess` |
| Lambda writes to DynamoDB | `AmazonDynamoDBFullAccess` |
| CodeDeploy deploys to EC2 | `AWSCodeDeployRole` |
| ECS runs containers | `AmazonECSTaskExecutionRole` |
| CloudWatch monitoring | `CloudWatchFullAccess` |

---

## Phase 4 — Custom IAM Policies

### Step 9: Write a Custom IAM Policy

> 💡 **Why Custom Policies?**  
> AWS Managed Policies are convenient but often **too broad** for production. Real cloud engineers write custom policies that grant access to **specific actions** on **specific resources**.
>
> ❌ **Too broad:** `AmazonS3FullAccess` — read/write/delete ALL buckets  
> ✅ **Just right:** Custom policy — only `GetObject` on only `prod-app-data` bucket

**Create the Custom Policy:**

1. IAM Console → **"Policies"** → **"Create policy"**
2. Click the **"JSON"** tab
3. Paste this policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificS3BucketReadOnly",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::prod-app-data",
        "arn:aws:s3:::prod-app-data/*"
      ]
    }
  ]
}
```

4. Click **"Next"**
5. **Policy name:**
   ```
   prod-s3-readonly-specific-bucket
   ```
6. **Description:**
   ```
   Allows read-only access to prod-app-data S3 bucket only
   ```
7. Click **"Create policy"**

**Policy Field Breakdown:**

| Field | Value | Meaning |
|---|---|---|
| `Version` | `2012-10-17` | Standard policy language version |
| `Sid` | `AllowSpecificS3BucketReadOnly` | Human-readable statement ID |
| `Effect` | `Allow` | Permit this action |
| `Action: s3:GetObject` | Download/read a file | |
| `Action: s3:ListBucket` | See files inside bucket | |
| `Resource` (1) | `arn:aws:s3:::prod-app-data` | The bucket itself |
| `Resource` (2) | `arn:aws:s3:::prod-app-data/*` | All objects inside bucket |

**Understanding the ARN Structure:**

```
arn:aws:s3:::prod-app-data
 │    │   │       │
 │    │   │       └── Resource name (bucket name)
 │    │   └────────── Service (s3)
 │    └────────────── AWS provider
 └─────────────────── Amazon Resource Name prefix
```

> ⚠️ **Common Production Mistake:**
> ```json
> ❌ "Resource": "*"                          ← Applies to ALL S3 buckets
> ✅ "Resource": "arn:aws:s3:::prod-app-data/*"  ← Specific bucket only
> ```

---

### Step 10: Attach Custom Policy & Test Deny

**Attach the custom policy to Bob:**

1. IAM Console → **"Users"** → click **`bob`**
2. **"Add permissions"** → **"Attach policies directly"**
3. Search and check ✅: `prod-s3-readonly-specific-bucket`
4. Click **"Next"** → **"Add permissions"**

**Create an Explicit Deny Policy (Production Guardrail):**

1. IAM Console → **"Policies"** → **"Create policy"** → **"JSON"** tab
2. Paste:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyEC2Termination",
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "ec2:DeleteSecurityGroup",
        "ec2:DeleteVpc"
      ],
      "Resource": "*"
    }
  ]
}
```

3. **Policy name:**
   ```
   prod-deny-destructive-ec2-actions
   ```
4. **Description:**
   ```
   Prevents developers from terminating instances or deleting critical network resources
   ```
5. Create and attach this policy to **`bob`**

**AWS Policy Evaluation Logic:**

```
Step 1 → Is there an explicit DENY?  → Yes ──→ ❌ DENIED (final)
         │
         No
         │
Step 2 → Is there an explicit ALLOW? → Yes ──→ ✅ ALLOWED (final)
         │
         No
         │
Step 3 → Default ──────────────────────────→ ❌ DENIED
```

> 🔑 **Key Rule:** An explicit **Deny ALWAYS wins** over any Allow — no exceptions!

**Bob's Final Permission Summary:**

| Action | Allowed? | Reason |
|---|---|---|
| Launch EC2 instance | ✅ Yes | EC2FullAccess from group |
| Terminate EC2 instance | ❌ No | Explicit Deny overrides Allow |
| Delete VPC | ❌ No | Explicit Deny overrides Allow |
| Read S3 bucket | ✅ Yes | Group + custom policy |
| Access IAM | ❌ No | No policy allows it |

---

## Phase 5 — Security Best Practices

### Step 11: Set Account Password Policy

> 💡 **Why Password Policy?**  
> In production, a Password Policy is a **compliance requirement** for SOC2, ISO 27001, PCI-DSS, and HIPAA. AWS enforces it at the account level so every IAM user must comply.

**Configure the Password Policy:**

1. IAM Console → left sidebar → **"Account settings"**
2. Under **"Password policy"** → click **"Edit"**
3. Configure these settings:

| Setting | Value | Reason |
|---|---|---|
| Minimum password length | `12` | Industry standard minimum |
| Require uppercase letters | ✅ | Complexity requirement |
| Require lowercase letters | ✅ | Complexity requirement |
| Require numbers | ✅ | Complexity requirement |
| Require special characters | ✅ | Characters like `!@#$` |
| Password expiration | ✅ → `90 days` | Force regular rotation |
| Prevent password reuse | ✅ → `5 passwords` | Can't reuse last 5 |

4. Click **"Save changes"**

**Valid vs Invalid Passwords After This Policy:**

```
✅ MyP@ssw0rd#2024    → Valid (12+ chars, upper, lower, number, special)
❌ password123        → Invalid (no uppercase, no special character)
❌ Admin@123          → Invalid (less than 12 characters)
❌ MyP@ssw0rd#2023    → Invalid (if used in last 5 passwords)
```

**Compliance Mapping:**

| Standard | Requirement Met |
|---|---|
| PCI-DSS | ✅ Min 7 chars, 90-day expiry |
| HIPAA | ✅ Complex passwords required |
| SOC2 | ✅ Strong passwords + MFA |
| ISO 27001 | ✅ Regular rotation required |

---

### Step 12: Generate IAM Credentials Report

The **IAM Credentials Report** is a CSV file showing the security status of every IAM user. Security teams generate this **weekly or monthly** for compliance audits.

**Generate the Report:**

1. IAM Console → left sidebar → **"Credential report"**
2. Click **"Download credential report"**
3. Open the downloaded CSV in **Excel or Google Sheets**

**Key Report Columns:**

| Column | Meaning | 🚩 Red Flag If... |
|---|---|---|
| `user` | IAM username | — |
| `password_last_used` | Last console login | Never logged in |
| `password_last_changed` | Last password rotation | Older than 90 days |
| `mfa_active` | Is MFA enabled? | `false` for any user |
| `access_key_1_active` | Has active access key? | `true` if unused |
| `access_key_1_last_used_date` | When key last used | Older than 90 days |

**What you will likely see in your report:**

| User | mfa_active | Red Flag? |
|---|---|---|
| `root` | false | ⚠️ Fix immediately! |
| `alice` | false | ⚠️ Should enable MFA |
| `bob` | false | ⚠️ Should enable MFA |
| `carol` | false | ⚠️ Should enable MFA |

**Automate in Production (AWS CLI):**

```bash
# Generate credentials report
aws iam generate-credential-report

# Download and decode the report
aws iam get-credential-report \
  --query 'Content' \
  --output text | base64 -d > iam-report.csv
```

> 💡 In real companies, this report is fed into **AWS Security Hub** or **SIEM tools** like Splunk to automatically trigger alerts when any user has `mfa_active = false`.

---

### Step 13: Enable MFA on Root Account

> 🚨 **This is the #1 security priority in AWS.**  
> If someone steals your root password **without MFA**, they own your entire AWS account. With MFA, they also need your physical device.

**Supported Authenticator Apps:**

| App | Platform | Backup Support |
|---|---|---|
| **Authy** | iOS & Android | ✅ Cloud backup |
| **Google Authenticator** | iOS & Android | ❌ No backup |
| **Microsoft Authenticator** | iOS & Android | ✅ Cloud backup |

> 📌 **Recommended:** Use **Authy** — it supports cloud backup of your MFA codes.

**Enable MFA on Root:**

1. Log in as **root user**
2. Click your **account name** (top right) → **"Security credentials"**
3. Scroll to **"Multi-factor authentication (MFA)"**
4. Click **"Assign MFA device"**
5. **Device name:** `root-mfa-device`
6. Select **"Authenticator app"** → Click **"Next"**
7. Open your Authenticator App → Tap **"+"** → **"Scan QR code"**
8. Scan the QR code shown on screen
9. Enter **MFA Code 1** from your app
10. Wait 30 seconds for code rotation
11. Enter **MFA Code 2** from your app
12. Click **"Add MFA"**

**Root Login Flow After MFA:**

```
Step 1 → Enter root email & password
         │
         ▼
Step 2 → AWS prompts for MFA code
         │
         ▼
Step 3 → Open authenticator app on phone
         │
         ▼
Step 4 → Enter 6-digit rotating code
         │
         ▼
Step 5 → ✅ Access Granted
```

**MFA Requirements by User Type:**

| User Type | MFA Requirement |
|---|---|
| Root account | ✅ **Mandatory — always** |
| Admin IAM users | ✅ Mandatory |
| Developer IAM users | ✅ Strongly recommended |
| Service accounts | ❌ Not applicable (use Roles instead) |

> ⚠️ **Critical Warning:** Never lose your MFA device without a backup! Account recovery requires contacting AWS Support with identity verification — a process that can take days.  
> **Production solution:** Store MFA backup codes in **AWS Secrets Manager** or a vault like **1Password**.

---

## Lab Cleanup

> 🧹 **Always clean up lab resources to avoid unexpected AWS charges.**

**Step 1 — Delete Users:**

1. IAM Console → **"Users"**
2. Select `bob` → **"Delete"** → Type `bob` to confirm
3. Repeat for `carol` and `alice`

**Step 2 — Delete Groups:**

1. IAM Console → **"User groups"**
2. Delete `prod-developers` → Confirm
3. Delete `prod-auditors` → Confirm
4. Delete `prod-admins` → Confirm

**Step 3 — Delete Custom Policies:**

1. IAM Console → **"Policies"** → Filter: **"Customer managed"**
2. Delete `prod-s3-readonly-specific-bucket`
3. Delete `prod-deny-destructive-ec2-actions`

**Step 4 — Delete IAM Role:**

1. IAM Console → **"Roles"**
2. Search `prod-ec2-s3-read-role`
3. Select → **"Delete"** → Type role name to confirm

**Step 5 — Reset Password Policy:**

1. IAM Console → **"Account settings"**
2. Click **"Edit"** on Password policy → **"Set to default"** → **"Save changes"**

---

## Best Practices Checklist

Use this checklist for every new AWS account or environment:

| Best Practice | Status |
|---|---|
| Root account has MFA enabled | ✅ |
| Root account not used for daily work | ✅ |
| All users organized into Groups | ✅ |
| Permissions assigned to Groups, not Users | ✅ |
| Least privilege applied to all policies | ✅ |
| Custom policies over overly broad managed policies | ✅ |
| Explicit Deny policies used as guardrails | ✅ |
| IAM Roles used for AWS service-to-service access | ✅ |
| No hardcoded credentials in code or servers | ✅ |
| Strong password policy enforced | ✅ |
| Credentials report reviewed regularly | ✅ |
| Access keys rotated every 90 days | ✅ |

---

## What to Learn Next

After completing this lab, continue your AWS security journey:

| Topic | Why Learn It | AWS Service |
|---|---|---|
| **IAM Identity Center (SSO)** | Manage access for large teams with Single Sign-On | AWS IAM Identity Center |
| **Service Control Policies (SCPs)** | Enforce policies across multiple AWS accounts | AWS Organizations |
| **Permission Boundaries** | Limit maximum permissions a role/user can have | IAM |
| **Audit Logging** | Track every IAM action ever taken | AWS CloudTrail |
| **Misconfiguration Detection** | Auto-detect IAM issues in real time | AWS Config |
| **Secrets Management** | Securely store and rotate credentials | AWS Secrets Manager |
| **Vulnerability Assessment** | Identify security issues across your account | AWS Trusted Advisor |

---

## Summary

You have successfully completed a **production-grade AWS IAM lab** covering:

```
Phase 1 ✅ → Users & Groups       (Team structure with least privilege)
Phase 2 ✅ → Testing Permissions  (Verify access controls work)
Phase 3 ✅ → IAM Roles            (Secure service-to-service access)
Phase 4 ✅ → Custom Policies      (Fine-grained least privilege)
Phase 5 ✅ → Security Hardening   (MFA + Password Policy + Audit)
```

> 🎓 You now understand AWS IAM at the level expected of a **Junior to Mid-level Cloud Engineer** in a real production environment!

---

## References

- [AWS IAM Documentation](https://docs.aws.amazon.com/iam/)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Security Hub](https://aws.amazon.com/security-hub/)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/)

---

*Document authored by **Sajal Jana** | AWS IAM Hands-On Lab Guide*
