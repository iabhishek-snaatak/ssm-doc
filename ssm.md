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


---

### DeveloperRole

Permissions:

- SSM Session access
- Limited system access
- No sudo privileges

Trust policy allows assume role only by Developer group.

---

## 🔐 IAM Policies

### Policy 1: Allow Assume Role (Attach to IAM Groups)

DevOps Group Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/DevOpsRole",
            "Condition": {
                "StringEquals": {
                    "sts:RoleSessionName": "${aws:username}"
                }
            }
        }
    ]
}
```
Developer Group Policy:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "sts:AssumeRole",
            "Resource": "arn:aws:iam::<ACCOUNT_ID>:role/DeveloperRole",
            "Condition": {
                "StringEquals": {
                    "sts:RoleSessionName": "${aws:username}"
                }
            }
        }
    ]
}
```
Policy 2: SSM Session Access (Attach to Roles) on EC2
This enables SSM access to EC2.And fetch instance,role session id details 

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeAssociation",
                "ssm:GetDeployablePatchSnapshotForInstance",
                "ssm:GetDocument",
                "ssm:DescribeDocument",
                "ssm:GetManifest",
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:ListAssociations",
                "ssm:ListInstanceAssociations",
                "ssm:PutInventory",
                "ssm:PutComplianceItems",
                "ssm:PutConfigurePackageResult",
                "ssm:UpdateAssociationStatus",
                "ssm:UpdateInstanceAssociationStatus",
                "ssm:UpdateInstanceInformation"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssmmessages:CreateControlChannel",
                "ssmmessages:CreateDataChannel",
                "ssmmessages:OpenControlChannel",
                "ssmmessages:OpenDataChannel"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2messages:AcknowledgeMessage",
                "ec2messages:DeleteMessage",
                "ec2messages:FailMessage",
                "ec2messages:GetEndpoint",
                "ec2messages:GetMessages",
                "ec2messages:SendReply"
            ],
            "Resource": "*"
        },
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ssm:DescribeSessions",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:ListGroupsForUser",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "ssm:DescribeInstanceInformation",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "sts:GetCallerIdentity",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeSessions",
                "ssm:TerminateSession"
            ],
            "Resource": "*"
        }
    ]
}
    ]
}
```
## 👤 Linux OS Level Configuration
Step 3: Create Linux Groups

sudo groupadd devops
sudo groupadd developer

Step 4: Configure Sudo Permissions

Create file: sudo nano /etc/sudoers.d/devops

Add:

%devops ALL=(ALL) ALL
%devops ALL=(ALL) !/bin/rm -rf *

Developer group has no sudo access.

## ⚙️ Automatic User Provisioning Script

Script Path:

sudo nano /usr/local/bin/dynamic-ssm-user.sh

add this and make it executable 

```
#!/bin/bash
set -e  # Exit on error

# Get IMDSv2 token
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Get instance ID
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)

------------------------------------------------------
SESSION_ID=$(aws ssm describe-sessions \
  --state Active \
  --filters key=Target,value=$INSTANCE_ID \
  --query "Sessions | sort_by(@,&StartDate)[-1].SessionId" \
  --output text 2>/dev/null || echo "")

# Get active SSM session owner (full ARN)
IAM_ARN=$(aws ssm describe-sessions \
  --state Active \
  --filters key=Target,value=$INSTANCE_ID \
  --query "Sessions | sort_by(@,&StartDate)[-1].Owner" \
  --output text 2>/dev/null || echo "None")

IAM_USER=$(echo "$IAM_ARN" | awk -F/ '{print $NF}')

# If no valid IAM user, exit silently (stay as ssm-user)
if [[ -z "$IAM_USER" || "$IAM_USER" == "None" || "$IAM_USER" == "null" ]]; then
  exit 0
fi

# Create user if they don't exist
if ! id "$IAM_USER" >/dev/null 2>&1; then
  sudo useradd -m -s /bin/bash "$IAM_USER"

  if [[ -d /home/ssm-user/.ssh ]]; then
    sudo cp -r /home/ssm-user/.ssh /home/$IAM_USER/
    sudo chown -R $IAM_USER:$IAM_USER /home/$IAM_USER/.ssh
  fi
fi

# Extract role name from ARN and add user to appropriate group
if [[ "$IAM_ARN" == *"DevOpsRole"* ]]; then
  if ! getent group devops >/dev/null 2>&1; then
    sudo groupadd devops
    logger "Created devops group"
  fi

  if ! id -nG "$IAM_USER" | grep -qw devops; then
    sudo usermod -aG devops "$IAM_USER"
    logger "Added $IAM_USER to devops group (Role: DevOpsRole)"
  fi

elif [[ "$IAM_ARN" == *"DeveloperRole"* ]]; then
  if ! getent group developer >/dev/null 2>&1; then
    sudo groupadd developer
    logger "Created developer group"
  fi

  if ! id -nG "$IAM_USER" | grep -qw developer; then
    sudo usermod -aG developer "$IAM_USER"
    logger "Added $IAM_USER to developer group (Role: DeveloperRole)"
  fi
fi

logger "Switching SSM session from ssm-user to $IAM_USER"


sudo -u "$IAM_USER" -i

# When IAM user exits, this runs
if [[ -n "$SESSION_ID" && "$SESSION_ID" != "None" ]]; then
  aws ssm terminate-session --session-id "$SESSION_ID" >/dev/null 2>&1 || true
fi

exit 0
```
