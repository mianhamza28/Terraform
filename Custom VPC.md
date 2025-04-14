To create a custom VPC in AWS using Terraform, follow these steps:

### 1. **Set Up the Terraform Configuration**
Create a file named `main.tf` and include the AWS provider:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

provider "aws" {
  region = "us-east-1" # Replace with your desired region
}
```

### 2. **Define the VPC**
Create a VPC with a CIDR block and DNS settings:

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "my-custom-vpc"
  }
}
```

### 3. **Create an Internet Gateway**
Attach an internet gateway to the VPC for public internet access:

```hcl
resource "aws_internet_gateway" "igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "my-igw"
  }
}
```

### 4. **Define Public and Private Subnets**
Create subnets in different Availability Zones (AZs):

```hcl
# Public Subnet
resource "aws_subnet" "public_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a" # Replace with your AZ
  map_public_ip_on_launch = true   # Assign public IP to instances

  tags = {
    Name = "public-subnet"
  }
}

# Private Subnet
resource "aws_subnet" "private_subnet" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b" # Replace with your AZ

  tags = {
    Name = "private-subnet"
  }
}
```

### 5. **Configure Route Tables**
- **Public Route Table**: Routes traffic to the internet gateway.
- **Private Route Table**: Default local route (optional: add NAT gateway for outbound traffic).

```hcl
# Public Route Table
resource "aws_route_table" "public_rt" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.igw.id
  }

  tags = {
    Name = "public-route-table"
  }
}

# Associate Public Subnet with Public Route Table
resource "aws_route_table_association" "public_assoc" {
  subnet_id      = aws_subnet.public_subnet.id
  route_table_id = aws_route_table.public_rt.id
}

# Private Route Table (optional: add NAT gateway route here)
resource "aws_route_table" "private_rt" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "private-route-table"
  }
}

# Associate Private Subnet with Private Route Table
resource "aws_route_table_association" "private_assoc" {
  subnet_id      = aws_subnet.private_subnet.id
  route_table_id = aws_route_table.private_rt.id
}
```

### 6. **(Optional) Add NAT Gateway for Private Subnet Outbound Access**
If you need private instances to access the internet (e.g., for updates), add a NAT gateway in the public subnet:

```hcl
# Allocate Elastic IP for NAT Gateway
resource "aws_eip" "nat_eip" {
  domain = "vpc"
}

# Create NAT Gateway in Public Subnet
resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = aws_subnet.public_subnet.id

  tags = {
    Name = "my-nat-gateway"
  }
}

# Add Route to Private Route Table for NAT Gateway
resource "aws_route" "private_nat_route" {
  route_table_id         = aws_route_table.private_rt.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.nat.id
}
```

### 7. **Define Security Groups**
Example security group allowing SSH and HTTP:

```hcl
resource "aws_security_group" "web_sg" {
  name        = "web-sg"
  description = "Allow HTTP/HTTPS and SSH traffic"
  vpc_id      = aws_vpc.my_vpc.id

  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "web-sg"
  }
}
```

### 8. **Output Important IDs**
Add outputs to retrieve VPC and subnet IDs:

```hcl
output "vpc_id" {
  value = aws_vpc.my_vpc.id
}

output "public_subnet_id" {
  value = aws_subnet.public_subnet.id
}

output "private_subnet_id" {
  value = aws_subnet.private_subnet.id
}
```

### 9. **Deploy the Infrastructure**
Run the following commands:

```bash
terraform init    # Initialize Terraform and providers
terraform plan    # Preview changes
terraform apply   # Deploy the VPC and resources
```

### Explanation:
- **VPC**: The core network with a `/16` CIDR block.
- **Subnets**: Public (with internet access) and private (isolated) subnets.
- **Internet Gateway**: Enables public subnet communication with the internet.
- **Route Tables**: Direct traffic between subnets and gateways.
- **NAT Gateway**: Allows private subnet instances to access the internet (optional).
- **Security Group**: Controls inbound/outbound traffic for instances.

This setup creates a highly available and isolated network environment in AWS. Adjust CIDR blocks, AZs, and security group rules as needed.
