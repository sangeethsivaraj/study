# Terraform - Modules (Reusable Infrastructure)

---

## Chapter 21: What are Modules?

### The Problem

You need the same infrastructure pattern in multiple places:
- Same VPC setup for dev, staging, and prod
- Same EC2 + security group + load balancer pattern for 10 services

Copy-pasting code is bad — one fix must be applied everywhere manually.

### The Solution: Modules

A module is a **reusable package of Terraform code**. Like a function in programming.

```
┌─────────────────────────────────────────────────────────────┐
│                     MODULE CONCEPT                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │  Module: "vpc"                                       │    │
│  │                                                      │    │
│  │  Inputs:              Creates:           Outputs:    │    │
│  │  - cidr_block         - VPC              - vpc_id    │    │
│  │  - name               - Subnets          - subnet_ids│    │
│  │  - az_count           - IGW              - igw_id    │    │
│  │                       - Route tables                  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  Usage:                                                      │
│  module "dev_vpc"  { source = "./modules/vpc", name = "dev" }│
│  module "prod_vpc" { source = "./modules/vpc", name = "prod"}│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Chapter 22: Creating Your First Module

### Module Structure

```
project/
├── main.tf                    ← Root module (calls child modules)
├── variables.tf
├── outputs.tf
├── modules/
│   ├── vpc/                   ← VPC module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── ec2/                   ← EC2 module
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── rds/                   ← RDS module
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
```

### Example: VPC Module

**modules/vpc/variables.tf:**
```hcl
variable "vpc_name" {
  description = "Name of the VPC"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.2.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.10.0/24", "10.0.11.0/24"]
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

**modules/vpc/main.tf:**
```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = var.vpc_name
  })
}

resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(var.tags, {
    Name = "${var.vpc_name}-igw"
  })
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id                  = aws_vpc.this.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.vpc_name}-public-${count.index + 1}"
    Tier = "public"
  })
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id            = aws_vpc.this.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = merge(var.tags, {
    Name = "${var.vpc_name}-private-${count.index + 1}"
    Tier = "private"
  })
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(var.tags, {
    Name = "${var.vpc_name}-public-rt"
  })
}

resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```

**modules/vpc/outputs.tf:**
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "IDs of public subnets"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "IDs of private subnets"
  value       = aws_subnet.private[*].id
}

output "igw_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.this.id
}
```

### Using the Module (Root main.tf)

```hcl
module "vpc" {
  source = "./modules/vpc"

  vpc_name             = "my-app-vpc"
  vpc_cidr             = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.11.0/24"]
  availability_zones   = ["us-east-1a", "us-east-1b"]

  tags = {
    Environment = "production"
    Project     = "my-app"
  }
}

# Use module outputs
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]
}
```

---

## Chapter 23: Module Sources

```hcl
# Local path
module "vpc" {
  source = "./modules/vpc"
}

# Terraform Registry (public modules)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# GitHub
module "vpc" {
  source = "github.com/terraform-aws-modules/terraform-aws-vpc?ref=v5.0.0"
}

# Git (any git repo)
module "vpc" {
  source = "git::https://github.com/user/repo.git//modules/vpc?ref=v1.0.0"
}

# S3 bucket
module "vpc" {
  source = "s3::https://s3-eu-west-1.amazonaws.com/my-bucket/vpc-module.zip"
}
```

### Using Public Registry Modules

The Terraform Registry (registry.terraform.io) has thousands of pre-built modules:

```hcl
# AWS VPC (full-featured)
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true    # Cost saving for non-prod

  tags = {
    Environment = "dev"
  }
}

# AWS EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "20.0.0"

  cluster_name    = "my-cluster"
  cluster_version = "1.29"
  vpc_id          = module.vpc.vpc_id
  subnet_ids      = module.vpc.private_subnets

  eks_managed_node_groups = {
    default = {
      instance_types = ["t3.medium"]
      min_size       = 1
      max_size       = 5
      desired_size   = 3
    }
  }
}
```

---

## Chapter 24: Module Patterns (Advanced)

### Composing Modules

```hcl
# Root main.tf — composes multiple modules together
module "networking" {
  source = "./modules/networking"
  # ...
}

module "database" {
  source = "./modules/database"

  vpc_id     = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
}

module "application" {
  source = "./modules/application"

  vpc_id         = module.networking.vpc_id
  subnet_ids     = module.networking.public_subnet_ids
  db_endpoint    = module.database.endpoint
  db_secret_arn  = module.database.secret_arn
}
```

### Module with for_each (Multiple Instances of a Module)

```hcl
variable "environments" {
  default = {
    dev = {
      instance_type = "t2.micro"
      instance_count = 1
    }
    staging = {
      instance_type = "t2.small"
      instance_count = 2
    }
    prod = {
      instance_type = "t2.large"
      instance_count = 3
    }
  }
}

module "app" {
  source   = "./modules/app"
  for_each = var.environments

  environment    = each.key
  instance_type  = each.value.instance_type
  instance_count = each.value.instance_count
}
```
