# Terraform - Advanced Topics

---

## Chapter 25: Remote State and State Management

### Backend Configuration (S3 Example)

```hcl
terraform {
  backend "s3" {
    bucket         = "company-terraform-state"
    key            = "production/app/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

### Create the State Bucket (Bootstrap)

```hcl
# bootstrap/main.tf — run this ONCE manually
resource "aws_s3_bucket" "state" {
  bucket = "company-terraform-state"

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"
    }
  }
}

resource "aws_dynamodb_table" "locks" {
  name         = "terraform-state-lock"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

### Reading Remote State (Cross-Project References)

```hcl
# In project B, read state from project A
data "terraform_remote_state" "networking" {
  backend = "s3"
  config = {
    bucket = "company-terraform-state"
    key    = "production/networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Use outputs from the other project
resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.networking.outputs.public_subnet_ids[0]
}
```

---

## Chapter 26: Sensitive Data and Secrets

### Never Hardcode Secrets!

```hcl
# ❌ NEVER DO THIS
variable "db_password" {
  default = "super-secret-password"    # This goes in state file and git!
}

# ✅ DO THIS: Mark as sensitive
variable "db_password" {
  type      = string
  sensitive = true
  # Provide via: TF_VAR_db_password env var, or .tfvars (gitignored)
}
```

### Using AWS Secrets Manager

```hcl
# Store secret in AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name = "production/db-password"
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = var.db_password
}

# Read secret in another config
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "production/db-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

### Using HashiCorp Vault

```hcl
provider "vault" {
  address = "https://vault.company.com"
}

data "vault_generic_secret" "db" {
  path = "secret/production/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db.data["username"]
  password = data.vault_generic_secret.db.data["password"]
}
```

---

## Chapter 27: Terraform in CI/CD

### GitHub Actions Pipeline

```yaml
# .github/workflows/terraform.yml
name: Terraform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TF_VERSION: "1.7.0"
  AWS_REGION: "us-east-1"

jobs:
  terraform:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
      pull-requests: write

    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/terraform-role
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: terraform init

      - name: Terraform Format Check
        run: terraform fmt -check

      - name: Terraform Validate
        run: terraform validate

      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color -out=tfplan
        continue-on-error: true

      # Comment plan output on PR
      - name: Comment Plan on PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const output = `#### Terraform Plan 📖
            \`\`\`
            ${{ steps.plan.outputs.stdout }}
            \`\`\`
            `;
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

      # Only apply on push to main
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

---

## Chapter 28: Project Structure for Large Teams

### Layout for Multi-Environment

```
infrastructure/
├── modules/                          ← Reusable modules
│   ├── vpc/
│   ├── ecs-service/
│   ├── rds/
│   └── s3-static-site/
│
├── environments/                     ← Per-environment config
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf               ← dev state location
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── backend.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── backend.tf
│       └── terraform.tfvars
│
└── global/                           ← Shared (IAM, DNS, etc.)
    ├── iam/
    ├── route53/
    └── s3/
```

### Each Environment (e.g., environments/production/main.tf)

```hcl
module "vpc" {
  source = "../../modules/vpc"

  vpc_name           = "production"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

module "database" {
  source = "../../modules/rds"

  vpc_id             = module.vpc.vpc_id
  subnet_ids         = module.vpc.private_subnet_ids
  instance_class     = "db.r6g.large"
  multi_az           = true
  deletion_protection = true
}

module "app" {
  source = "../../modules/ecs-service"

  vpc_id       = module.vpc.vpc_id
  subnet_ids   = module.vpc.private_subnet_ids
  db_endpoint  = module.database.endpoint
  desired_count = 6
}
```

---

## Chapter 29: Real-World Use Cases

### Use Case 1: Full AWS Web Application

```hcl
# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"

  name = "webapp-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.10.0/24", "10.0.11.0/24"]

  enable_nat_gateway = true
}

# Security Group
resource "aws_security_group" "alb" {
  name   = "alb-sg"
  vpc_id = module.vpc.vpc_id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
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

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "webapp-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = module.vpc.public_subnets
}

resource "aws_lb_target_group" "app" {
  name     = "webapp-tg"
  port     = 3000
  protocol = "HTTP"
  vpc_id   = module.vpc.vpc_id

  health_check {
    path                = "/health"
    healthy_threshold   = 2
    unhealthy_threshold = 10
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Auto Scaling Group
resource "aws_launch_template" "app" {
  name_prefix   = "webapp-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = "t3.small"

  user_data = base64encode(templatefile("userdata.sh", {
    db_endpoint = aws_db_instance.main.endpoint
  }))

  vpc_security_group_ids = [aws_security_group.app.id]
}

resource "aws_autoscaling_group" "app" {
  desired_capacity    = 2
  max_size            = 10
  min_size            = 1
  vpc_zone_identifier = module.vpc.private_subnets
  target_group_arns   = [aws_lb_target_group.app.arn]

  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "webapp-instance"
    propagate_at_launch = true
  }
}

# RDS Database
resource "aws_db_instance" "main" {
  identifier     = "webapp-db"
  engine         = "postgres"
  engine_version = "16.1"
  instance_class = "db.t3.medium"

  allocated_storage     = 20
  max_allocated_storage = 100

  db_name  = "webapp"
  username = "admin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]

  multi_az            = true
  skip_final_snapshot = false
  final_snapshot_identifier = "webapp-final-snapshot"

  backup_retention_period = 7
  deletion_protection     = true
}

resource "aws_db_subnet_group" "main" {
  name       = "webapp-db-subnet"
  subnet_ids = module.vpc.private_subnets
}
```

### Use Case 2: Static Website (S3 + CloudFront)

```hcl
resource "aws_s3_bucket" "website" {
  bucket = "my-website-bucket-unique-name"
}

resource "aws_s3_bucket_website_configuration" "website" {
  bucket = aws_s3_bucket.website.id

  index_document {
    suffix = "index.html"
  }
  error_document {
    key = "error.html"
  }
}

resource "aws_s3_bucket_public_access_block" "website" {
  bucket = aws_s3_bucket.website.id

  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_cloudfront_distribution" "website" {
  origin {
    domain_name = aws_s3_bucket_website_configuration.website.website_endpoint
    origin_id   = "S3-Website"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "http-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  enabled             = true
  default_root_object = "index.html"

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-Website"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies { forward = "none" }
    }
  }

  restrictions {
    geo_restriction { restriction_type = "none" }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

output "website_url" {
  value = "https://${aws_cloudfront_distribution.website.domain_name}"
}
```

### Use Case 3: ECS Fargate Service

```hcl
# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "webapp-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Task Definition
resource "aws_ecs_task_definition" "app" {
  family                   = "webapp"
  network_mode             = "awsvpc"
  requires_compatibilities = ["FARGATE"]
  cpu                      = "256"
  memory                   = "512"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn

  container_definitions = jsonencode([
    {
      name      = "app"
      image     = "${aws_ecr_repository.app.repository_url}:latest"
      essential = true
      portMappings = [
        {
          containerPort = 3000
          protocol      = "tcp"
        }
      ]
      environment = [
        { name = "NODE_ENV", value = "production" }
      ]
      secrets = [
        {
          name      = "DATABASE_URL"
          valueFrom = aws_secretsmanager_secret.db_url.arn
        }
      ]
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.app.name
          "awslogs-region"        = "us-east-1"
          "awslogs-stream-prefix" = "app"
        }
      }
    }
  ])
}

# ECS Service
resource "aws_ecs_service" "app" {
  name            = "webapp"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = 3
  launch_type     = "FARGATE"

  network_configuration {
    subnets          = module.vpc.private_subnets
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false
  }

  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app"
    container_port   = 3000
  }

  deployment_circuit_breaker {
    enable   = true
    rollback = true
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "app" {
  max_capacity       = 20
  min_capacity       = 3
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-auto-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.app.resource_id
  scalable_dimension = aws_appautoscaling_target.app.scalable_dimension
  service_namespace  = aws_appautoscaling_target.app.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

---

## Chapter 30: Testing Terraform

### Terraform Validate (Basic)

```bash
terraform validate
```

### Terraform Plan (Integration Test)

```bash
terraform plan -detailed-exitcode
# Exit code 0: No changes
# Exit code 1: Error
# Exit code 2: Changes needed
```

### Terratest (Go-based Testing)

```go
// test/vpc_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVpcModule(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../modules/vpc",
        Vars: map[string]interface{}{
            "vpc_name": "test-vpc",
            "vpc_cidr": "10.0.0.0/16",
        },
    }

    // Clean up after test
    defer terraform.Destroy(t, terraformOptions)

    // Deploy
    terraform.InitAndApply(t, terraformOptions)

    // Validate
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)
}
```

### Terraform Test (Built-in, v1.6+)

```hcl
# tests/vpc.tftest.hcl
run "create_vpc" {
  command = apply

  variables {
    vpc_name = "test-vpc"
    vpc_cidr = "10.0.0.0/16"
  }

  assert {
    condition     = aws_vpc.this.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR block is incorrect"
  }

  assert {
    condition     = aws_vpc.this.enable_dns_hostnames == true
    error_message = "DNS hostnames should be enabled"
  }
}
```

```bash
terraform test
```

---

## Chapter 31: Best Practices Summary

### Code Organization
1. **One module per concern** — VPC, ECS, RDS should be separate modules
2. **Keep root modules thin** — mostly module calls
3. **Separate state per environment** — dev and prod should never share state
4. **Use consistent naming** — `${project}-${environment}-${resource}`

### Security
5. **Never commit secrets** — use variables with `sensitive = true`
6. **Encrypt state** — enable S3 encryption, use state locking
7. **Use IAM roles** — not hardcoded credentials
8. **Enable `prevent_destroy`** for critical resources (databases, S3 buckets with data)

### Operations
9. **Always plan before apply** — review changes in CI/CD
10. **Use version constraints** — pin provider and module versions
11. **Tag everything** — environment, team, project, managed-by
12. **Use remote state** — never local state in team settings

### Code Quality
13. **Run `terraform fmt`** — consistent formatting
14. **Run `terraform validate`** — catch syntax errors early
15. **Use `tflint`** — catch common mistakes
16. **Document modules** — description on every variable and output

---

## Complete Terraform Command Cheat Sheet

| Command | Purpose |
|---------|---------|
| `terraform init` | Initialize, download providers |
| `terraform plan` | Preview changes |
| `terraform apply` | Apply changes |
| `terraform destroy` | Delete all resources |
| `terraform fmt` | Format code |
| `terraform validate` | Check syntax |
| `terraform show` | Show current state |
| `terraform output` | Show output values |
| `terraform state list` | List managed resources |
| `terraform state show resource` | Show resource details |
| `terraform state rm resource` | Remove from state |
| `terraform state mv old new` | Rename in state |
| `terraform import resource id` | Import existing resource |
| `terraform workspace list` | List workspaces |
| `terraform workspace new name` | Create workspace |
| `terraform workspace select name` | Switch workspace |
| `terraform graph` | Show dependency graph |
| `terraform providers` | List providers |
| `terraform refresh` | Sync state with reality |
| `terraform taint resource` | Mark for recreation |
| `terraform untaint resource` | Unmark |
| `terraform force-unlock id` | Remove stuck lock |
