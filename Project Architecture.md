# Hands-on Terraform Project: AWS Production-Style Platform

Since your goal is **beginner → expert Terraform**, I recommend **not doing many unrelated small projects**. Instead, build **one realistic AWS application platform** and progressively enhance it.

## Project: DevOps Community Platform on AWS

We will build an application similar to an **IT Community / Developer Community platform** where users can:

* Register/login
* Create profiles
* Create technical posts
* Upload PDFs, ZIPs, images and architecture diagrams
* Search/download community resources
* Use REST APIs
* Store application data
* Process uploaded files asynchronously
* Monitor the platform
* Deploy through CI/CD

The important part is that **you will build the infrastructure entirely with Terraform**.

---

# 1. What you will learn

By the end, you should have practiced:

| Level        | Terraform/AWS concepts                   |
| ------------ | ---------------------------------------- |
| Beginner     | Providers, resources, variables, outputs |
| Beginner+    | Data sources, locals, expressions        |
| Intermediate | VPC, subnets, security groups, IAM       |
| Intermediate | EC2, ALB, Auto Scaling                   |
| Intermediate | S3, RDS, CloudFront                      |
| Advanced     | Terraform modules                        |
| Advanced     | Remote state + S3 backend                |
| Advanced     | DynamoDB state locking concepts          |
| Advanced     | IAM least privilege                      |
| Advanced     | Secrets Manager / SSM                    |
| Advanced     | CloudWatch                               |
| Advanced     | Lambda + SQS                             |
| Advanced     | Event-driven architecture                |
| Expert       | Multi-environment architecture           |
| Expert       | CI/CD for Terraform                      |
| Expert       | Terraform testing/validation             |
| Expert       | Policy/security scanning                 |
| Expert       | Disaster recovery                        |
| Expert       | Cost optimization                        |
| Expert       | Production-grade architecture            |

---

# 2. Target Architecture

We can gradually arrive at something like this:

![Image](https://images.openai.com/static-rsc-4/xET1yBbTwU_AEg_56BjkPHVq9xqyao4GQObmdafRNOil0i0g4_-tlr16P0sE1qG5SnbXsju46rRciSQaW7aCjBXZusMkqRhIcQuqLwFDNp-8ZTQbgBHMTdYx8JGTpegxCAu8Db1JKCPB4dnlcG8Kj9Io9dr2O9rIeDwJMl9Ki3w-cTqQvqYTCFMgiwsifWty?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/XyX5W5GVJaXQtzbT3JwNRRCOemC8R8ot6FyoSc9h9tIC3oJQ-SvjPYd4g_G9V-GevX3sksPhAcD58JQ9UOJVrVU3g_HPjBAo_E90yJ_NTdvddgG1Rw4tCOPlu6nwAZQ0rqiEwYr_dCAvCmIHPIy9wCRS52LekFql4TVh8GJPOwg-PY0fsef4yQ-TB2yiuECv?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fUJhYkRNHVx3ydc7d9gbozyY8Yfi_x5MmG6aeiF1y-OeuahvQpsjEg8a0keghIjEcaNObChJ8yh27mrSTem7NX-8u4QhcVZgq9D0k3izs7SwUvqfUw-Dyrf9gJ0Ugs4JXBA_uZbfnm6YpOOzsRpB4yLiW7DQK4amfHIEpTYdHqfrV0PJjyZ-HvMnnzSO5eFm?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/5he7Iie72ZlxqxiB4oCX3gVODRV_IxGKf8Y5hl7P5VhgOgg51NKcHEWXR2624s4Ev4n8PllbbjzDPAPlURdzZTX1wuQKkxYvovJpjIOvxwh4YAQHZZ7-MCPLTMeU27MUwgtCgWTxk0kZpgudhe9dRznOzOjMu2ASAGYj9MB1JHGZbPr5DBl3S0CAdj8JuRpU?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/nc_VDNRXXgGxzUl-HKcgJDzYC9UReV3FBOq4u7y8LxINQT0ixhvdufAfJdGY7CII2oUsIZMAS08D47yHw1NTPP8MvuFmMwHwBVMoaPAUCapNsuYiJQUP8fxw7_h-cniKVNFE3tqH-gfLOjkCB82VWpjhNNBIQBqpPM3W8jUwFz1qXy-bJayCEQaURInWCUr4?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/tvTxF_42hiGVQEoRXcXgjOerMlvp0CadfzUSkLJBE5ECjg7hDFewNvp6MrJZ0UhwDm_cvze_uQecg8G0qj990l1TdByoPxUkj0Cmn6d--caByjw8RXALWnMlGYlov3p1a1Cga_I5_ku-Ja1bpZ_K0nMT2a4S6ouBfSRMh4Rf9WY__j4ALRtB9SH9SIXQsB3s?purpose=fullsize)

```text
                         Internet
                            |
                            v
                    +----------------+
                    |   Route 53     |
                    +-------+--------+
                            |
                            v
                    +----------------+
                    |   CloudFront   |
                    +---+--------+---+
                        |        |
              Static    |        | API
              Content   |        |
                        v        v
                  +---------+  +----------------+
                  |   S3    |  | Application LB |
                  +---------+  +-------+--------+
                                      |
                         +------------+------------+
                         |                         |
                         v                         v
                  +-------------+          +-------------+
                  | EC2 / ECS   |          | EC2 / ECS   |
                  | Application |          | Application |
                  +------+------+          +------+------+
                         |                        |
                         +------------+-----------+
                                      |
                     +----------------+----------------+
                     |                                 |
                     v                                 v
               +-----------+                     +-----------+
               |   RDS     |                     |  Redis    |
               | PostgreSQL|                     | ElastiCache|
               +-----------+                     +-----------+

                              File Upload
                                  |
                                  v
                            +-----------+
                            |    S3     |
                            +-----+-----+
                                  |
                              S3 Event
                                  |
                                  v
                            +-----------+
                            |   SQS     |
                            +-----+-----+
                                  |
                                  v
                            +-----------+
                            |  Lambda   |
                            +-----+-----+
                             |    |    |
                             v    v    v
                           Scan Process Notify
```

But **we won't create all of this on day one**.

We'll build it in stages.

---

# 3. Project Roadmap

## Phase 1 — Terraform Fundamentals

First, create something very simple.

### AWS resources

```text
Terraform
   |
   +-- Provider
   |
   +-- S3 Bucket
   |
   +-- IAM
   |
   +-- Outputs
```

You'll learn:

```hcl
terraform init
terraform plan
terraform apply
terraform destroy
```

And:

```hcl
provider
resource
variable
output
locals
data
```

### First exercise

Create:

```text
terraform-project/
│
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── README.md
```

Create an S3 bucket using Terraform.

Example:

```hcl
resource "aws_s3_bucket" "community" {
  bucket = var.bucket_name

  tags = {
    Name        = "community-storage"
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

Then:

```bash
terraform init
terraform validate
terraform plan
terraform apply
```

Finally:

```bash
terraform destroy
```

---

# 4. Phase 2 — Build the VPC

Now we start doing real infrastructure.

Create:

```text
VPC
 |
 +-- Internet Gateway
 |
 +-- Public Subnet AZ-A
 |
 +-- Public Subnet AZ-B
 |
 +-- Private Subnet AZ-A
 |
 +-- Private Subnet AZ-B
 |
 +-- Route Tables
 |
 +-- NAT Gateway
 |
 +-- Security Groups
```

You'll learn:

* CIDR blocks
* subnet design
* availability zones
* route tables
* Internet Gateway
* NAT Gateway
* security groups
* Terraform dependencies
* `for_each`
* `count`
* data sources

Example:

```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.project_name}-vpc"
  }
}
```

---

# 5. Phase 3 — EC2 Application Servers

Create:

```text
                    ALB
                     |
             +-------+-------+
             |               |
           EC2-1           EC2-2
             |               |
             +-------+-------+
                     |
                    RDS
```

Terraform resources:

```text
aws_instance
aws_launch_template
aws_autoscaling_group
aws_lb
aws_lb_listener
aws_lb_target_group
aws_security_group
```

You'll learn:

* AMI lookup
* EC2
* user data
* Launch Templates
* Auto Scaling
* Application Load Balancer
* target groups
* health checks

---

# 6. Phase 4 — Database Layer

Now introduce RDS.

```text
Application
     |
     v
   ALB
     |
     v
   EC2
     |
     v
+------------+
| PostgreSQL |
|    RDS     |
+------------+
```

Terraform:

```hcl
resource "aws_db_instance" "postgres" {
  engine         = "postgres"
  engine_version = var.postgres_version

  instance_class        = var.db_instance_class
  allocated_storage     = 20
  max_allocated_storage = 100

  db_name  = var.database_name
  username = var.database_username
  password = var.database_password

  multi_az = true

  skip_final_snapshot = true
}
```

But this phase should also teach you an important production lesson:

**Don't keep database passwords directly in `.tf` files.**

We'll later move secrets into:

```text
AWS Secrets Manager
       or
AWS Systems Manager Parameter Store
```

---

# 7. Phase 5 — Terraform Modules

This is where your Terraform skills start becoming serious.

Instead of:

```text
main.tf
```

we create:

```text
terraform/
│
├── environments/
│   │
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   │
│   ├── staging/
│   │   └── ...
│   │
│   └── prod/
│       └── ...
│
└── modules/
    │
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    │
    ├── ec2/
    │
    ├── alb/
    │
    ├── rds/
    │
    ├── s3/
    │
    └── iam/
```

Then:

```hcl
module "vpc" {
  source = "../../modules/vpc"

  project_name = var.project_name
  environment  = var.environment
  vpc_cidr     = var.vpc_cidr
}
```

This teaches you the difference between:

**Infrastructure code** and **reusable infrastructure modules**.

---

# 8. Phase 6 — Multiple Environments

Now create:

```text
DEV
 |
 +-- VPC
 +-- EC2
 +-- RDS
 +-- S3

STAGING
 |
 +-- VPC
 +-- EC2
 +-- RDS
 +-- S3

PRODUCTION
 |
 +-- VPC
 +-- EC2
 +-- RDS
 +-- S3
```

For example:

```text
environments/
├── dev/
├── staging/
└── prod/
```

You will learn:

```text
variable management
environment-specific configuration
module reuse
Terraform workspaces vs directory-based environments
```

We'll also discuss **when not to use Terraform workspaces**, which is an important real-world topic.

---

# 9. Phase 7 — Remote Terraform State

Now move away from:

```text
terraform.tfstate
```

on your laptop.

Architecture:

```text
Developer
    |
    v
Terraform
    |
    v
S3 Terraform State
    |
    v
State Locking
```

We'll implement:

```text
S3
 |
 +-- Terraform state
 |
 +-- versioning
 |
 +-- encryption
 |
 +-- restricted IAM access
```

You'll learn:

```hcl
terraform {
  backend "s3" {
    bucket = "..."
    key    = "..."
    region = "..."
  }
}
```

This is a major step toward professional Terraform usage.

---

# 10. Phase 8 — S3 File Upload Platform

This connects directly to the IT Community application you were working on.

We'll build:

```text
User
 |
 v
CloudFront
 |
 v
S3
 |
 +-- PDF
 +-- ZIP
 +-- Images
 +-- Architecture diagrams
 +-- Documentation
```

Terraform manages:

```text
S3 bucket
Bucket policy
Encryption
Versioning
Lifecycle rules
CORS
CloudFront
Origin Access Control
IAM
```

We'll also implement:

```text
Upload
   |
   v
S3
   |
   v
Event
   |
   v
SQS
   |
   v
Lambda
   |
   +---- Malware scanning
   |
   +---- File validation
   |
   +---- Metadata extraction
   |
   +---- Processing
```

This becomes a very good portfolio project.

---

# 11. Phase 9 — Security

Now harden the infrastructure.

We'll introduce:

```text
IAM
KMS
Secrets Manager
CloudTrail
GuardDuty
Security Hub
AWS Config
VPC Flow Logs
CloudWatch
```

Example:

```text
                AWS Account
                     |
        +------------+------------+
        |                         |
      IAM                       KMS
        |                         |
        v                         v
 Application ----------------> Encryption
        |
        v
      Secrets
```

You'll learn:

* least privilege
* IAM roles
* IAM policies
* KMS encryption
* secrets
* audit logging
* security monitoring

---

# 12. Phase 10 — Observability

We'll build:

```text
Application
     |
     +---- CloudWatch Logs
     |
     +---- CloudWatch Metrics
     |
     +---- CloudWatch Alarms
     |
     +---- CloudTrail
     |
     +---- X-Ray
```

For example:

```text
ALB
 |
 +-- 5xx errors
 |
 +-- latency
 |
 +-- request count
 |
 v
CloudWatch Alarm
 |
 v
SNS
 |
 v
Notification
```

Terraform will create the entire monitoring stack.

---

# 13. Phase 11 — CI/CD for Terraform

This is where we make the project DevOps-oriented.

Architecture:

```text
Developer
    |
    v
GitHub
    |
    v
Pull Request
    |
    v
GitHub Actions
    |
    +-- terraform fmt
    |
    +-- terraform validate
    |
    +-- terraform plan
    |
    +-- security scan
    |
    v
Approval
    |
    v
terraform apply
```

We'll introduce:

```text
GitHub Actions
Terraform
AWS IAM/OIDC
TFLint
Checkov
tfsec
Terraform validation
Terraform plan
Terraform apply
```

And importantly, **we won't store AWS access keys in GitHub**.

We'll use:

```text
GitHub OIDC
      |
      v
AWS IAM Role
      |
      v
AWS
```

---

# 14. Phase 12 — Terraform Testing

Now we'll test the infrastructure itself.

You'll learn:

```text
terraform test
```

along with validation and static analysis.

Pipeline:

```text
Terraform Code
      |
      v
terraform fmt
      |
      v
terraform validate
      |
      v
TFLint
      |
      v
Checkov
      |
      v
terraform test
      |
      v
terraform plan
      |
      v
Approval
      |
      v
terraform apply
```

---

# 15. Phase 13 — Advanced Terraform

Once the core infrastructure works, we'll deliberately introduce advanced Terraform concepts.

### Dynamic blocks

```hcl
dynamic "ingress" {
  for_each = var.ingress_rules

  content {
    from_port = ingress.value.from_port
    to_port   = ingress.value.to_port
    protocol  = ingress.value.protocol
  }
}
```

### `for_each`

```hcl
resource "aws_s3_bucket" "buckets" {
  for_each = var.buckets

  bucket = each.value
}
```

### `count`

```hcl
count = var.create_resource ? 1 : 0
```

### Conditional expressions

```hcl
instance_type = var.environment == "prod"
  ? "m7i.large"
  : "t3.micro"
```

### `locals`

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

### Data sources

```hcl
data "aws_availability_zones" "available" {}
```

### Outputs

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

---

# 16. Phase 14 — Production Architecture

Eventually the project becomes:

```text
                         Internet
                            |
                         Route53
                            |
                       CloudFront
                       /        \
                      /          \
                    S3            ALB
                                  |
                           Private Subnets
                                  |
                    +-------------+-------------+
                    |                           |
                  ECS                         ECS
                Service                     Service
                    |                           |
                    +-------------+-------------+
                                  |
                         +--------+--------+
                         |                 |
                        RDS             Redis
                         |
                         |
                  Multi-AZ Database


File Upload
     |
     v
    S3
     |
     v
   Event
     |
     v
    SQS
     |
     v
   Lambda
     |
     +---- Malware Scan
     +---- Processing
     +---- Metadata
     +---- Notification


Observability
     |
     +---- CloudWatch
     +---- CloudTrail
     +---- X-Ray
     +---- Alarms
     +---- SNS


Security
     |
     +---- IAM
     +---- KMS
     +---- WAF
     +---- GuardDuty
     +---- Security Hub
```

---

# 17. Final Terraform Repository

By the end, your GitHub repository could look like:

```text
aws-community-platform-terraform/
│
├── README.md
│
├── docs/
│   ├── architecture.md
│   ├── networking.md
│   ├── security.md
│   ├── disaster-recovery.md
│   └── cost-optimization.md
│
├── modules/
│   │
│   ├── vpc/
│   ├── security-groups/
│   ├── iam/
│   ├── alb/
│   ├── ec2/
│   ├── ecs/
│   ├── rds/
│   ├── redis/
│   ├── s3/
│   ├── cloudfront/
│   ├── route53/
│   ├── lambda/
│   ├── sqs/
│   ├── cloudwatch/
│   └── waf/
│
├── environments/
│   │
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   │
│   ├── staging/
│   │   └── ...
│   │
│   └── prod/
│       └── ...
│
├── bootstrap/
│   └── terraform-state/
│
├── tests/
│
├── .github/
│   └── workflows/
│       ├── terraform-plan.yml
│       ├── terraform-apply.yml
│       └── security-scan.yml
│
├── .gitignore
├── .terraform.lock.hcl
└── Makefile
```

---

# 18. Your Learning Path

I suggest we follow this exact progression:

```text
LEVEL 1
Terraform Basics
        ↓
LEVEL 2
S3 + IAM
        ↓
LEVEL 3
VPC
        ↓
LEVEL 4
EC2
        ↓
LEVEL 5
ALB + Auto Scaling
        ↓
LEVEL 6
RDS
        ↓
LEVEL 7
Terraform Modules
        ↓
LEVEL 8
Multiple Environments
        ↓
LEVEL 9
Remote State
        ↓
LEVEL 10
S3 + CloudFront
        ↓
LEVEL 11
Lambda + SQS
        ↓
LEVEL 12
Security
        ↓
LEVEL 13
Monitoring
        ↓
LEVEL 14
CI/CD
        ↓
LEVEL 15
Terraform Testing
        ↓
LEVEL 16
Production Architecture
        ↓
LEVEL 17
Cost Optimization
        ↓
LEVEL 18
Disaster Recovery
        ↓
LEVEL 19
Advanced Terraform
        ↓
LEVEL 20
Production-ready AWS IaC
```

## How I recommend we do the hands-on work

For **every phase**, I can give you:

1. **Architecture diagram**
2. **Real-world requirement**
3. **AWS services**
4. **Terraform folder structure**
5. **Complete Terraform code**
6. **Explanation of every `.tf` file**
7. `terraform init`
8. `terraform fmt`
9. `terraform validate`
10. `terraform plan`
11. `terraform apply`
12. AWS console verification
13. Troubleshooting exercises
14. `terraform destroy`
15. **Interview questions**
16. **Challenge exercises**
17. Then move to the next level

This way, you're not just copying Terraform—you'll understand **why each resource exists, how Terraform manages it, and how the design changes as the system becomes production-grade**.

### Start here: Phase 1

Our first hands-on milestone should be:

**`Terraform Beginner Project #1 — AWS S3 + IAM + Variables + Outputs + Tags`**

Then we'll deliberately make you modify and troubleshoot it before moving to the VPC. That will give you the foundation needed for the advanced phases.
