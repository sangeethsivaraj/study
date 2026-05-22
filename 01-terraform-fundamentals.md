# Terraform - Complete Guide from Scratch to Advanced

---

## Chapter 1: What is Terraform and Why Does It Exist?

### The Problem: Manual Infrastructure

Without Infrastructure as Code (IaC):
- You log into AWS/Azure console, click buttons to create servers
- No record of what you created or why
- Can't reproduce the same setup reliably
- One wrong click can destroy production
- Scaling means more manual clicking
- Different environments (dev/staging/prod) drift apart

### What is Terraform?

Terraform is an **Infrastructure as Code (IaC)** tool by HashiCorp. You write code that describes your desired infrastructure, and Terraform creates/modifies/destroys it automatically.

```
┌─────────────────────────────────────────────────────────────────┐
│                    HOW TERRAFORM WORKS                            │
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  .tf files   │    │  Terraform   │    │  Cloud Provider  │  │
│  │              │    │   Engine     │    │                  │  │
│  │ "I want:     │───►│              │───►│  AWS / Azure /   │  │
│  │  2 servers   │    │ Plan → Apply │    │  GCP / etc.      │  │
│  │  1 database  │    │              │    │                  │  │
│  │  1 network"  │    │              │    │  Creates real    │  │
│  │              │    │              │    │  resources!       │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│                    ┌──────────────┐                               │
│                    │ State File   │                               │
│                    │ (.tfstate)   │                               │
│                    │              │                               │
│                    │ Tracks what  │                               │
│                    │ exists in    │                               │
│                    │ real world   │                               │
│                    └──────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Explanation |
|---------|-------------|
| **Declarative** | You describe WHAT you want, not HOW to create it |
| **Version controlled** | Infrastructure changes tracked in Git |
| **Reproducible** | Same code = same infrastructure, every time |
| **Multi-cloud** | Works with AWS, Azure, GCP, Kubernetes, and 3000+ providers |
| **Plan before apply** | See what will change BEFORE making changes |
| **State management** | Knows what exists, what to add, what to remove |

### Terraform vs Others

| Tool | Type | When to Use |
|------|------|-------------|
| **Terraform** | Infrastructure provisioning | Create servers, databases, networks |
| **Ansible** | Configuration management | Install software, configure servers |
| **CloudFormation** | AWS-only IaC | If you only use AWS |
| **Pulumi** | IaC in real languages | If you prefer Python/TypeScript over HCL |
| **Docker** | Application packaging | Package and run applications |
| **Kubernetes** | Container orchestration | Manage containerized apps |

**Terraform creates the infrastructure. Ansible/Docker configure what runs ON it.**

---

## Chapter 2: Core Concepts

### 2.1 Providers

A provider is a plugin that lets Terraform talk to a specific platform (AWS, Azure, GCP, etc.).

```
┌────────────────────────────────────────────────────┐
│              Terraform Core                          │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │              Provider Plugins                │   │
│  │                                             │   │
│  │  ┌─────┐  ┌─────┐  ┌─────┐  ┌──────────┐ │   │
│  │  │ AWS │  │Azure│  │ GCP │  │Kubernetes│ │   │
│  │  └──┬──┘  └──┬──┘  └──┬──┘  └────┬─────┘ │   │
│  └─────┼────────┼────────┼──────────┼───────┘   │
│         │        │        │          │            │
└─────────┼────────┼────────┼──────────┼────────────┘
          ▼        ▼        ▼          ▼
        AWS      Azure     GCP     K8s Cluster
```

### 2.2 Resources

A resource is a single piece of infrastructure (a server, a database, a network, etc.).

### 2.3 State

Terraform keeps a record (state file) of what it has created. This is how it knows what exists, what needs updating, and what to destroy.

### 2.4 Plan

Before making changes, Terraform shows you a plan — what it WILL do. You review it, then apply.

### 2.5 Modules

Reusable packages of Terraform code. Like functions in programming.

---

## Chapter 3: Installation

### Windows
```powershell
# Using Chocolatey
choco install terraform

# Or using winget
winget install HashiCorp.Terraform

# Or download from: https://developer.hashicorp.com/terraform/downloads
# Extract and add to PATH
```

### Linux (Ubuntu)
```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Install
sudo apt update && sudo apt install terraform
```

### Mac
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

### Verify
```bash
terraform version
# Terraform v1.7.0
```

---

## Chapter 4: HCL (HashiCorp Configuration Language) — The Syntax

### Basic Structure

```hcl
# This is a comment

# Block type "resource", type "aws_instance", name "web"
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"

  tags = {
    Name = "MyWebServer"
  }
}
```

### Data Types

```hcl
# String
name = "hello"

# Number
count = 3

# Boolean
enabled = true

# List (array)
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# Map (object/dictionary)
tags = {
  Name        = "my-server"
  Environment = "production"
}
```

### References

```hcl
# Reference another resource's attribute
resource "aws_instance" "web" {
  subnet_id = aws_subnet.main.id
  #           ^^^^^^^^^^^^^^^^^ resource_type.resource_name.attribute
}
```

---

## Chapter 5: Your First Terraform Project

### 5.1 Project Structure

```
my-first-terraform/
├── main.tf          ← Main resources
├── variables.tf     ← Input variables
├── outputs.tf       ← Output values
├── providers.tf     ← Provider configuration
└── terraform.tfvars ← Variable values
```

### 5.2 providers.tf — Configure the Provider

```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"    # Any 5.x version
    }
  }
}

provider "aws" {
  region = "us-east-1"

  # Authentication (multiple options):
  # Option 1: Environment variables (RECOMMENDED)
  #   export AWS_ACCESS_KEY_ID="your-key"
  #   export AWS_SECRET_ACCESS_KEY="your-secret"
  
  # Option 2: AWS CLI profile
  # profile = "my-profile"
  
  # Option 3: Hardcoded (NEVER do this!)
  # access_key = "AKIA..."    # DON'T!
  # secret_key = "wJal..."    # DON'T!
}
```

### 5.3 variables.tf — Define Inputs

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_name" {
  description = "Name tag for the EC2 instance"
  type        = string
  # No default = required to provide value
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "development"

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}
```

### 5.4 main.tf — Define Resources

```hcl
# Create a VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name        = "${var.instance_name}-vpc"
    Environment = var.environment
  }
}

# Create a subnet
resource "aws_subnet" "main" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.instance_name}-subnet"
  }
}

# Create an EC2 instance
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  subnet_id     = aws_subnet.main.id

  tags = {
    Name        = var.instance_name
    Environment = var.environment
  }
}
```

### 5.5 outputs.tf — Define Outputs

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
}

output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}
```

### 5.6 terraform.tfvars — Provide Values

```hcl
instance_type = "t2.micro"
instance_name = "my-web-server"
environment   = "development"
```

---

## Chapter 6: The Terraform Workflow (Commands)

```
┌─────────────────────────────────────────────────────────────┐
│                                                              │
│   terraform init                                             │
│   ──────────────                                             │
│   Download providers, initialize backend                     │
│              │                                               │
│              ▼                                               │
│   terraform plan                                             │
│   ──────────────                                             │
│   Preview what will change (dry run)                         │
│              │                                               │
│              ▼                                               │
│   terraform apply                                            │
│   ───────────────                                            │
│   Actually create/modify/destroy resources                   │
│              │                                               │
│              ▼                                               │
│   terraform destroy                                          │
│   ─────────────────                                          │
│   Tear down everything                                       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Step by Step

```bash
# 1. Initialize (downloads providers, sets up backend)
terraform init

# 2. Format code (auto-fix formatting)
terraform fmt

# 3. Validate (check for syntax errors)
terraform validate

# 4. Plan (preview changes — ALWAYS do this!)
terraform plan

# Output shows:
# + aws_instance.web         (will be CREATED)
# ~ aws_instance.web         (will be MODIFIED)
# - aws_instance.web         (will be DESTROYED)

# 5. Apply (make the changes)
terraform apply
# Type "yes" to confirm

# Or auto-approve (use in CI/CD):
terraform apply -auto-approve

# 6. Show current state
terraform show

# 7. List resources in state
terraform state list

# 8. Destroy everything
terraform destroy
```

### Understanding the Plan Output

```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami                    = "ami-0c55b159cbfafe1f0"
      + instance_type          = "t2.micro"
      + id                     = (known after apply)
      + public_ip              = (known after apply)
      + tags                   = {
          + "Name" = "my-web-server"
        }
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**Symbols:**
- `+` = will be created
- `-` = will be destroyed
- `~` = will be modified in-place
- `-/+` = will be destroyed and recreated

---

## Chapter 7: State — How Terraform Tracks Reality

### What is State?

The state file (`terraform.tfstate`) is a JSON file that maps your code to real resources.

```
┌────────────────┐          ┌────────────────┐
│  Your Code     │          │  Real World    │
│                │          │                │
│ resource       │          │ AWS EC2        │
│ "aws_instance" │◄────────►│ i-0abc123def   │
│ "web"          │  STATE   │                │
│                │  FILE    │                │
└────────────────┘          └────────────────┘
```

### State Commands

```bash
# List all resources in state
terraform state list
# aws_vpc.main
# aws_subnet.main
# aws_instance.web

# Show details of a specific resource
terraform state show aws_instance.web

# Remove a resource from state (doesn't delete the real resource!)
terraform state rm aws_instance.web

# Move/rename a resource in state
terraform state mv aws_instance.web aws_instance.application

# Import an existing resource into state
terraform import aws_instance.web i-0abc123def456
```

### Remote State (Team Collaboration)

**Problem:** If the state file is local, only YOU have it. Teammates can't use Terraform.

**Solution:** Store state remotely (S3, Azure Blob, Terraform Cloud).

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"    # State locking!
  }
}
```

**State locking** prevents two people from modifying infrastructure simultaneously.

---

## Chapter 8: Variables — All Types and Patterns

### Variable Types

```hcl
# String
variable "name" {
  type    = string
  default = "my-app"
}

# Number
variable "instance_count" {
  type    = number
  default = 3
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List of strings
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map
variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Team        = "backend"
  }
}

# Object (structured)
variable "server_config" {
  type = object({
    instance_type = string
    disk_size     = number
    encrypted     = bool
  })
  default = {
    instance_type = "t2.micro"
    disk_size     = 20
    encrypted     = true
  }
}

# List of objects
variable "users" {
  type = list(object({
    name  = string
    role  = string
    email = string
  }))
}

# Sensitive (won't show in output/logs)
variable "db_password" {
  type      = string
  sensitive = true
}
```

### Providing Variable Values (Priority Order)

```bash
# 1. Command line (highest priority)
terraform apply -var="instance_type=t2.large"

# 2. Variable file
terraform apply -var-file="production.tfvars"

# 3. terraform.tfvars or *.auto.tfvars (auto-loaded)
# Just create the file, it's loaded automatically

# 4. Environment variables
export TF_VAR_instance_type="t2.large"
terraform apply

# 5. Default value in variable block (lowest priority)
```

### terraform.tfvars

```hcl
# terraform.tfvars
instance_type = "t2.micro"
instance_name = "web-server"
environment   = "production"

availability_zones = [
  "us-east-1a",
  "us-east-1b",
]

tags = {
  Project = "MyApp"
  Owner   = "DevOps Team"
}
```

### Multiple .tfvars Files (Per Environment)

```
environments/
├── dev.tfvars
├── staging.tfvars
└── prod.tfvars
```

```bash
terraform apply -var-file="environments/prod.tfvars"
```

---

## Chapter 9: Outputs

```hcl
# Simple output
output "server_ip" {
  description = "The public IP of the web server"
  value       = aws_instance.web.public_ip
}

# Sensitive output (hidden in terminal)
output "db_connection_string" {
  value     = "postgresql://${var.db_user}:${var.db_password}@${aws_db_instance.main.endpoint}/mydb"
  sensitive = true
}

# Output from a list of resources
output "all_server_ips" {
  value = aws_instance.web[*].public_ip
}
```

```bash
# View outputs after apply
terraform output
terraform output server_ip

# Get value in machine-readable format (for scripts)
terraform output -raw server_ip
terraform output -json
```

**Use case:** Pass outputs between Terraform configurations or to other tools (Ansible, scripts).

---

## Chapter 10: Data Sources — Read Existing Resources

Data sources let you READ information about infrastructure that already exists (not managed by this Terraform code).

```hcl
# Get the latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use it in a resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux.id   # Reference with "data." prefix
  instance_type = "t2.micro"
}
```

### Common Data Sources

```hcl
# Get current AWS account info
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

# Get current region
data "aws_region" "current" {}

# Get available AZs
data "aws_availability_zones" "available" {
  state = "available"
}

# Read a file
data "local_file" "config" {
  filename = "${path.module}/config.json"
}

# Get info about an existing VPC
data "aws_vpc" "existing" {
  filter {
    name   = "tag:Name"
    values = ["production-vpc"]
  }
}
```
