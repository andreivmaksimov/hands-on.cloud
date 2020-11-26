---
title: 'Terraform recipe - Managing AWS VPC - Creating Private Subnets'
date: '2020-10-04'
image: 'Terraform-recipe-Managing-AWS-VPC-Creating-Private-Subnets'
tags:
  - vpc
  - terraform
categories:
  - AWS
  - Terraform
authors:
  - Andrei Maksimov
---

In the previous article ([Terraform recipe – Managing AWS VPC – Creating Public Subnet](https://hands-on.cloud/terraform-recipe-managing-aws-vpc-creating-public-subnet/)), we’ve used Terraform to create a VPC, Internet Gateway, and Route Table to form Public Subnet. Also, we’ve tested our configuration by SSH-ing to the instance, which we’ve launched in our Public Subnet.

In this article, we’ll add to our VPC a couple of Private Subnets:

* Fully isolated Private Subnet.
* NAT-ed Private Subnet.

You may find a complete example for `.tf` file in my [GitHub repo](https://github.com/andreivmaksimov/terraform-recipe-managing-aws-vpc-creating-private-subnets).

## VPC with the public subnet.

Here’s a complete example from the previous article:

```hcl
resource "aws_vpc" "my_vpc" {
  cidr_block       = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = {
    Name = "My VPC"
  }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.0.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "Public Subnet"
  }
}

resource "aws_internet_gateway" "my_vpc_igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "My VPC - Internet Gateway"
  }
}

resource "aws_route_table" "my_vpc_us_east_1a_public" {
    vpc_id = aws_vpc.my_vpc.id

    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.my_vpc_igw.id
    }

    tags = {
        Name = "Public Subnet Route Table."
    }
}

resource "aws_route_table_association" "my_vpc_us_east_1a_public" {
    subnet_id = aws_subnet.public.id
    route_table_id = aws_route_table.my_vpc_us_east_1a_public.id
}

resource "aws_security_group" "allow_ssh" {
  name        = "allow_ssh_sg"
  description = "Allow SSH inbound connections"
  vpc_id = aws_vpc.my_vpc.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port       = 0
    to_port         = 0
    protocol        = "-1"
    cidr_blocks     = ["0.0.0.0/0"]
  }

  tags = {
    Name = "allow_ssh_sg"
  }
}

resource "aws_instance" "my_instance" {
  ami           = "ami-0ac019f4fcb7cb7e6"
  instance_type = "t2.micro"
  key_name = "Lenovo T410"
  vpc_security_group_ids = [ aws_security_group.allow_ssh.id ]
  subnet_id = aws_subnet.public.id
  associate_public_ip_address = true

  tags = {
    Name = "My Instance"
  }
}
```

## NAT-ed private subnet.

Instances launched in this subnet will be able to communicate with any services within VPC and go to the Internet using special AWS service NAT Gateway.

AWS recommends using this type of network if you’re running, for example, a public-facing web application, and you don’t want to make your back-end servers publicly accessible. A typical example is a multi-tier website, with web servers in a public subnet and the database servers in a private subnet. You can set up security and routing so that the web servers can communicate with the database servers.

Here’s how the infrastructure looks like:

{{< my-picture name="Terraform-recipe-Managing-AWS-VPC-Public-and-Private-NAT-Subnet" >}}

To extend our VPC with this NAT-ed Private network, we need to create the following resources:

* VPC NAT Gateway.
* Subnet.
* Route Table with route to VPC NAT Gateway.
* Association between Route Table and Subnet.

Let’s begin from Subnet, by declaring additional [aws_subnet](https://www.terraform.io/docs/providers/aws/r/subnet.html) resource:

```hcl
resource "aws_subnet" "nated" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.1.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "NAT-ed Subnet"
  }
}
```

Now let’s create NAT Gateway in a public subnet by declaring [aws_nat_gateway](https://www.terraform.io/docs/providers/aws/r/nat_gateway.html) and [aws_eip](https://www.terraform.io/docs/providers/aws/r/eip.html).

You can not launch NAT Gateway without Elastic IP address associated with it, that’s why [aws_eip](https://www.terraform.io/docs/providers/aws/r/eip.html) required:

```hcl
resource "aws_eip" "nat_gw_eip" {
  vpc = true
}

resource "aws_nat_gateway" "gw" {
  allocation_id = aws_eip.nat_gw_eip.id
  subnet_id     = aws_subnet.public.id
}
```

Now we need to create Main Route Table by declaring additional already know for you resources [aws_route_table](https://www.terraform.io/docs/providers/aws/r/route_table.html) and associate it with our NAT-ed Subnet ([aws_route_table_association](https://www.terraform.io/docs/providers/aws/r/route_table_association.html)):

```hcl
resource "aws_route_table" "my_vpc_us_east_1a_nated" {
    vpc_id = aws_vpc.my_vpc.id

    route {
        cidr_block = "0.0.0.0/0"
        nat_gateway_id = aws_nat_gateway.gw.id
    }

    tags = {
        Name = "Main Route Table for NAT-ed subnet"
    }
}

resource "aws_route_table_association" "my_vpc_us_east_1a_nated" {
    subnet_id = aws_subnet.nated.id
    route_table_id = aws_route_table.my_vpc_us_east_1a_nated.id
}
```

Now we’re ready to create private servers in our Private NAT-ed Subnet and they will have access to Internet, but will not be visible from the Internet.

I guess, you may do it using the instructions from the previous article.

## Fully isolated Private subnet.

Instances launched in this subnet will be able to communicate with instances within VPC, but will not be able to go to the Internet.

This type of networks are commonly used when you need to connect your organization network to the AWS cloud and want more strict control of network boundaries. Or, you do not want to give some of your servers access to the Internet.

Here’s how the infrastructure looks like:

{{< my-picture name="Terraform-recipe-Managing-AWS-VPC-Public-Private-NAT-and-Private-fully-isolated-Subnets" >}}

To implement fully isolated Private Subnet we need to create the following resources:

* Subnet.
* Route Table.
* Association between Route Table and Subnet.

Let’s start from Subnet:

```hcl
resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = "10.0.2.0/24"
  availability_zone = "us-east-1a"

  tags = {
    Name = "Isolated Private Subnet"
  }
}
```

Next we need to create additional Route Table with no routes declaration and associate it with our `private` Subnet:

```hcl
resource "aws_route_table" "my_vpc_us_east_1a_private" {
    vpc_id = aws_vpc.my_vpc.id

    tags = {
        Name = "Local Route Table for Isolated Private Subnet"
    }
}

resource "aws_route_table_association" "my_vpc_us_east_1a_private" {
    subnet_id = aws_subnet.private.id
    route_table_id = aws_route_table.my_vpc_us_east_1a_private.id
}
```

## Summary.

In this article you’ve learned how to create different types of AWS Private Subnets in your environment and differences between them. Hope, this article been helpful for you!

Stay tuned!
