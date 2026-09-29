# Phase 2 — Build a Production-Style AWS VPC

We are now moving from **Terraform fundamentals → AWS infrastructure design**.

The target architecture for this phase is:

```text
                         INTERNET
                             |
                             |
                    Internet Gateway
                             |
                    +--------+--------+
                    |       VPC       |
                    |  10.0.0.0/16    |
                    |                 |
                    |  Public Route   |
                    |      Table      |
                    |                 |
              +-----+------+    +-----+------+
              | Public      |    | Public      |
              | Subnet AZ-A |    | Subnet AZ-B |
              |10.0.1.0/24  |    |10.0.2.0/24  |
              +------+------+    +------+------+
                     |                   |
                     |             NAT Gateway
                     |                   |
                     |                   |
              +------+-------+    +------+-------+
              | Private      |    | Private      |
              | Subnet AZ-A  |    | Subnet AZ-B  |
              |10.0.11.0/24  |    |10.0.12.0/24  |
              +--------------+    +--------------+
```

This is a very common foundation for AWS applications.

---

# 1. What you will learn

By the end of Phase 2, you'll understand:

### Terraform

* Data sources
* Variables
* Locals
* Resource dependencies
* `for_each`
* Maps
* Lists
* Terraform expressions
* Outputs
* Module preparation

### AWS

* VPC
* CIDR
* Availability Zones
* Public subnet
* Private subnet
* Internet Gateway
* NAT Gateway
* Elastic IP
* Route Tables
* Route Table Associations
* Security Groups
* VPC DNS
* Network architecture

---

# 2. Phase 2 Architecture

We'll create:

```text
VPC
10.0.0.0/16
│
├── Availability Zone A
│   │
│   ├── Public Subnet
│   │   10.0.1.0/24
│   │
│   └── Private Subnet
│       10.0.11.0/24
│
└── Availability Zone B
    │
    ├── Public Subnet
    │   10.0.2.0/24
    │
    └── Private Subnet
        10.0.12.0/24
```

And:

```text
Internet
    |
    v
Internet Gateway
    |
    v
Public Route Table
    |
    +---- Public Subnet A
    |
    +---- Public Subnet B


Private Subnet A
       |
       v
   NAT Gateway
       |
       v
Internet Gateway
       |
       v
   Internet
```

---

# 3. Important Concept — Public vs Private Subnet

This is something you should understand before writing Terraform.

## Public subnet

A subnet becomes public when its route table contains:

```text
0.0.0.0/0
     |
     v
Internet Gateway
```

For example:

```text
Public Subnet
     |
     v
Route Table
     |
     +-- 10.0.0.0/16 → local
     |
     +-- 0.0.0.0/0 → Internet Gateway
```

Resources such as:

* Load Balancers
* Bastion hosts
* NAT Gateways

can be placed in public subnets depending on the architecture.

---

## Private subnet

Private subnet:

```text
Private Subnet
     |
     v
Route Table
     |
     +-- 10.0.0.0/16 → local
     |
     +-- 0.0.0.0/0 → NAT Gateway
```

Resources such as:

* Application servers
* Databases
* Internal services

can reside in private subnets.

The private resources can initiate outbound internet connections through NAT, but they aren't directly reachable from the internet through an Internet Gateway.

---

# 4. Create a New Phase 2 Project

Don't modify Phase 1.

Create:

```powershell
mkdir terraform-aws-phase2-vpc
cd terraform-aws-phase2-vpc
```

Create:

```text
terraform-aws-phase2-vpc/
│
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── locals.tf
├── data.tf
├── vpc.tf
├── subnets.tf
├── internet-gateway.tf
├── nat-gateway.tf
├── routes.tf
├── security-groups.tf
└── outputs.tf
```

This is deliberately more structured than Phase 1.

---

# 5. `provider.tf`

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

# 6. `variables.tf`

Create:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-south-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "terraform-phase2"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)

  default = [
    "10.0.1.0/24",
    "10.0.2.0/24"
  ]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)

  default = [
    "10.0.11.0/24",
    "10.0.12.0/24"
  ]
}
```

---

# 7. `terraform.tfvars`

```hcl
aws_region  = "ap-south-1"
project_name = "terraform-phase2"
environment  = "dev"

vpc_cidr = "10.0.0.0/16"

public_subnet_cidrs = [
  "10.0.1.0/24",
  "10.0.2.0/24"
]

private_subnet_cidrs = [
  "10.0.11.0/24",
  "10.0.12.0/24"
]
```

---

# 8. Introduce `data` Sources

Create `data.tf`.

This is our first major new Terraform concept.

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}
```

This asks AWS:

> "Tell Terraform which Availability Zones are currently available in this region."

We don't want to hardcode:

```text
ap-south-1a
ap-south-1b
```

Instead Terraform can discover them.

---

# 9. Create `locals.tf`

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  availability_zones = slice(
    data.aws_availability_zones.available.names,
    0,
    2
  )
}
```

This gives us two Availability Zones.

Conceptually:

```text
data.aws_availability_zones
            |
            v
   Available AZs in region
            |
            v
          slice()
            |
            v
       First 2 AZs
```

---

# 10. Create the VPC

Create `vpc.tf`:

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-vpc"
    }
  )
}
```

Important settings:

```hcl
enable_dns_support = true
```

and:

```hcl
enable_dns_hostnames = true
```

These will be important later when we deploy EC2, ECS, RDS, etc.

---

# 11. Create Internet Gateway

Add to `internet-gateway.tf`:

```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-igw"
    }
  )
}
```

Architecture:

```text
Internet
   |
   v
Internet Gateway
   |
   v
VPC
```

---

# 12. Create Public Subnets

Create `subnets.tf`:

```hcl
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id = aws_vpc.main.id

  cidr_block = var.public_subnet_cidrs[count.index]

  availability_zone = local.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-public-${count.index + 1}"
      Tier = "Public"
    }
  )
}
```

This is our first practical use of:

```hcl
count
```

Terraform creates:

```text
aws_subnet.public[0]
aws_subnet.public[1]
```

Instead of writing:

```hcl
resource "aws_subnet" "public_a" {}
resource "aws_subnet" "public_b" {}
```

---

# 13. Create Private Subnets

Add to `subnets.tf`:

```hcl
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id = aws_vpc.main.id

  cidr_block = var.private_subnet_cidrs[count.index]

  availability_zone = local.availability_zones[count.index]

  map_public_ip_on_launch = false

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-private-${count.index + 1}"
      Tier = "Private"
    }
  )
}
```

Now:

```text
VPC
│
├── Public 1
├── Public 2
├── Private 1
└── Private 2
```

---

# 14. Create Public Route Table

Create `routes.tf`.

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-public-rt"
    }
  )
}
```

Now create the internet route:

```hcl
resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"

  gateway_id = aws_internet_gateway.main.id
}
```

This creates:

```text
Public Route Table

10.0.0.0/16 → local
0.0.0.0/0   → Internet Gateway
```

---

# 15. Associate Public Subnets

Add:

```hcl
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id = aws_subnet.public[count.index].id

  route_table_id = aws_route_table.public.id
}
```

Now:

```text
Public Subnet A
      |
      +---- Public Route Table
                   |
                   v
             Internet Gateway


Public Subnet B
      |
      +---- Public Route Table
                   |
                   v
             Internet Gateway
```

---

# 16. Create NAT Gateway

This is where AWS architecture becomes more interesting.

A NAT Gateway requires an Elastic IP.

Create `nat-gateway.tf`:

```hcl
resource "aws_eip" "nat" {
  domain = "vpc"

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-nat-eip"
    }
  )
}
```

Then:

```hcl
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id

  subnet_id = aws_subnet.public[0].id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-nat"
    }
  )

  depends_on = [
    aws_internet_gateway.main
  ]
}
```

Our architecture is now:

```text
Private Subnet
      |
      v
 NAT Gateway
      |
      v
Public Subnet
      |
      v
Internet Gateway
      |
      v
 Internet
```

---

# 17. Private Route Table

Add to `routes.tf`:

```hcl
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-private-rt"
    }
  )
}
```

Then:

```hcl
resource "aws_route" "private_nat" {
  route_table_id         = aws_route_table.private.id
  destination_cidr_block = "0.0.0.0/0"

  nat_gateway_id = aws_nat_gateway.main.id
}
```

And associate:

```hcl
resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id = aws_subnet.private[count.index].id

  route_table_id = aws_route_table.private.id
}
```

---

# 18. Security Group

Create `security-groups.tf`.

For now, create a security group for future application servers:

```hcl
resource "aws_security_group" "application" {
  name        = "${var.project_name}-application-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
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
      Name = "${var.project_name}-application-sg"
    }
  )
}
```

We'll improve this security model significantly when we get to EC2/ALB.

---

# 19. Outputs

Create `outputs.tf`:

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR"
  value       = aws_vpc.main.cidr_block
}

output "availability_zones" {
  description = "Availability Zones used"
  value       = local.availability_zones
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.main.id
}

output "nat_gateway_id" {
  description = "NAT Gateway ID"
  value       = aws_nat_gateway.main.id
}

output "application_security_group_id" {
  description = "Application security group ID"
  value       = aws_security_group.application.id
}
```

Notice:

```hcl
aws_subnet.public[*].id
```

This is another Terraform concept:

**splat expression**.

---

# 20. Initialize

Run:

```powershell
terraform init
```

Then:

```powershell
terraform fmt -recursive
```

Then:

```powershell
terraform validate
```

You should get:

```text
Success! The configuration is valid.
```

---

# 21. VERY IMPORTANT — Run Plan

Now:

```powershell
terraform plan
```

You should see a relatively large plan.

Don't immediately apply it.

First understand what Terraform wants to create.

You should have roughly:

```text
1 VPC
1 Internet Gateway
4 Subnets
1 Public Route Table
1 Private Route Table
1 Public Route
1 Private Route
4 Route Table Associations
1 Elastic IP
1 NAT Gateway
1 Security Group
```

So we're now managing around **15+ AWS resources** instead of one S3 bucket.

---

# 22. Apply

Once you understand the plan:

```powershell
terraform apply
```

Enter:

```text
yes
```

⚠️ **Important:** NAT Gateway and Elastic IP usage can incur AWS charges. Don't leave this infrastructure running unnecessarily while practicing.

For this learning project, we'll destroy it after verification unless we're immediately continuing to Phase 3.

---

# 23. Verify the VPC

Run:

```powershell
terraform output
```

You should see something like:

```text
availability_zones = [
  "ap-south-1a",
  "ap-south-1b"
]

private_subnet_ids = [
  "...",
  "..."
]

public_subnet_ids = [
  "...",
  "..."
]

vpc_cidr = "10.0.0.0/16"
```

---

# 24. Verify using AWS CLI

Get the VPC:

```powershell
aws ec2 describe-vpcs `
  --filters "Name=tag:Project,Values=terraform-phase2"
```

Check subnets:

```powershell
aws ec2 describe-subnets `
  --filters "Name=tag:Project,Values=terraform-phase2"
```

Check route tables:

```powershell
aws ec2 describe-route-tables `
  --filters "Name=tag:Project,Values=terraform-phase2"
```

Check NAT:

```powershell
aws ec2 describe-nat-gateways `
  --filter "Name=tag:Project,Values=terraform-phase2"
```

---

# 25. What You Should See in AWS

Your VPC console should look conceptually like:

```text
VPC
│
├── Internet Gateway
│
├── Route Tables
│   ├── Public
│   └── Private
│
├── Subnets
│   ├── Public-A
│   ├── Public-B
│   ├── Private-A
│   └── Private-B
│
├── NAT Gateway
│
└── Security Group
```

---

# 26. The Most Important Terraform Concepts From Phase 2

You've now gone beyond basic resources.

### Data source

```hcl
data "aws_availability_zones" "available" {}
```

Means:

> Get information from AWS instead of creating something.

---

### Resource

```hcl
resource "aws_vpc" "main" {}
```

Means:

> Terraform should create/manage this infrastructure.

---

### Variable

```hcl
var.vpc_cidr
```

Means:

> Configuration supplied from outside the resource.

---

### Local

```hcl
local.common_tags
```

Means:

> Reusable calculated/configuration values inside Terraform.

---

### Resource reference

```hcl
aws_vpc.main.id
```

Means:

> Get the ID produced by another resource.

---

### Count

```hcl
count = length(var.public_subnet_cidrs)
```

Means:

> Create multiple instances of a resource.

---

### Splat

```hcl
aws_subnet.public[*].id
```

Means:

> Return the IDs of all instances.

---

### Explicit dependency

```hcl
depends_on = [
  aws_internet_gateway.main
]
```

Means:

> Explicitly tell Terraform that this resource must wait for another resource.

---

# 27. Your Phase 2 Challenge

Don't just copy the architecture.

I want you to modify it.

### Challenge 1

Change:

```text
10.0.0.0/16
```

to:

```text
10.10.0.0/16
```

Then calculate appropriate subnet CIDRs.

For example:

```text
Public A
Public B
Private A
Private B
```

---

### Challenge 2

Instead of using:

```hcl
count
```

for the subnets, try using:

```hcl
for_each
```

This is extremely important because you'll use `for_each` heavily in real Terraform projects.

---

### Challenge 3

Create another security group:

```text
database_security_group
```

Don't allow the database security group to accept traffic from:

```text
0.0.0.0/0
```

Instead, later we'll make:

```text
Application SG
       |
       | TCP 5432
       v
Database SG
```

This will prepare you for the RDS phase.

---

# 28. Cost Warning

Before leaving Phase 2 running, remember that:

```text
NAT Gateway
```

is not something you want running unnecessarily in a practice account.

When finished practicing:

```powershell
terraform destroy
```

Then confirm:

```powershell
aws ec2 describe-vpcs `
  --filters "Name=tag:Project,Values=terraform-phase2"
```

Ideally, the resources should be gone.

---

# Your Phase 2 completion checklist

```text
[ ] Create Phase 2 directory
[ ] provider.tf
[ ] variables.tf
[ ] terraform.tfvars
[ ] data.tf
[ ] locals.tf
[ ] VPC
[ ] Internet Gateway
[ ] 2 Public Subnets
[ ] 2 Private Subnets
[ ] Public Route Table
[ ] Private Route Table
[ ] NAT Gateway
[ ] Elastic IP
[ ] Route Associations
[ ] Security Group
[ ] Outputs

[ ] terraform init
[ ] terraform fmt
[ ] terraform validate
[ ] terraform plan
[ ] terraform apply

[ ] Verify VPC in AWS
[ ] Verify subnets
[ ] Verify route tables
[ ] Verify NAT Gateway

[ ] Complete Challenge 1
[ ] Complete Challenge 2
[ ] Complete Challenge 3

[ ] terraform destroy
```

### One important architectural note

For a **production** environment, we'd normally consider **one NAT Gateway per Availability Zone** for higher resilience, rather than the single-NAT design used here. We're intentionally starting with one to keep the Phase 2 learning objective manageable and reduce practice cost. We'll revisit the production version when we build the high-availability architecture.

**Next phase:** **Phase 3 — EC2 + Application Load Balancer + Auto Scaling**, where we'll actually put compute into the VPC and make the public/private subnet design useful.
