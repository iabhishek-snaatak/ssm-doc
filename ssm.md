# 🔐 Secure EC2 Access via AWS SSM with IAM-to-OS User & Group Mapping

## 📌 Overview

This solution enables secure access to EC2 instances using AWS Systems Manager (SSM) Session Manager **without exposing SSH ports**.  

It dynamically creates OS-level users and assigns them to appropriate Linux groups based on their IAM group membership.

This ensures:

- No SSH port exposure (Port 22 closed)
- No SSH keys required
- Centralized IAM-based access control
- Automatic OS user provisioning
- Role-based privilege enforcement at OS level
- Strong security and least-privilege model

---

## 🏗️ Architecture Flow

<img width="1536" height="1024" alt="architecture" src="https://github.com/user-attachments/assets/c609e945-8ed7-4859-9c39-c06705c17a69" />


---

## 👥 IAM Structure

### IAM Groups

Two IAM groups were created:

- DevOps
- Developer

### IAM Users

Users are assigned to groups:

| IAM User | IAM Group |
|--------|------------|
| devops-user | DevOps |
| developer-user | Developer |

---

## 🎭 IAM Roles

Two roles were created:

### DevOpsRole

Permissions:

- SSM Session access
- EC2 access via SSM
- Limited sudo access at OS level

Trust policy allows assume role only by DevOps group.


```

---

### DeveloperRole

Permissions:

- SSM Session access
- Limited system access
- No sudo privileges

Trust policy allows assume role only by Developer group.
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::101936531064:role/DeveloperRole"
        },
        {
            "Effect": "Allow",
            "Action": "ssm:StartSession",
            "Resource": [
                "arn:aws:ssm:*:*:document/SSM-RunAs-dev1",
                "arn:aws:ec2:*:*:instance/*"
            ]
        }
    ]
}
```
---

## 🔐 IAM Policies

### Policy 1: Allow Assume Role (Attach to IAM Groups)

DevOps Group Policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/DevOpsRole"
    }
  ]
}
