# RamcoNebula Terraform Project — Step by Step (Clear Explanation)

> Complete explanation of `CISInfra_Automation/nebula/` — every file, every resource, every connection.

---

# PART 1: THE BIG PICTURE

## What is Nebula?

Nebula is a Ramco client whose infrastructure spans **multiple AWS regions** and includes an **Application Load Balancer (ALB)**, VPC Peering across accounts, and both new and existing (pre-Terraform) infrastructure.

## Key Differences from RamcoKAL

| Aspect | RamcoKAL | Nebula |
|--------|----------|--------|
| Primary Region | ap-northeast-2 (Seoul) | ap-southeast-2 (Sydney) + ap-southeast-1 (Singapore) |
| State Bucket | `kal-state-file` | `terraform-state-8460` |
| Environments | dev, dm, sim, uat, prod, dr | sim, prod (2 regions) |
| Load Balancer | None | ALB with HTTPS + SSL |
| Existing Infra | Minimal | Heavy use of existing VPCs, subnets, SGs |
| Sub-clients | Only KAL | NBL (Nebula) + SKPT (Seek Prod) in same account |
| Extra: root folder | None | Cross-account VPC peering accepter + training SG |

## Project Structure

```
nebula/
├── main.tf                  ← ROOT: Cross-account peering (special purpose)
├── variable.tf
├── terraform.tfvars
├── data.tf
│
├── prod/
│   ├── ap-southeast-2/      ← PRODUCTION: Sydney (main region)
│   │   ├── provider.tf
│   │   ├── main.tf          ← VPC + SG + EC2 + EBS + ALB (5 modules!)
│   │   ├── data.tf          ← SSL certificate lookup
│   │   ├── network.tfvars
│   │   ├── sg.tfvars
│   │   ├── ec2.tfvars
│   │   ├── elb.tfvars       ← Load Balancer (unique to Nebula!)
│   │   └── disk.tfvars
│   │
│   └── ap-southeast-1/      ← PRODUCTION: Singapore
│       ├── provider.tf
│       ├── main.tf
│       ├── network.tfvars
│       ├── sg.tfvars
│       └── ec2.tfvars
│
├── sim/
│   └── ap-southeast-2/      ← SIMULATION: Sydney
│       ├── provider.tf
│       ├── main.tf
│       ├── network.tfvars
│       ├── sg.tfvars
│       └── ec2.tfvars
│
└── ec2-s3/                   ← S3 bucket access policies
    ├── main.tf
    └── provider.tf
```

---

# PART 2: THE ROOT FOLDER (nebula/main.tf) — Cross-Account Peering

This is a **special-purpose configuration** that doesn't create standard infra. It manages VPC peering between the Nebula account and a Training account.

## What It Does

```
┌─────────────────────────┐                    ┌─────────────────────────┐
│   TRAINING ACCOUNT      │                    │   NEBULA ACCOUNT        │
│   (different AWS acct)  │                    │   (this account)        │
│                         │    VPC Peering     │                         │
│   VPC: 10.90.0.0/23    │◄──────────────────►│   VPC: Nebula AD VPC   │
│                         │                    │                         │
│   EC2 instances with    │                    │   AD Server             │
│   private IPs           │                    │   (gets SG rules)       │
└─────────────────────────┘                    └─────────────────────────┘
```

### data.tf — Read Remote State from Training Account

```hcl
data "terraform_remote_state" "account_training" {
  backend = "s3"
  config = {
    bucket = "terraform-statefile-5248"
    key    = "test-terraform.tfstate"
    region = "ap-south-1"
  }
}
```

**What this does:** Reads the STATE FILE of another Terraform project (the training account). This gives access to:
- Private IPs of EC2 instances in the training account
- The VPC peering connection ID created from the training side

### main.tf — Accept Peering + Create SG + Add Route

```hcl
# 1. Get private IPs from training account state
locals {
  private_ips = data.terraform_remote_state.account_training.outputs.ec2_instances_private_ips
  peering_id  = data.terraform_remote_state.account_training.outputs.vpc_peering_id_output["tf_vpc_to_nebu_vpc"]
}

# 2. Create SG that allows traffic from those IPs
resource "aws_security_group" "training_sg" {
  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"       # All traffic
    cidr_blocks = [for ip in values(local.private_ips) : "${ip}/32"]
  }
  vpc_id = data.aws_vpc.nebula_vpc_id.id
}

# 3. Attach SG to the Nebula AD server
resource "aws_network_interface_sg_attachment" "sg_attachment" {
  security_group_id    = aws_security_group.training_sg.id
  network_interface_id = data.aws_instance.nebula_ad_existing_instance.network_interface_id
}

# 4. Accept the VPC peering request
resource "aws_vpc_peering_connection_accepter" "peer_accepter" {
  vpc_peering_connection_id = local.peering_id
  auto_accept               = true
}

# 5. Add route so traffic to training VPC goes through peering
resource "aws_route" "peering_route" {
  route_table_id            = data.aws_route_table.nebula_ad_rt.route_table_id
  destination_cidr_block    = "10.90.0.0/23"
  vpc_peering_connection_id = local.peering_id
}
```

**Flow:** Training account creates peering request → Nebula accepts → Nebula adds route → Nebula creates SG allowing training IPs → attaches to AD server.

---

# PART 3: PRODUCTION ap-southeast-2 (Main Environment)

## provider.tf

```hcl
terraform {
  backend "s3" {
    bucket = "terraform-state-8460"
    key    = "prod-ap-southeast-2.tfstate"
    region = "ap-southeast-2"
  }
}
provider "aws" {
  region = "ap-southeast-2"    # Sydney, Australia
}
```

## main.tf — 5 Modules (More Than KAL!)

```hcl
module "networks"         → VPC, Subnets, IGW, NAT, Routes, Peering
module "security_groups"  → SGs + Rules (new + existing)
module "ec2_instances"    → EC2, Keys, EIPs, Secrets
module "ebs_volumes"      → Extra disks
module "load_balancers"   → ALB + Target Group + Listener (NEW!)
```

### The ALB Module (Unique to Nebula)

```hcl
module "load_balancers" {
  source = "../../../modules/elb"

  alb                         = var.alb
  target_group                = var.target_group
  target_group_attachements   = var.target_group_attachements
  lb_listener                 = var.lb_listener
  lb_listener_certificate_arn = data.aws_acm_certificate.ramcoes_com.arn  ← SSL cert!

  security_group_ids = flatten([
    for alb_key, alb_value in var.alb : [
      for sg in alb_value.security_group_keys : module.security_groups.sg_ids[sg]
    ]
  ])

  subnet_ids = flatten([
    for alb_key, alb_value in var.alb : [
      for sn in alb_value.subnet_keys : module.networks.subnets_ids[sn]
    ]
  ])
}
```

**What the flatten+for does:** The ALB config has `security_group_keys = ["SKPTALB_SG01"]` and `subnet_keys = ["Seek_Prod_Public_Subnet", "Seek_Prod_Public_Subnet_02"]`. This code converts key names to actual IDs.

### data.tf — SSL Certificate Lookup

```hcl
data "aws_acm_certificate" "ramcoes_com" {
  domain      = "*.ramcoes.com"
  statuses    = ["ISSUED"]
  most_recent = true
}
```

Finds the wildcard SSL certificate `*.ramcoes.com` in AWS Certificate Manager. Used by the ALB for HTTPS.

---

## network.tfvars — TWO VPCs (Existing + New)

### Existing VPC (Pre-Terraform)

```hcl
existing_vpc_ids = {
  "RamcoNebula_PROD_AUS_VPC" = "vpc-0cb6a9dfd2203ddba"
}

existing_subnet_ids = {
  "NBL_PROD_AUS_SUBNET_WEB_02"    = "subnet-0da69cdbc7ff0a5d5"
  "NBL_PROD_AUS_SUBNET_APP_02"    = "subnet-089701993270f90e3"
  "NBL_PROD_NIFI_SUBNET_01"       = "subnet-04d539aaadba9ab42"
  "NBL_PROD_IPO_Fargate_Subnet"   = "subnet-0c0690d1fb12b5d96"
  "NBL_PROD_AUS_SUBNET_RM_01"     = "subnet-07f05c275a4c460c2"
  "NBL_PROD_AUS_SUBNET_RM_02"     = "subnet-066cfe9b3c1677397"
}
```

**These already exist in AWS** — Terraform doesn't create them but needs their IDs to place EC2 instances.

### New VPC (Terraform-managed): Seek Prod

```hcl
vpcs = {
  "Seek_Prod_VPC" = {
    cidr_block = "10.34.28.0/23"     # 512 IPs
    tags = { "Ticket" = "CIS-34413" }
  }
}
```

**Network layout:**

```
┌──────────── Seek_Prod_VPC (10.34.28.0/23) ─────────────────────────────┐
│                                                                          │
│  PUBLIC SUBNETS (with Internet Gateway):                                 │
│  ┌─────────────────────────┐  ┌─────────────────────────┐              │
│  │ Seek_Prod_Public_Subnet │  │ Seek_Prod_Public_Subnet_02│             │
│  │ 10.34.28.192/28 (AZ a)  │  │ 10.34.28.208/28 (AZ b)   │             │
│  │ [ALB] [Jump Server]     │  │ [ALB]                     │             │
│  └────────────┬────────────┘  └───────────────────────────┘             │
│               │ NAT Gateway                                              │
│               ▼                                                          │
│  PRIVATE SUBNETS (NAT Gateway for internet, no public IPs):             │
│  ┌─────────────────────────┐                                            │
│  │ Seek_Prod_Web_Subnet_01 │                                            │
│  │ 10.34.28.0/28 (AZ a)    │  [SKPTWEBAWSAU01 - Web Server]            │
│  └─────────────────────────┘                                            │
│  ┌─────────────────────────┐                                            │
│  │ Seek_Prod_App_Subnet_01 │                                            │
│  │ 10.34.28.16/28 (AZ a)   │  [SKPTAPPAWSAU01 - App Server]            │
│  └─────────────────────────┘                                            │
│  ┌─────────────────────────┐                                            │
│  │ Seek_Prod_Rm_Subnet_01  │                                            │
│  │ 10.34.28.32/28 (AZ a)   │  [SKPTRMSAWSAU01 - Database]              │
│  └─────────────────────────┘                                            │
│                                                                          │
│  VPC Peering → RamcoNebula VPC (10.33.52.0/23) in ap-southeast-1       │
└──────────────────────────────────────────────────────────────────────────┘
```

### Routing

```hcl
routes_new = {
  "Seek_Prod_Public_Subnet_Rt" = {
    routes = [
      { destination_cidr_block = "0.0.0.0/0", igw_key = "Seek_Prod_Igw" },
      { destination_cidr_block = "10.33.52.0/23", vpc_peering_connection_id_key = "SEEK_PROD-RamcoNebula" }
    ]
  }
  "Seek_Prod_Web_Subnet_Rt" = {
    routes = [
      { destination_cidr_block = "0.0.0.0/0", nat_gateway_key = "Seek_Prod_Ngw_01" },
      { destination_cidr_block = "10.33.52.0/23", vpc_peering_connection_id_key = "SEEK_PROD-RamcoNebula" }
    ]
  }
  # App and RM subnets have same pattern as Web
}
```

**Key difference from KAL:** 
- Public subnet → Internet via IGW (direct)
- Private subnets → Internet via NAT Gateway (no public IP exposed)
- All subnets → RamcoNebula VPC via peering

---

## ec2.tfvars — The Servers

### Two Groups of Servers

**Group 1: NBL (Nebula) — On EXISTING VPC**

| Server | Type | Purpose | Subnet (existing) |
|--------|------|---------|-------------------|
| NBLWEBAWSAUI02 | r6i.large | Web Layer | NBL_PROD_AUS_SUBNET_WEB_02 |
| NBLAPPAWSAUI02 | r6i.large | App Layer | NBL_PROD_AUS_SUBNET_APP_02 |
| NBLPINTAWSAUI01 | r6i.xlarge | NiFi/Integration | NBL_PROD_NIFI_SUBNET_01 |
| NBLIPCAWSAUI01 | r7g.xlarge | IPC | NBL_PROD_IPO_Fargate_Subnet |
| NBLIPOAWSAUI01 | r7g.xlarge | IPO | NBL_PROD_IPO_Fargate_Subnet |
| NBLRMSAWSAUI02 | r6i.xlarge | Database (new) | NBL_PROD_AUS_SUBNET_RM_02 |

**Group 2: SKPT (Seek) — On NEW Terraform-managed VPC**

| Server | Type | Purpose | Subnet (new) |
|--------|------|---------|--------------|
| SKPTWEBAWSAU01 | r6i.large | Web Server | Seek_Prod_Web_Subnet_01 |
| SKPTAPPAWSAU01 | r6i.xlarge | App Server | Seek_Prod_App_Subnet_01 |
| SKPTRMSAWSAU01 | r6i.xlarge | Database | Seek_Prod_Rm_Subnet_01 |
| SKPTJMPAWSAU01 | t3.large | Jump Server | Seek_Prod_Public_Subnet |

### Key Observation: existing_subnet_key vs subnet_key

```hcl
# NBL servers use EXISTING subnets:
"NBLWEBAWSAUI02" = {
  existing_subnet_key = "NBL_PROD_AUS_SUBNET_WEB_02"  ← Pre-existing, just ID
}

# SKPT servers use NEW Terraform-managed subnets:
"SKPTWEBAWSAU01" = {
  subnet_key = "Seek_Prod_Web_Subnet_01"               ← Created by this Terraform
}
```

### existing_security_group_name_keys vs security_group_keys

```hcl
# NBL servers reference EXISTING SGs by name:
"NBLWEBAWSAUI02" = {
  existing_security_group_id_keys = ["NBLWEBAWSAUI02_SG", "NBL_PROD_ILB_INSTACE_SG"]
}

# SKPT servers use NEW Terraform-managed SGs:
"SKPTWEBAWSAU01" = {
  security_group_keys = ["SKPTWEBAWSAU01_SG01"]
}
```

---

## elb.tfvars — Application Load Balancer

```hcl
alb = {
  "SEEK-PROD-TEST-ALB" = {
    enable_deletion_protection = true
    internal                   = false                    # Internet-facing
    load_balancer_type         = "application"
    security_group_keys        = ["SKPTALB_SG01"]
    subnet_keys                = ["Seek_Prod_Public_Subnet", "Seek_Prod_Public_Subnet_02"]
  }
}
```

**What this creates:**

```
     Internet Users
          │
          │ HTTPS (port 443)
          ▼
┌─────────────────────┐
│  SEEK-PROD-TEST-ALB │  (Application Load Balancer)
│  - Internet-facing  │
│  - SSL: *.ramcoes.com │
│  - 2 public subnets │
│  - SG: SKPTALB_SG01 │
└─────────┬───────────┘
          │ HTTPS (port 443)
          ▼
┌─────────────────────┐
│  Target Group       │
│  - Protocol: HTTPS  │
│  - Stickiness: ON   │
│  - Cookie: app_cookie│
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  SKPTWEBAWSAU01     │  (Web Server)
│  - Port 443         │
└─────────────────────┘
```

### Target Group (where traffic goes)

```hcl
target_group = {
  "SEEK-PROD-TEST-ALB" = {
    vpc_key                = "Seek_Prod_VPC"
    target_group_port      = 443
    target_group_protocol  = "HTTPS"
    health_check_path      = "/"
    stickiness_type        = "app_cookie"
    cookie_name            = "SEEK-PROD-APP-COOKIE"  # Session stickiness
  }
}
```

### Listener (what port ALB listens on)

```hcl
lb_listener = {
  "SEEK-PROD-TEST-ALB" = {
    listener_port     = 443
    listener_protocol = "HTTPS"
    ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"  # TLS 1.3
  }
}
```

### Target Attachment (which server receives traffic)

```hcl
target_group_attachements = {
  "SEEK-PROD-TEST-ALB" = {
    port                   = 443
    target_group_arn_key   = "SEEK-PROD-TEST-ALB"
    target_instance_id_key = "SKPTWEBAWSAU01"    # Traffic goes to this server
  }
}
```

---

## sg.tfvars — Security & Traffic

### The Jump Server Pattern

```
Internet → SKPTJMPAWSAU01 (Jump) → RDP to Web/App/RMS servers

The jump server is the ONLY entry point for admin access.
Internal servers don't accept RDP from internet directly.
```

```hcl
# Jump server SG:
"SKPTJMPAWSAU01_SG01" = {
  sg_rules = [
    # INBOUND: Only RSL can RDP to jump server
    { type="ingress", port=3389, cidr=["14.141.1.146/32"], desc="RSL Accops" },

    # OUTBOUND: Jump server can RDP to all internal servers
    { type="egress", port=3389, target=SKPTWEBAWSAU01_SG01, desc="Web" },
    { type="egress", port=3389, target=SKPTAPPAWSAU01_SG01, desc="App" },
    { type="egress", port=3389, target=SKPTRMSAWSAU01_SG01, desc="RMS" },
  ]
}

# Internal servers accept RDP ONLY from jump server:
"SKPTWEBAWSAU01_SG01" = {
  sg_rules = [
    { type="ingress", port=3389, source_sg=SKPTJMPAWSAU01_SG01, desc="Jump" },
  ]
}
```

### ALB → Web Server Flow

```hcl
# ALB accepts HTTPS from internet:
"SKPTALB_SG01" = {
  { type="ingress", port=443, cidr=["0.0.0.0/0"], desc="HTTPS" }
  { type="egress", port=443, target=SKPTWEBAWSAU01_SG01, desc="To Web" }
}

# Web server accepts HTTPS only from ALB:
"SKPTWEBAWSAU01_SG01" = {
  { type="ingress", port=443, source_sg=SKPTALB_SG01, desc="From ALB" }
}
```

---

## disk.tfvars — Storage

| Volume | Size | Type | Attached To | Device |
|--------|------|------|-------------|--------|
| NBLIPCAWSAUI01 | 300 GB | standard | NBLIPCAWSAUI01 | /dev/sdb |
| NBLIPOAWSAUI01 | 300 GB | standard | NBLIPOAWSAUI01 | /dev/sdb |
| SKPTRMSAWSAU01_D01 | 150 GB | standard | SKPTRMSAWSAU01 | /dev/xvdf |
| SKPTRMSAWSAU01_D02 | 50 GB | standard | SKPTRMSAWSAU01 | /dev/xvdg |
| NBLRMSAWSAUI02_D01 | 200 GB | **gp3** (SSD) | NBLRMSAWSAUI02 | /dev/xvdb |
| NBLRMSAWSAUI02_D02 | 150 GB | standard | NBLRMSAWSAUI02 | /dev/xvdc |
| NBLRMSAWSAUI02_D03 | 200 GB | standard | NBLRMSAWSAUI02 | /dev/xvdd |

---

# PART 4: COMPLETE ARCHITECTURE DIAGRAM

```
┌────────────────────────── AWS Account: Nebula (ap-southeast-2) ──────────────────────┐
│                                                                                       │
│  ┌───────── EXISTING VPC: RamcoNebula_PROD_AUS_VPC ──────────────────────────────┐  │
│  │                                                                                │  │
│  │  NBLWEBAWSAUI02 (Web)     NBLAPPAWSAUI02 (App)     NBLRMSAWSAUI02 (DB)       │  │
│  │  NBLPINTAWSAUI01 (NiFi)   NBLIPCAWSAUI01 (IPC)     NBLIPOAWSAUI01 (IPO)     │  │
│  │                                                                                │  │
│  └────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                       │
│  ┌───────── NEW VPC: Seek_Prod_VPC (10.34.28.0/23) ─────────────────────────────┐   │
│  │                                                                                │   │
│  │  PUBLIC:  [ALB] ─── [SKPTJMPAWSAU01 Jump]                                    │   │
│  │              │                  │ RDP                                           │   │
│  │              │ HTTPS            ▼                                              │   │
│  │  PRIVATE: [SKPTWEBAWSAU01] [SKPTAPPAWSAU01] [SKPTRMSAWSAU01]                │   │
│  │           (Web)             (App)             (Database)                       │   │
│  │                                                                                │   │
│  │  NAT GW → Internet (for private subnets)                                     │   │
│  │  VPC Peering → RamcoNebula VPC (10.33.52.0/23) in Singapore                  │   │
│  └────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                       │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

---

# PART 5: HOW TO RUN

```bash
# Navigate to prod Sydney
cd nebula/prod/ap-southeast-2/

# Initialize
terraform init

# Plan
terraform plan \
  -var-file="network.tfvars" \
  -var-file="sg.tfvars" \
  -var-file="ec2.tfvars" \
  -var-file="elb.tfvars" \
  -var-file="disk.tfvars"

# Apply
terraform apply \
  -var-file="network.tfvars" \
  -var-file="sg.tfvars" \
  -var-file="ec2.tfvars" \
  -var-file="elb.tfvars" \
  -var-file="disk.tfvars"
```

**Note:** Nebula prod has 5 var-files (not 4 like KAL) because of the elb.tfvars.

---

# PART 6: KEY DIFFERENCES FROM RAMCOKAL (Summary)

| Feature | RamcoKAL | Nebula |
|---------|----------|--------|
| ALB (Load Balancer) | ❌ No | ✅ Yes (HTTPS, SSL, stickiness) |
| NAT Gateway | ❌ No (all public) | ✅ Yes (private subnets) |
| Existing infrastructure | Minimal | Heavy (VPC, subnets, SGs pre-exist) |
| Multiple sub-clients | Just KAL | NBL + SKPT (Seek) in same state |
| Jump server pattern | Direct RDP to servers | Jump server → internal RDP |
| Cross-account peering | Simple | Complex (remote state + auto-accept) |
| SSL certificates | None | ACM wildcard (*.ramcoes.com) |
| Disk types | All standard | Mix of standard + gp3 (SSD) |
| Server states | All running | Mix (SKPTJMPAWSAU01 = stopped) |
| `routes_new` format | Not used (uses `routes`) | Used (nested routes per RT) |
