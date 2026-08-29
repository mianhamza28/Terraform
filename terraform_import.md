# Multiple Resources Import Karna (Bulk Import Strategy)

Jab aapke paas bohot saare resources hon (5 EC2, 3 VPC, ALB, S3, etc), to ek-ek karke manually import karna time-consuming hota hai. Iske liye kuch better tareeqe hain:

## Method 1: Terraform v1.5+ `import` Block + Auto Config Generation (Best Tareeqa)

Ye sabse modern aur recommended approach hai kyun ke ye **automatically HCL code bhi generate** kar deta hai.

### Step 1: Sab resources ke liye import blocks likhen

Ek `imports.tf` file banayen:

```hcl
# EC2 Instances
import {
  to = aws_instance.web1
  id = "i-0123456789abcdef0"
}
import {
  to = aws_instance.web2
  id = "i-0123456789abcdef1"
}
import {
  to = aws_instance.web3
  id = "i-0123456789abcdef2"
}
import {
  to = aws_instance.web4
  id = "i-0123456789abcdef3"
}
import {
  to = aws_instance.web5
  id = "i-0123456789abcdef4"
}

# VPCs
import {
  to = aws_vpc.vpc1
  id = "vpc-011122233"
}
import {
  to = aws_vpc.vpc2
  id = "vpc-011122234"
}
import {
  to = aws_vpc.vpc3
  id = "vpc-011122235"
}

# Application Load Balancer
import {
  to = aws_lb.my_alb
  id = "arn:aws:elasticloadbalancing:region:account-id:loadbalancer/app/my-alb/xxxxxxxx"
}

# S3 Bucket
import {
  to = aws_s3_bucket.my_bucket
  id = "my-existing-bucket-name"
}
```

### Step 2: Auto-generate karein `.tf` configuration

```bash
terraform plan -generate-config-out=generated_resources.tf
```

Ye command automatically saare resources ka **HCL code khud likh degi** ek naye file `generated_resources.tf` mein — aapko manually attributes likhne ki zaroorat nahi!

### Step 3: Review & Apply

```bash
terraform plan
terraform apply
```

Bas! Sab resources state mein aa jayengi aur config bhi ready ho jayegi.

---

## Method 2: Terraformer Tool (Sabse Fast Bulk Import)

Agar resources ki tadaad bohot zyada ho (jaise poora AWS account import karna ho), to **Terraformer** open-source tool use karen.

### Install
```bash
brew install terraformer
# ya GitHub se binary download karen
```

### Bulk Import Example

```bash
# Saare EC2 instances import karen
terraformer import aws --resources=ec2_instance --regions=us-east-1

# VPCs import karen
terraformer import aws --resources=vpc --regions=us-east-1

# S3 buckets import karen
terraformer import aws --resources=s3 --regions=us-east-1

# ALB import karen
terraformer import aws --resources=alb --regions=us-east-1

# Ek saath multiple resource types
terraformer import aws --resources=ec2_instance,vpc,s3,alb --regions=us-east-1
```

Ye tool automatically **`.tf` files** aur **state** dono generate kar deta hai — bohot time save hota hai.

---

## Method 3: Bash Script (CLI import loop)

Agar aap purana `terraform import` CLI approach use karna chahte hain, to script bana kar loop chala sakte hain:

```bash
#!/bin/bash

# EC2 instances list
instances=("i-001" "i-002" "i-003" "i-004" "i-005")
count=1

for id in "${instances[@]}"
do
  terraform import aws_instance.web$count $id
  count=$((count+1))
done

# VPCs
vpcs=("vpc-011" "vpc-022" "vpc-033")
count=1
for id in "${vpcs[@]}"
do
  terraform import aws_vpc.vpc$count $id
  count=$((count+1))
done

# S3
terraform import aws_s3_bucket.my_bucket my-bucket-name

# ALB
terraform import aws_lb.my_alb arn:aws:elasticloadbalancing:...
```

⚠️ Is method mein aapko manually `.tf` file mein resource blocks pehle se likhne parenge (empty ya basic), warna import fail hoga.

---

## Konsa Method Best Hai?

| Situation | Best Approach |
|---|---|
| Terraform v1.5+ hai, thoda control chahiye | `import` block + `-generate-config-out` |
| Bohot zyada resources (poora account) | Terraformer tool |
| Purana Terraform version, chhoti setup | Bash script + CLI import |

## Pro Tips 💡
- Har resource import karne ke baad **`terraform plan`** zaroor chalayen taake verify ho ke koi "diff/changes" na dikhe.
- Import se pehle **state ka backup** le lein (`terraform state pull > backup.tfstate`).
- Agar resources bohot complex hain (jaise security groups, IAM roles attached), to Terraformer zyada reliable rahega kyun ke wo dependencies bhi handle karta hai.

Agar aap batayen ke aapka Terraform version kya hai aur cloud provider AWS hi hai, to main aapke specific case ke hisaab se ready-made script bhi bana sakta hoon.
