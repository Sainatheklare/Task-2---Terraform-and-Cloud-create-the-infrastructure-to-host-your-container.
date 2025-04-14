# Task-2---Terraform-and-Cloud-create-the-infrastructure-to-host-your-container.
Awesome! You’ve got a clear and solid task here. Let’s break this down step-by-step and scaffold the Terraform setup for the ECS (server-based) variant using AWS.

✅ Terraform ECS-Based Deployment Plan
🌐 Infrastructure Overview
We’ll use the following:

VPC with 2 public + 2 private subnets (via module)

Internet Gateway & NAT Gateway

ECS Cluster (Fargate)

ECS Task Definition & Service to run container in private subnets

Application Load Balancer in public subnets

IAM Roles for ECS execution

🗂️ Suggested Directory Structure
css
Copy
Edit
.
├── app/
│   ├── Dockerfile
│   └── main.py
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── terraform.tfvars
│   └── README.md
✅ main.tf (core setup with modules)
h
Copy
Edit
provider "aws" {
  region = var.aws_region
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.2"

  name = "simple-time-vpc"
  cidr = "10.0.0.0/16"

  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
  enable_dns_hostnames = true

  tags = {
    Project = "SimpleTimeService"
  }
}

resource "aws_ecs_cluster" "this" {
  name = "simple-time-cluster"
}

resource "aws_iam_role" "ecs_task_execution" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "ecs_execution_policy" {
  role       = aws_iam_role.ecs_task_execution.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

resource "aws_ecs_task_definition" "app" {
  family                   = "simple-time-task"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_task_execution.arn

  container_definitions = jsonencode([{
    name      = "simple-time-container"
    image     = var.container_image
    portMappings = [{
      containerPort = 5000
      protocol      = "tcp"
    }]
  }])
}

resource "aws_lb" "app" {
  name               = "simple-time-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.lb_sg.id]
  subnets            = module.vpc.public_subnets
}

resource "aws_lb_target_group" "app" {
  name     = "simple-time-tg"
  port     = 5000
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  health_check {
    path                = "/"
    interval            = 30
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
    matcher             = "200"
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

resource "aws_security_group" "lb_sg" {
  name        = "lb-sg"
  description = "Allow HTTP"
  vpc_id      = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_security_group" "ecs_sg" {
  name   = "ecs-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port       = 5000
    to_port         = 5000
    protocol        = "tcp"
    security_groups = [aws_security_group.lb_sg.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_ecs_service" "app" {
  name            = "simple-time-service"
  cluster         = aws_ecs_cluster.this.id
  task_definition = aws_ecs_task_definition.app.arn
  launch_type     = "FARGATE"
  desired_count   = 1

  network_configuration {
    subnets         = module.vpc.private_subnets
    security_groups = [aws_security_group.ecs_sg.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "simple-time-container"
    container_port   = 5000
  }

  depends_on = [aws_lb_listener.http]
}
✅ variables.tf
hcl
Copy
Edit
variable "aws_region" {
  description = "AWS region"
  default     = "us-east-1"
}

variable "container_image" {
  description = "Docker image for SimpleTimeService"
}
✅ terraform.tfvars
hcl
Copy
Edit
container_image = "yourdockerhubusername/simpletimeservice:latest"
✅ README.md (in /terraform)
markdown
Copy
Edit
# Terraform Setup for SimpleTimeService on AWS

## 🧾 Prerequisites

- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)
- [Terraform](https://www.terraform.io/downloads)
- An AWS account with programmatic credentials set via `aws configure`

## ⚙️ Deploy Instructions

1. Clone the repo and go to the `terraform/` directory:

```bash
cd terraform
terraform init
terraform plan
terraform apply
After completion, Terraform will output the load balancer URL.

🛡️ Security
No AWS credentials are stored in this repository. Use environment variables or AWS CLI for authentication.

yaml
Copy
Edit

---

### 🏆 Bonus (Optional Add-ons)

Want to impress?

- ✅ Add a **remote backend config** using S3 + DynamoDB
- ✅ Set up **GitHub Actions** for CI/CD:
  - Build and push Docker image
  - Terraform apply on merge to `main`
- ✅ Use Terraform Workspaces for multi-env support

---

Would you like me to:
1. Package this Terraform code as a `.zip` you can upload
2. Generate a full GitHub repo (with dummy credentials removed)
3. Add CI/CD automation like GitHub Actions?

Let me know how hands-on you want to be with this!







