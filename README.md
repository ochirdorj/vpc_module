# vpc_module

A Terraform module that provisions a production-ready AWS VPC with public and private subnets, an Internet Gateway, a NAT Gateway, and associated route tables. The subnet tagging is Kubernetes/EKS-compatible out of the box.

## Architecture

```
VPC
├── Internet Gateway
├── Public Subnet 1 (AZ-1)  ──┐
├── Public Subnet 2 (AZ-2)  ──┤── Public Route Table → Internet Gateway
│       └── NAT Gateway (AZ-1 EIP)
├── Private Subnet 1 (AZ-1) ──┐
└── Private Subnet 2 (AZ-2) ──┴── Private Route Table → NAT Gateway
```

## Resources Created

| Resource | Count | Description |
|---|---|---|
| `aws_vpc` | 1 | VPC with DNS hostnames and DNS support enabled |
| `aws_internet_gateway` | 1 | Attached to the VPC |
| `aws_subnet` (public) | 2 | Public subnets; `map_public_ip_on_launch = true` |
| `aws_subnet` (private) | 2 | Private subnets; no public IP on launch |
| `aws_eip` | 1 | Elastic IP for the NAT Gateway |
| `aws_nat_gateway` | 1 | Placed in the first public subnet |
| `aws_route_table` (public) | 1 | Routes `0.0.0.0/0` to the Internet Gateway |
| `aws_route_table` (private) | 1 | Routes `0.0.0.0/0` to the NAT Gateway |
| `aws_route_table_association` | 4 | 2 public + 2 private subnet associations |

## Requirements

| Name | Version |
|---|---|
| Terraform | >= 1.0 |
| AWS Provider | ~> 6.36 |

## Usage

```hcl
module "vpc" {
  source = "./vpc_module"

  project             = "my-app"
  vpc_cidr            = "10.0.0.0/16"
  availability_zones  = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]

  tags = {
    Environment = "production"
    Team        = "platform"
  }
}
```

## Input Variables

| Name | Type | Required | Description |
|---|---|---|---|
| `project` | `string` | yes | Project name used as a prefix for all resource names |
| `vpc_cidr` | `string` | yes | CIDR block for the VPC (e.g. `10.0.0.0/16`) |
| `availability_zones` | `list(string)` | yes | List of two availability zones |
| `public_subnet_cidrs` | `list(string)` | yes | CIDR blocks for the two public subnets |
| `private_subnet_cidrs` | `list(string)` | yes | CIDR blocks for the two private subnets |
| `tags` | `map(string)` | yes | Common tags applied to all resources |

## Outputs

| Name | Description |
|---|---|
| `vpc_id` | ID of the VPC |
| `vpc_cidr` | CIDR block of the VPC |
| `public_subnet_ids` | List of public subnet IDs |
| `private_subnet_ids` | List of private subnet IDs |
| `nat_gateway_id` | ID of the NAT Gateway |
| `nat_gateway_ip` | Public Elastic IP address of the NAT Gateway |
| `internet_gateway_id` | ID of the Internet Gateway |
| `public_route_table_id` | ID of the public route table |
| `private_route_table_id` | ID of the private route table |

## Resource Naming

All resources are named using the pattern `{project}-{resource}`, for example:

- VPC: `my-app-vpc`
- Internet Gateway: `my-app-igw`
- Public subnets: `my-app-public-subnet-1`, `my-app-public-subnet-2`
- Private subnets: `my-app-private-subnet-1`, `my-app-private-subnet-2`
- NAT EIP: `my-app-nat-eip`
- NAT Gateway: `my-app-nat-gw`
- Route tables: `my-app-public-rt`, `my-app-private-rt`

## Kubernetes / EKS Compatibility

Public subnets are tagged with `kubernetes.io/role/elb = 1` and private subnets with `kubernetes.io/role/internal-elb = 1`, allowing the AWS Load Balancer Controller to automatically discover and use these subnets.

## License

See [LICENSE](LICENSE).
