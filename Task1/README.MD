# AWS Infrastructure Provisioning Using Terraform

Provisions a basic AWS environment with Terraform: a VPC, public subnet, internet gateway, route table, security group, SSH key pair and an EC2 instance running Apache. The configuration is parameterised with variables and exposes useful values as outputs.

## Architecture

```
                 Internet
                    |
             +------v------+
             | Internet GW |
             +------+------+
   +----------------v-----------------+
   | VPC (10.0.0.0/16)                |
   |  Route table: 0.0.0.0/0 -> IGW   |
   |  +----------------------------+  |
   |  | Public subnet (10.0.1.0/24)|  |
   |  |  +----------------------+  |  |
   |  |  | EC2 (Amazon Linux)   |  |  |
   |  |  | + Security Group     |  |  |
   |  |  |   SSH 22, HTTP 80    |  |  |
   |  |  +----------------------+  |  |
   |  +----------------------------+  |
   +----------------------------------+
```

## Project structure

```
.
├── providers.tf       # Terraform and AWS provider configuration
├── variables.tf       # Input variables with defaults
├── main.tf            # VPC, networking, security group, key pair, EC2
├── outputs.tf         # Output values (IP, URL, SSH command)
├── terraform.tfvars   # Your variable values (not committed)
└── .gitignore
```

## Resources created

| Resource | Purpose |
|---|---|
| `aws_vpc` | Isolated virtual network |
| `aws_internet_gateway` | Connects the VPC to the internet |
| `aws_subnet` | Public subnet with auto-assigned public IPs |
| `aws_route_table` and association | Routes outbound traffic to the internet gateway |
| `aws_security_group` | Allows SSH (22) and HTTP (80) inbound, all outbound |
| `aws_key_pair` | Uploads the public SSH key |
| `aws_instance` | EC2 server that installs Apache via `user_data` |

A data source (`aws_ami`) fetches the latest Amazon Linux 2023 AMI, so no AMI ID is hardcoded.

## Prerequisites

- AWS account and an IAM user with programmatic access
- [AWS CLI](https://aws.amazon.com/cli/) configured with `aws configure`
- [Terraform](https://developer.hashicorp.com/terraform/downloads) 1.5.0 or later
- An SSH key pair:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/terraform-key
```

Verify your credentials:

```bash
aws sts get-caller-identity
```

## Variables

| Variable | Description | Default |
|---|---|---|
| `aws_region` | AWS region | `ap-south-1` |
| `project_name` | Name prefix for all resources | `devops-sakha` |
| `environment` | Environment tag | `dev` |
| `vpc_cidr` | VPC CIDR block | `10.0.0.0/16` |
| `public_subnet_cidr` | Public subnet CIDR block | `10.0.1.0/24` |
| `instance_type` | EC2 instance type | `t3.micro` |
| `allowed_ssh_cidr` | CIDR allowed to SSH | `0.0.0.0/0` (change to your IP) |
| `public_key_path` | Path to the SSH public key | `~/.ssh/terraform-key.pub` |

Create a `terraform.tfvars` file to override the defaults:

```hcl
aws_region       = "ap-south-1"
project_name     = "devops-sakha"
environment      = "dev"
instance_type    = "t3.micro"
allowed_ssh_cidr = "YOUR.PUBLIC.IP.HERE/32"
```

Find your public IP with `curl ifconfig.me`. Restricting SSH to your own IP is a security best practice.

## Usage

```bash
# 1. Download the provider and initialise the working directory
terraform init

# 2. Format and validate the configuration
terraform fmt
terraform validate

# 3. Preview the changes
terraform plan

# 4. Create the infrastructure
terraform apply

# 5. View outputs
terraform output
```

## Testing

| Test | Command / Action | Expected result |
|---|---|---|
| Outputs | `terraform output` | Instance ID, public IP, URL shown |
| Web server | Open the `website_url` output in a browser | "Deployed with Terraform on AWS" |
| SSH | Run the `ssh_command` output | Shell prompt on the instance |
| Console | AWS Console, EC2 and VPC | Resources visible with the project tags |
| State | `terraform state list` | All resources listed |

The web page can take 1 to 2 minutes to appear because `user_data` runs on first boot.

## Cleanup

Destroy everything after testing to avoid charges:

```bash
terraform destroy
```

Then confirm in the AWS Console that the EC2 instance, VPC and related resources are gone.

## Outputs

| Output | Description |
|---|---|
| `instance_id` | ID of the EC2 instance |
| `instance_public_ip` | Public IP address |
| `instance_public_dns` | Public DNS name |
| `vpc_id` | ID of the VPC |
| `website_url` | URL to test the web server |
| `ssh_command` | Ready-made SSH command |

## Security notes

- Credentials are read from the AWS CLI configuration and are never hardcoded.
- State files, `.tfvars` files and SSH keys are excluded from Git through `.gitignore`.
- Restrict `allowed_ssh_cidr` to your own IP instead of `0.0.0.0/0`.


