Absolutely. We’ll start **Phase 1 from scratch**, and I’ll treat this like a hands-on lab rather than just giving you Terraform code.

# Phase 1 — Terraform Fundamentals

### Project 1: AWS S3 Infrastructure with Terraform

We will build:

```text
                    Terraform
                        |
                        v
                  AWS Provider
                        |
          +-------------+-------------+
          |                           |
          v                           v
      S3 Bucket                 S3 Bucket Tags
          |
          +-- Versioning
          |
          +-- Encryption
          |
          +-- Public Access Block
```

### What you'll learn

By the end of this lab you'll understand:

* Terraform installation/check
* AWS provider
* Terraform project structure
* `terraform init`
* `terraform fmt`
* `terraform validate`
* `terraform plan`
* `terraform apply`
* `terraform destroy`
* Resources
* Variables
* Outputs
* Locals
* Terraform state
* AWS tags
* Basic S3 security

---

# Step 1 — Verify your environment

Since you're working on Windows, open **PowerShell**.

Run:

```powershell
terraform version
```

You should get something similar to:

```text
Terraform v1.x.x
on windows_amd64
```

Then check AWS CLI:

```powershell
aws --version
```

And:

```powershell
aws sts get-caller-identity
```

The last command should return your AWS account information.

For example:

```json
{
    "UserId": "...",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/..."
}
```

**Don't paste access keys, secret keys, passwords, or tokens here.** If `aws sts get-caller-identity` works, your AWS CLI authentication is already usable by Terraform.

---

# Step 2 — Create the project

Create a directory:

```powershell
mkdir terraform-aws-phase1
cd terraform-aws-phase1
```

Create these files:

```text
terraform-aws-phase1/
│
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
└── terraform.tfvars
```

We're intentionally keeping this simple for Phase 1.

---

# Step 3 — Create `provider.tf`

Put this inside `provider.tf`:

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}
```

### What is happening?

This tells Terraform:

```text
Terraform
   |
   +-- Required Terraform version
   |
   +-- AWS provider
   |
   +-- AWS region
```

The provider is what allows Terraform to communicate with AWS.

---

# Step 4 — Create `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region where resources will be created"
  type        = string
  default     = "ap-south-1"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "terraform-phase1"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}

variable "bucket_name" {
  description = "Globally unique S3 bucket name"
  type        = string
}
```

Notice that we're **not hardcoding everything inside `main.tf`**.

For example:

```hcl
region = var.aws_region
```

means:

> Get the region from the variable called `aws_region`.

---

# Step 5 — Create `terraform.tfvars`

Use a globally unique bucket name.

For example:

```hcl
aws_region  = "ap-south-1"
project_name = "terraform-phase1"
environment  = "dev"

bucket_name = "farhan-terraform-phase1-2026"
```

⚠️ S3 bucket names are globally unique.

If that name already exists, change it to something unique, such as:

```text
farhan-terraform-phase1-2026-001
```

Don't use your AWS account number in files that you'll commit to a public repository.

---

# Step 6 — Create `locals.tf`

Now let's introduce **locals**.

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = "DevOps"
  }
}
```

Instead of repeatedly writing tags:

```hcl
Project     = "terraform-phase1"
Environment = "dev"
ManagedBy   = "Terraform"
```

we can simply use:

```hcl
tags = local.common_tags
```

This becomes very useful later when we have dozens of AWS resources.

---

# Step 7 — Create `main.tf`

Now we'll create the S3 bucket.

```hcl
resource "aws_s3_bucket" "community_storage" {
  bucket = var.bucket_name

  tags = local.common_tags
}
```

That's your first Terraform resource.

Understand this carefully:

```text
resource "aws_s3_bucket" "community_storage"
          |             |
          |             +-- Terraform local name
          |
          +-- AWS resource type
```

Terraform identifies the resource internally as:

```text
aws_s3_bucket.community_storage
```

---

# Step 8 — Enable S3 versioning

Add this to `main.tf`:

```hcl
resource "aws_s3_bucket_versioning" "community_storage" {
  bucket = aws_s3_bucket.community_storage.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

Notice this:

```hcl
bucket = aws_s3_bucket.community_storage.id
```

Terraform automatically understands that the versioning configuration depends on the bucket.

This is your first introduction to **Terraform resource references/dependencies**.

---

# Step 9 — Enable encryption

Add:

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "community_storage" {
  bucket = aws_s3_bucket.community_storage.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

Now objects stored in this bucket will have server-side encryption enabled.

---

# Step 10 — Block public access

This is important.

Add:

```hcl
resource "aws_s3_bucket_public_access_block" "community_storage" {
  bucket = aws_s3_bucket.community_storage.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

Our Phase 1 bucket therefore looks like:

```text
S3 Bucket
   |
   +-- Versioning
   |
   +-- Encryption
   |
   +-- Public Access Block
   |
   +-- Tags
```

---

# Step 11 — Create outputs

Create `outputs.tf`:

```hcl
output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.community_storage.bucket
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.community_storage.arn
}

output "bucket_region" {
  description = "AWS region of the S3 bucket"
  value       = var.aws_region
}
```

Outputs allow Terraform to show useful information after deployment.

---

# Step 12 — Your complete project

You should now have:

```text
terraform-aws-phase1/
│
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
└── terraform.tfvars
```

And conceptually:

```text
provider.tf
     |
     v
AWS Provider
     |
     +----------------------+
     |                      |
variables.tf             locals.tf
     |                      |
     +----------+-----------+
                |
                v
             main.tf
                |
                v
           S3 Bucket
                |
       +--------+---------+
       |        |         |
       v        v         v
   Versioning Encryption Security
                |
                v
           outputs.tf
```

---

# Step 13 — Initialize Terraform

From your project directory:

```powershell
terraform init
```

You should see something similar to:

```text
Initializing the backend...

Initializing provider plugins...

- Finding hashicorp/aws versions matching "~> 6.0"...
- Installing hashicorp/aws...

Terraform has been successfully initialized!
```

Terraform will create:

```text
.terraform/
.terraform.lock.hcl
```

Don't manually edit these.

---

# Step 14 — Format the code

Run:

```powershell
terraform fmt
```

Terraform will automatically format your `.tf` files.

Then:

```powershell
terraform fmt -check
```

If everything is formatted correctly, it should return successfully.

---

# Step 15 — Validate

Run:

```powershell
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

This checks the Terraform configuration syntax and internal consistency.

---

# Step 16 — Create the execution plan

Now:

```powershell
terraform plan
```

Terraform should show approximately:

```text
Plan: 4 to add, 0 to change, 0 to destroy.
```

The resources are:

```text
aws_s3_bucket.community_storage
aws_s3_bucket_versioning.community_storage
aws_s3_bucket_server_side_encryption_configuration.community_storage
aws_s3_bucket_public_access_block.community_storage
```

### Important concept

`terraform plan` **does not create anything**.

It answers:

> "If I apply this configuration, what will Terraform change?"

---

# Step 17 — Apply

Now:

```powershell
terraform apply
```

Terraform will show the plan again and ask:

```text
Do you want to perform these actions?

  Enter a value:
```

Type:

```text
yes
```

Terraform will create the S3 infrastructure.

You should eventually see:

```text
Apply complete! Resources: 4 added, 0 changed, 0 destroyed.
```

And:

```text
Outputs:

bucket_arn = "arn:aws:s3:::..."
bucket_name = "..."
bucket_region = "ap-south-1"
```

🎉 **You have just created AWS infrastructure using Terraform.**

---

# Step 18 — Verify from AWS CLI

Run:

```powershell
aws s3 ls
```

You should see your bucket.

Then:

```powershell
aws s3api get-bucket-versioning `
  --bucket YOUR_BUCKET_NAME
```

You should get:

```json
{
    "Status": "Enabled"
}
```

Check encryption:

```powershell
aws s3api get-bucket-encryption `
  --bucket YOUR_BUCKET_NAME
```

And public access:

```powershell
aws s3api get-public-access-block `
  --bucket YOUR_BUCKET_NAME
```

This is important because you're learning to verify that **Terraform's desired state actually matches AWS's real state**.

---

# Step 19 — Understand Terraform State

After `terraform apply`, look at your directory:

```text
terraform-aws-phase1/
│
├── .terraform/
├── .terraform.lock.hcl
├── terraform.tfstate
│
├── provider.tf
├── main.tf
├── variables.tf
├── outputs.tf
├── locals.tf
└── terraform.tfvars
```

The most important new file is:

```text
terraform.tfstate
```

Terraform uses this to track infrastructure it manages.

Conceptually:

```text
Terraform Configuration
        |
        v
Desired State
        |
        v
terraform.tfstate
        |
        v
Actual AWS Infrastructure
```

Later, we'll move this state into an **S3 remote backend** and introduce team-safe state management.

---

# Step 20 — Make your first change

This is an important exercise.

Change:

```hcl
Environment = var.environment
```

through your `terraform.tfvars`:

```hcl
environment = "dev"
```

Now change:

```hcl
environment = "dev"
```

to:

```hcl
environment = "test"
```

Run:

```powershell
terraform plan
```

Terraform should detect that the tags need to change.

Then:

```powershell
terraform apply
```

This demonstrates one of Terraform's core principles:

```text
Configuration
      ↓
Terraform Plan
      ↓
Difference
      ↓
Apply
      ↓
AWS
```

---

# Step 21 — Destroy the infrastructure

Once you've verified everything:

```powershell
terraform destroy
```

Terraform will show the resources it intends to delete.

Enter:

```text
yes
```

Then verify:

```powershell
aws s3 ls
```

The bucket should no longer exist.

---

# Phase 1 Challenge

Before we move to Phase 2, I want you to **modify the project yourself**.

Add these two variables:

```text
bucket_owner
enable_versioning
```

Then make Terraform support:

```text
enable_versioning = true
```

and conditionally enable/disable versioning.

You'll need to figure out how to use:

```hcl
count
```

or:

```hcl
for_each
```

for the versioning resource.

This is your first real Terraform exercise rather than copy/paste.

### Your Phase 1 checklist

```text
[ ] terraform version
[ ] aws sts get-caller-identity
[ ] Create project
[ ] provider.tf
[ ] variables.tf
[ ] terraform.tfvars
[ ] locals.tf
[ ] main.tf
[ ] outputs.tf
[ ] terraform init
[ ] terraform fmt
[ ] terraform validate
[ ] terraform plan
[ ] terraform apply
[ ] Verify S3
[ ] Understand terraform.tfstate
[ ] Modify infrastructure
[ ] terraform destroy
[ ] Complete challenge
```

**Don't move to Phase 2 yet.** Do this lab yourself first. If you paste the output of `terraform validate` or `terraform plan` here (with any sensitive information removed), I can review it and troubleshoot it with you like a real Terraform hands-on session.
