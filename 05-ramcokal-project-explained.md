# RamcoKAL Terraform Project — Step by Step (Clear Explanation)

> This document explains your ACTUAL project at `CISInfra_Automation/RamcoKAL/` line by line, so you understand every piece without any doubt.

---

# PART 1: THE BIG PICTURE

## What Does This Project Do?

This Terraform project creates AWS infrastructure for **RamcoKAL** (a Korean client). It builds:

| Resource | What it is | Real-world analogy |
|----------|-----------|-------------------|
| VPC | A private network in AWS | Your own private building |
| Subnet | A section inside the VPC | Floors in the building |
| Internet Gateway | Connects VPC to internet | The main door to outside |
| Route Table | Rules for where traffic goes | Signboards directing people |
| Security Group | Firewall rules per server | Security guards at each room |
| EC2 Instance | A virtual server | A computer in a room |
| EBS Volume | Extra disk attached to EC2 | External hard drive plugged in |
| Elastic IP | A fixed public IP address | A permanent phone number |
| Key Pair | SSH key to access server | A physical key to enter a room |
| VPC Peering | Connects two VPCs together | A bridge between two buildings |

## Why 7 Folders?

Each folder is a SEPARATE ENVIRONMENT with its own infrastructure:

```
RamcoKAL/
├── dev/    → Development environment (for developers to test)
├── dm/     → Data Migration environment (for migrating data)
├── sim/    → Simulation environment (pre-UAT testing)
├── uat/    → User Acceptance Testing (client tests here)
├── prod/   → Production (LIVE system, real users!)
├── dr/     → Disaster Recovery (backup in different region)
└── ec2-s3/ → S3 bucket access policies for servers
```

**Why separate?** Each environment has its OWN:
- VPC (own network, own IP range)
- Servers (different sizes, different count)
- Security rules
- State file (so they don't interfere with each other)

---

# PART 2: THE FOLDER STRUCTURE EXPLAINED

```
CISInfra_Automation/                    ← The entire project (git repo)
│
├── modules/                            ← SHARED CODE (used by ALL environments)
│   ├── vpc/main.tf                     ← Code to create VPC, Subnets, IGW, Routes
│   ├── ec2/main.tf                     ← Code to create EC2, Keys, EIPs
│   ├── security_groups/main.tf         ← Code to create Security Groups + Rules
│   └── ebs/main.tf                     ← Code to create EBS Volumes
│
├── RamcoKAL/                           ← THIS CLIENT'S INFRASTRUCTURE
│   └── dev/ap-northeast-2/             ← DEV environment, Seoul region
│       ├── provider.tf                 ← WHERE to create (AWS, Seoul, State file)
│       ├── main.tf                     ← WHAT to create (calls modules)
│       ├── variable-network.tf         ← Variable DEFINITIONS for networking
│       ├── variable-ec2.tf             ← Variable DEFINITIONS for servers
│       ├── variable-iam.tf             ← Variable DEFINITIONS for IAM
│       ├── network.tfvars              ← Actual VALUES for networking
│       ├── sg.tfvars                   ← Actual VALUES for security groups
│       ├── ec2.tfvars                  ← Actual VALUES for servers
│       ├── iam.tfvars                  ← Actual VALUES for IAM roles
│       └── ebs.tfvars                  ← Actual VALUES for extra disks
```

### The Key Idea: Separation of LOGIC and DATA

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  MODULES (Logic) = "HOW to create things"                         │
│  ─────────────────────────────────────                            │
│  Written ONCE. Contains resource blocks with for_each.            │
│  Does NOT contain any actual values like IP addresses.            │
│                                                                    │
│  .tfvars (Data) = "WHAT specifically to create"                   │
│  ─────────────────────────────────────────                        │
│  Different per environment. Contains actual IPs, names, AMIs.     │
│  Same module code + different data = different environment.       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

# PART 3: FILE BY FILE EXPLANATION

## File 1: provider.tf

**Purpose:** Tells Terraform WHERE to connect and WHERE to save state.

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
  backend "s3" {
    bucket = "kal-state-file"
    key    = "dev-ap-northeast-2.tfstate"
    region = "ap-northeast-2"
  }
}

provider "aws" {
  region = "ap-northeast-2"
}
```

**Line by line:**

| Line | What it does |
|------|-------------|
| `required_providers { aws }` | "I need the AWS plugin to talk to Amazon" |
| `backend "s3"` | "Save my state file in this S3 bucket" |
| `bucket = "kal-state-file"` | The S3 bucket name that stores state |
| `key = "dev-ap-northeast-2.tfstate"` | The filename inside the bucket (unique per env!) |
| `region = "ap-northeast-2"` | S3 bucket is in Seoul |
| `provider "aws" { region }` | "Create all resources in Seoul (ap-northeast-2)" |

**Why S3 backend?**
- State file tracks what Terraform has created
- Storing in S3 means your TEAM can share it
- If state was local (on your laptop), nobody else could run Terraform

**Each environment uses a different key:**
- dev → `dev-ap-northeast-2.tfstate`
- prod → `prod-ap-northeast-2.tfstate`
- dr → `dr-us-east-1.tfstate`

This prevents dev changes from touching prod state.

---

## File 2: main.tf

**Purpose:** The orchestrator. Calls each module in order and passes data between them.

```hcl
module "networks" {
  source = "../../../modules/vpc"

  vpcs              = var.vpcs
  subnets           = var.subnets
  internet_gateways = var.internet_gateways
  route_tables      = var.route_tables
  routes            = var.routes
  vpc_peering       = var.vpc_peering
}
```

**What this block does:**
1. `source = "../../../modules/vpc"` → "Use the code from the shared vpc module"
2. `vpcs = var.vpcs` → "Pass the vpcs variable (from network.tfvars) into the module"
3. The module receives these values and creates VPC, subnets, IGW, routes, and peering

```hcl
module "security_groups" {
  source = "../../../modules/security_groups"

  security_groups      = var.security_groups
  security_group_rules = var.security_group_rules
  vpc_ids              = module.networks.vpc_ids      ← THIS IS KEY!
}
```

**What `module.networks.vpc_ids` means:**
- The networks module OUTPUTS the IDs of VPCs it created
- The security_groups module NEEDS those IDs to attach SGs to the correct VPC
- This creates a DEPENDENCY: networks must be created FIRST

```hcl
module "ec2_instances" {
  source = "../../../modules/ec2"

  subnets_ids        = module.networks.subnets_ids         ← From VPC module
  security_group_ids = module.security_groups.sg_ids        ← From SG module
  ec2_instances      = var.ec2_instances
  ec2_instance_keys  = var.ec2_instance_keys
  secrets            = var.secrets
  instance_eips      = var.instance_eips
  ec2_states         = var.ec2_states
  existing_roles     = var.existing_roles
}
```

**What happens here:**
- EC2 needs subnet IDs (to know WHERE to place servers)
- EC2 needs security group IDs (to know WHICH firewall to attach)
- Both come from previous modules' outputs

```hcl
module "ebs_volumes" {
  source = "../../../modules/ebs"

  ebs_volumes        = var.ebs_volumes
  volume_attachments = var.volume_attachments
  instances_ids      = module.ec2_instances.instance_ids    ← From EC2 module
}
```

**EBS needs instance IDs** to know which server to attach the disk to.

### The Complete Flow (Execution Order):

```
Step 1: terraform reads provider.tf
        → Connects to AWS ap-northeast-2
        → Connects to S3 for state file

Step 2: terraform reads main.tf
        → Sees 4 module calls
        → Calculates dependencies from outputs/inputs

Step 3: EXECUTION ORDER (automatic):
        ┌─────────────────┐
        │  1. VPC Module  │  Creates VPC, Subnets, IGW, Routes, Peering
        └────────┬────────┘
                 │ outputs: vpc_ids, subnets_ids
                 ▼
        ┌─────────────────────┐
        │ 2. Security Groups  │  Creates SGs and all rules
        └────────┬────────────┘
                 │ outputs: sg_ids
                 ▼
        ┌─────────────────┐
        │  3. EC2 Module  │  Creates servers, keys, EIPs, secrets
        └────────┬────────┘
                 │ outputs: instance_ids
                 ▼
        ┌─────────────────┐
        │  4. EBS Module  │  Creates extra disks, attaches to servers
        └─────────────────┘
```

---

## File 3: variable-network.tf (Variable DEFINITIONS)

**Purpose:** Defines the SHAPE of the data. Like a form template — it says what fields exist but not the values.

```hcl
variable "vpcs" {
  type = map(object({
    cidr_block = string
    tags       = map(string)
  }))
  default = {}
}
```

**Breaking this down:**

```
variable "vpcs"                    ← Variable name
  type = map(object({...}))       ← It's a MAP of objects

  What does "map(object({...}))" mean?

  MAP = a key-value dictionary, like:
  {
    "key1" = { ... }
    "key2" = { ... }
  }

  OBJECT = a structured value with specific fields:
  {
    cidr_block = "10.0.0.0/16"    ← must be string
    tags = { "Env" = "DEV" }      ← must be map of strings
  }

  Combined: A dictionary where each entry has cidr_block and tags.
```

**Example of what this variable ACCEPTS:**
```hcl
vpcs = {
  "MyVPC" = {
    cidr_block = "10.0.0.0/16"
    tags = { "Environment" = "DEV" }
  }
  "AnotherVPC" = {
    cidr_block = "172.16.0.0/16"
    tags = { "Environment" = "PROD" }
  }
}
```

The `default = {}` means if no value is provided, it's an empty map (create nothing).

---

## File 4: network.tfvars (Actual VALUES)

**Purpose:** The actual data — IP addresses, names, configurations for DEV environment.

```hcl
vpcs = {
  "KORDVPC" = {
    cidr_block = "10.205.4.0/24"
    tags = {
      "Environment" = "DEV"
    }
  }
}
```

**This tells Terraform:** "Create ONE VPC named KORDVPC with IP range 10.205.4.0/24"

```hcl
subnets = {
  "KORDWAPSN01" = {
    availability_zone = "ap-northeast-2a"
    cidr_block        = "10.205.4.0/27"
    route_table       = "KORDRT"
    vpc_key           = "KORDVPC"
    tags = { "Environment" = "DEV" }
  }
  "KORDELASN01" = {
    availability_zone = "ap-northeast-2a"
    cidr_block        = "10.205.4.32/27"
    route_table       = "KORDRT"
    vpc_key           = "KORDVPC"
    tags = { "Environment" = "DEV" }
  }
  "KORDRMSN01" = {
    availability_zone = "ap-northeast-2a"
    cidr_block        = "10.205.4.64/27"
    route_table       = "KORDRT"
    vpc_key           = "KORDVPC"
    tags = { "Environment" = "DEV" }
  }
}
```

**This creates 3 subnets inside KORDVPC:**

```
VPC: KORDVPC (10.205.4.0/24 = 256 total IPs)
│
├── KORDWAPSN01: 10.205.4.0/27   = IPs 10.205.4.0 to 10.205.4.31   (Web App)
├── KORDELASN01: 10.205.4.32/27  = IPs 10.205.4.32 to 10.205.4.63  (Elasticsearch)
└── KORDRMSN01:  10.205.4.64/27  = IPs 10.205.4.64 to 10.205.4.95  (RMS/Database)
```

**`vpc_key = "KORDVPC"`** — This tells the module "put this subnet inside the VPC named KORDVPC". The module uses this key to look up the VPC's ID.

**`route_table = "KORDRT"`** — Associate this subnet with route table KORDRT.

```hcl
internet_gateways = {
  "KORDIGW" = {
    vpc_key = "KORDVPC"
    tags = { "Environment" = "DEV" }
  }
}
```

**Creates an Internet Gateway** and attaches it to KORDVPC. Without this, nothing in the VPC can reach the internet.

```hcl
route_tables = {
  "KORDRT" = {
    vpc_key = "KORDVPC"
    tags = { "Environment" = "DEV" }
  }
}

routes = {
  "KORDRT_IGW_ROUTE" = {
    destination_cidr_block = "0.0.0.0/0"
    igw_key                = "KORDIGW"
    route_table_key        = "KORDRT"
  }
  "KORDRT_KoreanVPC_Peering_Route" = {
    destination_cidr_block        = "10.33.152.0/23"
    vpc_peering_connection_id_key = "KORDVPC-Korean_VPC"
    route_table_key               = "KORDRT"
  }
}
```

**Route table with 2 routes:**
1. `0.0.0.0/0 → IGW` = "All internet traffic goes through the Internet Gateway"
2. `10.33.152.0/23 → VPC Peering` = "Traffic to 10.33.152.x goes through VPC peering to RamcoPro account"

```hcl
vpc_peering = {
  "KORDVPC-Korean_VPC" = {
    peer_owner_id = "502364528748"
    peer_region   = "ap-northeast-2"
    peer_vpc_id   = "vpc-07a1bfc70d9432111"
    vpc_id_key    = "KORDVPC"
    tags = { "Environment" = "DEV" }
  }
}
```

**VPC Peering:** Connects KORDVPC to another VPC (`vpc-07a1bfc70d9432111`) in a different AWS account (`502364528748` = RamcoPro account). This allows the DEV servers to communicate with servers in RamcoPro's VPC.


---

## File 5: sg.tfvars (Security Group Rules)

**Purpose:** Defines which traffic is allowed to/from each server.

### Step 1: Define Security Groups (the container)

```hcl
security_groups = {
  "KORDMWAPSE03_SG01" = {
    vpc_key     = "KORDVPC"
    description = "Security group for KORDMWAPSE03"
    tags        = { "Environment" = "DEV" }
  }
}
```

This creates an EMPTY security group attached to KORDVPC. Think of it as creating an empty box labeled "Rules for KORDMWAPSE03".

### Step 2: Define Rules (what traffic is allowed)

```hcl
security_group_rules = {
  "KORDMWAPSE03_SG01" = {
    sg_rules = [
      {
        type                  = "ingress"          ← INBOUND traffic
        protocol              = "tcp"
        from_port             = 443
        to_port               = 443
        cidr_blocks           = ["0.0.0.0/0"]     ← From ANYWHERE
        security_group_key    = "KORDMWAPSE03_SG01"
        description           = "HTTPS Ingress"
      },
      {
        type                       = "ingress"
        protocol                   = "tcp"
        from_port                  = 135
        to_port                    = 135
        source_security_group_key  = "KORDMELASE01_SG01"  ← Only from Elasticsearch
        security_group_key         = "KORDMWAPSE03_SG01"
        description                = "KORDMELASE01 to KORDMWAPSE03 135 Ingress"
      },
    ]
  }
}
```

**Understanding each field:**

| Field | Meaning | Example |
|-------|---------|---------|
| `type` | Direction: `ingress` (in) or `egress` (out) | "ingress" = incoming |
| `protocol` | TCP, UDP, or -1 (all) | "tcp" |
| `from_port` / `to_port` | Port range allowed | 443 to 443 = only port 443 |
| `cidr_blocks` | IP addresses allowed | "0.0.0.0/0" = everyone |
| `source_security_group_key` | Allow from specific SG | Only from Elasticsearch server |
| `security_group_key` | Which SG this rule belongs to | Attach to WAPS server's SG |
| `description` | Human-readable label | For documentation |

### Traffic Map for DEV:

```
                    INTERNET
                       │
                       │ 443 (HTTPS)
                       │ 3389 (RDP from specific IPs)
                       ▼
              ┌─────────────────┐
              │  KORDMWAPSE03   │
              │  (Web App)      │
              └───┬────────┬────┘
                  │        │
         135,54567│        │54525,1434
           9200   │        │(SQL ports)
                  ▼        ▼
         ┌────────────┐  ┌────────────┐
         │KORDMELASE01│  │KORDMRMSE03 │
         │(Elastic    │  │(Database/  │
         │ Search)    │  │ RMS)       │
         └────────────┘  └────────────┘

    All servers → Internet (80, 443, 587 for email)
    All servers ← RDP access from specific admin IPs only
```

**Key security rules explained:**

| Rule | What it means |
|------|--------------|
| Port 443 in from 0.0.0.0/0 | Web App accepts HTTPS from internet (users access the app) |
| Port 3389 from 172.17.10.238 | RDP access only from CIS Accops IP (admin remote desktop) |
| Port 54525 from WAPS to RMS | Web App connects to SQL database (custom SQL port) |
| Port 9200 from WAPS to ELAS | Web App sends data to Elasticsearch for search |
| Port 587 out to 0.0.0.0/0 | All servers can send email via AWS SES |
| Port 445 (SMB) | File sharing between servers |

---

## File 6: ec2.tfvars (Server Definitions)

```hcl
ec2_instances = {
  "KORDMWAPSE03" = {
    ami               = "ami-014f818f59586f8bd"
    instance_type     = "r6i.xlarge"
    subnet_key        = "KORDWAPSN01"
    security_group_keys = ["KORDMWAPSE03_SG01"]
    existing_role_key = "SSM-EC2-Instance-Profile"
    tags = { "Environment" = "DEV" }
  }
}
```

**Field by field:**

| Field | Value | Meaning |
|-------|-------|---------|
| `"KORDMWAPSE03"` | (map key) | Server name (becomes the Name tag) |
| `ami` | `"ami-014f818f..."` | The OS image (Windows Server in this case) |
| `instance_type` | `"r6i.xlarge"` | Server size: 4 vCPU, 32 GB RAM |
| `subnet_key` | `"KORDWAPSN01"` | Put in the Web App subnet |
| `security_group_keys` | `["KORDMWAPSE03_SG01"]` | Attach this security group |
| `existing_role_key` | `"SSM-EC2-Instance-Profile"` | Use this existing IAM role (for AWS Systems Manager) |
| `tags` | `{ "Environment" = "DEV" }` | Additional tags |

**How `subnet_key` works:**
```
ec2.tfvars:  subnet_key = "KORDWAPSN01"
                              │
                              ▼
Module looks up: var.subnets_ids["KORDWAPSN01"]
                              │
                              ▼
Gets the actual subnet ID: "subnet-0abc123..."
                              │
                              ▼
Creates EC2 with: subnet_id = "subnet-0abc123..."
```

### Instance EIPs (Elastic IPs)

```hcl
instance_eips = {
  "KORDMWAPSE03" = {
    domain          = "vpc"
    instance_id_key = "KORDMWAPSE03"
    tags = { "Environment" = "DEV" }
  }
}
```

**What this does:** Assigns a FIXED public IP to KORDMWAPSE03. Without an EIP, the public IP changes every time the server restarts.

### Key Pairs and Secrets

```hcl
ec2_instance_keys = {
  "KORDMRMSE03" = {
    tags = { "Environment" = "DEV" }
  }
}

secrets = {
  "KORDMRMSE03" = {
    recovery_window_in_days = 0
    tags = { "Environment" = "DEV" }
  }
}
```

**What happens:**
1. Terraform generates a TLS key pair (public + private key)
2. Creates an AWS Key Pair with the public key
3. Stores the PRIVATE key in AWS Secrets Manager
4. Attaches the key pair to the EC2 instance

This allows SSH/RDP access to the server using the private key stored in Secrets Manager.

### EC2 States

```hcl
ec2_states = {
  "KORDMWAPSE03" = { state = "running" }
  "KORDMELASE01" = { state = "running" }
}
```

**Controls whether servers are running or stopped.** If you change "running" to "stopped", Terraform will stop the server on next apply.

---

## File 7: ebs.tfvars (Extra Disks)

```hcl
ebs_volumes = {
  "KORDMRMSE03_D01" = {
    size              = 250                    ← 250 GB
    type              = "standard"             ← Magnetic (cheapest)
    availability_zone = "ap-northeast-2a"      ← Must match server's AZ!
    tags = { "Environment" = "DEV", "Name" = "KORDMRMSE03" }
  }
  "KORDMRMSE03_D02" = {
    size              = 1024                   ← 1 TB (for database data!)
    type              = "standard"
    availability_zone = "ap-northeast-2a"
    tags = { "Environment" = "DEV", "Name" = "KORDMRMSE03" }
  }
}

volume_attachments = {
  "KORDMRMSE03_D01" = {
    device_name     = "xvdd"                  ← Linux device name (D: drive on Windows)
    instance_id_key = "KORDMRMSE03"           ← Attach to this server
  }
  "KORDMRMSE03_D02" = {
    device_name     = "xvde"                  ← E: drive on Windows
    instance_id_key = "KORDMRMSE03"
  }
}
```

**What this creates:**
- KORDMRMSE03 (database server) gets TWO extra disks:
  - D: drive = 250 GB (for database binaries/logs)
  - E: drive = 1 TB (for database data files)

**Why separate from root volume?** The root volume (C:) has the OS. Data disks are separate so you can:
- Snapshot data independently
- Resize without rebuilding the server
- Keep data if instance is terminated

---

## File 8: iam.tfvars

```hcl
existing_roles = {
  "SSM-EC2-Instance-Profile" = "SSM-EC2-Instance-Profile"
}
```

**What this means:** There's an IAM role called "SSM-EC2-Instance-Profile" that ALREADY EXISTS in AWS (not created by Terraform). All EC2 instances use this role to enable AWS Systems Manager (SSM) — which allows remote management without SSH/RDP.

---

# PART 4: HOW THE MODULES WORK INTERNALLY

## VPC Module (modules/vpc/main.tf)

```hcl
resource "aws_vpc" "vpc" {
  for_each   = var.vpcs
  cidr_block = each.value.cidr_block

  tags = merge(
    { "Name" = each.key, "CreatedBy" = "Terraform" },
    each.value.tags
  )
  lifecycle {
    ignore_changes = all
  }
}
```

**Step-by-step execution:**

```
INPUT (from network.tfvars via main.tf):
  var.vpcs = {
    "KORDVPC" = { cidr_block = "10.205.4.0/24", tags = { "Environment" = "DEV" } }
  }

FOR_EACH LOOP:
  Iteration 1:
    each.key   = "KORDVPC"
    each.value = { cidr_block = "10.205.4.0/24", tags = { "Environment" = "DEV" } }

CREATES:
  aws_vpc.vpc["KORDVPC"] with:
    cidr_block = "10.205.4.0/24"
    tags = {
      Name        = "KORDVPC"
      CreatedBy   = "Terraform"
      Environment = "DEV"
    }

LIFECYCLE:
  ignore_changes = all  → Once created, NEVER modify even if values change
```

**The output passes IDs to other modules:**

```hcl
# modules/vpc/output.tf
output "vpc_ids" {
  value = { for k, v in aws_vpc.vpc : k => v.id }
}
# Result: { "KORDVPC" = "vpc-0abc123def456" }

output "subnets_ids" {
  value = { for k, v in aws_subnet.subnets : k => v.id }
}
# Result: { "KORDWAPSN01" = "subnet-111", "KORDELASN01" = "subnet-222", "KORDRMSN01" = "subnet-333" }
```

---

## Security Groups Module (modules/security_groups/main.tf)

### Step 1: Create the Security Groups

```hcl
resource "aws_security_group" "sg" {
  for_each = var.security_groups

  name   = each.key
  vpc_id = coalesce(
    try(var.vpc_ids[each.value.vpc_key], null),
    try(var.existing_vpc_ids[each.value.existing_vpc_key], null)
  )
  tags = merge({ "Name" = each.key, "CreatedBy" = "Terraform" }, each.value.tags)

  # Default rules added to ALL security groups:
  ingress {
    cidr_blocks = ["10.40.3.9/32"]
    from_port   = 5985
    to_port     = 5985
    protocol    = "tcp"
    description = "Allow WinRM from Ansible Control node"
  }
  ingress {
    cidr_blocks = ["172.18.83.16/32", "172.18.83.18/32"]
    from_port   = 3389
    to_port     = 3389
    protocol    = "tcp"
    description = "CyberArk RDP Access"
  }

  lifecycle { ignore_changes = all }
}
```

**Important:** Every SG automatically gets:
- WinRM (5985) from Ansible server — for automated configuration
- RDP (3389) from CyberArk — for privileged access management
- These are hardcoded in the module (team policy)

### Step 2: Flatten Nested Rules into Flat Map

```hcl
locals {
  flattened_rules = flatten([
    for sg_key, sg in var.security_group_rules : [
      for rule in sg.sg_rules : {
        key          = "${sg_key}-${rule.type}-${rule.from_port}-${rule.to_port}-${rule.protocol}-${rule.description}"
        type         = rule.type
        from_port    = rule.from_port
        to_port      = rule.to_port
        protocol     = rule.protocol
        sg_id        = aws_security_group.sg[rule.security_group_key].id
        source_sg_id = try(aws_security_group.sg[rule.source_security_group_key].id, null)
        cidr_blocks  = try(rule.cidr_blocks, null)
        description  = rule.description
      }
    ]
  ])
}
```

**Why is this needed?** The input is NESTED:
```
{
  "SG1" = { sg_rules = [rule1, rule2, rule3] }
  "SG2" = { sg_rules = [rule4, rule5] }
}
```

But `for_each` needs a FLAT map:
```
{
  "SG1-ingress-443-tcp-HTTPS" = { ... }
  "SG1-egress-80-tcp-HTTP"    = { ... }
  "SG2-ingress-22-tcp-SSH"    = { ... }
}
```

The `flatten()` + nested `for` expressions convert nested → flat.

### Step 3: Create Each Rule

```hcl
resource "aws_security_group_rule" "rules" {
  for_each = { for rule in local.flattened_rules : rule.key => rule }

  type                     = each.value.type
  from_port                = each.value.from_port
  to_port                  = each.value.to_port
  protocol                 = each.value.protocol
  security_group_id        = each.value.sg_id
  source_security_group_id = each.value.source_sg_id
  cidr_blocks              = each.value.cidr_blocks
  description              = each.value.description
}
```

---

## EC2 Module (modules/ec2/main.tf)

```hcl
resource "aws_instance" "ec2" {
  for_each      = var.ec2_instances
  ami           = each.value.ami
  instance_type = each.value.instance_type

  # Look up subnet ID by key name
  subnet_id = try(
    var.subnets_ids[each.value.subnet_key],
    var.existing_subnet_ids[each.value.existing_subnet_key]
  )

  # Look up security group IDs by key names
  vpc_security_group_ids = [
    for sg_key in coalesce(each.value.security_group_keys, []) :
      var.security_group_ids[sg_key]
  ]

  # Look up IAM role
  iam_instance_profile = coalesce(
    try(var.instance_profiles[each.value.instance_profile_key], null),
    try(var.existing_roles[each.value.existing_role_key], null)
  )

  disable_api_termination = true    ← PROTECT from accidental deletion!

  tags = merge(
    { "Name" = each.key, "CreatedBy" = "Terraform" },
    each.value.tags
  )

  lifecycle {
    ignore_changes = [tags, instance_type, vpc_security_group_ids, user_data]
  }
}
```

**The `try()` pattern explained:**

```hcl
subnet_id = try(
  var.subnets_ids[each.value.subnet_key],           # Try this first
  var.existing_subnet_ids[each.value.existing_subnet_key]  # If first fails, try this
)
```

This allows the ec2 to use EITHER:
- A subnet created by Terraform (subnet_key)
- OR a pre-existing subnet (existing_subnet_key)

If `subnet_key` is set, it uses the new subnet. If it's null but `existing_subnet_key` is set, it uses the existing one.

---

## EBS Module (modules/ebs/main.tf)

```hcl
resource "aws_ebs_volume" "ebs_volume" {
  for_each          = var.ebs_volumes
  availability_zone = each.value.availability_zone
  size              = each.value.size
  type              = each.value.type
  encrypted         = true                         ← Always encrypted!
  tags              = merge({ "CreatedBy" = "Terraform" }, each.value.tags)
}

resource "aws_volume_attachment" "ebs_att" {
  for_each     = var.volume_attachments
  device_name  = each.value.device_name            ← "xvdd" = D: drive
  volume_id    = aws_ebs_volume.ebs_volume[each.key].id
  instance_id  = var.instances_ids[each.value.instance_id_key]  ← From EC2 module output
  force_detach = true
}
```

**The key lookup chain:**
```
ebs.tfvars:             instance_id_key = "KORDMRMSE03"
                                   │
                                   ▼
EBS module receives:    var.instances_ids["KORDMRMSE03"]
                                   │
                                   ▼
This came from:         module.ec2_instances.instance_ids (output)
                                   │
                                   ▼
Which is:               { "KORDMRMSE03" = "i-0abc123..." }
                                   │
                                   ▼
Result:                 instance_id = "i-0abc123..."
```


---

# PART 5: HOW TO RUN (Step by Step Commands)

## Step 1: Navigate to the Environment

```bash
cd CISInfra_Automation/RamcoKAL/dev/ap-northeast-2/
```

You MUST be in the correct environment folder. Running from the wrong folder = changing the wrong environment.

## Step 2: Initialize Terraform

```bash
terraform init
```

**What this does:**
1. Downloads the AWS provider plugin
2. Connects to the S3 backend (`kal-state-file` bucket)
3. Downloads the state file (`dev-ap-northeast-2.tfstate`)
4. Sets up the `.terraform/` local directory

**You only need to run this ONCE** (or when you change backend/provider config).

## Step 3: Plan (Preview Changes)

```bash
terraform plan \
  -var-file="network.tfvars" \
  -var-file="sg.tfvars" \
  -var-file="ec2.tfvars" \
  -var-file="iam.tfvars" \
  -var-file="ebs.tfvars"
```

**What this does:**
1. Reads ALL .tf files in the current directory
2. Reads ALL -var-file values
3. Compares desired state (your code) with actual state (state file)
4. Shows what will be ADDED, CHANGED, or DESTROYED

**Example output:**
```
Plan: 4 to add, 0 to change, 0 to destroy.

  + aws_vpc.vpc["KORDVPC"]                    (will be CREATED)
  + aws_subnet.subnets["KORDWAPSN01"]         (will be CREATED)
  + aws_instance.ec2["KORDMWAPSE03"]          (will be CREATED)
  + aws_ebs_volume.ebs_volume["KORDMRMSE03_D01"] (will be CREATED)
```

**Symbols:**
- `+` = will be created (new resource)
- `~` = will be modified (change in-place)
- `-` = will be destroyed (deleted!)
- `-/+` = will be destroyed and recreated

## Step 4: Apply (Actually Create)

```bash
terraform apply \
  -var-file="network.tfvars" \
  -var-file="sg.tfvars" \
  -var-file="ec2.tfvars" \
  -var-file="iam.tfvars" \
  -var-file="ebs.tfvars"
```

Terraform shows the plan and asks: `Do you want to perform these actions? (yes/no)`

Type `yes` to proceed.

## Step 5: Verify

```bash
# List all resources Terraform manages
terraform state list

# Show details of a specific resource
terraform state show module.ec2_instances.aws_instance.ec2[\"KORDMWAPSE03\"]

# Show all outputs
terraform output
```

---

# PART 6: HOW TO ADD A NEW SERVER (Complete Walkthrough)

Let's say you get a ticket: **"CIS-99999: Create new Application Server KORDNEWSE01"**

### Step 1: Edit sg.tfvars — Add Security Group

```hcl
# Add to security_groups map:
"KORDNEWSE01_SG01" = {
  vpc_key     = "KORDVPC"
  description = "Security group for KORDNEWSE01"
  tags        = { "Environment" = "DEV" }
}

# Add to security_group_rules map:
"KORDNEWSE01_SG01" = {
  sg_rules = [
    {
      type               = "ingress"
      protocol           = "tcp"
      from_port          = 443
      to_port            = 443
      cidr_blocks        = ["0.0.0.0/0"]
      security_group_key = "KORDNEWSE01_SG01"
      description        = "HTTPS Inbound"
    },
    {
      type               = "ingress"
      protocol           = "tcp"
      from_port          = 3389
      to_port            = 3389
      cidr_blocks        = ["172.17.10.238/32"]
      security_group_key = "KORDNEWSE01_SG01"
      description        = "CIS Accops RDP"
    },
    {
      type               = "egress"
      protocol           = "tcp"
      from_port          = 443
      to_port            = 443
      cidr_blocks        = ["0.0.0.0/0"]
      security_group_key = "KORDNEWSE01_SG01"
      description        = "HTTPS Outbound"
    }
  ]
}
```

### Step 2: Edit ec2.tfvars — Add Server Definition

```hcl
# Add to ec2_instances map:
"KORDNEWSE01" = {
  ami               = "ami-014f818f59586f8bd"      # Same Windows AMI as WAPS
  instance_type     = "r6i.large"                  # 2 vCPU, 16 GB
  subnet_key        = "KORDWAPSN01"                # In web app subnet
  security_group_keys = ["KORDNEWSE01_SG01"]
  existing_role_key = "SSM-EC2-Instance-Profile"
  tags = {
    "Environment" = "DEV"
    "CIS-Ticket"  = "CIS-99999"
  }
}

# Add to ec2_states map:
"KORDNEWSE01" = {
  state = "running"
}
```

### Step 3: (Optional) Add EIP if public IP needed

```hcl
# Add to instance_eips map in ec2.tfvars:
"KORDNEWSE01" = {
  domain          = "vpc"
  instance_id_key = "KORDNEWSE01"
  tags = { "Environment" = "DEV" }
}
```

### Step 4: (Optional) Add Extra Disk if needed

```hcl
# Add to ebs.tfvars:
ebs_volumes = {
  "KORDNEWSE01_D01" = {
    size              = 100
    type              = "gp3"                      # SSD (faster than standard)
    availability_zone = "ap-northeast-2a"
    tags = { "Environment" = "DEV", "Name" = "KORDNEWSE01" }
  }
}

volume_attachments = {
  "KORDNEWSE01_D01" = {
    device_name     = "xvdd"
    instance_id_key = "KORDNEWSE01"
  }
}
```

### Step 5: Plan and Apply

```bash
terraform plan \
  -var-file="network.tfvars" \
  -var-file="sg.tfvars" \
  -var-file="ec2.tfvars" \
  -var-file="iam.tfvars" \
  -var-file="ebs.tfvars"

# Review the plan — should show new resources being created
# Then:

terraform apply \
  -var-file="network.tfvars" \
  -var-file="sg.tfvars" \
  -var-file="ec2.tfvars" \
  -var-file="iam.tfvars" \
  -var-file="ebs.tfvars"
```

---

# PART 7: COMPLETE DATA FLOW DIAGRAM

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                          TERRAFORM EXECUTION FLOW                              │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                    INPUT FILES (You edit these)                          │ │
│  │                                                                          │ │
│  │  network.tfvars        sg.tfvars         ec2.tfvars         ebs.tfvars  │ │
│  │  ┌──────────┐         ┌──────────┐      ┌──────────┐      ┌─────────┐ │ │
│  │  │ vpcs     │         │ security │      │ ec2_     │      │ ebs_    │ │ │
│  │  │ subnets  │         │ _groups  │      │ instances│      │ volumes │ │ │
│  │  │ igw      │         │ sg_rules │      │ eips     │      │ attach  │ │ │
│  │  │ routes   │         │          │      │ keys     │      │ ments   │ │ │
│  │  │ peering  │         │          │      │ states   │      │         │ │ │
│  │  └─────┬────┘         └────┬─────┘      └────┬─────┘      └────┬────┘ │ │
│  └────────┼────────────────────┼──────────────────┼──────────────────┼──────┘ │
│           │                    │                  │                  │         │
│           ▼                    ▼                  ▼                  ▼         │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                      main.tf (Orchestrator)                             │ │
│  │                                                                          │ │
│  │   module "networks"    →  module "security_groups"  →  module "ec2"  → │ │
│  │        │                         │                         │            │ │
│  │        │ vpc_ids                 │ sg_ids                  │ inst_ids   │ │
│  │        │ subnet_ids              │                         │            │ │
│  │        └─────────────────────────┘─────────────────────────┘            │ │
│  │                                                     │                    │ │
│  │                                          module "ebs" ◄──────────────── │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                         │                                     │
│                                         ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                        AWS (Real World)                                 │ │
│  │                                                                          │ │
│  │   VPC ──► Subnets ──► IGW ──► Routes ──► SGs ──► EC2 ──► EBS          │ │
│  │   Peering connections                       EIPs, Keys, Secrets         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
│                                         │                                     │
│                                         ▼                                     │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │              STATE FILE (S3: kal-state-file/dev-ap-northeast-2.tfstate) │ │
│  │                                                                          │ │
│  │   Records: what was created, resource IDs, current configuration         │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

# PART 8: KEY TERRAFORM FUNCTIONS USED (Quick Reference)

| Function | What it does | Example in this project |
|----------|-------------|------------------------|
| `for_each` | Create multiple resources from a map | One VPC per entry in `var.vpcs` |
| `each.key` | The map key in current iteration | "KORDVPC", "KORDMWAPSE03" |
| `each.value` | The map value in current iteration | `{ cidr_block = "...", tags = {...} }` |
| `try(a, b)` | Return `a`, or `b` if `a` fails | `try(var.subnets_ids[key], var.existing[key])` |
| `coalesce(a, b)` | Return first non-null | `coalesce(new_vpc_id, existing_vpc_id)` |
| `merge(map1, map2)` | Combine two maps | `merge({Name="x"}, each.value.tags)` |
| `flatten([])` | Nested list → flat list | Converting nested SG rules |
| `concat(list1, list2)` | Join two lists | Combining new + existing SG IDs |
| `contains(list, val)` | Check if list has value | Checking if key exists |
| `lookup(map, key, default)` | Safe map access | Getting value with fallback |
| `lifecycle { ignore_changes }` | Don't update after creation | Protecting from drift |

---

# PART 9: DIFFERENCES BETWEEN DEV AND PROD

| Aspect | Dev (main.tf) | Prod (main.tf) |
|--------|---------------|----------------|
| State key | `dev-ap-northeast-2.tfstate` | `prod-ap-northeast-2.tfstate` |
| NAT Gateway | Not used | Yes (`nat_gateway`, `nat_gateway_eip`) |
| Routes format | `routes` (old format) | `routes_new` (nested format) |
| Existing SGs | Not used | `existing_security_group_names` |
| Module names | `module "networks"` | `module "vpc"` |

Prod has more features because production needs:
- NAT Gateway (private subnets need internet access without public IPs)
- References to existing security groups (created outside Terraform)
- More complex routing

---

# PART 10: COMMON OPERATIONS

| I want to... | Do this |
|-------------|---------|
| Add a new server | Add to `ec2.tfvars` + `sg.tfvars`, then plan/apply |
| Add a security group rule | Edit `sg.tfvars`, add rule to sg_rules list |
| Add extra disk to server | Add to `ebs.tfvars` (volumes + attachments) |
| Stop a server | Change `ec2_states` → `state = "stopped"`, apply |
| Start a server | Change `ec2_states` → `state = "running"`, apply |
| Add EIP to server | Add to `instance_eips` in `ec2.tfvars` |
| Add new subnet | Add to `subnets` in `network.tfvars` |
| Add VPC peering | Add to `vpc_peering` in `network.tfvars` |
| See what Terraform manages | `terraform state list` |
| See server details | `terraform state show module.ec2_instances.aws_instance.ec2[\"NAME\"]` |
| Remove from Terraform (keep in AWS) | `terraform state rm <resource_address>` |
| Import existing resource | `terraform import <address> <aws-id>` |

---

# PART 11: THINGS TO BE CAREFUL ABOUT

1. **Always run `terraform plan` first** — NEVER apply without reviewing the plan
2. **Check you're in the correct folder** — dev/ vs prod/ is life or death
3. **All 5 var-files are required** — missing one causes errors
4. **`ignore_changes = all`** means Terraform won't fix drift — manual changes stick
5. **`disable_api_termination = true`** — you can't terminate EC2 from console without disabling first
6. **Key pairs are shared** — if one server uses a key pair, ALL servers with the same key pair entry share it
7. **State file is critical** — if it's lost/corrupted, Terraform doesn't know what exists
8. **VPC peering needs acceptance** — creating a peering request ≠ the other account accepted it
