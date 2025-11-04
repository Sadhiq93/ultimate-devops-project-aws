# Code explanation for VPC module

## **1. Creating a VPC**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name                                           = "${var.cluster_name}-vpc"
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
  }
}
```
- Creates a **VPC (Virtual Private Cloud)** using the CIDR block specified in `var.vpc_cidr`.
- Enables **DNS support** and **DNS hostnames** for instances in the VPC.
- Adds tags for Kubernetes cluster integration and identification.

---

## **2. Creating Private Subnets**
```hcl
resource "aws_subnet" "private" {
  count             = length(var.private_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  tags = {
    Name                                           = "${var.cluster_name}-private-${count.index + 1}"
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
    "kubernetes.io/role/internal-elb"              = "1"
  }
}
```
- Creates multiple **private subnets** based on `private_subnet_cidrs`.
- Each subnet is assigned an **availability zone** from `availability_zones`.
- Tags define Kubernetes cluster association and **internal load balancer** role.

---

## **3. Creating Public Subnets**
```hcl
resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name                                           = "${var.cluster_name}-public-${count.index + 1}"
    "kubernetes.io/cluster/${var.cluster_name}"    = "shared"
    "kubernetes.io/role/elb"                       = "1"
  }
}
```
- Creates multiple **public subnets** based on `public_subnet_cidrs`.
- Assigns **public IP addresses** to instances on launch.
- Tags specify Kubernetes cluster association and **public load balancer** role.

---

## **4. Creating an Internet Gateway**
```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.cluster_name}-igw"
  }
}
```
- Creates an **Internet Gateway** to enable internet access for public subnets.

---

## **5. Creating NAT Gateway and Elastic IPs**
```hcl
resource "aws_eip" "nat" {
  count = length(var.public_subnet_cidrs)
  domain = "vpc"

  tags = {
    Name = "${var.cluster_name}-nat-${count.index + 1}"
  }
}

resource "aws_nat_gateway" "main" {
  count         = length(var.public_subnet_cidrs)
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "${var.cluster_name}-nat-${count.index + 1}"
  }
}
```
- **Elastic IPs (EIPs)** are created for the **NAT gateways**.
- **NAT gateways** allow private subnet instances to access the internet securely.
- Each NAT gateway is associated with a **public subnet**.

---

## **6. Creating Route Tables**
```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.cluster_name}-public"
  }
}
```
- **Public route table** routes internet traffic (`0.0.0.0/0`) to the Internet Gateway.

```hcl
resource "aws_route_table" "private" {
  count  = length(var.private_subnet_cidrs)
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "${var.cluster_name}-private-${count.index + 1}"
  }
}
```
- **Private route tables** route internet traffic through **NAT gateways**.

---

## **7. Associating Route Tables with Subnets**
```hcl
resource "aws_route_table_association" "private" {
  count          = length(var.private_subnet_cidrs)
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```
- Associates each **private subnet** with a **private route table**.

```hcl
resource "aws_route_table_association" "public" {
  count          = length(var.public_subnet_cidrs)
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}
```
- Associates all **public subnets** with the **public route table**.

------------------------------------

exapmple:


cidr_block = var.public_subnet_cidrs[count.index] expalin terraform line count.index

ChatGPT said:
Sure — let’s break down this Terraform line step by step:


cidr_block = var.public_subnet_cidrs[count.index]
This line is assigning a CIDR block to a subnet resource in Terraform. Let’s focus on the count.index part, since that’s what you asked about.

🔹 count.index — What it means
count.index is a built-in Terraform variable that represents the current iteration number when you are using the count meta-argument in a resource block.
When you use count, Terraform will create multiple instances of a resource — one for each index value, starting at 0.
Example:


resource "aws_subnet" "public" {
  count      = length(var.public_subnet_cidrs)
  vpc_id     = aws_vpc.main.id
  cidr_block = var.public_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]
}
Here’s what happens:
* count = length(var.public_subnet_cidrs) → if there are 3 CIDRs, Terraform makes 3 subnets.
* On each iteration:
    * When count.index = 0 → it uses var.public_subnet_cidrs[0]
    * When count.index = 1 → it uses var.public_subnet_cidrs[1]
    * When count.index = 2 → it uses var.public_subnet_cidrs[2]
So count.index is like the loop counter in a for-loop.

🔹 In summary
* count.index → current resource instance number (starting from 0).
* var.public_subnet_cidrs[count.index] → picks the matching CIDR from the list variable public_subnet_cidrs.
* This lets you dynamically create multiple subnets, each with a unique CIDR block.
---------------------------------------------------------------

You said:
give me var file for this

ChatGPT said:
Sure! Here's an example variables.tf file that defines the variable public_subnet_cidrs (and a few related ones commonly used with it, like vpc_cidr and azs):

📄 variables.tf


# VPC CIDR block
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

# Public subnet CIDR blocks
variable "public_subnet_cidrs" {
  description = "List of CIDR blocks for public subnets"
  type        = list(string)
  default     = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]
}

# Availability zones to place subnets in
variable "azs" {
  description = "List of availability zones"
  type        = list(string)
  default     = [
    "us-east-1a",
    "us-east-1b",
    "us-east-1c"
  ]
}

# AWS region
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

Example usage in your main configuration (main.tf)


provider "aws" {
  region = var.region
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
}

resource "aws_subnet" "public" {
  count             = length(var.public_subnet_cidrs)
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnet_cidrs[count.index]
  availability_zone = var.azs[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "PublicSubnet-${count.index + 1}"
  }
}

✅ How it works:
* The variable file defines lists of CIDRs and availability zones.
* Terraform loops over them using count.index.
* Each subnet gets a unique CIDR and AZ.


