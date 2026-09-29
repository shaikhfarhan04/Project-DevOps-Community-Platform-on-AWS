# Phase 3 — EC2 + Application Load Balancer + Auto Scaling

Now we're moving from **networking infrastructure** to a real **high-availability application tier**.

Our Phase 2 VPC becomes the foundation, and Phase 3 adds:

```text
                         INTERNET
                             |
                             v
                    +----------------+
                    |      ALB       |
                    | Public Subnets |
                    +-------+--------+
                            |
                 +----------+----------+
                 |                     |
                 v                     v
          +-------------+       +-------------+
          | EC2 Instance|       | EC2 Instance|
          |   AZ-A      |       |   AZ-B      |
          | Private     |       | Private     |
          +------+------+       +------+------+
                 |                     |
                 +----------+----------+
                            |
                       Auto Scaling
                            |
                    +-------+-------+
                    |               |
                 Scale Out       Scale In
```

AWS recommends distributing an Auto Scaling group across multiple Availability Zones, and an Application Load Balancer must use at least two AZs. The ALB can route traffic to healthy targets while Auto Scaling can replace unhealthy instances. ([AWS Documentation][1])

![Image](https://images.openai.com/static-rsc-4/s_xCKQ0UYweeymM9Z6SKEBrdtNIdmj0Zv43_rMjlzUW_T5NeX7rmJu4ph-F9QFbpBNaaIT4MIcIx_-53BYKN9_Uu1X8_pJZ_c7msdoPGWV4FryEm5P-_djzaSCnlZmLmG2fZNkrZekJtL_7Rdgq-poL-z40T6PXo8eChIxU6wNjkVLnebYXpM1WkjpHEkv3d?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/8MWGKDjDu9SWBuEy8GoEUGRxyFRtwPUX2nCinuPNWCwUKQyQZfo6LL63zdw6X71I2_f3VpoKTQxcMIcUnxp1BxkqxQhaSzHF3NfUurWhXvBj3wTL6x9l0qSOhS3OwhoSqBKA9LQW5JyO6QR5Nt_hG9H3R52wj-LbPWwmvcIlsM5kS1MQQXBsod3jQ_I-RG19?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/IDJ5_bOTvLS1irCYRoC-_6YEjr_UZbQDFUCoYkXliS8nnquY9cyk4Q9cv2ZisfCgCokhDqi4q2bfVyp0VSpCmGHiVOgbOc7py1_IVyDXS5mVaQpaZhyG538NOSL-iiaYzj1_hPAFLDRpNeRrFNpdoA6hb7YkyLMDDFARd_2QLBfcHqWRSleD3GFZa_uTyBHH?purpose=fullsize)

![Image](https://images.openai.com/static-rsc-4/fBDcbNLPoEMIDdUr3NBmdqLqfmnxH523uBDz8AzcKQtogR1jmpKF2kgn0ogzlRzg2_i3smaj7saLEmRHrOu1PIGMdqM0vWxZlwTGug7dBPEa8YTMY4ZvmtprJ-qqyRZ6VAxlepdpP-hfc_umjTgTT4rtsk0URTl5_OgmlGKKK--9ykaTAR_-2cm52lQ9fHiD?purpose=fullsize)

---

# 1. What we'll learn

This phase introduces:

### Terraform

* `aws_launch_template`
* `aws_autoscaling_group`
* `aws_lb`
* `aws_lb_target_group`
* `aws_lb_listener`
* `aws_security_group`
* IAM roles
* Instance profiles
* Data sources
* SSM public parameters
* `user_data`
* `templatefile` concepts
* Auto Scaling policies
* Health checks
* Resource dependencies

### AWS

* EC2
* Launch Templates
* Application Load Balancer
* Target Groups
* Auto Scaling Groups
* IAM
* Security Groups
* CloudWatch-based scaling
* Private EC2 instances
* High availability across AZs

---

# 2. Our final architecture

We'll use the VPC you created in Phase 2.

```text
                              Internet
                                 |
                                 v
                         Internet Gateway
                                 |
                                 v
                    +-----------------------+
                    | Application Load      |
                    | Balancer              |
                    |                       |
                    | Public Subnet AZ-A    |
                    | Public Subnet AZ-B    |
                    +-----------+-----------+
                                |
                         Target Group
                                |
                 +--------------+--------------+
                 |                             |
                 v                             v
        +----------------+            +----------------+
        | EC2            |            | EC2            |
        | AZ-A           |            | AZ-B           |
        | Private Subnet |            | Private Subnet |
        +-------+--------+            +-------+--------+
                |                             |
                +-------------+---------------+
                              |
                         Auto Scaling
                              |
                    +---------+---------+
                    |                   |
                 Scale Out           Scale In
```

And the EC2 instances reach the internet for package installation through:

```text
EC2 Private Subnet
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

This is much better than putting application EC2 instances directly into public subnets.

---

# 3. Important security design

We are going to create **two different security groups**.

```text
                 Internet
                    |
                    v
             +-------------+
             | ALB SG      |
             |             |
             | HTTP :80    |
             | from 0.0.0.0/0
             +------+------+
                    |
                    | TCP 80
                    v
             +-------------+
             | EC2 SG      |
             |             |
             | HTTP :80    |
             | ONLY from  |
             | ALB SG      |
             +-------------+
```

So:

```text
Internet → ALB → EC2
```

not:

```text
Internet → EC2
```

This is an important production networking pattern.

AWS's load-balancing guidance similarly describes allowing the instance security group to receive the listener/health-check traffic from the load balancer. ([AWS Documentation][2])

---

# 4. Create Phase 3 project

Don't modify your Phase 2 project.

Create:

```powershell
mkdir terraform-aws-phase3-compute
cd terraform-aws-phase3-compute
```

We'll reference the Phase 2 VPC resources through **remote state later**. For this first exercise, however, we'll keep things simple and use the Phase 2 resource IDs as variables.

Create:

```text
terraform-aws-phase3-compute/
│
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── data.tf
├── locals.tf
│
├── iam.tf
├── security-groups.tf
├── launch-template.tf
├── target-group.tf
├── load-balancer.tf
├── autoscaling.tf
│
└── outputs.tf
```

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
  default     = "terraform-phase3"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "dev"
}

variable "vpc_id" {
  description = "VPC ID from Phase 2"
  type        = string
}

variable "public_subnet_ids" {
  description = "Public subnet IDs from Phase 2"
  type        = list(string)
}

variable "private_subnet_ids" {
  description = "Private subnet IDs from Phase 2"
  type        = list(string)
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

variable "desired_capacity" {
  description = "Desired number of EC2 instances"
  type        = number
  default     = 2
}

variable "min_size" {
  description = "Minimum ASG capacity"
  type        = number
  default     = 2
}

variable "max_size" {
  description = "Maximum ASG capacity"
  type        = number
  default     = 4
}
```

---

# 7. Get Phase 2 IDs

From your Phase 2 directory:

```powershell
terraform output
```

You'll need:

```text
vpc_id
public_subnet_ids
private_subnet_ids
```

For example:

```text
vpc_id = "vpc-xxxxxxxx"

public_subnet_ids = [
  "subnet-aaaa",
  "subnet-bbbb"
]

private_subnet_ids = [
  "subnet-cccc",
  "subnet-dddd"
]
```

Don't copy my example IDs.

Use your actual Phase 2 outputs.

---

# 8. `terraform.tfvars`

Put your actual values here:

```hcl
aws_region  = "ap-south-1"
project_name = "terraform-phase3"
environment  = "dev"

vpc_id = "YOUR_VPC_ID"

public_subnet_ids = [
  "YOUR_PUBLIC_SUBNET_A",
  "YOUR_PUBLIC_SUBNET_B"
]

private_subnet_ids = [
  "YOUR_PRIVATE_SUBNET_A",
  "YOUR_PRIVATE_SUBNET_B"
]

instance_type = "t3.micro"

desired_capacity = 2
min_size         = 2
max_size         = 4
```

---

# 9. Data source — Amazon Linux 2023

Here's an important improvement over hardcoding an AMI ID.

Create `data.tf`:

```hcl
data "aws_ssm_parameter" "amazon_linux" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}
```

Instead of:

```hcl
ami = "ami-xxxxxxxx"
```

we dynamically obtain the current Amazon Linux 2023 AMI published through AWS's public SSM parameter.

This makes the configuration less region-specific.

---

# 10. `locals.tf`

```hcl
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }

  name_prefix = "${var.project_name}-${var.environment}"
}
```

---

# 11. IAM Role for EC2

Our EC2 instances shouldn't use AWS access keys.

Instead:

```text
EC2
 |
 v
IAM Instance Profile
 |
 v
IAM Role
```

Create `iam.tf`:

```hcl
resource "aws_iam_role" "ec2" {
  name = "${local.name_prefix}-ec2-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "ec2.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = local.common_tags
}
```

---

# 12. Attach SSM permissions

We're going to attach:

```text
AmazonSSMManagedInstanceCore
```

This will later allow us to manage instances through Systems Manager rather than opening SSH to the internet.

```hcl
resource "aws_iam_role_policy_attachment" "ssm" {
  role = aws_iam_role.ec2.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}
```

Create the instance profile:

```hcl
resource "aws_iam_instance_profile" "ec2" {
  name = "${local.name_prefix}-ec2-profile"

  role = aws_iam_role.ec2.name

  tags = local.common_tags
}
```

---

# 13. ALB Security Group

Create `security-groups.tf`:

```hcl
resource "aws_security_group" "alb" {
  name        = "${local.name_prefix}-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = var.vpc_id

  ingress {
    description = "HTTP from Internet"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-alb-sg"
    }
  )
}
```

---

# 14. EC2 Security Group

Add:

```hcl
resource "aws_security_group" "application" {
  name        = "${local.name_prefix}-app-sg"
  description = "Security group for application EC2 instances"
  vpc_id      = var.vpc_id

  ingress {
    description     = "HTTP from Application Load Balancer"
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    description = "Outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-app-sg"
    }
  )
}
```

Notice this:

```hcl
security_groups = [aws_security_group.alb.id]
```

We are **not** saying:

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

for EC2 port 80.

Only the ALB can reach the EC2 application port.

---

# 15. Create Launch Template

Create `launch-template.tf`.

```hcl
resource "aws_launch_template" "application" {
  name_prefix = "${local.name_prefix}-"

  image_id = data.aws_ssm_parameter.amazon_linux.value

  instance_type = var.instance_type

  iam_instance_profile {
    name = aws_iam_instance_profile.ec2.name
  }

  vpc_security_group_ids = [
    aws_security_group.application.id
  ]

  user_data = base64encode(<<-EOF
    #!/bin/bash

    dnf update -y

    dnf install -y nginx

    systemctl enable nginx
    systemctl start nginx

    cat > /usr/share/nginx/html/index.html <<HTML
    <!DOCTYPE html>
    <html>
    <head>
        <title>Terraform Phase 3</title>
    </head>
    <body>
        <h1>Terraform AWS Phase 3</h1>
        <p>Application Load Balancer + EC2 + Auto Scaling</p>
        <p>Hostname: $(hostname)</p>
        <p>Instance ID: $(TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"); curl -sH "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)</p>
    </body>
    </html>
    HTML
  EOF
  )

  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"
    http_put_response_hop_limit = 1
  }

  tag_specifications {
    resource_type = "instance"

    tags = merge(
      local.common_tags,
      {
        Name = "${local.name_prefix}-instance"
      }
    )
  }

  tags = local.common_tags
}
```

---

# 16. Why use a Launch Template?

Instead of manually creating EC2 instances:

```text
EC2-1
EC2-2
EC2-3
```

we define the blueprint once:

```text
Launch Template
      |
      +-- AMI
      +-- Instance Type
      +-- Security Group
      +-- IAM Role
      +-- User Data
      +-- Metadata Settings
```

Then Auto Scaling uses the blueprint.

AWS describes the launch template as the configuration from which instances in the Auto Scaling group are launched. ([AWS Documentation][2])

---

# 17. Create Target Group

Create `target-group.tf`:

```hcl
resource "aws_lb_target_group" "application" {
  name = "${var.project_name}-tg"

  port     = 80
  protocol = "HTTP"

  vpc_id = var.vpc_id

  target_type = "instance"

  health_check {
    enabled = true

    protocol = "HTTP"
    path     = "/"

    port = "traffic-port"

    healthy_threshold   = 2
    unhealthy_threshold = 3

    timeout  = 5
    interval = 30

    matcher = "200"
  }

  tags = local.common_tags
}
```

The Target Group represents the EC2 instances receiving traffic.

```text
ALB
 |
 v
Target Group
 |
 +---- EC2
 |
 +---- EC2
```

---

# 18. Create Application Load Balancer

Create `load-balancer.tf`:

```hcl
resource "aws_lb" "application" {
  name = "${var.project_name}-alb"

  internal = false

  load_balancer_type = "application"

  security_groups = [
    aws_security_group.alb.id
  ]

  subnets = var.public_subnet_ids

  enable_deletion_protection = false

  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-alb"
    }
  )
}
```

The ALB goes into the **public subnets**.

Because we supplied two public subnets:

```hcl
subnets = var.public_subnet_ids
```

the ALB spans two AZs.

---

# 19. Create ALB Listener

Add to `load-balancer.tf`:

```hcl
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.application.arn

  port     = 80
  protocol = "HTTP"

  default_action {
    type = "forward"

    target_group_arn = aws_lb_target_group.application.arn
  }
}
```

Traffic now flows:

```text
Internet
   |
   | HTTP :80
   v
ALB
   |
   | HTTP :80
   v
Target Group
   |
   v
EC2
```

---

# 20. Create Auto Scaling Group

Create `autoscaling.tf`:

```hcl
resource "aws_autoscaling_group" "application" {
  name = "${local.name_prefix}-asg"

  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity

  vpc_zone_identifier = var.private_subnet_ids

  health_check_type = "ELB"

  health_check_grace_period = 120

  target_group_arns = [
    aws_lb_target_group.application.arn
  ]

  launch_template {
    id = aws_launch_template.application.id

    version = aws_launch_template.application.latest_version
  }

  tag {
    key                 = "Name"
    value               = "${local.name_prefix}-asg-instance"
    propagate_at_launch = true
  }

  tag {
    key                 = "Project"
    value               = var.project_name
    propagate_at_launch = true
  }

  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }

  tag {
    key                 = "ManagedBy"
    value               = "Terraform"
    propagate_at_launch = true
  }
}
```

This is a key architectural point:

```text
ALB
 |
 v
Auto Scaling Group
 |
 +---- Private Subnet A
 |         |
 |        EC2
 |
 +---- Private Subnet B
           |
          EC2
```

AWS Auto Scaling can distribute instances across the enabled AZs and rebalance them when necessary. ([AWS Documentation][1])

---

# 21. Add Auto Scaling Policy

Now we make it actually scale.

Add to `autoscaling.tf`:

```hcl
resource "aws_autoscaling_policy" "cpu" {
  name = "${local.name_prefix}-cpu-scaling"

  policy_type = "TargetTrackingScaling"

  autoscaling_group_name = aws_autoscaling_group.application.name

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }

    target_value = 50.0
  }
}
```

Conceptually:

```text
                CPU
                 |
        +--------+--------+
        |                 |
      > 50%             < 50%
        |                 |
        v                 v
    Scale Out          Scale In
```

AWS supports target-tracking policies for Auto Scaling groups, including CPU-based scaling. ([AWS Documentation][2])

---

# 22. Outputs

Create `outputs.tf`:

```hcl
output "alb_dns_name" {
  description = "Application Load Balancer DNS name"
  value       = aws_lb.application.dns_name
}

output "alb_url" {
  description = "Application URL"
  value       = "http://${aws_lb.application.dns_name}"
}

output "target_group_arn" {
  description = "Target group ARN"
  value       = aws_lb_target_group.application.arn
}

output "autoscaling_group_name" {
  description = "Auto Scaling Group name"
  value       = aws_autoscaling_group.application.name
}

output "launch_template_id" {
  description = "Launch Template ID"
  value       = aws_launch_template.application.id
}

output "application_security_group_id" {
  description = "Application security group ID"
  value       = aws_security_group.application.id
}
```

---

# 23. Your Complete Project

You should now have:

```text
terraform-aws-phase3-compute/
│
├── provider.tf
├── variables.tf
├── terraform.tfvars
├── data.tf
├── locals.tf
│
├── iam.tf
├── security-groups.tf
├── launch-template.tf
├── target-group.tf
├── load-balancer.tf
├── autoscaling.tf
│
└── outputs.tf
```

---

# 24. Initialize Terraform

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

Expected:

```text
Success! The configuration is valid.
```

---

# 25. Run Plan

Now:

```powershell
terraform plan
```

Don't rush to apply.

Look carefully at what Terraform wants to create.

You should see resources roughly corresponding to:

```text
IAM Role
IAM Policy Attachment
IAM Instance Profile

ALB Security Group
EC2 Security Group

Launch Template

Target Group

Application Load Balancer
ALB Listener

Auto Scaling Group
Auto Scaling Policy
```

---

# 26. Apply

Once you've reviewed the plan:

```powershell
terraform apply
```

Then:

```text
yes
```

The ASG should launch two instances.

The instances will:

```text
1. Launch from Amazon Linux
2. Receive IAM role
3. Install nginx
4. Start nginx
5. Register with Target Group
6. Pass ALB health check
7. Receive traffic
```

---

# 27. Get the application URL

Run:

```powershell
terraform output alb_url
```

You'll get something similar to:

```text
"http://terraform-phase3-alb-xxxxxxxx.ap-south-1.elb.amazonaws.com"
```

Open it in your browser.

You should see:

```text
Terraform AWS Phase 3

Application Load Balancer + EC2 + Auto Scaling

Hostname: ip-10-0-11-xxx
Instance ID: i-xxxxxxxx
```

Refresh the page several times.

You may eventually see a different hostname/instance when traffic is distributed across the healthy targets.

---

# 28. Verify the Target Group

Use:

```powershell
aws elbv2 describe-target-health `
  --target-group-arn YOUR_TARGET_GROUP_ARN
```

You want to see:

```text
State: healthy
```

for the instances.

The ALB only routes requests to healthy targets, and with the ASG health-check integration enabled, unhealthy instances can be replaced. ([AWS Documentation][2])

---

# 29. Verify Auto Scaling

Run:

```powershell
aws autoscaling describe-auto-scaling-groups `
  --auto-scaling-group-names YOUR_ASG_NAME
```

You should see:

```text
MinSize:     2
DesiredSize: 2
MaxSize:     4
```

---

# 30. Verify EC2 Instances

Run:

```powershell
aws ec2 describe-instances `
  --filters "Name=tag:Project,Values=terraform-phase3"
```

You should see two instances.

Conceptually:

```text
                 Auto Scaling Group
                         |
              +----------+----------+
              |                     |
              v                     v
          EC2 Instance A       EC2 Instance B
          Private Subnet A     Private Subnet B
          AZ-A                 AZ-B
```

---

# 31. Test Auto Scaling

Now we do something more interesting.

We want to generate CPU load on the EC2 instances and observe Auto Scaling.

Because we're deliberately avoiding public SSH access, we'll use **AWS Systems Manager** later for administration.

First verify the instances have the SSM role:

```text
AmazonSSMManagedInstanceCore
```

Then check:

```powershell
aws ssm describe-instance-information
```

If the instances appear there, they can be managed through SSM.

---

# 32. Why we didn't open SSH

You might be tempted to add:

```hcl
ingress {
  from_port   = 22
  to_port     = 22
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

**Don't do that.**

Our architecture is:

```text
Internet
   |
   v
ALB
   |
   v
EC2
```

There is no reason for the application instances to expose SSH directly to the internet.

We'll use:

```text
AWS Systems Manager
        |
        v
      EC2
```

instead.

This is also why we added the IAM instance profile.

---

# 33. Test the Auto Scaling policy

The target tracking policy is:

```hcl
target_value = 50.0
```

So AWS attempts to keep the group's average CPU utilization around the configured target.

Conceptually:

```text
                    CPU Load
                       |
                       v
               +---------------+
               | CloudWatch    |
               | ASG CPU       |
               +-------+-------+
                       |
                       v
                Scaling Policy
                       |
              +--------+--------+
              |                 |
          High CPU           Low CPU
              |                 |
              v                 v
          Scale Out          Scale In
```

Don't expect an immediate scale-out just because you refresh the browser. Auto Scaling uses CloudWatch metrics and scaling evaluation rather than request-by-request scaling.

---

# 34. Important Terraform Dependency Chain

Look at what we've built:

```text
VPC
 |
 +-- Public Subnets
 |       |
 |       v
 |      ALB
 |
 +-- Private Subnets
         |
         v
     Auto Scaling
         |
         v
   Launch Template
         |
         +-- AMI
         +-- IAM
         +-- Security Group
         +-- User Data
```

Terraform builds a dependency graph from references such as:

```hcl
vpc_id = var.vpc_id
```

and:

```hcl
aws_security_group.application.id
```

and:

```hcl
aws_lb_target_group.application.arn
```

Understanding this graph becomes extremely important when your Terraform projects become large.

---

# 35. One thing we'll improve later

Right now Phase 3 gets VPC IDs through:

```hcl
terraform.tfvars
```

That's intentional for learning.

But this isn't how I want our final architecture to work.

Eventually:

```text
Phase 2
   |
   v
Remote Terraform State
   |
   v
Phase 3
```

Then Phase 3 can consume:

```hcl
data "terraform_remote_state" "network" {
  ...
}
```

and retrieve:

```text
VPC ID
Public Subnets
Private Subnets
```

without manually copying IDs.

We'll introduce that in the **Remote State / Modules phase**.

---

# 36. Another improvement we'll make later

Our current architecture has:

```text
               NAT Gateway
                    |
                    v
          Private EC2 Instances
```

Phase 2 created a single NAT Gateway.

For a more resilient production design, we'd normally evaluate using a NAT Gateway in each AZ so a single-AZ NAT dependency doesn't become a failure point.

Later we'll redesign this:

```text
                 Internet
                    |
            Internet Gateway
                    |
       +------------+------------+
       |                         |
    NAT-A                     NAT-B
       |                         |
       v                         v
Private-A                   Private-B
```

This is part of the production-hardening stage.

---

# 37. Phase 3 Challenges

Don't just stop after seeing nginx.

## Challenge 1 — Change the application

Modify the `user_data` so the page says:

```text
IT Community Platform
Environment: DEV
Managed by: Terraform
```

Also display:

```text
Availability Zone
Instance ID
Hostname
```

---

## Challenge 2 — Change ASG capacity

Change:

```hcl
desired_capacity = 2
min_size         = 2
max_size         = 4
```

to:

```hcl
desired_capacity = 3
min_size         = 2
max_size         = 6
```

Run:

```powershell
terraform plan
```

Before applying, predict what Terraform will change.

Then:

```powershell
terraform apply
```

Check:

```powershell
aws autoscaling describe-auto-scaling-groups ...
```

---

# 38. Challenge 3 — Make the EC2 SG stricter

Currently:

```text
ALB → EC2 :80
```

Create a separate security group rule for future application traffic.

Think about this:

```text
ALB SG
  |
  | 80
  v
Application SG
```

and later:

```text
Application SG
  |
  | 5432
  v
Database SG
```

This will prepare you for Phase 4.

---

# 39. Challenge 4 — Break the application

This is a very good hands-on exercise.

Temporarily modify the user-data so nginx doesn't start:

```bash
systemctl stop nginx
```

Then watch the Target Group.

The instance should eventually become:

```text
unhealthy
```

The ASG's ELB health check integration should cause the unhealthy instance to be replaced. AWS documents this interaction between load-balancer health checks and Auto Scaling. ([AWS Documentation][2])

This is one of the most important things to actually observe rather than merely read about.

---

# 40. Phase 3 Completion Checklist

```text
PHASE 3
────────────────────────────────────

Terraform
[ ] provider
[ ] variables
[ ] data sources
[ ] locals
[ ] IAM
[ ] security groups
[ ] launch template
[ ] target group
[ ] ALB
[ ] listener
[ ] Auto Scaling
[ ] scaling policy
[ ] outputs

AWS
[ ] EC2
[ ] IAM Role
[ ] IAM Instance Profile
[ ] Application Load Balancer
[ ] Target Group
[ ] Auto Scaling Group
[ ] CloudWatch scaling
[ ] Private EC2
[ ] Multi-AZ deployment

Testing
[ ] terraform init
[ ] terraform fmt
[ ] terraform validate
[ ] terraform plan
[ ] terraform apply
[ ] Open ALB URL
[ ] Check target health
[ ] Check EC2 instances
[ ] Check ASG
[ ] Test unhealthy instance
[ ] Test scaling

Cleanup
[ ] terraform destroy
```

## What you've built

You now have the core of a real three-tier AWS platform:

```text
              ┌───────────────────────┐
              │       INTERNET        │
              └───────────┬───────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │       ALB       │
                 │  Public Subnets │
                 └────────┬────────┘
                          │
                 ┌────────▼────────┐
                 │   Target Group  │
                 └────────┬────────┘
                          │
             ┌────────────┴────────────┐
             │                         │
             ▼                         ▼
       ┌─────────────┐          ┌─────────────┐
       │ EC2 / AZ-A  │          │ EC2 / AZ-B  │
       │   Private   │          │   Private   │
       └──────┬──────┘          └──────┬──────┘
              │                        │
              └──────────┬─────────────┘
                         ▼
                  Auto Scaling Group
                         │
                    CloudWatch
                         │
                    Scale Out/In
```

**Next milestone: Phase 4 — RDS PostgreSQL + Database Security Group + Secrets Manager + private database subnets.** That phase will teach you how to connect the application tier to a properly isolated database tier, and we'll deliberately avoid putting database credentials into Terraform code.

[1]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-benefits.html?utm_source=chatgpt.com "Auto Scaling benefits for application architecture - Amazon EC2 Auto Scaling"
[2]: https://docs.aws.amazon.com/autoscaling/ec2/userguide/tutorial-ec2-auto-scaling-load-balancer.html?utm_source=chatgpt.com "Tutorial: Set up a scaled and load-balanced application - Amazon EC2 Auto Scaling"
