## Mujy Comprehensive Guide du , agr meray koi user aye uss ny application ku deploy krna hu toh Mai ussy kaisy Setup kr k du , meri IAM keys bhi safe rahy aur Limits bhi lagi rahi ? Fully Secure  / Steps by Steps each and everything ?

# Complete Secure AWS Setup Guide — New User ke liye Application Deploy karna

---

## Big Picture — Pura Flow Samjho Pehle

```
Tum (Admin/Owner)
        ↓
IAM User banao (Limited permissions)
        ↓
Terraform ke liye restricted access do
        ↓
User deploy kare — sirf allowed resources
        ↓
Tum monitor karo — billing + logs
```

---

# PHASE 1: Admin Account Secure karo (Pehle Apna Ghar Theek karo)

## Step 1: Root Account Lock karo

```
AWS Console Login → Root account (email wala)
→ My Account → MFA Enable karo (Google Authenticator)
→ Root Access Keys — DELETE karo (zaroorat nahi)
```

```
✅ Root account sirf billing dekhne ke liye
✅ Har kaam IAM User se karo
✅ Root keys kabhi mat banao
```

## Step 2: Admin IAM User banao (Apne liye)

```
IAM → Users → Create User
Name: admin-yourname
✅ AWS Console access — Yes
Password: Strong rakho + MFA lagao
Policy: AdministratorAccess (sirf tumhare liye)
```

**Ab Root se logout — Admin IAM User se kaam karo hamesha.**

---

# PHASE 2: New User ke liye Setup

## Step 3: Dedicated IAM User banao

```
IAM → Users → Create User
```

```
Name:        deploy-user-ahsan        ← User ka naam
Console:     No                       ← Console access nahi (sirf CLI/Terraform)
Programmatic: Yes                     ← Access Key milegi
```

## Step 4: IAM Group banao (Best Practice)

```
IAM → User Groups → Create Group
Name: terraform-deployers
```

Agar baad mein aur users aaye — group mein daalo, alag alag policy mat lagao.

## Step 5: Custom Policy banao — Sab Limits ke Saath

```
IAM → Policies → Create Policy → JSON tab
```

```json
{
  "Version": "2012-10-17",
  "Statement": [

    {
      "Sid": "EC2LimitedLaunch",
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-south-1",
          "ec2:InstanceType": [
            "t2.nano",
            "t2.micro",
            "t2.small",
            "t2.medium",
            "t2.large"
          ]
        }
      }
    },

    {
      "Sid": "EC2SupportingResources",
      "Effect": "Allow",
      "Action": "ec2:RunInstances",
      "Resource": [
        "arn:aws:ec2:*:*:subnet/*",
        "arn:aws:ec2:*:*:network-interface/*",
        "arn:aws:ec2:*:*:security-group/*",
        "arn:aws:ec2:*:*:volume/*",
        "arn:aws:ec2:*::image/*",
        "arn:aws:ec2:*:*:key-pair/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-south-1"
        }
      }
    },

    {
      "Sid": "EC2GeneralActions",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeImages",
        "ec2:DescribeKeyPairs",
        "ec2:DescribeSecurityGroups",
        "ec2:DescribeSubnets",
        "ec2:DescribeVpcs",
        "ec2:DescribeInstanceTypes",
        "ec2:DescribeAvailabilityZones",
        "ec2:CreateTags",
        "ec2:StopInstances",
        "ec2:StartInstances",
        "ec2:TerminateInstances",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:CreateKeyPair",
        "ec2:DeleteKeyPair"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "ap-south-1"
        }
      }
    },

    {
      "Sid": "DenyLargeInstances",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": {
        "StringNotEquals": {
          "ec2:InstanceType": [
            "t2.nano",
            "t2.micro",
            "t2.small",
            "t2.medium",
            "t2.large"
          ]
        }
      }
    },

    {
      "Sid": "S3LimitedAccess",
      "Effect": "Allow",
      "Action": [
        "s3:CreateBucket",
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:PutBucketTagging"
      ],
      "Resource": [
        "arn:aws:s3:::deploy-user-ahsan-*",
        "arn:aws:s3:::deploy-user-ahsan-*/*"
      ]
    },

    {
      "Sid": "TerraformStateAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket",
        "s3:DeleteObject"
      ],
      "Resource": [
        "arn:aws:s3:::ahsan-terraform-state",
        "arn:aws:s3:::ahsan-terraform-state/*"
      ]
    },

    {
      "Sid": "DynamoDBForTerraformLock",
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:DeleteItem",
        "dynamodb:DescribeTable"
      ],
      "Resource": "arn:aws:dynamodb:ap-south-1:*:table/terraform-lock"
    },

    {
      "Sid": "IAMReadAndPassOnly",
      "Effect": "Allow",
      "Action": [
        "iam:GetRole",
        "iam:PassRole",
        "iam:ListRoles",
        "iam:GetInstanceProfile",
        "iam:ListInstanceProfiles"
      ],
      "Resource": "*"
    },

    {
      "Sid": "DenyAllDangerousActions",
      "Effect": "Deny",
      "Action": [
        "iam:CreateUser",
        "iam:DeleteUser",
        "iam:UpdateUser",
        "iam:CreateAccessKey",
        "iam:DeleteAccessKey",
        "iam:AttachUserPolicy",
        "iam:DetachUserPolicy",
        "iam:PutUserPolicy",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "organizations:*",
        "account:*",
        "billing:*",
        "aws-portal:*",
        "support:*",
        "cloudtrail:DeleteTrail",
        "cloudtrail:StopLogging",
        "guardduty:DeleteDetector",
        "config:DeleteConfigRule"
      ],
      "Resource": "*"
    },

    {
      "Sid": "DenyOtherRegions",
      "Effect": "Deny",
      "Action": [
        "ec2:*",
        "rds:*",
        "eks:*",
        "lambda:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": "ap-south-1"
        }
      }
    }

  ]
}
```

```
Policy Name: TerraformDeployerPolicy-Ahsan
→ Save karo
→ Group "terraform-deployers" se attach karo
→ User "deploy-user-ahsan" ko group mein daalo
```

---

# PHASE 3: Terraform State Setup (Tumhara Admin kaam)

## Step 6: S3 Bucket banao State ke liye (Tum banao, user nahi)

```bash
# Tumhare admin credentials se run karo
aws s3api create-bucket \
  --bucket ahsan-terraform-state \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Versioning on karo — galti ho toh wapas ja sako
aws s3api put-bucket-versioning \
  --bucket ahsan-terraform-state \
  --versioning-configuration Status=Enabled

# Encryption on karo
aws s3api put-bucket-encryption \
  --bucket ahsan-terraform-state \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

# Public access bilkul band
aws s3api put-public-access-block \
  --bucket ahsan-terraform-state \
  --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,\
     BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

## Step 7: DynamoDB Table banao (State Lock ke liye)

```bash
aws dynamodb create-table \
  --table-name terraform-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

**Ye isliye:** Agar do log ek saath Terraform run karein — conflict na ho.

---

# PHASE 4: Access Keys Safely Dena

## Step 8: Access Keys Generate karo

```
IAM → Users → deploy-user-ahsan
→ Security Credentials tab
→ Create Access Key
→ Use case: CLI
→ Download CSV — safely rakho
```

## Step 9: User ko Keys Safely Dena

```
❌ WhatsApp pe mat bhejo
❌ Email pe mat bhejo  
❌ Slack pe mat bhejo

✅ Password Manager se share karo (1Password, Bitwarden)
✅ AWS Secrets Manager use karo
✅ Encrypted message (Signal)
✅ Direct mil ke dena
```

### AWS Secrets Manager se Share karna (Best Way)

```bash
# Tum store karo
aws secretsmanager create-secret \
  --name "deploy-user-ahsan-keys" \
  --description "Ahsan ke liye terraform credentials" \
  --secret-string '{
    "aws_access_key_id": "AKIA...",
    "aws_secret_access_key": "xyz..."
  }' \
  --region ap-south-1

# User ko sirf secret read karne ki permission do
# Wo console se ya CLI se le sakta hai
```

---

# PHASE 5: User ke liye Terraform Project Setup

## Step 10: Folder Structure

```
my-app-infrastructure/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
├── terraform.tfvars        ← .gitignore mein daalo
├── .gitignore
└── modules/
    ├── ec2/
    │   ├── main.tf
    │   └── variables.tf
    └── s3/
        ├── main.tf
        └── variables.tf
```

## Step 11: backend.tf — State Remote Store karo

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "ahsan-terraform-state"
    key            = "myapp/production/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraform-lock"
    encrypt        = true
  }
}
```

## Step 12: variables.tf — Validation ke Saath

```hcl
# variables.tf

variable "aws_region" {
  description = "AWS Region"
  type        = string
  default     = "ap-south-1"

  validation {
    condition     = var.aws_region == "ap-south-1"
    error_message = "Sirf ap-south-1 region allowed hai."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"

  validation {
    condition = contains([
      "t2.nano", "t2.micro", "t2.small",
      "t2.medium", "t2.large"
    ], var.instance_type)
    error_message = "Sirf t2.large tak allowed hai."
  }
}

variable "instance_count" {
  description = "Kitne EC2 instances chahiye"
  type        = number
  default     = 1

  validation {
    condition     = var.instance_count <= 3
    error_message = "Maximum 3 instances allowed hain."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "staging"], var.environment)
    error_message = "Sirf dev ya staging allowed hai, production nahi."
  }
}
```

## Step 13: main.tf — Complete Setup

```hcl
# main.tf

terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region  = var.aws_region
  profile = "deploy-user-ahsan"

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
      Owner       = "ahsan"
      Project     = "my-app"
    }
  }
}

# EC2 Instance
resource "aws_instance" "app_server" {
  count         = var.instance_count
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type

  tags = {
    Name = "app-server-${var.environment}-${count.index + 1}"
  }

  lifecycle {
    prevent_destroy = false

    precondition {
      condition = contains([
        "t2.nano", "t2.micro", "t2.small",
        "t2.medium", "t2.large"
      ], var.instance_type)
      error_message = "Instance type allowed range se bahar hai."
    }
  }
}

# Latest Amazon Linux AMI automatically fetch karo
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

## Step 14: .gitignore — Zaroor Lagao

```gitignore
# .gitignore

# Terraform files
.terraform/
.terraform.lock.hcl
*.tfstate
*.tfstate.backup
*.tfstate.*.backup

# Variables (sensitive data)
terraform.tfvars
*.auto.tfvars
override.tf

# Keys aur secrets
*.pem
*.key
.env
.env.*
secrets.tf
credentials

# OS files
.DS_Store
Thumbs.db
```

## Step 15: terraform.tfvars — User ki Values

```hcl
# terraform.tfvars — .gitignore mein hai, safe hai

aws_region     = "ap-south-1"
instance_type  = "t2.micro"
instance_count = 1
environment    = "dev"
```

---

# PHASE 6: User ke Machine pe Setup

## Step 16: AWS CLI Configure karo

```bash
# User apni machine pe run kare
aws configure --profile deploy-user-ahsan
```

```
AWS Access Key ID:     [Jo tumne diya]
AWS Secret Access Key: [Jo tumne diya]
Default region:        ap-south-1
Default output format: json
```

## Step 17: Terraform Commands — User Kya Run Kare

```bash
# 1. Pehli baar — providers download karo
terraform init

# 2. Kya banega — preview dekho (kuch nahi banta)
terraform plan

# 3. Actual deploy karo
terraform apply

# Type karo: yes

# 4. Sab hatana ho toh
terraform destroy
```

---

# PHASE 7: Monitoring aur Alerts (Tumhara Kaam)

## Step 18: Budget Alert lagao

```
AWS Console → Billing → Budgets → Create Budget
→ Budget type: Cost
→ Amount: $50/month
→ Alert: 80% pe ($40 pe) email aaye
→ Email: tumhara@email.com
```

## Step 19: CloudTrail — Sab Activity Log ho

```bash
# Tum banao admin se
aws cloudtrail create-trail \
  --name main-trail \
  --s3-bucket-name ahsan-terraform-state \
  --include-global-service-events \
  --is-multi-region-trail \
  --region ap-south-1

aws cloudtrail start-logging \
  --name main-trail \
  --region ap-south-1
```

**Isse:** User ny kya create kiya, kya delete kiya — sab log mein rahega.

## Step 20: AWS Config Rule — Instance Type Check

```bash
# Agar koi manually console sy bada instance banaye
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "approved-instance-types",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "DESIRED_INSTANCE_TYPE"
    },
    "InputParameters": "{\"instanceType\":\"t2.nano,t2.micro,t2.small,t2.medium,t2.large\"}"
  }' \
  --region ap-south-1
```

---

# Complete Checklist

```
ADMIN SETUP
  ✅ Root account MFA on
  ✅ Root keys deleted
  ✅ Admin IAM User with MFA
  ✅ CloudTrail enabled
  ✅ Budget alert set

USER SETUP
  ✅ IAM User created (no console access)
  ✅ Custom policy with all limits
  ✅ Instance type restricted (max t2.large)
  ✅ Region locked (ap-south-1 only)
  ✅ Dangerous actions denied
  ✅ Access keys via secure channel

TERRAFORM SETUP
  ✅ Remote state in S3 (encrypted)
  ✅ State lock in DynamoDB
  ✅ .gitignore complete
  ✅ Variable validation
  ✅ No hardcoded keys

MONITORING
  ✅ CloudTrail logging
  ✅ Budget alert
  ✅ AWS Config rules
```

---

## Security Layers — Summary

```
Layer 1: IAM Policy      → Kya kar sakta hai
Layer 2: Region Lock     → Kahan kar sakta hai  
Layer 3: Instance Limit  → Kitna bara bana sakta hai
Layer 4: Terraform Valid → Code level check
Layer 5: CloudTrail      → Sab kuch log
Layer 6: Budget Alert    → Overspend protection
Layer 7: AWS Config      → Continuous compliance check
```

**7 layers of security = Fully protected setup.** User deploy kar sakta hai — tumhara account aur billing dono safe.
