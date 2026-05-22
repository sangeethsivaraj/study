# Terraform - Intermediate Concepts

---

## Chapter 11: Resource Dependencies

### Implicit Dependencies (Automatic)

When you reference one resource inside another, Terraform automatically knows the order.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id      # ← Implicit dependency!
  cidr_block = "10.0.1.0/24"
}
# Terraform knows: create VPC first, then subnet
```

### Explicit Dependencies (depends_on)

When there's no direct reference but order still matters:

```hcl
resource "aws_iam_role_policy" "example" {
  role   = aws_iam_role.example.name
  policy = "..."
}

resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  depends_on = [aws_iam_role_policy.example]
  # Wait for IAM policy to exist before creating instance
}
```

### Dependency Graph

```bash
# Visualize dependencies
terraform graph | dot -Tpng > graph.png

# Or just view as text
terraform graph
```

```
                ┌──────────┐
                │   VPC    │
                └────┬─────┘
              ┌──────┴──────┐
              ▼             ▼
        ┌──────────┐  ┌──────────┐
        │ Subnet A │  │ Subnet B │
        └────┬─────┘  └────┬─────┘
             │              │
             ▼              ▼
        ┌──────────┐  ┌──────────┐
        │   EC2    │  │   RDS    │
        └──────────┘  └──────────┘
```

---

## Chapter 12: Count and For_Each (Creating Multiple Resources)

### 12.1 Count (Simple Multiples)

```hcl
# Create 3 identical instances
resource "aws_instance" "web" {
  count = 3

  ami           = "ami-abc123"
  instance_type = "t2.micro"

  tags = {
    Name = "web-server-${count.index}"    # web-server-0, web-server-1, web-server-2
  }
}

# Reference: aws_instance.web[0], aws_instance.web[1], etc.
output "instance_ips" {
  value = aws_instance.web[*].public_ip    # Splat expression: all IPs
}
```

### Conditional Creation with Count

```hcl
variable "create_database" {
  type    = bool
  default = true
}

resource "aws_db_instance" "main" {
  count = var.create_database ? 1 : 0    # Create only if true

  engine         = "postgres"
  instance_class = "db.t3.micro"
}
```

### 12.2 For_Each (Better for Distinct Resources)

```hcl
# Create instances from a map
variable "servers" {
  default = {
    web = {
      instance_type = "t2.micro"
      ami           = "ami-abc123"
    }
    api = {
      instance_type = "t2.small"
      ami           = "ami-abc123"
    }
    worker = {
      instance_type = "t2.medium"
      ami           = "ami-abc123"
    }
  }
}

resource "aws_instance" "servers" {
  for_each = var.servers

  ami           = each.value.ami
  instance_type = each.value.instance_type

  tags = {
    Name = each.key    # "web", "api", "worker"
  }
}

# Reference: aws_instance.servers["web"], aws_instance.servers["api"]
output "web_ip" {
  value = aws_instance.servers["web"].public_ip
}
```

### For_Each with a Set

```hcl
variable "subnet_cidrs" {
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "main" {
  for_each = toset(var.subnet_cidrs)

  vpc_id     = aws_vpc.main.id
  cidr_block = each.value

  tags = {
    Name = "subnet-${each.value}"
  }
}
```

### Count vs For_Each

| Feature | count | for_each |
|---------|-------|----------|
| Index | Numeric (0, 1, 2) | String keys |
| Removing middle item | Recreates subsequent items! | Only removes that item |
| Use when | Simple multiples, conditionals | Distinct resources with identifiers |

**Why for_each is usually better:** If you have 3 items and remove the second, `count` shifts indices (item 2 becomes item 1) causing unnecessary recreation. `for_each` uses stable keys.

---

## Chapter 13: Expressions and Functions

### String Interpolation

```hcl
name = "app-${var.environment}-${var.region}"
# Result: "app-production-us-east-1"
```

### Conditional Expression (Ternary)

```hcl
instance_type = var.environment == "production" ? "t2.large" : "t2.micro"
```

### For Expressions (Transform Lists/Maps)

```hcl
# Transform a list
variable "names" {
  default = ["alice", "bob", "charlie"]
}

output "upper_names" {
  value = [for name in var.names : upper(name)]
  # ["ALICE", "BOB", "CHARLIE"]
}

# Filter a list
output "long_names" {
  value = [for name in var.names : upper(name) if length(name) > 3]
  # ["ALICE", "CHARLIE"]
}

# Transform to a map
output "name_lengths" {
  value = { for name in var.names : name => length(name) }
  # { alice = 5, bob = 3, charlie = 7 }
}
```

### Common Built-in Functions

```hcl
# String functions
upper("hello")              # "HELLO"
lower("HELLO")              # "hello"
replace("hello", "l", "L") # "heLLo"
split(",", "a,b,c")        # ["a", "b", "c"]
join("-", ["a", "b", "c"]) # "a-b-c"
format("Hello, %s!", "world") # "Hello, world!"
trimspace("  hello  ")     # "hello"

# Numeric functions
min(1, 2, 3)               # 1
max(1, 2, 3)               # 3
ceil(4.2)                  # 5
floor(4.9)                 # 4

# Collection functions
length(["a", "b"])         # 2
contains(["a", "b"], "a") # true
merge({a=1}, {b=2})       # {a=1, b=2}
keys({a=1, b=2})          # ["a", "b"]
values({a=1, b=2})        # [1, 2]
lookup({a=1}, "a", 0)     # 1 (with default 0)
flatten([[1,2],[3,4]])     # [1,2,3,4]
distinct([1,1,2,2,3])     # [1,2,3]
concat([1,2], [3,4])      # [1,2,3,4]

# Type conversion
tostring(123)              # "123"
tonumber("123")            # 123
tolist(toset(["a","b"]))  # ["a","b"]
toset(["a","a","b"])      # ["a","b"] (unique)

# File functions
file("${path.module}/script.sh")           # Read file content
filebase64("${path.module}/image.png")     # Read as base64
templatefile("${path.module}/config.tpl", { port = 8080 })

# Encoding
jsonencode({name = "app", port = 80})      # JSON string
yamlencode({name = "app"})                 # YAML string
base64encode("hello")                      # base64
```

### Template Files

**templates/userdata.tpl:**
```bash
#!/bin/bash
echo "Setting up ${server_name}"
apt-get update
apt-get install -y ${packages}
echo "Server running on port ${port}"
```

**main.tf:**
```hcl
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  user_data = templatefile("${path.module}/templates/userdata.tpl", {
    server_name = "web-01"
    packages    = "nginx curl"
    port        = 80
  })
}
```

---

## Chapter 14: Locals (Computed Values)

Locals are like local variables — computed once, reused many times.

```hcl
locals {
  # Common tags for all resources
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "terraform"
    Team        = "platform"
  }

  # Computed name prefix
  name_prefix = "${var.project_name}-${var.environment}"

  # Conditional logic
  is_production = var.environment == "production"
  instance_type = local.is_production ? "t2.large" : "t2.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = local.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}

resource "aws_instance" "api" {
  ami           = "ami-abc123"
  instance_type = local.instance_type

  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-api"
  })
}
```

**When to use locals vs variables:**
- **Variables** = inputs from outside (user provides them)
- **Locals** = internal computed values (calculated from variables/resources)

---

## Chapter 15: Lifecycle Rules

Control how Terraform manages resource creation, updates, and destruction.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  lifecycle {
    # Don't destroy before creating replacement (zero downtime)
    create_before_destroy = true

    # Ignore changes to tags (someone might change them manually)
    ignore_changes = [tags, ami]

    # Never destroy this resource (protection!)
    prevent_destroy = true

    # Custom condition that must be true
    precondition {
      condition     = var.instance_type != "t2.nano"
      error_message = "Instance type t2.nano is too small!"
    }

    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP!"
    }
  }
}
```

| Rule | What it does |
|------|-------------|
| `create_before_destroy` | New resource created first, then old is destroyed |
| `prevent_destroy` | Terraform refuses to destroy (safety for databases!) |
| `ignore_changes` | Don't react to changes in specified attributes |
| `replace_triggered_by` | Force replacement when another resource changes |

---

## Chapter 16: Dynamic Blocks

For generating repeated nested blocks dynamically:

```hcl
variable "ingress_rules" {
  default = [
    { port = 80, description = "HTTP" },
    { port = 443, description = "HTTPS" },
    { port = 22, description = "SSH" },
  ]
}

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id

  dynamic "ingress" {
    for_each = var.ingress_rules

    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = ingress.value.description
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

Without dynamic block, you'd have to write each `ingress` block manually.

---

## Chapter 17: Provisioners (Run Scripts After Creation)

Provisioners run commands on a resource AFTER it's created. Use sparingly — prefer user_data or configuration management tools.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"

  # Run on the remote machine after creation
  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx",
    ]

    connection {
      type        = "ssh"
      user        = "ubuntu"
      private_key = file("~/.ssh/id_rsa")
      host        = self.public_ip
    }
  }

  # Run on YOUR machine (where Terraform runs)
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }

  # Run when resource is DESTROYED
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Server destroyed!' >> destroy.log"
  }
}
```

**Best practice:** Prefer `user_data` for EC2 initialization. Use Ansible/Chef for complex configuration. Provisioners are a last resort.

---

## Chapter 18: Workspaces (Multiple Environments)

Workspaces let you manage multiple state files from the same code (e.g., dev, staging, prod).

```bash
# List workspaces
terraform workspace list
# * default

# Create a new workspace
terraform workspace new staging
terraform workspace new production

# Switch workspace
terraform workspace select staging

# Show current
terraform workspace show
```

### Using Workspace in Code

```hcl
locals {
  environment = terraform.workspace    # "default", "staging", "production"

  instance_type = {
    default    = "t2.micro"
    staging    = "t2.small"
    production = "t2.large"
  }
}

resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = local.instance_type[terraform.workspace]

  tags = {
    Name        = "web-${terraform.workspace}"
    Environment = terraform.workspace
  }
}
```

```bash
# Deploy to staging
terraform workspace select staging
terraform apply

# Deploy to production
terraform workspace select production
terraform apply
```

**Note:** Many teams prefer separate directories or var-files per environment instead of workspaces. Both approaches work.

---

## Chapter 19: Terraform Import (Adopt Existing Resources)

If infrastructure already exists (created manually), you can import it into Terraform management.

### Traditional Import

```bash
# 1. Write the resource block first
# main.tf:
# resource "aws_instance" "existing" {
#   # Will be filled in after import
# }

# 2. Import
terraform import aws_instance.existing i-0abc123def456789

# 3. Run terraform plan to see what attributes to fill in
terraform plan

# 4. Copy the real values into your .tf file until plan shows no changes
```

### Import Block (Terraform 1.5+, preferred)

```hcl
import {
  to = aws_instance.existing
  id = "i-0abc123def456789"
}

resource "aws_instance" "existing" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"
  # Fill in remaining attributes
}
```

```bash
# Generate the configuration automatically
terraform plan -generate-config-out=generated.tf
```

---

## Chapter 20: Moved Blocks (Refactoring)

When you rename or restructure resources without destroying and recreating them:

```hcl
# You renamed aws_instance.web to aws_instance.application
moved {
  from = aws_instance.web
  to   = aws_instance.application
}

resource "aws_instance" "application" {
  ami           = "ami-abc123"
  instance_type = "t2.micro"
}
```

Terraform moves the state entry instead of destroying and recreating.
