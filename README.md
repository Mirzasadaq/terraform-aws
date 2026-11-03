# Terraform AWS Practice Repository

Hands-on Terraform labs where I practice Infrastructure as Code (IaC) on AWS, building from a first EC2 instance up to variables, remote state, meta-arguments, dependencies and provisioners. Each lab lives in its own folder and can be run independently.

![Terraform](https://img.shields.io/badge/Terraform-IaC-7B42BC?logo=terraform&logoColor=white)
![AWS](https://img.shields.io/badge/AWS-ap--south--1-FF9900?logo=amazonaws&logoColor=white)
![Provider](https://img.shields.io/badge/aws%20provider-6.59%20--%206.62-blue)

---

## Table of Contents

- [Labs at a Glance](#labs-at-a-glance)
- [Repository Structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [How to Run a Lab](#how-to-run-a-lab)
- [Lab Details](#lab-details)
- [Key Learnings](#key-learnings)
- [Security Notes](#security-notes)
- [About Me](#about-me)

---

## Labs at a Glance

| Lab | Topic | Concepts Practiced |
|-----|-------|--------------------|
| [001](./terraform_lab_001) | First resources | Provider pinning, `aws_instance`, `aws_s3_bucket`, `aws_vpc` |
| [002](./terraform_lab_002) | Variables, data source, remote backend | `variable`, `terraform.tfvars`, `output`, `data "aws_ami"`, S3 backend with state locking |
| [003](./terraform_lab_003-meta-argument-count) | `count` meta-argument | Multiple identical resources, `count.index` |
| [004](./terraform_lab_004-meta-argument-for_each) | `for_each` meta-argument | Map variable, `each.key` / `each.value` |
| [005](./terraform_lab_005-meta-argument-alias) | Provider `alias` | Multi-region deployment (Mumbai and N. Virginia) |
| [006](./terraform_lab_006-dependency) | Dependencies | Implicit vs explicit (`depends_on`), community module |
| [007](./terraform_lab_007-provisioners) | Provisioners | `local-exec`, `remote-exec`, `tls_private_key`, `local_file` |

---

## Repository Structure

```text
.
├── terraform_lab_001/
│   ├── provider.tf
│   └── resource.tf
├── terraform_lab_002/
│   ├── backend.tf
│   ├── output.tf
│   ├── provider.tf
│   ├── resource.tf
│   ├── terraform.tfvars
│   └── variable.tf
├── terraform_lab_003-meta-argument-count/
│   └── count.tf
├── terraform_lab_004-meta-argument-for_each/
│   └── for_each.tf
├── terraform_lab_005-meta-argument-alias/
│   └── alias.tf
├── terraform_lab_006-dependency/
│   ├── explicit-dependency/
│   │   └── resource.tf
│   └── implicit-dependency/
│       └── resource.tf
├── terraform_lab_007-provisioners/
│   ├── local-exec/
│   │   └── resource.tf
│   └── remote-exec/
│       └── resource.tf
└── .gitignore
```

Each lab also contains a `.terraform.lock.hcl` file that pins the provider versions used.

---

## Prerequisites

- [Terraform](https://developer.hashicorp.com/terraform/install) 1.10 or newer (needed for S3 native state locking with `use_lockfile`)
- [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) configured with credentials (`aws configure`)
- An AWS account with permissions for EC2, S3, VPC, SQS and Elastic IP
- Default region used in these labs: `ap-south-1` (Mumbai)

> Never hard-code AWS access keys in `.tf` files. Use `aws configure` or environment variables.

---

## How to Run a Lab

```bash
# 1. Go to the lab folder
cd terraform_lab_002

# 2. Download providers and initialise the backend
terraform init

# 3. Check formatting and validity
terraform fmt
terraform validate

# 4. Preview the changes
terraform plan

# 5. Create the infrastructure
terraform apply

# 6. Delete everything when done (avoid unexpected AWS bills)
terraform destroy
```

---

## Lab Details

### Lab 001: First Terraform Resources

Provisions the three basic building blocks of AWS from a single configuration.

- Pins the AWS provider with a `terraform { required_providers { ... } }` block (`hashicorp/aws` 6.59.0)
- Creates 5 EC2 instances (`t3.micro`) using `count`
- Creates an S3 bucket and a VPC (`10.0.0.0/16`)

### Lab 002: Variables, Data Sources, Outputs and Remote Backend

Refactors Lab 001 so nothing is hard-coded.

- **Input variables** (`variable.tf`) with values supplied through `terraform.tfvars`
- **Data source** `aws_ami` that automatically looks up the latest Amazon Linux 2023 AMI instead of hard-coding an AMI ID
- **Outputs** (`output.tf`) that print the private and public IPs of all instances using the splat expression `[*]`
- **Remote backend** (`backend.tf`): state is stored in an S3 bucket with native locking (`use_lockfile = true`), so no DynamoDB table is needed

```hcl
terraform {
  backend "s3" {
    bucket       = "<your-state-bucket>"
    key          = "dev/terraform.tfstate"
    region       = "ap-south-1"
    use_lockfile = true
  }
}
```

> The S3 state bucket must exist before running `terraform init`. Replace the bucket name with your own.

### Lab 003: `count` Meta-Argument

Creates multiple identical EC2 instances from a single resource block and names them with `count.index` (`ec2server0`, `ec2server1`). The file also keeps a commented, variable-driven version (`count = var.instance_count`) for comparison.

### Lab 004: `for_each` Meta-Argument

Creates a different server for each entry in a map, a typical DevOps toolchain layout:

| Key | Instance Type |
|-----|---------------|
| `jenkins` | `t3.micro` |
| `nexus` | `t3.small` |
| `sonarqube` | `c7i-flex.large` |

Uses `each.key` for tags and `each.value` for the instance type. Unlike `count`, resources are tracked by key, so removing one entry does not shift or recreate the others.

### Lab 005: Provider Alias (Multi-Region)

Defines two AWS providers, the default one for `ap-south-1` and an aliased one (`alias = "us"`) for `us-east-1`, and deploys an EC2 instance in each region by setting `provider = aws.us` on the resource. AMI IDs are region-specific, so each region uses its own AMI.

### Lab 006: Resource Dependencies

**Implicit dependency** (`implicit-dependency/`)
An Elastic IP references `aws_instance.example_a.id`, so Terraform automatically creates the instance first and attaches the EIP to it. Terraform builds the dependency graph from references.

**Explicit dependency** (`explicit-dependency/`)
Uses `depends_on` where there is no direct reference:
- The EC2 instance waits for the S3 bucket
- The SQS module ([`terraform-aws-modules/sqs/aws`](https://registry.terraform.io/modules/terraform-aws-modules/sqs/aws/latest), v5.2.2) waits for both the instance and the bucket

### Lab 007: Provisioners

**`local-exec`** runs a command on the machine where Terraform runs. After the instance is created, its public IP is appended to a local file:

```hcl
provisioner "local-exec" {
  command = "echo ${self.public_ip} >> ec2_ips.txt"
}
```

**`remote-exec`** runs commands on the new server over SSH. This lab builds an end-to-end MySQL server:

1. `tls_private_key` generates an RSA 4096 key pair
2. `aws_key_pair` uploads the public key to AWS
3. `aws_security_group` allows SSH (22) and MySQL (3306)
4. `aws_instance` launches an Ubuntu server and installs MySQL through the `remote-exec` provisioner
5. `local_file` saves the private key locally with `0600` permissions
6. An `output` prints the server's public IP

---

## Key Learnings

- Pinning provider versions and committing `.terraform.lock.hcl` for reproducible runs
- Making configurations reusable with variables, `tfvars`, data sources and outputs
- Storing state remotely in S3 with locking instead of keeping it on a laptop
- Choosing between `count` (identical copies) and `for_each` (distinct, key-based resources)
- Working across regions with provider aliases
- Understanding how Terraform builds its dependency graph, and when `depends_on` is needed
- Using provisioners, and knowing they are a last resort compared with user data or configuration management tools
- Using a community module from the Terraform Registry

---

## Security Notes

These are learning labs, not production configurations.

- **Lab 007 (`remote-exec`)** opens SSH (22) and MySQL (3306) to `0.0.0.0/0` and uses a demo database password in the script. In a real environment I would restrict the CIDR ranges to known IPs, keep MySQL in a private subnet, and fetch credentials from AWS Secrets Manager or SSM Parameter Store.
- The generated private key (`mykey.pem`) and any `*.tfstate` files must **never** be committed (see `.gitignore`).
- Always run `terraform destroy` after practising to avoid unnecessary AWS charges.

---

## About Me

**Mirza Sadique Baig**: aspiring DevOps Engineer, learning and practising AWS, Terraform and CI/CD tools.

- GitHub: [Mirzasadaq](https://github.com/Mirzasadaq)
- LinkedIn: [mirza-devops-engineer](https://www.linkedin.com/in/mirza-devops-engineer/)
- Email: mirzasadaq@gmail.com

If you find this repository helpful, feel free to give it a star.
