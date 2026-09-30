Perfect. Since you destroyed Phases 2 and 3, we’ll treat **Phase 4 as a fresh deployment**, but we’ll recreate the minimum networking required for RDS.

The target architecture is:

```text
                         AWS VPC
                    10.0.0.0/16
                           |
          +----------------+----------------+
          |                                 |
          v                                 v
   Private DB Subnet A              Private DB Subnet B
      AZ-a: 10.0.11.0/24              AZ-b: 10.0.12.0/24
          |                                 |
          +----------------+----------------+
                           |
                    DB Subnet Group
                           |
                           v
                  +-------------------+
                  |   RDS PostgreSQL  |
                  |                   |
                  | Public Access: NO |
                  | Port: 5432        |
                  | Encrypted         |
                  +---------+---------+
                            |
                       DB Security Group
                            |
                    TCP 5432 from
                    Application SG
```

AWS recommends using private subnets for private RDS databases, and an RDS DB subnet group uses subnets across at least two Availability Zones. ([AWS Documentation][1])

## Phase 4 objectives

We'll learn:

1. Create a VPC specifically for the exercise
2. Create **2 private database subnets**
3. Create an RDS DB subnet group
4. Create an **Application Security Group**
5. Create a **Database Security Group**
6. Allow PostgreSQL `5432` only from the Application SG
7. Deploy PostgreSQL RDS privately
8. Enable storage encryption
9. Enable automated backups
10. Use **AWS Secrets Manager for the master password**
11. Retrieve the secret ARN through Terraform
12. Understand RDS endpoint/DNS
13. Verify the database is not publicly accessible
14. Destroy everything afterward to control cost

For this exercise, we will use RDS's `manage_master_user_password = true`. RDS generates and manages the master password in Secrets Manager, avoiding a hard-coded password in your Terraform configuration. ([AWS Documentation][2])

---

# 1. Create the Phase 4 project

In PowerShell:

```powershell
mkdir terraform-aws-phase4-rds
cd terraform-aws-phase4-rds
```

Create:

```text
terraform-aws-phase4-rds/
│
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── data.tf
├── locals.tf
│
├── vpc.tf
├── subnets.tf
├── routes.tf
│
├── security-groups.tf
├── db-subnet-group.tf
├── rds.tf
│
└── outputs.tf
```

---

# 2. `provider.tf`

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

---

# 3. `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "terraform-phase4"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "VPC CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "db_subnet_a_cidr" {
  description = "Database subnet A CIDR"
  type        = string
  default     = "10.0.11.0/24"
}

variable "db_subnet_b_cidr" {
  description = "Database subnet B CIDR"
  type        = string
  default     = "10.0.12.0/24"
}

variable "db_instance_class" {
  description = "RDS instance class"
  type        = string
  default     = "db.t3.micro"
}

variable "db_name" {
  description = "Database name"
  type        = string
  default     = "itcommunity"
}

variable "db_username" {
  description = "Database master username"
  type        = string
  default     = "appadmin"
}

variable "db_allocated_storage" {
  description = "Allocated database storage in GB"
  type        = number
  default     = 20
}
```

---

# 4. `terraform.tfvars`

Keep this simple:

```hcl
aws_region   = "ap-south-1"
project_name = "terraform-phase4"
environment  = "dev"

vpc_cidr = "10.0.0.0/16"

db_subnet_a_cidr = "10.0.11.0/24"
db_subnet_b_cidr = "10.0.12.0/24"

db_instance_class    = "db.t3.micro"
db_name              = "itcommunity"
db_username          = "appadmin"
db_allocated_storage = 20
```

Notice what is **not** here:

```text
password = "MyPassword123"
```

We intentionally don't put the database password in this file.

---

# 5. `data.tf`

We need Availability Zones.

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

---

# 6. `locals.tf`

```hcl
locals {
  name_prefix = "${var.project_name}-${var.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
```

---

# 7. Create VPC

## `vpc.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-vpc"
    }
  )
}
```

We need DNS support because the RDS endpoint is a DNS name.

---

# 8. Create private database subnets

## `subnets.tf`

```hcl
resource "aws_subnet" "db_a" {
  vpc_id = aws_vpc.main.id

  cidr_block = var.db_subnet_a_cidr

  availability_zone = data.aws_availability_zones.available.names[0]

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-db-subnet-a"
      Tier = "database"
    }
  )
}

resource "aws_subnet" "db_b" {
  vpc_id = aws_vpc.main.id

  cidr_block = var.db_subnet_b_cidr

  availability_zone = data.aws_availability_zones.available.names[1]

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-db-subnet-b"
      Tier = "database"
    }
  )
}
```

Important concept:

```text
db_a → AZ 1
db_b → AZ 2
```

The DB subnet group will contain both.

AWS specifically documents that RDS DB subnet groups should have subnets in different Availability Zones. ([AWS Documentation][3])

---

# 9. Private route tables

We don't need a NAT Gateway for the database.

This is also an important **cost-saving decision**.

## `routes.tf`

```hcl
resource "aws_route_table" "database" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-db-route-table"
    }
  )
}

resource "aws_route_table_association" "db_a" {
  subnet_id = aws_subnet.db_a.id

  route_table_id = aws_route_table.database.id
}

resource "aws_route_table_association" "db_b" {
  subnet_id = aws_subnet.db_b.id

  route_table_id = aws_route_table.database.id
}
```

There is intentionally **no**:

```hcl
0.0.0.0/0 → Internet Gateway
```

and no:

```hcl
0.0.0.0/0 → NAT Gateway
```

Therefore these are private database subnets.

---

# 10. Application Security Group

Even though our Phase 3 EC2 instances are currently destroyed, we'll create the SG now.

This teaches an important AWS security pattern:

```text
Application SG
       |
       | TCP 5432
       v
Database SG
```

## `security-groups.tf`

```hcl
resource "aws_security_group" "application" {
  name        = "${local.name_prefix}-application-sg"
  description = "Security group representing application servers"
  vpc_id      = aws_vpc.main.id

  egress {
    description = "Allow outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-application-sg"
    }
  )
}
```

---

# 11. Database Security Group

Add this to the same file.

```hcl
resource "aws_security_group" "database" {
  name        = "${local.name_prefix}-database-sg"
  description = "Security group for PostgreSQL RDS"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "PostgreSQL from application servers"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
  }

  egress {
    description = "Allow outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-database-sg"
    }
  )
}
```

This is a **very important Terraform/AWS concept**.

We are NOT doing this:

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

for PostgreSQL.

Instead:

```hcl
security_groups = [aws_security_group.application.id]
```

means:

> Only resources belonging to the Application Security Group can initiate connections to PostgreSQL on port 5432.

AWS documents security groups as the mechanism controlling traffic to RDS instances. ([AWS Documentation][4])

---

# 12. Create DB subnet group

Create:

## `db-subnet-group.tf`

```hcl
resource "aws_db_subnet_group" "database" {
  name = "${local.name_prefix}-db-subnet-group"

  subnet_ids = [
    aws_subnet.db_a.id,
    aws_subnet.db_b.id
  ]

  description = "Private subnet group for PostgreSQL RDS"

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-db-subnet-group"
    }
  )
}
```

Terraform's `aws_db_subnet_group` resource takes the subnet IDs that RDS can use for the DB instance. ([Terraform Registry][5])

---

# 13. Create RDS PostgreSQL

Now the main resource.

## `rds.tf`

```hcl
resource "aws_db_instance" "postgresql" {
  identifier = "${local.name_prefix}-postgresql"

  engine = "postgres"

  instance_class = var.db_instance_class

  allocated_storage = var.db_allocated_storage

  storage_type = "gp3"

  storage_encrypted = true

  db_name  = var.db_name
  username = var.db_username

  port = 5432

  db_subnet_group_name = aws_db_subnet_group.database.name

  vpc_security_group_ids = [
    aws_security_group.database.id
  ]

  publicly_accessible = false

  multi_az = false

  backup_retention_period = 1

  backup_window = "18:00-19:00"

  maintenance_window = "sun:19:00-sun:20:00"

  auto_minor_version_upgrade = true

  deletion_protection = false

  skip_final_snapshot = true

  manage_master_user_password = true

  copy_tags_to_snapshot = true

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-postgresql"
    }
  )
}
```

### Important settings

#### Private database

```hcl
publicly_accessible = false
```

This means the RDS instance doesn't receive a public IP and is intended to be accessed from within the VPC/private network. ([AWS Documentation][1])

#### PostgreSQL

```hcl
engine = "postgres"
```

#### PostgreSQL port

```hcl
port = 5432
```

#### Encryption

```hcl
storage_encrypted = true
```

#### No Multi-AZ for our learning environment

```hcl
multi_az = false
```

We're doing this specifically to reduce the practice cost.

Later we'll build:

```text
Production RDS
     |
     +--- Primary AZ
     |
     +--- Standby AZ
```

#### Secrets Manager

Most importantly:

```hcl
manage_master_user_password = true
```

RDS generates and manages the master password in Secrets Manager rather than requiring us to put the password directly in Terraform configuration. ([AWS Documentation][2])

---

# 14. Outputs

## `outputs.tf`

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "db_subnet_group_name" {
  description = "RDS DB subnet group"
  value       = aws_db_subnet_group.database.name
}

output "database_security_group_id" {
  description = "Database security group ID"
  value       = aws_security_group.database.id
}

output "application_security_group_id" {
  description = "Application security group ID"
  value       = aws_security_group.application.id
}

output "db_instance_identifier" {
  description = "RDS instance identifier"
  value       = aws_db_instance.postgresql.identifier
}

output "db_endpoint" {
  description = "RDS PostgreSQL endpoint"
  value       = aws_db_instance.postgresql.endpoint
}

output "db_address" {
  description = "RDS PostgreSQL hostname"
  value       = aws_db_instance.postgresql.address
}

output "db_port" {
  description = "PostgreSQL port"
  value       = aws_db_instance.postgresql.port
}

output "master_user_secret_arn" {
  description = "Secrets Manager ARN containing the RDS master credentials"
  value       = aws_db_instance.postgresql.master_user_secret[0].secret_arn
}
```

Notice that we're **not outputting the password**.

---

# 15. Format and validate

Run:

```powershell
terraform fmt -recursive
```

Then:

```powershell
terraform validate
```

Expected:

```text
Success! The configuration is valid.
```

Then:

```powershell
terraform init
```

Then:

```powershell
terraform plan
```

---

# 16. What Terraform should create

The plan should contain approximately:

```text
VPC
 └── VPC

Subnets
 ├── DB Subnet A
 └── DB Subnet B

Routing
 ├── DB Route Table
 ├── Association A
 └── Association B

Security
 ├── Application Security Group
 └── Database Security Group

RDS
 ├── DB Subnet Group
 └── PostgreSQL DB Instance

Secrets
 └── RDS-managed Secrets Manager secret
```

Before applying, **check the plan carefully**.

You should NOT see:

```text
aws_nat_gateway
aws_instance
aws_lb
aws_autoscaling_group
```

because we don't need them for this phase.

---

# 17. Apply

Once the plan looks correct:

```powershell
terraform apply
```

Terraform will ask:

```text
Do you want to perform these actions?
  Enter a value: yes
```

Enter:

```text
yes
```

RDS will take considerably longer to create than an S3 bucket or security group.

---

# 18. Check outputs

After completion:

```powershell
terraform output
```

You should see something similar to:

```text
application_security_group_id = "sg-..."
database_security_group_id    = "sg-..."
db_address                    = "terraform-phase4-dev-postgresql.xxxxx.ap-south-1.rds.amazonaws.com"
db_endpoint                   = "terraform-phase4-dev-postgresql.xxxxx.ap-south-1.rds.amazonaws.com:5432"
db_port                      = 5432
db_subnet_group_name         = "terraform-phase4-dev-db-subnet-group"
db_instance_identifier      = "terraform-phase4-dev-postgresql"
master_user_secret_arn       = "arn:aws:secretsmanager:..."
vpc_id                       = "vpc-..."
```

---

# 19. Check RDS status

Run:

```powershell
aws rds describe-db-instances `
  --db-instance-identifier terraform-phase4-dev-postgresql `
  --query "DBInstances[0].DBInstanceStatus"
```

Eventually:

```text
"available"
```

---

# 20. Verify it is private

Run:

```powershell
aws rds describe-db-instances `
  --db-instance-identifier terraform-phase4-dev-postgresql `
  --query "DBInstances[0].PubliclyAccessible"
```

Expected:

```text
false
```

This is one of the most important verification steps.

AWS explicitly documents `PubliclyAccessible = false` as the configuration for a DB instance accessible only through the VPC/private network. ([AWS Documentation][6])

---

# 21. Verify Secrets Manager

Get the ARN:

```powershell
terraform output master_user_secret_arn
```

Then:

```powershell
aws secretsmanager describe-secret `
  --secret-id YOUR_SECRET_ARN
```

You should see the secret metadata.

Don't paste the secret value into chat.

The architecture is:

```text
Terraform
    |
    v
RDS
    |
    | generates/manages password
    v
AWS Secrets Manager
    |
    | stores credentials
    v
Application
```

This is much better than:

```text
terraform.tfvars
       |
       v
password = "..."
```

---

# 22. Verify the security group

Run:

```powershell
aws ec2 describe-security-groups `
  --group-ids YOUR_DATABASE_SECURITY_GROUP_ID
```

You should find:

```text
Protocol: TCP
Port: 5432
Source: Application Security Group
```

You should **not** find:

```text
0.0.0.0/0
```

for PostgreSQL.

---

# 23. Important: Why can't your laptop connect?

At this stage:

```text
Your Laptop
     |
     X
     |
Internet
     |
     X
     |
Private RDS
```

That's intentional.

The database is:

```text
PRIVATE
```

AWS explains that a private RDS instance isn't reachable directly from the internet; access must come through the VPC or an appropriate private connectivity mechanism. ([AWS Documentation][7])

Later, when we bring our Phase 3 application back:

```text
                         Internet
                            |
                            v
                          ALB
                            |
                            v
                     Application EC2
                            |
                         TCP 5432
                            |
                            v
                    +---------------+
                    | RDS PostgreSQL|
                    | Private       |
                    +---------------+
```

The security relationship becomes:

```text
ALB SG
  |
  v
Application SG
  |
  | TCP 5432
  v
Database SG
  |
  v
RDS
```

That is the architecture you should be able to explain in an AWS/DevOps interview.

---

# 24. Cost-conscious settings we're using

Because you're deliberately practicing and destroying resources afterward, I've configured this as a learning environment:

| Setting             | Phase 4            |
| ------------------- | ------------------ |
| RDS                 | PostgreSQL         |
| Instance            | `db.t3.micro`      |
| Storage             | 20 GB              |
| Storage type        | gp3                |
| Encryption          | Enabled            |
| Public access       | **No**             |
| Multi-AZ            | **No**             |
| Backup retention    | 1 day              |
| NAT Gateway         | **None**           |
| EC2                 | None               |
| ALB                 | None               |
| Secrets Manager     | RDS-managed secret |
| Deletion protection | No                 |
| Final snapshot      | Skipped            |

This avoids recreating the NAT Gateway from Phase 2 just for the database.

**RDS and Secrets Manager can still incur charges**, so after completing the exercises, destroy the stack.

---

# 25. Destroy after practicing

When you're finished:

```powershell
terraform destroy
```

Then:

```text
yes
```

Afterwards verify:

```powershell
aws rds describe-db-instances
```

and:

```powershell
aws ec2 describe-vpcs
```

to make sure the practice resources are gone.

---

# Phase 4 challenge

Don't just apply the configuration yet. First try to answer these yourself:

### Challenge 1

Why is this:

```hcl
security_groups = [aws_security_group.application.id]
```

better than:

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

for PostgreSQL?

### Challenge 2

Why do we need:

```text
DB Subnet A → AZ-A
DB Subnet B → AZ-B
```

instead of putting both subnets in the same AZ?

### Challenge 3

What is the difference between:

```hcl
publicly_accessible = false
```

and:

```hcl
storage_encrypted = true
```

### Challenge 4

Where is the database password?

```text
terraform.tfvars       ❌
variables.tf            ❌
rds.tf password         ❌
AWS Secrets Manager     ?
```

### Challenge 5

What happens when we eventually connect Phase 3 EC2 to Phase 4 RDS?

Think about this flow:

```text
EC2
 |
 | TCP 5432
 v
RDS Security Group
 |
 v
PostgreSQL
```

---

## Your Phase 4 learning progression

I recommend doing it in this order:

```text
STEP 1  → Create project files
STEP 2  → terraform init
STEP 3  → terraform fmt
STEP 4  → terraform validate
STEP 5  → terraform plan
STEP 6  → Inspect plan
STEP 7  → terraform apply
STEP 8  → Verify RDS
STEP 9  → Verify private access
STEP 10 → Verify Security Groups
STEP 11 → Verify Secrets Manager
STEP 12 → Answer challenges
STEP 13 → terraform destroy
```

Once you have run **`terraform plan`**, paste the plan output here. I can review it with you **before you spend money on the RDS creation**.

[1]: https://docs.aws.amazon.com/AmazonRDS/latest/gettingstartedguide/security-public-private.html?utm_source=chatgpt.com "Setting up public or private access in Amazon RDS - Amazon Relational Database Service"
[2]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html?utm_source=chatgpt.com "Password management with Amazon RDS and AWS Secrets Manager - Amazon Relational Database Service"
[3]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_VPC.WorkingWithRDSInstanceinaVPC.html?utm_source=chatgpt.com "Working with a DB instance in a VPC - Amazon Relational Database Service"
[4]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/infrastructure-security.html?utm_source=chatgpt.com "Infrastructure security in Amazon RDS - Amazon Relational Database Service"
[5]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_subnet_group.html?utm_source=chatgpt.com "aws_db_subnet_group | Resources | hashicorp/aws | Terraform | Terraform Registry"
[6]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_CreateDBInstance.Settings.html?utm_source=chatgpt.com "Settings for DB instances - Amazon Relational Database Service"
[7]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToPostgreSQLInstance.html?utm_source=chatgpt.com "Connecting to a DB instance running the PostgreSQL database engine - Amazon Relational Database Service"
