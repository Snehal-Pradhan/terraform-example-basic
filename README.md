# terraform-example-basic

Production-grade AWS infrastructure provisioned with Terraform — no hardcoded values, dynamic AMI lookup, remote state with locking, and prebuilt community modules.

## Architecture

```
┌──────────────────────────── VPC 10.0.0.0/16 ────────────────────────────┐
│                                                                         │
│  Public Subnets                    Private Subnets                      │
│  ├─ 10.0.1.0/24 (eu-west-1a)      ├─ 10.0.11.0/24 (eu-west-1a)          │
│  │  └─ EC2 (t3.micro, Ubuntu)     └─ 10.0.12.0/24 (eu-west-1b)          │
│  └─ 10.0.2.0/24 (eu-west-1b)                                            │
│       │                              NAT Gateway ◄── EIP                │
│       └── Internet Gateway                                              │
└─────────────────────────────────────────────────────────────────────────┘
```

## What gets created

| Resource | Details |
|----------|---------|
| VPC | CIDR from variable, DNS hostnames enabled |
| Public subnets | 2 AZs, route to IGW |
| Private subnets | 2 AZs, route to NAT |
| Internet Gateway | Public internet access |
| NAT Gateway | Outbound internet for private subnets |
| Security Group | Inbound: SSH from allowed CIDRs only. Outbound: all |
| EC2 instance | Ubuntu 24.04 LTS, t3.micro, encrypted gp3 root volume |

## Project structure

```
.
├── versions.tf        # Terraform + provider version pinning
├── variables.tf       # All inputs with validation rules
├── providers.tf       # AWS provider + default tags
├── backend.tf         # Partial S3 backend config
├── backend.hcl        # Backend values (gitignored)
├── data.tf            # Dynamic Ubuntu AMI + AZ lookup
├── main.tf            # VPC, SG, EC2 via prebuilt modules
├── outputs.tf         # Useful outputs
├── dev.tfvars         # Environment values (gitignored)
├── dev.tfvars.example # Template for teammates
└── bootstrap/         # One-time S3 state bucket setup
```

## Prerequisites

- Terraform >= 1.9.0
- AWS CLI configured with credentials
- An EC2 key pair in your target region (or plan for SSM Session Manager)

## Usage

### 1. Bootstrap remote state (one time only)

```bash
cd bootstrap
# edit terraform.tfvars with a globally unique bucket name
terraform init
terraform apply
cd ..
```

### 2. Configure backend

Edit `backend.hcl` with your state bucket name:

```hcl
bucket         = "your-tfstate-bucket"
key            = "env:/dev/terraform.tfstate"
region         = "eu-west-1"
encrypt        = true
use_lockfile   = true
```

### 3. Deploy

```bash
cp dev.tfvars.example dev.tfvars   # then edit with your values
terraform init -backend-config=backend.hcl
terraform fmt
terraform validate
terraform plan -var-file=dev.tfvars
terraform apply -var-file=dev.tfvars
```

### 4. Connect

```bash
ssh -i your-key.pem ubuntu@$(terraform output -raw instance_public_ip)
```

### 5. Tear down

```bash
terraform destroy -var-file=dev.tfvars
```

## Design principles

- **No hardcoding** — region, CIDRs, instance type, volume size, SSH CIDRs all flow from `variables.tf`
- **Dynamic AMI** — latest Ubuntu 24.04 LTS resolved from Canonical (`099720109477`) at plan time
- **Input validation** — bad values fail at plan, not apply
- **Remote state** — S3 with native S3 locking (`use_lockfile`), versioning, KMS encryption
- **Prebuilt modules** — [VPC](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws), [Security Group](https://registry.terraform.io/modules/terraform-aws-modules/security-group/aws), [EC2](https://registry.terraform.io/modules/terraform-aws-modules/ec2-instance/aws)
- **Default tags** — every resource tagged with Project, Environment, ManagedBy

## Secrets never go in git

`*.tfvars`, `*.pem`, `backend.hcl`, and all state files are gitignored. Teammates copy `dev.tfvars.example` and fill in their own values.
